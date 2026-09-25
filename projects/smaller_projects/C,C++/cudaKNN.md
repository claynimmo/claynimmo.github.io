---
layout: default
title: Portfolio |  CUDA Brute-Force KNN
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# CUDA GPU Accelerated Brute Force KNN Algorithm
The cuda code was made to massively increase the performance of another sequential program, where the KNN algorithm went from 4 minutes to half a second. Streams where implemented as an attempt to increase occupancy (as intial profiling showed it was low), but this made no impact, and so is set to one to essentially disable it. The version that fixed the occupancy is the topKKernelParallel, where both the improved and original implementation are included for use. The program must be run in another program, first calling initDataset, then computeTopKBatch, then freeDataset.
```cpp
#include <cuda_runtime.h>
#include <cstdio>
#include <cstdlib>
#include <float.h>

#define NUM_STREAMS 1

static double* d_data = nullptr;
static double* d_sources = nullptr;
static double* d_out = nullptr;
static int g_rows = 0;
static int g_cols = 0;


extern "C" __declspec(dllexport)
int initDataset(const double* data, int batch, int rows, int cols) {
    g_rows = rows;
    g_cols = cols;

    size_t bytes = sizeof(double) * rows * cols;

    cudaMalloc(&d_data, bytes);
    cudaMemcpy(d_data, data, bytes, cudaMemcpyHostToDevice);

    cudaMalloc(&d_sources, sizeof(double) * g_cols * batch);

    cudaMalloc(&d_out, sizeof(double) * g_rows * batch);
    

    return 1;
}

extern "C" __declspec(dllexport)
void freeDataset() {
    if (d_data) cudaFree(d_data);
    if (d_sources) cudaFree(d_sources);
    if (d_out) cudaFree(d_out);
    d_data = nullptr;
    d_sources = nullptr;
    d_out = nullptr;
}

__global__
void batchedDistancesKernel(const double* __restrict__ sources, const double* __restrict__ data,
                            double* out, int rows, int cols, int batch) {
    int row = blockIdx.x * blockDim.x + threadIdx.x;
    int q   = blockIdx.y;
                              
    if (row >= rows || q >= batch) return;

    int srcBase = q * cols;

    double sum = 0.0;
    for (int j = 0; j < cols; j++) {
        double d = data[j * rows + row];
        double s = sources[srcBase + j];
        double diff = s - d;
        sum += diff * diff;
    }

    out[q * rows + row] = sum;
}

// old unused function, replaced since profiling showed only 0.02% occupancy
__global__ void topKKernel(double* d_out, int* d_result,
                           int g_rows, int batch, int k)
{
    
    int q = blockIdx.x * blockDim.x + threadIdx.x;
    if (q >= batch) return;

    double* distRow = d_out + q * g_rows;

    for (int kk = 0; kk < k; kk++) {
        double best = DBL_MAX;
        int bestIdx = -1;

        for (int r = 0; r < g_rows; r++) {
            double v = distRow[r];
            if (v < best) {
                best = v;
                bestIdx = r;
            }
        }

        d_result[q * k + kk] = bestIdx;

        distRow[bestIdx] = DBL_MAX;
    }
}

// swapped to using parrallel reduction to better utilize occupancy. This now uses hundreds of blocks instead of the profiled 4, with each block using threads to scan the rows better utilizing the GPU resources
// however, since this is only a minor algorithm, this change only sees ~60-100ms runtime improvement total for 200 000 rows
/*
algorithm explaination:
    original behaviour: for every k, search all rows and find the minimum and marking it as seen. Then, write to d_result
    new behaviour: chunk rows using blockDims and the thread id and perform local search (for example, find min in 0-500 and 501-1000 in parrallel).
                    The algorithm then waits for all minimums to be found, before performing another linear search on all found minimums. The found minimum is marked seen, just like before

    "Occupancy is the ratio of the number of active warps per multiprocessor to the maximum number of possible active warps"
    how it improves occupancy: before, the linear scan was done all at once, so there were very little warps. Now, it schedules hundreds of warps in the chunking, and so improving the ratio

    change in parrallism: old was process level (one thread per block, multiple blocks per query), new is thread level (multiple threads per block, on block per query)
    so, if batch is small, (ie 4), there was a low amount of processes, and so very little parrallelism (hence the low occupancy).
    With the new verion, even if the batch is small, the chunking ensures there is sufficient parrallism.
*/
__global__ void topKKernelParallel(double* d_out, int* d_result,
                                   int g_rows, int batch, int k) {
    int q = blockIdx.x;  // one block per query
    if (q >= batch) return;

    double* distRow = d_out + q * g_rows;

    extern __shared__ double sharedVals[];
    int* sharedIdx = (int*)&sharedVals[blockDim.x];

    for (int kk = 0; kk < k; kk++) {
        double localBest = DBL_MAX;
        int localIdx  = -1;

        // each thread scans its chunk
        for (int r = threadIdx.x; r < g_rows; r += blockDim.x) {
            double v = distRow[r];
            if(v == DBL_MAX){
                continue;
            }
            if (v < localBest) {
                localBest = v;
                localIdx  = r;
            }
        }

        sharedVals[threadIdx.x] = localBest;
        sharedIdx[threadIdx.x]  = localIdx;
        __syncthreads();

        // reduction
        for (int stride = blockDim.x / 2; stride > 0; stride >>= 1) {
            if (threadIdx.x < stride) {
                double candVal = sharedVals[threadIdx.x + stride];
                int    candIdx = sharedIdx[threadIdx.x + stride];

                // only consider candidates with valid index
                if (candIdx >= 0 && candVal < sharedVals[threadIdx.x]) {
                    sharedVals[threadIdx.x] = candVal;
                    sharedIdx[threadIdx.x]  = candIdx;
                }
            }
            __syncthreads();
        }

        if (threadIdx.x == 0) {
            int bestIdx = sharedIdx[0];

            if (bestIdx >= 0) {
                d_result[q * k + kk] = bestIdx;
                distRow[bestIdx] = DBL_MAX;
            } else {
                // no valid index left; fill with sentinel
                d_result[q * k + kk] = -1;
            }
        }
        __syncthreads();
    }
}


extern "C" __declspec(dllexport)
int* computeTopKBatch(const double* sources, int batch, int k, int batch_size) {
    if (!d_data) return nullptr;

    const int batch_size = 512;

    cudaStream_t streams[NUM_STREAMS];
    for (int i = 0; i < NUM_STREAMS; i++) {
        cudaStreamCreate(&streams[i]);
    }

    int* d_result;
    cudaMalloc(&d_result, sizeof(int) * batch * k);
    int* result = (int*)malloc(sizeof(int) * batch * k);

    int chunk_size = (batch + NUM_STREAMS - 1) / NUM_STREAMS;

    for (int i = 0; i < NUM_STREAMS; i++) {
        int offset = i * chunk_size;
        int this_chunk = min(chunk_size, batch - offset);

        cudaMemcpyAsync(
            d_sources + offset * g_cols,
            sources   + offset * g_cols,
            sizeof(double) * g_cols * this_chunk,
            cudaMemcpyHostToDevice,
            streams[i]
        );

        dim3 block(batch_size);
        dim3 grid((g_rows + block.x - 1) / block.x, this_chunk);

        batchedDistancesKernel<<<grid, block, 0, streams[i]>>>(
            d_sources + offset * g_cols,
            d_data,
            d_out + offset * g_rows,
            g_rows, g_cols, this_chunk
        );
        int threads = 256;
        int blocks  = this_chunk;
        size_t shmem = sizeof(double) * threads + sizeof(int) * threads;

        topKKernelParallel<<<blocks, threads, shmem, streams[i]>>>(
            d_out + offset * g_rows,
            d_result + offset * k,
            g_rows, this_chunk, k
        );

        cudaMemcpyAsync(
            result + offset * k,
            d_result + offset * k,
            sizeof(int) * this_chunk * k,
            cudaMemcpyDeviceToHost,
            streams[i]
        );
    }


    for (int i = 0; i < NUM_STREAMS; i++) {
        cudaStreamSynchronize(streams[i]);
    }

    return result;
}

extern "C" __declspec(dllexport)
int getDeviceCount() {
    int count = 0;
    cudaGetDeviceCount(&count);
    return count;
}
```