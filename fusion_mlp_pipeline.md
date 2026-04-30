# Fusion MLP Training Pipeline

## What it is
A small neural network that takes radar + vision features as input and outputs
a single threat probability. Replaces the simple AND-rule with a learned model.

## Data Collection (field sessions, ~3 sessions of 2 hours each)

Fly drone at fixed height (5–10 m) over a flat area.
Log radar serial output + YOLO detections simultaneously.
Label each 1-second window as: threat=1 or threat=0

Positive samples (threat=1):
  - Person walking toward drone
  - Person running
  - Person crouching + moving
  - Two people approaching together

Negative samples (threat=0):
  - Empty field
  - Tree branches moving in wind (radar picks this up, camera does not)
  - Drone's own vibration artifacts
  - Animals (dog, bird)
  - Parked vehicles

Target: 300 positive + 300 negative = 600 labeled windows
Store as CSV: timestamp, 6 features, label

## Feature Extraction

For each 1-second window:

  radar_doppler_velocity:
    LD2410 outputs target speed in cm/s
    Take max speed observed in window

  radar_signal_strength:
    LD2410 outputs energy value (0–100)
    Take mean energy in window

  yolo_confidence:
    YOLOv8-pose outputs confidence per detection (0.0–1.0)
    Take max confidence in window; 0.0 if no detection

  yolo_bbox_area:
    Bounding box width * height in pixels, normalized by frame area
    0.0 if no detection

  track_duration_seconds:
    ByteTrack track age for highest-confidence track in window
    0.0 if no active track

  track_displacement_rate:
    Pixels/second of track centroid movement (ByteTrack output)
    Proxy for actual movement speed

## Model Architecture

```python
import torch
import torch.nn as nn

class FusionMLP(nn.Module):
    def __init__(self):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(6, 64),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(64, 32),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(32, 16),
            nn.ReLU(),
            nn.Linear(16, 1),
            nn.Sigmoid()
        )

    def forward(self, x):
        return self.net(x).squeeze()
```

~3,500 parameters. Inference: <1 ms on RPi 4. Exportable to ONNX.

## Training

```python
# train.py
import pandas as pd
import torch
from torch.utils.data import DataLoader, TensorDataset
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler

df = pd.read_csv("field_data.csv")
X = df[["radar_doppler_velocity", "radar_signal_strength",
        "yolo_confidence", "yolo_bbox_area",
        "track_duration_seconds", "track_displacement_rate"]].values
y = df["label"].values.astype("float32")

scaler = StandardScaler()
X = scaler.fit_transform(X)  # save scaler for inference

X_train, X_val, y_train, y_val = train_test_split(X, y, test_size=0.2, stratify=y)

train_ds = TensorDataset(torch.tensor(X_train, dtype=torch.float32),
                          torch.tensor(y_train))
val_ds   = TensorDataset(torch.tensor(X_val, dtype=torch.float32),
                          torch.tensor(y_val))

model = FusionMLP()
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
criterion = nn.BCELoss()

for epoch in range(100):
    model.train()
    for xb, yb in DataLoader(train_ds, batch_size=32, shuffle=True):
        pred = model(xb)
        loss = criterion(pred, yb)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

# Export to ONNX for RPi 4 inference
dummy = torch.zeros(1, 6)
torch.onnx.export(model, dummy, "fusion_mlp.onnx")
```

## Inference on RPi 4 (live)

```python
import onnxruntime as ort
import numpy as np

session = ort.InferenceSession("fusion_mlp.onnx")
# scaler loaded from pickle

def get_threat_score(radar_vel, radar_strength, yolo_conf,
                     bbox_area, track_dur, displacement_rate):
    features = np.array([[radar_vel, radar_strength, yolo_conf,
                          bbox_area, track_dur, displacement_rate]],
                        dtype=np.float32)
    features = scaler.transform(features)
    score = session.run(None, {"input": features})[0][0]
    return float(score)   # 0.0 → 1.0

# In main loop:
score = get_threat_score(...)
if score > 0.75:
    emit_alarm(score)     # WebSocket → war room + soldier apps
```

## Evaluation (the paper result)

Compare on held-out test set:

| Model          | Precision | Recall | F1   | False Positive Rate |
|----------------|-----------|--------|------|---------------------|
| Vision only    | ?         | ?      | ?    | ?                   |
| Radar only     | ?         | ?      | ?    | ?                   |
| Fused MLP      | ?         | ?      | ?    | ?                   |

Expected: fused model has lower FPR than either single-modal baseline.
This table = your paper's main result (IEEE Sensors format, ~6 pages).

## What makes this publishable
- No prior work does this on a student-budget aerial platform
- LD2410 is an ₹500 sensor — showing CFAR+MLP fusion works on COTS hardware is novel
- Comparison table above is directly reproducible by other researchers
- Training data collection method (field sessions, labeling protocol) is replicable
