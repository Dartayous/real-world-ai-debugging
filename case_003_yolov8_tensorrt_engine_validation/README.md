# Case 003 — YOLOv8 TensorRT Engine Validation

## Original Issue

GitHub Issue:
https://github.com/ioannismihailidis/venvBuilderTD/issues/2

The issue involved validating whether a YOLO TensorRT `.engine` model was functioning correctly during inference and determining whether TensorRT export itself was broken.

The investigation focused on:

- `.pt` vs `.engine` inference behavior
- TensorRT export validation
- CUDA environment verification
- detection consistency
- safe inference handling

---

# Problem Summary

The goal was to verify that:

1. A YOLO PyTorch `.pt` model correctly detected objects
2. The exported TensorRT `.engine` model also detected objects
3. TensorRT inference executed successfully on GPU
4. Inference code safely handled edge cases such as missing detections

The investigation also explored why TensorRT inference may produce slightly different results than native PyTorch inference.

---

# Root Cause Analysis

The issue was ultimately not caused by a broken TensorRT engine.

The TensorRT export completed successfully, and inference executed correctly on CUDA hardware.

The primary investigation instead became:

- validating TensorRT correctness
- comparing `.pt` and `.engine` outputs
- understanding deployment tradeoffs between PyTorch and TensorRT

An important realization was that TensorRT optimization may slightly alter detection outputs due to:

- kernel fusion
- precision optimization
- tensor rounding
- confidence threshold shifts
- postprocessing differences

TensorRT prioritizes inference speed and deployment efficiency rather than bit-perfect parity with native PyTorch execution.

---

# Reproduction Steps

## STEP 1 — Create a Clean Python Environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

---

## STEP 2 — Install Required Packages

```bash
pip install ultralytics torch torchvision
```

---

## STEP 3 — Verify CUDA Availability

```python
import torch

print(torch.__version__)
print(torch.cuda.is_available())
print(torch.version.cuda)
```

Expected result:

```text
True
```

confirming CUDA-enabled PyTorch access.

---

## STEP 4 — Export TensorRT Engine

```python
from ultralytics import YOLO

model = YOLO("yolov8n.pt")

model.export(
    format="engine",
    device=0
)
```

TensorRT export successfully produced:

```text
yolov8n.engine
```

---

## STEP 5 — Compare `.pt` vs `.engine` Inference

PyTorch inference:

```python
from ultralytics import YOLO

model = YOLO("yolov8n.pt")
results = model("images/bus.jpg")
```

TensorRT inference:

```python
from ultralytics import YOLO

model = YOLO("yolov8n.engine", task="detect")
results = model("images/bus.jpg")
```

---

## STEP 6 — Validate Detection Safety

```python
boxes = results[0].boxes

print("boxes is None:", boxes is None)
print("num detections:", 0 if boxes is None else len(boxes))
```

This ensured the inference pipeline safely handled cases where no detections were returned.

---

# Proposed Fixes / Validation Strategy

## 1. Verify CUDA Before TensorRT Export

Always confirm:

```python
torch.cuda.is_available()
```

before attempting TensorRT export.

TensorRT requires proper GPU visibility through:

- NVIDIA drivers
- CUDA runtime
- CUDA-enabled PyTorch builds

---

## 2. Compare Native and Optimized Inference

Validate both:

- `.pt`
- `.engine`

models using the same image and confidence thresholds.

This helps detect:

- missing detections
- confidence shifts
- postprocessing inconsistencies

---

## 3. Handle Empty Detection Cases

Inference code should safely handle:

```python
results[0].boxes is None
```

to avoid runtime crashes during deployment.

---

# Technical Findings

Additional observations during investigation:

- TensorRT inference was substantially faster than native PyTorch inference
- TensorRT optimization may slightly alter detection outputs
- `.engine` models are optimized deployment artifacts
- `.pt` models remain more flexible and easier to debug

Observed behavior:

## PyTorch `.pt`

Detected:

- 4 persons
- 1 bus
- 1 stop sign

---

## TensorRT `.engine`

Detected:

- 4 persons
- 1 bus

The stop sign detection disappeared during TensorRT inference, likely due to small confidence or precision differences introduced during optimization.

---

# Lessons Learned

## Technical Lessons

- TensorRT prioritizes speed and deployment efficiency
- Optimized inference may not exactly match PyTorch outputs
- CUDA visibility depends on drivers, runtime compatibility, and PyTorch builds
- Deployment engineering involves balancing:
  - speed
  - accuracy
  - memory
  - latency
  - stability

---

## New Concepts Learned

- Ultralytics
- YOLO deployment pipelines
- TensorRT engines
- CUDA-enabled PyTorch
- `.pt` model files
- `.engine` deployment artifacts
- inference optimization
- GPU tensor execution
- detection postprocessing
- TensorRT validation workflows

---

# Key Takeaway

Modern AI deployment is not simply about training models.

Real-world deployment engineering requires validating:

- inference correctness
- CUDA compatibility
- runtime behavior
- optimized execution
- deployment stability

A model that works correctly in PyTorch may still behave differently after TensorRT optimization, making validation and reproducibility essential parts of production AI systems.