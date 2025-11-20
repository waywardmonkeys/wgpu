# Plan: Metal shader pipeline and MSL generation

## Context

- Area: WGSL → internal IR → MSL generation, shader validation, specialization, and Metal library/pipeline compilation.
- Relevant wgpu files:
  - `naga` crate (IR and backends), especially the MSL backend.
  - `wgpu-core` shader module handling (WGSL to Naga IR).
  - `wgpu-hal/src/metal/device.rs`:
    - `CompiledShader`.
    - `Device::load_shader` using `ShaderModuleSource::Naga`.
  - `wgpu-hal/src/metal/mod.rs`:
    - `ShaderModuleSource` enum (`Naga` vs `Passthrough`).
- Relevant WebKit files:
  - `../WebKit/Source/WebGPU/WGSL/*`:
    - WGSL parser, IR, validation, and MSL codegen (`WGSLShaderModule.h` and related).
  - `../WebKit/Source/WebGPU/WebGPU/ShaderModule.*`:
    - WGSL checking, preparation, generation to MSL, and Metal library creation.
  - `../WebKit/Source/WebGPU/WebGPU/RenderPipeline.*`, `ComputePipeline.*`:
    - Usage of shader reflection for pipeline creation.

## Current behavior (wgpu Metal)

- WGSL front end:
  - WGSL is parsed and validated by Naga (shared across backends).
  - Naga produces a backend-agnostic IR that is then lowered to MSL, SPIR-V, etc.
- MSL generation for Metal:
  - `Device::load_shader`:
    - Calls `naga::back::pipeline_constants::process_overrides` to resolve `override` constants per entry point using supplied `stage.constants`.
    - Computes bounds check policies based on `stage.module.bounds_checks`.
    - Configures `naga::back::msl::Options`:
      - Sets `lang_version` from `PrivateCapabilities.msl_version`.
      - Supplies `per_entry_point_map` using `PipelineLayout`’s `per_stage_map[NagaStage]` for binding/resource layout.
      - Configures bounds-check policies (index, buffer, image) and disables binding-array checks for now.
      - Propagates `zero_initialize_workgroup_memory` and `force_loop_bounding`.
    - Sets `naga::back::msl::PipelineOptions`:
      - Entry point and stage.
      - `allow_and_force_point_size` when primitive topology is points.
      - Enables vertex-pulling transform with per-vertex buffer mappings.
    - Calls `naga::back::msl::write_string` to generate MSL source and back-end info.
  - Metal compilation:
    - Builds `MTLCompileOptions`, setting:
      - `language_version` to `PrivateCapabilities.msl_version`.
      - `preserve_invariance` when supported (`supports_preserve_invariance`).
    - Calls `new_library_with_source` to compile the MSL into a `MTLLibrary`.
    - Looks up the translated entry point name from Naga’s `entry_point_names` and retrieves a `MTLFunction`.
  - Reflection into `CompiledShader`:
    - Uses Naga IR and module info to:
      - Collect workgroup sizes for compute/task/mesh.
      - Track which `WorkGroup` variables exist and their sizes.
      - Build `sized_bindings` for runtime-sized arrays and vertex buffer sizes.
      - Compute an `immutable_buffer_mask` for read-only buffers.
- Error handling:
  - Shader compilation failures are reported as `PipelineError::PipelineConstants` (override processing) or `PipelineError::Linkage` (MSL generation or Metal compilation), with logs including generated MSL.

## WebKit behavior / findings (initial)

- Key WebKit references:
  - `Source/WebGPU/WGSL/WGSLShaderModule.h` and related WGSL sources:
    - `WGSLShaderModule` (parsing, type checking, call graph).
    - `WGSL::prepare` (layout-aware preparation and reflection).
    - `WGSL::generate` (WGSL → MSL generation).
  - `Source/WebGPU/WebGPU/ShaderModule.h/.mm`:
    - `Device::createShaderModule`.
    - `ShaderModule::createLibrary`.
    - `earlyCompileShaderModule`.
    - `ShaderModule` constructor and reflection helpers (fragment/vertex I/O, usage flags).
  - `Source/WebGPU/WebGPU/RenderPipeline.mm` / `ComputePipeline.mm`:
    - Use of shader reflection when building Metal pipeline state.

