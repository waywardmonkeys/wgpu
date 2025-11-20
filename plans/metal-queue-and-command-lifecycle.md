# Plan: Metal queue and command lifecycle

## Context

- Area: GPU queue behavior, command buffer creation/commit, encoder usage, and work-done semantics.
- Relevant wgpu files:
  - `wgpu-core` queue and command buffer modules.
  - `wgpu-hal/src/metal/mod.rs` and `command.rs`:
    - `Queue::submit`, `Queue::present`.
    - `CommandEncoder` implementation for Metal.
- Relevant WebKit files:
  - `../WebKit/Source/WebGPU/WebGPU/Queue.*`.
  - `../WebKit/Source/WebGPU/WebGPU/CommandEncoder.*`.
  - `../WebKit/Source/WebGPU/WebGPU/CommandBuffer.*`.

## Current behavior (wgpu Metal, conceptually)

- Queue:
  - Creates a Metal command queue with a fixed max command buffer count (`MAX_COMMAND_BUFFERS`).
  - `Queue::submit` commits command buffers and manages fences using `MTLSharedEvent` (when supported) and completion handlers.
- Command encoder:
  - Manages render/compute/blit encoders.
  - Implements timestamp queries with workarounds for Metal quirks.
  - Performs minimal state tracking for command buffer lifecycle (e.g. `begin_encoding`, `end_encoding`, `discard_encoding`).

## WebKit behavior / findings (initial)

- Key WebKit references:
  - `Source/WebGPU/WebGPU/Queue.mm`:
    - `Queue::Queue` (initialization).
    - `Queue::commandBufferWithDescriptor`.
    - `Queue::ensureBlitCommandEncoder`, `Queue::finalizeBlitCommandEncoder`.
    - `Queue::onSubmittedWorkDone`, `Queue::onSubmittedWorkScheduled`.
    - `Queue::removeMTLCommandBuffer`, `Queue::makeInvalid`.
  - `Source/WebGPU/WebGPU/CommandEncoder.mm`:
    - `CommandEncoder::beginComputePass` / render pass begin.
    - `CommandEncoder::discardCommandBuffer`.
    - `CommandEncoder::endEncoding`.
  - `Source/WebGPU/WebGPU/CommandBuffer.mm`:
    - Command buffer submission and completion handling.

- Queue:
  - `Queue.mm`:
    - Tracks created-but-not-committed command buffers in `m_createdNotCommittedBuffers`.
    - Enforces a maximum outstanding command buffer count (e.g. 1000); exceeding this can trigger device loss.
    - Manages a blit command encoder used for queue-level copies, with `ensureBlitCommandEncoder` and `finalizeBlitCommandEncoder`.
    - Implements `onSubmittedWorkDone` and `onSubmittedWorkScheduled` callbacks with careful handling of idle and device-lost states.
    - Integrates with `Device` for timestamp buffer tracking and encoder association.
- Command encoder:
  - `CommandEncoder.mm`:
    - Validates encoder state before beginning passes (no reuse after commit, command buffer status checks).
    - Uses `Queue` helpers to finalize encoders and ensure a consistent encoder-per-command-buffer mapping.
    - Integrates timestamp writes into pass descriptors when supported (as described in the timestamps plan).
    - Provides `discardCommandBuffer` and `endEncoding` paths that ensure encoders are correctly ended and command buffers are removed from queue tracking.
- Command buffer:
  - `CommandBuffer.mm`:
    - Wraps `MTLCommandBuffer` and ensures proper submission, error propagation, and mapping to WebGPU `GPUCommandBuffer` semantics.

## Deltas vs wgpu (initial)

- Command buffer accounting:
  - WebKit:
    - Tracks created-but-not-committed command buffers in `m_createdNotCommittedBuffers` and enforces a maximum outstanding count.
    - Actively cleans up and invalidates the queue when limits are exceeded or when the device is lost.
  - wgpu:
    - Uses a fixed `MAX_COMMAND_BUFFERS` per `MTLCommandQueue` but does not mirror WebKit’s explicit bookkeeping and device-loss signaling.

- Queue-level blit encoder usage:
  - WebKit:
    - Maintains a shared queue-level blit encoder for copies (`ensureBlitCommandEncoder`, `finalizeBlitCommandEncoder`), and ensures it is properly ended/committed in work-done callbacks.
  - wgpu:
    - Manages encoders within `CommandEncoder` but does not have an equivalent queue-level abstraction; some patterns might be worth borrowing for robustness and efficiency.

- Encoder lifecycle and state checks:
  - WebKit:
    - Validates encoder state before use (e.g., ensures encoders are not used after command buffer commit).
    - Centralizes encoder ending/removal in `CommandEncoder::discardCommandBuffer` and `CommandEncoder::endEncoding`, ensuring queue bookkeeping stays consistent.
  - wgpu:
    - Keeps less detailed state in the Metal command encoder; there may be edge cases where state transitions are less strictly enforced.

## Known issues / open questions

- Outstanding command buffers:
  - Is wgpu’s `MAX_COMMAND_BUFFERS` policy aligned with WebKit’s behavior and Metal best practices?
- Work-done semantics:
  - How closely do wgpu’s fence and callback behaviors match WebKit’s `onSubmittedWorkDone` semantics, especially in edge cases like device loss or failed submissions?
- Encoder lifecycle:
  - Are there scenarios where wgpu’s Metal backend leaves encoders open or mis-manages transitions between render/compute/blit encoders compared to WebKit’s stricter checks?

## Possible improvement directions

- Tighten command buffer accounting:
  - Use WebKit’s patterns for tracking created vs committed command buffers to avoid leaks and ensure timely device loss signaling when limits are exceeded.
- Align work-done semantics:
  - Review wgpu’s queue callbacks/fence behavior against WebKit’s and the spec; adjust where necessary for consistency.
- Strengthen encoder validation:
  - Borrow CommandEncoder state checks from WebKit (e.g. preventing use after commit, invalid states) and reflect them in wgpu’s validation and error messages.

## Next steps when resuming

- Pick a specific lifecycle aspect (e.g. outstanding command buffer limits) and:
  - Map WebKit’s behavior to current wgpu Metal behavior.
  - Propose adjustments with clear rationale and expected impact.
