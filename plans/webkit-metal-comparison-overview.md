# Plan: WebKit vs wgpu Metal comparison (overview)

## Context

- Goal: systematically compare wgpu’s Metal backend with WebKit’s WebGPU Metal implementation, and identify concrete improvements for wgpu.
- This plan is a top-level index that ties together the more focused Metal plans:
  - `metal-timestamps-and-queries.md`
  - `metal-capabilities-and-limits.md`
  - `metal-memory-and-resources.md`
  - `metal-pipeline-layout-and-argument-buffers.md`
  - `metal-surface-and-presentation.md`

## Current situation

- wgpu’s Metal backend:
  - Mature, but initially designed before Apple’s own WebGPU implementation.
  - Implements many workarounds for driver quirks (timestamps, blit encoders, etc.).
  - Uses older feature-detection patterns (heavy `MTLFeatureSet` usage) and conservative capabilities.
- WebKit’s WebGPU Metal:
  - Source now available under:
    - `../WebKit/Source/WebGPU/WebGPU` (core C++/Obj-C Metal implementation).
    - `../WebKit/Source/WebCore/Modules/WebGPU` (JS-facing layer, limits exposure).
  - Reflects Apple’s current mapping of WebGPU concepts onto Metal:
    - Uses `HardwareCapabilities` to derive limits and features per `MTLGPUFamily`.
    - Implements timestamp queries via `MTLCounterSet` and encoder sample buffers, with Intel-specific disabling.
    - Manages swapchain-like presentation via `PresentationContextIOSurface` rather than direct `CAMetalLayer` use.

## Comparison axes

- Timestamps and queries:
  - Calibration vs constants, sampling points, dummy encoders.
- Capabilities and limits:
  - Texture formats, read-write tiers, float filtering, MSAA, downlevel limits.
- Memory and resources:
  - Storage modes, heaps, memoryless textures, hazard tracking.
- Pipeline layout and argument buffers:
  - Bind group/binding mapping, push constants, runtime array handling, argument buffer usage.
- Surface and presentation:
  - CAMetalLayer configuration, present modes, maximum drawable count, HDR/color space.

## Plan for future work

- Phase 1: Discovery
  - Already done at a high level:
    - Identified WebKit’s WebGPU directories and main entry points for device, queue, surface, pipelines, and resources.
- Phase 2: Per-area comparison
  - For each focused plan:
    - We have started extracting WebKit behavior and deltas for:
      - `metal-timestamps-and-queries.md`
      - `metal-capabilities-and-limits.md`
      - `metal-memory-and-resources.md`
      - `metal-pipeline-layout-and-argument-buffers.md`
      - `metal-surface-and-presentation.md`
    - Remaining work is to refine these summaries and tie them to concrete change proposals in wgpu.
- Phase 3: Prioritization
  - Rank potential changes for wgpu based on:
    - Correctness (spec compliance, bug fixes).
    - Performance (CPU/GPU overhead, memory usage).
    - Implementation complexity and risk.
- Phase 4: Implementation
  - For high-priority items, draft concrete patches to wgpu’s Metal backend:
    - Keep changes minimal and focused per area.
    - Add or update tests where possible.
  - Validate against CTS and representative workloads.

## Next steps when resuming

- For each focused plan:
  - Refine the newly added “WebKit behavior / findings” and “Deltas vs wgpu (initial)” sections into concrete, implementable wgpu changes.
  - Prioritize which deltas to address first (timestamps, limits, memory, pipeline layout, or presentation), and start designing patches.
