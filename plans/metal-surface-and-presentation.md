# Plan: Metal surface and presentation

## Context

- Area: CAMetalLayer integration, surface formats, present modes, and frame latency on Metal.
- Relevant files:
  - `wgpu-hal/src/metal/surface.rs`:
    - `Surface`, `SurfaceTexture`, `SurfaceCapabilities`.
    - Integration with `CAMetalLayer`, drawable acquisition, and resize handling.
  - `wgpu-hal/src/metal/mod.rs`:
    - `Queue::present`, `Instance::create_surface`.
  - `wgpu-hal/src/metal/adapter.rs`:
    - `surface_capabilities`.

## Current behavior (wgpu Metal)

- Surface formats:
  - `surface_capabilities` exposes:
    - `Bgra8Unorm`.
    - `Bgra8UnormSrgb`.
    - `Rgba16Float`.
    - Optionally `Rgb10a2Unorm` when supported.
- Present modes:
  - If `can_set_display_sync` is true:
    - Exposes `PresentMode::Fifo` and `PresentMode::Immediate`.
  - Otherwise:
    - Exposes only `PresentMode::Fifo`.
- Maximum frame latency:
  - If `can_set_maximum_drawables_count`:
    - Uses `1..=2`.
  - Otherwise:
    - Uses `2..=2` (effectively triple-buffered default behavior, but constrained).
- Presentation path:
  - `Queue::present`:
    - Creates a new command buffer.
    - If `present_with_transaction` is false:
      - Calls `present_drawable` on the command buffer and commits.
    - If true:
      - Commits, waits until scheduled, then calls `drawable.present()` directly.

## Known issues / open questions

- Format exposure:
  - Are we exposing all the formats WebGPU clients expect on Metal (e.g. HDR surfaces where supported)?
  - Are we too conservative in not exposing additional formats that CAMetalLayer can support?
- Present mode semantics:
  - Mapping `Immediate` to `displaySyncEnabled = false` and/or various CAMetalLayer properties may differ from WebKit’s behavior.
  - Need to ensure alignment with the WebGPU spec’s definition of present modes on macOS/iOS.
- Frame latency:
  - Hard-coded ranges may not match ideal frame pacing on all devices/monitors.
  - WebKit may have more nuanced logic for `maximumDrawableCount` and vsync behavior.

-## WebKit behavior / findings
-
- Key WebKit references:
  - `Source/WebGPU/WebGPU/PresentationContext.h/.mm`:
    - `PresentationContext::create`.
    - `PresentationContext::getPreferredFormat`.
  - `Source/WebGPU/WebGPU/PresentationContextIOSurface.h/.mm`:
    - `PresentationContextIOSurface::configure`.
    - `PresentationContextIOSurface::getCurrentTexture`.
    - `PresentationContextIOSurface::present`.

- Surface abstraction:
  - `PresentationContext` is the abstract surface/swap chain type.
  - For now, `PresentationContext::create` always returns a `PresentationContextIOSurface`:
    - `PresentationContextIOSurface::create(const WGPUSurfaceDescriptor&, const Instance&)`.
  - `getPreferredFormat` returns `WGPUTextureFormat_BGRA8Unorm` unconditionally.
- Allowed formats and configuration:
  - `PresentationContextIOSurface::configure(Device&, const WGPUSwapChainDescriptor&)`:
    - Clears existing render buffers and sets up an “invalid” texture as a fallback.
    - Defines the set of allowed formats:
      - Context texture formats: BGRA8Unorm, RGBA8Unorm, RGBA16Float.
      - View formats: any allowed format after removing an sRGB suffix.
    - Clamps width/height to `device.limits().maxTextureDimension2D`.
    - Chooses `effectiveFormat`:
      - If requested format is allowed: use it.
      - Otherwise: fall back to BGRA8Unorm.
    - Validates:
      - Requested format is in the allowed set.
      - Width/height are non-zero and within `maxTextureDimension2D`.
      - Each `viewFormat` is in the allowed view set.
      - `BGRA8Unorm` + `STORAGE_BINDING` is only allowed if `BGRA8UnormStorage` feature is enabled.
    - Fills a `WGPUTextureDescriptor` and runs `device.errorValidatingTextureCreation` to ensure consistency between WebGPU and Metal.
