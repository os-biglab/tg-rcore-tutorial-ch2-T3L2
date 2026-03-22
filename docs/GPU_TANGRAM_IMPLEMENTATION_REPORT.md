# CH2-T3L2 GPU Tangram Implementation Report

## 1. Requirement Mapping

Implemented target behavior:

- Based on chapter-2 batch OS, dynamically compose tangram `OS` on screen.
- One user program triggers one rendering step (one tangram block).
- Tangram data organized by blocks, not as a monolithic image.
- Execution remains in S-mode/U-mode split of chapter-2.
- Global static storage for rendering data to avoid large stack pressure.
- User programs write directly into the mapped VirtIO-GPU framebuffer; the kernel only provides framebuffer metadata and flush service.

## 2. Implementation Summary

## 2.1 Cargo and Dependencies

- Kernel crate (`tg-rcore-tutorial-ch2-T3L2`):
  - `virtio-drivers = "0.1.0"`
  - `tg-kernel-alloc = "0.1.0"`
- User crate (`tg-rcore-tutorial-user-T3L2`):
  - Package metadata updated to T3L2 naming/version.
  - Added local `[workspace]` section for independent build by kernel build script.
  - Uses the new framebuffer query syscall to obtain the mapped framebuffer pointer, then renders in place and calls the flush syscall.

## 2.2 Build/Runner Integration

- `.cargo/config.toml` enables VirtIO-GPU in QEMU runner.
- `TG_USER_LOCAL_DIR` points to T3L2 user crate.
- User app list configured via `cases.toml`:
  - `t3l2_step_00` … `t3l2_step_13`.
- CH2 user app load base is set to `0x8100_0000` to keep runtime app memory separate from kernel DMA/framebuffer memory.

## 2.3 Kernel GPU Path

1. Scan MMIO window (`0x1000_1000 + n * 0x1000`) for VirtIO-GPU device (`device_id = 16`).
2. Construct `MmioTransport`.
3. Construct `VirtIOGpu::<SimpleHal, MmioTransport>`.
4. `setup_framebuffer`, clear background, initial `flush`.
5. Expose framebuffer access to user programs through syscalls:
  - `SYSCALL_FRAMEBUFFER` returns framebuffer pointer, length, width, and height.
  - `SYSCALL_FRAMEBUFFER_FLUSH` flushes GPU updates after user-space drawing.
6. After each app exits:
  - the next user step draws directly into the mapped framebuffer,
  - then explicitly requests `gpu.flush()` via syscall,
  - increment step counter.

## 2.4 Tangram Block Renderer

- Implemented in `src/tangram.rs`.
- Piece data is decomposed into indexed blocks.
- Renderer writes BGRA pixels in framebuffer.
- Per-step rendering composes full `OS` progressively.
- The actual drawing happens in user space against the shared framebuffer mapping; the kernel no longer copies a user framebuffer into the GPU framebuffer.

## 3. Key Bug and Root Cause

## 3.1 Symptom

- First step rendered successfully.
- Next `gpu.flush()` failed with `IoError`.

## 3.2 Diagnosis

- `virtio-drivers` reports response-type mismatch as generic `IoError`.
- Binary symbol inspection showed memory overlap:
  - user app load address: `0x8040_0000`
  - previous kernel heap range covered this address.
- Running app overwrote heap-backed kernel objects/state, causing later GPU command failure.

## 3.3 Fix

- Reduced kernel heap size in CH2-T3L2 from `2 MiB` to `1 MiB`.
- Ensured heap range no longer overlaps user app load region.

## 4. Validation

Re-ran `cargo run` for CH2-T3L2:

- GPU initialized successfully.
- All programs `app0` … `app13` ran and exited normally.
- Steps `00` … `13` rendered and flushed without panic.
- Kernel stays in idle wait after the last step (QEMU does not immediately exit), so the final composed image remains on screen.

## 5. Final Status

CH2-T3L2 now satisfies the dynamic batch tangram requirement:

- Stable batch execution.
- Stable per-step GPU flush.
- Block-based tangram composition pipeline fully functional.
- User-space direct framebuffer rendering avoids the extra copy from user buffer to kernel framebuffer.
