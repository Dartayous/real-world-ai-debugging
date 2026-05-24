# Case 011 — TensorRT End-to-End Postprocessing Analysis

## Original Issue

GitHub Issue:
https://github.com/branes-ai/auto-drone/issues/17

The issue involved TensorRT YOLO inference behavior and questions surrounding end-to-end deployment, postprocessing, and detection pipeline handling.

The investigation focused on understanding:

- TensorRT inference pipelines
- YOLO deployment structure
- end-to-end inference behavior
- postprocessing responsibilities
- NMS handling
- deployment architecture tradeoffs

The issue appeared to involve confusion around where detection postprocessing should occur during TensorRT deployment.

---

# Problem Summary

Modern object detection systems involve more than neural network inference alone.

A YOLO deployment pipeline typically includes:

- image preprocessing
- model inference
- confidence filtering
- bounding box decoding
- Non-Maximum Suppression (NMS)
- final detection formatting

The investigation explored how TensorRT deployment may handle these stages differently depending on export configuration and deployment architecture.

---

# Root Cause Analysis

The core realization was that:

```text
TensorRT only optimizes what is actually inside the exported graph.
```

Some YOLO export pipelines include:

- NMS
- decoding
- postprocessing

inside the TensorRT engine itself.

Other pipelines leave postprocessing outside the engine in Python or C++ runtime code.

This creates two major deployment styles:

---

# Deployment Style 1 — End-to-End Engine

TensorRT engine includes:

- inference
- box decoding
- NMS
- postprocessing

Output becomes:

```text
final detections
```

Advantages:

- very fast
- simplified runtime
- minimal Python overhead

Disadvantages:

- less flexible
- harder to debug
- engine becomes more specialized

---

# Deployment Style 2 — Split Pipeline

TensorRT handles only:

- neural network inference

while Python/C++ handles:

- decoding
- confidence filtering
- NMS
- tracking logic

Advantages:

- easier debugging
- easier customization
- more flexible deployment

Disadvantages:

- slightly slower
- more runtime overhead

---

# Understanding Postprocessing

Object detection models do not directly output final bounding boxes.

Raw outputs usually contain:

- candidate boxes
- objectness scores
- class scores

Postprocessing converts raw tensor outputs into usable detections.

---

# What NMS Does

Non-Maximum Suppression (NMS) removes duplicate detections.

Example:

```text
5 overlapping boxes around the same car
```

NMS keeps:

```text
the best box
```

and removes the others.

This prevents multiple detections for the same object.

---

# Reproduction Steps

## STEP 1 — Export TensorRT Engine

Typical export:

```python
from ultralytics import YOLO

model = YOLO("yolov8n.pt")

model.export(
    format="engine",
    device=0
)
```

---

## STEP 2 — Inspect Runtime Outputs

Compare:

- raw TensorRT outputs
- final detections
- postprocessed results

Determine whether:

- NMS exists inside the engine
- postprocessing occurs externally

---

## STEP 3 — Compare Deployment Architectures

Analyze whether the deployment pipeline uses:

```text
End-to-End TensorRT
```

or:

```text
TensorRT + external postprocessing
```

---

# Proposed Fixes / Recommendations

## 1. Clarify Pipeline Boundaries

Always determine:

```text
What is inside the engine?
What remains outside the engine?
```

before debugging deployment behavior.

---

## 2. Validate NMS Placement

Ensure only one NMS stage exists.

Duplicate NMS stages can cause:

- missing detections
- inconsistent results
- over-filtering

---

## 3. Compare PyTorch vs TensorRT Outputs

Validate:

- raw outputs
- confidence thresholds
- NMS behavior
- decoded detections

to ensure deployment consistency.

---

## 4. Prefer Simpler Pipelines During Debugging

When troubleshooting deployment:

- disable unnecessary optimizations
- isolate inference stages
- validate outputs step-by-step

---

# Technical Findings

Additional observations during investigation:

- Deployment systems often blur the boundary between inference and postprocessing
- TensorRT engines may include custom postprocessing plugins
- Detection pipelines are highly sensitive to threshold handling
- Deployment architectures vary substantially between repositories

The investigation reinforced that:

```text
Inference ≠ final detections
```

Many critical operations occur after raw neural network execution.

---

# Lessons Learned

## Technical Lessons

- Object detection pipelines contain multiple stages beyond inference
- TensorRT may or may not include postprocessing internally
- NMS placement is critical in deployment systems
- End-to-end optimization trades flexibility for speed
- Deployment debugging requires understanding full pipeline architecture

---

## New Concepts Learned

- end-to-end TensorRT
- detection postprocessing
- bounding box decoding
- NMS
- deployment architecture
- inference pipelines
- TensorRT plugins
- confidence filtering
- raw model outputs
- deployment optimization

---

# Additional Engineering Insights

## Why Deployment Pipelines Become Complex

Research code often focuses on:

- training
- accuracy
- experimentation

Production deployment focuses on:

- latency
- throughput
- memory efficiency
- runtime stability

This causes deployment systems to aggressively optimize and fuse pipeline stages.

---

## Why TensorRT Is Powerful

TensorRT can:

- fuse kernels
- optimize tensor execution
- reduce precision
- optimize memory access
- accelerate inference dramatically

But increased optimization often reduces:

- transparency
- debuggability
- flexibility

---

# Key Takeaway

Modern AI deployment systems involve far more than neural network inference alone.

Real-world object detection pipelines depend on:

- preprocessing
- inference
- decoding
- NMS
- postprocessing
- runtime optimization

Successful deployment engineering requires understanding the entire inference pipeline, not just the neural network itself.