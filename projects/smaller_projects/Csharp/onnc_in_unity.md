---
layout: default
title: Portfolio |  Onnx with Unity
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Onnx integration with Unity games
This code is the Unity application stage using the output onnx file from the python siamese model. The code has some post processing of the output, like immediately checking for a perfect match, or raising or lowering the scores to make it more equitable.

```csharp
using System.Collections;
using System.Collections.Generic;
using System.IO;
using System;
using UnityEngine;
using Unity.Barracuda;

public class MapOutputEncoder : MonoBehaviour{

    [SerializeField] private List<Sprite> hallways;
    [SerializeField] private List<Sprite> rooms;
    [SerializeField] private List<Sprite> allMeaningOverlays;
    [SerializeField] private List<Sprite> allNonMeaningOverlays;

    readonly HashSet<Sprite> meaningOverlays = new();
    readonly HashSet<Sprite> nonMeaningOverlays = new();

    readonly HashSet<Sprite> hallwaySet = new();
    readonly HashSet<Sprite> roomsSet = new();

    [SerializeField] private string filePath = "Assets/MachineLearning/TrainingData/";
    [SerializeField] private string jsonPath = "Assets/MachineLearning/data.jsonl";
    [SerializeField] private int mapCode = 0;

    [SerializeField] private MapDrawer trueMap;
    [SerializeField] private MapDrawer inputMap;

    public static int channelCount = 3;

    /** channels:
    0: main sprite
    1: overlay sprite
    2: main sprite area
    **/

    [Serializable]
    public class JsonEntry {
        public string truth;
        public string input;
        public float score;
    }

    public NNModel onnxModel;
    private IWorker worker;

    void OnApplicationQuit(){
        //EmitTensor();
        Debug.Log(EvaluateModel());
    }

    void OnDestroy(){
        worker?.Dispose();
    }

    // function to map all sprites to integers, allowing for it to be emit in a usable tensor
    void RegisterAssets(){
        foreach(Sprite s in hallways)
            hallwaySet.Add(s);
        foreach(Sprite s in rooms)
            roomsSet.Add(s);
        foreach(Sprite s in allMeaningOverlays)
            meaningOverlays.Add(s);
        foreach(Sprite s in allNonMeaningOverlays)
            nonMeaningOverlays.Add(s);
    }

    void Start(){
        RegisterAssets();
        var model = ModelLoader.Load(onnxModel);
        worker = WorkerFactory.CreateWorker(WorkerFactory.Type.Auto, model);
    }

    private static float[,,,] ToNHWC(float[,,] src){
        int H = src.GetLength(0);
        int W = src.GetLength(1);
        int C = src.GetLength(2);

        float[,,,] dst = new float[1, H, W, C];

        for (int h = 0; h < H; h++)
            for (int w = 0; w < W; w++)
                for (int c = 0; c < C; c++)
                    dst[0, h, w, c] = src[h, w, c];

        return dst;
    }

    // create a tensor to be input in the ML model
    private static Tensor CreateTensor(float[,,,] arr){
        int N = arr.GetLength(0);
        int H = arr.GetLength(1);
        int W = arr.GetLength(2);
        int C = arr.GetLength(3);

        float[] flat = new float[N * H * W * C];
        Buffer.BlockCopy(arr, 0, flat, 0, flat.Length * sizeof(float));

        var shape = new TensorShape(N, H, W, C);
        return new Tensor(shape, flat);
    }

    /// <summary>
    /// get the output of the model, by inputing the true and user drawn map
    /// </summary>
    /// <returns>float score between 0 and 1</returns>
    public float EvaluateModel(){
        float[,,] groundTruth = GetMapTensor(trueMap);
        float[,,] input = GetMapTensor(inputMap);

        // basic test to ensure a perfect map gives 1, in case gombert is stupid (it does not like handing out perfect marks)
        int cellWidth = trueMap.cellWidth;
        int cellHeight = trueMap.cellHeight;
        bool notPerfect = false;
        for(int x = 0; x < cellWidth; x++){
            for(int y = 0; y < cellHeight; y++){
                if(groundTruth[x,y,0] != -1){
                    for(int i = 0; i < channelCount; i++)
                        if(input[x,y,i] != groundTruth[x,y,i])
                            notPerfect = true;
                }
            }
        }  
        if(!notPerfect){return 1;}

        float[,,,] truthNHWC = ToNHWC(groundTruth);
        float[,,,] inputNHWC = ToNHWC(input);

        Tensor truthTensor = CreateTensor(truthNHWC);
        Tensor inputTensor = CreateTensor(inputNHWC);

        var inputs = new Dictionary<string, Tensor>{
            { "truth", truthTensor },
            { "input", inputTensor }
        };

        worker.Execute(inputs);

        Tensor output = worker.PeekOutput("score");
        float score = output[0];

        truthTensor.Dispose();
        inputTensor.Dispose();
        output.Dispose();

        float biasShift = 0.08f; // improve the score when it is already really high, since the model never seams to give in the 90s, even if it is 99% correct
        float lowBiasShift = 0.04f; // downgrade score when it does poor, since the model overrates weaker scores;
        float maximumNonPerfectScore = 0.98f; // if it is not perfect from the earlier check, make it impossible to return 1

        if(score > 0.8){score += biasShift;}
        if(score < 0.3f){score -= lowBiasShift;}
        score = Mathf.Clamp(score, 0, maximumNonPerfectScore);
        return score;
    }

    // function to obtain and save training data from the truth map
    private void EmitTrainingData(){
        int cellWidth = trueMap.cellWidth;
        int cellHeight = trueMap.cellHeight;
        float[,,] groundTruth = GetMapTensor(trueMap);
        string groundTruthPath = filePath+"true"+mapCode;

        TrainingAugments a = new();
        OutputToFile(groundTruth, groundTruthPath, cellWidth, cellHeight);

        AddToJson(groundTruthPath, a.AllPerfect(groundTruth, cellWidth, cellHeight), 1, filePath+"perfect"+mapCode, cellWidth, cellHeight); // add the ground truth and write it to a file

        // get the augmented training data tensors and the score
        var (extendedHallwayLeft,  extendedHallwayLeftScore)  = a.ExtendHallwayHorizontal(groundTruth, cellWidth, cellHeight, 0);
        var (extendedHallwayRight, extendedHallwayRightScore) = a.ExtendHallwayHorizontal(groundTruth, cellWidth, cellHeight, 1);
        var (extendedHallwayUp,  extendedHallwayUpScore)      = a.ExtendHallwayVertical(groundTruth, cellWidth, cellHeight, 1);
        var (extendedHallwayDown, extendedHallwayDownScore)   = a.ExtendHallwayVertical(groundTruth, cellWidth, cellHeight, 0);
        var (colorSwap, colorSwapScore) = a.SwapColor(groundTruth, cellWidth, cellHeight);
        var (overlayChange, overlayChangeScore) = a.AdjustOverlay(groundTruth, cellWidth, cellHeight);
        var (removedChunk, removedChunkScore) = a.RemoveChunk(groundTruth, cellWidth, cellHeight);
        var (removedLarge, removedLargeScore) = a.RemoveLargeRegion(groundTruth, cellWidth, cellHeight);
        var (roomShift, roomShiftScore) = a.LargeRoomWrongPosition(groundTruth, cellWidth, cellHeight);
        var (brokenConn, brokenConnScore) = a.BreakConnectivity(groundTruth, cellWidth, cellHeight);
        var (distortedRoom, distortedRoomScore) = a.DistortRoomShape(groundTruth, cellWidth, cellHeight);
        var (misplacedOverlay, misplacedOverlayScore) = a.MisplaceOverlays(groundTruth, cellWidth, cellHeight);
        var (combined, combinedScore) = a.CombinedErrors(groundTruth, cellWidth, cellHeight);
        var (hallwaysGoodRoomsBad, hallwaysGoodRoomsBadScore) = a.PerfectHallwaysWrongRooms(groundTruth, cellWidth, cellHeight);

        // add the truth, input, output mapping to a json file, while saving the tensor itself to binary
        if(extendedHallwayLeftScore != -1) {AddToJson(groundTruthPath, extendedHallwayLeft, extendedHallwayLeftScore,  filePath+"leftextended"+mapCode, cellWidth, cellHeight);}
        if(extendedHallwayRightScore != -1){AddToJson(groundTruthPath, extendedHallwayRight,extendedHallwayRightScore, filePath+"rightextended"+mapCode, cellWidth, cellHeight);}
        if(extendedHallwayUpScore != -1)   {AddToJson(groundTruthPath, extendedHallwayUp,   extendedHallwayUpScore,    filePath+"upextended"+mapCode, cellWidth, cellHeight);}
        if(extendedHallwayDownScore != -1) {AddToJson(groundTruthPath, extendedHallwayDown, extendedHallwayDownScore,  filePath+"downextended"+mapCode, cellWidth, cellHeight);}
        if(colorSwapScore != -1){AddToJson(groundTruthPath, colorSwap, colorSwapScore, filePath+"colorswap"+mapCode, cellWidth, cellHeight);}
        if(overlayChangeScore != -1){AddToJson(groundTruthPath, overlayChange, overlayChangeScore, filePath+"overlaychange"+mapCode, cellWidth, cellHeight);}
        if(removedChunkScore != -1){AddToJson(groundTruthPath, removedChunk, removedChunkScore, filePath+"chunk"+mapCode, cellWidth, cellHeight);}
        if(removedLargeScore != -1){AddToJson(groundTruthPath, removedLarge, removedLargeScore, filePath+"largedestruction"+mapCode, cellWidth, cellHeight);}
        if(roomShiftScore != -1){AddToJson(groundTruthPath, roomShift, roomShiftScore, filePath+"roomshift"+mapCode, cellWidth, cellHeight);}
        if(brokenConnScore != -1){AddToJson(groundTruthPath, brokenConn, brokenConnScore, filePath+"brokenconn"+mapCode, cellWidth, cellHeight);}
        if(distortedRoomScore != -1){AddToJson(groundTruthPath, distortedRoom, distortedRoomScore, filePath+"distortedroom"+mapCode, cellWidth, cellHeight);}
        if(misplacedOverlayScore != -1){AddToJson(groundTruthPath, misplacedOverlay, misplacedOverlayScore, filePath+"misplacedoverlay"+mapCode, cellWidth, cellHeight);}
        if(combinedScore != -1){AddToJson(groundTruthPath, combined, combinedScore, filePath+"combined"+mapCode, cellWidth, cellHeight);}
        if(hallwaysGoodRoomsBadScore != -1){AddToJson(groundTruthPath, hallwaysGoodRoomsBad, hallwaysGoodRoomsBadScore, filePath+"hallwaysgoodroomsbad"+mapCode, cellWidth, cellHeight);}
        AddToJson(groundTruthPath, a.AllGone(cellWidth, cellHeight), 0, filePath+"nothing"+mapCode, cellWidth, cellHeight);
    }

  
    // output a tensor to a binary file
    private void OutputToFile(float[,,] tensor, string filePath, int cellWidth, int cellHeight){
        using (BinaryWriter writer = new(File.Open(filePath, FileMode.Create))) {
            for (int x = 0; x < cellWidth; x++)
                for (int y = 0; y < cellHeight; y++)
                    for (int c = 0; c < channelCount; c++)
                        writer.Write(tensor[x, y, c]);
        }
        Debug.Log("ouput to " + filePath);
    }

    // create a line in json for the supervised training data
    private void AddToJson(string groundTruthPath, float[,,] input, float score, string filePath, int cellWidth, int cellHeight){
        OutputToFile(input, filePath, cellWidth, cellHeight);

        JsonEntry entry = new(){
            truth = groundTruthPath,
            input = filePath,
            score = score
        };
        string jsonLine = JsonUtility.ToJson(entry);

        using StreamWriter writer = new(jsonPath, append: true);
        writer.WriteLine(jsonLine);
    }

    // function converts a the grid representing the map into a tensor
    private float[,,] GetMapTensor(MapDrawer map){
        Grid<MapGridCell> m = map.GetMap();
        int cellWidth = map.cellWidth;
        int cellHeight = map.cellHeight;

        float[,,] tensor = InitializeTensor(cellWidth, cellHeight);

        for(int x = 0; x < cellWidth; x++){
            for(int y = 0; y < cellHeight; y++){
                MapGridCell cell = m.GetValue(x, y);

                int spriteVal = 0;
                if(hallwaySet.Contains(cell.sprite)){spriteVal = 0;}
                else if(roomsSet.Contains(cell.sprite)){spriteVal = 1;}
                else{spriteVal = -1;}

                tensor[x, y, 0] = spriteVal;
                int overlayVal = 0;
                if(meaningOverlays.Contains(cell.overlay)){ overlayVal = 1;}
                else if(nonMeaningOverlays.Contains(cell.overlay)){ overlayVal = 0;}
                else{ overlayVal = -1;}
                tensor[x, y, 1] = overlayVal;
                tensor[x, y, 2] = AreaColor(cell.color);
            }
        }

        return tensor;
    }

    // initialize a default tensor with the proper shapes
    public static float[,,] InitializeTensor(int cellWidth, int cellHeight){
        return new float[cellWidth,cellHeight,channelCount];
    }

    // maps the area colour to its index, to reduce channels from 4 (rgba) to 1
    private static int AreaColor(Color col){
        if (col == Color.green)   return (int)AreaType.Green;
        if (col == Color.yellow)  return (int)AreaType.Yellow;
        if (col == Color.magenta) return (int)AreaType.Purple;
        if (col == Color.red)     return (int)AreaType.Red;
        if (col == Color.cyan)    return (int)AreaType.Blue;

        return (int)AreaType.Default;
    }

}

```