# case_012_fishdoorbell_yolo_model_detector

## 1. Original Issue

Repository:

fishDoorbell

Issue:

Add a second detector backend (`ModelDetector`) that runs YOLO-family models through Ultralytics on CUDA while preserving the existing detector architecture.

The repository originally supported only motion-based detection using OpenCV background subtraction. The maintainer wanted a second detector implementation that could use trained YOLO models while keeping the existing pipeline unchanged.

Key requirements included:

* Implement a `ModelDetector`
* Support YOLO-family models through Ultralytics
* Support CUDA execution
* Fail loudly when CUDA is unavailable
* Keep motion detection as the default backend
* Add CLI flags for model selection and configuration
* Maintain compatibility with the existing `DetectionDecider` and `FrameWriter`
* Add test coverage
* Support TensorRT `.engine` models through Ultralytics

---

## 2. Problem Summary

The project architecture already contained a detector abstraction through the `Detector` protocol.

The challenge was not fixing a bug.

The challenge was implementing a new detector backend while preserving the existing architecture and satisfying all acceptance criteria.

The new detector needed to:

* Integrate through the existing interface
* Produce standard `Detection` objects
* Operate on CUDA
* Support YOLO `.pt` and TensorRT `.engine` models
* Leave downstream code unchanged

---

## 3. Root Cause Analysis

This was a feature implementation issue rather than a software defect.

The repository already had:

```text
FrameSource
    ↓
MotionDetector
    ↓
DetectionDecider
    ↓
FrameWriter
```

The maintainer intentionally designed the project around a detector abstraction.

The correct solution was not to modify downstream systems.

The correct solution was to implement a second detector backend that emitted the same `Detection` objects as the motion detector.

This demonstrates the value of interface-based design.

---

## 4. Investigation

### Repository Inspection

Project structure:

```text
src/fish_spotter/
├── cli.py
├── detector.py
├── motion.py
├── decider.py
├── writer.py
└── source.py
```

Inspection of:

```bash
grep -R "detector\|motion\|argparse\|cuda\|YOLO" -n .
```

revealed:

* MotionDetector already existed
* Detector protocol already existed
* CLI already existed
* No model-based detector existed

---

### Understanding the Detector Protocol

`detector.py` defined:

```python
class Detector(Protocol):
    def detect(self, frame):
        ...
```

This meant any detector backend only needed to return a list of `Detection` objects.

The rest of the system would continue working automatically.

---

### Understanding Optional Dependencies

The project already contained:

```toml
[project.optional-dependencies]
gpu = ["torch", "ultralytics"]
```

This allowed:

```bash
pip install -e ".[gpu]"
```

while keeping the default installation lightweight.

This satisfied the maintainer's requirement that GPU dependencies remain optional.

---

### CUDA Validation

The issue explicitly stated:

> Fail loudly if CUDA is unavailable.

The implementation therefore checks:

```python
torch.cuda.is_available()
```

before model execution begins.

If CUDA is unavailable:

```python
raise RuntimeError(...)
```

This prevents silent CPU fallback.

---

## 5. Reproduction Steps

### Create Environment

```bash
python3.11 -m venv .venv
source .venv/bin/activate
```

Install project:

```bash
pip install -e ".[test]"
```

Install GPU dependencies:

```bash
pip install -e ".[gpu]"
```

---

### Verify CUDA

```python
import torch

print(torch.cuda.is_available())
print(torch.cuda.get_device_name(0))
```

Expected:

```text
True
NVIDIA GeForce RTX 2080 SUPER
```

---

### Verify Tests

```bash
pytest -v
```

Result:

```text
19 passed
```

---

## 6. Implementation

### New Files

Created:

```text
src/fish_spotter/model.py
tests/test_model.py
```

Modified:

```text
src/fish_spotter/cli.py
```

---

### New CLI Options

Added:

```text
--detector
--weights
--conf
--device
--imgsz
```

Example:

