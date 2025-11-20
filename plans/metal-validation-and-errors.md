# Plan: Metal/WebGPU validation and error reporting

## Context

- Area: validation of descriptors and commands, and mapping of validation errors to user-facing error messages.
- Relevant wgpu files:
  - `wgpu-core` validation layers:
    - Device/buffer/texture/bind group/pipeline descriptor validation.
    - Command buffer recording validation.
  - `wgpu-hal/src/metal/*`:
    - Backend-specific checks (e.g. format support, timestamp support).
- Relevant WebKit files:
  - `../WebKit/Source/WebGPU/WebGPU/Buffer.*`, `Texture.*`, `BindGroup.*`, `BindGroupLayout.*`, `PipelineLayout.*`, `RenderPipeline.*`, `ComputePipeline.*`, `Queue.*`, `CommandEncoder.*`.
  - WebCore layer: `../WebKit/Source/WebCore/Modules/WebGPU/*.cpp` (JS-level error handling).

## Current behavior (wgpu Metal, conceptually)

- Validation is largely centralized in `wgpu-core`, with Metal backend contributing:
  - Capability checks (e.g. texture format support, timestamp query support).
  - Some platform-specific restrictions (e.g. surface formats, present modes).
- Error reporting:
  - Errors flow through Rust `Result` types and enums (`DeviceError`, `SurfaceError`, `PipelineError`).
  - Messages tend to be concise and not always directly mapped back to WebGPU spec phrasing.

## WebKit behavior / findings (initial)

- Key WebKit references:
  - `Source/WebGPU/WebGPU/Buffer.mm`:
    - `validateCreateBuffer`.
  - `Source/WebGPU/WebGPU/Texture.mm`:
    - Texture descriptor validation and `Device::createTexture`.
  - `Source/WebGPU/WebGPU/BindGroupLayout.mm`:
    - `BindGroupLayout::errorValidatingDynamicOffsets`.
  - `Source/WebGPU/WebGPU/PipelineLayout.mm`:
    - `Device::createPipelineLayout`.
    - `PipelineLayout::errorValidatingBindGroupCompatibility`.
  - `Source/WebGPU/WebGPU/RenderPassEncoder.mm`:
    - `RenderPassEncoder::setBindGroup`.
    - `RenderPassEncoder::setIndexBuffer`.
  - `Source/WebGPU/WebGPU/ComputePassEncoder.mm`:
    - `ComputePassEncoder::dispatch`.
    - `ComputePassEncoder::dispatchIndirect`.

- Spec-driven validation:
  - WebKit’s C++/Obj-C layer closely follows the WebGPU spec’s “abstract operations” in code:
    - `Buffer.mm` and `validateCreateBuffer` implement “Validating GPUBufferDescriptor”.
    - `Texture.mm` mirrors “Validating GPUTextureDescriptor” and related rules (format, size, usage vs limits).
    - `BindGroup.mm` and `BindGroupLayout.mm` validate:
      - Bind group entries vs layout (expected vs missing buffers/samplers/textures).
      - Dynamic buffer presence and sizes for bindings with dynamic offsets.
      - External texture constraints (format, dimension, mip count, sample count).
    - `PipelineLayout.mm`:
      - Enforces per-stage resource limits across all bind groups (uniform/storage buffers, samplers, sampled and storage textures).
      - Accumulates counts per stage and compares them to `deviceLimits.max*PerShaderStage` values, emitting detailed messages like:
        - `"Resource usage limits exceeded: uniformBufferCount(%u) > deviceLimits.maxUniformBuffersPerShaderStage(%u) || storageBufferCount(%u) > deviceLimits.maxStorageBuffersPerShaderStage(%u) || …"`.
      - Checks device consistency and bind group layout validity before creating the pipeline layout.
  - Many validation errors include exact conditions and numeric values (e.g. buffer sizes, offsets, alignments, limit values, dynamic offset counts).
- Error surfacing:
  - Errors are reported via `generateAValidationError` or `VALIDATION_ERROR(...)` macros, constructing `NSString` messages.
  - At the WebCore layer, these are wrapped into `GPUValidationError` or `GPUDeviceLost` events with spec-like messages.
  - Some errors intentionally reveal detailed numeric context to aid debugging.
