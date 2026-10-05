---
layout: default
title: Portfolio |  Matrix Operations
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Matrix
The code contains primitive matrix operations, such as sum, add, scale, multiply, and transpose.

```cpp
#include <stdint.h>

void matrix_sum(int16_t *a, uint8_t *dimension, int16_t *b, int16_t *result){
    uint8_t cols = dimension[1];
    for(int row = 0; row < dimension[0]; row++){
        for(int column = 0; column < cols; column++){
            result[row*cols + column] = a[row*cols + column] + b[row*cols+column];
        }
    }
}

void matrix_add(int16_t *a, uint8_t *dimension, int16_t scalar, int16_t *result){
    uint8_t cols = dimension[1];
    for(int row = 0; row < dimension[0]; row++){
        for(int column = 0; column < dimension[1]; column++){
            result[row*cols + column] = a[row*cols + column] + scalar;
        }
    }
}

void matrix_scale(int16_t *a, uint8_t *dimension, int16_t scalar, int16_t *result){
    uint8_t cols = dimension[1];
    for(int row = 0; row < dimension[0]; row++){
        for(int column = 0; column < dimension[1]; column++){
            result[row*cols + column] = a[row*cols + column] * scalar;
        }
    }
}

void matrix_transpose(int16_t *a, uint8_t *dimension, int16_t *result){
    uint8_t cols = dimension[1];
    uint8_t rows = dimension[0];
    if(cols != rows){return;}

    for(int row = 0; row < dimension[0]; row++){
        for(int column = 0; column < dimension[1]; column++){
            result[row*cols + column] = a[row*cols + column];
            result[column*cols + row] = a[column*cols + row];
        }
    }
}


void matrix_mul(int16_t *a, uint8_t *dimension_a, int16_t *b, uint8_t *dimension_b, int16_t *result){
    uint8_t m = dimension_a[0];
    uint8_t p = dimension_a[1];
    uint8_t n = dimension_b[1];

    for(uint8_t row = 0; row < m; row ++){ //repeat over each row in A

        for(uint8_t col = 0; col < n; col++){ //loop over each value in the current row
            result[row * n + col] = 0;
            for(uint8_t k = 0; k < p; k++){ //loop over the columns
                uint16_t a_index = row * p + k;
                uint16_t b_index = k * n + col;
                result[row * n + col] += a[a_index] * b[b_index];
            }
        }
    }
}
```