- WGSL path:
  - Front end:
    - `WGSLShaderModule` in `Source/WebGPU/WGSL` handles parsing, type checking, and call graph construction for WGSL.
    - Builds internal WGSL IR and tracks usage of various builtins and language features (e.g. workgroup uniform loads, external textures, math operations).
  - Generation:
    - `ShaderModule.mm`:
      - Extracts WGSL code and compilation hints from `WGPUShaderModuleDescriptor`.
      - Converts WebGPU `PipelineLayout` hints into WGSL `PipelineLayout` hints for early compilation.
      - Calls `WGSL::prepare` with pipeline layout hints to perform layout-aware preparation and reflection.
      - Calls `WGSL::generate` to produce MSL source, providing:
        - `WGSL::DeviceState` with `appleGPUFamily` and `shaderValidationEnabled`.
      - Collects reflection info (`entryPoints`, `entryPointInformation`) for later use by pipelines.
  - Metal compilation:
    - `ShaderModule::createLibrary`:
      - Configures `MTLCompileOptions` with:
        - Preprocessor macro `__wgslMetalAppleGPUFamily` based on `DeviceState.appleGPUFamily`.
        - Math mode and floating-point function settings:
          - Controlled by user defaults like `WebKitWebGPUEnableSafeMathMode` and `WebKitWebGPUEnablePrecisionMathFunctions`.
        - `preserveInvariance` when requested by `DeviceState`.
      - Calls `newLibraryWithSource` and wraps errors with a WebGPU-specific NSError domain and a message including the MSL source.
    - Stores the resulting `MTLLibrary` per `ShaderModule`, with per-entry-point reflection state (fragment outputs, inputs, usage flags).

## Known issues / open questions

- Mapping differences:
  - How closely does Naga’s MSL backend match WebKit’s WGSL→MSL generator in:
    - Builtin mapping (e.g. `@builtin(position)`, `@builtin(sample_index)`).
    - Varying interpolation and location packing.
    - Robustness (bounds checks, zero-initialization, loop bounding).
  - Are there known MSL or Metal quirks that WebKit handles explicitly (e.g. family-specific workarounds) that wgpu/Naga does not?
- Specialization and pipeline layout:
  - How does WebKit use pipeline layout hints and reflection to:
    - Optimize bindings and push constants.
    - Avoid recompilation per pipeline when layouts change?
  - Could wgpu benefit from a similar “pipeline layout hint” system to improve pipeline creation latency?
- Math mode and precision:
  - WebKit exposes toggles for safe vs relaxed math and precise vs fast math functions.
  - WebKit’s implementation uses newer `MTLCompileOptions` fields (`mathMode`, `mathFloatingPointFunctions`) where available, and only falls back to `fastMathEnabled` on older SDKs.
  - wgpu currently relies on Naga’s choices and Metal defaults, and through the `metal` crate only has access to the older `fastMathEnabled` knob, not the newer math mode APIs.
- Cross-backend portability:
  - WebKit’s WGSL pipeline is Metal-only in this code; wgpu must support multiple backends with a single IR. How much of WebKit’s behavior makes sense to mirror vs learn from conceptually?

## Possible improvement directions

- Investigate alignment between Naga MSL and WebKit WGSL→MSL:
  - Identify specific areas where semantics differ or where Metal shaders behave differently (e.g. derivatives, interpolation, special builtins).
  - Cross-check against WebGPU CTS failures on Metal to see if differences explain known bugs.
- Consider pipeline layout hints:
  - Explore a mechanism for passing pipeline layout information into Naga’s MSL backend earlier, similar to WebKit’s use of WGSL pipeline layout hints, to reduce specialization overhead or improve error messages.
- Math and invariance settings:
  - Short term (with current `metal` crate):
    - Add a backend-local configuration (e.g., env var or device feature flag) that controls `CompileOptions::set_fast_math_enabled`, allowing opt-in “safer math” for debugging and correctness-sensitive workloads.
    - Clearly treat this as a transitional knob, since `fastMathEnabled` is deprecated in favor of finer-grained math controls on newer SDKs.
  - Longer term (with objc2-metal or an updated Metal binding):
    - Design a wgpu-level abstraction for shader math behavior (e.g. a small enum or flags for “default”, “safe/precise”, “fast”) that:
      - Maps to `mathMode` / `mathFloatingPointFunctions` on modern Metal.
      - Falls back to `fastMathEnabled` on older Metal toolchains.
    - Avoid baking the deprecated `fastMathEnabled` concept directly into wgpu’s public API; instead, hide it behind this abstraction so migration to objc2-metal is straightforward.
- Error reporting:
  - Improve error messages from Naga/Metal compilation by borrowing patterns from WebKit (e.g. including generated MSL in the error in a controlled way and mapping errors back to WGSL locations where possible).

## Next steps when resuming

- Deep-dive Naga’s MSL backend and compare:
  - Builtins, interpolation, robustness, and layout mapping against WebKit’s WGSL→MSL.
  - Family-specific behavior that WebKit might encode via `appleGPUFamily` macros or `DeviceState`.
- Identify at least one concrete, tractable change:
  - For example, improved error reporting or a small math-mode/invariance tweak, and prototype it in wgpu’s Metal backend.
