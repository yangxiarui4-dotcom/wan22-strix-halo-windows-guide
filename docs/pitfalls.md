# Pitfalls and diagnostics (gfx1151 / ROCm / Windows)

Everything below was hit in practice. Ordered roughly by how much time it cost us.

## 1. Pure-noise output: the text encoder is the prime suspect

**Symptom:** sampling runs to completion, VAE decodes fine, but the video is pure
static noise (colorful blocks or uniform grain).

**Root cause we found:** the text-embedding path, not the diffusion weights.

- kijai's *WanVideo T5 Text Encoder Loader* **does not accept `fp8_scaled` UMT5**
  (e.g. `umt5_xxl_fp8_e4m3fn_scaled.safetensors`) — it errors out.
- The workaround of using native *Load CLIP* + *CLIP Text Encode* + a text-embed
  bridge node **silently produces garbage embeddings for Wan 2.2** → the model
  receives noise as conditioning → noise out. No error anywhere.

**Fix:** use a **bf16 or fp16** UMT5 (`umt5-xxl-enc-bf16.safetensors`) with the
wrapper's own *T5 Text Encoder Loader → WanVideo TextEncode* chain.

Before blaming weights, samplers, or precision, fix the text path first and re-test
with a tiny job (512×512, 17 frames, ~1–4 min).

## 2. gfx1151 has no hardware FP8 — but emulated FP8 is still worth it

The log shows `emulated ops: float8_e4m3fn...`. Counter-intuitively, the fp8_scaled
diffusion weights are **faster** than fp16/bf16 on this APU (~400 vs ~640 s/it at
832×480 cfg 3.0-equivalent sizes): unified memory makes sampling bandwidth-bound,
and fp8 halves the weight bytes moved per step.

Note the asymmetry: fp8_scaled **diffusion weights work**; fp8_scaled **UMT5 does not**
(unsupported by the wrapper's loader).

## 3. `offload_device` on a unified-memory APU is a trap

Setting `load_device: offload_device` in the WanVideo Model Loader parks the 14B model
in the CPU-side memory partition (~32 GB on a 128 GB machine). Symptoms:

- RAM usage ~97%, the whole OS crawls
- sampling appears frozen (progress bar stalls for many minutes)

`hipMemGetInfo`-style tools only report the VRAM aperture (~110 GB), so monitors can
look "fine" while the machine is suffocating. **Use `main_device`.** With the fp8
14B model, peak reserved is only ~21 GB — there is no reason to offload at all.

## 4. Task Manager GPU % is meaningless under ROCm

Windows Task Manager shows 4–8% GPU while the card is actually saturated. Check:

- **AMD Software (Adrenalin) performance tab** — shows true utilization (100% while sampling)
- or Task Manager → GPU → switch one graph to **Compute_0**
- power draw is also a good proxy: ~85–100 W pulsing under load vs ~20–30 W idle

## 5. Wrong VAE = tensor shape errors

- `wan2.2_vae.safetensors` is **48-channel** and only pairs with the 5B TI2V model.
- The 14B T2V/I2V models need the **16-channel** `wan_2.1_vae.safetensors`.
- Error signature: `tensor a (16) vs tensor b (48)` in conv3d during decode.

After adding a model file, **restart ComfyUI** — the dropdown lists are cached and F5
in the browser is not enough.

## 6. Scheduler choice matters for distilled LoRAs

`dpm++_sde` injects fresh noise every step and fought the lightx2v 4-step distilled
LoRA in our tests (pure noise output). **Use `euler`** (or unipc) with step-distill
LoRAs.

## 7. Base precision: use bf16

Wan 2.2 was trained in bf16. We set `base_precision: bf16` in the Model Loader.
fp16 base precision risks overflow artifacts (suspected contributor to early noise
runs); bf16 cost us nothing in speed.

## 8. T2V vs I2V model files

Loading an I2V checkpoint into a T2V workflow fails with a channel mismatch
(`36 channels vs 16`). Check the log line `Model variant detected` after loading —
it should say t2v.

## 9. Frame count is rounded to 4n+1

Asking for 81 frames yields 89 in some node versions (latent temporal factor 4).
Harmless — you get 5.5 s instead of 5.0 s at 16 fps.

## 10. Native ComfyUI Wan 2.2 nodes: not viable here (as of 2026-09)

With the official Comfy-Org weights and native nodes, sampling never progressed at a
usable rate on gfx1151/ROCm. The WanVideoWrapper path works end to end. If you test
native nodes again on a newer ROCm/PyTorch, use a tiny job first.

## Quick smoke-test protocol

Whenever you change anything (model file, precision, scheduler, text path):

1. 512×512, 17 frames, seed fixed, same prompt
2. 6 steps + LoRA ≈ 1 min, or 20 steps bare ≈ 4 min
3. Only trust the decoded video, never the progress bar
