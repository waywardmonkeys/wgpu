# Plan: Metal capabilities and limits

## Context

- Area: feature detection, limits, and format capabilities for Metal.
- Relevant files:
  - `wgpu-hal/src/metal/adapter.rs`:
    - `PrivateCapabilities` struct and `PrivateCapabilities::new`.
    - `texture_format_capabilities`.
    - `capabilities()` and `downlevel_capabilities()`.
    - `map_format`.
  - `wgpu-hal/src/metal/mod.rs`:
    - `PrivateCapabilities` usage within `AdapterShared`.
- Current design themes:
  - Uses a mix of `MTLFeatureSet` and `MTLGPUFamily` checks (with a `family_check` switch).
  - Maintains a large set of booleans for format and feature flags (ASTC, BC, EAC/ETC, depth formats, float filtering, etc.).
  - Derives wgpu `TextureFormatCapabilities`, `Capabilities`, and downlevel feature sets from Metal’s feature sets and families.

## Current behavior (wgpu Metal)

- `PrivateCapabilities::new`:
  - Infers OS and device family support using:
    - OS version helpers (`version.at_least`).
    - `device.supports_family(MTLGPUFamily::...)`.
    - `device.supports_feature_set(MTLFeatureSet::...)`.
  - Sets flags such as:
    - `resource_heaps`, `argument_buffers`, `shared_textures`.
    - `texture_cube_array`, `depth_clip_mode`, `supports_float_filtering`.
    - Fine-grained format flags (e.g. `format_rgba8_srgb_all`, `format_r32float_all`, `format_astc_hdr`).
- `texture_format_capabilities`:
  - Maps `wgt::TextureFormat` to a bitmask (`TextureFormatCapabilities`) using:
    - Base “all” capabilities (sampled, color attachment, resolve).
    - Tiered read-write texture support (`MTLReadWriteTextureTier`).
    - Conditional flags controlled by `PrivateCapabilities`.
  - This function is large and intricate; it encodes a de facto “policy table” for Metal texture formats.
- `capabilities()` and `downlevel_capabilities()`:
  - Build wgpu’s `Capabilities` and `DownlevelCapabilities` structs from `PrivateCapabilities`.
  - Control limits like max threads per group, max texture size, multiview, MSAA support, etc.

## Known issues / open questions

- `MTLFeatureSet` deprecation:
  - Comments already note `MTLFeatureSet` is superseded by `MTLGpuFamily`, but many checks still rely on feature sets.
  - Long term, this may break or become misleading as new OS versions and GPU families appear.
- Format coverage:
  - The format capability matrix is hand-maintained and may be conservative or stale.
  - Potential mismatches with:
    - What current Metal drivers actually support.
    - What Apple’s WebGPU implementation exposes.
  - Compression formats (BC, ASTC, EAC/ETC) and HDR capabilities may differ across families more than wgpu’s current flags capture.
- Limits and downlevel flags:
  - Some limits (e.g. `max_threads_per_group`, `max_texture_layers`) are inferred from device properties; others are hard-coded or clamped heuristically.
  - Need to verify alignment with WebGPU spec minimums and typical WebKit values.

## WebKit behavior / findings

- Key WebKit references:
  - `Source/WebGPU/WebGPU/HardwareCapabilities.h`:
    - `struct HardwareCapabilities`, `BaseCapabilities`.
  - `Source/WebGPU/WebGPU/HardwareCapabilities.mm`:
    - `baseCapabilities`.
    - `baseFeatures`.
    - Per-family functions: `apple4`, `apple5`, `apple6`, `apple7`, `mac2`.
    - `mergeLimits`, `mergeFeatures`, `mergeBaseCapabilities`.
    - `rawHardwareCapabilities`, `hardwareCapabilities`.
    - `defaultLimits`, `anyLimitIsBetterThan`.

- Overall structure:
  - `HardwareCapabilities` holds:
    - A `WGPULimits` struct (`limits`).
    - A sorted `Vector<WGPUFeatureName>` (`features`).
    - A `BaseCapabilities` struct with:
      - `argumentBuffersTier`.
      - `timestampCounterSet` / `statisticCounterSet`.
      - `memoryBarrierLimit`.
      - `supportsNonPrivateDepthStencilTextures`.
      - `canPresentRGB10A2PixelFormats`.
      - `supportsResidencySets`.
  - `hardwareCapabilities(id<MTLDevice>)`:
    - Computes raw capabilities from Metal GPU families.
    - Applies spec-driven merging rules across multiple supported families.
    - Validates the resulting limits against WebGPU’s default limits.
