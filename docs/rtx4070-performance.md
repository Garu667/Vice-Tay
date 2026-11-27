# RTX 4070 performance work

Select `-Machine 4070` for the RTX 4070 / Ryzen 7 7800X3D / 32 GB
machine. Automatic selection gives it to NVIDIA cards with less than 16 GB
(read with `nvidia-smi`); larger NVIDIA cards keep the 4090 profile. The 4070
profile keeps the console
selection (Series X by default), output mode, texture mip policy, and the
four-shader wave64 allowlist unchanged.

Changes:

- Eight recording, six under-write, four landing, one mapping and three
  pipeline compilation workers instead of the 16-core 4090 machine's counts.
- A 64 KiB minimum for GPU-resident title buffers, a 1536 MiB buffer budget,
  and a separate 1024 MiB budget for memory windows read by shaders.
- With `VICTORY_CARD_RESERVE_BUDGETS=1`, texture admission reserves explicit
  buffer budgets in full and includes
  unused window budgets. Windows may use that reserved room under the same
  97% limit as buffers. Previously, a larger buffer setting reserved at most
  1 GiB, and windows were refused at 80% occupancy without any reservation.

This trades some texture admission headroom for fewer shader accesses over
PCIe and fewer CPU snapshots. It does not remove rendering passes, change
wave64 semantics, lower loop limits, or enable changed-word-only uploads.
The 4070 profile enables the reservation outside `-Safe`. With the switch unset
or set to `0`, the texture reservation is capped at 1 GiB for buffers and
the window occupancy threshold stays at 80%. `-More` can override individual settings.

## Shader research

