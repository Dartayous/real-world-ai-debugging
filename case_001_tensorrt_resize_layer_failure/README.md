# Case 001 — TensorRT Resize Layer Failure (Faster R-CNN)

## Original Issue

GitHub Issue:
https://github.com/Alex-Derhacobian/tensorrt_frcnn_bug

The issue involved a TensorRT failure around:

```text
Resize_62
```

during Faster R-CNN ONNX → TensorRT conversion.

---

# Problem Summary

TensorRT 8.0.1 appeared to struggle with a dynamically-shaped ONNX `Resize` operation originating from the Faster R-CNN Feature Pyramid Network (FPN) path.

Faster R-CNN exports highly dynamic ONNX graphs involving:

- Feature Pyramid Networks (FPN)
- Proposal generation
- ROI pooling/alignment
- Bounding-box decoding
- Resize operations
- NMS-style logic

These graph patterns are significantly more complex than single-stage detectors such as YOLO.

---

# Root Cause Analysis

The likely root cause was not a corrupted ONNX layer, but rather TensorRT 8.0.1 struggling with dynamically-generated shape tensors feeding into a Resize operation.

TensorRT 8.0.1 is now considered very old, and newer TensorRT releases provide substantially improved:

- ONNX parser support
- Dynamic shape handling
- Resize operator support
- Detection-model compatibility

Faster R-CNN models are especially sensitive because they internally generate shape-dependent feature alignment operations.

---

# Reproduction Steps

## STEP 1 — Inspect the ONNX graph

Install debugging tools:

```bash
pip install onnx onnxsim polygraphy
```

Inspect the model:

```bash
polygraphy inspect model frcnn_opset=11.onnx --show layers attrs weights
```

---

## STEP 2 — Sanitize the ONNX graph

Apply graph cleanup and constant folding:

```bash
polygraphy surgeon sanitize frcnn_opset=11.onnx \
  --fold-constants \
  -o frcnn_sanitized.onnx
```

Retry TensorRT conversion:

```bash
trtexec --onnx=frcnn_sanitized.onnx \
  --verbose \
  --workspace=4096
```

---

# Proposed Fixes

## 1. Use Staticized Shapes

Even though the model input was fixed at:

```text
1x3x1080x1440
```

Faster R-CNN internally still generates dynamic shape logic.

The goal is to reduce dynamically-generated Resize operations.

---

## 2. Upgrade TensorRT

TensorRT 8.0.1 is extremely old by modern deployment standards.

Newer TensorRT releases contain major improvements for:

- ONNX parsing
- Dynamic tensor support
- Resize handling
- Detection model export

---

## 3. Split the Model Pipeline

If full TensorRT export still fails:

- Backbone + FPN → TensorRT
- ROI Heads / postprocessing → PyTorch or ONNX Runtime

This hybrid deployment strategy is often more stable for two-stage detectors.

---

# Technical Findings

Additional observations during investigation:

- TensorRT failures were likely tied to dynamic Resize graph generation
- ONNX graph simplification may reduce parser failures
- Faster R-CNN export paths are significantly more fragile than YOLO export paths
- TensorRT deployment for Torchvision detection models often requires graph cleanup

Additional code observations:

- Missing `:` identified in `check_model.py`
- `onnx.load(filename)` may require `onnx_path`

---

# Lessons Learned

## Technical Lessons

- TensorRT is highly sensitive to dynamic graph structures
- Faster R-CNN export paths are substantially more complex than YOLO
- ONNX graph simplification can reduce parser instability
- Two-stage detectors introduce many deployment edge cases

---

## New Concepts Learned

- R-CNN
- Fast R-CNN
- Faster R-CNN
- Feature Pyramid Networks (FPN)
- ROI Pooling / ROI Align
- ONNX graph sanitization
- TensorRT parser limitations
- Dynamic shape tensors
- Non-Maximum Suppression (NMS)

---

# Key Takeaway

Many deployment failures are not caused by "broken models," but by incompatibilities between:

- ONNX graph structure
- dynamic tensor logic
- TensorRT parser support
- deployment runtime expectations

Modern AI engineering often involves debugging the interfaces BETWEEN systems rather than the model itself.