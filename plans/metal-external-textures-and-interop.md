# Plan: Metal external textures and interop

## Context

- Area: integration of external image sources (video, camera, platform images) as WebGPU textures, focusing on Metal behavior.
- Relevant wgpu files:
  - Currently limited external texture support (mostly via WebGPU API abstractions, not Metal-specific code).
- Relevant WebKit files:
  - `../WebKit/Source/WebGPU/WebGPU/ExternalTexture.*`.
  - WebCore: `GPUExternalTexture.*` and related glue.

## Current behavior (wgpu Metal, conceptually)

- wgpu has minimal external texture support, and Metal backend does not have detailed platform-specific handling comparable to WebKit’s integration with IOSurface and media pipelines.

## WebKit behavior / findings (initial)

- Key WebKit references:
  - `Source/WebGPU/WebGPU/ExternalTexture.h/.mm`:
    - External texture creation and validation.
    - Rules for format, dimension, mip levels, and sample count.
  - WebCore:
    - `Source/WebCore/Modules/WebGPU/GPUExternalTexture.*` (JS-facing external texture wrapper and integration).

- External textures:
  - `ExternalTexture.mm`:
    - Validates that the source texture/view meets WebGPU’s external texture constraints (format, dimension, mip levels, sample count).
    - Integrates with WebKit’s media infrastructure to ensure frames are synchronized and in a supported color space.
    - Enforces usage and binding rules specific to external textures (e.g. only certain slots and usages allowed).
- Policy:
  - Certain formats and usages are whitelisted for external textures, and validation errors are clear when constraints are violated.

## Deltas vs wgpu (initial)

- External texture support:
  - WebKit:
    - Has a fully specified external texture path, integrated with the browser’s media stack (pixel formats, color spaces, synchronization).
  - wgpu:
    - Only has minimal, API-level external texture concepts; the Metal backend does not yet have platform-specific external texture behavior comparable to WebKit’s.

- Validation and constraints:
  - WebKit:
    - Validates format, dimension, mip levels, and sample count against a well-defined whitelist for external textures.
    - Provides clear error messages when constraints are violated.
  - wgpu:
    - Does not yet enforce a Metal-specific external texture policy; most of this space is open for design.

## Known issues / open questions

- Applicability to native wgpu:
  - Which parts of WebKit’s external texture behavior make sense to port to a native, non-browser context?
  - How should wgpu model external images on Metal (IOSurface, CVPixelBuffer, etc.)?

## Possible improvement directions

- Design a Metal-oriented external texture story:
  - Take WebKit’s constraints and validation as a starting point for defining a safe subset of external texture behavior on Metal.
  - Consider interop with common native media APIs (AVFoundation) and shared image surfaces.

## Next steps when resuming

- Survey WebKit’s `ExternalTexture` and WebCore’s `GPUExternalTexture` to enumerate constraints and behaviors.
- Propose a minimal, portable external texture model for wgpu’s Metal backend.
