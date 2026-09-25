---
layout: default
title: Portfolio |  Siamese Machine Learning Model
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Siamese Model

This model was created for a unity project, where it compares the features of the real map, and a user drawn map, to grade the user's map from 0 to 100. This is designed as the main scoring system for a game where the user explores a dungeon and maps it out, where the similarity score determines how well they drew the map. Machine learning was used to reduce the penalty for small mistakes with large differences, like making one hallway a cell to long, shifting the entire map. The model is output to an onnx file to be used by the Unity game.

```python
import json
import numpy as np
import onnx
import torch
import torch.onnx
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import Dataset, DataLoader
import os

from sklearn.model_selection import train_test_split
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
DATA_DIR = os.path.join(BASE_DIR, "TrainingData")
JSON_PATH = os.path.join(BASE_DIR, "data.jsonl")

CELL_WIDTH = 20
CELL_HEIGHT = 20
CHANNELS = 3
EPOCHS = 20


def resolve_path(unity_path):
    filename = os.path.basename(unity_path)
    return os.path.join(DATA_DIR, filename)

def load_tensor(unity_path):
    full = resolve_path(unity_path)
    data = np.fromfile(full, dtype=np.float32)
    data = data.reshape(CELL_WIDTH, CELL_HEIGHT, CHANNELS)
    data = np.transpose(data, (2, 1, 0))  # (C, H, W)
    return torch.tensor(data, dtype=torch.float32)

class MapDataset(Dataset):
    def __init__(self, samples_or_path):
        # If it's a string, treat it as a file path
        if isinstance(samples_or_path, str):
            self.samples = []
            with open(samples_or_path, "r") as f:
                for line in f:
                    self.samples.append(json.loads(line))
        else:
            # Otherwise assume it's already a list of dicts
            self.samples = samples_or_path

    def __len__(self):
        return len(self.samples)

    def __getitem__(self, idx):
        e = self.samples[idx]
        truth = load_tensor(e["truth"])
        inp = load_tensor(e["input"])
        score = torch.tensor([e["score"]], dtype=torch.float32)
        return truth, inp, score


all_samples = []
with open(JSON_PATH, "r") as f:
    for line in f:
        all_samples.append(json.loads(line))

train_samples, val_samples = train_test_split(all_samples, test_size=0.2, shuffle=True)

train_dataset = MapDataset(train_samples)
val_dataset = MapDataset(val_samples)

train_loader = DataLoader(train_dataset, batch_size=16, shuffle=True)
val_loader = DataLoader(val_dataset, batch_size=16, shuffle=False)


class SimilarityModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.encoder = nn.Sequential(
            nn.Conv2d(3, 16, 3, padding=1), nn.ReLU(),
            nn.Conv2d(16, 32, 3, padding=1), nn.ReLU(),
            nn.MaxPool2d(2),

            nn.Conv2d(32, 64, 3, padding=1), nn.ReLU(),
            nn.MaxPool2d(2),

            nn.Conv2d(64, 128, 3, padding=1), nn.ReLU(),
            nn.AdaptiveAvgPool2d((1,1))
        )

        self.fc = nn.Sequential(
            nn.Linear(128 * 2, 64),
            nn.ReLU(),
            nn.Linear(64, 1),
            nn.Sigmoid()
        )

    def forward(self, a, b):
        ea = self.encoder(a).view(a.size(0), -1)
        eb = self.encoder(b).view(b.size(0), -1)
        x = torch.cat([ea, eb], dim=1)
        return self.fc(x)

model = SimilarityModel()
opt = optim.Adam(model.parameters(), lr=1e-3)
loss_fn = nn.MSELoss()

for epoch in range(EPOCHS):
    model.train()
    train_loss = 0
    for truth, inp, score in train_loader:
        pred = model(truth, inp)
        loss = loss_fn(pred, score)
        opt.zero_grad()
        loss.backward()
        opt.step()
        train_loss += loss.item()

    model.eval()
    val_loss = 0
    with torch.no_grad():
        for truth, inp, score in val_loader:
            pred = model(truth, inp)
            loss = loss_fn(pred, score)
            val_loss += loss.item()

    print(f"Epoch {epoch} Train {train_loss/len(train_loader):.4f}  Val {val_loss/len(val_loader):.4f}")


torch.save(model.state_dict(), os.path.join(BASE_DIR, "similarity_model.pth"))

dummy_truth = torch.randn(1, 3, CELL_HEIGHT, CELL_WIDTH)
dummy_input = torch.randn(1, 3, CELL_HEIGHT, CELL_WIDTH)

model.eval()



onnx_path = os.path.join(BASE_DIR, "gombert.onnx")

torch.onnx.export(
    model,
    (dummy_truth, dummy_input),
    onnx_path,
    input_names=["truth", "input"],
    output_names=["score"],
    opset_version=13,
    export_params=True,
    do_constant_folding=True
)


print("Exported ONNX model to:", onnx_path)

onnx_model = onnx.load(onnx_path)
onnx.checker.check_model(onnx_model)
print("Model is Valid")
```