```bash
fish-spotter \
  --source video.mp4 \
  --detector model \
  --weights yolo11n.pt \
  --conf 0.25 \
  --device auto \
  --imgsz 640
```

---

### ModelDetector

Implemented:

```python
ModelDetector
```

Responsibilities:

* Load YOLO model
* Validate CUDA availability
* Execute inference
* Convert YOLO outputs into standard Detection objects
* Preserve compatibility with existing architecture

---

### Auto Device Selection

Implemented:

```python
device="auto"
```

Behavior:

```text
CUDA available → use CUDA
CUDA unavailable → fail loudly
```

---

### Detection Conversion

YOLO outputs:

```text
Bounding Box
Class ID
Confidence
```

were converted into:

```python
Detection(
    bbox=...,
    area=...,
    confidence=...,
    label=...
)
```

This preserved compatibility with:

```text
DetectionDecider
FrameWriter
```

without modification.

---

## 7. Validation

### Unit Tests

Added:

```text
tests/test_model.py
```

Coverage included:

* Detection conversion
* CUDA validation
* Detector protocol behavior

Result:

```text
19 passed
```

---

### CLI Smoke Test

Generated test video:

```python
import cv2
import numpy as np
```

Created:

```text
smoke.mp4
```

Executed:

```bash
fish-spotter \
  --source smoke.mp4 \
  --detector model \
  --weights yolo11n.pt \
  --conf 0.25 \
  --device auto \
  --imgsz 640 \
  --out ./smoke_captures
```

Output:

```text
INFO Using CUDA device for ModelDetector:
NVIDIA GeForce RTX 2080 SUPER
```

The command completed successfully without errors.

This verified:

* CLI integration
* CUDA detection
* YOLO loading
* Model execution path
* Detector compatibility

---

## 8. Pull Request Workflow

Created branch:

```text
add-model-detector
```

Commit:

```text
e0e9350
Add YOLO-based ModelDetector with CUDA support
```

Forked repository:

```text
Dartayous/fishDoorbell
```

Opened Pull Request:

```text
Add YOLO-based ModelDetector with CUDA support
```

---

## 9. Technical Findings

### Ultralytics

Ultralytics automatically supports:

```text
.pt
.engine
```

through the same:

```python
YOLO(weights)
```

API.

This means TensorRT support required no additional code path.

---

### Detector Interfaces

A well-designed interface allows:

```text
MotionDetector
```

and

```text
ModelDetector
```

to be swapped with a single CLI flag.

This minimizes downstream changes.

---

### Optional Dependencies

Optional extras:

```toml
gpu = ["torch", "ultralytics"]
```

allow advanced features without increasing installation complexity for all users.

---

### CUDA Fail-Loud Behavior

Silent fallback can hide deployment issues.

Failing loudly:

```python
raise RuntimeError(...)
```

provides clearer diagnostics and more predictable behavior.

---

## 10. Lessons Learned

### Lesson 1

A working implementation is not the same as satisfying a specification.

Acceptance criteria determine success.

---

### Lesson 2

Interface-based design allows major functionality changes with minimal disruption.

---

### Lesson 3

Optional dependencies are a clean way to separate lightweight and advanced installations.

---

### Lesson 4

CUDA availability should be validated before model execution begins.

---

### Lesson 5

Tests must evolve alongside features.

Adding functionality often requires updating or extending existing tests.

---

### Lesson 6

Fork → Branch → Commit → Push → Pull Request is the standard open-source contribution workflow.

---

### Lesson 7

Many real-world engineering tasks are feature implementations rather than bug fixes.

Understanding requirements is often more important than writing code quickly.

---

## 11. Final Outcome

Successfully implemented a YOLO-based detector backend for FishDoorbell.

Completed:

* Feature implementation
* CUDA validation
* CLI integration
* Unit testing
* Smoke testing
* Fork workflow
* Pull request creation

This case represents a complete software engineering workflow from issue analysis to tested implementation and pull request submission.
