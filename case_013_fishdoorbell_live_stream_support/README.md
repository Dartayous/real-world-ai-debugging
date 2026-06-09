# case_013_fishdoorbell_live_stream_support

## Repository

fishDoorbell

## Issue

Live Stream Support with Reconnect/Backoff (#4)

PR Status: Open

---

# 1. Original Issue

The Fish Spotter project originally supported only local video files.

The maintainer wanted to extend the pipeline so it could operate directly on the live Visdeurbel fish migration stream.

The requirements included:

* Resolve a playable stream URL from the Visdeurbel website
* Support live stream processing
* Sample frames at a configurable FPS
* Automatically reconnect if the stream drops
* Prevent single dropped frames from crashing the pipeline
* Preserve existing file-based functionality
* Add test coverage for reconnect behavior

The existing file-based workflow had to remain unchanged.

---

# 2. Problem Summary

This was not a machine learning problem.

This was a production video-ingestion problem.

The challenge involved:

* Resolving a playable stream URL from a webpage
* Handling unreliable network streams
* Preventing transient failures from stopping the application
* Preserving backward compatibility with the existing architecture

The new functionality needed to coexist with the existing file-mode workflow.

---

# 3. Root Cause Analysis

The Visdeurbel page does not expose a direct video file.

Instead, the page loads the live stream dynamically.

Initial assumptions were:

1. Streamlink would resolve the stream automatically.
2. yt-dlp would resolve the stream automatically.

Neither approach successfully resolved the Visdeurbel page.

Further investigation of the page source revealed an HLS stream endpoint embedded in the HTML:

```text
https://visdeurbel.videostreams.nl/hls/visdeurbel/index.m3u8
```

This demonstrated that the stream existed but required additional discovery logic.

---

# 4. Investigation Process

## Step 1 — Review Existing Architecture

The existing architecture consisted of:

```text
FrameSource
     ↓
MotionDetector
     ↓
DetectionDecider
     ↓
FrameWriter
```

The safest solution was to extend FrameSource rather than modifying downstream components.

---

## Step 2 — Add Stream Resolution

Created:

```python
StreamResolver
```

Resolution attempts occur in this order:

1. Streamlink
2. yt-dlp
3. HTML HLS extraction fallback

This provides multiple paths to obtain a playable stream URL.

---

## Step 3 — Add Live Mode

FrameSource was extended with:

```python
live=True
fps=3
```

Live mode:

* Opens the stream
* Samples frames at the requested FPS
* Continues yielding frames to the pipeline

The rest of the pipeline remains unchanged.

---

## Step 4 — Add Reconnect Logic

Live streams can fail temporarily.

A transient failure should not terminate the process.

Reconnect behavior was implemented using:

```python
reconnect_backoff
max_consecutive_failures
```

Behavior:

* Ignore isolated frame failures
* Reconnect after sustained failures
* Continue processing after recovery

---

## Step 5 — Add Unit Tests

A fake capture source was created to simulate:

```text
frame
frame
failure
failure
failure
reconnect
frame
```

The test verifies that:

* The iterator survives failure
* Reconnect occurs
* Frames continue after recovery

---

# 5. Final Implementation

Added:

### StreamResolver

Responsible for converting a webpage URL into a playable stream URL.

Supports:

* Streamlink
* yt-dlp
* HTML HLS extraction

---

### Live FrameSource Mode

Supports:

```bash
fish-spotter --url https://visdeurbel.nl/en/
```

and

```bash
fish-spotter --fps 3
```

---

### Reconnect Backoff

Supports automatic recovery from:

* Network interruptions
* Temporary stream outages
* Decoder failures

---

### Test Coverage

Added reconnect simulation testing.

Verified:

* Recovery after transient failure
* Existing file-mode functionality remains intact

---

# 6. Validation

## Unit Tests

```text
18 passed
```

---

## Real Stream Validation

Resolved:

```text
https://visdeurbel.videostreams.nl/hls/visdeurbel/index.m3u8
```

Successfully opened with OpenCV.

Successfully received:

```text
1920x1080 frames
```

Sample output:

```text
0 (1080, 1920, 3)
1 (1080, 1920, 3)
2 (1080, 1920, 3)
```

Live smoke test completed successfully.

---

# 7. Engineering Lessons Learned

## Lesson 1 — Third-Party Tools Fail

The original plan relied on:

* Streamlink
* yt-dlp

Neither solved the problem completely.

Real-world engineering often requires investigation beyond the expected toolchain.

---

## Lesson 2 — Verify Assumptions

The stream existed.

The tools simply failed to locate it.

Inspecting the actual webpage revealed the underlying HLS endpoint.

Always verify assumptions with direct evidence.

---

## Lesson 3 — Resilience Matters

Local files are predictable.

Live streams are not.

Production systems must assume:

* Disconnects
* Timeouts
* Corrupt frames
* Temporary outages

Recovery logic is often more important than the happy path.

---

## Lesson 4 — Preserve Existing Architecture

The best solution was not a rewrite.

The existing pipeline was already well designed.

By extending FrameSource and introducing StreamResolver, all downstream components continued to work unchanged.

This minimized risk and reduced maintenance cost.

---

## Lesson 5 — Test Failure Scenarios

Most bugs occur during failure conditions.

The reconnect test validated behavior that would otherwise be difficult to reproduce consistently.

Testing failure paths is often more valuable than testing success paths.

---

# 8. Outcome

Successfully implemented live-stream support for Fish Spotter.

Added:

* Stream resolution
* Live stream ingestion
* Configurable FPS sampling
* Automatic reconnect with backoff
* Recovery testing

The implementation preserved backward compatibility while extending the project to operate on a real-world live video stream.

PR submitted to the repository maintainer for review.
