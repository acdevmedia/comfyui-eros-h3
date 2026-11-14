## Speed and length

### Why long clips are slow
H3 turns every ~0.7 s of video (17 frames) into 5 latent frames, and every latent frame at 1024×576 into 576 tokens. A 30 s clip is about 125,000 tokens. Each sampling step compares every token with every other one (attention), so **doubling the length roughly quadruples the attention time**. Text encoding and decoding barely matter; nearly all the time is the 8 sampling steps.

Measured on an A100 80 GB at 1024×576 with full (dense) attention: **15 s ≈ 6 min, 40 s ≈ 31 min** (≈ 3.9 min per step).

### What this workflow does by default
Both settings are inside **02 · Shot settings** (double-click it). They are applied in this order: model → H3 sampling shift → **Fast attention** → **Sparse attention**, the same chain as Comfy-Org's own FastH3 template.

**Fast attention (int8)**: *Model Attention Backend* set to *comfy kitchen attention*. Attention is computed with 8-bit quantised queries and keys (SageAttention-style) instead of 16-bit. It runs on every step that sparse attention does not handle. The quality difference is normally not visible.

**Sparse attention (long clips)**: *Model Sparse Attention*, method *sol-attn*. For each block of 64 tokens, the kernel measures which other blocks actually matter and computes those exactly. It skips the rest, plus keeps 256 extra top-scoring tokens. It needs no special model weights.
- **tau 1.3**: how selective it is. Higher is sparser and faster. Per the node's guidance, 1.0 keeps ~16 % of blocks, 1.5 ~7 %, 2.0 ~2.7 %.
- **start_percent 0.2**: the first 20 % of the noise schedule stays dense. With Eros (8 steps, shift 12) that is steps 1–2, which set composition and motion. Standard (20 steps) keeps steps 1–4 dense.
- **sink_conditioning exact_kv_and_rows**: the prompt, starting image and audio stay exact, and the audio is computed densely, so speech and sound are not thinned.
- **min_tokens 12,288**: very small clips stay dense, where sparse would not help.

The gain grows with length. The estimate is about **2–2.5× faster at 30–40 s** on an A100, and more on Blackwell, where the kernel is tuned. These are estimates, not measurements yet.

### Trading speed against quality
- **Softer detail, shimmer or flicker?** Lower tau to 1.0, or raise start_percent to 0.3–0.4.
- **Want more speed?** Raise tau to 1.5. Check fine texture and faces.
- **Turn it off for an A/B**: select *Sparse attention* and bypass it (Ctrl+B), and set *Fast attention* to *pytorch attention*. Keep the same seed.

### Length
Duration has no upper limit; the default is 30 s. H3 was trained on clips of about **5–15 s** (124–362 frames). Longer clips do generate (40 s has been run), but are outside training. In the second half, watch for repeating or stalled motion, identity drift and audio drifting out of sync.

### GPU
The RTX PRO 6000 Blackwell (the kernels are tuned for it) or H100 should be fastest. The A100 works. The model, text encoder and VAEs need about 45 GB, plus room for long clips: use 80 GB or more.

### Other speed levers
- **Resolution**: 832×480 has about two-thirds the tokens of 1024×576, so attention takes about half the time.
- **Steps**: Eros is distilled for 8; fewer steps lose detail and motion.
- **CFG above 1** runs a second pass every step (about 2× slower). Leave it at 1 unless you need the negative prompt.
