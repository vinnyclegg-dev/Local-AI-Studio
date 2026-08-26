# What else this studio could run — August 2026

*Companion to [MODEL_REVIEW_2026-08.md](MODEL_REVIEW_2026-08.md). That document asks whether
anything installed has been superseded. This one asks a different question: what **kinds** of
model is the studio not running at all, and what could the models already on disk do if
pointed at a different job?*

Nothing here is installed or applied. Same hardware envelope: **16 GB VRAM, one heavy model
resident at a time.**

---

## Part 1 — Gaps worth filling

Ranked by how much they unlock relative to the work.

### 1. An upscaler — the biggest single gap in the stack

**There is no upscaler anywhere in this studio.** Not for images, not for video. That is the
most conspicuous absence, and it compounds a limit the Video tab already documents: H3 stops
animating below its native resolution, so renders are hard-capped at a **768 px short edge**.
Everything the Video tab produces is, by construction, sub-HD.

**SeedVR2** is the answer the ComfyUI ecosystem settled on in 2026: a **one-step** diffusion
restoration model (adversarially trained against real data) that upscales and restores in a
single pass rather than iterating. It handles 480p/720p → 1080p/4K, and its batch processing
(45–200 frames) is designed specifically to avoid the temporal flicker that kills naive
per-frame upscaling.

- **Fits:** the 7B runs on 16 GB using **BlockSwap**; there is also a 3B and FP8 variants.
- **Serves three tabs, not one:** H3 clips, FLUX stills, and sprite sheets.
- **Why it matters most:** it changes the Video tab's status from "draft generator" to
  "produces something deliverable". Nothing else on this list has that effect.

### 2. Sound effects — the missing third leg of game audio

The studio generates **music** (ACE-Step) and **voice** (Kokoro/Chatterbox/Zonos/XTTS). For a
studio whose Sprite tab is explicitly aimed at game assets, the absent category is **SFX and
foley** — footsteps, impacts, UI clicks, ambience.

Two candidates, and the licence difference decides it:

| | Licence | Modes | ComfyUI |
|---|---|---|---|
| **Sony Woosh** | Open weights, **non-commercial** | Flow/DFlow (text→audio), **VFlow/DVFlow (video→audio)** | Official nodes |
| **Stable Audio Open** | Commercial-safe, fine-tunable | text→audio, up to 180 s | Community |

Woosh's **video→audio** mode is the interesting one: point it at a silent H3 clip and it
generates matching foley. Stable Audio Open is the one to build on if output is ever meant to
ship, and it is popular with game studios precisely because it can be fine-tuned on a custom
sound library.

