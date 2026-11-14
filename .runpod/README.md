# 10Eros H3 video for ComfyUI on Runpod

Image and text to video with sound, using [10Eros-Max H3 beta7 fix1](https://huggingface.co/TenStrip/10Eros-Max) (MiniMax H3 with an 8-step turbo merged in), in ComfyUI on Runpod.

- **Image:** `ghcr.io/acdevmedia/erosh3-comfyui:h3-beta7-fix1`. Every passing `develop` build moves this tag forward. `sha-<full commit>` tags pin a single build.
- **Status:** generation, speed defaults and 30 s clips have been run on an A100 80 GB (see [validation](docs/VALIDATION.md)). The Runpod template is private; public Hub submission is deferred.

## Quick start

1. Deploy the **ACDEV 10Eros H3 beta7 fix1 ComfyUI Video** template on an **80 GB GPU** (A100 80 GB tested; H100 or RTX PRO 6000 Blackwell should be faster). It needs a CUDA 13 host driver and a 200 GB container disk.
2. On first boot the pod downloads and verifies about 63 GB of models, then starts ComfyUI on port **8188**. Download progress in the log is not generation progress.
3. In ComfyUI open **Workflows → H3 video (10Eros beta7 fix1)**, write your prompt, add a starting image for image-to-video, and queue.

## Workflows

| File | Model | Steps | Use |
|---|---|---|---|
| 01 Eros - Image to Video | 10Eros beta7 fix1 | 8 | Recommended |
| 02 Eros - Text to Video | 10Eros beta7 fix1 | 8 | Recommended |
| 03 Standard 20-step - Image to Video | Base MiniMax H3 | 20 | Comparison, slower |
| 04 Standard 20-step - Text to Video | Base MiniMax H3 | 20 | Comparison, slower |

No workflow loads a LoRA. The turbo is already merged into 10Eros, and the separate Comfy-Org turbo adapters (which also reduced motion) were removed.

## The canvas

Every workflow has the same four panels as the LTX and Eros bundles:

- **01 · Describe your shot**: the prompt. H3 also generates the audio from it, so describe the sound and any dialogue.
- **Starting image** (image-to-video only): upload your own image. `reference.png` is only a placeholder.
- **02 · Shot settings**:
  - **Width and Height**: rounded to multiples of 32, so 960×540 becomes 960×544.
  - **Exact Frame**: with a starting image, Width × Height sets the area and the image sets the shape, so a portrait upload renders portrait. Turn Exact Frame on to use your exact size, centre-cropping the image.
  - **Duration**: in seconds, default 30 s, no upper limit.
  - **Seed**: randomized *before* every queue, so the number shown is the one used. Switch its seed mode to fixed, increment or decrement to control takes.
  - **Steps**.
  - **CFG**: 1 by default. Above 1 it uses the negative prompt and roughly doubles the time per step.
  - **Negative prompt**.
  - The whole pipeline sits inside this panel; double-click it to inspect or edit.
- **03 · Your video**: the MP4 (24 fps, with audio). The seed actually used is saved in its metadata.
- **About speed and length**: the [Speed and length](#speed-and-length) section below, on the canvas.

API graphs in `versions/h3-beta7-fix1/api/` use seed 0, which makes the server draw a fresh random seed per job.

## Speed and length

### Why long clips are slow
H3 turns every ~0.7 s of video (17 frames) into 5 latent frames, and every latent frame at 1024×576 into 576 tokens. A 30 s clip is about 125,000 tokens. Each sampling step compares every token with every other one (attention), so **doubling the length roughly quadruples the attention time**. Text encoding and decoding barely matter; nearly all the time is the sampling steps.

### What the workflows do by default
Both settings are inside **02 · Shot settings** (double-click it). They are applied in this order: model → H3 sampling shift → **Fast attention** → **Sparse attention**, the same chain as Comfy-Org's own FastH3 template.

**Fast attention (int8)**: *Model Attention Backend* set to *comfy kitchen attention*. Attention is computed with 8-bit quantised queries and keys (SageAttention-style) instead of 16-bit. It runs on every step that sparse attention does not handle.

**Sparse attention (long clips)**: *Model Sparse Attention*, method *sol-attn*. For each block of 64 tokens, the kernel measures which other blocks actually matter and computes those exactly. It skips the rest, plus keeps 256 extra top-scoring tokens. It needs no special model weights.
- **tau 1.3**: how selective it is. Higher is sparser and faster. Per the node's guidance, 1.0 keeps ~16 % of blocks, 1.5 ~7 %, 2.0 ~2.7 %.
- **start_percent 0.2**: the first 20 % of the noise schedule stays dense. With Eros (8 steps, shift 12) that is steps 1–2, which set composition and motion. Standard (20 steps) keeps steps 1–4 dense.
- **sink_conditioning exact_kv_and_rows**: the prompt, starting image and audio stay exact, and the audio is computed densely, so speech and sound are not thinned.
- **min_tokens 12,288**: very small clips stay dense, where sparse would not help.

### Measured speed
A100 80 GB, Eros:

| Clip | Dense attention (old) | Default (int8 + sparse) |
|---|---|---|
| 10 s, 1024×576, text-to-video, same seed | 2.7 min | **1.6 min** |
| 15 s, 1024×576 | about 6 min | n/a |
| 30 s, 960×544 | n/a | **about 7.5 min**, consistently |
| 40 s, 1024×576, text-to-video | about 31 min | n/a |

The gain grows with length: on a 30 s clip a dense step took 105 s and a sparse step 44 s. The same-seed 10 s pair looked comparable in composition, faces, texture, motion and audio. The one difference seen was a less natural car splash in the sparse version, so watch splashes and fast water.

### Trading speed against quality
- **Softer detail, shimmer or flicker?** Lower tau to 1.0, or raise start_percent to 0.3–0.4.
- **Want more speed?** Raise tau to 1.5. Check fine texture and faces.
- **Turn it off for an A/B**: select *Sparse attention* and bypass it (Ctrl+B), and set *Fast attention* to *pytorch attention*. Keep the same seed.

### Length
Duration has no upper limit; the default is 30 s. H3 was trained on clips of about **5–15 s** (124–362 frames). Longer clips work (30 s and 40 s have been run), but are outside training. In the second half, watch for repeating or stalled motion, identity drift and audio drifting out of sync.

### GPU
The RTX PRO 6000 Blackwell (the sparse kernel is tuned for it) or H100 should be fastest. The A100 80 GB is tested. The model, text encoder and VAEs need about 45 GB, plus room for long clips: use 80 GB or more.

### Other speed levers
- **Resolution**: 960×544 has about 11 % fewer tokens than 1024×576. 832×480 has about two-thirds as many, so attention takes about half the time.
- **Steps**: Eros is distilled for 8; fewer steps lose detail and motion.
- **CFG above 1** runs a second pass every step (about 2× slower). Leave it at 1 unless you need the negative prompt.

## Pod settings

| Variable | Default | Meaning |
|---|---|---|
| `MODEL_MODE` | `all` | `all` installs every workflow (about 63 GB). `eros` installs only Eros (about 42 GB); `standard` only base H3. |
| `HF_TOKEN` | Runpod secret | Optional, for gated Hugging Face downloads. |
| `COPYPARTY_USERNAME`, `COPYPARTY_PASSWORD` | Runpod secrets | Enable the read-only output file browser on port 3923. Without both it stays off. |
| `COMFYUI_ATTENTION` | `pytorch` | Attention for the text encoder and VAEs. The H3 model's attention is set by the workflow. `sage` is an opt-in experiment with a boot-time GPU check. |
| `COMFYUI_FAST` | off | ComfyUI `--fast` features (`fp16_accumulation fp8_matrix_mult cublas_ops autotune`). Upstream labels them potentially quality-reducing. |
| `COMFYUI_RESERVE_VRAM` | unset | VRAM in GB that ComfyUI keeps free. |
| `VERIFY_MODELS` | unset | `full` re-hashes every model on this boot instead of trusting the verification record. |
| `OMP_NUM_THREADS`, `MKL_NUM_THREADS` | from the CPU quota | Thread overrides. Generation is GPU-bound, so these rarely matter. |
| `COMFYUI_DIR` | `/workspace/ComfyUI` | Where models, inputs, outputs and user data live. |

Protect access to port 8188 through your Runpod proxy and access settings.

## Storage and updates

- **What persists:** models, inputs, outputs and workflows live under `/workspace/ComfyUI`. They survive restarts, but are lost when the pod is deleted unless `/workspace` is a persistent volume. The image contains no weights or credentials.
- **Model checks:** every model is pinned by revision, size and SHA256. Each file is verified once; later boots trust the record while the file is unchanged.
- **Workflows are never overwritten.** Your edits survive restarts and image updates. To get an updated workflow, delete or rename your copy and restart. A volume from an older image may also hold Turbo workflows and adapter LoRAs that no longer work; delete them.
- **Pods don't update themselves.** A running pod keeps the image it started with. Deploy a new pod from the template to get the latest `develop` build.

## Development

- **Regenerate workflows:** `python3 scripts/build-workflows.py`. Generation is deterministic and offline; commit the generated files. The canvas note is copied from this README's *Speed and length* section, so regenerate after editing it (CI fails if you forget).
- **Tests:** `python3 tests/test-bundle.py`, `python3 tests/test-hub.py` and `bash tests/test-cpu-threads.sh`.
- **CI** (`.github/workflows/build.yml`):
  - runs on every push to `develop`;
  - runs the source checks and builds the image;
  - starts ComfyUI on the CPU to validate every API graph against the real node schemas (`scripts/smoke-schemas.py`, no weights needed);
  - then publishes the model tag and `sha-<commit>`.
- **Runpod template:** `python3 scripts/create-runpod-template.py` prints the payload from `config/runpod-template.json`. Add `--apply` (with `RUNPOD_API_KEY`) to create or update the private template. It only accepts the model tag, never a digest or `sha-` tag. For a private registry, pass `--registry-auth-id`.
- **New model generation:** add it as its own `versions/<version>/` profile with its own image tag. `VIDEO_VERSION` identifies the image; it cannot switch models.

More detail: [research and design decisions](docs/research.md), [validation record](docs/VALIDATION.md).

## Runpod Hub

Hub serves the ComfyUI HTTP/WebSocket interface on port 8000; readiness is on 8001. Its readiness check does not generate a video.
