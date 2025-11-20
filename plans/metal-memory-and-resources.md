# Plan: Metal memory and resources

## Context

- Area: buffer/texture creation, storage modes, heaps, and resource lifetime on Metal.
- Relevant files:
  - `wgpu-hal/src/metal/device.rs`:
    - `create_buffer`, `destroy_buffer`, `create_texture`, `destroy_texture`.
    - Resource counters.
  - `wgpu-hal/src/metal/mod.rs`:
    - `PrivateCapabilities` flags for `resource_heaps`, `shared_textures`, `supports_memoryless_storage`, etc.
  - `wgpu-hal/src/metal/command.rs`:
    - Copy operations, clears, and transitions (no-op on Metal).

## Current behavior (wgpu Metal)

- Buffers:
  - `create_buffer`:
    - Chooses `MTLResourceOptions::StorageModeShared` if usage includes `MAP_READ` or `MAP_WRITE`.
    - Otherwise uses `StorageModePrivate`.
    - Always sets `CPUCacheModeWriteCombined` when `MAP_WRITE` is present.
    - Does not use `StorageModeManaged` or `HazardTrackingModeUntracked`.
  - Destruction:
    - Resource counters are decremented; relies on Metal’s ARC/retains for actual lifetime.
- Textures:
  - `create_texture`:
    - Chooses storage mode based on usage and capabilities, but does not aggressively exploit memoryless or heap-backed textures.
    - Maps `wgt::TextureDescriptor` to `MTLTextureDescriptor`, honoring format, usage, mip levels, etc.
  - `create_texture_view`:
    - Reuses the original texture when view is full-range and same format/type.
    - Otherwise uses `new_texture_view_from_slice`.
- Heaps and memoryless:
  - `PrivateCapabilities` has flags for `resource_heaps` and `supports_memoryless_storage`.
  - There is no current heap-based allocation strategy in the Metal backend (no explicit `MTLHeap` usage).
  - Memoryless textures (e.g. transient depth/color) are not explicitly allocated, even when supported.

## Known issues / open questions

- Storage mode heuristics:
  - Simple “mappable → shared, otherwise private” heuristic may leave performance on the table:
    - Some read-only buffers could benefit from specific cache modes.
    - Some short-lived resources could be memoryless.
  - No differentiation based on access patterns beyond `MAP_READ`/`MAP_WRITE`.
- Heaps:
  - Metal heaps can improve allocation patterns and reduce fragmentation, especially for many similarly sized textures/buffers.
  - wgpu currently does not use heaps on Metal at all.
- Resource lifetime and hazard tracking:
  - Metal supports `HazardTrackingModeUntracked` and explicit synchronization, but wgpu currently doesn’t take advantage of this.
  - Potential to reduce overhead on advanced users/hardware by disabling automatic hazard tracking when safe.

## WebKit behavior / findings

- Key WebKit references:
  - `Source/WebGPU/WebGPU/Buffer.mm`:
    - `validateCreateBuffer`.
    - `storageMode` helper.
    - `Device::safeCreateBuffer`.
    - `Device::createBuffer`.
  - `Source/WebGPU/WebGPU/Texture.mm`:
    - `Device::createTexture`.
    - `storageMode` helper for textures.
    - `Texture::pixelFormat`, `Texture::usage`.

- Buffers:
  - `Buffer.mm` / `Device::createBuffer`:
    - Validates descriptors according to the WebGPU spec:
      - Usage bits must be non-zero and within the allowed mask.
      - `MAP_READ` and `MAP_WRITE` are mutually exclusive and restricted to copy usages.
      - `mappedAtCreation` requires size to be a multiple of 4.
      - Size must not exceed `device.limits().maxBufferSize`.
    - Storage mode selection (`storageMode(hasUnifiedMemory, usage, mappedAtCreation)`):
      - If the device has unified memory: `StorageModeShared`.
      - On macOS/macCatalyst:
        - If usage includes `MapRead`, `MapWrite`, or `Index`, or `mappedAtCreation` is true: `StorageModeManaged`.
        - Otherwise: `StorageModePrivate`.
      - On iOS family without unified memory: `StorageModePrivate`.
    - `Device::safeCreateBuffer`:
      - Builds `MTLResourceOptions` with explicit `CPUCacheMode`, `StorageMode`, and `HazardTrackingMode`.
      - Current calls use the default cache mode and hazard tracking mode but centralize allocation for future tuning.
    - For indirect-draw buffers, WebKit creates additional shared buffers to hold Metal-specific indirect argument structures.
