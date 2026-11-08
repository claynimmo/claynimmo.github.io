---
layout: default
title: Data Driven Programming in Unity
tags: [Unity Engine, Programming, Data]
---

[Home](../../../../index.md) / [Tutorials](../index.md) /

## Overview
Data driven programming involves saving, storing, and loading data using I/O files. This can be done using JSON, XML, YAML or CSV files. The most common applications of data driven programming are:
- Save data
  - Storing the player’s state in a file is the only way to have persistent data between different play sessions.
- Dialogue options
  - Storing dialogue in files increases maintainability, and makes it much easier to add other language support.
- Reusable objects
  - Storing links to textures, models, equipment stats, materials, colours etc can make it far easier to add more items in the future, and decouples the data from the scene.
  - Minecraft is the biggest example of this, where all new blocks are data driven, allowing them to integrate seamlessly with existing blocks, and to properly render in the procedural generation.
- Default settings
  - Instead of sitting idle in runtime memory, default settings can be stored in disk, improving runtime performance, and decoupling the settings from specific objects.

## Setting up the Data
In C#, JSON objects are loaded directly into classes. An example of a JSON file and its corresponding C# class is given, to assist in constructing your own custom file:

```json
{
  "stringValue": "Hello World",
  "integerValue": 42,
  "floatValue": 3.14159,
  "booleanValue": true,
  "nullValue": null,

  "nestedObject": {
    "firstName": "Jane",
    "lastName": "Doe"
  },

  "listValue": [
    "Red",
    "Green",
    "Blue"
  ]
}
```

```csharp
public class SampleData
{
    public string StringValue { get; set; }
    public int IntegerValue { get; set; }
    public double FloatValue { get; set; }
    public bool BooleanValue { get; set; }
    public string? NullValue { get; set; }
    public NestedObject NestedObject { get; set; } = new();
    public List<string> ListValue { get; set; } = new();
}

public class NestedObject
{
    public string FirstName { get; set; } = "";
    public string LastName { get; set; } = "";
}
```

To create a save file for user settings, it minimally requires camera sensitivity and music and effects volume. Music and sound effects are best separated, so the user can balance the audio to improve their user experience.

```csharp
[System.Serializable]
public class SettingsValues{
    public float cameraSensitivity;
    public float musicVolume;
    public float otherVolume;
}
```

Note the System.Serializable tag above the class. This is required for Unity’s `JsonUtility`, and for the data to appear in the editor. `JsonUtility` also prefers public variables over properties, hence the removal of `{get; set;}`.

## Saving a Class Instance to File

With the settings class object, the user’s runtime data must be saved to file to use across sessions. The following class loads settings from a JSON file into a runtime object:

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.EventSystems;
using System.IO;

public class Settings
{
    private const string presetFileName = "SaveData/settings_preset.json";

    [SerializeField] private Slider sensitivity;
    [SerializeField] private Slider cameraDistance;

    [SerializeField] private Slider musicVolume;
    [SerializeField] private Slider otherVolume;

    private SettingsValues settings;

    public void SaveSettingsToFile(){
        string filePath = Path.Combine(Application.persistentDataPath, presetFileName);
        string json = JsonUtility.ToJson(settings,true);
        string directory = Path.GetDirectoryName(filePath);
        if(!Directory.Exists(filePath)){
            Directory.CreateDirectory(directory);
        }
        File.WriteAllText(filePath,json);
    }