- Differences in behavior vs wgpu:
  - WebKit often checks more conditions directly at the WebGPU API layer (e.g. device mismatch for bind groups/pipeline layouts, dynamic offset count and alignment) and ties them to specific API calls in messages (e.g. “GPURenderPassEncoder.setBindGroup: bind group is nil” or “dynamicOffsetCount(...) in setBindGroupCall does not equal the dynamicBufferCount(...) in bind group layout”).
  - Dynamic offset validation is particularly detailed:
    - `BindGroupLayout::errorValidatingDynamicOffsets`:
      - Verifies `dynamicOffsets.size() == dynamicBufferCount()`.
      - Ensures each dynamic buffer exists, and `dynamicOffset + bindingSize <= bufferSize` without overflow.
      - Checks `dynamicOffset` alignment separately for uniform vs storage buffers (using `minUniformBufferOffsetAlignment` / `minStorageBufferOffsetAlignment`).
    - Errors precisely name the offending index and values.
  - Bind group usage and compatibility:
    - On `RenderPassEncoder::setBindGroup`:
      - Validates that the bind group is valid and device-compatible (`isValidToUseWith`).
      - Ensures a `BindGroupLayout` exists; otherwise errors with `"GPURenderPassEncoder.setBindGroup: bind group is nil"`.
      - Delegates dynamic offset checks to `BindGroupLayout::errorValidatingDynamicOffsets` and prefixes any error with `"GPURenderPassEncoder.setBindGroup: …"`.
    - Before dispatch/draw in `ComputePassEncoder` / `RenderPassEncoder`:
      - Uses `PipelineLayout::errorValidatingBindGroupCompatibility` to ensure:
        - The number of set bind groups matches the pipeline’s expectation.
        - Each pipeline bind group index has a corresponding set group with a compatible layout.
      - Reports mismatches with messages like:
        - `"number of bind groups set(%u) is less than the pipeline uses(%zu)"`.
        - `"can not find bind group in pipeline for bindGroup index %zu"`.
  - Resource usage tracking:
    - `RenderPassEncoder` and `ComputePassEncoder` accumulate resource usage per buffer/texture within a pass.
    - If a resource is used with an incompatible combination of usages, they invalidate the encoder with messages such as:
      - `"Bind group has incompatible usage list: <usage bitmap as string>"`.

  - Dispatch and indirect dispatch:
    - `ComputePassEncoder::dispatch`:
      - Validates the threadgroup counts `x`, `y`, and `z` against `limits().maxComputeWorkgroupsPerDimension`.
      - Emits an error string that shows each dimension vs the limit, e.g.:
        - `"x(%u) > dimensionMax(%u) || y(%u) > dimensionMax(%u) || z(%u) > dimensionMax(%u)"`.
    - `ComputePassEncoder::dispatchIndirect`:
      - Validates the indirect buffer and offset by checking:
        - `indirectOffset` is a multiple of 4.
        - `indirectBuffer.usage()` includes `WGPUBufferUsage_Indirect`.
        - `indirectOffset + 3 * sizeof(uint32_t)` does not overflow and is ≤ `indirectBuffer.initialSize()`.
      - On failure, reports a detailed error including:
        - The offset, usage bits, overflow status, computed sum, and buffer size.
      - Uses a small internal Metal compute shader (`csDispatchClamp`) to clamp indirect arguments to the legal `maxComputeWorkgroupsPerDimension` range before issuing the actual indirect dispatch, ensuring robustness even if the buffer contents are out of range.

  - Index buffer and draw validation:
    - `RenderPassEncoder::setIndexBuffer`:
      - Checks:
        - Buffer/device compatibility (`isValidToUseWith`).
        - That the buffer usage includes `WGPUBufferUsage_Index`.
        - That `offset` is correctly aligned (2 or 4 bytes depending on index format).
        - That `offset + size` is in range and does not overflow `buffer.initialSize()`.
      - Invalidates the encoder with clear messages if any of these conditions fail.

## Areas where WebKit exceeds wgpu today (improvement targets)

- Dynamic offsets:
  - WebKit:
    - Validates dynamic offsets in `BindGroupLayout::errorValidatingDynamicOffsets` with:
      - Exact count matching (`dynamicOffsets.size() == dynamicBufferCount()`).
      - Per-buffer existence checks.
      - Overflow-safe `dynamicOffset + bindingSize <= bufferSize` checks.
      - Separate alignment rules for uniform vs storage buffers.
    - Emits precise messages with indices and numeric values.
  - wgpu:
    - Performs dynamic offset count/alignment checks in `wgpu-core`, but:
      - Messages are shorter and less specific.
      - Alignment/bounds checks and `minBindingSize` handling should be re-audited against WebKit/spec for Metal.

