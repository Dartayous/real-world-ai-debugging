# Case 004 — Nimbus Zero-Padding Stitch Bug

## Original Issue

GitHub Issue:
https://github.com/angelolab/Nimbus-Inference/issues/59

The issue involved a segmentation inference failure related to image tiling and stitching behavior during Nimbus inference.

The original investigation suggested that certain combinations of:

- image dimensions
- model magnification
- tile size
- input shape
- padding calculations

could cause failures during tiled segmentation reconstruction.

---

# Problem Summary

The bug was ultimately traced to a slicing edge case inside the tile stitching logic.

The failure occurred when bottom or right-side padding values became:

```python
0
```

The original stitching logic used negative slicing:

```python
stitched[:, :, padding[0] : -padding[1], padding[2] : -padding[3]]
```

However:

```python
-0 == 0
```

in Python.

This accidentally converted intended “slice-to-end” behavior into:

```python
array[start:0]
```

which collapses the output dimension to zero width or zero height.

The result was an invalid stitched segmentation tensor.

---

# Root Cause Analysis

The issue was not caused by:

- corrupt images
- GPU memory failure
- missing segmentation masks
- invalid datasets
- failed model inference

Instead, the failure originated from Python slice semantics interacting with zero-padding edge cases.

The original code assumed:

```python
-padding_value
```

would safely represent “crop from the end.”

But when:

```python
padding_value == 0
```

Python interprets:

```python
-0
```

as:

```python
0
```

This transformed:

```python
array[start:-0]
```

into:

```python
array[start:0]
```

which returns an empty slice.

---

# Reproduction Steps

## STEP 1 — Locate Padding and Stitching Logic

Search the repository for tile and stitching operations:

```bash
grep -R "padding\|pad\|resize\|stitched\|tile\|input_shape" -n src tests | head -200
```

This helped isolate the tile reconstruction path.

---

## STEP 2 — Inspect the Stitching Logic

Original code:

```python
stitched = stitched[:, :, padding[0] : -padding[1], padding[2] : -padding[3]]
```

Potential failure case:

```python
padding = [2, 0, 3, 0]
```

which becomes:

```python
stitched[:, :, 2:0, 3:0]
```

producing zero-sized dimensions.

---

## STEP 3 — Create Regression Test

A dedicated regression test was added:

```python
def test_stitch_tiles_handles_zero_end_padding():
```

The test intentionally reproduced:

- zero bottom padding
- zero right padding

to validate the failure condition.

---

## STEP 4 — Validate Failure

Pytest confirmed the bug:

```text
AssertionError: assert (1, 1, 0, 0) == (1, 1, 8, 7)
```

demonstrating that stitching dimensions collapsed to zero.

---

# Proposed Fix

## Original Unsafe Logic

```python
stitched = stitched[:, :, padding[0] : -padding[1], padding[2] : -padding[3]]
```

---

## Safer Replacement

```python
h_end = None if padding[1] == 0 else -padding[1]
w_end = None if padding[3] == 0 else -padding[3]

stitched = stitched[:, :, padding[0]:h_end, padding[2]:w_end]
```

---

# Why the Fix Works

In Python slicing:

```python
array[start:None]
```

means:

```python
array[start:]
```

or:

> continue to the true end of the tensor.

This safely preserves dimensions when padding values are zero.

---

# Technical Findings

Additional observations during investigation:

- The issue was caused by slice semantics rather than segmentation logic
- Python negative indexing introduces subtle edge cases
- Tiled segmentation pipelines are highly sensitive to padding calculations
- Edge-case testing is critical for image stitching systems

A regression test successfully validated the fix:

```text
1 passed
```

confirming correct handling of zero-padding conditions.

---

# Lessons Learned

## Technical Lessons

- Python slicing behavior can create hidden tensor edge cases
- `-0` is interpreted as `0`
- Image stitching systems require careful boundary handling
- Regression testing is critical after fixing inference bugs
- Small tensor slicing mistakes can invalidate entire segmentation outputs

---

## New Concepts Learned

- pytest
- regression testing
- tensor slicing
- negative indexing
- tile stitching
- segmentation tiling pipelines
- padding edge cases
- OpenCV headless environments
- inference reconstruction logic

---

# Additional Engineering Insights

## What “Headless” Means

The project used:

```text
opencv-python-headless
```

A “headless” package excludes GUI/display window functionality.

This is common in:

- servers
- cloud systems
- Docker containers
- inference pipelines

where image display windows are unnecessary.

---

## What `pytest` Is

`pytest` is a general Python testing framework used to validate software behavior.

It is not specific to computer vision.

In this investigation, `pytest` was used to:

- reproduce the stitching failure
- validate the regression test
- confirm the fix worked correctly

---

# Key Takeaway

Many inference failures are not caused by neural networks themselves, but by subtle tensor manipulation bugs surrounding the pipeline.

Modern AI engineering requires understanding not only models, but also:

- tensor operations
- slicing semantics
- reconstruction logic
- boundary conditions
- and testing infrastructure

Small numerical or indexing assumptions can silently break entire inference systems.