- Per-family capability tables:
  - Functions like `apple4`, `apple5`, `apple6`, `apple7`, `mac2` build a `HardwareCapabilities` for each `MTLGPUFamily`:
    - Start from `baseCapabilities(device)`:
      - Fills argument buffer tier and counter sets.
      - Leaves some fields to be filled by the caller (e.g. `supportsNonPrivateDepthStencilTextures`).
    - Customize `baseCapabilities` flags:
      - e.g. `supportsNonPrivateDepthStencilTextures = true` on some families.
      - Configure `memoryBarrierLimit` based on `isShaderValidationEnabled(device)`.
    - Build a feature list via `baseFeatures(device, baseCapabilities)` then append family-specific features:
      - e.g. `TextureCompressionETC2`, `TextureCompressionASTC`, `TextureCompressionASTCSliced3D`, `TextureFormatsTier1`, `Float32Filterable`, etc.
    - Construct `limits`:
      - Use a mix of constants and helper functions (`maxBufferSize(device)`, `multipleOf4`, etc.).
      - Some values are placeholders (`maxUniformBufferBindingSize = 0`) to be filled later once all families are merged.
    - Sort the feature vector.
- Merging across GPU families:
  - `rawHardwareCapabilities(device)`:
    - Skips virtual hardware (VMs) unless explicitly allowed (`isPhysicalHardware` guard).
    - For each supported `MTLGPUFamily` (Apple4–7, Mac2), calls the corresponding function and merges into a single `HardwareCapabilities`:
      - `mergeLimits`:
        - Uses spec classes:
          - “maximum” limits: `max(...)` (e.g. texture dimensions, workgroup sizes, buffer sizes).
          - “alignment” limits: `min(roundUpToPowerOfTwo(...))` for min-alignments.
        - Applies a CTS workaround (`workaroundCTSBindGroupLimit`) to clamp some high limits down to 1000 (bindgroup-related counts).
      - `mergeFeatures`:
        - Merges two sorted feature vectors, deduplicated (like a set union).
      - `mergeBaseCapabilities`:
        - Asserts that `argumentBuffersTier` and `timestampCounterSet`/`statisticCounterSet` are consistent.
        - Combines booleans with OR, and clamps `memoryBarrierLimit` to the smaller value.
    - After merging:
      - Sets `limits.maxUniformBufferBindingSize` and `limits.maxStorageBufferBindingSize` to `maxBufferSize(device)`.
  - `hardwareCapabilities(device)`:
    - Calls `rawHardwareCapabilities`.
    - Validates that no limit is “better” than WebGPU’s default limits (`anyLimitIsBetterThan(defaultLimits(), result->limits)`):
      - If any implementation limit exceeds default limits in a way that violates the spec’s default behavior, returns `std::nullopt`.
- Default limits and WebGPU integration:
  - `defaultLimits()` returns WebGPU’s spec default limits for Metal, including:
    - `maxTextureDimension2D = 8192`, `maxTextureArrayLayers = 256`, etc.
    - `maxBufferSize = defaultMaxBufferSize` (derived from Metal device properties).
    - Bind group and per-stage binding counts aligned with the spec.
  - WebCore layer (`GPUSupportedLimits.*`):
    - Exposes whatever `HardwareCapabilities.limits` contains directly to JS via `GPUSupportedLimits` getters.
    - No extra logic there; all policy is in `HardwareCapabilities`.

## Questions to investigate (later, including WebKit)

- Which `MTLGPUFamily`+OS combinations correspond to which WebGPU feature sets in WebKit?
- How does WebKit:
  - Decide which texture formats are supported and in what roles (sampled, storage, color attachment, depth/stencil)?
  - Handle BC, ASTC, and ETC formats on Apple Silicon and Intel macOS?
  - Map Metal limits to WebGPU limits (e.g. workgroup sizes, buffer sizes, attachment counts)?
- Does WebKit rely on newer APIs (e.g. `supports32BitFloatFiltering`, `supportsFamily`) instead of `MTLFeatureSet`?

