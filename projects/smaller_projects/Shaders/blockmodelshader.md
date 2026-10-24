---
layout: default
title: Portfolio |  Blockmodel GPU Instancing
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Blockmodel Shader with GPU Instancing
Block models contain hundreds of thousands of cubes. Due to Unity's draw call system, creating this many independent cubes would tank performance. So, it can either combine each cube into one mesh, or GPU instance the cubes. GPU instancing is perferred, since it gives each cube access to the InstanceID variable. The combined mesh method requires embedding the data into the mesh itself, using either the UVs or vertex colours, which does noy allow any of the values to be changed after creation.

This shader colours each block using its InstanceID to index a structured buffer, input into the material instance. The shader itself is created using ShaderGraph, to automatically include shadows and lighting. The block grade must be obtained using a custom HLSL function.

<img src="/assets/files/shaders/images/blockmodel.png" alt="Blockmodel render">

<img src="/assets/files/shaders/images/blockmode_shadergraph.png" alt="Blockmodel shadergraph">

The shader itself is fairly simple. It gets the grade from a custom function using the exposed InstanceID variable. It then clamps the value between 0 and 1, and rounds down to the nearest colour band. This rounded value is then used to lerp between two colour, giving the solid, but different, colour for every block.

The custom function used is:
```hlsl
#ifndef BLOCK_DATA_INCLUDED
#define BLOCK_DATA_INCLUDED

struct BlockData{
    float grade;
    uint domain;
    float density;
    float selected;
};

StructuredBuffer<BlockData> _Blocks;
uint _BlockCount;

void GetBlockGrade_float(float InstanceID, out float Grade){
    if(InstanceID >= _BlockCount){
        Grade = 0.0;
        return;
    }
    Grade = _Blocks[InstanceID].grade;
}

#endif
```

For the block model to actually render the different colours, the StructuredBuffer must be insert into the material instance. This is done in runtime, by creating a ComputeBuffer. The buffer must be freed on destroy, otherwise a memory leak will occur.

```csharp
using System.Collections;
using System.Collections.Generic;
using System.Runtime.InteropServices;
using UnityEngine;
using System;

public class GradeBuffer
{
    
    [HideInInspector] public ComputeBuffer buffer;

    public void CreateBuffer(BlockData[] data){
        buffer = new ComputeBuffer(data.Length,Marshal.SizeOf<BlockData>());
        buffer.SetData(data);
    }

    public void FreeBuffer(){
        buffer?.Release();
        buffer = null;
    }
}

[StructLayout(LayoutKind.Sequential)]
public struct BlockData
{
    public float Grade;
    public uint Domain;
    public float Density;
    public float Selected;
}

public class CSVLoader
{
    public static Dictionary<string, List<string>> Load(string path){
        Dictionary<string, List<string>> columns = new();

        string[] lines = File.ReadAllLines(path);

        if(lines.Length == 0){
            return columns;
        }

        string[] headers = lines[0].Split(',');

        foreach(string header in headers){
            columns[header] = new List<string>();
        }

        for(int row = 1; row < lines.Length; row++){
            string[] values = lines[row].Split(',');
            for(int col = 0; col < headers.Length; col++){
                columns[headers[col]].Add(col < values.Length ? values[col] : string.Empty);
            }
        }

        return columns;
    }
}
```

The GPU instancing in Unity is done using Graphics.RenderMeshInstanced, instantiating a mesh instance for every 4x4 matrix encoding the position, scale and rotation of the mesh. Since it is GPU instanced, other components or collision cannot be added to it.

```csharp
using System.Collections;
using System.Collections.Generic;
using System.Runtime.InteropServices;
using System.IO;
using UnityEngine;

public class BlockModelGenerator : MonoBehaviour
{
    public Material mat;
    public Mesh cubeMesh;

    private string xField = "XC";
    private string yField = "YC";
    private string zField = "ZC";
    private string gradeField = "CU";

    private float scale = 1;

    List<GradeBuffer> buffs;

    private Matrix4x4[] matrices;

    void Awake(){
        CreateModel();
    }

    void CreateModel(){

        string path = Path.Combine(Application.streamingAssetsPath,"test_model.csv");

        var data = CSVLoader.Load(path);

        buffs = new();
        GradeBuffer buffer = new();
        int rows = data[gradeField].Count;
        BlockData[] blocks = new BlockData[rows];

        // find min and max values. The coordinates are used to centre the block around 0
        // otherwise, depending on the data, the coordinates could be in the 10 000s.
        float minG = int.MaxValue;
        float maxG = int.MinValue;
        float minX = float.MaxValue;
        float maxX = float.MinValue;
        float minY = float.MaxValue;
        float maxY = float.MinValue;
        float minZ = float.MaxValue;
        float maxZ = float.MinValue;

        for(int i = 0; i < rows; i++){
            float x = float.Parse(data[xField][i]);
            float y = float.Parse(data[yField][i]);
            float z = float.Parse(data[zField][i]);
            float g = float.Parse(data[gradeField][i]);

            minX = Mathf.Min(minX, x);
            maxX = Mathf.Max(maxX, x);
            minY = Mathf.Min(minY, y);
            maxY = Mathf.Max(maxY, y);
            minZ = Mathf.Min(minZ, z);
            maxZ = Mathf.Max(maxZ, z);
            minG = Mathf.Min(minG, g);
            maxG = Mathf.Max(maxG, g);
        }

        // normalize grade, so that the shader can more accuratly lerp between the min and maximum grade
        for(int i = 0; i < rows; i++){
            blocks[i].Grade = (float.Parse(data[gradeField][i])) - minG / (maxG - minG);
        }

        buffs.Add(buffer);
        buffer.CreateBuffer(blocks);

        mat.SetInt("_BlockCount", blocks.Length);
        mat.SetBuffer("_Blocks", buffer.buffer);

        Vector3 centre = new(
            (minX + maxX) * 0.5f,
            (minY + maxY) * 0.5f,
            (minZ + maxZ) * 0.5f
        );
        Quaternion modelRotation = Quaternion.Euler(90, 0, 0);

        Matrix4x4[] matrices = new Matrix4x4[rows];
        for(int i = 0; i < rows; i++){
            Vector3 pos = new(
                float.Parse(data[xField][i]),
                float.Parse(data[yField][i]),
                float.Parse(data[zField][i])
            );
            pos -= centre;
            pos = modelRotation * pos;
            matrices[i] = Matrix4x4.TRS(pos, modelRotation, new Vector3(100,100,30) * scale);
        }
        this.matrices = matrices;
    }

    void Update(){
        Graphics.RenderMeshInstanced(new RenderParams(mat), cubeMesh, 0, matrices);
    }
    

    void OnDestroy(){
        foreach(GradeBuffer buffer in buffs){
            buffer.FreeBuffer();
        }
    }
}
```