- Textures:
  - `Texture.mm` / `Device::createTexture`:
    - Validates creation via `errorValidatingTextureCreation`, ensuring descriptor consistency with WebGPU spec and device capabilities.
    - Computes usage flags with `Texture::usage(descriptor.usage, descriptor.format)` and sets `MTLTextureDescriptor.usage` accordingly.
    - Derives `MTLTextureType` and size from `WGPUTextureDescriptor` (dimension, sampleCount, array layers).
    - Maps `WGPUTextureFormat` to `MTLPixelFormat` with `Texture::pixelFormat`, returning a validation error if invalid.
    - Storage mode selection (`storageMode(hasUnifiedMemory, supportsNonPrivateDepthStencilTextures)`):
      - If `supportsNonPrivateDepthStencilTextures` is false:
        - Always uses `StorageModePrivate` (notably for depth/stencil on some platforms).
      - Otherwise:
        - If the device has unified memory: `StorageModeShared`.
        - On macOS/macCatalyst: `StorageModeManaged`.
        - On iOS family without unified memory: `StorageModePrivate`.
    - No explicit use of `MTLHeap` or memoryless textures in this layer; allocations are direct per-texture.
- Heaps and hazard tracking:
  - A grep search shows no `MTLHeap` usage in `Source/WebGPU/WebGPU`.
  - `Device::safeCreateBuffer` takes a `hazardTrackingMode` argument but current usage doesn’t aggressively switch to `HazardTrackingModeUntracked`.
  - Resource ownership is tracked via `setOwnerWithIdentity` and internal maps (e.g. `m_bufferMap`), rather than custom heap management.

## Questions to investigate (later, including WebKit)

- Does WebKit’s WebGPU Metal:
  - Use `MTLHeap` for textures/buffers, or stick with plain device allocations?
  - Allocate memoryless attachments for transient depth/color surfaces?
  - Choose different storage modes/cache modes for different buffer usages?
- How does WebKit manage resource lifetime and hazard tracking:
  - Does it ever opt into `HazardTrackingModeUntracked`?
  - How does it synchronize between encoders/queues?

## Possible improvement directions

- Refine storage mode selection:
  - Evaluate more granular mapping from `wgt::BufferUses`/`TextureUses` to:
    - `StorageModeShared` vs `Private` vs `Memoryless`.
    - CPU cache modes (default vs write-combined).
  - Consider separating “persistent mapping” from “one-time upload” patterns.
  - Introduce optional heap-backed allocations:
    - Add a capability/feature path to allocate some textures/buffers from `MTLHeap`s when supported.
    - Start with obvious candidates (e.g. render targets of similar size) and keep a fallback path.
- Explore memoryless attachments:
  - Use `supports_memoryless_storage` to allocate memoryless depth/color where valid and beneficial.
  - Validate behavior with CTS and typical applications to ensure no lifetime surprises.

## Next steps when resuming

- Re-read `create_buffer` and `create_texture` in `device.rs` to map all storage mode choices.
- Once WebKit sources are available:
  - Locate their resource allocation paths (buffer/texture creation).
  - Compare:
    - Storage modes, cache modes.
    - Heap usage.
    - Memoryless attachments.
  - Identify safe incremental improvements that maintain wgpu’s portability while matching Apple’s best practices where possible.

## Deltas vs wgpu (initial)

- Storage mode policy:
  - WebKit:
    - Buffers: chooses between `Shared`, `Managed`, and `Private` based on unified memory, mapping, and usage (with `Managed` used for many CPU-visible buffers on macOS).
    - Textures: uses `StorageModePrivate` when non-private depth/stencil is unsupported; otherwise `Shared` on unified memory devices and `Managed` on macOS/macCatalyst.
  - wgpu:
    - Buffers: uses `StorageModeShared` for any mappable buffer, `StorageModePrivate` otherwise; no `Managed` usage.
    - Textures: storage mode selection is driven by `PrivateCapabilities` but does not mirror WebKit’s depth/stencil-specific policy.
- Heaps and memoryless:
  - Both WebKit and wgpu’s Metal backends currently:
    - Do not use `MTLHeap` for buffer/texture allocation.
    - Do not explicitly allocate memoryless textures at this layer, even when Metal supports them.
- Hazard tracking and cache modes:
  - WebKit:
    - Centralizes buffer creation in `safeCreateBuffer`, with explicit parameters for cache and hazard tracking modes, but currently uses defaults with comments noting future tuning possibilities.
  - wgpu:
    - Sets `CPUCacheModeWriteCombined` for writable buffers and leaves hazard tracking at Metal’s defaults.