## Deltas vs wgpu (initial)

- Feature detection:
  - WebKit:
    - Uses `MTLGPUFamily`-centric capability functions (`apple4`, `apple5`, `apple6`, `apple7`, `mac2`) and `baseCapabilities`/`baseFeatures`.
    - Relies on modern APIs like `supportsFamily`, `supportsBCTextureCompression`, `supports32BitFloatFiltering`, etc.
  - wgpu:
    - Mixes `MTLFeatureSet` and `MTLGPUFamily` checks, with a `family_check` flag controlling which path to use.
    - Maintains many per-format/per-feature booleans in `PrivateCapabilities` instead of a single `BaseCapabilities`-like struct.
- Limits policy:
  - WebKit:
    - Encodes WebGPU’s limit classes explicitly:
      - “maximum” limits merged via `max`.
      - “alignment” limits merged via `min(roundUpToPowerOfTwo(...))`.
    - Applies a CTS-specific clamp (`workaroundCTSBindGroupLimit`) to avoid excessively high per-stage binding counts.
    - After merging families, sets buffer binding size limits to the device’s maximum buffer size.
    - Validates that implementation limits are not “better than” the spec’s default limits in a way that would break the default behavior, and may disable WebGPU on out-of-bounds hardware.
  - wgpu:
    - Derives limits in `PrivateCapabilities` and `capabilities()` with a mix of:
      - Hard-coded constants.
      - Direct reads from Metal device properties.
      - Some clamping to WebGPU minima (e.g. via `.min()` with spec values).
    - Does not currently implement the same explicit “limit class” merging model across multiple GPU families.
- Texture formats and compression:
  - WebKit:
    - Uses features like `TextureCompressionBC`, `TextureCompressionETC2`, `TextureCompressionASTC`, `TextureCompressionASTCSliced3D`, and `TextureFormatsTier1` to describe format capabilities at the WebGPU level.
    - The mapping from these features to actual Metal formats is encapsulated in the WebGPU C++ layer (and likely the shader/texture implementation), but the feature exposure is clearly per-family.
  - wgpu:
    - Uses `PrivateCapabilities` boolean fields to directly encode per-format support (e.g. `format_bc`, `format_eac_etc`, `format_astc`, `format_astc_hdr`, `format_astc_3d`).
    - The `texture_format_capabilities` table is hand-maintained, with behavior inferred from a combination of OS/family checks and these booleans.
- Base capabilities:
  - WebKit:
    - Keeps `BaseCapabilities` focused on low-level Metal facts (argument buffer tier, counter sets, residency, presentation capabilities).
  - wgpu:
    - Stores more detailed format and feature flags directly in `PrivateCapabilities`, somewhat conflating base hardware facts with WebGPU policy decisions.

## Possible improvement directions

- Modernize feature detection:
  - Prefer `MTLGPUFamily` and newer API properties where available.
  - Treat `MTLFeatureSet` usage as legacy, used only when absolutely necessary for older OS targets.
  - Potentially split capabilities by GPU family and OS version in a more data-driven way.
- Audit and update format capabilities:
  - Cross-check `texture_format_capabilities` against:
    - Apple’s documentation for pixel formats and storage tiers.
    - WebKit’s WebGPU format exposure.
  - Identify formats where we may:
    - Be overly conservative (missing storage, attachment, or MSAA support).
    - Be overly optimistic (claiming capabilities that Metal does not guarantee).
- Improve test coverage:
  - Add targeted tests (or CTS focused runs) to validate:
    - ASTC/BC/EAC/ETC availability on different devices.
    - Depth/stencil format behavior, especially `Depth24Plus` vs `Depth32Float`.
  - Use runtime assertions or debugging tools to detect mismatches early (e.g. failed attachment creation due to unsupported formats).

## Next steps when resuming

- Re-read `PrivateCapabilities::new` in full to map:
  - How each OS and family combination sets the capability booleans.
  - How these booleans feed into `TextureFormatCapabilities` and `Capabilities`.
- Once WebKit sources are available:
  - Find WebKit’s Metal-specific capabilities/limits computation.
  - Extract their format and limit policy:
    - Per-OS, per-GPU family if applicable.
  - Compare with wgpu’s `PrivateCapabilities` and `texture_format_capabilities`, and enumerate concrete deltas.
