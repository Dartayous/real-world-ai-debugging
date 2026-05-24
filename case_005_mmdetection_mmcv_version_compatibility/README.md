# Case 005 — MMDetection / MMCV Version Compatibility Analysis

## Original Issue

GitHub Issue:
https://github.com/BinItAI/visdet/issues/303

The issue involved inference instability and segmentation faults occurring inside an OpenMMLab-based object detection environment.

The reported environment included:

```text
MMDetection: 2.28.2+
MMCV: 1.7.2
PyTorch: 2.4.0.dev
TorchVision: 0.19.0.dev
```

along with:

```text
MMCV CUDA Compiler: not available
```

The system crashed during inference with:

```text
Segmentation fault (core dumped)
```

---

# Problem Summary

The failure appeared to originate from major compatibility mismatches across the OpenMMLab software stack.

The environment combined:

- older OpenMMLab framework versions
- very new nightly PyTorch builds
- CUDA-dependent compiled operators
- potentially incompatible binary extensions

The issue was likely not the detection model itself, but instability caused by mismatched framework layers and compiled CUDA components.

---

# Root Cause Analysis

The OpenMMLab ecosystem is highly version-sensitive.

Frameworks such as:

- MMDetection
- MMSegmentation
- MMDeploy

depend heavily on:

- MMCV
- PyTorch
- TorchVision
- CUDA
- compiled C++/CUDA operators

The reported environment mixed:

## Older OpenMMLab Stack

```text
MMDetection 2.28.2
MMCV 1.7.2
```

with:

## Very New Nightly Builds

```text
PyTorch 2.4.0.dev
TorchVision 0.19.0.dev
```

This created a strong likelihood of:

- binary incompatibility
- CUDA extension mismatch
- compiled operator instability
- segmentation faults during runtime

---

# Understanding the OpenMMLab Stack

A major lesson learned during investigation was how OpenMMLab frameworks are layered.

Framework hierarchy:

```text
PyTorch
   ↓
MMCV
   ↓
MMDetection
```

---

## What MMCV Is

`MMCV` is the core infrastructure layer of OpenMMLab.

It provides:

- CUDA operators
- tensor utilities
- convolution modules
- training helpers
- deployment utilities
- compiled GPU extensions

Many advanced OpenMMLab operations depend directly on MMCV CUDA extensions.

---

## What MMDetection Is

`MMDetection` is the object detection framework built on top of:

- PyTorch
- MMCV

It provides implementations for:

- Faster R-CNN
- Mask R-CNN
- RetinaNet
- DETR
- Cascade R-CNN
- YOLO integrations

along with complete training and inference pipelines.

---

# Reproduction Steps

## STEP 1 — Inspect Installed Versions

Verify environment versions:

```python
import torch
import mmcv
import mmdet

print(torch.__version__)
print(mmcv.__version__)
print(mmdet.__version__)
```

---

## STEP 2 — Identify Version Mismatch

Compare:

- MMDetection compatibility requirements
- MMCV compatibility requirements
- installed PyTorch versions

Nightly PyTorch builds were likely incompatible with the older OpenMMLab ecosystem.

---

## STEP 3 — Verify CUDA Extension Availability

The environment reported:

```text
MMCV CUDA Compiler: not available
```

This strongly suggested that compiled CUDA extensions were missing or incompatible.

---

## STEP 4 — Rebuild Stable Environment

Suggested stable environment:

```bash
conda create -n mmdet228 python=3.8 -y
conda activate mmdet228

pip install torch==1.13.1 torchvision==0.14.1 \
  --index-url https://download.pytorch.org/whl/cu117

pip install -U openmim

mim install "mmcv-full==1.7.2"

pip install "mmdet==2.28.2"
```

---

# Proposed Fixes

## 1. Avoid Mixing Old Frameworks with Nightly Builds

Nightly PyTorch versions may not remain compatible with older OpenMMLab ecosystems.

---

## 2. Align OpenMMLab Versions Carefully

OpenMMLab frameworks require closely aligned versions between:

- MMDetection
- MMCV
- MMEngine
- PyTorch
- TorchVision

---

## 3. Verify CUDA Extension Compilation

Many MMCV operators depend on:

- native C++
- CUDA kernels
- compiled GPU extensions

Missing or incompatible CUDA extensions can cause native crashes and segmentation faults.

---

## 4. Prefer Stable Release Builds

Stable released environments are generally safer than nightly development builds when working with compiled CV frameworks.

---

# Technical Findings

Additional observations during investigation:

- OpenMMLab environments are extremely sensitive to dependency mismatches
- MMCV acts as a foundational dependency layer
- Many OpenMMLab operators rely on compiled CUDA extensions
- Segmentation faults often indicate low-level binary incompatibility rather than Python logic errors

The investigation demonstrated how fragile AI/CV ecosystems become when combining:

- CUDA
- compiled extensions
- evolving frameworks
- nightly PyTorch builds

---

# Lessons Learned

## Technical Lessons

- AI/CV ecosystems are deeply layered software stacks
- Version alignment is critical in OpenMMLab environments
- Native CUDA extensions can fail below Python-level visibility
- Segmentation faults often originate from binary/runtime incompatibility
- Stable reproducible environments are essential in production ML systems

---

## New Concepts Learned

- OpenMMLab
- MMCV
- MMDetection
- framework layering
- CUDA extensions
- binary compatibility
- compiled operators
- segmentation faults
- dependency ecosystems
- environment reproducibility

---

# Additional Engineering Insights

## Why OpenMMLab Environments Are Fragile

OpenMMLab systems operate as a deeply interconnected stack:

```text
Python
PyTorch
CUDA
TorchVision
MMCV
MMDetection
MMDeploy
TensorRT
```

A mismatch at any layer can destabilize the entire runtime.

---

## Why Containers Became Important in AI/CV

Modern AI environments are difficult to reproduce consistently.

Technologies such as Docker help preserve:

- exact package versions
- CUDA compatibility
- compiler environments
- runtime dependencies

reducing environment drift and deployment instability.

---

# Key Takeaway

Modern AI engineering is not only about neural networks.

Real-world computer vision systems depend heavily on:

- runtime compatibility
- infrastructure stability
- CUDA extension support
- framework version alignment
- deployment reproducibility

Many ML/CV failures are ecosystem failures rather than model failures.