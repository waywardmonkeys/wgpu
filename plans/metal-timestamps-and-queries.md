# Plan: Metal timestamps and queries

## Context

- Area: timestamp queries, occlusion queries, GPU–CPU time correlation on Metal.
- Relevant files:
  - `wgpu-hal/src/metal/adapter.rs`: hard-coded `timestamp_period` in `Adapter::open`.
  - `wgpu-hal/src/metal/command.rs`: query encoding, `write_timestamp`, dummy blit encoder, visibility queries.
  - `wgpu-hal/src/metal/time.rs`: presentation timing utilities.
  - `wgpu-hal/src/metal/mod.rs`: `TimestampQuerySupport` flags, `PrivateCapabilities`.
- Current design:
  - Timestamp period is inferred from the Metal device name (`"Intel"` → `83.333`, otherwise `1.0`).
  - Query support flags (`TimestampQuerySupport`) drive which encoders can use `sample_counters_in_buffer`.
  - When Metal counter sampling has quirks (e.g. blit encoders needing a real operation), wgpu works around this with a dummy blit encoder and `fill_buffer` of a query buffer.

## Current behavior (wgpu Metal)

- `Adapter::open`:
  - Creates a command queue with `MAX_COMMAND_BUFFERS = 4096`.
  - Sets `timestamp_period` based on a heuristic:
    - If device name starts with `"Intel"` → `83.333`.
    - Otherwise → `1.0` (Apple Silicon, AMD).
- `TimestampQuerySupport`:
  - Encodes which encoders (`blit`, `render`, `compute`) support timestamp queries, and whether they're usable across wgpu passes.
- `CommandState` and queries:
  - `pending_timer_queries` is a FIFO for timestamp requests when no suitable encoder is active.
  - `enter_blit`:
    - If there are pending queries and `ON_BLIT_ENCODER` is not supported, creates a dummy blit encoder:
      - Attaches counter sample buffers.
      - Uses `set_start_of_encoder_sample_index(COUNTER_DONT_SAMPLE)` and `set_end_of_encoder_sample_index`.
      - Issues a dummy `fill_buffer` to ensure the encoder is not optimized away.
      - Ends the encoder and then re-enters a fresh blit encoder for real work.
- `write_timestamp`:
  - Prefers to record timestamps in an existing encoder that supports `sample_counters_in_buffer`.
  - If no suitable encoder is open, pushes the query into `pending_timer_queries`, and forces a future blit encoder to flush them.

## Known issues / open questions

- GPU tick → nanoseconds conversion:
  - Hard-coded per-vendor constants are brittle and device-specific.
  - No use of `device.sample_timestamps` to calibrate GPU vs CPU timelines.
  - No mechanism to adapt if new GPU families appear with different timestamp characteristics.
- Behavior on devices without counter sampling support:
  - How many devices are we effectively disabling timestamp queries on?
  - Are there fallback paths we should consider (e.g. approximate timing via command buffer completion)?
- Performance and robustness:
  - Extra dummy blit encoders and `fill_buffer` operations add overhead, especially in timestamp-heavy workloads.
  - The heuristics around when to use `start_of_encoder_sample_index` vs `end_of_encoder_sample_index` are based on observed bugs, not formal guarantees.
- CTS and spec alignment:
  - How well does the current implementation satisfy WebGPU’s requirements on timestamp behavior (monotonicity, units, precision)?
  - Are there CTS expectations that we are barely meeting or failing on some hardware?

## WebKit behavior / findings

- Key WebKit references:
  - `Source/WebGPU/WebGPU/HardwareCapabilities.mm`:
    - `baseCapabilities` (timestamp counter set discovery).
    - `baseFeatures` (adding `WGPUFeatureName_TimestampQuery`).
  - `Source/WebGPU/WebGPU/QuerySet.mm`:
    - `Device::createQuerySet`.
    - `QuerySet::counterSampleBufferWithOffsetForDevice`.
    - `QuerySet::counterSampleBufferWithOffset`.
  - `Source/WebGPU/WebGPU/CommandEncoder.mm`:
    - `errorValidatingTimestampWrites`.
    - `CommandEncoder::beginComputePass` / render pass paths that attach counter sample buffers.
  - `Source/WebGPU/WebGPU/Device.mm`:
    - `Device::enableEncoderTimestamps`.
    - `Device::timestampsBuffer`.
    - `Device::resolveTimestampsForBuffer`.