    public SettingsData GetSettings(){
        return settings;
    }

}
```

The main principle is to obtain the desired output file path relative to the process, using `Application.persistantDataPath`, creating the directory if it does not exist, and writing the new JSON to the file. Converting the object to JSON is done using `JsonUtility.ToJson`, inputting the target class. The Boolean input determines if the output JSON is formatted. To prevent stalling for large files, Unity allows ToJson to run in background threads, so long as the object is not written to. An example asynchronous function is:

```csharp
public async Task SaveSettingsToFile(){
    SettingsData settingsCopy = settings;

    string filePath = Path.Combine(Application.persistentDataPath, presetFileName);

    await Task.Run(() => {
        string json = JsonUtility.ToJson(settingsCopy, true);
        string? directory = Path.GetDirectoryName(filePath);
        if(!string.IsNullOrEmpty(directory) && !Directory.Exists(directory)){
            Directory.CreateDirectory(directory);
        }
        File.WriteAllText(filePath, json);
    });
}
```

## Loading Data from a File

The most important step of data driven programming is loading the data on startup. Loading data is simply the inverse of saving, using read instead of write, and `JsonUtility.FromJson` instead of `ToJson`. The function to load the data written by the save function is:

```csharp
public void LoadSettingsFromFile(){
    string filePath = Path.Combine(Application.persistentDataPath, presetFileName);
    if(!File.Exists(filePath)){
        return;
    }
    string json = File.ReadAllText(filePath);

    SettingsValues newSettings = JsonUtility.FromJson<SettingsValues>(json);
    settings = newSettings;
    sensitivity.value = settings.cameraSensitivity;
    musicVolume.value = settings.musicVolume;
    otherVolume.value = settings.otherVolume;
}
```

For presets or default values, these can be stored in the reserved `Resources` folder. Reading streaming assets is slightly different to standard files, requiring loading the data as a `TextAsset` first, which must be unloaded after use. The function to access this data, assuming a file named `settings.json` is inside the `Resources` folder is:

```csharp
public void LoadDefaultSettingsFromFile(){
    TextAsset jsonFile = Resources.Load<TextAsset>("settings");
    string json = jsonFile.text;

    SettingsValues newSettings = JsonUtility.FromJson<SettingsValues>(json);
    sensitivity.value = settings.cameraSensitivity;
    musicVolume.value = settings.musicVolume;
    otherVolume.value = settings.otherVolume;
    
    settings = newSettings;

    Resources.UnloadAsset(jsonFile);
}
```

## Full Code

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.EventSystems;
using System.IO;

public class Settings
{
    private const string presetFileName = "SaveData/settings_preset.json";

    [SerializeField] private Slider sensitivity;
    [SerializeField] private Slider cameraDistance;

    [SerializeField] private Slider musicVolume;
    [SerializeField] private Slider otherVolume;

    private SettingsValues settings;

    void Awake(){
        LoadSettingsFromFile();
    }

    public void LoadDefaultSettingsFromFile(){
        TextAsset jsonFile = Resources.Load<TextAsset>("settings");
        string json = jsonFile.text;

        SettingsValues newSettings = JsonUtility.FromJson<SettingsValues>(json);
        sensitivity.value = settings.cameraSensitivity;
        musicVolume.value = settings.musicVolume;
        otherVolume.value = settings.otherVolume;
        
        settings = newSettings;

        Resources.UnloadAsset(jsonFile);
    }

    public void LoadSettingsFromFile(){
        string filePath = Path.Combine(Application.persistentDataPath, presetFileName);
        if(!File.Exists(filePath)){
            return;
        }
        string json = File.ReadAllText(filePath);

        SettingsValues newSettings = JsonUtility.FromJson<SettingsValues>(json);
        settings = newSettings;
        sensitivity.value = settings.cameraSensitivity;
        musicVolume.value = settings.musicVolume;
        otherVolume.value = settings.otherVolume;
    }

    public void UpdateSettings(){
        settings.cameraSensitivity = Mathf.Round(sensitivity.value*100)/100;
        settings.musicVolume = Mathf.Round(musicVolume.value*100)/100;
        settings.otherVolume = Mathf.Round(otherVolume.value*100)/100;
    }

    public void SaveSettingsToFile(){
        string filePath = Path.Combine(Application.persistentDataPath, presetFileName);
        string json = JsonUtility.ToJson(preset,true);
        string directory = Path.GetDirectoryName(filePath);
        if(!Directory.Exists(filePath)){
            Directory.CreateDirectory(directory);
        }
        File.WriteAllText(filePath,json);
    }

    public SettingsData GetSettings(){
        return settings;
    }
}
```

## Extras

For small amounts of data, Unity also exposes `PlayerPrefs`. `PlayerPrefs` is an unencrypted local registry that can store simple float, int, and string variables between game sessions. For the amount of settings given in this tutorial, PlayerPrefs are sufficient. However, when scaling for pages of settings, such as in Monster Hunter, it is better to maintain the JSON format. Do not store sensitive information in PlayerPrefs. See the official documentation for more [details](https://docs.unity.com/en-us/engine/6000.6/script-reference/unityengine/playerprefs).