# Model downloads

All files verified working together on gfx1151 / ROCm / Windows. ModelScope mirrors
are listed because huggingface.co (and hf-mirror.com) may be unreachable from some
networks; the same files exist on Hugging Face under the same repo names.

Install the downloader once (inside your ComfyUI venv):

```powershell
pip install modelscope
```

Suggested layout (point `extra_model_paths.yaml` at it):

```
D:\models\wan\
  diffusion_models\
  text_encoders\
  vae\
  loras\
```

Example `extra_model_paths.yaml` in your ComfyUI folder:

```yaml
wan:
  base_path: D:\models\wan\
  diffusion_models: diffusion_models
  text_encoders: text_encoders
  vae: vae
  loras: loras
```

## 1. Diffusion model (required) — ~15 GB

```powershell
modelscope download --model Kijai/WanVideo_comfy_fp8_scaled ^
  "T2V/Wan2_2-T2V-A14B-HIGH_fp8_e4m3fn_scaled_KJ.safetensors" ^
  --local_dir "D:\models\wan\diffusion_models"
```

HF source: https://huggingface.co/Kijai/WanVideo_comfy_fp8_scaled/tree/main/T2V

Only the HIGH-noise model is needed for single-model T2V runs. There is also a
fused variant `Wan2_2-T2V-A14B-HIGH_4_steps-250928-dyno-lightx2v_...` with the
speed LoRA baked in (untested here).

## 2. Text encoder (required) — ~11 GB

```powershell
modelscope download --model Kijai/WanVideo_comfy ^
  "umt5-xxl-enc-bf16.safetensors" ^
  --local_dir "D:\models\wan\text_encoders"
```

HF source: https://huggingface.co/Kijai/WanVideo_comfy

**Do not use `umt5_xxl_fp8_e4m3fn_scaled` builds** — unsupported by the wrapper's
T5 loader, and the native-CLIP workaround silently yields noise (see pitfalls #1).

## 3. VAE (required) — ~254 MB

```powershell
modelscope download --model Comfy-Org/Wan_2.2_ComfyUI_Repackaged ^
  "split_files/vae/wan_2.1_vae.safetensors" ^
  --local_dir "D:\models\wan\vae"
```

The 14B models need the **2.1** VAE (16 channels), not the 2.2 VAE (48 ch, 5B-only).

## 4. Speed LoRA (recommended) — ~1.5 GB

```powershell
modelscope download --model Comfy-Org/Wan_2.2_ComfyUI_Repackaged ^
  "split_files/loras/wan2.2_t2v_lightx2v_4steps_lora_v1.1_high_noise.safetensors" ^
  --local_dir "D:\models\wan\loras"
```

Use at strength 1.0 with 4–6 steps, cfg 1.0, euler.

## Listing files in a repo (useful when names change)

```powershell
python -c "from modelscope.hub.api import HubApi; [print(f['Path']) for f in HubApi().get_model_files('Kijai/WanVideo_comfy', recursive=True)]"
```

## After downloading

Restart ComfyUI — model dropdowns are cached and browser refresh is not enough.

Launch:

```powershell
cd D:\ComfyUI
.\venv\Scripts\Activate.ps1
python main.py --listen 0.0.0.0 --port 8188
```

Then open http://127.0.0.1:8188 (not 0.0.0.0).
