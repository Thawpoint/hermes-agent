# RunPod ComfyUI + Qwen Setup Plan

**Created:** 2026-05-26  
**Goal:** Get a RunPod pod running with ComfyUI + Qwen Image, using network storage for persistent models (reusing the WAN setup pattern).

---

## Your existing ComfyUI lane reference (what to match)

Your local NBN Qwen lane on 8190 runs:

| Component | File |
|---|---|
| UNet | `qwen_image_fp8_e4m3fn.safetensors` |
| CLIP | `qwen_2.5_vl_7b_fp8_scaled.safetensors` |
| VAE | `qwen_image_vae.safetensors` |
| Lightning LoRA | `Qwen-Image-Lightning-8steps-V2.0-bf16.safetensors` |
| Style LoRA | `nbn_style_v6e6.safetensors` (or `nbn_style_v4e2.safetensors`) |
| Lapis LoRA | `lapis_663_v2e6.safetensors` |
| Anora LoRA | `Anora_663_v1e3.safetensors` |
| Sampler/scheduler | Euler / simple |
| Steps | 8 |
| CFG | 1.0 |
| Shift | 3.0 |
| Canvases | 1024x1024, 1216x832, 1344x896 |

---

## Step 1: Choose GPU

| GPU | VRAM | Price (Jan 2026) | Works for Qwen? |
|---|---|---|---|
| RTX 5090 | 32 GB | ~$0.90/hr | ✔ FP8 fits comfortably, can do higher resolutions |
| RTX 4090 | 24 GB | ~$0.34/hr | ✔ FP8 fits but tight (~86% VRAM, no headroom for extras) |
| RTX 6000 Ada | 48 GB | ~$0.79/hr | ✔ Overkill, but room for BF16 if you ever want to try |

**Recommendation:** RTX 5090. You're already running on a 3080 locally — the point of RunPod is faster compute, and the 5090 gives you headroom.

---

## Step 2: Network Volume

Your WAN setup used a network volume — same pattern here.

### If you have an existing network volume:
Check if it already has ComfyUI + models in `/workspace`. If so, skip the download steps and verify paths.

### If creating a new network volume:
- **Size:** 200 GB minimum (Qwen fp8 alone is ~30 GB, plus text encoder, VAE, LoRAs, ComfyUI itself, and output space)
- **Region:** Pick the same region as your GPU to avoid transfer costs
- **Cost:** ~$0.022/hr while kept ($0.53/day, ~$16/month) — delete when done with the project to stop storage billing

---

## Step 3: Deploy the Pod