- Metal texture configuration:
  - Builds a `MTLTextureDescriptor` via `texture2DDescriptorWithPixelFormat:width:height:mipmapped:NO`.
  - Sets `textureDescriptor.usage` based on the WebGPU usage and format (`Texture::usage(descriptor.usage, effectiveFormat)`), then ORs in `MTLTextureUsageRenderTarget | MTLTextureUsageShaderRead`.
  - Storage mode:
    - macOS / Mac Catalyst:
      - If WebGPU is enabled by default and the device has unified memory: `StorageModeShared`.
      - Otherwise: `StorageModeManaged`.
    - Other platforms (iOS family): `StorageModeShared`.
  - Validates that any existing `IOSurface` objects match the configured width/height; otherwise, emits a validation error.
- Presentation path:
  - `PresentationContextIOSurface::present(uint32_t currentIndex)` (not shown in full here) is responsible for enqueuing presentation of the IOSurface-backed texture to the system compositor, using the configured storage and usage.
  - `getCurrentTexture(currentIndex)`:
    - Returns an invalid texture and emits a validation error if:
      - The number of `IOSurface`s doesn’t match the number of render buffers, or
      - `currentIndex` is out of range.
    - Otherwise:
      - If a per-buffer luminance clamp texture exists, recreates it if needed and returns it.
      - Else, recreates the primary texture if needed, resets its “cleared” state, and returns it.
  - `getCurrentTextureView()` is currently `RELEASE_ASSERT_NOT_REACHED` (used via `getCurrentTexture` instead).
- Present modes and frame latency:
  - WebKit’s current `PresentationContext` implementation is IOSurface-centric and doesn’t explicitly expose multiple present modes or `maximumDrawableCount`/`displaySyncEnabled` tuning knobs at this layer.
  - Those details are encoded in how IOSurface-backed textures are integrated with the platform compositor, not directly via CAMetalLayer properties in this code.

## Questions to investigate (later, including WebKit)

- How does WebKit’s WebGPU:
  - Configure `CAMetalLayer` (pixel format, `presentsWithTransaction`, `maximumDrawableCount`, `displaySyncEnabled`)?
  - Map WebGPU present modes to actual Metal/QuartzCore behavior?
  - Handle resizing, HiDPI, and color space / HDR?
- Are there known Apple guidance docs or samples that recommend specific layer/presentation settings for low-latency or power-saving modes?

## Deltas vs wgpu (initial)

- Surface type:
  - WebKit:
    - Uses an IOSurface-based presentation context for WebGPU, with textures created by WebGPU and then bound to the compositor via IOSurfaces.
  - wgpu:
    - Uses `CAMetalLayer` directly (via `Surface` in `surface.rs` and `Instance::create_surface` in `mod.rs`) to obtain `CAMetalDrawable`s.
- Preferred and allowed formats:
  - WebKit:
    - Preferred format: BGRA8Unorm.
    - Allowed swapchain formats: BGRA8Unorm, RGBA8Unorm, RGBA16Float, plus sRGB variants via view formats.
    - Explicitly validates view formats and storage usage (e.g. BGRA8UnormStorage feature).
  - wgpu:
    - Surface capabilities expose BGRA8Unorm, BGRA8UnormSrgb, RGBA16Float, and optionally Rgb10a2Unorm.
    - Format selection and validation are handled differently (via `surface_capabilities` and global feature flags), and the mapping is more tightly coupled to CAMetalLayer.
- Storage modes and usage:
  - WebKit:
    - Chooses `StorageModeShared` vs `Managed` based on platform and whether the device has unified memory and WebGPU is “by default”.
    - Always ensures textures are usable as render targets and shader-readable.
  - wgpu:
    - Texture storage mode selection is driven by `PrivateCapabilities` and usage flags, but doesn’t go through an IOSurface abstraction.
- Present modes / frame latency:
  - WebKit:
    - Presentation behavior is mediated by IOSurface and the compositor; present modes and frame latency controls are not surfaced in this code.
  - wgpu:
    - Explicitly exposes `PresentMode::Fifo` and `PresentMode::Immediate`, and uses `maximum_frame_latency` ranges tied to CAMetalLayer’s `maximumDrawableCount` behavior.

## Possible improvement directions

- Deepen surface format support:
  - Consider exposing additional formats when Metal and CAMetalLayer support them and CTS allows.
  - Ensure sRGB and HDR behavior are correct and spec-compliant.
- Tune present modes and latency:
  - Align `maximum_frame_latency` and present behavior with WebKit’s choices.
  - Provide better defaults for high-refresh-rate and low-power systems.

## Next steps when resuming

- Re-read `surface.rs` (layer setup, drawable acquisition, transaction-based present).
- Once WebKit sources are available:
  - Locate their CAMetalLayer configuration and WebGPU surface code.
  - Map differences in formats, present modes, and frame latency decisions.
