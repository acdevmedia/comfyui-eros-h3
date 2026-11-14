# Validation status

Checked 13 November 2026.

Container build and all CPU smoke checks passed for release candidate `v0.1.0-rc.2`: [GitHub Actions](https://github.com/acdevmedia/comfyui-eros-h3/actions/runs/43521507064). Both image installation and ComfyUI startup succeeded. Every packaged API workflow passed live node-schema validation. Authenticated output listing, downloads, thumbnails, ZIP downloads and read-only access checks passed; zip/unzip and AWS CLI are installed, and `aws --version` succeeds. Source control/frame-grid/seed/hash-cache tests and CPU thread quota tests also passed.

Release images are public in GHCR; anonymous manifest access was verified. A private, digest-pinned Runpod test template has been created. Creating the template does not start a Pod or incur GPU usage.

Image digest: `ghcr.io/acdevmedia/erosh3-comfyui@sha256:f0da72fdc0fae4613893aa86520f3fac5d45af629000de7eaa0906fd46023041`.

GPU smoke test, 13–14 November 2026, Runpod A100 80GB PCIe (driver 595.91.07), rc.2 image with the custom_nodes boot fix applied through the Pod entrypoint (the rc.2 image itself fails at ComfyUI startup; fixed in source since). PyTorch attention, API workflows, 1344×768, 5 s requested (124 frames, 24 fps, AAC audio):

| Workflow | Result | Execution | Sampled peak VRAM |
|---|---|---|---|
| eros I2V, cold | success | 168.6 s (8 steps × 17.8 s) | ~43 GB |
| eros I2V, warm re-roll | success | 158.9 s (encodes cached) | ~45 GB |
| eros T2V | success | 147.6 s | ~45 GB |
| turbo4 I2V | success | 95.1 s | ~44 GB |

Visual check of eros I2V frames: identity and composition held, prompted motion and push-in present, no flicker seen. Audio present without clipping; speech intelligibility not yet judged by ear. turbo8 and standard (CFG 1) were stopped unfinished when the Pod was terminated. The new subgraph UI workflows were not loaded in the GPU session.

Not yet validated: GPU weight loading/generation, peak VRAM, timing, warm re-roll caching on GPU, reference fidelity, frontend playback, and H3 audio synchronization. No GPU performance or minimum-VRAM claim follows from these CPU results. Public template/Hub submission remains deferred until GPU tests pass.
