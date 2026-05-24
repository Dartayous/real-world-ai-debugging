# Case 008 — Rectangular YOLO TensorRT Inference

## Original Issue

GitHub Issue:
https://github.com/Linaom1214/TensorRT-For-YOLO-Series/issues/158

The issue involved TensorRT export and inference behavior when using rectangular (non-square) YOLO input resolutions.

The investigation focused on:

- rectangular ONNX export
- TensorRT engine conversion
- CUDA device visibility
- deployment compatibility
- inference behavior using non-square image dimensions

Example input shape:

```text
320x640
```

instead of the more common:

```text
640x640
```

---

# Problem Summary

The export pipeline successfully generated a rectangular ONNX model, but TensorRT engine export failed with:

```text
ValueError: Invalid CUDA 'device=0' requested
```

despite:

```python
torch.cuda.is_available()
```

returning:

```text
True
```

The investigation became focused on understanding:

- CUDA visibility
- TensorRT export environments
- rectangular inference support
- differences between ONNX and TensorRT export behavior

---

# Root Cause Analysis

The issue appeared to involve inconsistent CUDA visibility between:

- direct PyTorch runtime execution
- Ultralytics export logic
- TensorRT export initialization

A major clue was:

```python
torch.cuda.is_available() == True
```

while Ultralytics internally reported:

```text
torch.cuda.is_available(): False
```

during export.

This suggested that:

- subprocess environment handling
- CUDA initialization state
- TensorRT export runtime behavior

was behaving differently than direct PyTorch execution.

---

# Understanding Rectangular Inference

YOLO models are often trained and exported using square inputs such as:

```text
640x640
```

However, rectangular inference uses dimensions such as:

```text
320x640
```

Advantages include:

- preserving aspect ratio
- reducing image distortion
- improving efficiency for widescreen inputs
- lowering inference cost

However, rectangular shapes introduce additional complexity during:

- ONNX export
- TensorRT optimization
- static shape inference
- deployment validation

---

# Reproduction Steps

## STEP 1 — Create Clean Environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

---

## STEP 2 — Verify CUDA Availability

```python
import torch

print(torch.__version__)
print(torch.cuda.is_available())
print(torch.version.cuda)
```

Expected:

```text
True
```

---

## STEP 3 — Export Rectangular ONNX Model

Example export:

```python
from ultralytics import YOLO

model = YOLO("yolo11n.pt")

model.export(
    format="onnx",
    imgsz=[320, 640]
)
```

ONNX export completed successfully.

---

## STEP 4 — Attempt TensorRT Export

```python
model.export(
    format="engine",
    imgsz=[320, 640],
    device=0
)
```

TensorRT export failed with:

```text
ValueError: Invalid CUDA 'device=0' requested
```

---

## STEP 5 — Validate CUDA Directly

Direct CUDA tensor test:

```python
import torch

x = torch.tensor([1.0], device="cuda:0")

print(x)
print(torch.cuda.get_device_name(0))
```

Result:

```text
cuda tensor ok
NVIDIA GeForce RTX 2080 SUPER
```

This confirmed CUDA itself was functioning correctly.

---

# Proposed Fixes

## 1. Verify CUDA Inside Export Runtime

TensorRT export pipelines may initialize CUDA differently than standard PyTorch execution.

Always validate:

```python
torch.cuda.is_available()
```

inside the actual export environment.

---

## 2. Compare ONNX vs TensorRT Behavior

The investigation showed:

| Export Type | Result |
|---|---|
| ONNX | Successful |
| TensorRT | Failed |

This helped isolate the issue specifically to TensorRT export initialization.

---

## 3. Test Square vs Rectangular Shapes

TensorRT optimization paths may behave differently for:

- square inputs
- rectangular inputs
- static shapes
- dynamic shapes

---

## 4. Validate Deployment Toolchain Versions

TensorRT export pipelines depend heavily on:

- PyTorch
- CUDA
- TensorRT
- ONNX
- Ultralytics

Version mismatches can affect export behavior.

---

# Technical Findings

Additional observations during investigation:

- CUDA itself was functioning correctly
- PyTorch CUDA tensors executed successfully
- ONNX export supported rectangular inference correctly
- TensorRT export behaved differently than direct runtime execution

The issue demonstrated how deployment pipelines may contain:

- separate CUDA initialization paths
- isolated runtime environments
- export-specific compatibility behavior

---

# Lessons Learned

## Technical Lessons

- TensorRT export behavior may differ from direct PyTorch execution
- CUDA visibility is not always globally consistent across runtimes
- Rectangular inference introduces additional deployment complexity
- ONNX export success does not guarantee TensorRT export success
- Deployment pipelines require independent validation at each stage

---

## New Concepts Learned

- rectangular inference
- TensorRT export
- ONNX export
- CUDA runtime visibility
- deployment validation
- static shapes
- dynamic shapes
- inference optimization
- TensorRT engines
- export runtime behavior

---

# Additional Engineering Insights

## Why Rectangular Inference Matters

Most real-world camera feeds are not square.

Rectangular inference can:

- preserve image geometry
- reduce unnecessary padding
- improve runtime efficiency
- better match deployment conditions

However, deployment frameworks are often optimized primarily around square input assumptions.

---

## Why Deployment Is Harder Than Training

A model may:

- train correctly
- infer correctly
- export partially

yet still fail during deployment optimization.

Deployment engineering introduces additional layers involving:

- graph conversion
- runtime optimization
- operator compatibility
- memory planning
- CUDA initialization
- hardware-specific execution

---

# Key Takeaway

Modern AI deployment systems involve far more than neural network inference.

Production deployment requires validating:

- export behavior
- runtime compatibility
- CUDA initialization
- optimization pipelines
- shape handling
- deployment reproducibility

Many deployment failures originate from infrastructure-level runtime behavior rather than the model itself.