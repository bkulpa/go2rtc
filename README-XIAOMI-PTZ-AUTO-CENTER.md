# Xiaomi PTZ Feedback, Absolute Positioning and Auto-Center for go2rtc

> Hardware-validated open-source contribution built on top of Xiaomi MISS PTZ work by [@korasino](https://github.com/korasino).

## Overview

This project extends Xiaomi PTZ support in go2rtc with position feedback, absolute positioning and a feedback-driven auto-center flow.

The work was developed and validated on a physical Xiaomi camera reporting the model identifier:

`xiaomi.camera.c01a01`

The implementation is currently published as a Draft Pull Request and is being coordinated with the existing upstream Xiaomi ONVIF PTZ work.

## What I implemented

My contribution adds:

- correct CS2 command ID decoding for incoming PTZ feedback
- handling of trailing NUL bytes in Xiaomi `0x113` motor responses
- PTZ position refresh using motor operation `6`
- absolute positioning using motor operation `13`
- feedback-driven auto-center
- regression and unit tests for the protocol fixes and centering logic

The auto-center flow uses real position feedback instead of assuming that a motor command succeeded:

```text
RefreshPosition
      |
      v
SetPosition(50, 50)
      |
      v
wait for motor movement
      |
      v
RefreshPosition
      |
      v
verify target tolerance
```

If the camera does not reach the expected coordinates, the operation can retry and verifies the final position from fresh device feedback.

## Protocol findings

During real-device testing I found two issues that prevented reliable PTZ feedback.

### 1. CS2 command ID endianness

Incoming CS2 command IDs were being read as little-endian, while the received command bytes use big-endian representation.

Example:

```text
00 00 10 01 -> 0x1001
```

Correct decoding allowed the Xiaomi motor response to be recognized and decrypted as `0x113`.

### 2. NUL-padded motor responses

The camera returns valid JSON with a trailing NUL byte.

Example:

```text
{"ret":0,"angle":48,"elevation":0}\x00
```

The parser now trims the trailing NUL padding before JSON decoding.

## Hardware validation

The implementation was tested on real hardware rather than only with mocked protocol data.

Verified behavior on `xiaomi.camera.c01a01`:

- operation `6` returns current PTZ position through `0x113`
- operation `13` performs absolute positioning
- requested position `48/50` resulted in reported position `47/49`
- auto-center moved the camera from `45/37` to `49/49`
- a separate position refresh confirmed the final `49/49` state
- video remained active during PTZ operations

The target center is intentionally verified with a tolerance because the camera firmware does not always report the exact requested coordinate.

## Validation

The final implementation was checked with:

```text
go test
go test -race
go vet
git diff --check
```

Targeted package tests passed for:

- `pkg/xiaomi/miss/cs2`
- `pkg/xiaomi/miss`
- `internal/xiaomi`

## My commits

### CS2 PTZ feedback fix

[`c88e1c9` — fix(xiaomi): handle CS2 PTZ feedback correctly](https://github.com/bkulpa/go2rtc/commit/c88e1c980d7ac565bd23cee2b992a7ca34f19990)

This commit contains the CS2 endianness fix, NUL-response handling and regression tests.

### Feedback-driven auto-center

[`a23db13` — feat(xiaomi): add feedback-driven PTZ auto-center](https://github.com/bkulpa/go2rtc/commit/a23db132dc2c5e8d7a1460ea88c7d3a3c550d1dc)

This commit adds absolute-position-based auto-center, API integration and unit tests.

## Open-source context

This work builds on the Xiaomi MISS PTZ implementation created by [@korasino](https://github.com/korasino/go2rtc/tree/feat/xiaomi-miss-ptz).

I intentionally kept my work in separate commits so the authorship and scope of the follow-up changes remain clear.

Current Draft PR:

- [korasino/go2rtc#1 — Xiaomi: add PTZ feedback, absolute positioning and auto-center](https://github.com/korasino/go2rtc/pull/1)

Related upstream discussion:

- [AlexxIT/go2rtc#2162 — Add PTZ support for Xiaomi cameras via xiaomi:// protocol exposed as ONVIF](https://github.com/AlexxIT/go2rtc/issues/2162)
- [AlexxIT/go2rtc#2531 — Xiaomi pan/tilt motor control + ONVIF PTZ](https://github.com/AlexxIT/go2rtc/pull/2531)

## Status

This is an active open-source contribution and has **not yet been merged into the official go2rtc master branch**.

The code, hardware test results and Draft PR are public and available for review.