- Feature exposure:
  - `HardwareCapabilities::baseCapabilities` discovers a timestamp counter set by:
    - Checking `supportsCounterSampling:MTLCounterSamplingPointAtStageBoundary`.
    - Scanning `device.counterSets` for `MTLCommonCounterSetTimestamp`.
  - If a timestamp counter set is found, WebKit exposes `WGPUFeatureName_TimestampQuery` in `baseFeatures`.
  - On Intel (x86_64), timestamp counters are disabled entirely via `isIntel` guard.
- QuerySet management:
  - `QuerySet.mm`:
    - For `WGPUQueryType_Timestamp`, uses a shared pool of `MTLCounterSampleBuffer` objects:
      - Maintains static arrays of counter sample buffers and free ranges (`m_counterSampleBuffers`, `m_counterSampleBufferFreeRanges`).
      - `counterSampleBufferWithOffsetForDevice`:
        - Allocates each sample buffer with `sampleCount = 32 KB / sizeof(uint64_t)` and `storageMode = Shared`.
        - Uses `baseCapabilities().timestampCounterSet` as the counter set.
        - Carves subranges out of these buffers and tracks offsets per `QuerySet`.
    - For `WGPUQueryType_Occlusion`, allocates a private `MTLBuffer` and stores visibility results there.
  - Each `QuerySet` tracks the `CommandEncoder`s that reference it so it can invalidate submits if the set is destroyed.
- Command encoder integration:
  - `CommandEncoder.mm`:
    - Validates timestamp writes:
      - Ensures the device has `WGPUFeatureName_TimestampQuery`.
      - Ensures the `QuerySet` type is `Timestamp` and that indices are in range and distinct.
    - When beginning a compute pass:
      - If the descriptor has `timestampWrites`, obtains the `CounterSampleBuffer` from the `QuerySet` and associates the encoder with the set.
      - If `Device::enableEncoderTimestamps()` is on **or** a counter sample buffer is present:
        - Attaches a sample buffer to `MTLComputePassDescriptor.sampleBufferAttachments[0]`.
        - Uses `startOfEncoderSampleIndex` / `endOfEncoderSampleIndex`, adjusted by the per-QuerySet offset, or falls back to a small range (0–1) when not explicitly requested.
        - Asks `Device` to track the sample buffer for later resolve.
    - Similar logic applies on the render path (attach `sampleBuffer` for render passes when needed).
- Device-side timestamp handling:
  - `Device.mm`:
    - `enableEncoderTimestamps()` is toggled at runtime via a Darwin notification; when enabled, encoder timestamps are attached even without explicit WebGPU `timestampWrites`.
    - `timestampsBuffer(commandBuffer, count)`:
      - Creates an ad-hoc `MTLCounterSampleBuffer` with the device’s `timestampCounterSet`, `sampleCount = count`, `storageMode = Shared`.
      - Tracks it per command buffer so it can be resolved later.
    - `resolveTimestampsForBuffer(commandBuffer)`:
      - For each tracked `MTLCounterSampleBuffer`, creates a `MTLBlitCommandEncoder` on the same command buffer.
      - Calls `resolveCounters:inRange:destinationBuffer:destinationOffset:` into a new `MTLBuffer`.
      - Registers a `completedHandler` that:
        - Interprets the resolved data as `MTLCounterResultTimestamp` structs.
        - Logs encoder duration (difference between paired timestamps) to the console (debugging / profiling aid).
  - WebKit does **not** appear to compute or expose a numeric `timestampPeriod` to JavaScript; instead, it focuses on using Metal counters correctly and providing diagnostics.

