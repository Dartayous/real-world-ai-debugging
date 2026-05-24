# Case 006 — MMCV CARAFE ONNX Export Analysis

## Original Issue

GitHub Issue:
https://github.com/HVision-NKU/OffSeg/issues/3

The issue involved failure when attempting to export the OffSeg segmentation model to ONNX using MMDeploy.

The export pipeline failed because unsupported operators inside the model graph could not be converted cleanly into ONNX format.

The investigation focused on:

- ONNX export compatibility
- MMDeploy limitations
- custom MMCV operators
- deployment constraints in OpenMMLab segmentation models

---

# Problem Summary

The OffSeg segmentation model relied on custom OpenMMLab/MMCV CUDA operators that were not fully supported by ONNX export pipelines.

The export path failed not because the model itself was broken, but because deployment runtimes such as:

- ONNX
- TensorRT
- MMDeploy

only support specific operator sets.

The investigation identified:

```python
from mmcv.ops.carafe import carafe
```

inside:

```text
freqfusion.py
```

as a likely root cause.

---

# Root Cause Analysis

The OffSeg architecture used advanced custom CUDA operators implemented through MMCV.

One major operator identified was:

```text
CARAFE
```

which stands for:

```text
Content-Aware ReAssembly of FEatures
```

CARAFE performs intelligent feature upsampling and reconstruction during segmentation.

Unlike standard PyTorch operations such as:

```python
torch.nn.Upsample()
```

CARAFE depends on:

- custom CUDA kernels
- MMCV compiled extensions
- nonstandard operator implementations

These operators are difficult for ONNX exporters to translate into portable runtime graphs.

---

# Why ONNX Export Failed

ONNX export pipelines work best with:

- standard PyTorch tensor operations
- supported ONNX operator sets
- portable graph representations

Custom CUDA operators often cannot be represented cleanly inside ONNX graphs.

As a result:

- export may fail
- graph conversion may break
- unsupported operator errors may occur
- deployment runtimes may reject the model

---

# Reproduction Steps

## STEP 1 — Locate OffSeg Configuration

Search for OffSeg model configs:

```bash
find local_configs/offseg -iname "*.py"
```

---

## STEP 2 — Inspect OffSeg Decode Head

Locate the decode head implementation:

```bash
grep -R "class OffSegHead" -n mmseg
```

---

## STEP 3 — Inspect Custom Operators

Review imported operators:

```python
from mmcv.ops.carafe import carafe
```

This revealed the use of MMCV-specific CUDA operators.

---

## STEP 4 — Search for CARAFE Usage

```bash
grep -R "carafe" -n docs requirements mmseg
```

This identified multiple CARAFE operations inside:

```text
freqfusion.py
```

---

# Proposed Fixes

## 1. Replace Unsupported Operators

Replace CARAFE-based operations with ONNX-friendly alternatives where possible.

Potential alternatives:

- `torch.nn.Upsample`
- interpolation layers
- standard convolution blocks

---

## 2. Create Custom ONNX Symbolics

Advanced deployment workflows may implement:

- custom ONNX symbolic functions
- TensorRT plugins
- deployment-specific operator handlers

However, this significantly increases engineering complexity.

---

## 3. Use Hybrid Deployment

Possible deployment strategy:

- backbone → ONNX/TensorRT
- unsupported decode head → PyTorch runtime

This avoids full export failure while preserving performance gains.

---

## 4. Validate Operator Compatibility Early

Before designing deployment pipelines, verify:

- ONNX support
- TensorRT compatibility
- MMDeploy operator support

for all custom layers.

---

# Technical Findings

Additional observations during investigation:

- Training frameworks and deployment frameworks are not identical ecosystems
- A model may train successfully but still fail deployment export
- OpenMMLab frequently uses highly specialized CUDA operators
- Deployment engineering is often constrained by operator compatibility rather than model accuracy

The investigation highlighted the growing gap between:

- research flexibility
- deployment portability

---

# Lessons Learned

## Technical Lessons

- Not all PyTorch operations are ONNX-exportable
- Custom CUDA operators complicate deployment pipelines
- Deployment systems support limited operator sets
- ONNX compatibility should be considered during model architecture design
- AI deployment engineering differs significantly from model training

---

## New Concepts Learned

- MMDeploy
- ONNX export
- CARAFE
- custom CUDA operators
- OpenMMLab deployment
- operator portability
- symbolic export
- TensorRT plugins
- deployment runtimes
- graph conversion

---

# Additional Engineering Insights

## Three Categories of PyTorch Operations

During investigation, operations naturally separated into:

| Operator Type | ONNX Compatibility |
|---|---|
| Standard PyTorch ops | Usually supported |
| TorchVision custom ops | Sometimes supported |
| MMCV custom CUDA ops | Frequently problematic |

This explains why some models export cleanly while others fail despite working perfectly during training.

---

## Research vs Deployment

Research models prioritize:

- accuracy
- flexibility
- architectural innovation

Deployment systems prioritize:

- portability
- runtime stability
- operator compatibility
- inference efficiency

These priorities sometimes conflict.

---

# Key Takeaway

A model successfully training in PyTorch does not guarantee successful deployment.

Modern AI deployment engineering requires understanding:

- operator compatibility
- graph conversion
- runtime support
- export constraints
- deployment ecosystem limitations

Many deployment failures originate not from broken models, but from unsupported operations inside otherwise valid architectures.