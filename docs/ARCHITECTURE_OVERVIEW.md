# CH2-T3L2 Architecture Overview

## 1. Goal

CH2-T3L2 extends the chapter-2 batch OS to support **dynamic, step-by-step tangram rendering** of the `OS` pattern on a VirtIO-GPU framebuffer.

- Kernel stays in S-mode.
- User apps run in U-mode and execute in batch order.
- Each user app corresponds to one tangram block update.
- Tangram block data is globally stored in kernel static data.
- User apps obtain the framebuffer pointer through a syscall and write directly into the GPU framebuffer mapping.

## 2. Runtime Pipeline

1. Boot into `_start`, then enter `rust_main`.
2. Zero BSS, initialize console/logging.
3. Initialize kernel heap allocator.
4. Initialize syscall dispatch (`io` + `process`).
5. Discover VirtIO-GPU MMIO base by scanning slots.
6. Create `MmioTransport` and `VirtIOGpu<SimpleHal, MmioTransport>`.
7. Setup framebuffer, clear background, first `flush`.
8. Iterate user apps in batch mode:
   - Load app to fixed address.
   - Run app in U-mode via `LocalContext::user`.
  - On `exit`, the user program has already drawn one tangram block directly into the mapped framebuffer.
  - User program then requests a framebuffer flush through syscall.
9. Enter idle wait (no shutdown) when all apps finish, keeping the final frame visible.

## 3. Core Modules

### 3.1 `src/main.rs`

- Kernel boot path and batch scheduler loop.
- Trap/syscall handling (`UserEnvCall` path).
- VirtIO-GPU initialization and per-step flush.
- Framebuffer query/flush syscalls that expose the mapped GPU framebuffer to user space.
- DMA HAL implementation (`SimpleHal`) and static DMA pool.
- Kernel allocator bootstrap (`tg-kernel-alloc`).

### 3.2 `src/tangram.rs`

- Defines tangram piece decomposition into indexed blocks.
- Exposes:
  - `BLOCK_COUNT`
  - `render_block(framebuffer, width, height, idx)`
- Uses software triangle rasterization into BGRA framebuffer memory.

## 4. Memory/Layout Notes

- User apps are loaded at `0x8100_0000` for CH2-T3L2 to avoid overlap with kernel heap and DMA/framebuffer regions.
- GPU driver internal state and queue metadata can depend on kernel heap allocations.
- Heap/DMA/framebuffer regions must not overlap app load region.
- CH2-T3L2 sets a conservative heap size (`1 MiB`) to avoid overlap-related corruption.

## 5. Data & Control Separation

- **Control plane**: batch loop + trap handling + syscall dispatch.
- **Render plane**: user-space framebuffer writes + GPU `flush`.
- **Data plane**: static tangram block descriptors and transform parameters.

This separation keeps per-app logic simple while ensuring deterministic visual progression.