*(Also noted: `LTX-2.3-22b-LoRA-Foley-V2A` is a dedicated video→audio adapter, but it pulls in
the LTX stack this studio doesn't otherwise run.)*

### 3. Image → 3D

**TRELLIS 2** turns a single image into a textured 3D asset inside ComfyUI. It is the only
top-tier image-to-3D option that is both open-source **and** free for commercial use — Hunyuan3D
2.1 is open but has historically trailed on texture fidelity.

The fit here is unusually neat: Sprite Studio already takes **one reference image** and
chains it through ReferenceLatent for consistency. The same input produces a 3D asset with no
new user-facing concept. Originally a 24 GB model, community quantised workflows have brought
it down to **8 GB**, so it fits comfortably.

### 4. An embedding model — the cheapest thing on this list

The studio has produced thousands of artifacts — stories, compositions, lyrics, prompts,
transcripts, sprite sets, videos — across `stories/`, `compositions/`, `audiobooks/`,
`lullabies/`, `sprites/`, `videos/` and `voices/`. **None of it is searchable.**

An embedding model fixes that for almost no VRAM:

| Model | Size | Note |
|---|---|---|
| `nomic-embed-text` | ~0.3 GB | Best starting point for English; long context, fast on CPU; most-pulled on Ollama |
| Qwen3-Embedding-0.6B | ~1.5 GB | 70.7 MTEB-eng-v2; multilingual, instruction-aware, flexible dimensions |
| BGE-M3 | ~2 GB | If you want hybrid dense+sparse over 100+ languages |

**Why this is architecturally free:** at 0.3 GB it never needs to claim the Manager's
one-model slot, so it can stay resident alongside whatever else is loaded. It could even run
on CPU. This is the lowest-cost, highest-leverage addition on the list.

### 5. Speech enhancement — improves what's already installed

Voice Studio (XTTS fine-tuning) and Chatterbox cloning both live or die on the quality of
their **reference clips**, which come from a `refs/` folder of user recordings — i.e. whatever
microphone and room the user had.

**Resemble Enhance** runs two modules: a denoiser that separates speech from noise, and an
enhancer that repairs distortion and **extends bandwidth**. **DeepFilterNet** is the lighter,
lower-complexity full-band alternative.

Run as an automatic pass on upload, this improves every clone downstream without the user
doing anything. It is a small model doing quiet work at the front of an existing pipeline —
the kind of addition that is easy to overlook and disproportionately effective.

---

## Part 2 — Repurposing: models doing something other than their label

This is the more interesting half. Several of these need **no new model at all**.

### 1. RIFE frame interpolation → roughly halve H3 render time ⭐

The strongest idea in this document.

H3 costs about **3 minutes of compute per second of video** on this card — ~31 minutes for a
10-second clip. The cost scales with frame count, and `h3gen.py` constrains frame count to
`% 17 == 5` by the latent grid, at a fixed 24 fps.

**RIFE** (Real-Time Intermediate Flow Estimation) synthesises intermediate frames between
existing ones. The standard use is making 24 fps footage smoother. The repurposing: **generate
at half the frame count and interpolate 2× back up to 24 fps.** Mature ComfyUI nodes already
exist (`ComfyUI-Frame-Interpolation`, rife47/rife49), with fp16/bf16 precision options and
optional `torch.compile`.

**This is exactly the case the "validate speedups on output" rule exists for.** Interpolation
fails on fast motion, occlusion and cuts — the very content people generate video for. A
wall-clock halving means nothing if the artifact degrades. A/B at a fixed seed on a
motion-heavy prompt, judge the video, then decide. If it only holds for slow/static shots,
it is still worth having as an opt-in mode rather than a default.

Secondary use with no risk attached: interpolate a *finished* 24 fps clip up to 48/60 fps for
smoothness, which is the conventional application.

### 2. Depth Anything V3 → parallax game backgrounds and 2D lighting

Depth estimation is normally a step inside a 3D or ControlNet pipeline. Used directly, one
generated image becomes a **layered parallax background**: depth map → threshold into
foreground/mid/background planes → scroll at different rates. That is a standard 2D game
technique that normally requires an artist to draw each layer separately.

The V3 ComfyUI nodes also output confidence maps, ray origin/direction, camera intrinsics and
sky masks. The sky mask alone is a free background separation, and the depth gradient gives
**normal maps for dynamic 2D lighting** on sprites.

Net effect: extends Sprite Studio from *characters* to *whole scenes*.

### 3. SAM 3 → auto-slice sprites into rig-ready parts

Sprite Studio currently produces frame-by-frame animation — every frame a fresh FLUX
reference-edit. The alternative production method, skeletal animation, needs a character
**separated into parts**.

SAM 3 does **text-prompt** segmentation. Prompting "arm", "head", "leg" against a generated
sprite yields exactly those separable pieces, output as masks compatible with Impact Pack
SEGS — and ComfyUI's Impact Pack is *already installed*. That turns one generated sprite into
a rig, rather than nine into a flipbook.

### 4. The vision model you already have → automatic best-of-N

Image generation is inherently best-of-N: generate four, pick one. The studio already loads
a vision-language model for the Language tab.

Point it at its own output — score each candidate against the generating prompt and surface
the winner first. No new model, no new download; a scoring loop around an existing worker.
The same pattern extends to Sprite Studio's per-frame re-roll (which frame broke consistency?)
and, with an audio-capable model, to picking between music takes.

### 5. Demucs → sound-effect isolation from footage

Demucs is installed for music separation. Applied to a **film or gameplay clip**, the "other"
stem is effectively ambience and foley with the music and dialogue pulled out — a free sample
library from footage already on disk. `htdemucs_6s` splits six ways, giving even finer
extraction. No new model, no new environment: `lullabykit/.venv` already has it.

### 6. Parakeet + an embedding model → full-text search over everything

Combine Part 1 §4 with the STT model already installed. Every audiobook, every generated
clip's dialogue, every voice note becomes text; embed it alongside the story and composition
JSON, and the studio gains semantic search across its entire history. As a by-product, the
same transcripts give **automatic captions** for video output.

### 7. basic-pitch + FCPE → pitch-correct cloned vocals

`lullabykit/pipeline.py` uses basic-pitch to transcribe melody and FCPE to trace a continuous,
unquantised **f0 curve**. That curve is a pitch reference. Applied to a TTS or cloned vocal
instead of an analysed song, the same machinery becomes **pitch correction** — or, pointed at
a target melody, turns spoken narration into singing. Both models are already in the venv.

### 8. ACE-Step audio-to-audio → an SFX texture pass

ACE-Step is already used three separate ways as an audio-to-audio *polish* pass at low
denoise (Lullaby Remix, Melody Match polish, Composer polish). The same trick applied to a
raw or synthesised sound effect re-textures it — a cheap way to make placeholder SFX sit in
the same sonic world as generated music, using a model that is already wired and warmed.

---

## Part 3 — Deliberately not recommended

- **Lip-sync / talking head.** The LeapTalk avatar cam was built and then removed on
  2026-08-09 because the quality was not publishable. Nothing shipped since changes that
  verdict, and the HF cache still holds 1 KB stubs from the attempt.
- **Anything requiring more than 16 GB resident.** The Manager's one-model-at-a-time design is
  what keeps this studio predictable. A model that needs partial CPU offload does not just run
  slowly — it breaks the assumption the whole VRAM budget is built on.
- **A second image model.** FLUX.2 Klein covers generation, editing and sprites through one
  set of weights. Adding Qwen-Image for text-in-image would mean another 20 GB class model
  competing for the same slot, to fix a problem this studio does not have.

---

## Suggested order, if any of this is picked up

1. **Embedding model** — smallest, cheapest, doesn't compete for VRAM, and immediately makes
   the studio's own back catalogue useful.
2. **SeedVR2 upscaler** — largest single quality unlock, serves three tabs.
3. **RIFE experiment** — potentially halves the studio's slowest operation; cost is one
   afternoon and an honest A/B.
4. **Speech enhancement** — small, invisible, improves two existing tabs.
5. **SFX generation** — new capability, completes the game-audio set.
6. **Depth / SAM repurposing** — highest ceiling for Sprite Studio, most design work.
7. **Image → 3D** — genuinely new territory; do it when 2D is finished.

---

## Sources

- [SeedVR2 in ComfyUI (official docs)](https://docs.comfy.org/tutorials/utility/seedvr2) ·
  [ComfyUI-SeedVR2_VideoUpscaler](https://github.com/numz/ComfyUI-SeedVR2_VideoUpscaler)
- [Woosh SFX generation in ComfyUI](https://www.runcomfy.com/comfyui-workflows/woosh-sound-effect-generation-in-comfyui-text2audio-videosync) ·
  [LTX Foley V2A adapter](https://huggingface.co/Lightricks/LTX-2.3-22b-LoRA-Foley-V2A) ·
  [AI foley guide 2026](https://www.aimagicx.com/blog/ai-sound-effects-generation-foley-guide-2026)
- [TRELLIS 2 in ComfyUI](https://trellis2.app/blog/comfyui-3d-model-generator-microsoft) ·
  [Best image-to-3D models 2026](https://trellis2.app/blog/best-image-to-3d-models-huggingface)
- [Best Ollama embedding models 2026](https://www.morphllm.com/ollama-embedding-models) ·
  [Open-source embedding models (BentoML)](https://www.bentoml.com/blog/a-guide-to-open-source-embedding-models) ·
  [Local embedding models for RAG](https://www.recal.so/blog/best-local-embedding-models-rag-2026)
- [Resemble Enhance](https://github.com/resemble-ai/resemble-enhance) ·
  [ClearerVoice-Studio](https://arxiv.org/html/2506.19398v1)
- [ComfyUI-Frame-Interpolation](https://github.com/Fannovel16/ComfyUI-Frame-Interpolation) ·
  [RIFE VFI node guide](https://www.instasd.com/comfyui/custom-nodes/comfyui-frame-interpolation/rife-vfi)
- [ComfyUI-DepthAnythingV3](https://github.com/PozzettiAndrea/ComfyUI-DepthAnythingV3) ·
  [ComfyUI-TBG-SAM3](https://github.com/Ltamann/ComfyUI-TBG-SAM3)