- [NVIDIA: Advanced API Performance — Intrinsics](https://developer.nvidia.com/blog/advanced-api-performance-intrinsics/)
  recommends wave operations to reduce shared-memory accesses and barriers.
  That is applicable only when the required communication stays within a
  host wave; it does not make two independent 32-lane waves one 64-lane wave.
- [Microsoft: HLSL Wave Size](https://microsoft.github.io/DirectX-Specs/d3d/HLSL_SM_6_6_WaveSize.html)
  requires the selected size to be supported by the device. The RTX 4070
  reports a minimum and maximum of 32; selecting WaveSize(64) is not a solution.
- [Microsoft: Wave Intrinsics](https://github.com/Microsoft/DirectXShaderCompiler/wiki/Wave-Intrinsics)
  specifies active-lane behavior. Replacing cross-wave exchanges with local
  shuffles can change EXEC-mask behavior and output.
- [AMD: RDNA Performance Guide](https://gpuopen.com/learn/rdna-performance-guide/)
  discusses register pressure, wave operations, and avoiding CPU readbacks.
  Host-side memory placement must also match the discrete GPU memory model.
- [Microsoft: Uploading resources](https://learn.microsoft.com/en-us/windows/win32/direct3d12/uploading-resources)
  recommends default resources for data read repeatedly by the GPU rather
  than repeatedly reading upload memory. This informs the buffer/window
  placement changes here.

Vice-Tay already contains group-uniform barrier checks and wave64 correctness
fixes. They remain intact. The observed baseline spends substantial time on
submission preparation and CPU snapshots, so memory placement is the first
optimization tested; no shader speedup is claimed without shader timings.

## Local validation (2026-11-26)

Nine `shadow::window_tests` passed, including reservation accounting, overlapping
windows, tail coverage, and eviction. The diagnostic application build passed.
The original and patched runs both used Series X, diagnostic builds, info logs,
and disabled DRED/memory-map walks. The user loaded the same gameplay scene
and reported that rendering looked the same with the new profile.

Last eight logged samples with more than 1,000 draws per frame:

| | Original 4090 profile | New 4070 profile |
|---|---:|---:|
| Mean logged frame time | 213.4 ms | 76.6 ms |
| FPS sample range | 2.8–7.8 | 5.9–25.0 |
| Mean logged draws/frame | 1,705 | 1,882 |

These are exploratory gameplay samples, not a locked-camera benchmark. Streaming,
camera position, and pipeline-cache warming can affect the comparison. They do
not establish the individual contribution of each setting or long-session
stability. Later sampled frames had no CPU buffer snapshots. Memory pressure
remains: the patched run reached about 25.6 GB private memory on this 32 GB PC.

Raw logs for this local test are in the sibling checkout's ignored `target`
folder: `D:\Games\cool_fox-archive\target\vice-baseline.err.log` and
`D:\Games\cool_fox-archive\target\vice-4070.err.log`.

## Series S check on an RTX 5070 Ti (2026-11-26)

An RTX 5070 Ti (16 GiB, driver 621.24), Core i5-14600KF, and 32 GiB of system
RAM ran the Series S profile with 1920×1080 full-screen output. Both runs used
the same native Windows release executable based on `e1067ca`, the same local
water-height readback fixes and measurement code, and identical starting copies of
the checkpoint, translated shaders, and host pipeline library. Pipeline scheduling
history was disabled in both. The explicit 4070 preset from [#69](https://github.com/TaynanGT/Vice-Tay/pull/69)
ran first. It changes the worker counts and the GPU buffer and window budgets together.

The stationary sample starts 400 seconds after launch with the camera facing
the bridge and zero pipelines pending. The open-water sample covers complete
intervals between 4 and 20 seconds after applying the same forward input at
480 seconds. FPS uses frames accepted by DXGI divided by elapsed time.

| Metric | Default NVIDIA, stationary | 4070 preset, stationary | Default NVIDIA, open water | 4070 preset, open water |
|---|---:|---:|---:|---:|
| Presentations / seconds | 919 / 55.72 | 1,206 / 55.95 | 242 / 15.47 | 289 / 14.40 |
| Average FPS | 16.49 | **21.56** | 15.65 | **20.07** |
| Seconds in intervals below 20 FPS | 55.72 | 1.01 | 15.47 | 4.10 |
| Draws per frame, interval range | 3,530–3,584 | 3,438–3,462 | 3,549–4,029 | 3,521–3,845 |
| CPU submission preparation, ms/frame | 48.07 | 37.17 | 50.87 | 39.48 |
| Peak private memory, MiB | 22,300 | 24,083 | 23,038 | 24,146 |
| Minimum DXGI local budget, MiB | 15,227 | 14,157 | 15,227 | 14,190 |
| Peak DXGI local usage, MiB | 9,012 | 10,879 | 9,653 | 10,862 |
| Peak waiting texture slots | 8,372 | 6,823 | 6,694 | 8,382 |
| Texture attempts deferred for local GPU memory | 0 | 0 | 0 | 0 |

Both runs showed the boat, water reflections, wake, and nearby scenery. With
the preset, a longer acceleration toward the shore averaged 17.60 FPS over
33.06 seconds and one interval dropped to 3.51 FPS. The boats reached different
positions during the longer run. Sustained 20 FPS through shoreline streaming
and longer-session behavior still need validation.

## Series S check on an RTX 3080 10 GB (2026-11-27)

An RTX 3080 10 GB (driver 610.88, Windows 11 26200), Core i5-13600K
(6 performance and 8 efficient cores, 20 threads), and 32 GB of RAM ran the
Series S profile from `0ef44bc`, diagnostics build, full screen, in the prison
cell: the character standing at the window for three minutes. Each row is one
run. FPS is the median of the last 60 once-a-second frame lines in the world
(3 to 29 fps, 4,300 to 4,500 draws a frame); the pipelines the first run drew
with were made by `warm --used` before the others.

| Run | Settings | FPS median | p10 | ms a frame: preparing / on the GPU / copying back | Peak VRAM | GPU busy |
|---|---|---:|---:|---|---:|---:|
| 1 | automatic (the 4090 profile), cold pipeline cache | 7.3 | 6.4 | 55 / 66 / 49 | 5.8 GB | 82 % |
| 2 | `-Machine 4070` | **14.4** | 12.8 | 47 / 6 / 35 | 7.3 GB | 69 % |
| 3 | `-Machine 4070`, `VICTORY_D3D12_DRED=0`, `VICTORY_OUTPUT=1920x1080` | 15.2 | 14.2 | 47 / 7 / 30 | 7.2 GB | 71 % |
| 4 | as 3, with `VICTORY_PLACED_ORDER=1` | 14.6 | 10.7 | 46 / 7 / 30 | 7.6 GB | 73 % |

The 4070 profile doubles the frame rate on this card: with the 4090 profile
the card spends 66 ms a frame (busy 82 % for 7 fps), most likely reading
title buffers that the 4090 profile leaves in system memory over PCIe; the
4070 profile's 64 KiB minimum and 1,536 MiB budget for card-resident buffers
fit in 10 GB here (7.3 GB at most). Run 1 also had a cold pipeline cache,
which drops draws but should not account for 59 ms of card time.
`VICTORY_PLACED_ORDER=1` (from the 4090 profile) did not help on this machine.
With the 4070 profile the card is 70 % busy: the limit is now the CPU side
(preparing about 47 ms and copying back 30 to 35 ms a frame). No device was
lost in any run. Open-world streaming and longer sessions are not covered yet.
