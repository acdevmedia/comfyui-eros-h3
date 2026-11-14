# Validation status

Checked 13 November 2026.

Container build and all CPU smoke checks passed for release candidate `v0.1.0-rc.2`: [GitHub Actions](https://github.com/acdevmedia/comfyui-eros-h3/actions/runs/43521507064). Both image installation and ComfyUI startup succeeded. Every packaged API workflow passed live node-schema validation. Authenticated output listing, downloads, thumbnails, ZIP downloads and read-only access checks passed; zip/unzip and AWS CLI are installed, and `aws --version` succeeds. Source control/frame-grid/seed/hash-cache tests and CPU thread quota tests also passed.

Release images are public in GHCR; anonymous manifest access was verified. A private Runpod test template has been created; it follows the model-version tag (it was briefly digest-pinned). Creating the template does not start a Pod or incur GPU usage.

Image digest: `ghcr.io/acdevmedia/erosh3-comfyui@sha256:f0da72fdc0fae4613893aa86520f3fac5d45af629000de7eaa0906fd46023041`.

GPU smoke test, 13–14 November 2026, Runpod A100 80GB PCIe (driver 595.91.07), rc.2 image with the custom_nodes boot fix applied through the Pod entrypoint (the rc.2 image itself fails at ComfyUI startup; fixed in source since). PyTorch attention, API workflows, 1344×768, 5 s requested (124 frames, 24 fps, AAC audio):

| Workflow | Result | Execution | Sampled peak VRAM |
|---|---|---|---|
| eros I2V, cold | success | 168.6 s (8 steps × 17.8 s) | ~43 GB |
| eros I2V, warm re-roll | success | 158.9 s (encodes cached) | ~45 GB |
| eros T2V | success | 147.6 s | ~45 GB |
| turbo4 I2V | success | 95.1 s | ~44 GB |

Visual check of eros I2V frames: identity and composition held, prompted motion and push-in present, no flicker seen. Audio present without clipping; speech intelligibility not yet judged by ear. The turbo4/turbo8 modes have since been removed (10Eros has turbo merged in). turbo8 and standard (CFG 1) were stopped unfinished when the Pod was terminated. The new subgraph UI workflows were not loaded in the GPU session.

Since validated on GPU: weight loading, generation, timing and warm re-roll caching (above and below). Still open: a minimum-VRAM figure below 80 GB, reference-image fidelity scoring, and H3 audio/lip synchronization judged by ear. Public template/Hub submission remains deferred.

## Speed defaults on GPU (14 November 2026)

Runpod A100-SXM4-80GB, image `sha-50038b9`, Eros text-to-video, int8 attention + sol-attn sparse attention (tau 1.3, start 0.2). The log confirmed the sparse path ran ("sparse producer path"); steps 1–2 stayed dense as designed.

| Run | Result |
|---|---|
| 10 s, 1024×576, seed 424242, default | 98 s total; sampling 81 s (2 × 15 s int8 dense + 6 × 8.5 s sparse) |
| Same, fully dense PyTorch attention | 160 s total; sampling 145 s (8 × 18 s) |
| 30 s, 1024×576, default (stopped after 3 steps) | 105 s per int8 dense step, 44 s per sparse step |
| 30 s, 960×544, default (user runs, repeated) | about 7.5 min, consistently |
| Earlier dense baseline, 1024×576 | 15 s ≈ 6 min; 40 s ≈ 31 min (232 s per step) |

Same-seed 10 s comparison, five frame pairs: composition, faces, raincoat texture, reflections and story beats matched in quality. The clips differ in detail, as expected when attention changes. Motion (mean absolute frame difference) was 10.95 sparse against 10.66 dense. Audio was present in both (mean −27 / −29 dB, no clipping). One artefact: a flat, disc-shaped car splash at 3 s in the sparse clip. Not yet judged: long-clip drift past 15 s, and lip sync by ear.