- Bind group vs pipeline layout compatibility:
  - WebKit:
    - Uses `PipelineLayout::errorValidatingBindGroupCompatibility` to ensure:
      - The number of set bind groups matches the pipeline’s bind group count.
      - Each bind group index has a layout-compatible group.
    - Provides clear failure messages with counts and indices.
    - Caches validation per `(bindGroupIndex, pipelineId, maxDynamicOffset)` to avoid redundant work, and revalidates when dynamic offsets change.
  - wgpu:
    - Validates compatibility conceptually in `wgpu-core`, but:
      - Semantics and error detail are less explicit.
      - There is an opportunity to mirror WebKit’s structured compatibility checks and caching model.

- Per-stage resource limits:
  - WebKit:
    - At pipeline layout creation, aggregates:
      - Uniform/storage buffers, samplers, sampled textures, storage textures per stage across all bind groups.
    - Compares totals against `max*PerShaderStage` limits, with detailed messages listing each count vs limit.
  - wgpu:
    - Enforces per-stage limits, but:
      - Aggregation logic and error reporting are less transparent.
      - Using WebKit’s pattern as a reference would help ensure consistency and better diagnostics.

- Resource usage tracking within passes:
  - WebKit:
    - Tracks buffer/texture usage across a pass in `RenderPassEncoder` / `ComputePassEncoder`.
    - Detects incompatible usage combinations and reports them with a named usage set (`BindGroup::usageName`).
  - wgpu:
    - Has usage tracking in pass validation, but:
      - The exact conflict rules and messages are less user-facing.
      - Aligning with WebKit’s usage model could clarify edge cases and improve error messages.

- Dispatch and indirect dispatch validation:
  - WebKit:
    - Validates direct dispatch dimensions against device limits.
    - Uses explicit, overflow-safe checks for indirect-dispatch arguments, and sanitizes them via a dedicated compute shader before dispatch.
  - wgpu:
    - Enforces WebGPU limits and some indirect buffer checks, but:
      - The Metal-specific robustness measures (like clamping via a helper shader) and error messages are less comprehensive.
      - There is room to align behavior and diagnostics with WebKit, especially on Metal.

- Error messaging style:
  - WebKit:
    - Names the API in error strings (`GPURenderPassEncoder.setBindGroup`, `GPUComputePassEncoder.dispatchIndirect`).
    - Prints actual vs limit values in many errors.
    - Uses wording close to the spec’s abstract operations.
  - wgpu:
    - Error messages are generally shorter and less descriptive.
    - There is room to standardize and enrich messages, especially for validation failures that users commonly hit (buffers, textures, bind groups, dispatch/draw).

## Possible improvement directions

- Tighten validation semantics (wgpu-core, guided by WebKit):
  - Revisit dynamic offset validation:
    - Ensure counts, bounds, and alignment checks match spec and WebKit behavior.
    - Make sure `minBindingSize` / binding size rules are consistent.
  - Make bind group vs pipeline layout compatibility checks more explicit:
    - Introduce structured compatibility errors (missing group, layout mismatch, extra group).
    - Consider caching successful validations per pipeline/layout/dynamic-offset combination.
  - Align per-stage resource limit aggregation with WebKit’s model where appropriate.
  - Audit dispatch and indirect-dispatch validation:
    - Ensure dimension and indirect-argument checks match spec and WebKit’s Metal behavior.
    - Decide whether host-side checks are sufficient or if a small Metal helper shader (similar to WebKit’s `csDispatchClamp`) is warranted for extra robustness.

- Improve error messages:
  - Include:
    - The high-level operation or API (e.g., “render pass set_bind_group”).
    - Actual vs limit values for limit-related failures.
    - Indices and sizes for buffer/texture and dynamic-offset errors.
  - Use spec terminology where it helps developers cross-reference the spec.

- Capture Metal-specific quirks in validation:
  - Where WebKit has extra checks motivated by Metal behavior (e.g. external textures, certain format constraints), treat them as hints for Metal-specific validation in wgpu’s core/backend boundary rather than leaving them to runtime failures.

## Next steps when resuming

- Choose one validation area (e.g. dynamic offsets or bind group layout compatibility) and:
  - Map WebKit’s checks to equivalent code in `wgpu-core`.
  - Identify missing or diverging behaviors.
  - Propose concrete changes to wgpu’s validation and messaging.
