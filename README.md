# Wan 2.2 on AMD Strix Halo (gfx1151) — Windows + ROCm Guide

A verified working setup for running **Wan 2.2 14B text-to-video** in ComfyUI on the
AMD Ryzen AI Max+ 395 (Radeon 8060S, gfx1151, 128 GB unified memory) under **Windows 11**
with **PyTorch ROCm nightlies** — no NVIDIA GPU, no WSL, no dual boot.

As of this writing we found no other confirmed report of WanVideoWrapper running on this
platform, so this repo documents the full recipe, the pitfalls that silently produce
pure-noise output, and measured production performance.

## Hardware

| Component | Spec |
|---|---|
| APU | AMD Ryzen AI Max+ 395 (Strix Halo) |
| GPU | Radeon 8060S, gfx1151 (RDNA 3.5, 40 CUs) |
| Memory | 128 GB LPDDR5X unified (~110 GB exposed to GPU) |
| OS | Windows 11 |
| Power mode tested | 80 W sustained (see [performance](#measured-performance)) |

## Software stack

| Piece | Version |
|---|---|
| ComfyUI | 0.34.0 |
| Python | 3.12.10 (venv) |
| PyTorch | 2.12.0a0+rocm7.13 nightly (ROCm 7.x, gfx1151 wheels) |
| Custom nodes | [ComfyUI-WanVideoWrapper](https://github.com/kijai/ComfyUI-WanVideoWrapper) (kijai) |

> **Important:** ComfyUI's *native* Wan 2.2 nodes were not usable on this platform
> (sampling effectively never progresses). The kijai WanVideoWrapper route is the
> working path.

## Quick start

1. Download the four model files → [docs/model-list.md](docs/model-list.md)
2. Skim the pitfalls (especially #1, the text encoder) → [docs/pitfalls.md](docs/pitfalls.md)
3. Import a workflow from `workflows/` into ComfyUI and run

## Verified model combination

All downloads work from ModelScope mirrors (HF unreachable in some regions). Exact
commands in [docs/model-list.md](docs/model-list.md).

| Role | File | Notes |
|---|---|---|
| Diffusion model | `Wan2_2-T2V-A14B-HIGH_fp8_e4m3fn_scaled_KJ.safetensors` (Kijai/WanVideo_comfy_fp8_scaled) | ~15 GB, kijai's own packaging |
| Text encoder | `umt5-xxl-enc-bf16.safetensors` (Kijai/WanVideo_comfy) | **must be bf16/fp16 — NOT fp8_scaled** |
| VAE | `wan_2.1_vae.safetensors` (Comfy-Org repackaged) | 16-channel. The `wan2.2_vae` is 48-ch for the 5B TI2V model only |
| Speed LoRA | `wan2.2_t2v_lightx2v_4steps_lora_v1.1_high_noise.safetensors` | strength 1.0 |

## Workflows

| File | Purpose | Config |
|---|---|---|
| `workflows/wan22_t2v_14b_draft_6step_lightx2v_gfx1151.json` | Draft / batch production | lightx2v LoRA, **6 steps, cfg 1.0, euler** |
| `workflows/wan22_t2v_14b_final_20step_gfx1151.json` | Final quality | no LoRA, **20 steps, cfg 3.0, euler** |

Both use: KJ fp8_scaled HIGH model (`base_precision: bf16`, `main_device`, sdpa),
wrapper-native T5 chain (bf16 umt5), `wan_2.1_vae` (bf16), Enhance-A-Video weight 2.0,
832×480 / 89 frames (5.5 s @ 15–16 fps).

The bypassed (purple) native-CLIP group in these workflows is kept **only** as a
warning exhibit — it silently produces pure noise on this platform. Do not enable it.

## Measured performance (80 W mode, GPU ~78 °C sustained)

| Workload | Speed | Total time | Peak VRAM |
|---|---|---|---|
| 512×512 / 17 frames, 6 steps + LoRA (cfg 1.0) | 8.24 s/it | **58 s** | 16.4 GB |
| 832×480 / 89 frames, 6 steps + LoRA (cfg 1.0) | 396.4 s/it | **40 min 16 s** | 21.4 GB |
| 832×480 / 81 frames, 20 steps bare (cfg 3.0) | ~640 s/it | ~3.5 h | ~56 GB |

Notes:
- The fp8 (emulated) KJ weights are **~40% faster** than bf16/fp16 weights here —
  unified memory makes this workload bandwidth-bound, and fp8 halves weight traffic.
- 80 W is the sweet spot; the 120 W chassis mode is expected to gain little since the
  bottleneck is not the power wall.
- Running two ComfyUI instances in parallel is unlikely to scale: both share the same
  LPDDR5X bandwidth pool.

## Documentation

- [docs/pitfalls.md](docs/pitfalls.md) — everything that silently breaks and how to diagnose it
- [docs/model-list.md](docs/model-list.md) — exact download commands (ModelScope + HF)

## License

Documentation: CC BY 4.0. Models are subject to their own licenses (Wan 2.2: Apache 2.0).