1. Go to [RunPod Console → Pods](https://www.runpod.io/console/pods)
2. Click **+ Deploy**
3. Select GPU: **RTX 5090** (Community Cloud)
4. Search template: **`ashleykza/comfyui:cu128-py312-v0.10.0`**
   - This template includes PyTorch 2.8+, CUDA 12.8, and ComfyUI pre-installed
   - RTX 5090 requires PyTorch 2.8+ — standard ComfyUI templates won't work
5. **Network Volume:** Select your existing WAN volume or create a new one
6. **Volume mount path:** `/workspace`
7. **Container disk:** 50 GB (temporary — all your models go on the network volume)
8. Click **Deploy**
9. Wait for the pod to show "Running"

---

## Step 4: Access ComfyUI

- **HTTP:** Click "Connect" → HTTP Service (port 3000) — this opens ComfyUI in your browser
- **SSH:** Use `Connect → SSH` or `ssh -p <port> root@<ip>` (your public key needs to be in RunPod settings)
- **Web Terminal:** Click "Web Terminal" on the pod page for a browser-based shell

---

## Step 5: First Session — Download Models (if new volume)

Run this from the Web Terminal or SSH. All models go to the network volume at `/workspace/ComfyUI/models/`:

```bash
# Base directory
cd /workspace/ComfyUI/models

# FP8 diffusion model (20.4 GB)
mkdir -p diffusion_models
wget -P diffusion_models \
  https://huggingface.co/Comfy-Org/Qwen-Image_ComfyUI/resolve/main/split_files/diffusion_models/qwen_image_fp8_e4m3fn.safetensors

# Text encoder (9.4 GB)
mkdir -p text_encoders
wget -P text_encoders \
  https://huggingface.co/Comfy-Org/Qwen-Image_ComfyUI/resolve/main/split_files/text_encoders/qwen_2.5_vl_7b_fp8_scaled.safetensors

# VAE (0.25 GB)
mkdir -p vae
wget -P vae \
  https://huggingface.co/Comfy-Org/Qwen-Image_ComfyUI/resolve/main/split_files/vae/qwen_image_vae.safetensors

# Lightning LoRA
mkdir -p loras
wget -P loras \
  https://huggingface.co/lightx2v/Qwen-Image-Lightning/resolve/main/Qwen-Image-Lightning-8steps-V2.0-bf16.safetensors
```

> **Note:** Your NBN style and character LoRAs (`nbn_style_v6e6`, `lapis_663_v2e6`, `Anora_663_v1e3`) are custom — upload them via SCP or download from wherever they're hosted.

---

## Step 6: Install Custom Nodes (if needed)

Most of these come pre-installed on the `ashleykza` template. Verify in ComfyUI Manager:

- **ComfyUI Manager** — should already be there
- **ComfyUI-GGUF** — only if you plan to use GGUF quantized models (not needed for FP8)

If anything is missing:
1. Open ComfyUI → click **Manager** (top-right)
2. Search for the missing node → **Install**
3. Restart ComfyUI

---

## Step 7: Verify Everything Works

1. Drag your NBN Qwen workflow JSON onto the canvas
2. Make sure all nodes show green (no red boxes)
3. Check the **Load Diffusion Model** node points to `qwen_image_fp8_e4m3fn.safetensors`
4. Check the **Load CLIP** node points to `qwen_2.5_vl_7b_fp8_scaled.safetensors`
5. Check the Lightning LoRA node is wired and set to model strength 1.0
6. Enter a short test prompt, set steps to 8, and queue one generation
7. Verify it completes without errors

---

## Step 8: Stop When Done

**Always stop the pod when you're not actively generating.** GPU billing continues while running.

- **Stop:** Stops GPU billing but keeps the pod config + network volume. Can restart in minutes. Network volume billing continues.
- **Terminate:** Destroys the pod completely. Network volume data persists. Use this when you're done for a while.

### Cost reference:
- Running 5090: ~$0.90/hr
- Stopped pod + volume: ~$0.022/hr (~$0.53/day)
- Restart from stop: 2–5 minutes (model already on network volume)

**Rule:** Stop after each session. Terminate if you won't use it this week. Delete the network volume only when the project is fully done.

---

## Step 9: Subsequent Sessions

Re-deploy is fast because models are on the network volume:

1. Select the same GPU type and template
2. Attach the same network volume
3. Pod comes up with all models ready
4. Load your workflow and start generating

---

## Automation Ideas (future)

Once the manual setup is proven:
- Save the workflow as a JSON template on the network volume
- Use `runpodctl` CLI to deploy/stop from your desktop
- Write a Hermes skill for one-command RunPod session launch
- Use the RunPod Serverless pattern for queue-based generation (see Vicky profile's `runpod-serverless-gpu` skill)

---

## Open Questions

1. **Existing network volume?** Do you already have a WAN network volume with `/workspace` data, or are we creating from scratch? If it exists, what's the volume ID?
2. **NBN LoRA hosting?** Where are your custom LoRAs hosted — do they need to be uploaded, or are they on a HuggingFace/custom URL?
3. **GPU preference?** 5090 ($0.90/hr) or 4090 ($0.34/hr)?
4. **Workflow JSON?** Which specific workflow file should I pre-load — the sprite variant workflow, the lapis lounger workflow, or a fresh Qwen T2I template?