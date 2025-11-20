# Plan: Metal color space and HDR handling

## Context

- Area: color space selection, tone mapping, HDR handling, and their interaction with Metal textures and presentation.
- Relevant wgpu files:
  - `wgpu-hal/src/metal/surface.rs` and related surface/presentation code.
- Relevant WebKit files:
  - `../WebKit/Source/WebGPU/WebGPU/PresentationContextIOSurface.*`.
  - WebCore: GPUCanvas/WebGPU canvas integration code (color space and tone mapping).

## Current behavior (wgpu Metal, conceptually)

- Surface capabilities:
  - Surface formats are limited to a small set (BGRA8Unorm, BGRA8UnormSrgb, RGBA16Float, optionally RGB10A2Unorm).
  - Color space and HDR handling are not deeply modeled beyond choosing formats and present modes.

## WebKit behavior / findings (initial)

- Key WebKit references:
  - `Source/WebGPU/WebGPU/PresentationContextIOSurface.h/.mm`:
    - `PresentationContextIOSurface::configure` (colorSpace, toneMappingMode, compositeAlphaMode).
    - `PresentationContextIOSurface::renderBuffersWereRecreated`.
    - `PresentationContextIOSurface::present`.
  - WebCore canvas integration (for context):
    - `Source/WebCore/Modules/WebGPU/*` files that configure WebGPU canvases and color spaces.

- In `PresentationContextIOSurface.mm`:
  - The swap chain configuration includes:
    - `colorSpace`, `toneMappingMode`, and `compositeAlphaMode` from the `WGPUSwapChainDescriptor`.
  - `configure`:
    - Validates formats and view formats with respect to color space and storage features (e.g. BGRA8UnormStorage).
    - Sets up textures with appropriate formats and usage for HDR (e.g. RGBA16Float) and tone mapping.
    - Optionally uses a “luminance clamp” texture when tone mapping is needed for RGBA16Float.
- Color/HDR integration:
  - WebKit ties together:
    - Swap chain format.
    - Color space.
    - Tone mapping mode.
    - Compositing alpha mode.
  - This is designed to match browser expectations for canvas, CSS color spaces, and HDR displays.

## Known issues / open questions

- Mapping to native apps:
  - Which of WebKit’s color/HDR policies should wgpu adopt or learn from for native applications?
  - How should wgpu expose color space and HDR options in a backend-agnostic way that still maps cleanly to Metal?

## Possible improvement directions

- Expand and clarify surface color/HDR options:
  - Consider surfacing explicit color space and tone mapping choices in wgpu’s surface configuration model.
  - Use WebKit’s validation and format choices as a reference for safe, spec-aligned behavior on Metal.

## Next steps when resuming

- Deep-dive `PresentationContextIOSurface`’s handling of color, HDR formats, and tone mapping.
- Propose a minimal set of color/HDR options for wgpu’s Metal surface that align with WebGPU expectations.