## Questions to investigate (later, including WebKit)

- How does WebKit’s WebGPU Metal backend:
  - Use `MTLCounterSampleBuffer`, `MTLCounterSamplingPoint`, and `sample_counters_in_buffer` across render/compute passes?
  - Handle devices where counter sampling is partially supported or buggy?
  - Integrate timestamps with presentation timing / frame pacing?
- Does WebKit perform any internal calibration (outside the WebGPU layer) to convert Metal timestamps to CPU time for tooling or higher-level telemetry?
- Are there platform-specific quirks (macOS vs iOS, Intel vs Apple Silicon) beyond the “disable timestamp counters on Intel” behavior?

## Deltas vs wgpu (initial)

- Feature gating:
  - WebKit uses `supportsCounterSampling:MTLCounterSamplingPointAtStageBoundary` + presence of a `MTLCommonCounterSetTimestamp` counter set to gate `TimestampQuery`, and explicitly disables it on Intel.
  - wgpu uses its own `PrivateCapabilities::timestamp_query_support` flags and still exposes timestamp queries on Intel with heuristic tick periods.
- Query set backing:
  - WebKit:
    - Uses shared `MTLCounterSampleBuffer` pools per device and allocates ranges per `QuerySet`, tracking offsets explicitly.
  - wgpu:
    - Creates one `CounterSampleBuffer` per query set (in `adapter.rs` / `mod.rs` via `QuerySet`), and uses workarounds (dummy blit encoder) for known Metal bugs.
- Encoder integration:
  - WebKit:
    - Attaches sample buffers via `sampleBufferAttachments` on pass descriptors, using start/end indices computed from the WebGPU descriptor and the `QuerySet` offset.
    - Has a device-wide toggle (`enableEncoderTimestamps`) to attach additional timestamp buffers for profiling.
  - wgpu:
    - Uses `sample_counters_in_buffer` directly on encoders and falls back to creating dummy blit encoders and `fill_buffer` operations when sampling is not supported on the currently open encoder.
- Resolution and diagnostics:
  - WebKit:
    - Resolves counters into dedicated `MTLBuffer`s per command buffer and logs encoder times for debugging when encoder timestamps are enabled.
  - wgpu:
    - Resolves timestamp query results into user-provided buffers through `copy_query_results`, but does not currently have internal debug logging for raw Metal counters.

## Possible improvement directions

- Introduce calibration of `timestamp_period`:
  - On startup, perform a short calibration sequence:
    - Use `device.sample_timestamps` repeatedly to measure GPU tick deltas vs CPU time deltas.
    - Fit a line (or simple average ratio) to estimate tick → nanosecond conversion.
  - Cache per-device results and optionally persist/short-circuit for repeated runs.
  - Provide a fallback path if calibration fails or is too slow (e.g. retain current heuristics).
- Tighten feature detection and flags:
  - Align `TimestampQuerySupport` with the latest Metal counter documentation and Apple samples.
  - Revisit which encoders actually support timestamp queries on modern macOS/iOS.
- Reduce dummy work:
  - Investigate whether newer OS versions still require the dummy blit write.
  - If WebKit or Dawn have a more targeted workaround (e.g. using specific sampling points), adopt similar techniques.
- Improve diagnostics:
  - Add optional logging/metrics for timestamp usage:
    - Number of dummy blit encoders created.
    - Time spent in calibration.
    - Whether timestamps are disabled on a given device/OS combination.

## Next steps when resuming

- Re-read `adapter.rs` and `command.rs` around:
  - `timestamp_period` initialization.
  - `TimestampQuerySupport` construction.
  - Query set creation and counter sample buffer allocation.
- Once WebKit sources are available:
  - Find the Metal-specific WebGPU device/queue implementation.
  - Locate their timestamp/query handling and compare:
    - Calibration or constants.
    - Encoder/workaround patterns.
    - Supported feature combinations.
  - Update this plan with concrete deltas and candidate patches.
