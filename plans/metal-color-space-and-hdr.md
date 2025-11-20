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

## Deltas vs wgpu (initial)

- Color space modeling:
  - WebKit:
    - Treats color space as a first-class part of the swap chain descriptor (`colorSpace`).
    - Validates formats and view formats against color space and enabled features (e.g. BGRA8UnormStorage).
  - wgpu:
    - Currently treats color space implicitly through format choice and present mode; explicit color space selection is not surfaced in the Metal backend.

- HDR and tone mapping:
  - WebKit:
    - Supports HDR workflows (e.g. RGBA16Float) with an explicit `toneMappingMode`.
    - Uses luminance clamp textures when necessary to implement standard tone mapping.
  - wgpu:
    - Exposes HDR-capable formats (like RGBA16Float) in surface capabilities but does not deeply model tone mapping behavior or provide HDR-specific configuration knobs.

- Alpha/compositing behavior:
  - WebKit:
    - Incorporates `compositeAlphaMode` into swap chain configuration, aligning with how canvases are composited into the browser UI.
  - wgpu:
    - Metal backend does not yet expose fine-grained composite alpha choices for surfaces.

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
