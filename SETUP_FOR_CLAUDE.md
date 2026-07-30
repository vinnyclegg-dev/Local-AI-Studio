# Local AI Studio — Self-Contained Setup Runbook (for Claude Code)

> **Read this first.** You are Claude Code. This single Markdown file is a complete, self-contained
> installer for the **Local AI Studio** — a private, local, zero-API-cost AI workstation that runs
> Large-Language, Image, Music, Speech-to-Text, Text-to-Speech and voice-cloning models on the user's
> own NVIDIA GPU, exposed through one tabbed web app. **Everything you need is in this file**: the full
> source of every program is embedded verbatim in the *EMBEDDED SOURCE FILES* section at the bottom.
> Follow the phases in order. Do not fetch any external repository for the application code — write the
> embedded files to disk exactly as given.
>
> This stack is **self-contained and has nothing to do with any earlier project** the user may have
> tried before. Build only what is described here.

## What you are building

A local web app at **http://127.0.0.1:8800** with these tabs / features:

| Tab | What it does | Backend |
|---|---|---|
| 🧠 Language | Code / research / vision prompts to a local LLM, with a 🔓 Unlocked (uncensored) option | Ollama |
| 🎨 Image — Generate | Text→image (FLUX.2 Klein) | ComfyUI |
| ✂️ Image — Edit | Reference-guided edit / remove / reframe / outpaint | ComfyUI |
| 🕹️ Sprite Studio | One reference image → style-matched 2D game sprites: single actions (9 built-ins with hand-written pose scripts, or a custom motion) or a full animation set (idle/walk/run/jump/fall/crouch/attack/hurt/death). True transparent backgrounds (rembg), per-action strips + combined sheet with engine-ready JSON, per-frame ↻ re-roll with automatic backups; sets persist in `sprites\` | ComfyUI (FLUX.2 ReferenceLatent) + `spritekit.py` in the ComfyUI venv |
| 🎼 Composer | A brief → a finished, arranged, mixed instrumental, as a 3-step wizard (**Set up → Arrange → Export**). The local LLM plans the *musical direction* (instruments + role, key, tempo, section structure, chord progression per section, mix and automation moves); `composerkit.py` writes the actual notes, because note-level output from a 20B local model isn't reliable enough to listen to. Instruments come from the bundled SoundFonts played by FluidSynth — free, already on disk, and the GM bank fits the 800 MB "plugin budget"; piano tracks are routed to the sampled Salamander Grand. Every track gets its own stem and channel strip: saturation, tone shelves, a time-varying lowpass, tempo-synced ping-pong delay and convolution reverb with **automated** send levels, plus volume/pan automation, risers, sub impacts, downlifters, drop silences and kick sidechaining. A **performance pass** (within-note CC11 swells, guitar strums/piano rolls, correlated timing drift with per-role feel, drum round-robin) is what stops it sounding like MIDI. Step 2 is a DAW-style editor: clip grid per track × section, channel strips, chord/energy/FX editing, and a **piano roll** for hand-editing notes — edits re-render in place with no LLM call. Optional ACE-Step re-texture pass ships *alongside* the clean master for A/B. Outputs master MP3/WAV, per-instrument FLAC stems, multitrack MIDI with pan/volume CCs, plus `score.json` and `arrangement.md` | Ollama (planning) + `composerkit.py` (FluidSynth + numpy/scipy, CPU) |
| 🎵 Music Generation | Full songs & instrumentals from style tags + lyrics (**ACE-Step 1.5 XL**, hybrid planning-LM + DiT; **turbo default** for smooth vocals + speed, sft selectable for max prompt adherence). Structure/vocal/energy lyric-tag toolbar, BPM/Key/time-signature (Auto = model decides), planner creativity/guidance sliders, annotated sampler/scheduler picks, auto-retry on degenerate seeds, -1 dB peak limiting; remix mode | ComfyUI (git checkout) |
| 🎹 Lullaby | Any song → a soft lullaby instrumental. A workbench splits the song into 6 stems (vocals/guitar/piano/other/bass/drums) with scrubbable previews so you choose what carries into the result, then one of three engines: **Remix** (default — drums dropped, dynamics flattened, then ACE-Step audio2audio re-imagines it with lullaby tags at a user-set denoise; closest to the original) · **Piano** (melody transcribed + key/chords detected, rebuilt as a rocking piano + music-box arrangement at 55–88bpm on a sampled grand) · **Melody Match** (traces each sung NOTE's actual continuous pitch curve via FCPE — no scale-snap/quantization, only real note boundaries — onto a portamento-capable instrument: cello/violin/flute/synth voice/music box; per-track **Route** selector lets some ticked stems go through Melody Match while others go through a full Piano-style rebuilt arrangement in the same render, mixed together; optional ACE-Step polish pass on the result) | `lullabykit` (2-pass Demucs + basic-pitch/FCPE + librosa + FluidSynth) + ACE-Step |
| ✂️ Track Splitter | Any song → its 6 individual instrument tracks (vocals/guitar/piano/other/bass/drums), each with its own scrubbable player and download, a "download all" zip, and a persistent library of past splits — shares its separation cache with the Lullaby tab | `lullabykit` (Demucs) |
| 🎙️ Speech → Text | Transcribe audio | NeMo Parakeet (conda env) |
| 🔊 Text → Speech | Narration (Kokoro) and voice-cloning (Chatterbox) | conda envs |
| 🗣️ Voice Studio | Fine-tune & reuse a personal XTTS-v2 voice | conda env |
| 📖 Story Maker | Timeline-driven multi-scene story generation | koboldcpp (Cydonia-24B) / Ollama |
| 📚 Audiobook | Story project or pasted text → chaptered MP3s with any loaded narrator. Natural pacing (sentence/paragraph/chapter pause tiers), engine-aware chunk sizes, click-free joins, one consistent XTTS narrator (pinned seed), EBU R128 loudness normalization (−19 LUFS) | loaded TTS worker + ffmpeg |

Plus two control tools: **`studioctl.ps1`** (CLI) and **`studio_gui.pyw`** (a visual control panel) that
start / stop / monitor the whole stack (Ollama + ComfyUI + the studio server), and a **Desktop shortcut**
(created in Phase 6) that launches the visual control panel with one click.

## Hard requirements & ground rules

- **OS:** Windows 10/11 (the control tools, conda envs and install paths assume Windows).
- **GPU:** an NVIDIA GPU with a current driver. **~16 GB VRAM** is the design target. The studio loads
  **one heavy model at a time** — this is by design, never run two large models concurrently.
- **Disk:** ~**95 GB** free for the required model set (~**115 GB** with the optional uncensored image
  model and Unlocked LLM). The largest items: ACE-Step 1.5 files ~20 GB, gpt-oss:20b ~14 GB,
  Cydonia GGUF ~13 GB, FLUX.2 Klein ~12 GB, ComfyUI venv ~8 GB, `lullabykit` ~5 GB (its own
  torch/Demucs/basic-pitch venv + bundled FluidSynth + soundfonts).
- **Paths are portable.** Everywhere below, `%USERPROFILE%` is the current user's home
  (PowerShell: `$HOME`). Never hard-code another machine's user folder.
- **Idempotent.** Every step is gated on a check — re-running this runbook must not damage an existing
  install. If a component is already present and healthy, skip its install.

---

## PHASE 0 — Preflight detection

Run these checks and record what is already present (install the rest in Phase 1):

```powershell
nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv,noheader   # GPU present?
where.exe conda; conda --version                                                 # Miniconda?
& "$env:LOCALAPPDATA\Programs\Ollama\ollama.exe" --version                       # Ollama?
Test-Path "$env:USERPROFILE\comfyui-src\main.py"                                  # ComfyUI checkout? (Phase 4 clones it)
where.exe ffmpeg                                                                  # ffmpeg (optional, STT convenience)
where.exe git                                                                     # git
& "$env:ProgramFiles\Tailscale\tailscale.exe" version                            # Tailscale (optional, remote access)
```

> **tkinter note:** the default python.org Python usually ships **without tkinter**, which the visual
> control panel needs. You do **not** need to fix this — the launcher `Studio Control Panel.cmd`
> automatically selects a tkinter-capable Python from the conda envs you create in Phase 3.

---

## PHASE 1 — Install missing prerequisites

Install only what Phase 0 reported as missing. Use winget where possible.

```powershell
winget install --id Anaconda.Miniconda3 -e --accept-source-agreements --accept-package-agreements
winget install --id Ollama.Ollama       -e --accept-source-agreements --accept-package-agreements
winget install --id Gyan.FFmpeg         -e --accept-source-agreements --accept-package-agreements
winget install --id Git.Git             -e --accept-source-agreements --accept-package-agreements
winget install --id Tailscale.Tailscale -e --accept-source-agreements --accept-package-agreements   # optional
```

> **ComfyUI Desktop is NOT required.** The studio runs ComfyUI **headless from a git checkout**
> (Phase 4) with its data directory at `%USERPROFILE%\Documents\ComfyUI` — that folder is created by
> Phases 2/5 and ComfyUI creates its `input/output/user` subfolders on first start. Install the Desktop
> app (`winget install --id Comfy.ComfyUI-Desktop -e`, data dir set to the same `Documents\ComfyUI`) only if the
> user also wants the node-graph GUI; it shares the models but is otherwise independent.

---

## PHASE 2 — Write all application source files

Create the folder tree, then write **every file** from the *EMBEDDED SOURCE FILES* section to the path
shown in its heading. Write the bytes **exactly** as given (do not reformat, re-indent, or "improve").

```powershell
$dirs = @(
  "$HOME\local-ai-studio",
  "$HOME\local-ai-studio\logs",
  "$HOME\local-ai-studio\voices",
  "$HOME\local-ai-studio\stories",
  "$HOME\local-ai-studio\sprites",
  "$HOME\local-ai-studio\compositions",
  "$HOME\local-ai-studio\lullabykit",
  "$HOME\.claude\skills\local-llm\scripts",
  "$HOME\.claude\skills\local-image\scripts",
  "$HOME\.claude\skills\local-stt\scripts",
  "$HOME\.claude\skills\local-tts\scripts",
  "$HOME\.claude\skills\local-voice\scripts",
  "$HOME\.claude\skills\local-music\scripts",
  "$HOME\Documents\ComfyUI\custom_nodes"
)
$dirs | ForEach-Object { New-Item -ItemType Directory -Force -Path $_ | Out-Null }
```

Now write each embedded file (use the Write tool; create parent folders if needed). The complete list is
in *EMBEDDED SOURCE FILES* — 27 files in total.

---

## PHASE 3 — Create the conda environments

Four conda envs back the audio / voice workers. Create each with the **exact** pins below (these match a
known-good install). Run from an Anaconda Prompt or after `conda init powershell`.

```powershell
# 1) XTTS-v2 voice fine-tuning + inference  (NON-COMMERCIAL: Coqui CPML, personal/artistic use only)
conda create -y -n xtts python=3.11
conda run -n xtts pip install torch==2.6.0+cu124 torchaudio==2.6.0+cu124 --index-url https://download.pytorch.org/whl/cu124
conda run -n xtts pip install coqui-tts==0.27.5 "transformers>=4.57,<5"
#   ^ pin transformers <5: coqui-tts imports isin_mps_friendly, which transformers 5.x removed.

# 2) Chatterbox TTS / voice cloning
conda create -y -n chatterbox python=3.11
conda run -n chatterbox pip install torch==2.6.0+cu124 torchaudio==2.6.0+cu124 --index-url https://download.pytorch.org/whl/cu124
conda run -n chatterbox pip install chatterbox-tts==0.1.7
conda run -n chatterbox pip install "setuptools==75.8.0"
#   ^ pin setuptools: the Perth watermarker still imports pkg_resources, dropped by newer setuptools.

# 3) NeMo Parakeet speech-to-text
conda create -y -n nemo-asr python=3.11
conda run -n nemo-asr pip install torch==2.6.0+cu124 torchaudio==2.6.0+cu124 --index-url https://download.pytorch.org/whl/cu124
conda run -n nemo-asr pip install "nemo_toolkit[asr]==2.7.3"

# 4) Kokoro TTS (fast narration)
conda create -y -n kokoro python=3.11
conda run -n kokoro pip install kokoro==0.9.4 soundfile
conda run -n kokoro python -m spacy download en_core_web_sm
```

> Each env is deliberately separate — their torch / transformers / setuptools requirements conflict and
> cannot share one environment.

---

## PHASE 4 — ComfyUI headless runtime (git checkout + venv + custom nodes)

The studio runs a **current git checkout** of ComfyUI (ACE-Step 1.5 needs a newer ComfyUI than the
Desktop app ships) with its data directory at `%USERPROFILE%\Documents\ComfyUI`, and uses two custom
nodes (already written in Phase 2): `ram_websocket_save.py` streams generated images back in-memory
(no disk writes) and `ace15_studio_encode.py` exposes ACE-Step 1.5's text encode with optional
bpm/key/timesignature (empty = the model decides).

```powershell
git clone --depth 1 https://github.com/comfyanonymous/ComfyUI.git "$HOME\comfyui-src"
$comfyCode = "$HOME\comfyui-src"
python -m venv "$comfyCode\.venv"
$venvPy = "$comfyCode\.venv\Scripts\python.exe"
& $venvPy -m pip install --upgrade pip
& $venvPy -m pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu129
& $venvPy -m pip install -r "$comfyCode\requirements.txt"
& $venvPy -m pip install rembg onnxruntime   # Sprite Studio: transparent-background cutout (spritekit.py)
```

The custom nodes load on the next ComfyUI start (Phase 6 launches it). Models go in Phase 5.

---

## PHASE 5 — Download the required models

Only the models the studio actually uses are listed. **Do not** download anything in the "NOT used" list
in Appendix A.

### 5a. Ollama LLMs (Language + Story Maker)

```powershell
& "$env:LOCALAPPDATA\Programs\Ollama\ollama.exe" pull gpt-oss:20b
& "$env:LOCALAPPDATA\Programs\Ollama\ollama.exe" pull qwen3-vl:8b
# Optional — the "Unlocked" (uncensored / abliterated) text model behind the Language
# tab's 🔓 Unlocked toggle and uncensored Story Maker prose. Same gpt-oss-20b weights
# with refusals removed, so quality matches the standard model. Configurable via the
# OLLAMA_UNLOCKED env var (default tag shown):
& "$env:LOCALAPPDATA\Programs\Ollama\ollama.exe" pull huihui_ai/gpt-oss-abliterated:20b
```

### 5b. ComfyUI FLUX.2 Klein image models

Download into the three model folders under `%USERPROFILE%\Documents\ComfyUI\models\`. The 4B base model
lives in a **gated** Black Forest Labs repo — if the download 401/403s, open the repo in a browser, accept
the license, then retry with a HF token (`huggingface-cli login`). **Verify each filename and that the file
is non-trivial in size after download** (a tiny file usually means an HTML error page).

```powershell
$m = "$HOME\Documents\ComfyUI\models"
New-Item -ItemType Directory -Force "$m\diffusion_models","$m\text_encoders","$m\vae" | Out-Null

# Diffusion model (4B base, fp8) -> diffusion_models\  (~3.9 GB)  [gated BFL repo]
curl.exe -L -o "$m\diffusion_models\flux-2-klein-base-4b-fp8.safetensors" `
  "https://huggingface.co/black-forest-labs/FLUX.2-klein-base-4b-fp8/resolve/main/flux-2-klein-base-4b-fp8.safetensors"

# Text encoder (Qwen3-4B) -> text_encoders\  (~7.7 GB)
curl.exe -L -o "$m\text_encoders\qwen_3_4b.safetensors" `
  "https://huggingface.co/Comfy-Org/flux2-klein-4B/resolve/main/split_files/text_encoders/qwen_3_4b.safetensors"

# VAE -> vae\  (~0.3 GB)
curl.exe -L -o "$m\vae\flux2-vae.safetensors" `
  "https://huggingface.co/Comfy-Org/flux2-dev/resolve/main/split_files/vae/flux2-vae.safetensors"
```

> **If any URL has moved**, find the current Comfy-Org / Black-Forest-Labs FLUX.2 Klein 4B files on
> Hugging Face (filenames `flux-2-klein-base-4b-fp8.safetensors`, `qwen_3_4b.safetensors`,
> `flux2-vae.safetensors`) and download those. The official guide is
> https://docs.comfy.org/tutorials/flux/flux-2-klein .

**Optional — second image model (uncensored, +~17 GB).** The Image tabs expose a `miraclein9b` option in
addition to the default `base4b`. It is a community **uncensored** FLUX.2 Klein 9B checkpoint and is
**NOT commercial-safe — personal/artistic use only**. Skip it unless the user wants that option. If
included, download `miraclein-9b-fp8.safetensors` into `diffusion_models\` and its matching encoder
`qwen_3_8b_fp8mixed.safetensors` into `text_encoders\` (locate current copies on Hugging Face; note
Civitai is geo-blocked in the UK, so prefer HF / CivArchive mirrors). The 4B and 9B encoders are **not**
interchangeable — a mismatch throws a `mat1/mat2 shapes` error.

### 5c. Speech / voice models (auto-download on first use)

These download themselves the first time their worker runs — just trigger each once:

```powershell
# Parakeet STT (~2.3 GB) downloads into the HF cache on first transcribe
conda run -n nemo-asr python "$HOME\.claude\skills\local-stt\scripts\transcribe.py" --help

# Kokoro voice pack (~0.3 GB)
conda run -n kokoro python -c "from kokoro import KPipeline; KPipeline(lang_code='a')"

# Chatterbox (~3 GB) — model pulls on first load
conda run -n chatterbox python -c "from chatterbox.tts import ChatterboxTTS; print('chatterbox import ok')"
```

### 5d. XTTS-v2 base + the two training-only files

XTTS auto-downloads `model.pth / config.json / vocab.json / speakers_xtts.pth` on first use, but **training
also needs `dvae.pth` and `mel_stats.pth`**, which the inference download skips. Fetch them manually:

```powershell
$tts = "$env:LOCALAPPDATA\tts\tts_models--multilingual--multi-dataset--xtts_v2"
New-Item -ItemType Directory -Force $tts | Out-Null
$env:COQUI_TOS_AGREED = "1"
# trigger base download:
conda run -n xtts python -c "import os; os.environ['COQUI_TOS_AGREED']='1'; from TTS.api import TTS; TTS('tts_models/multilingual/multi-dataset/xtts_v2')"
# fetch the two training extras:
curl.exe -L -o "$tts\dvae.pth"      "https://huggingface.co/coqui/XTTS-v2/resolve/main/dvae.pth"
curl.exe -L -o "$tts\mel_stats.pth" "https://huggingface.co/coqui/XTTS-v2/resolve/main/mel_stats.pth"
```

### 5e. Story Maker fiction backend — koboldcpp + Cydonia-24B (long-form novels)

Story Maker can generate on the Ollama LLM, but gpt-oss is a *reasoning* model: on long-form
prose it loops and leaks its chain-of-thought. For real novels the studio instead uses a
**separate koboldcpp server (`:5001`)** running a **creative fiction model** with the **DRY
sampler** (purpose-built to stop repetition). It's a normal one-model-at-a-time worker — loading
the "Story" model frees Ollama/ComfyUI from VRAM, and `server.py` launches/stops it on demand.

- **Model:** `TheDrummer_Cydonia-24B-v4.3` — a Mistral-Small-24B fiction finetune — quant
  **IQ4_XS (~12.8 GB)**, chosen so a 24B fits 16 GB with room for context (koboldcpp runs it with
  `--flashattention --quantkv 1` for an 8-bit KV cache → ~12k context).
- **Defaults** (override via env vars read by `server.py`): `KOBOLD_DIR=%USERPROFILE%\koboldcpp`,
  `KOBOLD_EXE=$KOBOLD_DIR\koboldcpp.exe`, `KOBOLD_MODEL=$KOBOLD_DIR\models\Cydonia-24B-v4.3-IQ4_XS.gguf`,
  `KOBOLD_CTX=12288`.

```powershell
$kobo = "$HOME\koboldcpp"; New-Item -ItemType Directory -Force "$kobo\models" | Out-Null
# 1) koboldcpp one-file CUDA build (~600 MB)
curl.exe -L -o "$kobo\koboldcpp.exe" `
  "https://github.com/LostRuins/koboldcpp/releases/download/v1.116.1/koboldcpp.exe"
# 2) the fiction model (~12.8 GB)
curl.exe -L -o "$kobo\models\Cydonia-24B-v4.3-IQ4_XS.gguf" `
  "https://huggingface.co/bartowski/TheDrummer_Cydonia-24B-v4.3-GGUF/resolve/main/TheDrummer_Cydonia-24B-v4.3-IQ4_XS.gguf"
```

The studio starts koboldcpp automatically when you click **Load story model** in Story Maker (it
holds ~14 GB while resident); **Stop model** / loading any other worker frees it. `server.py`
routes story generation through it (`story_write` → `kobold_chat` with DRY) and applies a
`_clean_prose` safety net (strips any stray repetition/meta). Manual launch, if ever needed:

```powershell
& "$HOME\koboldcpp\koboldcpp.exe" --model "$HOME\koboldcpp\models\Cydonia-24B-v4.3-IQ4_XS.gguf" `
  --usecublas --gpulayers 999 --contextsize 12288 --flashattention --quantkv 1 `
  --host 127.0.0.1 --port 5001 --skiplauncher --quiet
```

> **License:** Cydonia is a finetune of Mistral-Small (Apache-2.0 base). Model *output* is yours to
> use; treat the model as personal/artistic like the other creative models here.


### 5f. Audiobook module — ffmpeg + (optional) Zonos expressive TTS

The **Audiobook** tab turns a Story Maker project *or* pasted/uploaded text into chaptered **MP3s**
(one per chapter + a combined book), narrated by **whichever TTS voice is loaded** — Kokoro,
Chatterbox, a fine-tuned XTTS voice, or Zonos. The narration pipeline is tuned for natural pacing:
chunking is on sentence boundaries with **engine-aware chunk sizes** (Kokoro 600 chars, Chatterbox
280, Zonos 350, XTTS 240 — bigger chunks = fewer prosody resets), each chunk carries a **pause tier**
rendered as real silence (350 ms between sentences, 800 ms between paragraphs, 1.4 s at chapter ends),
workers apply **5 ms edge fades** (no clicks at joins), XTTS pins a **fixed seed per chunk** (one
consistent narrator across a whole book), and each chapter is **loudness-normalized to -19 LUFS**
(EBU R128, the audiobook standard) during the **ffmpeg** MP3 encode (ffmpeg required). Files are
written to `%USERPROFILE%\local-ai-studio\audiobooks\` and streamed at `/audiobooks/...`.

```powershell
# ffmpeg (required for MP3 export) — server.py auto-locates the WinGet install
winget install -e --id Gyan.FFmpeg --accept-package-agreements --accept-source-agreements
```

**Optional — Zonos (max-quality expressive narrator).** Zonos-v0.1 is a newer, very expressive TTS
(Apache-2.0). It is Linux-leaning; on Windows it needs **espeak-ng**. Use the **transformer** variant
(the hybrid needs `mamba-ssm`, painful on Windows). Skip this and use Kokoro/Chatterbox for zero fuss.

**Install from an editable clone** — the plain `pip install git+…Zonos.git` wheel *omits* Zonos's
`zonos.backbone` subpackage and then fails to import (`ModuleNotFoundError`); an editable install from a
clone includes it. Two further Windows issues are handled automatically by the studio's `zonos_tts.py`
(no manual library edits): it forces **eager mode** (Zonos's `torch.compile` needs MSVC `cl.exe`, absent
here) and builds the speaker-embedding **mel filterbank on CPU** (works around a torchaudio CPU/CUDA
device error in Zonos speaker cloning — without it every chunk gets a different random voice).

```powershell
# espeak-ng (phonemizer backend); server.py points PHONEMIZER_ESPEAK_LIBRARY at its DLL
winget install -e --id eSpeak-NG.eSpeak-NG --accept-package-agreements --accept-source-agreements

# dedicated conda env + PyTorch (cu124) + Zonos installed EDITABLE from a kept clone
conda create -n zonos python=3.11 -y
conda run -n zonos pip install torch==2.5.1 torchaudio==2.5.1 --index-url https://download.pytorch.org/whl/cu124
git clone https://github.com/Zyphra/Zonos.git "$HOME\zonos-src"
conda run -n zonos pip install -e "$HOME\zonos-src"   # editable → includes the backbone subpackage
```

Zonos sounds best with a **speaker reference** (also required for a *consistent* narrator): a clean
10–30 s WAV at `%USERPROFILE%\.claude\skills\local-tts\scripts\zonos_default_speaker.wav` is used as the
default narrator; or pass a per-book reference. Without one it falls back to a fixed-seed voice.

---

### 5g. Music tab — ACE-Step 1.5 XL (ComfyUI split files, turbo + sft)

The **Music** tab generates full tracks (vocals + instruments, or instrumental) with **ACE-Step 1.5
XL** (hybrid planning-LM + 4B DiT; MIT license; quality between Suno 4.5 and 5) running on the
existing ComfyUI backend — no extra env or server. Two DiT variants are installed (~30 GB total with
the shared encoders/VAE): **xl-turbo** (default — distilled, 8 steps, no CFG → smoother, artifact-free
vocals, ~6× faster) and **xl-sft** (50 steps + CFG 7 → strongest prompt adherence; selectable in the
Music tab's Advanced → Model variant):

```powershell
$M = "$HOME\Documents\ComfyUI\models"
New-Item -ItemType Directory -Force "$M\diffusion_models","$M\text_encoders","$M\vae" | Out-Null
$repo = "https://huggingface.co/Comfy-Org/ace_step_1.5_ComfyUI_files/resolve/main/split_files"
curl.exe -L -o "$M\diffusion_models\acestep_v1.5_xl_turbo_bf16.safetensors" "$repo/diffusion_models/acestep_v1.5_xl_turbo_bf16.safetensors"
curl.exe -L -o "$M\diffusion_models\acestep_v1.5_xl_sft_bf16.safetensors" "$repo/diffusion_models/acestep_v1.5_xl_sft_bf16.safetensors"
curl.exe -L -o "$M\text_encoders\qwen_0.6b_ace15.safetensors" "$repo/text_encoders/qwen_0.6b_ace15.safetensors"
curl.exe -L -o "$M\text_encoders\qwen_4b_ace15.safetensors" "$repo/text_encoders/qwen_4b_ace15.safetensors"
curl.exe -L -o "$M\vae\ace_1.5_vae.safetensors" "$repo/vae/ace_1.5_vae.safetensors"
```

## PHASE 6 — Launch & verify

```powershell
# Start the whole stack (Ollama -> ComfyUI headless -> Studio server), waiting for each to be healthy:
powershell -ExecutionPolicy Bypass -File "$HOME\local-ai-studio\studioctl.ps1" start
# Health:
(Invoke-WebRequest http://127.0.0.1:8800/health -UseBasicParsing).Content     # -> {"ok": true}
(Invoke-WebRequest http://127.0.0.1:8188/system_stats -UseBasicParsing).StatusCode  # ComfyUI 200
(Invoke-WebRequest http://127.0.0.1:11434/api/tags -UseBasicParsing).StatusCode     # Ollama 200
```

Then open **http://127.0.0.1:8800** and smoke-test each tab:
1. **Language** → Load → ask "say hello" (returns text).
2. **Image — Generate** → Load → 1 step, any prompt (returns an image).
3. **Sprites** → with the image model still loaded from step 2: drop in any character image,
   **Single action** → idle, frame size 256, Generate → 4 frames render one-by-one with live
   progress, then get transparent backgrounds + an `idle_sheet.png` strip (checkerboard behind the
   thumbs proves real alpha). Files land in `local-ai-studio\sprites\<set>\`.
4. **Text → Speech** → Load (Kokoro) → speak a line; then **Speech → Text** → Load → transcribe that wav.
5. **Voice Studio** loads without error.
6. **Story Maker** → **Load story model** (koboldcpp + Cydonia-24B; ~40s, frees other models) →
   Open a story → **Generate** writes chapter-by-chapter. Verify `:5001` answers:
   `(Invoke-WebRequest http://127.0.0.1:5001/api/v1/model -UseBasicParsing).StatusCode` (200).
7. **Music** → Load (warms ACE-Step 1.5 into VRAM) → add a style tag, tick **Instrumental**,
   set duration to 0:30 → Generate (~30–90 s) → an audio player appears (save via **⬇ Save track**).
   **Verify the track is audibly music, not near-silence** — musicgen.py auto-rerolls degenerate
   seeds, but a levels check (`ffmpeg -af volumedetect`: mean should be ≳ −25 dB) proves the chain.
8. **Audiobook** → with Kokoro still loaded from step 4, paste two short paragraphs → Render →
   chapter MP3 plays with natural sentence/paragraph pauses.

The visual control panel is **`Studio Control Panel.vbs`** (double-click — launches with no console
window at all; the `.cmd` variant does the same but flashes a batch console) — Start All / Stop All /
live status lights / GPU bar.

### Desktop shortcut

For one-click launch, create a Desktop shortcut to the visual control panel. This is idempotent — skip
if the shortcut and icon already exist and are healthy. `studio.ico` is embedded below as base64 (it's a
binary asset, so it isn't one of the text source files in *EMBEDDED SOURCE FILES*); decode it once,
then create the `.lnk`:

```powershell
$icoPath = "$HOME\local-ai-studio\studio.ico"
if (-not (Test-Path $icoPath)) {
    $icoB64 = @'
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
'@
    [IO.File]::WriteAllBytes($icoPath, [Convert]::FromBase64String(($icoB64 -replace "\s+", "")))
}
$shortcutPath = "$([Environment]::GetFolderPath('Desktop'))\Local AI Studio.lnk"
if (-not (Test-Path $shortcutPath)) {
    $sh  = New-Object -ComObject WScript.Shell
    $lnk = $sh.CreateShortcut($shortcutPath)
    $lnk.TargetPath       = "$HOME\local-ai-studio\Studio Control Panel.vbs"
    $lnk.WorkingDirectory = "$HOME\local-ai-studio"
    $lnk.IconLocation     = "$icoPath,0"
    $lnk.Description      = "Local AI Studio - start/stop/monitor the whole stack (visual control panel)"
    $lnk.Save()
}
```

> **Windows console-window rule:** Windows Terminal (the Win11 default) *ignores*
> `Start-Process -WindowStyle Hidden`, and uv-created venv pythons are launcher shims whose child
> interpreter allocates a fresh console under a DETACHED parent. Every service/subprocess spawn in this
> stack therefore uses **CREATE_NO_WINDOW** (`NOWIN` in server.py/musicgen.py, `Start-HiddenSvc` in
> studioctl.ps1, `spawn()` in studio_gui.pyw). Keep doing that in any new spawn code or cmd windows
> pop up while the studio runs.

---

## PHASE 7 — Optional remote access & shutdown

```powershell
# Remote HTTPS over your tailnet (no open ports), if Tailscale is set up:
powershell -ExecutionPolicy Bypass -File "$HOME\local-ai-studio\studioctl.ps1" start -Remote
# Stop the stack (leaves Ollama running by default):
powershell -ExecutionPolicy Bypass -File "$HOME\local-ai-studio\studioctl.ps1" stop
```

---

## PHASE 8 — Lullaby Studio kit (song → soft-piano lullaby)

The Lullaby tab needs its self-contained kit at `%USERPROFILE%\local-ai-studio\lullabykit\`:
the pipeline script (embedded above as `lullabykit\pipeline.py`), a private venv, bundled
FluidSynth binaries, and the FluidR3 GM soundfont. Nothing installs system-wide.

```powershell
$KIT = "$HOME\local-ai-studio\lullabykit"
New-Item -ItemType Directory -Force "$KIT\bin", "$KIT\soundfonts", "$KIT\output", "$KIT\work" | Out-Null

# 1) venv + Python deps (CUDA 12.9 torch; ~3.5GB download)
python -m venv "$KIT\.venv"
& "$KIT\.venv\Scripts\python.exe" -m pip install --upgrade pip
& "$KIT\.venv\Scripts\python.exe" -m pip install torch --index-url https://download.pytorch.org/whl/cu129
& "$KIT\.venv\Scripts\python.exe" -m pip install demucs pretty_midi mido soundfile "basic-pitch[onnx]" torchfcpe

# 1b) torchaudio shim (melody-match engine): torchfcpe's only torchaudio dependency is
#     transforms.Resample, but the only torchaudio wheel published for this torch build
#     (2.9.0+cu129) is 2.8.0+cu129 — a mismatched ABI whose compiled extension fails to
#     load. Rather than downgrade torch, drop in a local package that satisfies the one
#     import with a librosa-backed Resample and skips the real torchaudio wheel entirely.
$TA = "$KIT\.venv\Lib\site-packages\torchaudio"
New-Item -ItemType Directory -Force $TA | Out-Null
Set-Content "$TA\__init__.py" @'
"""Local shim: torchfcpe only needs torchaudio.transforms.Resample; the real
wheel available for this torch build has an incompatible compiled extension."""
'@
Set-Content "$TA\transforms.py" @'
import numpy as np, torch, torch.nn as nn, librosa

class Resample(nn.Module):
    def __init__(self, orig_freq, new_freq, lowpass_filter_width=64, **_ignored):
        super().__init__()
        self.orig_freq, self.new_freq = int(orig_freq), int(new_freq)

    def forward(self, waveform: torch.Tensor) -> torch.Tensor:
        if self.orig_freq == self.new_freq:
            return waveform
        device, dtype = waveform.device, waveform.dtype
        arr = waveform.detach().cpu().numpy().astype(np.float32)
        flat = arr.reshape(-1, arr.shape[-1])
        out = np.stack([librosa.resample(row, orig_sr=self.orig_freq,
                        target_sr=self.new_freq, res_type="soxr_hq") for row in flat])
        out = out.reshape(*arr.shape[:-1], out.shape[-1])
        return torch.from_numpy(out).to(device=device, dtype=dtype)
'@

# 2) FluidSynth (portable Windows build, unzipped in place)
Invoke-WebRequest -Uri "https://github.com/FluidSynth/fluidsynth/releases/download/v2.4.7/fluidsynth-2.4.7-win10-x64.zip" -OutFile "$KIT\bin\fluidsynth.zip"
Expand-Archive "$KIT\bin\fluidsynth.zip" "$KIT\bin\fluidsynth" -Force; Remove-Item "$KIT\bin\fluidsynth.zip"

# 3) FluidR3 GM soundfont (~148MB; used for the music-box / string-pad layers)
Invoke-WebRequest -Uri "https://github.com/pianobooster/fluid-soundfont/releases/download/v3.1/FluidR3_GM.sf2" -OutFile "$KIT\soundfonts\FluidR3_GM.sf2"

# 4) Salamander Grand Piano SF2 (~296MB download / 1.2GB extracted; CC-BY, real
#    sampled Yamaha C5, 16 velocity layers — this is the main piano sound. The
#    pipeline falls back to FluidR3's GM piano if it's missing.)
Invoke-WebRequest -Uri "https://freepats.zenvoid.org/Piano/SalamanderGrandPiano/SalamanderGrandPiano-SF2-V3+20200602.tar.xz" -OutFile "$KIT\soundfonts\salamander.tar.xz"
tar -xf "$KIT\soundfonts\salamander.tar.xz" -C "$KIT\soundfonts"; Remove-Item "$KIT\soundfonts\salamander.tar.xz"

# 5) smoke test (any local song/video file; writes <name>_lullaby.wav under lullabykit\output)
& "$KIT\.venv\Scripts\python.exe" "$KIT\pipeline.py" "C:\path\to\some\song.mp3"
```

Notes:
- The Lullaby tab has **three engines**, selected in the UI:
  - **Remix (default)** — closest to the original song. `LullabyJob` runs the
    pipeline through its `mix` stage only (`--until-stage mix`: extract → Demucs →
    drum-free stem mix), flattens the mix's dynamics with slow heavy compression
    (`REMIX_FLATTEN` — this is what keeps the loud second half of a song from
    dragging the lullaby with it; measured: +14dB energy rise without it, +0dB
    with), then ACE-Step audio2audio re-imagines it (turbo, lullaby tags,
    denoise 0.45-0.75 from the UI slider, default 0.55), and a soft mastering
    chain (`REMIX_MASTER`: -4dB high shelf, 10kHz lowpass, loudnorm -17 LUFS
    LRA 6, long fades) finishes it. Note: including the vocal stem here
    deliberately low-pass-filters it at 3.2kHz first (strips consonants/sibilance
    so ACE's regeneration doesn't produce "ghost vocal" artifacts) — selecting
    vocals-only won't sound like a clear melody, by design.
  - **Piano** — the symbolic v2 arrangement engine described below.
  - **Melody Match** — for when Piano's scale-snap + eighth-grid quantization
    mangles a florid or heavily-ornamented vocal line into something
    unrecognizable (measured on a real case: -0.03 F0 correlation with the
    original vocal — essentially unrelated). Traces the singer's actual
    continuous pitch curve with **FCPE** (`torchfcpe`, ~10ms resolution), then
    **segments it into distinct notes** (a rolling-median smoothing + rounded-
    semitone run detection, with sub-80ms runs merged into whichever neighbor
    is closer in pitch — this matters: an earlier version bent through an
    entire silence-delimited phrase as ONE note, which sounds like a siren/
    theremin wail, not a melody) — each note gets its own MIDI note-on, with
    only the small residual deviation (vibrato, slight scoop) carried as
    pitch-bend (±4 semitones). Plays back on one **portamento-capable**
    instrument (cello default, or violin/flute/synth voice/music box) — a
    fixed-pitch instrument like piano can't glide between notes at all, which
    literature backs as the wrong vehicle for tracing a vocal line. Round-trip
    validated (re-extract F0 from the render, correlate against the original
    vocal) at 0.91-0.99 correlation across two real songs away from note
    attacks, vs. Piano's -0.03 to ~0.6 on material it struggles with.
    **Per-track routing**: each ticked Tracks-panel stem has its own Route
    selector — `melody` stems are mixed together and traced onto the chosen
    instrument; `arranged` stems instead run through Piano's own
    transcribe+arrange pipeline (rebuilt, quantized) — e.g. vocals traced
    faithfully on cello while an existing piano part gets a proper rebuilt
    accompaniment, then both mixed into one render (`arrange()`'s bpm clamp is
    skipped for the arranged half here so it scales by the same tempo_scale
    as the traced half, rather than drifting apart over a long song). An
    optional ACE-Step polish pass (same mechanism as Piano's) can run over the
    combined result afterward.
- Pipeline stages (v2, the "musical arrangement engine"): ffmpeg audio extract →
  Demucs stem separation (GPU, transient) → music analysis (librosa beat grid + bar
  phase, Krumhansl-Schmuckler key detection, per-bar diatonic chord recognition via
  chroma template matching with a bass-stem root prior) + vocal-stem melody
  transcription (basic-pitch; falls back to the 'other' stem for instrumentals) →
  arrangement rebuilt on a clean grid at 55-88bpm (melody cleaned: monophony,
  octave-jump repair, scale snapping, eighth-note quantization, phrase-shaped
  velocities; rocking broken-chord left hand; music-box doubling; optional string
  pad; per-bar sustain pedal) → dual FluidSynth render (Salamander piano + FluidR3
  layers, large-room reverb) → ffmpeg master (shelf EQ, compression, loudnorm -16
  LUFS, 44.1kHz, long fades).
- The server runs it via `LullabyJob` (`/api/lullaby_start`, `/api/lullaby_status`,
  outputs served from `/lullabies/`). It refuses to start while a studio model is
  loaded — the GPU must be free for Demucs.
- Optional **ACE-Step polish** (experimental, OFF by default): after the render, the
  job loads the music worker and runs the result through ACE-Step audio2audio with
  lullaby tags, then re-normalizes loudness. Testing showed ACE conditions poorly on
  sparse solo-piano input (near-zero crest factor, harmony drifts off the arrangement
  at any denoise, both turbo and sft) — it works well on full-band songs, so the
  toggle is kept for experimentation. Failures are non-fatal (the clean render ships).

---

## Appendix A — Required vs. NOT-used models

**REQUIRED (install these):**
- Ollama: `gpt-oss:20b`, `qwen3-vl:8b` (optional Unlocked text model:
  `huihui_ai/gpt-oss-abliterated:20b` — drives the Language tab's 🔓 Unlocked toggle and
  uncensored Story Maker prose; override the tag with the `OLLAMA_UNLOCKED` env var)
- ComfyUI: `flux-2-klein-base-4b-fp8.safetensors`, `qwen_3_4b.safetensors`, `flux2-vae.safetensors`
  (optional uncensored: `miraclein-9b-fp8.safetensors` + `qwen_3_8b_fp8mixed.safetensors`)
- Auto: `nvidia/parakeet-tdt-0.6b-v2`, `hexgrad/Kokoro-82M`, `ResembleAI/chatterbox`, Coqui `XTTS-v2`
  (+ `dvae.pth`, `mel_stats.pth`)

- koboldcpp: `koboldcpp.exe` + `Cydonia-24B-v4.3-IQ4_XS.gguf` (Story Maker fiction backend — see PHASE 5e).
- ffmpeg (required for the Audiobook MP3 export — see PHASE 5f). Optional: **espeak-ng + a `zonos` conda env** for the Zonos narrator.
- ComfyUI: ACE-Step 1.5 split files (`acestep_v1.5_xl_turbo_bf16` [default] +
  `acestep_v1.5_xl_sft_bf16` + `qwen_0.6b_ace15` + `qwen_4b_ace15` + `ace_1.5_vae`) — Music tab,
  see PHASE 5g.
- ComfyUI venv pips: `rembg` + `onnxruntime` (Sprite Studio transparency — installed in PHASE 4;
  rembg's `u2net` weights auto-download to `%USERPROFILE%\.u2net` on first cutout).
- `lullabykit`: its own venv (torch cu129 + `demucs` + `basic-pitch[onnx]` + `pretty_midi`/`mido`/
  `soundfile`), a portable FluidSynth build, the FluidR3 GM soundfont, and the Salamander Grand Piano
  soundfont (falls back to FluidR3's GM piano if missing) — Lullaby and Track Splitter tabs, see PHASE 8.

**NOT used by the studio — do NOT install (these are leftovers from unrelated experiments):**
- `qwen3.5:9b` (appears in the studio's stop-list but is never mapped to a task).
- Any GGUF chat models (e.g. Qwen2.5-Coder-14B-GGUF, DeepSeek-R1-Distill-Qwen-14B-GGUF).
- The `ace` / `ace_step` conda envs, the standalone ACE-Step repo (`c:\AI Env\ACE-Step`), and the old
  v1 checkpoint `ace_step_v1_3.5b.safetensors` — the Music tab uses the **ACE-Step 1.5 split files**
  from PHASE 5g instead.
- FLUX.1-schnell (`flux1-schnell-fp8.safetensors`) — the studio uses FLUX.2 Klein, not schnell.

## Appendix B — Quick troubleshooting

| Symptom | Cause / fix |
|---|---|
| ComfyUI crashes at startup with a `UnicodeEncodeError` | Launch with `PYTHONUTF8=1` (studioctl already sets this). |
| Image gen errors `mat1/mat2 shapes` | Text-encoder ↔ diffusion-model size mismatch (4B encoder with 9B model or vice-versa). |
| STT returns an **empty** transcript, no error | VRAM starvation — a big model (Ollama/ComfyUI) is resident. Stop it first; the studio loads one model at a time. |
| Language tab: "empty response (is the Language model loaded?)" | gpt-oss are **reasoning** models — the hidden reasoning can eat the whole token budget. `ask.py` defaults gpt-oss to `reasoning_effort=low` and the studio gives a larger budget; if it recurs, raise `--max-tokens` or shorten the prompt. |
| Chatterbox output is garbled | Always provide a clean 5–15 s reference clip (`--ref`); reference-free is unstable. |
| XTTS import breaks after an update | `transformers` 5.x removed `isin_mps_friendly`; pin `transformers>=4.57,<5` in the `xtts` env. |
| Microphone blocked in the browser | Mic needs a secure context — use `http://127.0.0.1:8800` (localhost) or the Tailscale HTTPS URL. |
| Visual panel won't open | Default Python lacks tkinter; launch via `Studio Control Panel.cmd`, which picks a conda-env Python. |
| Port already in use | `studioctl.ps1 stop`, or find the owner with `Get-NetTCPConnection -LocalPort 8800`. |
| Music track is near-silent, or vocals are broken-up/crackly | ACE-Step 1.5's planning LM collapses on **rare seeds** (deterministic per seed). musicgen.py auto-detects near-silence and rerolls up to 2× when the seed wasn't pinned; for a *partially* rough track just regenerate (new seed) or try the annotated `er_sde` + `linear_quadratic` sampler/scheduler picks. |
| ACE-Step output is garbage with `--use-sage-attention` | Known incompatibility — never launch ComfyUI with that flag for this stack. |
| ComfyUI logs `Failed to initialize database ... unable to open database file` at startup | Harmless for headless/API use — generation works regardless. |
| ComfyUI won't load ACE-Step 1.5 models (`unknown model` / missing nodes) | The ComfyUI checkout is too old — `git -C "$HOME\comfyui-src" pull` and reinstall requirements (ACE-Step 1.5 needs a 2026+ build; the Desktop app's bundled ComfyUI is not used). |

---

# EMBEDDED SOURCE FILES

Below is the complete source of every program in the stack. For each, **create the file at the exact path
in its heading** and paste its contents verbatim (strip nothing). There are 27 files.

## File 1 of 27 — `%USERPROFILE%\local-ai-studio\server.py`

```python
#!/usr/bin/env python3
"""Local AI Studio — one local web app with a panel per local-model worker.

Pure stdlib backend. Explicit per-worker LOAD / STOP: only one heavy model is
resident at a time (16GB), and the user controls which. Loading a worker first
unloads everything else. STT/TTS run as persistent subprocesses (model stays in
VRAM between calls); Ollama uses keep_alive; ComfyUI is warmed by a 1-step gen.

Run:  python server.py   ->  open http://127.0.0.1:8800
"""
from __future__ import annotations

import base64
import json
import os
import queue
import re
import shutil
import socket
import subprocess
import sys
import threading
import time
import urllib.request
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

HERE = os.path.dirname(os.path.abspath(__file__))
HOME = os.path.expanduser("~")
SKILLS = os.path.join(HOME, ".claude", "skills")
ENVS = os.path.join(HOME, ".conda", "envs")
OLLAMA = os.path.join(os.environ.get("LOCALAPPDATA", ""), "Programs", "Ollama", "ollama.exe")
OLLAMA_URL = "http://127.0.0.1:11434"
COMFY = "http://127.0.0.1:8188"
# ComfyUI's input dir — edit sources get staged here (the one disk write we keep
# for image editing). Swept clean when the image worker is stopped.
COMFY_INPUT = os.environ.get("COMFYUI_INPUT") or os.path.join(HOME, "Documents", "ComfyUI", "input")
TMP = os.path.join(HERE, "_tmp")
os.makedirs(TMP, exist_ok=True)


def cleanup_edit_inputs():
    """Delete staged edit-source images (the uploaded inputs) from ComfyUI's input
    dir and our scratch copies, so an uploaded image never lingers on disk past the
    editing session. Called when the image worker is unloaded/stopped."""
    for d, pref in ((COMFY_INPUT, "edit_"), (TMP, "edit_src_")):
        try:
            for fn in os.listdir(d):
                if fn.startswith(pref) and fn.lower().endswith((".png", ".jpg", ".jpeg", ".webp")):
                    try:
                        os.remove(os.path.join(d, fn))
                    except Exception:
                        pass
        except Exception:
            pass


def cleanup_music_inputs():
    """Delete staged remix-source audio (ComfyUI input dir), our scratch copies, and
    any track that slipped past musicgen.py's immediate delete (Windows can hold a
    brief lock on fresh files) when the music worker is unloaded — same no-lingering
    rule as edit images."""
    comfy_music_out = os.path.join(os.path.dirname(COMFY_INPUT), "output", "studio_music")
    for d, pref in ((COMFY_INPUT, "music_"), (TMP, "music_"), (comfy_music_out, "m_")):
        try:
            for fn in os.listdir(d):
                if fn.startswith(pref):
                    try:
                        os.remove(os.path.join(d, fn))
                    except Exception:
                        pass
        except Exception:
            pass

ASK = os.path.join(SKILLS, "local-llm", "scripts", "ask.py")
GEN = os.path.join(SKILLS, "local-image", "scripts", "gen.py")
MUSIC = os.path.join(SKILLS, "local-music", "scripts", "musicgen.py")
STT = os.path.join(SKILLS, "local-stt", "scripts", "transcribe.py")
TTS = os.path.join(SKILLS, "local-tts", "scripts", "speak.py")
VOICE = os.path.join(SKILLS, "local-tts", "scripts", "voicechange.py")

# Voice fine-tuning (XTTS) — train/save/use personal voice models
XTTS_PY = os.path.join(ENVS, "xtts", "python.exe")
XTTS_TRAIN = os.path.join(SKILLS, "local-voice", "scripts", "xtts_train.py")
XTTS_INFER = os.path.join(SKILLS, "local-voice", "scripts", "xtts_infer.py")
SCRIPTS_JSON = os.path.join(SKILLS, "local-voice", "scripts", "scripts.json")
VOICES = os.path.join(HERE, "voices")      # saved voice models live here, one dir each
os.makedirs(VOICES, exist_ok=True)

STORIES = os.path.join(HERE, "stories")    # one JSON file per Story Maker project
os.makedirs(STORIES, exist_ok=True)

# "Unlocked" (uncensored / abliterated) text model — same gpt-oss-20b weights with
# the refusal directions removed, so quality matches the standard model. Configurable
# via the OLLAMA_UNLOCKED env var; pull it with `ollama pull <tag>`.
OLLAMA_UNLOCKED = os.environ.get("OLLAMA_UNLOCKED", "huihui_ai/gpt-oss-abliterated:20b")
OLLAMA_MODELS = ["gpt-oss:20b", "qwen3-vl:8b", "qwen3.5:9b", OLLAMA_UNLOCKED]
OLLAMA_FOR_TASK = {"code": "gpt-oss:20b", "research": "gpt-oss:20b", "vision": "qwen3-vl:8b"}
ENV_PY = {e: os.path.join(ENVS, e, "python.exe") for e in ("nemo-asr", "kokoro", "chatterbox")}

# ---- Story Maker generation backend: koboldcpp + a fiction model + DRY sampler ----
# gpt-oss is a reasoning/code model and loops / leaks its chain-of-thought on long
# prose. For novels we instead run koboldcpp (a separate local server, :5001) holding a
# creative GGUF — a Mistral-Small-24B fiction finetune ("Cydonia") — with the DRY
# sampler, which is purpose-built to stop repetition. It holds ~13GB, so it is a normal
# one-model-at-a-time worker: loading it frees Ollama/ComfyUI. Story Maker uses it when
# loaded; if only the Ollama LLM is loaded it still falls back to that (with penalties).
KOBOLD_DIR = os.environ.get("KOBOLD_DIR", os.path.join(HOME, "koboldcpp"))
KOBOLD_EXE = os.environ.get("KOBOLD_EXE", os.path.join(KOBOLD_DIR, "koboldcpp.exe"))
KOBOLD_MODEL = os.environ.get("KOBOLD_MODEL",
                              os.path.join(KOBOLD_DIR, "models", "Cydonia-24B-v4.3-IQ4_XS.gguf"))
KOBOLD_URL = "http://127.0.0.1:5001"
KOBOLD_CTX = int(os.environ.get("KOBOLD_CTX", "12288"))
STORY_MODEL_LABEL = "Cydonia-24B"
_KOBOLD_PROC = None

# ---- Audiobook module (Story project / pasted text -> chaptered MP3) ---------
# Engine-agnostic: it narrates with whichever TTS worker is loaded (Kokoro / Chatterbox
# / a fine-tuned XTTS voice / Zonos), chunks the text, synthesizes chunk-by-chunk, and
# stitches per-chapter + full-book MP3s with ffmpeg.
AUDIOBOOKS = os.path.join(HERE, "audiobooks")
os.makedirs(AUDIOBOOKS, exist_ok=True)
# Reference-clip library for cloning engines (Zonos/Chatterbox) — drop wav/mp3 clips here.
REFS = os.path.join(HERE, "refs")
os.makedirs(REFS, exist_ok=True)

# ---- Lullaby Studio (song -> soft solo-piano lullaby instrumental) -------------
# Self-contained kit under lullabykit/: its own venv (torch/demucs/basic-pitch),
# bundled FluidSynth + FluidR3 soundfont. The pipeline is a transient GPU job
# (demucs runs for ~30s then frees VRAM), not a resident worker.
LULLABYKIT = os.path.join(HERE, "lullabykit")
LULLABY_PY = os.path.join(LULLABYKIT, ".venv", "Scripts", "python.exe")
LULLABY_PIPELINE = os.path.join(LULLABYKIT, "pipeline.py")
LULLABIES = os.path.join(HERE, "lullabies")
os.makedirs(LULLABIES, exist_ok=True)

# ---- Composer (brief -> arranged, mixed, automated instrumental) --------------
# The LLM plans (instruments, structure, chords, FX, automation); composerkit.py
# writes the notes and does the mixing. It runs under lullabykit's venv, which
# already has numpy/scipy/soundfile/pretty_midi plus the bundled FluidSynth and
# the FluidR3 GM soundfont — the "plugin library" the planner picks from. No GPU
# is involved in the render, so it can't collide with a resident model.
COMPOSERKIT = os.path.join(HERE, "composerkit.py")
COMPOSITIONS = os.path.join(HERE, "compositions")
os.makedirs(COMPOSITIONS, exist_ok=True)

# ---- Sprite Studio (2D game sprites from one reference image) -----------------
# Every frame is a FLUX.2 Klein reference-edit (gen.py --image): the uploaded art is
# chained through ReferenceLatent, which is what keeps character/style consistent
# across frames. Post-processing (rembg transparent cutout, resize, strip + combined
# sheets) runs in spritekit.py under ComfyUI's venv python (has Pillow; rembg optional).
SPRITES = os.path.join(HERE, "sprites")
os.makedirs(SPRITES, exist_ok=True)
SPRITEKIT = os.path.join(HERE, "spritekit.py")
# ComfyUI's venv python (canonical: the comfyui-src runtime checkout; some installs
# also carry a venv in the Documents data dir — accept either).
COMFY_PY = os.environ.get("COMFYUI_PY") or next(
    (p for p in (os.path.join(HOME, "comfyui-src", ".venv", "Scripts", "python.exe"),
                 os.path.join(HOME, "Documents", "ComfyUI", ".venv", "Scripts", "python.exe"))
     if os.path.isfile(p)), "python")
# Zonos (optional, max-quality expressive TTS) runs in its own conda env and needs the
# espeak-ng phonemizer backend.
ZONOS_PY = os.path.join(ENVS, "zonos", "python.exe")
ZONOS_TTS = os.path.join(SKILLS, "local-tts", "scripts", "zonos_tts.py")
ESPEAK_DLL = os.environ.get("PHONEMIZER_ESPEAK_LIBRARY",
                            r"C:\Program Files\eSpeak NG\libespeak-ng.dll")
if os.path.isfile(ESPEAK_DLL):   # let phonemizer (Zonos) find espeak-ng
    os.environ.setdefault("PHONEMIZER_ESPEAK_LIBRARY", ESPEAK_DLL)
    os.environ["PATH"] = os.path.dirname(ESPEAK_DLL) + os.pathsep + os.environ.get("PATH", "")


def _ffmpeg() -> str:
    import glob
    cands = [os.environ.get("FFMPEG_EXE"), shutil.which("ffmpeg")]
    cands += glob.glob(os.path.join(os.environ.get("LOCALAPPDATA", ""), "Microsoft",
                       "WinGet", "Packages", "Gyan.FFmpeg*", "**", "ffmpeg.exe"),
                       recursive=True)
    for c in cands:
        if c and os.path.isfile(c):
            return c
    raise RuntimeError("ffmpeg not found — install it: winget install Gyan.FFmpeg")


def llm_model_for(task: str, unlocked=False) -> str:
    """Resolve the Ollama tag for a Language request. The Unlocked option applies to
    text tasks only — vision always uses the vision model."""
    if unlocked and task != "vision":
        return OLLAMA_UNLOCKED
    return OLLAMA_FOR_TASK.get(task, "gpt-oss:20b")


# Suppress console windows for every child process. When the server itself is
# launched detached / windowless (Studio Control Panel), each bare subprocess
# spawn otherwise flashes a cmd window — the GPU poll alone did it every 5s.
NOWIN = {"creationflags": 0x08000000} if os.name == "nt" else {}   # CREATE_NO_WINDOW


def _run(cmd: list, timeout: int = 600):
    env = dict(os.environ, PYTHONUTF8="1", PYTHONIOENCODING="utf-8")
    p = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE,
                         text=True, encoding="utf-8", errors="replace", env=env, **NOWIN)
    try:
        out, err = p.communicate(timeout=timeout)
    except subprocess.TimeoutExpired:
        # Windows: kill the WHOLE tree. subprocess.run's own kill() leaves the child's
        # conhost holding the stdout pipe open, so communicate() blocks forever and the
        # caller's MGR.lock is never released (every later request then hangs on it).
        subprocess.run(["taskkill", "/T", "/F", "/PID", str(p.pid)],
                       capture_output=True, timeout=15, **NOWIN)
        try:
            out, err = p.communicate(timeout=10)
        except Exception:
            out, err = "", ""
        err = (err or "") + f"\n[timed out after {timeout}s — process tree killed]"
        return 1, (out or "").strip(), err.strip()
    return p.returncode, (out or "").strip(), (err or "").strip()


_GPU_CACHE = {"t": 0.0, "data": {"used": 0, "total": 0, "util": 0}}


def gpu_stats():
    """VRAM used/total (MiB) + utilisation for the topbar GPU bar. Cached 2s so
    multiple browser tabs polling every 5s don't each spawn nvidia-smi."""
    if time.time() - _GPU_CACHE["t"] > 2:
        try:
            rc, out, _ = _run(["nvidia-smi", "--query-gpu=memory.used,memory.total,utilization.gpu",
                               "--format=csv,noheader,nounits"], timeout=10)
            used, total, util = [int(x.strip()) for x in out.split(",")]
            _GPU_CACHE["data"] = {"used": used, "total": total, "util": util}
        except Exception:
            pass
        _GPU_CACHE["t"] = time.time()
    return _GPU_CACHE["data"]


def ollama_stop():
    if not os.path.isfile(OLLAMA):
        return
    for m in OLLAMA_MODELS:
        try:
            subprocess.run([OLLAMA, "stop", m], capture_output=True, timeout=20, **NOWIN)
        except Exception:
            pass


def ollama_load(model: str):
    data = json.dumps({"model": model, "prompt": "ok", "stream": False,
                       "keep_alive": -1, "options": {"num_predict": 1}}).encode()
    req = urllib.request.Request(f"{OLLAMA_URL}/api/generate", data=data,
                                 headers={"Content-Type": "application/json"})
    urllib.request.urlopen(req, timeout=300).read()


def comfy_free():
    try:
        req = urllib.request.Request(f"{COMFY}/free", method="POST",
                                     data=json.dumps({"unload_models": True, "free_memory": True}).encode(),
                                     headers={"Content-Type": "application/json"})
        urllib.request.urlopen(req, timeout=10).read()
        time.sleep(1.5)
    except Exception:
        pass


def comfy_warmup(model: str):
    rc, sout, err = _run([sys.executable, GEN, "--prompt", "warmup", "--out", "-",
                          "--model", model, "--size", "1:1", "--steps", "1"], timeout=400)
    if rc != 0 or not (sout or "").startswith("data:image"):
        raise RuntimeError(err or sout or "ComfyUI warmup failed (is it running on :8188?)")


def music_warmup():
    """Pull the ACE-Step 1.5 models into VRAM with a 1-step, 5-second throwaway gen
    (planning-LM sampling skipped — loading is all that matters here)."""
    rc, sout, err = _run([sys.executable, MUSIC, "--tags", "warmup", "--out", "-",
                          "--seconds", "5", "--steps", "1", "--no-audio-codes"], timeout=400)
    if rc != 0 or not (sout or "").strip().startswith("{"):
        raise RuntimeError(err or sout or "ACE-Step warmup failed (is ComfyUI running on :8188?)")


# ---- koboldcpp (Story Maker fiction backend) --------------------------------
def kobold_running() -> bool:
    try:
        urllib.request.urlopen(f"{KOBOLD_URL}/api/v1/model", timeout=3).read()
        return True
    except Exception:
        return False


def kobold_start():
    """Launch koboldcpp headless with the fiction model, flash attention, and an 8-bit
    KV cache so a 24B Q4 fits 16GB with a usable context window. Idempotent (no-op if a
    koboldcpp is already answering on :5001). Blocks until the model has loaded."""
    global _KOBOLD_PROC
    if kobold_running():
        return
    if not os.path.isfile(KOBOLD_EXE):
        raise RuntimeError(f"koboldcpp not found: {KOBOLD_EXE}")
    if not os.path.isfile(KOBOLD_MODEL):
        raise RuntimeError(f"story model not found: {KOBOLD_MODEL}")
    args = [KOBOLD_EXE, "--model", KOBOLD_MODEL, "--usecublas", "--gpulayers", "999",
            "--contextsize", str(KOBOLD_CTX), "--flashattention", "--quantkv", "1",
            "--host", "127.0.0.1", "--port", "5001", "--skiplauncher", "--quiet"]
    log = open(os.path.join(TMP, "kobold.log"), "w", encoding="utf-8")
    _KOBOLD_PROC = subprocess.Popen(args, stdout=log, stderr=subprocess.STDOUT, cwd=KOBOLD_DIR, **NOWIN)
    for _ in range(180):                       # 13GB load -> ~30-90s
        if kobold_running():
            return
        if _KOBOLD_PROC.poll() is not None:
            raise RuntimeError("koboldcpp exited during startup (see _tmp/kobold.log)")
        time.sleep(1)
    kobold_stop()
    raise RuntimeError("koboldcpp did not become ready in time (see _tmp/kobold.log)")


def kobold_stop():
    global _KOBOLD_PROC
    try:
        if _KOBOLD_PROC and _KOBOLD_PROC.poll() is None:
            _KOBOLD_PROC.terminate()
            try:
                _KOBOLD_PROC.wait(timeout=10)
            except Exception:
                _KOBOLD_PROC.kill()
    except Exception:
        pass
    _KOBOLD_PROC = None
    # Failsafe: kill any orphan still holding :5001 (e.g. a pre-existing instance).
    try:
        rc, out, _ = _run(["powershell", "-NoProfile", "-Command",
                           "(Get-NetTCPConnection -LocalPort 5001 -State Listen "
                           "-ErrorAction SilentlyContinue).OwningProcess"], timeout=15)
        for pid in {p.strip() for p in (out or "").split() if p.strip().isdigit()}:
            subprocess.run(["taskkill", "/F", "/PID", pid], capture_output=True, timeout=10, **NOWIN)
    except Exception:
        pass


def kobold_chat(messages: list, *, max_tokens: int = 1200, temperature: float = 0.9,
                top_p: float = 0.95, min_p: float = 0.03, rep_pen: float = 1.05,
                dry: bool = True, timeout: int = 600) -> str:
    """One chat completion from koboldcpp's OpenAI-compatible endpoint, with the DRY
    sampler + min_p (koboldcpp reads these extra fields). koboldcpp applies the model's
    own chat template from the GGUF, so we just pass system/user messages."""
    payload = {
        "model": "cydonia", "messages": messages, "max_tokens": max_tokens,
        "temperature": temperature, "top_p": top_p, "top_k": 0, "min_p": min_p,
        "rep_pen": rep_pen, "rep_pen_range": 2048,
    }
    if dry:
        payload.update({"dry_multiplier": 0.8, "dry_base": 1.75, "dry_allowed_length": 2,
                        "dry_sequence_breakers": ["\n", ":", "\"", "*", "”", "“"]})
    req = urllib.request.Request(f"{KOBOLD_URL}/v1/chat/completions",
                                 data=json.dumps(payload).encode("utf-8"),
                                 headers={"Content-Type": "application/json"}, method="POST")
    with urllib.request.urlopen(req, timeout=timeout) as r:
        j = json.loads(r.read().decode("utf-8"))
    return (j["choices"][0]["message"].get("content") or "").strip()


# ---- persistent subprocess worker (STT / TTS) -------------------------------
class Worker:
    def __init__(self, py: str, script: str, extra: list, ready_timeout: int = 240):
        if not os.path.isfile(py):
            raise RuntimeError(f"python env not found: {py}")
        env = dict(os.environ, PYTHONUTF8="1", PYTHONIOENCODING="utf-8")
        self.log = open(os.path.join(TMP, "worker_err.log"), "w", encoding="utf-8")
        self.p = subprocess.Popen([py, script, "--serve", *extra],
                                  stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=self.log,
                                  text=True, encoding="utf-8", errors="replace", env=env, bufsize=1,
                                  **NOWIN)
        self.q: queue.Queue = queue.Queue()
        threading.Thread(target=self._reader, daemon=True).start()
        first = self._get(ready_timeout)
        if not first or not first.get("ready"):
            self.stop()
            raise RuntimeError((first or {}).get("error", "worker failed to load (see worker_err.log)"))

    def _reader(self):
        for line in self.p.stdout:
            line = line.strip()
            if line:
                self.q.put(line)
        self.q.put(None)

    def _get(self, timeout):
        try:
            line = self.q.get(timeout=timeout)
        except queue.Empty:
            return None
        if line is None:
            return None
        try:
            return json.loads(line)
        except Exception:
            return None

    def call(self, obj: dict, timeout: int = 300) -> dict:
        self.p.stdin.write(json.dumps(obj) + "\n")
        self.p.stdin.flush()
        r = self._get(timeout)
        if r is None:
            raise RuntimeError("worker timed out or exited")
        if r.get("error"):
            raise RuntimeError(r["error"])
        return r

    def stop(self):
        try:
            self.p.terminate()
            self.p.wait(timeout=10)
        except Exception:
            try:
                self.p.kill()
            except Exception:
                pass
        try:
            self.log.close()
        except Exception:
            pass


# ---- load/stop manager (one heavy model resident at a time) -----------------
class Manager:
    def __init__(self):
        self.lock = threading.RLock()
        self.key = None          # e.g. "llm:gpt-oss:20b", "image:base4b", "stt", "tts:kokoro"
        self.workers = {}        # "stt"|"tts" -> Worker

    def status(self):
        return {"key": self.key}

    def _unload_all(self):
        was_image = (self.key or "").startswith("image:")
        was_music = (self.key or "") == "music"
        for w in list(self.workers.values()):
            w.stop()
        self.workers.clear()
        ollama_stop()
        comfy_free()
        kobold_stop()
        if was_image:
            cleanup_edit_inputs()   # uploaded edit sources deleted when image model stops
        if was_music:
            cleanup_music_inputs()  # uploaded remix sources deleted when music model stops
        self.key = None

    def load(self, worker: str, params: dict):
        with self.lock:
            if TRAINJOB.state in ("preparing", "training"):
                raise RuntimeError("a voice is training — wait for it to finish")
            self._unload_all()
            try:
                if worker == "llm":
                    model = llm_model_for(params.get("task", "code"), params.get("unlocked"))
                    ollama_load(model)
                    self.key = f"llm:{model}"
                elif worker == "image":
                    model = params.get("model", "base4b")
                    comfy_warmup(model)
                    self.key = f"image:{model}"
                elif worker == "stt":
                    self.workers["stt"] = Worker(ENV_PY["nemo-asr"], STT, [])
                    self.key = "stt"
                elif worker == "tts":
                    engine = params.get("engine", "kokoro")
                    if engine == "zonos":
                        if not os.path.isfile(ZONOS_PY):
                            raise RuntimeError("Zonos is not installed (see setup PHASE 5f)")
                        self.workers["tts"] = Worker(ZONOS_PY, ZONOS_TTS, [], ready_timeout=420)
                    else:
                        env = "chatterbox" if engine == "chatterbox" else "kokoro"
                        self.workers["tts"] = Worker(ENV_PY[env], TTS, ["--engine", engine])
                    self.key = f"tts:{engine}"
                elif worker == "voice":
                    self.workers["voice"] = Worker(ENV_PY["chatterbox"], VOICE, [], ready_timeout=420)
                    self.key = "voice"
                elif worker == "voicemodel":
                    name = params.get("name", "")
                    mdir = os.path.join(VOICES, name)
                    if not os.path.isfile(os.path.join(mdir, "model.pth")):
                        raise RuntimeError(f"voice model '{name}' not found")
                    self.workers["voicemodel"] = Worker(XTTS_PY, XTTS_INFER,
                                                        ["--model-dir", mdir], ready_timeout=300)
                    self.key = f"voicemodel:{name}"
                elif worker == "music":
                    music_warmup()
                    self.key = "music"
                elif worker == "story":
                    kobold_start()
                    self.key = f"story:{STORY_MODEL_LABEL}"
                else:
                    raise RuntimeError(f"unknown worker {worker}")
            except Exception:
                self._unload_all()
                raise
            return self.status()

    def stop(self):
        with self.lock:
            self._unload_all()
            return self.status()


MGR = Manager()


# ---- saved voice-model library ----------------------------------------------
def list_voices() -> list:
    out = []
    for n in sorted(os.listdir(VOICES)):
        d = os.path.join(VOICES, n)
        if os.path.isfile(os.path.join(d, "model.pth")):
            meta = {}
            mp = os.path.join(d, "meta.json")
            if os.path.isfile(mp):
                try:
                    meta = json.load(open(mp))
                except Exception:
                    pass
            out.append({"name": n, "samples": meta.get("samples")})
    return out


def delete_voice(name: str):
    if MGR.key == f"voicemodel:{name}":
        MGR.stop()
    d = os.path.join(VOICES, name)
    if os.path.isdir(d):
        shutil.rmtree(d, ignore_errors=True)
    return {"ok": True}


def oneshot_stt(audio_b64: str) -> str:
    """Transcribe audio with a transient Parakeet run (small; coexists with XTTS)."""
    apath = _write_wav(audio_b64, "vm_stt_in.wav")
    rc, out, err = _run([ENV_PY["nemo-asr"], STT, apath], timeout=300)
    text = (out.splitlines() or [""])[-1].strip() if out else ""
    if not text:
        raise RuntimeError("could not transcribe the audio")
    return text


def do_voicemodel(text: str, audio_b64, controls: dict = None) -> dict:
    with MGR.lock:
        w = MGR.workers.get("voicemodel")
        if not w:
            raise RuntimeError("voice model not loaded — pick a voice and click Load")
        if not (text or "").strip() and audio_b64:
            text = oneshot_stt(audio_b64)
        text = (text or "").strip()
        if not text:
            raise RuntimeError("no text or speech provided")
        out = os.path.join(TMP, f"vm_{int(time.time())}.wav")
        w.call({"text": text, "out": out, **(controls or {})}, timeout=300)
        if not os.path.isfile(out):
            raise RuntimeError("synthesis produced no audio")
        with open(out, "rb") as f:
            return {"audio": "data:audio/wav;base64," + base64.b64encode(f.read()).decode(),
                    "text": text}


# ---- async training job (one at a time) -------------------------------------
class TrainJob:
    def __init__(self):
        self.lock = threading.Lock()
        self._reset()

    def _reset(self):
        self.state = "idle"   # idle | preparing | training | done | error
        self.epoch = 0
        self.total = 0
        self.name = None
        self.message = ""
        self.proc = None

    def status(self) -> dict:
        return {"state": self.state, "epoch": self.epoch, "total": self.total,
                "name": self.name, "message": self.message}

    def start(self, name: str, language: str, epochs: int, samples: list) -> dict:
        name = re.sub(r"[^A-Za-z0-9_-]", "_", (name or "").strip()) or "voice"
        with self.lock:
            if self.state in ("preparing", "training"):
                raise RuntimeError("a training job is already running")
            self._reset()
            self.state = "preparing"
            self.name = name
            self.total = epochs
        threading.Thread(target=self._run, args=(name, language, epochs, samples),
                         daemon=True).start()
        return self.status()

    def _run(self, name, language, epochs, samples):
        ds = os.path.join(VOICES, "_datasets", name)
        try:
            MGR.stop()  # free the GPU for training
            wavs_dir = os.path.join(ds, "wavs")
            shutil.rmtree(ds, ignore_errors=True)
            os.makedirs(wavs_dir, exist_ok=True)
            rows = []
            for i, s in enumerate(samples):
                b = s.get("audio", "")
                txt = (s.get("text") or "").replace("|", " ").strip()
                if not b or not txt:
                    continue
                fn = f"wavs/clip_{i:03d}.wav"
                with open(os.path.join(ds, fn), "wb") as f:
                    f.write(base64.b64decode(b.split(",")[-1]))
                rows.append((fn, txt, "voice"))
            if len(rows) < 4:
                raise RuntimeError("need at least 4 recorded samples")
            n_eval = max(1, len(rows) // 8)

            def write_csv(path, rs):
                with open(path, "w", encoding="utf-8", newline="") as f:
                    f.write("audio_file|text|speaker_name\n")
                    for r in rs:
                        f.write("|".join(r) + "\n")
            write_csv(os.path.join(ds, "metadata_eval.csv"), rows[:n_eval])
            write_csv(os.path.join(ds, "metadata_train.csv"), rows[n_eval:])

            outdir = os.path.join(VOICES, name)
            shutil.rmtree(outdir, ignore_errors=True)
            cmd = [XTTS_PY, XTTS_TRAIN, "--dataset-dir", ds, "--out-dir", outdir,
                   "--language", language, "--epochs", str(epochs),
                   "--batch-size", "2", "--grad-accum", "4"]
            env = dict(os.environ, COQUI_TOS_AGREED="1", PYTHONUTF8="1", PYTHONIOENCODING="utf-8")
            self.state = "training"
            self.proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
                                         text=True, encoding="utf-8", errors="replace",
                                         env=env, bufsize=1, **NOWIN)
            for line in self.proc.stdout:
                m = re.search(r"EPOCH:\s*(\d+)/(\d+)", line)
                if m:
                    self.epoch = int(m.group(1)) + 1
            rc = self.proc.wait()
            if rc == 0 and os.path.isfile(os.path.join(outdir, "model.pth")):
                self.state, self.message, self.epoch = "done", "saved", self.total
            else:
                self.state, self.message = "error", f"training failed (exit {rc})"
        except Exception as e:
            self.state, self.message = "error", str(e)
        finally:
            shutil.rmtree(ds, ignore_errors=True)


TRAINJOB = TrainJob()


# ---- worker calls (assume the right model is already loaded) -----------------
def current_llm_model():
    """The Ollama tag currently resident (from the Manager key), or None.
    Generation always uses what is actually loaded in VRAM."""
    k = MGR.key or ""
    return k[4:] if k.startswith("llm:") else None


def llm_raw(prompt: str, *, task: str = "research", max_tokens: int = 800,
            system: str = None, temperature: float = None, image_path: str = None,
            timeout: int = 300, model: str = None, top_p: float = None,
            frequency_penalty: float = None, presence_penalty: float = None) -> str:
    """One-shot local-LLM call via ask.py. Shared by the Language tab and the
    Story Maker orchestrator. The Manager lock is the caller's responsibility for
    multi-step jobs; single calls take it here. `model` overrides the tag (used to
    run on whichever Standard/Unlocked model is loaded). The penalty/top_p knobs
    curb the repetition loops that long creative generations are prone to."""
    cmd = [sys.executable, ASK, "--task", task, "--max-tokens", str(int(max_tokens))]
    if model:
        cmd += ["--model", model]
    if system:
        cmd += ["--system", system]
    if temperature is not None:
        cmd += ["--temperature", str(temperature)]
    if top_p is not None:
        cmd += ["--top-p", str(top_p)]
    if frequency_penalty is not None:
        cmd += ["--frequency-penalty", str(frequency_penalty)]
    if presence_penalty is not None:
        cmd += ["--presence-penalty", str(presence_penalty)]
    if image_path:
        cmd += ["--image", image_path]
    cmd.append(prompt or "Describe this.")
    rc, out, err = _run(cmd, timeout=timeout)
    if rc != 0 or not out:
        raise RuntimeError(err or "empty response (is the Language model loaded?)")
    return out


def do_llm(task: str, prompt: str, image_b64, unlocked=False) -> str:
    with MGR.lock:
        ipath = None
        if image_b64:
            ipath = os.path.join(TMP, "vision_in.png")
            with open(ipath, "wb") as f:
                f.write(base64.b64decode(image_b64.split(",")[-1]))
        # run on whatever is actually loaded; fall back to the requested variant
        model = current_llm_model() or llm_model_for(task, unlocked)
        # gpt-oss reasoning models need headroom beyond the visible answer (the hidden
        # reasoning is also billed against max_tokens), so give the Language tab room.
        return llm_raw(prompt, task=task, max_tokens=2048, image_path=ipath, model=model)


def do_image(prompt: str, size: str, steps: int, seed: int, model: str = "base4b") -> str:
    with MGR.lock:
        cmd = [sys.executable, GEN, "--prompt", prompt, "--out", "-",
               "--size", size, "--steps", str(steps), "--model", model]
        if seed >= 0:
            cmd += ["--seed", str(seed)]
        rc, sout, err = _run(cmd, timeout=600)
        if rc != 0 or not sout.startswith("data:image"):
            raise RuntimeError(err or sout or "image generation failed")
        # RAM-only: gen.py streamed the PNG over ComfyUI's websocket, never to disk.
        return sout


def do_edit(prompt: str, images_b64: list, model: str = "base4b",
            size: str = "", steps: int = 20, seed: int = -1, cfg: float = 0) -> str:
    with MGR.lock:
        cmd = [sys.executable, GEN, "--prompt", prompt, "--out", "-",
               "--steps", str(steps), "--model", model]
        if cfg and cfg > 0:
            cmd += ["--cfg", str(cfg)]
        if size:
            cmd += ["--size", size]
        if seed >= 0:
            cmd += ["--seed", str(seed)]
        # The uploaded source(s) are written to scratch so gen.py can stage them into
        # ComfyUI's input dir. Both copies are swept by cleanup_edit_inputs() when the
        # image worker is stopped.
        for i, b in enumerate(images_b64):
            if not b:
                continue
            ipath = os.path.join(TMP, f"edit_src_{i}.png")
            with open(ipath, "wb") as f:
                f.write(base64.b64decode(b.split(",")[-1]))
            cmd += ["--image", ipath]
        if "--image" not in cmd:
            raise RuntimeError("no source image provided")
        rc, sout, err = _run(cmd, timeout=600)
        if rc != 0 or not sout.startswith("data:image"):
            raise RuntimeError(err or sout or "edit failed")
        # RAM-only output: the edited PNG came back over the websocket, not via disk.
        return sout


def do_music(d: dict) -> dict:
    """Generate music with ACE-Step 1.5 (via musicgen.py -> ComfyUI). Exposes the full
    ComfyUI control set. Returns {"audios": [dataURL, ...], "seed": N}."""
    with MGR.lock:
        cmd = [sys.executable, MUSIC, "--out", "-",
               "--tags", (d.get("tags") or "").strip(),
               "--model", ("sft" if d.get("model") == "sft" else "turbo"),
               "--seconds", str(float(d.get("seconds", 120))),
               "--shift", str(float(d.get("shift", 3.0))),
               "--sampler", d.get("sampler", "euler"),
               "--scheduler", d.get("scheduler", "simple"),
               "--bpm", str(int(d.get("bpm", 0) or 0)),
               "--keyscale", (d.get("keyscale") or "").strip(),
               "--timesignature", (str(d.get("timesignature") or "")).strip(),
               "--language", (d.get("language") or "en").strip() or "en",
               "--lm-temperature", str(float(d.get("lm_temperature", 0.85))),
               "--lm-cfg", str(float(d.get("lm_cfg", 2.0))),
               "--denoise", str(float(d.get("denoise", 1.0))),
               "--batch", str(max(1, min(4, int(d.get("batch", 1))))),
               "--format", d.get("format", "mp3")]
        if d.get("steps") is not None:       # absent -> musicgen's per-model default (8 turbo / 50 sft)
            cmd += ["--steps", str(int(d["steps"]))]
        if d.get("cfg") is not None:         # absent -> per-model default (1 turbo / 7 sft)
            cmd += ["--cfg", str(float(d["cfg"]))]
        if d.get("quality"):
            cmd += ["--quality", d["quality"]]
        if int(d.get("seed", -1)) >= 0:
            cmd += ["--seed", str(int(d["seed"]))]
        # Lyrics go via a scratch file (they can be long; keeps the cmdline small).
        lyrics = d.get("lyrics") or ""
        if lyrics.strip():
            lpath = os.path.join(TMP, "music_lyrics.txt")
            with open(lpath, "w", encoding="utf-8") as f:
                f.write(lyrics)
            cmd += ["--lyrics-file", lpath]
        # Optional remix source; staged copies are swept by cleanup_music_inputs().
        if d.get("audio"):
            apath = os.path.join(TMP, "music_src" + (os.path.splitext(d.get("filename") or "")[1] or ".wav"))
            with open(apath, "wb") as f:
                f.write(base64.b64decode(d["audio"].split(",")[-1]))
            cmd += ["--audio", apath]
        rc, sout, err = _run(cmd, timeout=1200)
        line = (sout or "").strip().splitlines()[-1] if (sout or "").strip() else ""
        if rc != 0 or not line.startswith("{"):
            raise RuntimeError(err or sout or "music generation failed")
        return json.loads(line)


def _write_wav(b64: str, name: str) -> str:
    path = os.path.join(TMP, name)
    with open(path, "wb") as f:
        f.write(base64.b64decode(b64.split(",")[-1]))
    return path


def do_voice(mode: str, text: str, source_b64, target_b64, method: str = "tts") -> str:
    with MGR.lock:
        w = MGR.workers.get("voice")
        if not w:
            raise RuntimeError("Voice Changer model not loaded — click Load on that tab first")
        if not target_b64:
            raise RuntimeError("provide a target voice (record or upload)")
        target = _write_wav(target_b64, "voice_target.wav")
        out = os.path.join(TMP, f"voice_{int(time.time())}.wav")
        req = {"mode": mode, "target": target, "out": out, "method": method}
        if mode == "voice":
            if not source_b64:
                raise RuntimeError("provide source audio to convert (record or upload)")
            req["source"] = _write_wav(source_b64, "voice_source.wav")
        else:
            req["text"] = text
        w.call(req, timeout=300)
        if not os.path.isfile(out):
            raise RuntimeError("voice generation produced no audio")
        with open(out, "rb") as f:
            return "data:audio/wav;base64," + base64.b64encode(f.read()).decode()


def do_stt(audio_b64: str, filename: str) -> str:
    with MGR.lock:
        w = MGR.workers.get("stt")
        if not w:
            raise RuntimeError("Speech→Text model not loaded — click Load on that tab first")
        ext = os.path.splitext(filename or "a.wav")[1] or ".wav"
        apath = os.path.join(TMP, "stt_in" + ext)
        with open(apath, "wb") as f:
            f.write(base64.b64decode(audio_b64.split(",")[-1]))
        r = w.call({"in": apath}, timeout=300)
        text = (r.get("text") or "").strip()
        if not text:
            raise RuntimeError("empty transcript (check the audio)")
        return text


def _tts_controls(d: dict) -> dict:
    """Optional narration controls understood by the TTS workers (each uses what it knows):
    speed (Kokoro/Zonos), pitch_std + emotion (Zonos), exaggeration + cfg (Chatterbox)."""
    return {k: d[k] for k in ("speed", "pitch_std", "emotion", "exaggeration", "cfg")
            if d.get(k) is not None}


def list_refs() -> list:
    try:
        return sorted(f for f in os.listdir(REFS) if f.lower().endswith((".wav", ".mp3", ".m4a")))
    except Exception:
        return []


def _resolve_ref(d: dict):
    """Resolve a cloning reference to a local wav PATH from any of: an uploaded clip
    (`ref` = base64 data URL, any format → converted), a clip in the refs/ library
    (`ref_lib` = filename), or a saved XTTS voice's sample (`ref_voice` = voice name)."""
    r = d.get("ref")
    if isinstance(r, str) and r.startswith("data:"):
        raw = base64.b64decode(r.split(",")[-1])
        tin = os.path.join(TMP, f"refin_{int(time.time() * 1000)}")
        with open(tin, "wb") as f:
            f.write(raw)
        out = tin + ".wav"
        subprocess.run([_ffmpeg(), "-y", "-i", tin, "-ac", "1", "-ar", "24000", out],
                       capture_output=True, **NOWIN)
        return out if os.path.isfile(out) else None
    rl = d.get("ref_lib")
    if rl:
        p = os.path.join(REFS, os.path.basename(rl))
        if os.path.isfile(p):
            return p
    rv = d.get("ref_voice")
    if rv:
        rdir = os.path.join(VOICES, os.path.basename(rv), "reference")
        if os.path.isdir(rdir):
            for f in sorted(os.listdir(rdir)):
                if f.lower().endswith(".wav"):
                    return os.path.join(rdir, f)
    return None


def do_tts(text: str, voice: str, ref_path, controls: dict = None) -> str:
    with MGR.lock:
        w = MGR.workers.get("tts")
        if not w:
            raise RuntimeError("Text→Speech model not loaded — click Load on that tab first")
        out = os.path.join(TMP, f"tts_{int(time.time())}.wav")
        req = {"text": text, "out": out, "voice": voice, **(controls or {})}
        if ref_path:
            req["ref"] = ref_path
        w.call(req, timeout=300)
        if not os.path.isfile(out):
            raise RuntimeError("tts produced no audio")
        with open(out, "rb") as f:
            return "data:audio/wav;base64," + base64.b64encode(f.read()).decode()


# ---- Story Maker: project storage -------------------------------------------
def _story_path(sid: str) -> str:
    sid = re.sub(r"[^A-Za-z0-9_-]", "_", (sid or "").strip()) or "story"
    return os.path.join(STORIES, sid + ".json")


def _atomic_write(path: str, text: str):
    tmp = path + ".tmp"
    with open(tmp, "w", encoding="utf-8") as f:
        f.write(text)
    os.replace(tmp, path)


def story_list() -> dict:
    out = []
    for fn in sorted(os.listdir(STORIES)):
        if not fn.endswith(".json"):
            continue
        p = os.path.join(STORIES, fn)
        try:
            d = json.load(open(p, encoding="utf-8"))
            out.append({"id": d.get("id") or fn[:-5],
                        "title": d.get("title") or "Untitled",
                        "updated": os.path.getmtime(p),
                        "beats": len(d.get("beats", []))})
        except Exception:
            pass
    out.sort(key=lambda s: s.get("updated", 0), reverse=True)
    return {"stories": out}


def story_load(sid: str) -> dict:
    p = _story_path(sid)
    if not os.path.isfile(p):
        raise RuntimeError("story not found")
    return {"story": json.load(open(p, encoding="utf-8"))}


def story_save(story: dict) -> dict:
    if not isinstance(story, dict) or not story.get("id"):
        raise RuntimeError("story needs an id")
    story["updated"] = time.time()
    _atomic_write(_story_path(story["id"]), json.dumps(story, ensure_ascii=False, indent=1))
    return {"ok": True, "id": story["id"]}


def story_delete(sid: str) -> dict:
    if STORYJOB.state == "running" and STORYJOB.sid == sid:
        raise RuntimeError("this story is currently generating")
    p = _story_path(sid)
    if os.path.isfile(p):
        os.remove(p)
    return {"ok": True}


# ---- Story Maker: orchestration ---------------------------------------------
WRITER_SYSTEM = (
    "You are a master novelist. You write vivid, immersive prose one scene at a time as part of a "
    "longer continuous novel. Stay perfectly consistent with the running summary and the previous "
    "scene. Never contradict established facts. Never re-introduce things already established. "
    "CRITICAL: never repeat a sentence, phrase, or clause; never restate the premise or a "
    "character's description — reveal character through action and dialogue instead. Always move "
    "the scene forward with concrete new events. If you have nothing new to add, end the scene. "
    "Write only the scene's prose — no headings, no notes, no commentary, no meta text."
)

# Sampling knobs for creative generation — the penalties are what stop a local model
# from collapsing into "…in the process of becoming a place that was in the process of…" loops.
STORY_TOP_P = 0.92
STORY_FREQ_PENALTY = 1.1
STORY_PRES_PENALTY = 0.8


def story_write(system: str, user: str, *, max_tokens: int, temperature: float = 0.85,
                timeout: int = 600) -> str:
    """Generate story text on whichever backend is loaded. Prefers the koboldcpp fiction
    model (DRY sampler — far better long-form) when the 'story:' worker is resident, and
    falls back to the Ollama LLM via ask.py (with repetition penalties) otherwise."""
    if (MGR.key or "").startswith("story:"):
        msgs = ([{"role": "system", "content": system}] if system else [])
        msgs.append({"role": "user", "content": user})
        return kobold_chat(msgs, max_tokens=max_tokens, temperature=temperature, timeout=timeout)
    return llm_raw(user, task="research", system=system, max_tokens=max_tokens,
                   temperature=temperature, timeout=min(timeout, 400),
                   model=current_llm_model(), top_p=STORY_TOP_P,
                   frequency_penalty=STORY_FREQ_PENALTY, presence_penalty=STORY_PRES_PENALTY)


def _norm_tok(t: str) -> str:
    return re.sub(r"[^\w]", "", t).lower()


def _collapse_runs(s: str) -> str:
    """Collapse immediately-repeated word runs ('peace, peace, peace' -> 'peace';
    'a sense of X a sense of X' -> 'a sense of X'). Punctuation-insensitive."""
    toks = s.split()
    if not toks:
        return s
    for span in (7, 6, 5, 4, 3, 2, 1):
        norm = [_norm_tok(t) for t in toks]
        res, j = [], 0
        while j < len(toks):
            if j >= span and norm[j:j + span] == norm[j - span:j]:
                j += span            # drop this repeated run
                continue
            res.append(toks[j])
            j += 1
        toks = res
    return " ".join(toks)


def _dedupe(text: str) -> str:
    """Safety net against a local model's repetition loops. Collapses repeated word
    runs, drops sentences that exactly repeat an earlier one (near-certain degeneration
    in this genre), and drops consecutive duplicate paragraphs. Conservative enough to
    leave normal prose intact."""
    if not text:
        return text
    seen_sents = set()
    out_paras, prev_para, prev_set = [], None, set()
    for para in re.split(r"\n{2,}", text):
        s = _collapse_runs(para.strip())
        if not s:
            continue
        kept = []
        for snt in re.split(r"(?<=[.!?])\s+", s):
            key = re.sub(r"\s+", " ", snt.strip().lower())
            if len(key) > 12 and key in seen_sents:   # global exact-sentence repeat
                continue
            cur_set = set(re.findall(r"\w+", key))
            if len(cur_set) >= 6 and prev_set:        # consecutive near-duplicate
                inter = len(cur_set & prev_set)
                jaccard = inter / max(1, len(cur_set | prev_set))
                overlap = inter / min(len(cur_set), len(prev_set))   # catches containment
                if jaccard > 0.82 or overlap > 0.9:
                    continue
            if key:
                seen_sents.add(key)
                prev_set = cur_set
            kept.append(snt)
        s = " ".join(kept).strip()
        k = s.lower()
        if k and k == prev_para:
            continue
        if s:
            out_paras.append(s)
            prev_para = k
    out = "\n\n".join(out_paras)
    # Trim a short dangling final fragment (a pass cut off mid-sentence, e.g. "She").
    if out and not re.search(r'[.!?"”’]\s*$', out):
        idx = max(out.rfind(c) for c in '.!?"”’')
        if idx > 0 and len(out) - idx < 240:
            out = out[:idx + 1]
    return out


# Reasoning/meta leakage from gpt-oss ("We need to continue. The user requested 875
# words. We'll continue…") sometimes bleeds into the answer channel. Strip any sentence
# that is clearly the model talking about the task rather than telling the story.
# STRONG: unmistakable meta — strip even if the sentence contains a quote.
_META_STRONG_RE = re.compile(
    r"(the user (?:requested|asked|wants|said)|word count|\b\d+\s+words?\b|more words"
    r"|as an ai|as a language model|we'?ll continue|we will continue"
    r"|from where we left|continue from where|pick up (?:from|where)"
    r"|let'?s (?:produce|continue|write)|to be continued|the scene continues)",
    re.IGNORECASE)
# WEAK: probably meta, but could appear in dialogue — strip only outside quotes.
_META_WEAK_RE = re.compile(
    r"(we(?:'ll| will| need| have| should| can| must)\b"
    r"|i(?:'ll| will| am going to| need to| should)\b"
    r"|here (?:is|'s) (?:the|my|what)|current text|produce (?:about|roughly|the))",
    re.IGNORECASE)


def _strip_meta(text: str) -> str:
    """Remove sentences that are model planning/meta rather than story prose. Strong meta
    markers are always removed; weaker ones are removed only when the sentence has no quotes
    (so it can't be a line of dialogue)."""
    if not text:
        return text
    # Un-glue run-together sentences ("produce.Ace listened") so meta can be isolated.
    # Only before a capital LETTER — never before a quote, or we'd split "decent?" into
    # a sentence + a lone closing-quote fragment that then gets dropped.
    text = re.sub(r'([.!?])([A-Z])', r'\1 \2', text)
    out_paras = []
    for para in re.split(r"\n{2,}", text):
        kept = []
        for snt in re.split(r"(?<=[.!?])\s+", para.strip()):
            s = snt.strip()
            if not s or not re.search(r"\w", s):   # empty or punctuation-only fragment
                continue
            if _META_STRONG_RE.search(s):
                continue
            has_quote = any(q in s for q in '"“”')
            if not has_quote and _META_WEAK_RE.search(s):
                continue
            kept.append(s)
        if kept:
            out_paras.append(" ".join(kept))
    return "\n\n".join(out_paras)


def _clean_prose(text: str) -> str:
    """Full cleanup pipeline for one generated passage: strip meta, then de-duplicate."""
    return _dedupe(_strip_meta(text or ""))


def _reading_order(story: dict) -> list:
    """Flatten timelines/beats/flashbacks into linear reading order.

    Each item: {key, beat?, kind:'beat'|'flashback'|'flashback_return', parallel?:timeline, fb?}.
    Main timeline beats by `order`; a parallel timeline's beats are spliced in right after its
    `anchorBeatId`; flashbacks open at their startAnchorBeat and return at their endAnchorBeat.
    """
    beats = story.get("beats", [])
    timelines = story.get("timelines", [])
    parallels = [t for t in timelines if t.get("kind") == "parallel"]
    main_ids = {t["id"] for t in timelines if t.get("kind") != "parallel"}
    # Some beats may sit on the implicit main lane even with no timeline row.
    main_beats = sorted([b for b in beats if b.get("timelineId") in main_ids
                         or b.get("timelineId") in (None, "", "main")],
                        key=lambda b: b.get("order", 0))
    par_by_anchor: dict = {}
    for t in parallels:
        bs = sorted([b for b in beats if b.get("timelineId") == t["id"]],
                    key=lambda b: b.get("order", 0))
        par_by_anchor.setdefault(t.get("anchorBeatId"), []).append((t, bs))
    fb_start: dict = {}
    fb_end: dict = {}
    for fb in story.get("flashbacks", []):
        fb_start.setdefault(fb.get("startAnchorBeatId"), []).append(fb)
        fb_end.setdefault(fb.get("endAnchorBeatId"), []).append(fb)

    seq = []
    for b in main_beats:
        for fb in fb_start.get(b["id"], []):
            seq.append({"key": "fbin:" + fb["id"], "kind": "flashback", "fb": fb})
        seq.append({"key": b["id"], "kind": "beat", "beat": b})
        for fb in fb_end.get(b["id"], []):
            seq.append({"key": "fbout:" + fb["id"], "kind": "flashback_return", "fb": fb})
        for (t, bs) in par_by_anchor.get(b["id"], []):
            for pb in bs:
                seq.append({"key": pb["id"], "kind": "beat", "beat": pb, "parallel": t})
    return seq


def _active_traits(ch: dict, beat_pos: dict, cur_idx: int) -> list:
    """Traits/abilities a character has acquired by the current reading position.
    A trait with no acquiredAtBeat (or an unknown beat) is treated as always-on."""
    out = []
    for tr in ch.get("traits", []):
        at = tr.get("acquiredAtBeat")
        pos = beat_pos.get(at, -1) if at else -1
        if pos <= cur_idx:
            out.append(tr)
    return out


def _scene_brief(story: dict, item: dict, beat_pos: dict, cur_idx: int) -> tuple:
    """Returns (label, goal_block, instruction) for one reading-order item."""
    chars = {c["id"]: c for c in story.get("characters", [])}
    locs = {l["id"]: l for l in story.get("locations", [])}
    if item["kind"] in ("flashback", "flashback_return"):
        fb = item["fb"]
        if item["kind"] == "flashback_return":
            return (fb.get("label") or "Flashback", "",
                    "Write ONE short paragraph that transitions OUT of the flashback and back to the "
                    "present-day storyline. Make the return feel deliberate.")
        goal = f"Flashback goal: {fb.get('label','')} — {fb.get('summary','')}".strip()
        return (fb.get("label") or "Flashback", goal,
                "This scene is a FLASHBACK to an earlier time. Open with a clear transition into "
                "memory and write it in a slightly distinct texture from the present-day scenes.")
    b = item["beat"]
    lines = []
    loc = locs.get(b.get("locationId"))
    if loc:
        lines.append(f"Setting: {loc.get('name','')} — {loc.get('description','')}".strip())
    present = []
    for cid in b.get("characterIds", []):
        c = chars.get(cid)
        if not c:
            continue
        traits = _active_traits(c, beat_pos, cur_idx)
        tstr = "; ".join(f"{t.get('label','')} ({t.get('kind','trait')})" for t in traits)
        present.append(f"- {c.get('name','?')}"
                       + (f" [{c.get('role','')}]" if c.get("role") else "")
                       + (f" — {c.get('description','')}" if c.get("description") else "")
                       + (f" — known abilities/traits now: {tstr}" if tstr else " — no special abilities yet"))
    if present:
        lines.append("Present in this scene:\n" + "\n".join(present))
    goal = b.get("summary") or b.get("title") or "Continue the story."
    lines.append(f"What happens (the beat to dramatize): {b.get('title','')} — {goal}")
    if b.get("notes"):
        lines.append(f"Notes: {b['notes']}")
    instr = ""
    if item.get("parallel"):
        instr = ("This happens SIMULTANEOUSLY with the previous main-timeline events (a parallel "
                 "thread). Open with a 'Meanwhile' style shift so the reader feels the concurrency.")
    return (b.get("title") or "Scene", "\n".join(lines), instr)


def _tail(text: str, words: int = 150) -> str:
    w = (text or "").split()
    return " ".join(w[-words:])


class StoryJob:
    """Async, one-at-a-time full-story generator. Mirrors TrainJob's lifecycle."""

    def __init__(self):
        self.lock = threading.Lock()
        self._reset()

    def _reset(self):
        self.state = "idle"     # idle | running | done | error
        self.step = 0
        self.total = 0
        self.sid = None
        self.message = ""
        self.current = ""

    def status(self) -> dict:
        return {"state": self.state, "step": self.step, "total": self.total,
                "sid": self.sid, "message": self.message, "current": self.current}

    def start(self, sid: str) -> dict:
        with self.lock:
            if self.state == "running":
                raise RuntimeError("a story is already generating — wait for it to finish")
            if not (MGR.key or "").startswith(("story:", "llm:")):
                raise RuntimeError("load the Story model (or the Language model) first")
            story = json.load(open(_story_path(sid), encoding="utf-8"))
            seq = _reading_order(story)
            if not seq:
                raise RuntimeError("add at least one plot point to the timeline first")
            self._reset()
            self.state = "running"
            self.sid = sid
            self.total = len(seq)
        threading.Thread(target=self._run, args=(sid,), daemon=True).start()
        return self.status()

    def _gen_scene(self, story, item, beat_pos, idx, synopsis, tail) -> str:
        label, goal, instr = _scene_brief(story, item, beat_pos, idx)
        self.current = label
        pov = story.get("pov", "third-person limited")
        tense = story.get("tense", "past")
        style = story.get("styleNotes", "")
        genre = story.get("genre", "")
        head = (f"Continuous novel. Genre: {genre}. Point of view: {pov}. Tense: {tense}-tense."
                + (f" Style: {style}." if style else ""))
        base = [head, ""]
        base.append("STORY SO FAR (running summary):\n" + (synopsis or "This is the very opening of the story."))
        if tail:
            base.append("\nEND OF THE PREVIOUS SCENE (continue seamlessly from here):\n" + tail)
        base.append("\nWRITE THE NEXT SCENE:")
        if goal:
            base.append(goal)
        if instr:
            base.append(instr)

        short = item["kind"] == "flashback_return"
        try:
            chapter_words = int(story.get("chapterWords") or 0)
        except (TypeError, ValueError):
            chapter_words = 0

        # Chapter mode (opt-in via story["chapterWords"]): build a long, multi-pass
        # chapter for immersive beats so a novel can reach ~100k words in ~40 beats.
        if chapter_words > 0 and not short:
            return self._gen_chapter(base, label, pov, tense, chapter_words)

        # Legacy short-scene behavior (any story without chapterWords, and all
        # flashback-return transitions).
        body = list(base)
        body.append(f"\nWrite {'one short paragraph' if short else '2–4 paragraphs'} of immersive "
                    f"{tense}-tense, {pov} prose. Show the scene; do not summarize or add headings.")
        return _clean_prose(story_write(WRITER_SYSTEM, "\n".join(body),
                                        max_tokens=500 if short else 1400, temperature=0.8))

    def _gen_chapter(self, base, label, pov, tense, target) -> str:
        """Write one chapter as a few forward-moving passes. Length is driven by the number
        of passes (NOT by telling the model a word count — that makes gpt-oss 'think out loud'
        about the target and leak meta text into the prose). We stop as soon as the model
        winds down, so a chapter is never padded with repetition to hit a number."""
        parts, words, passes = [], 0, 0
        max_passes = 4
        while passes < max_passes:
            body = list(base)
            if not parts:
                body.append(
                    f"\nWrite this as immersive {tense}-tense, {pov} prose — several linked scenes of "
                    f"action, dialogue, and sensory detail that dramatize the above. Write ONLY story "
                    f"prose: no headings, notes, planning, word counts, or meta commentary of any kind. "
                    f"Do not restate the premise or character descriptions, and never repeat a sentence "
                    f"or phrase.")
            else:
                body.append(
                    "\nHere is the end of what you have written so far:\n"
                    + _tail("\n\n".join(parts), 200)
                    + "\n\nContinue the scene from exactly there, writing ONLY new story prose that "
                    "moves events forward. Do not repeat or rephrase anything above. Absolutely no "
                    "planning, notes, word counts, or meta commentary — story text only.")
            self.current = f"{label} — {words}w" if parts else label
            chunk = _clean_prose((story_write(WRITER_SYSTEM, "\n".join(body),
                             max_tokens=1400, temperature=0.85) or "").strip())
            new_words = len(chunk.split())
            if chunk:
                parts.append(chunk)
                words += new_words
            passes += 1
            # Stop when the model winds down (little new prose) or we've reached length.
            if new_words < 120 or words >= int(target * 0.9):
                break
        return _clean_prose("\n\n".join(parts))

    def _run(self, sid):
        try:
            with MGR.lock:
                story = json.load(open(_story_path(sid), encoding="utf-8"))
                seq = _reading_order(story)
                beat_pos = {it["key"]: i for i, it in enumerate(seq)}
                # also map raw beat ids -> position for trait resolution
                for i, it in enumerate(seq):
                    if it["kind"] == "beat":
                        beat_pos[it["beat"]["id"]] = i
                gen = story.setdefault("generation", {})
                scenes = gen.setdefault("scenes", {})
                synopsis = ""
                tail = ""
                for i, item in enumerate(seq):
                    self.step = i + 1
                    prose = self._gen_scene(story, item, beat_pos, i, synopsis, tail)
                    label, _, _ = _scene_brief(story, item, beat_pos, i)
                    scenes[item["key"]] = {"prose": prose, "label": label, "status": "done",
                                           "synopsisBefore": synopsis, "tailBefore": tail,
                                           "parallel": bool(item.get("parallel")),
                                           "kind": item["kind"]}
                    gen["order"] = [it["key"] for it in seq]
                    story_save(story)   # incremental persist so the UI can stream
                    tail = _tail(prose)
                    # update rolling synopsis (map-reduce) to keep context small
                    try:
                        synopsis = story_write(
                            "You compress a novel-in-progress into a running summary.",
                            "Update the running summary of a novel so a writer can continue it "
                            "consistently. Keep it under 180 words. Include every plot fact, "
                            "character state, relationship, and unresolved thread. Output ONLY the "
                            "summary.\n\nCURRENT SUMMARY:\n" + (synopsis or "(none yet)")
                            + "\n\nNEW SCENE:\n" + _tail(prose, 700),
                            max_tokens=600, temperature=0.2, timeout=300)
                    except Exception:
                        pass  # keep prior synopsis on a summarize hiccup
                    gen["rollingSynopsis"] = synopsis
                    story_save(story)
                gen["fullStory"] = _assemble_markdown(story, seq, scenes)
                story_save(story)
            self.state, self.message = "done", "story complete"
        except Exception as e:
            self.state, self.message = "error", str(e)


def _assemble_markdown(story, seq, scenes) -> str:
    parts = [f"# {story.get('title','Untitled')}", ""]
    for item in seq:
        sc = scenes.get(item["key"])
        if not sc or not sc.get("prose"):
            continue
        parts.append(sc["prose"].strip())
        parts.append("")
        parts.append("* * *")
        parts.append("")
    while parts and parts[-1] in ("", "* * *"):
        parts.pop()
    return "\n".join(parts)


def story_generate(sid: str) -> dict:
    return STORYJOB.start(sid)


def story_regen_scene(sid: str, key: str) -> dict:
    """Regenerate one scene's prose, reusing its stored bounded context."""
    with MGR.lock:
        if not (MGR.key or "").startswith(("story:", "llm:")):
            raise RuntimeError("load the Story model (or the Language model) first")
        story = json.load(open(_story_path(sid), encoding="utf-8"))
        seq = _reading_order(story)
        beat_pos = {it["key"]: i for i, it in enumerate(seq)}
        for i, it in enumerate(seq):
            if it["kind"] == "beat":
                beat_pos[it["beat"]["id"]] = i
        item = next((it for it in seq if it["key"] == key), None)
        if not item:
            raise RuntimeError("scene not found (resave the story first)")
        idx = next(i for i, it in enumerate(seq) if it["key"] == key)
        scenes = story.setdefault("generation", {}).setdefault("scenes", {})
        prev = scenes.get(key, {})
        prose = STORYJOB._gen_scene(story, item, beat_pos, idx,
                                    prev.get("synopsisBefore", ""), prev.get("tailBefore", ""))
        label, _, _ = _scene_brief(story, item, beat_pos, idx)
        scenes[key] = {**prev, "prose": prose, "label": label, "status": "done",
                       "kind": item["kind"], "parallel": bool(item.get("parallel"))}
        story["generation"]["fullStory"] = _assemble_markdown(story, seq, scenes)
        story_save(story)
        return {"prose": prose, "key": key}


def story_export(sid: str) -> dict:
    story = json.load(open(_story_path(sid), encoding="utf-8"))
    gen = story.get("generation", {})
    md = gen.get("fullStory")
    if not md:
        seq = _reading_order(story)
        md = _assemble_markdown(story, seq, gen.get("scenes", {}))
    return {"markdown": md, "title": story.get("title", "story")}


STORYJOB = StoryJob()


# ---- Audiobook: text -> chaptered MP3 ---------------------------------------
def _safe_name(name: str) -> str:
    return re.sub(r"[^A-Za-z0-9 _-]", "", (name or "").strip())[:60].strip() or "audiobook"


def _split_sentences(text: str) -> list:
    return [s for s in re.split(r"(?<=[.!?])\s+", (text or "").strip()) if s.strip()]


    # Narration pause tiers (ms of silence appended after a chunk). Butt-joining
    # chunk WAVs with no gap is the main thing that made audiobooks sound rushed
    # and robotic — natural narration breathes between sentences and paragraphs.
SENT_GAP_MS, PARA_GAP_MS, CHAP_GAP_MS = 350, 800, 1400

# Per-engine chunk budget: each chunk is synthesized as an isolated utterance, so
# bigger chunks = fewer prosody resets = more natural flow. Kokoro handles whole
# paragraphs; XTTS has a hard ~250-char context; the rest sit in between.
ENGINE_CHUNK_CHARS = {"tts:kokoro": 600, "tts:chatterbox": 280, "tts:zonos": 350}
XTTS_CHUNK_CHARS = 240


def _chunk_text(text: str, max_chars: int = 350) -> list:
    """Chunk prose into TTS-sized pieces on sentence boundaries (never mid-sentence,
    except for pathologically long sentences which are split on word boundaries).
    Returns [(chunk, pause_ms)] — the silence to append after each chunk (sentence
    gap inside a paragraph, a longer beat at paragraph ends)."""
    chunks = []
    for para in re.split(r"\n{2,}", (text or "").strip()):
        para = " ".join(para.split())
        if not para:
            continue
        if len(para) <= max_chars:
            chunks.append([para, SENT_GAP_MS])
        else:
            cur = ""
            for s in _split_sentences(para):
                if len(s) > max_chars:
                    if cur:
                        chunks.append([cur, SENT_GAP_MS]); cur = ""
                    buf = ""
                    for w in s.split():
                        if len(buf) + len(w) + 1 > max_chars:
                            chunks.append([buf, SENT_GAP_MS]); buf = w
                        else:
                            buf = (buf + " " + w).strip()
                    if buf:
                        chunks.append([buf, SENT_GAP_MS])
                elif len(cur) + len(s) + 1 > max_chars:
                    chunks.append([cur, SENT_GAP_MS]); cur = s
                else:
                    cur = (cur + " " + s).strip()
            if cur:
                chunks.append([cur, SENT_GAP_MS])
        chunks[-1][1] = PARA_GAP_MS
    return [tuple(c) for c in chunks]


def _audiobook_sections(source: str, sid: str, text: str, title: str):
    """Return (book_title, [(chapter_title, body_text), ...])."""
    if source == "story":
        story = json.load(open(_story_path(sid), encoding="utf-8"))
        gen = story.get("generation", {}) or {}
        scenes = gen.get("scenes", {}) or {}
        order = gen.get("order") or list(scenes.keys())
        secs = []
        for i, k in enumerate(order, 1):
            sc = scenes.get(k) or {}
            prose = (sc.get("prose") or "").strip()
            if prose:
                secs.append((sc.get("label") or f"Chapter {i}", prose))
        if not secs:
            raise RuntimeError("this story has no generated prose yet — Generate it first")
        return (story.get("title") or "Audiobook"), secs
    # pasted / uploaded text
    t = (text or "").strip()
    if not t:
        raise RuntimeError("no text provided")
    secs = []
    if re.search(r"^\s{0,3}#{1,6}\s", t, re.M):                 # markdown headings -> chapters
        cur_title, cur = "Chapter 1", []
        for line in t.split("\n"):
            m = re.match(r"^\s{0,3}#{1,6}\s+(.*)", line)
            if m:
                if cur:
                    secs.append((cur_title, "\n".join(cur).strip())); cur = []
                cur_title = m.group(1).strip() or f"Chapter {len(secs) + 1}"
            else:
                cur.append(line)
        if cur:
            secs.append((cur_title, "\n".join(cur).strip()))
    elif "* * *" in t:                                          # story-export scene breaks
        secs = [(f"Chapter {i + 1}", b.strip()) for i, b in enumerate(t.split("* * *"))]
    else:
        secs = [("Audiobook", t)]
    secs = [(ti, bo) for ti, bo in secs if bo.strip()]
    if not secs:
        raise RuntimeError("no usable text after splitting")
    return (_safe_name(title) if title else "Audiobook"), secs


def _wavs_to_mp3(wavs: list, mp3: str):
    """Concatenate chunk WAVs and encode one chapter MP3 in a single ffmpeg pass. ffmpeg
    reads float32 (format 3) WAVs that Python's stdlib `wave` module cannot, and resamples
    to a uniform MP3 — so mixed sample rates/formats across engines are handled. EBU R128
    loudness normalization (-19 LUFS, audiobook standard) evens out level drift between
    chunks/engines; loudnorm upsamples internally, so pin the output rate."""
    lst = mp3 + ".txt"
    with open(lst, "w", encoding="utf-8") as f:
        for w in wavs:
            f.write("file '" + os.path.abspath(w).replace("'", "'\\''") + "'\n")
    try:
        r = subprocess.run([_ffmpeg(), "-y", "-f", "concat", "-safe", "0", "-i", lst,
                            "-af", "loudnorm=I=-19:TP=-2:LRA=11", "-ar", "44100",
                            "-b:a", "128k", "-ac", "1", mp3],
                           capture_output=True, text=True, timeout=1800, **NOWIN)
    finally:
        try:
            os.remove(lst)
        except Exception:
            pass
    if r.returncode != 0 or not os.path.isfile(mp3):
        raise RuntimeError("ffmpeg mp3 encode failed: " + (r.stderr or "")[-300:])


def _concat_mp3s(mp3s: list, out: str):
    lst = out + ".txt"
    with open(lst, "w", encoding="utf-8") as f:
        for m in mp3s:
            f.write("file '" + m.replace("'", "'\\''") + "'\n")
    try:
        subprocess.run([_ffmpeg(), "-y", "-f", "concat", "-safe", "0", "-i", lst,
                        "-c", "copy", out], capture_output=True, text=True, timeout=1800, **NOWIN)
    finally:
        try:
            os.remove(lst)
        except Exception:
            pass


class AudiobookJob:
    """Async narration job: chunk each chapter, synth on the loaded TTS worker, stitch
    per-chapter + full-book MP3s. Mirrors StoryJob/TrainJob's lifecycle."""

    def __init__(self):
        self.lock = threading.Lock()
        self._reset()

    def _reset(self):
        self.state = "idle"; self.step = 0; self.total = 0
        self.message = ""; self.current = ""; self.book = None; self.files = []

    def status(self) -> dict:
        return {"state": self.state, "step": self.step, "total": self.total,
                "message": self.message, "current": self.current,
                "book": self.book, "files": self.files}

    def start(self, params: dict) -> dict:
        with self.lock:
            if self.state == "running":
                raise RuntimeError("an audiobook is already rendering — wait for it to finish")
            if not (MGR.key or "").startswith(("tts:", "voicemodel:")):
                raise RuntimeError("load a narration voice first (Kokoro / Chatterbox / Zonos / your voice)")
            title, secs = _audiobook_sections(params.get("source", "story"), params.get("id", ""),
                                              params.get("text", ""), params.get("title", ""))
            max_chars = (XTTS_CHUNK_CHARS if (MGR.key or "").startswith("voicemodel:")
                         else ENGINE_CHUNK_CHARS.get(MGR.key or "", 350))
            plan = [(ti, _chunk_text(bo, max_chars)) for ti, bo in secs]
            plan = [(ti, ch) for ti, ch in plan if ch]
            total = sum(len(ch) for _, ch in plan)
            if not total:
                raise RuntimeError("nothing to narrate")
            self._reset()
            self.state = "running"; self.total = total
            self.book = _safe_name(title)
            self._params = dict(params); self._plan = plan; self._title = title
        threading.Thread(target=self._run, daemon=True).start()
        return self.status()

    def _run(self):
        try:
            with MGR.lock:
                worker = MGR.workers.get("tts") or MGR.workers.get("voicemodel")
                if worker is None:
                    raise RuntimeError("the narration voice was unloaded")
                voice = self._params.get("voice") or "af_heart"
                ref = _resolve_ref(self._params)         # cloning reference (Zonos/Chatterbox)
                controls = _tts_controls(self._params)   # speed / pitch / emotion / etc.
                bookdir = os.path.join(AUDIOBOOKS, self.book)
                os.makedirs(bookdir, exist_ok=True)
                tmp = os.path.join(bookdir, "_wav")
                os.makedirs(tmp, exist_ok=True)
                chapter_mp3s, done = [], 0
                for ci, (ti, chunks) in enumerate(self._plan, 1):
                    wavs = []
                    for j, (ch, pause_ms) in enumerate(chunks):
                        self.current = f"Ch {ci}/{len(self._plan)} — {ti}"
                        outw = os.path.join(tmp, f"c{ci:03d}_{j:04d}.wav")
                        if j == len(chunks) - 1:            # let the chapter end settle
                            pause_ms = max(pause_ms, CHAP_GAP_MS)
                        req = {"text": ch, "out": outw, "pad_ms": pause_ms, **controls}
                        if MGR.key.startswith("tts:") and not MGR.key.endswith(":zonos"):
                            req["voice"] = voice
                        if ref:
                            req["ref"] = ref
                        worker.call(req, timeout=600)
                        wavs.append(outw); done += 1; self.step = done
                    mp3 = os.path.join(bookdir, f"{ci:02d} - {_safe_name(ti)}.mp3")
                    _wavs_to_mp3(wavs, mp3)
                    chapter_mp3s.append(mp3)
                    self.files = [os.path.relpath(m, AUDIOBOOKS).replace("\\", "/") for m in chapter_mp3s]
                full = os.path.join(bookdir, f"{self.book}.mp3")
                if len(chapter_mp3s) > 1:
                    _concat_mp3s(chapter_mp3s, full)
                    self.files = self.files + [os.path.relpath(full, AUDIOBOOKS).replace("\\", "/")]
                shutil.rmtree(tmp, ignore_errors=True)
            self.state, self.message = "done", "audiobook complete"
        except Exception as e:
            self.state, self.message = "error", str(e)


AUDIOBOOKJOB = AudiobookJob()


class LullabyJob:
    """Async lullaby job, two engines:

    remix (default) — the recipe the ear tests picked: demucs drops the drums,
    the stem mix gets its dynamics flattened (so the loud second half can't drag
    the lullaby with it), then ACE-Step audio2audio re-imagines it with lullaby
    tags at moderate denoise, and a soft mastering chain keeps it gentle
    throughout. Closely resembles the original song.

    piano — the symbolic engine in lullabykit/pipeline.py: vocal melody
    transcription + key/chord detection, rebuilt as a rocking piano + music-box
    arrangement at 55-88 bpm, rendered with the Salamander grand.

    Mirrors AudiobookJob's lifecycle; pipeline progress comes from its
    "== N/6 ... ==" stage markers."""

    STAGE_LABELS = {1: "extracting audio", 2: "separating stems (GPU)",
                    3: "mixing stems", 4: "analyzing key/beats/chords + melody",
                    5: "arranging", 6: "rendering + mastering"}
    # melody-match has no transcribe/arrange stages of its own (no scale-snap,
    # no grid quantize — it traces the continuous pitch curve directly), so
    # its "== N/6 ..." markers land on different steps than the arranged engine
    MELODY_MATCH_LABELS = {1: "extracting audio", 2: "separating stems (GPU)",
                           3: "mixing stems", 4: "tracking pitch (FCPE) + rendering",
                           6: "mastering"}

    CONTINUOUS_INSTRUMENTS = ("cello", "violin", "flute", "synth_voice", "music_box")

    POLISH_TAGS = ("lullaby, solo piano, music box, celesta, gentle, calm, dreamy, "
                   "bedtime, instrumental, slow tempo, quiet, warm, soothing, "
                   "sleep music, soft, no drums, no vocals")

    # fallback for the Piano engine when the user renders without ever
    # touching the Tracks panel sliders (mirrors pipeline.py's default)
    DEFAULT_STEM_WEIGHTS = {"vocals": 1.0, "guitar": 1.0, "piano": 1.0,
                            "other": 1.0, "bass": 0.0, "drums": 0.0}

    REMIX_TAGS = ("lullaby, very soft felt piano, music box, hushed, delicate, "
                  "sparse, minimal, gentle, calm, dreamy, bedtime, instrumental, "
                  "slow tempo, very quiet, warm, soothing, sleep music for babies, "
                  "pianissimo, no drums, no vocals, no bass")
    REMIX_FLATTEN = ("acompressor=threshold=0.08:ratio=6:attack=150:release=900:"
                     "makeup=1.3,loudnorm=I=-20:TP=-2:LRA=4")
    # linear loudnorm: dynamic mode rides gain up in the quiet outro and
    # amplifies end-of-track artifacts
    REMIX_MASTER = ("highshelf=f=3800:g=-6,lowpass=f=7000,"
                    "loudnorm=I=-21:TP=-2:LRA=5:linear=true,"
                    "afade=t=in:d=3,areverse,afade=t=in:d=10,areverse")

    def __init__(self):
        self.lock = threading.Lock()
        self._reset()

    def _reset(self):
        self.state = "idle"; self.step = 0; self.total = 6
        self.message = ""; self.current = ""; self.name = None; self.files = []
        self.phase = ""

    def status(self) -> dict:
        return {"state": self.state, "step": self.step, "total": self.total,
                "message": self.message, "current": self.current,
                "name": self.name, "files": self.files, "phase": self.phase}

    def _begin(self, params: dict, need_audio: bool = True):
        """Shared validation + upload staging. Caller holds self.lock."""
        if self.state == "running":
            raise RuntimeError("a lullaby job is already running — wait for it to finish")
        if MGR.key:
            raise RuntimeError("stop the loaded model first — Lullaby runs its own engine "
                               "and needs the GPU")
        if not os.path.isfile(LULLABY_PY):
            raise RuntimeError("lullabykit is not installed (lullabykit/.venv missing)")
        if need_audio:
            audio = params.get("audio") or ""
            if not audio:
                raise RuntimeError("upload a song first")
            name = _safe_name(os.path.splitext(params.get("filename") or "song")[0]) or "song"
            ext = os.path.splitext(params.get("filename") or "")[1].lower() or ".mp3"
            src = os.path.join(TMP, f"lullaby_src_{name}{ext}")
            with open(src, "wb") as f:
                f.write(base64.b64decode(audio.split(",")[-1]))
        else:
            name = _safe_name(params.get("name") or "")
            if not name:
                raise RuntimeError("no song name given")
            # dummy path: cached-stage runs never read the source file, the
            # pipeline only derives its work dir from the basename
            src = os.path.join(TMP, f"lullaby_src_{name}.mp3")
        self._reset()
        self.state = "running"; self.name = name
        self._src = src

    def _knobs(self, params: dict):
        self._mode = params.get("mode", "remix")
        self._tempo = min(0.95, max(0.5, float(params.get("tempo", 0.72))))
        self._denoise = min(0.75, max(0.45, float(params.get("denoise", 0.60))))
        self._slowdown = min(1.0, max(0.8, float(params.get("slowdown", 1.0))))
        self._melody = params.get("melody", "auto")
        if self._melody not in ("auto", "vocals", "instruments", "vocals_other"):
            self._melody = "auto"
        self._focus = params.get("focus", "both")
        if self._focus not in ("both", "vocals", "instruments"):
            self._focus = "both"
        self._instrument = params.get("instrument", "cello")
        if self._instrument not in self.CONTINUOUS_INSTRUMENTS:
            self._instrument = "cello"
        # multitrack workbench: per-stem levels chosen by the user
        stems = params.get("stems")
        self._stems = None
        if isinstance(stems, dict) and stems:
            self._stems = {k: min(1.5, max(0.0, float(v)))
                           for k, v in stems.items()
                           if k in ("vocals", "guitar", "piano", "other", "bass", "drums")}
        # melody-match only: stems routed to the arranged (transcribe+quantize+
        # piano) pipeline instead of the solo instrument — e.g. vocals traced
        # on cello via `stems` above while an existing piano part is rebuilt
        # as a full arrangement via this, mixed together in one render
        arranged_stems = params.get("arranged_stems")
        self._arranged_stems = None
        if isinstance(arranged_stems, dict) and arranged_stems:
            self._arranged_stems = {k: min(1.5, max(0.0, float(v)))
                                    for k, v in arranged_stems.items()
                                    if k in ("vocals", "guitar", "piano", "other", "bass", "drums")}
        # ACE-Step a2a conditions poorly on sparse solo-piano/instrument input
        # — default OFF
        self._polish = (self._mode in ("piano", "melody-match")
                       and bool(params.get("polish", False)))
        if self._polish:
            self.total = 7   # extra ACE-Step polish stage

    def start(self, params: dict) -> dict:
        """One-shot: full pipeline + render in a single job (legacy path)."""
        with self.lock:
            self._begin(params)
            self._knobs(params)
            self.phase = "full"
        threading.Thread(target=self._run, daemon=True).start()
        return self.status()

    def analyze(self, params: dict) -> dict:
        """Workbench phase 1: stems, key/beats/chords, BOTH melody candidates
        (scores + piano-roll notes + audition previews), remix input, waveform.
        Everything cached so phase-2 renders are fast."""
        with self.lock:
            self._begin(params)
            self.phase = "analyze"
            self.total = 4
        threading.Thread(target=self._run_analyze, daemon=True).start()
        return self.status()

    def render(self, params: dict) -> dict:
        """Workbench phase 2: render from the cached analysis with the user's
        choices (engine, melody source, focus, denoise, tempo...)."""
        with self.lock:
            self._begin(params, need_audio=False)
            workdir = os.path.join(LULLABYKIT, "work", f"lullaby_src_{self.name}")
            if not os.path.isfile(os.path.join(workdir, "analysis.json")):
                self._reset()
                raise RuntimeError("analyze the song first")
            self._knobs(params)
            self.phase = "render"
        threading.Thread(target=self._run_render, daemon=True).start()
        return self.status()

    def _pipeline(self, extra: list) -> list:
        """Run lullabykit/pipeline.py streaming its stage markers into status."""
        env = dict(os.environ, PYTHONUTF8="1", PYTHONIOENCODING="utf-8",
                   PYTHONUNBUFFERED="1")
        p = subprocess.Popen([LULLABY_PY, LULLABY_PIPELINE, self._src, *extra],
                             stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
                             text=True, encoding="utf-8", errors="replace", env=env,
                             bufsize=1, **NOWIN)
        tail = []
        for line in p.stdout:
            line = line.strip()
            if not line:
                continue
            tail.append(line); tail[:] = tail[-30:]
            m = re.match(r"^== (\d)/6 ", line)
            if m:
                self.step = int(m.group(1))
                labels = (self.MELODY_MATCH_LABELS
                         if getattr(self, "_mode", None) == "melody-match"
                         else self.STAGE_LABELS)
                self.current = labels.get(self.step, line)
        p.wait()
        if p.returncode != 0:
            raise RuntimeError("pipeline failed: " + " | ".join(tail[-4:]))
        return tail

    def _remix(self, outbase: str, from_cache: bool = False) -> list:
        """ACE-Step remix engine: stems 1-3 via the pipeline, then flatten ->
        audio2audio -> soft master. Returns output files. from_cache rebuilds
        only the remix input (with the chosen focus) from already-separated
        stems — the fast path after a workbench analyze."""
        pre = ["--until-stage", "mix", "--remix-prep", "--remix-focus", self._focus]
        if self._stems:
            pre += ["--stem-weights", json.dumps(self._stems)]
        if from_cache:
            pre = ["--from-stage", "mix"] + pre
        self._pipeline(pre)
        stem = os.path.splitext(os.path.basename(self._src))[0]
        # remix_input.wav: vocals+other only, HPSS harmonic-only (drops the riser
        # sweeps / cymbal swells ACE renders as artifacts in buildups), tail
        # trimmed + faded so the model never sees the messy outro
        rin = os.path.join(LULLABYKIT, "work", stem, "remix_input.wav")
        if not os.path.isfile(rin):
            raise RuntimeError("remix input missing after separation")

        self.step, self.current = 4, "softening dynamics"
        soften = self.REMIX_FLATTEN
        if self._slowdown < 0.999:
            soften += f",atempo={self._slowdown}"
        flat = os.path.join(TMP, "lullaby_flat.wav")
        rc, _, err = _run([_ffmpeg(), "-y", "-i", rin, "-af", soften,
                           "-ar", "44100", flat], timeout=600)
        if rc != 0:
            raise RuntimeError("flatten failed: " + err[-200:])

        self.step, self.current = 5, "re-imagining with ACE-Step"
        raw = outbase + "_raw.flac"
        try:
            MGR.load("music", {})
            cmd = [sys.executable, MUSIC, "--out", raw, "--tags", self.REMIX_TAGS,
                   "--model", "turbo", "--audio", flat,
                   "--denoise", str(self._denoise), "--format", "flac"]
            rc, sout, err = _run(cmd, timeout=1800)
            if rc != 0 or not os.path.isfile(raw):
                raise RuntimeError(err or sout or "remix failed")
        finally:
            try:
                MGR.stop()
            except Exception:
                pass

        self.step, self.current = 6, "soft mastering"
        mp3, wav = outbase + ".mp3", outbase + ".wav"
        for dst, enc in ((mp3, ["-c:a", "libmp3lame", "-b:a", "192k"]),
                         (wav, [])):
            rc, _, err = _run([_ffmpeg(), "-y", "-i", raw, "-af", self.REMIX_MASTER,
                               "-ar", "44100", *enc, dst], timeout=600)
            if rc != 0:
                raise RuntimeError("master failed: " + err[-200:])
        os.remove(raw)
        try:
            os.remove(flat)
        except Exception:
            pass
        return [mp3, wav]

    def _melody_match(self, outbase: str, from_cache: bool = False) -> list:
        """melody-match engine: traces the singer's actual continuous pitch
        curve (FCPE, no scale-snap / no grid quantize) on a single
        portamento-capable instrument — see pipeline.py's module docstring."""
        pre = ["--engine", "melody-match", "--continuous-instrument", self._instrument,
               "--tempo-scale", str(self._tempo)]
        if self._stems:
            pre += ["--melody-stems", json.dumps(self._stems)]
        if self._arranged_stems:
            pre += ["--arranged-stems", json.dumps(self._arranged_stems)]
        if from_cache:
            pre = ["--from-stage", "mix"] + pre
        self._pipeline([*pre, "--out", outbase])
        return [outbase + ".mp3", outbase + ".wav"]

    def _outbase(self) -> str:
        outdir = os.path.join(LULLABIES, self.name)
        os.makedirs(outdir, exist_ok=True)
        return os.path.join(outdir, f"{self.name}_lullaby")

    def _run_analyze(self):
        try:
            with MGR.lock:
                self._pipeline(["--until-stage", "transcribe", "--remix-prep",
                                "--analysis-extras"])
            self.step = self.total
            self.state = "done"
            self.message = "analysis ready — pick a melody and an engine, then render"
        except Exception as e:
            self.state, self.message = "error", str(e)
        finally:
            try:
                os.remove(self._src)
            except Exception:
                pass

    def _run_render(self):
        try:
            with MGR.lock:
                outbase = self._outbase()
                if self._mode == "remix":
                    files = self._remix(outbase, from_cache=True)
                elif self._mode == "melody-match":
                    files = self._melody_match(outbase, from_cache=True)
                    files = self._apply_polish(outbase, files)
                else:
                    # the Tracks panel selection IS the melody source — re-run
                    # transcription from whatever the user ticked/weighted
                    # (stems/separation stay cached; only transcribe+arrange+
                    # render re-run, so this is fast even though it "restarts"
                    # from an earlier stage than the remix engine does)
                    stems = self._stems or self.DEFAULT_STEM_WEIGHTS
                    self._pipeline(["--from-stage", "transcribe",
                                    "--melody-stems", json.dumps(stems),
                                    "--tempo-scale", str(self._tempo),
                                    "--out", outbase])
                    files = self._finish_piano(outbase)
                    files = self._apply_polish(outbase, files)
                self.files = [os.path.relpath(f, LULLABIES).replace("\\", "/")
                              for f in files]
                self.step = self.total
            self.state, self.message = "done", "lullaby complete " + (self.message or "")
        except Exception as e:
            self.state, self.message = "error", str(e)

    def _finish_piano(self, outbase: str) -> list:
        """mp3-encode the piano render (+ optional experimental ACE polish)."""
        wav = outbase + ".wav"
        if not os.path.isfile(wav):
            raise RuntimeError("pipeline produced no audio")
        mp3 = outbase + ".mp3"
        rc, _, err = _run([_ffmpeg(), "-y", "-i", wav, "-codec:a", "libmp3lame",
                           "-b:a", "192k", mp3], timeout=300)
        if rc != 0:
            raise RuntimeError("mp3 encode failed: " + err[-300:])
        return [mp3, wav]

    def _run(self):
        try:
            with MGR.lock:
                outbase = self._outbase()

                if self._mode == "remix":
                    self.files = [os.path.relpath(f, LULLABIES).replace("\\", "/")
                                  for f in self._remix(outbase)]
                    self.step = self.total
                    self.state, self.message = "done", "lullaby complete"
                    return

                if self._mode == "melody-match":
                    files = self._apply_polish(outbase, self._melody_match(outbase))
                    self.files = [os.path.relpath(f, LULLABIES).replace("\\", "/")
                                  for f in files]
                    self.step = self.total
                    self.state, self.message = "done", "lullaby complete"
                    return

                self._pipeline(["--tempo-scale", str(self._tempo),
                                "--melody-source", self._melody, "--out", outbase])
                outputs = self._finish_piano(outbase)
                outputs = self._apply_polish(outbase, outputs)

                self.files = [os.path.relpath(f, LULLABIES).replace("\\", "/")
                              for f in outputs]
                self.step = self.total
            self.state, self.message = "done", "lullaby complete " + (self.message or "")
        except Exception as e:
            self.state, self.message = "error", str(e)
        finally:
            try:
                os.remove(self._src)
            except Exception:
                pass

    def _apply_polish(self, outbase: str, files: list) -> list:
        """Optional ACE-Step audio2audio polish pass over an already-rendered
        lullaby wav — re-textures the sound while keeping its notes/tempo.
        Applies to Piano or Melody Match output alike. Non-fatal: on any
        failure the clean render ships as-is. Prepends the polished file to
        `files` on success."""
        if not self._polish:
            return files
        wav = outbase + ".wav"
        if not os.path.isfile(wav):
            return files
        self.step, self.current = self.total, "polishing with ACE-Step"
        try:
            MGR.load("music", {})
            cmd = [sys.executable, MUSIC, "--out", "-",
                   "--tags", self.POLISH_TAGS, "--model", "turbo",
                   "--audio", wav, "--denoise", "0.40", "--format", "flac"]
            rc, sout, err = _run(cmd, timeout=1200)
            line = (sout or "").strip().splitlines()[-1] if (sout or "").strip() else ""
            if rc != 0 or not line.startswith("{"):
                raise RuntimeError(err or "polish failed")
            raw = outbase + "_polish_raw.flac"
            with open(raw, "wb") as f:
                f.write(base64.b64decode(json.loads(line)["audios"][0].split(",")[-1]))
            # ACE renders hot (~-1dB mean) — bring it back to lullaby loudness
            # and restore the fade-out before shipping
            polished = outbase + "_polished.mp3"
            rc, _, err = _run([_ffmpeg(), "-y", "-i", raw, "-af",
                               "loudnorm=I=-16:TP=-1.5:LRA=9,"
                               "afade=t=in:d=3,areverse,afade=t=in:d=7,areverse",
                               "-ar", "44100", "-c:a", "libmp3lame",
                               "-b:a", "192k", polished], timeout=600)
            os.remove(raw)
            if rc != 0:
                raise RuntimeError("polish master failed: " + err[-200:])
            return [polished] + list(files)
        except Exception as pe:
            self.message = f"(polish skipped: {pe})"
            return files
        finally:
            try:
                MGR.stop()
            except Exception:
                pass


LULLABYJOB = LullabyJob()

NOTE_NAMES = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"]


def lullaby_info(name: str) -> dict:
    """Workbench payload for an analyzed song: key/tempo, per-bar chords, both
    melody candidates (scores + notes for the piano-roll), waveform envelope,
    and audition preview URLs."""
    name = _safe_name(name)
    work = os.path.join(LULLABYKIT, "work", f"lullaby_src_{name}")
    apath = os.path.join(work, "analysis.json")
    if not os.path.isfile(apath):
        raise RuntimeError("song not analyzed yet")
    analysis = json.load(open(apath, encoding="utf-8"))
    out = {"name": name,
           "key": NOTE_NAMES[analysis["key_pc"]] + (" minor" if analysis["key_mode"] == "minor"
                                                    else " major"),
           "tempo": round(analysis["tempo"]),
           "bars": [{"start": round(b["start"], 2),
                     "chord": NOTE_NAMES[b["root"]] + ("m" if b["qual"] == "min" else "")}
                    for b in analysis["bars"]]}
    wpath = os.path.join(work, "waveform.json")
    if os.path.isfile(wpath):
        out["waveform"] = json.load(open(wpath, encoding="utf-8"))
    spath = os.path.join(work, "stems.json")
    if os.path.isfile(spath):
        stems = json.load(open(spath, encoding="utf-8"))
        out["stems"] = {
            k: {**v, "preview": f"/lullaby_preview/{name}/stem_{k}.mp3"}
            for k, v in stems.items()}
    return out


# ---- Track Splitter (any song -> 6 individual instrument tracks) --------------
# This is deliberately just the Lullaby tab's analyze phase (same two-pass
# Demucs separation, same stem previews) with no arrangement step — and it
# stages its input under the identical lullaby_src_<name> work-dir name, so a
# song split here is already cached if you later open it in the Lullaby tab
# (and vice versa): the pipeline's separate-stage cache guard skips re-running
# Demucs when the stems are already on disk.
SPLITS = os.path.join(HERE, "splits")
os.makedirs(SPLITS, exist_ok=True)


def split_list() -> list:
    if not os.path.isdir(SPLITS):
        return []
    out = []
    for name in sorted(os.listdir(SPLITS)):
        d = os.path.join(SPLITS, name)
        if os.path.isdir(d):
            out.append({"name": name, "files": sorted(os.listdir(d))})
    return out


def split_zip_bytes(name: str) -> bytes:
    import io
    import zipfile
    base = os.path.normpath(os.path.join(SPLITS, _safe_name(name)))
    if not (base.startswith(os.path.normpath(SPLITS) + os.sep) and os.path.isdir(base)):
        raise RuntimeError("split set not found")
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, "w", zipfile.ZIP_DEFLATED) as z:
        for fn in os.listdir(base):
            z.write(os.path.join(base, fn), fn)
    return buf.getvalue()


class SplitJob:
    """Async stem-splitting job: separates a song into 6 instrument tracks and
    saves them as individual downloadable files. Mirrors LullabyJob's
    lifecycle but stops after separation — no arrangement, no remix."""

    STAGE_LABELS = {1: "extracting audio", 2: "separating stems (GPU)"}

    def __init__(self):
        self.lock = threading.Lock()
        self._reset()

    def _reset(self):
        self.state = "idle"; self.step = 0; self.total = 3
        self.message = ""; self.current = ""; self.name = None; self.files = []

    def status(self) -> dict:
        return {"state": self.state, "step": self.step, "total": self.total,
                "message": self.message, "current": self.current,
                "name": self.name, "files": self.files}

    def start(self, params: dict) -> dict:
        with self.lock:
            if self.state == "running":
                raise RuntimeError("a split is already running — wait for it to finish")
            if MGR.key:
                raise RuntimeError("stop the loaded model first — Track Splitter runs "
                                   "its own engine and needs the GPU")
            if not os.path.isfile(LULLABY_PY):
                raise RuntimeError("lullabykit is not installed (lullabykit/.venv missing)")
            audio = params.get("audio") or ""
            if not audio:
                raise RuntimeError("upload a song first")
            name = _safe_name(os.path.splitext(params.get("filename") or "song")[0]) or "song"
            ext = os.path.splitext(params.get("filename") or "")[1].lower() or ".mp3"
            # same lullaby_src_<name> naming as LullabyJob — shares its cache
            src = os.path.join(TMP, f"lullaby_src_{name}{ext}")
            with open(src, "wb") as f:
                f.write(base64.b64decode(audio.split(",")[-1]))
            self._reset()
            self.state = "running"; self.name = name
            self._src = src
            self._format = "wav" if params.get("format") == "wav" else "mp3"
        threading.Thread(target=self._run, daemon=True).start()
        return self.status()

    def _run(self):
        try:
            with MGR.lock:
                env = dict(os.environ, PYTHONUTF8="1", PYTHONIOENCODING="utf-8",
                           PYTHONUNBUFFERED="1")
                p = subprocess.Popen(
                    [LULLABY_PY, LULLABY_PIPELINE, self._src,
                     "--until-stage", "transcribe", "--analysis-extras"],
                    stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
                    text=True, encoding="utf-8", errors="replace", env=env,
                    bufsize=1, **NOWIN)
                tail = []
                for line in p.stdout:
                    line = line.strip()
                    if not line:
                        continue
                    tail.append(line); tail[:] = tail[-30:]
                    m = re.match(r"^== (\d)/6 ", line)
                    if m:
                        self.step = min(int(m.group(1)), 2)
                        self.current = self.STAGE_LABELS.get(self.step, line)
                p.wait()
                if p.returncode != 0:
                    raise RuntimeError("split failed: " + " | ".join(tail[-4:]))

                workdir = os.path.join(LULLABYKIT, "work", f"lullaby_src_{self.name}")
                stem_dir = os.path.join(workdir, "stems", "htdemucs_ft", "raw")
                if not os.path.isdir(stem_dir):
                    raise RuntimeError("no stems were produced")

                self.step, self.current = 3, "saving tracks"
                outdir = os.path.join(SPLITS, self.name)
                os.makedirs(outdir, exist_ok=True)
                files = []
                for stem in ("vocals", "guitar", "piano", "other", "bass", "drums"):
                    src_wav = os.path.join(stem_dir, f"{stem}.wav")
                    if not os.path.isfile(src_wav):
                        continue
                    if self._format == "wav":
                        dst = os.path.join(outdir, f"{stem}.wav")
                        shutil.copyfile(src_wav, dst)
                    else:
                        dst = os.path.join(outdir, f"{stem}.mp3")
                        rc, _, err = _run([_ffmpeg(), "-y", "-i", src_wav, "-codec:a",
                                           "libmp3lame", "-b:a", "192k", dst], timeout=300)
                        if rc != 0:
                            raise RuntimeError(f"mp3 encode failed for {stem}: {err[-200:]}")
                    files.append(os.path.relpath(dst, SPLITS).replace("\\", "/"))
                if not files:
                    raise RuntimeError("no tracks were saved")
                self.files = files
                self.step = self.total
            self.state, self.message = "done", "tracks saved"
        except Exception as e:
            self.state, self.message = "error", str(e)
        finally:
            try:
                os.remove(self._src)
            except Exception:
                pass


SPLITJOB = SplitJob()


# ---- Sprite Studio ------------------------------------------------------------
# Pose script per action: one line per frame, written to read as an animation loop
# at these counts. Dict order = row order on the combined sheet.
SPRITE_ACTIONS = {
    "idle": [
        "standing relaxed at rest, arms loose at the sides, mid exhale",
        "standing relaxed, chest slightly risen on the inhale, head lifted a touch",
        "standing relaxed at the top of the breath, shoulders slightly raised",
        "standing relaxed, settling back down on the exhale, shoulders dropping",
    ],
    "walk": [
        "mid-step with the right foot planted forward, left foot back on its toes, arms swinging opposite to the legs",
        "weight settling low onto the front right foot, both knees bent mid-stride",
        "legs crossing beneath the body mid-step, standing tall, arms at mid swing",
        "pushing off the right foot, body at its highest point, left leg reaching forward",
        "mid-step with the left leg planted forward and knee bent, right leg trailing behind on its toes, right arm swung forward",
        "legs crossing beneath the body mid-step, back heel lifting off the ground",
    ],
    "run": [
        "sprinting, leaning far forward, right foot striking the ground, arms pumping",
        "sprinting, pushing off hard from the right foot, body low and leaning into the sprint",
        "sprinting airborne, both feet off the ground, legs scissored wide mid-stride",
        "sprinting, left foot striking the ground, opposite arm pumping forward",
        "sprinting, pushing off hard from the left foot, back leg kicked up high behind",
        "sprinting airborne in full stride, front leg reaching far ahead, back leg kicked up high behind",
    ],
    "jump": [
        "crouched low about to jump, knees deeply bent, arms swung back behind the body",
        "leaping upward, legs fully extended pushing off the ground, arms thrown upward",
        "airborne at the top of the jump arc, knees tucked up, arms out for balance",
        "landing from a jump, both knees deeply bent absorbing the impact, body crouched low, arms stretched out forward",
    ],
    "fall": [
        "falling through the air, arms raised, legs trailing, hair and clothes blown upward",
        "falling fast, body tilted forward, arms flailing slightly",
    ],
    "crouch": [
        "crouching down low on bent knees, one hand near the ground, head up and alert",
        "fully crouched, low and compact, weight coiled, ready to spring",
    ],
    "attack": [
        "winding up a punch, torso coiled back, fist pulled far back behind the head, front arm guarding",
        "maximum wind-up, fist still drawn back, weight loaded on the back foot",
        "lunging forward mid-punch, fist sweeping through the air",
        "punch fully extended at the end of the strike, arm straight out",
        "recovering balance after the punch, fists returning to a guard position",
    ],
    "hurt": [
        "flinching in pain from a hit, bent backward at the waist, head thrown back, eyes squeezed shut, one arm clutching the ribs",
        "staggering off balance from a hit, one foot slid back, grimacing, guarding with an arm",
    ],
    "death": [
        "struck down, body arched sharply backward, arms flung wide, head thrown back, eyes squeezed shut in pain",
        "collapsing, dropped onto both knees, body slumping forward, arms hanging limp, eyes closed",
        "toppling over, body nearly horizontal in the air, one arm reaching out",
        "crumpled on the ground on his side, propped weakly on one elbow, head drooping, wearing his brown leather boots",
        "lying still, flat on his back on the ground, eyes closed, fully at rest",
    ],
}

SPRITE_VIEWS = {
    "side": "Strict side view, character facing right",
    "front": "Front view, character facing the viewer",
    "topdown": "Top-down view from directly above",
    "34": "Three-quarter view",
}


def _sprite_poses(action: str, custom: str, n: int) -> list:
    """Pose script for one action. Known actions use their hand-written frames
    (resampled if the user overrides the count); custom actions get a generic
    anticipation -> peak -> follow-through split across n frames."""
    base = SPRITE_ACTIONS.get(action)
    if base is None:
        what = (custom or action or "the action").strip()
        def phase(i):
            t = i / max(n - 1, 1)
            if t < 0.34:
                return "the anticipation and wind-up of the motion"
            if t < 0.67:
                return "the peak of the motion"
            return "the follow-through and recovery of the motion"
        return [f"performing {what} — frame showing {phase(i)}" for i in range(n)]
    if not n or n == len(base):
        return base
    return [base[round(i * (len(base) - 1) / max(n - 1, 1))] for i in range(n)]


def _sprite_prompt(desc: str, action: str, pose: str, i: int, n: int, view: str) -> str:
    who = (desc or "").strip() or "the exact character from the reference image"
    v = SPRITE_VIEWS.get(view, SPRITE_VIEWS["side"])
    # The view is stated twice — once up front and once right after the pose. With a
    # single mention, pose lines that don't pin the body's orientation let the model
    # snap back to the reference image's orientation (front-facing ref -> front frame).
    return (f"2D game sprite of {who}. {v}. "
            f"Animation: {action}, frame {i + 1} of {n} — {pose}. {v}, mid-motion. "
            "Exactly the same character, art style, proportions, outfit and color palette "
            "as the reference image. Full body, whole character fully inside the frame, "
            "centered, plain solid white background, no text, no watermark, no border.")


def sprite_zip_bytes(name: str) -> bytes:
    """Zip one finished sprite project (in memory) for the ⬇ Download set button."""
    import io
    import zipfile
    base = os.path.normpath(os.path.join(SPRITES, _safe_name(name)))
    if not (base.startswith(os.path.normpath(SPRITES) + os.sep) and os.path.isdir(base)):
        raise RuntimeError("sprite set not found")
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, "w", zipfile.ZIP_DEFLATED) as z:
        for root, dirs, fns in os.walk(base):
            dirs[:] = [d for d in dirs if not d.startswith("_")]   # skip _backups etc.
            for fn in fns:
                p = os.path.join(root, fn)
                z.write(p, os.path.relpath(p, os.path.dirname(base)))
    return buf.getvalue()


class SpriteJob:
    """Async sprite-set job: one reference-edit generation per frame, then per-action
    cutout/resize/strip-sheet and one combined sheet via spritekit.py. Mirrors
    AudiobookJob's lifecycle, plus a cancel flag checked between frames (a full set
    is ~35 generations and can run 20+ minutes)."""

    def __init__(self):
        self.lock = threading.Lock()
        self._reset()

    def _reset(self):
        self.state = "idle"; self.step = 0; self.total = 0
        self.message = ""; self.current = ""; self.name = None
        self.files = []; self.stop_flag = False

    def status(self) -> dict:
        return {"state": self.state, "step": self.step, "total": self.total,
                "message": self.message, "current": self.current,
                "name": self.name, "files": self.files}

    def cancel(self) -> dict:
        if self.state == "running":
            self.stop_flag = True
            self.current = "cancelling after this frame…"
        return self.status()

    def start(self, d: dict) -> dict:
        with self.lock:
            if self.state == "running":
                raise RuntimeError("a sprite set is already rendering — wait for it or cancel it")
            if not (MGR.key or "").startswith("image:"):
                raise RuntimeError("load the image model first (top of this tab)")
            ref = d.get("image") or ""
            if not ref:
                raise RuntimeError("upload a reference image — it defines the character and style")
            if d.get("mode") == "set":
                picked = set(d.get("actions") or [])
                plan = [(a, p) for a, p in SPRITE_ACTIONS.items() if a in picked]
                if not plan:
                    raise RuntimeError("pick at least one action for the set")
            else:
                sel = (d.get("action") or "idle").strip()
                custom = (d.get("custom") or "").strip()
                if sel == "custom" and not custom:
                    raise RuntimeError("describe the custom action")
                act = (_safe_name(custom).lower().replace(" ", "-") if sel == "custom" else sel)
                default_n = len(SPRITE_ACTIONS.get(sel, [])) or 4
                n = max(2, min(12, int(d.get("frames") or 0) or default_n))
                plan = [(act, _sprite_poses(sel, custom, n))]
            raw = (d.get("name") or "").strip()
            base = _safe_name(raw) if raw else "sprites_" + time.strftime("%Y%m%d_%H%M%S")
            name, k = base, 2
            while os.path.exists(os.path.join(SPRITES, name)):
                name, k = f"{base}_{k}", k + 1
            refpath = os.path.join(TMP, "sprite_ref.png")
            with open(refpath, "wb") as f:
                f.write(base64.b64decode(ref.split(",")[-1]))
            self._reset()
            self.state = "running"; self.name = name
            self.total = sum(len(p) for _, p in plan)
            self._plan = plan; self._params = dict(d); self._ref = refpath
        threading.Thread(target=self._run, daemon=True).start()
        return self.status()

    def _run(self):
        try:
            with MGR.lock:
                if not (MGR.key or "").startswith("image:"):
                    raise RuntimeError("the image model was unloaded")
                model = MGR.key.split(":", 1)[1]
                d = self._params
                view = d.get("view") or "side"
                desc = (d.get("desc") or "").strip()
                steps = max(4, min(50, int(d.get("steps") or 24)))
                cfg = float(d.get("cfg") or 2.5)
                seed = int(d.get("seed", -1))
                size = int(d.get("size") or 256)
                if size not in (128, 256, 512, 1024):
                    size = 256
                cutout = bool(d.get("cutout", True))
                if cutout:
                    rc, _o, _e = _run([COMFY_PY, "-c", "import rembg"], timeout=120)
                    if rc != 0:   # degrade instead of failing a long job over transparency
                        cutout = False
                        self.message = ("transparency skipped — install with: "
                                        "ComfyUI\\.venv\\Scripts\\pip install rembg onnxruntime")
                proj = os.path.join(SPRITES, self.name)
                os.makedirs(proj, exist_ok=True)
                shutil.copyfile(self._ref, os.path.join(proj, "reference.png"))
                # Persist the generation settings so single frames can be re-rolled
                # later (new session, new server) without re-entering everything.
                with open(os.path.join(proj, "job.json"), "w", encoding="utf-8") as f:
                    json.dump({"desc": desc, "view": view, "size": size, "cutout": cutout,
                               "steps": steps, "cfg": cfg, "model": model,
                               "order": [a for a, _ in self._plan],
                               "poses": {a: p for a, p in self._plan}}, f, indent=2)

                def rel(p):
                    return os.path.relpath(p, SPRITES).replace("\\", "/")

                done = 0
                for act, poses in self._plan:
                    adir = os.path.join(proj, act)
                    os.makedirs(adir, exist_ok=True)
                    for i, pose in enumerate(poses):
                        if self.stop_flag:
                            raise RuntimeError("cancelled")
                        self.current = f"{act} — frame {i + 1}/{len(poses)}"
                        out = os.path.join(adir, f"frame_{i + 1:02d}.png")
                        cmd = [sys.executable, GEN, "--prompt",
                               _sprite_prompt(desc, act, pose, i, len(poses), view),
                               "--image", self._ref, "--out", out,
                               "--width", "1024", "--height", "1024",
                               "--steps", str(steps), "--cfg", str(cfg), "--model", model]
                        if seed >= 0:
                            cmd += ["--seed", str(seed + done)]
                        rc, sout, err = _run(cmd, timeout=600)
                        if rc != 0 or not os.path.isfile(out):
                            raise RuntimeError(err or sout or f"frame failed ({act} {i + 1})")
                        done += 1; self.step = done
                        self.files = self.files + [rel(out)]   # raw frame appears in the UI now
                    # post-process the finished action while the next one renders: the
                    # frames are rewritten in place (cutout + resize) + a strip sheet.
                    self.current = f"{act} — cutout & sheet"
                    sheet = os.path.join(proj, f"{act}_sheet.png")
                    kcmd = [COMFY_PY, SPRITEKIT, "action", "--dir", adir,
                            "--size", str(size), "--sheet", sheet] + (["--cutout"] if cutout else [])
                    rc, sout, err = _run(kcmd, timeout=900)
                    if rc == 3 and cutout:   # rembg broke mid-job — degrade and redo opaque
                        cutout = False
                        self.message = "transparency skipped — rembg failed (see server log)"
                        rc, sout, err = _run([c for c in kcmd if c != "--cutout"], timeout=900)
                    if rc != 0:
                        raise RuntimeError(err or sout or f"post-processing failed ({act})")
                    self.files = self.files + [rel(sheet)]     # sheet arriving = frames processed
                combined = os.path.join(proj, "spritesheet.png")
                rc, sout, err = _run([COMFY_PY, SPRITEKIT, "combine", "--project", proj,
                                      "--size", str(size),
                                      "--order", ",".join(a for a, _ in self._plan),
                                      "--out", combined], timeout=600)
                if rc != 0:
                    raise RuntimeError(err or sout or "combined sheet failed")
                self.files = self.files + [rel(combined),
                                           rel(os.path.splitext(combined)[0] + ".json")]
            self.state = "done"
            self.message = ((self.message + " · ") if self.message else "") + \
                f"sprite set complete — {done} frames"
        except Exception as e:
            self.state = "cancelled" if str(e) == "cancelled" else "error"
            self.message = str(e)
        finally:
            try:
                os.remove(self._ref)
            except Exception:
                pass


    def reroll(self, d: dict) -> dict:
        """Regenerate ONE frame of a finished set (new seed, optionally a custom pose
        and higher cfg), re-cutout/resize just that frame, and rebuild the action's
        strip + the combined sheet. Runs as a 1-frame job through the same lifecycle."""
        with self.lock:
            if self.state == "running":
                raise RuntimeError("a sprite job is already running — wait for it to finish")
            if not (MGR.key or "").startswith("image:"):
                raise RuntimeError("load the image model first (top of this tab)")
            name = _safe_name(d.get("name") or "")
            act = re.sub(r"[^A-Za-z0-9 _-]", "", d.get("action") or "")
            idx = int(d.get("frame") or 0)
            proj = os.path.join(SPRITES, name)
            adir = os.path.join(proj, act)
            n = len([f for f in os.listdir(adir) if f.startswith("frame_")]) \
                if os.path.isdir(adir) else 0
            if not (os.path.isfile(os.path.join(proj, "reference.png")) and 1 <= idx <= n):
                raise RuntimeError("frame not found")
            job = {}
            try:
                with open(os.path.join(proj, "job.json"), encoding="utf-8") as f:
                    job = json.load(f)
            except Exception:
                pass
            job.update({k: v for k, v in d.items() if v not in (None, "")})
            self._reset()
            self.state = "running"; self.name = name; self.total = 1
            self.current = f"{act} — re-rolling frame {idx}/{n}"
            self._rr = (proj, act, idx, n, job)
        threading.Thread(target=self._run_reroll, daemon=True).start()
        return self.status()

    def _run_reroll(self):
        try:
            with MGR.lock:
                if not (MGR.key or "").startswith("image:"):
                    raise RuntimeError("the image model was unloaded")
                proj, act, idx, n, job = self._rr
                model = MGR.key.split(":", 1)[1]
                pose = (job.get("pose") or "").strip()
                if not pose:
                    saved = (job.get("poses") or {}).get(act) or []
                    pose = saved[idx - 1] if len(saved) >= idx else _sprite_poses(act, act, n)[idx - 1]
                view = job.get("view") or "side"
                size = int(job.get("size") or 512)
                # Default one notch stronger than the original pass: re-rolls exist
                # because the pose didn't land, and more guidance is the usual cure.
                cfg = float(job.get("cfg_override") or (float(job.get("cfg") or 2.5) + 1.0))
                steps = max(4, min(50, int(job.get("steps") or 24)))
                seed = int(job.get("seed", -1))
                out = os.path.join(proj, act, f"frame_{idx:02d}.png")
                # A re-roll can come out WORSE than what it replaces — keep the old
                # frame in _backups (excluded from the zip) so nothing is ever lost.
                if os.path.isfile(out):
                    bdir = os.path.join(proj, "_backups")
                    os.makedirs(bdir, exist_ok=True)
                    shutil.copyfile(out, os.path.join(
                        bdir, f"{act}_frame_{idx:02d}_{int(time.time())}.png"))
                cmd = [sys.executable, GEN, "--prompt",
                       _sprite_prompt(job.get("desc") or "", act, pose, idx - 1, n, view),
                       "--image", os.path.join(proj, "reference.png"), "--out", out,
                       "--width", "1024", "--height", "1024",
                       "--steps", str(steps), "--cfg", str(cfg), "--model", model]
                if seed >= 0:
                    cmd += ["--seed", str(seed)]
                rc, sout, err = _run(cmd, timeout=600)
                if rc != 0 or not os.path.isfile(out):
                    raise RuntimeError(err or sout or "re-roll generation failed")
                self.step = 1

                def rel(p):
                    return os.path.relpath(p, SPRITES).replace("\\", "/")

                self.current = f"{act} — cutout & sheets"
                kcmd = [COMFY_PY, SPRITEKIT, "frame", "--file", out, "--size", str(size)]
                if job.get("cutout", True):
                    kcmd += ["--cutout"]
                rc, sout, err = _run(kcmd, timeout=600)
                if rc == 3:
                    rc, sout, err = _run([c for c in kcmd if c != "--cutout"], timeout=600)
                if rc != 0:
                    raise RuntimeError(err or sout or "re-roll post-processing failed")
                sheet = os.path.join(proj, f"{act}_sheet.png")
                rc, sout, err = _run([COMFY_PY, SPRITEKIT, "strip", "--dir",
                                      os.path.join(proj, act), "--size", str(size),
                                      "--sheet", sheet], timeout=600)
                if rc != 0:
                    raise RuntimeError(err or sout or "strip rebuild failed")
                combined = os.path.join(proj, "spritesheet.png")
                order = ",".join(job.get("order") or [])
                rc, sout, err = _run([COMFY_PY, SPRITEKIT, "combine", "--project", proj,
                                      "--size", str(size), "--order", order,
                                      "--out", combined], timeout=600)
                if rc != 0:
                    raise RuntimeError(err or sout or "combined sheet rebuild failed")
                self.files = [rel(out), rel(sheet), rel(combined)]
            self.state, self.message = "done", f"re-rolled {act} frame {idx}"
        except Exception as e:
            self.state, self.message = "error", str(e)


SPRITEJOB = SpriteJob()


# ---- Composer ----------------------------------------------------------------
# "Figure out which instruments the song needs, find free plugins for them under
# 800 MB, then compose/mix/automate it in a DAW" — done entirely on this machine:
# the plugin hunt becomes a scan of the bundled GM SoundFont (FluidR3 at 141 MB),
# the DAW becomes composerkit.py's render/mix engine, and the LLM does the part a
# producer would.

_COMPOSER_LIB = {"t": 0.0, "data": None}


def composer_library() -> dict:
    """The instrument catalog + soundfont budget scan, cached briefly (it stats
    a 1.2 GB file, and the Composer tab asks for it on every page load)."""
    if _COMPOSER_LIB["data"] and time.time() - _COMPOSER_LIB["t"] < 120:
        return _COMPOSER_LIB["data"]
    rc, out, err = _run([LULLABY_PY, COMPOSERKIT, "library"], timeout=120)
    if rc != 0 or not out.startswith("{"):
        raise RuntimeError(err or out or "could not read the instrument library")
    data = json.loads(out)
    data["genres"] = COMPOSER_GENRES
    _COMPOSER_LIB.update(t=time.time(), data=data)
    return data


COMPOSER_GENRES = ["synth-pop", "cinematic", "lo-fi", "rock", "edm", "jazz",
                   "hip hop", "ambient"]

COMPOSER_SYSTEM = (
    "You are an award-winning composer, arranger and mix engineer. You work in a "
    "fixed local studio: one sampler (FluidSynth) loaded with the General MIDI "
    "instrument set, and a mixing engine that can do saturation, tone shelves, "
    "filter sweeps, tempo-synced delay, convolution reverb, volume/pan/FX "
    "automation, risers, impacts, downlifters, drops and kick sidechaining.\n"
    "You do NOT write note data. You decide the musical direction and the mix, "
    "and the studio's arranger writes the parts from your plan.\n"
    "Answer with a single JSON object and nothing else — no prose, no markdown "
    "fence, no commentary."
)


def _composer_menu_text(lib: dict) -> str:
    lines = []
    for group, items in lib["instruments"].items():
        picks = ", ".join(f"{i['program']}={i['name']}" for i in items)
        lines.append(f"  {group}: {picks}")
    return "\n".join(lines)


def _composer_prompt(brief: str, genre: str, seconds: int, lib: dict,
                     tracks: int, key: str, bpm: int) -> str:
    bars_hint = max(24, int(round(seconds / 2.0)))     # ~2s a bar at pop tempi
    fixed = ""
    if key:
        fixed += f"\nKey: {key} (use this)"
    else:
        # Without a band to aim at, the planner copies the example skeleton's key
        # and tempo every single time — every song came back A minor at 110 bpm.
        hint = (lib.get("genre_hints") or {}).get(genre.lower(), {})
        home = hint.get("bpm")
        fixed += ("\nKey: choose one that suits the brief — commit to a real "
                  "choice, major or minor, and do not reuse the example's")
        if home:
            fixed += (f"\nTempo: pick something in the {int(home * 0.85)}-"
                      f"{int(home * 1.15)} bpm range for {genre}, chosen for THIS "
                      f"brief — not the example's number")
    if bpm:
        fixed += f"\nTempo: {bpm} bpm (use this)"
    roles_text = "\n".join(f"  {r}: {d}" for r, d in lib["roles"].items())
    return f"""Compose an original instrumental song.

BRIEF: {brief or f'an outstanding {genre} instrumental'}
Style/genre: {genre}
Target length: about {seconds // 60}:{seconds % 60:02d} \
(roughly {bars_hint} bars in total across all sections)
Instrument count: {tracks} tracks{fixed}

STEP 1 — decide which instruments this song needs, and pick each one's patch
from the installed library (program number = GM patch, all free and already
local):
{_composer_menu_text(lib)}

STEP 2 — give every track one ROLE the arranger can write. These are the only
roles that exist:
{roles_text}

STEP 3 — mix and automate it. Per track: level 0-1.5, pan -1 (hard left) to +1
(hard right), reverb 0-1, delay 0-1, drive 0-1, tone
(neutral/warm/bright/dark/thin), and up to 3 automation moves:
  {{"param":"cutoff|volume|pan|reverb|delay","mode":"ramp","from":0.2,"to":1.0,"section":"build"}}
  {{"param":"pan","mode":"autopan","depth":0.5,"bars":4}}
A "ramp" with a section name sweeps only inside that section; without one it
sweeps across the whole song.

STEP 4 — structure it. Each section: bars (2-32), energy 0-1 (drives velocity,
kit density and mix loudness), the chord progression as symbols (Am, F, Cmaj7,
G7, Bb/D — the progression loops to fill the section), which tracks play, and
section FX from: riser (builds out of this section into the next), impact (hits
this section's downbeat), downlifter (falls out of this section), drop (a beat
of silence before the downbeat), filter_sweep (opens a lowpass across it).
Leave tracks OUT of quiet sections — an arrangement that drops instruments and
brings them back is what makes a song build.

Reply with exactly this JSON shape (the numbers below are FORMAT examples — pick
your own values for every one of them):
{{"title":"...","genre":"{genre}","bpm":0,"key":"...","time_signature":4,
 "sidechain":true,"swing":0.0,
 "notes":"2-4 sentences: why these instruments, and how the mix/automation works",
 "instruments":[{{"track":"pad","role":"pad","program":89,"level":0.5,"pan":-0.3,
   "reverb":0.7,"delay":0.1,"drive":0.0,"tone":"warm",
   "automation":[{{"param":"cutoff","mode":"ramp","from":0.2,"to":1.0}}]}}],
 "sections":[{{"name":"intro","bars":8,"energy":0.3,"chords":["Am","F","C","G"],
   "tracks":["pad"],"fx":["riser"]}}]}}

Rules: time_signature is 3 or 4. bpm 50-200. {tracks} instruments, each with a
distinct track name, including exactly one "drums" track unless the style has no
kit. 5-9 sections. Every name in a section's "tracks" must match a track name.
JSON only."""


def _extract_json(text: str) -> dict:
    """Pull the plan out of whatever the model actually said. Local models fence
    their JSON, prepend "Here's the song:", or emit reasoning first — so take the
    outermost balanced {...} rather than trusting the whole response."""
    s = (text or "").strip()
    if s.startswith("```"):
        s = re.sub(r"^```[a-zA-Z]*\s*", "", s)
        s = re.sub(r"\s*```\s*$", "", s)
    start = s.find("{")
    if start < 0:
        raise RuntimeError("the planner returned no JSON")
    depth, in_str, esc = 0, False, False
    for i in range(start, len(s)):
        c = s[i]
        if in_str:
            if esc:
                esc = False
            elif c == "\\":
                esc = True
            elif c == '"':
                in_str = False
            continue
        if c == '"':
            in_str = True
        elif c == "{":
            depth += 1
        elif c == "}":
            depth -= 1
            if depth == 0:
                return _loads_lenient(s[start:i + 1])
    raise RuntimeError("the planner's JSON was truncated (try fewer tracks)")


def _loads_lenient(blob: str) -> dict:
    """json.loads, then the same blob with the slips a local model actually makes
    patched out. Each repair is tried cumulatively, cheapest first."""
    attempts = [blob]
    # trailing commas before a close — by far the most common
    attempts.append(re.sub(r",\s*([}\]])", r"\1", attempts[-1]))
    # a missing comma between adjacent members ("}{", "] \"key\"", "0.5 \"key\"")
    attempts.append(re.sub(r"([}\]0-9truefalse\"])\s*\n\s*([{\"])", r"\1,\2",
                           attempts[-1]))
    # python literals leaking into the JSON
    attempts.append(re.sub(r"\b(True|False|None)\b",
                           lambda m: {"True": "true", "False": "false",
                                      "None": "null"}[m.group(1)], attempts[-1]))
    last = None
    for cand in attempts:
        try:
            out = json.loads(cand)
            if isinstance(out, dict):
                return out
        except json.JSONDecodeError as e:
            last = e
    raise RuntimeError(f"the planner's JSON wouldn't parse ({last})")


def composer_list() -> list:
    out = []
    for name in sorted(os.listdir(COMPOSITIONS)):
        d = os.path.join(COMPOSITIONS, name)
        if not os.path.isdir(d):
            continue
        item = {"name": name, "files": []}
        try:
            with open(os.path.join(d, "score.json"), encoding="utf-8") as f:
                sc = json.load(f)
            item.update(title=sc.get("title"), genre=sc.get("genre"),
                        key=sc.get("key"), bpm=sc.get("bpm"),
                        duration=sc.get("duration"))
        except Exception:
            pass
        for fn in sorted(os.listdir(d)):
            if fn.lower().endswith((".mp3", ".wav", ".mid", ".flac")):
                item["files"].append(fn)
        out.append(item)
    return out


def composer_info(name: str) -> dict:
    """Everything the tab shows about one finished composition: the plan it was
    rendered from, the notes, and the stem list."""
    name = _safe_name(name)
    d = os.path.normpath(os.path.join(COMPOSITIONS, name))
    if not d.startswith(os.path.normpath(COMPOSITIONS) + os.sep) or not os.path.isdir(d):
        raise RuntimeError("composition not found")
    with open(os.path.join(d, "score.json"), encoding="utf-8") as f:
        score = json.load(f)
    stems = []
    sdir = os.path.join(d, "stems")
    if os.path.isdir(sdir):
        stems = [f"{name}/stems/{fn}" for fn in sorted(os.listdir(sdir))
                 if fn.lower().endswith((".flac", ".wav"))]
    md = ""
    mdp = os.path.join(d, "arrangement.md")
    if os.path.isfile(mdp):
        with open(mdp, encoding="utf-8") as f:
            md = f.read()
    return {"name": name, "score": score, "stems": stems, "arrangement": md}


def composer_zip_bytes(name: str) -> bytes:
    """Zip a finished composition — master, multitrack MIDI, stems and score."""
    import io
    import zipfile
    base = os.path.normpath(os.path.join(COMPOSITIONS, _safe_name(name)))
    if not (base.startswith(os.path.normpath(COMPOSITIONS) + os.sep)
            and os.path.isdir(base)):
        raise RuntimeError("composition not found")
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, "w", zipfile.ZIP_DEFLATED) as z:
        for root, dirs, fns in os.walk(base):
            dirs[:] = [x for x in dirs if not x.startswith("_")]
            for fn in fns:
                p = os.path.join(root, fn)
                z.write(p, os.path.relpath(p, os.path.dirname(base)))
    return buf.getvalue()


def composer_notes(name: str) -> dict:
    """The composed notes of a saved song, for the piano-roll editor."""
    name = _safe_name(name)
    d = os.path.normpath(os.path.join(COMPOSITIONS, name))
    if not d.startswith(os.path.normpath(COMPOSITIONS) + os.sep):
        raise RuntimeError("composition not found")
    spath = os.path.join(d, "score.json")
    if not os.path.isfile(spath):
        raise RuntimeError("composition not found")
    rc, out, err = _run([LULLABY_PY, COMPOSERKIT, "notes", "--score", spath],
                        timeout=300)
    if rc != 0 or not out.startswith("{"):
        raise RuntimeError(err or out or "could not read the notes")
    return json.loads(out)


class ComposerJob:
    """Two-phase job: PLAN (one LLM call, needs the Language model and so the
    Manager lock) then RENDER (composerkit subprocess — CPU only, so the lock is
    released first and loading a model mid-render is allowed).

    Progress during the render comes from the progress.json composerkit writes
    per stage; without it a 30-90s subprocess would be a blank spinner."""

    def __init__(self):
        self.lock = threading.Lock()
        self._proc = None
        self._reset()

    def _reset(self):
        self.state = "idle"; self.step = 0; self.total = 0
        self.message = ""; self.current = ""; self.name = None
        self.files = []; self.plan = None; self.stop_flag = False
        self._proc = None; self._live = None

    def status(self) -> dict:
        step, total, current = self.step, self.total, self.current
        if self.state == "running" and self.name:
            # the subprocess owns the fine-grained progress
            try:
                p = os.path.join(COMPOSITIONS, self.name, "progress.json")
                with open(p, encoding="utf-8") as f:
                    live = json.load(f)
                # +1/+1 keeps the planning phase visible as the first step
                self._live = (int(live.get("step", 0)) + 1,
                              int(live.get("total", 0)) + 1,
                              live.get("current") or current)
            except Exception:
                # a poll that lands mid-write must not fall back to the coarse
                # 1-of-3 counter — that made the bar visibly jump backwards
                pass
            if self._live:
                step, total, current = self._live
        return {"state": self.state, "step": step, "total": total,
                "message": self.message, "current": current, "name": self.name,
                "files": self.files, "plan": self.plan}

    def cancel(self) -> dict:
        if self.state == "running":
            self.stop_flag = True
            self.current = "cancelling…"
            p = self._proc
            if p and p.poll() is None:
                try:
                    subprocess.run(["taskkill", "/T", "/F", "/PID", str(p.pid)],
                                   capture_output=True, timeout=15, **NOWIN)
                except Exception:
                    pass
        return self.status()

    def start(self, d: dict) -> dict:
        with self.lock:
            if self.state == "running":
                raise RuntimeError("a song is already being composed — wait for it "
                                   "or cancel it")
            if not os.path.isfile(LULLABY_PY):
                raise RuntimeError("lullabykit's venv is missing — the Composer "
                                   "renders with it (see setup)")
            raw = (d.get("name") or d.get("title") or "").strip()
            base = _safe_name(raw) if raw else "song_" + time.strftime("%Y%m%d_%H%M%S")
            name, k = base, 2
            while os.path.exists(os.path.join(COMPOSITIONS, name)):
                name, k = f"{base}_{k}", k + 1
            self._reset()
            self.state = "running"; self.name = name
            self.total = 3; self.step = 0
            self.current = "planning the arrangement"
            self._params = dict(d)
        threading.Thread(target=self._run, daemon=True).start()
        return self.status()

    def rerender(self, d: dict) -> dict:
        """Render an edited score straight from the editor — no planner call.

        Overwrites the song's own folder: the arrangement lives in score.json,
        which is written every render, so nothing about the composition is lost
        while iterating. Only the audio is replaced."""
        with self.lock:
            if self.state == "running":
                raise RuntimeError("a song is already rendering — wait for it "
                                   "or cancel it")
            score = d.get("score")
            if not isinstance(score, dict) or not score.get("instruments"):
                raise RuntimeError("no score to render")
            name = _safe_name(d.get("name") or "")
            if not name or not os.path.isdir(os.path.join(COMPOSITIONS, name)):
                raise RuntimeError("unknown composition")
            self._reset()
            self.state = "running"; self.name = name
            self.total = 3; self.step = 1
            self.current = "re-rendering your edits"
            self._params = {"stems": d.get("stems", True),
                            "polish": d.get("polish", False),
                            "polish_denoise": d.get("polish_denoise", 0.35),
                            "_score": score}
        threading.Thread(target=self._run, daemon=True).start()
        return self.status()

    def _plan_with_llm(self, d: dict, lib: dict) -> dict:
        """One LLM call under the Manager lock. Any failure is non-fatal: the
        genre template still produces a complete song, which matters because the
        render is the slow part and shouldn't be lost to a bad JSON day."""
        brief = (d.get("brief") or "").strip()
        genre = (d.get("genre") or "synth-pop").strip()
        seconds = max(30, min(420, int(d.get("seconds") or 150)))
        tracks = max(3, min(9, int(d.get("tracks") or 6)))
        prompt = _composer_prompt(brief, genre, seconds, lib, tracks,
                                  (d.get("key") or "").strip(),
                                  int(d.get("bpm") or 0))
        with MGR.lock:
            model = current_llm_model()
            if not model:
                raise RuntimeError("no Language model is loaded")
            # 0.65, not 0.9: the creative decisions here are WHICH instruments and
            # chords, not the JSON syntax, and high temperature mostly buys
            # malformed structure. On a parse failure retry once, colder — the
            # plan is seconds of compute and the render is minutes, so it is
            # always worth a second try before falling back to a template.
            plan, err = None, None
            for temperature in (0.65, 0.3):
                text = llm_raw(prompt, task="research", system=COMPOSER_SYSTEM,
                               max_tokens=4096, temperature=temperature,
                               top_p=0.95, model=model, timeout=420)
                try:
                    plan = _extract_json(text)
                    break
                except Exception as e:
                    err = e
                    self.current = "planner returned bad JSON — retrying colder"
            if plan is None:
                raise err or RuntimeError("the planner produced no usable plan")
        plan["planner"] = model
        return plan

    def _apply_polish(self, outdir: str, result: dict) -> list:
        """Optional ACE-Step audio2audio pass over the finished master.

        Answers "would ACE-Step help the MIDI-ness?" — it does, by replacing
        sampled timbres with learned ones, but it is makeup rather than surgery:
        it regenerates the whole mix, so transients soften, it can drift off the
        grid, and it cannot be applied per-instrument (the stems stay dry).
        Kept deliberately optional and non-fatal — the clean render always ships,
        and the polished version is offered ALONGSIDE it rather than replacing
        it, so the two can be compared.

        Costs a model swap: ACE-Step is a resident GPU worker, so loading it
        unloads the planner. That is fine here because planning is finished."""
        files = list(self.files)
        if not self._params.get("polish"):
            return files
        wav = os.path.join(outdir, f"{self.name}.wav")
        if not os.path.isfile(wav):
            return files
        self.current = "polishing with ACE-Step"
        sc = result.get("score") or {}
        insts = ", ".join(t["instrument"] for t in sc.get("instruments", [])[:5])
        tags = (f"{sc.get('genre', 'instrumental')}, instrumental, "
                f"{insts}, {sc.get('bpm', 110)} bpm, "
                f"clean acoustic recording, natural instrument timbre, warm analog")
        raw = os.path.join(outdir, "_polish_raw.flac")
        try:
            MGR.load("music", {})
            rc, sout, err = _run(
                [sys.executable, MUSIC, "--out", "-", "--tags", tags,
                 "--model", "turbo", "--audio", wav,
                 "--denoise", str(float(self._params.get("polish_denoise") or 0.35)),
                 "--format", "flac"], timeout=1800)
            line = (sout or "").strip().splitlines()[-1] if (sout or "").strip() else ""
            if rc != 0 or not line.startswith("{"):
                raise RuntimeError(err or "polish failed")
            with open(raw, "wb") as f:
                f.write(base64.b64decode(json.loads(line)["audios"][0].split(",")[-1]))
            out = os.path.join(outdir, f"{self.name}_polished.mp3")
            # ACE renders hot; bring it back to the same loudness as the clean
            # master so an A/B isn't just "the louder one sounds better"
            rc, _o, err = _run([_ffmpeg(), "-y", "-v", "error", "-i", raw, "-af",
                                "loudnorm=I=-14:TP=-1.0:LRA=11",
                                "-ar", "44100", "-c:a", "libmp3lame", "-q:a", "1",
                                out], timeout=900)
            if rc != 0:
                raise RuntimeError("polish master failed: " + err[-200:])
            return [os.path.relpath(out, COMPOSITIONS).replace("\\", "/")] + files
        except Exception as e:
            self.message = ((self.message + " · ") if self.message else "") + \
                f"polish skipped ({e})"
            return files
        finally:
            for p in (raw,):
                try:
                    os.remove(p)
                except Exception:
                    pass
            try:
                MGR.stop()
            except Exception:
                pass

    def _run(self):
        d = self._params
        try:
            # the editor path supplies a finished score; skip planning entirely
            if d.get("_score"):
                self._render(d["_score"], d)
                return
            lib = composer_library()
            seconds = max(30, min(420, int(d.get("seconds") or 150)))
            genre = (d.get("genre") or "synth-pop").strip()

            plan, note = None, ""
            if d.get("use_llm", True) and current_llm_model():
                try:
                    plan = self._plan_with_llm(d, lib)
                except Exception as e:
                    note = f"planner fell back to the {genre} template ({e})"
            elif d.get("use_llm", True):
                note = (f"no Language model loaded — arranged from the {genre} "
                        f"template")
            if plan is None:
                plan = {"genre": genre, "planner": "template"}
            plan.setdefault("genre", genre)
            plan.setdefault("title", (d.get("title") or "").strip() or None)
            if not plan.get("title"):
                plan["title"] = _safe_name(self.name).replace("_", " ").title()
            plan["brief"] = (d.get("brief") or "").strip()
            if d.get("key"):
                plan["key"] = d["key"]
            if int(d.get("bpm") or 0):
                plan["bpm"] = int(d["bpm"])
            if int(d.get("seed", -1)) >= 0:
                plan["seed"] = int(d["seed"])
            # composerkit fits these AFTER any template substitution, so they
            # apply whether the structure came from the planner or a template
            plan["target_seconds"] = seconds
            plan["target_tracks"] = max(3, min(9, int(d.get("tracks") or 6)))
            plan["humanize"] = bool(d.get("humanize", True))
            if note:
                self.message = note

            if self.stop_flag:
                raise RuntimeError("cancelled")
            self.step = 1
            self.current = "writing the parts"
            self._render(plan, d)
        except Exception as e:
            self.state = "cancelled" if str(e) == "cancelled" else "error"
            self.message = str(e)
            self._proc = None
        finally:
            try:
                os.remove(os.path.join(COMPOSITIONS, self.name, "progress.json"))
            except Exception:
                pass

    def _render(self, plan: dict, d: dict):
        """Hand a plan to composerkit and collect the result. Shared by the
        compose-from-a-brief path and the editor's re-render, so both get the
        same progress reporting, cancellation and polish handling."""
        try:
            outdir = os.path.join(COMPOSITIONS, self.name)
            os.makedirs(outdir, exist_ok=True)
            planpath = os.path.join(outdir, "plan.json")
            with open(planpath, "w", encoding="utf-8") as f:
                json.dump(plan, f, indent=2)

            cmd = [LULLABY_PY, COMPOSERKIT, "render", "--score", planpath,
                   "--out", os.path.join(outdir, self.name)]
            if not d.get("stems", True):
                cmd.append("--no-stems")
            env = dict(os.environ, PYTHONUTF8="1", PYTHONIOENCODING="utf-8")
            self._proc = subprocess.Popen(cmd, stdout=subprocess.PIPE,
                                          stderr=subprocess.PIPE, text=True,
                                          encoding="utf-8", errors="replace",
                                          env=env, **NOWIN)
            out, err = self._proc.communicate(timeout=1800)
            rc = self._proc.returncode
            self._proc = None
            if self.stop_flag:
                raise RuntimeError("cancelled")
            line = (out or "").strip().splitlines()[-1] if (out or "").strip() else ""
            if rc != 0 or not line.startswith("{"):
                raise RuntimeError((err or out or "render failed").strip()[-400:])
            result = json.loads(line)

            self.plan = result.get("score")
            self.files = [os.path.relpath(f, COMPOSITIONS).replace("\\", "/")
                          for f in result.get("files", [])]
            # polish appends to the list the UI actually reads, and works in the
            # same COMPOSITIONS-relative form
            self.files = self._apply_polish(outdir, result)
            self.step = self.total
            self.state = "done"
            self.current = ""
            mins = int(result.get("duration", 0) // 60)
            secs = int(result.get("duration", 0) % 60)
            self.message = ((self.message + " · ") if self.message else "") + \
                f"{mins}:{secs:02d} · {len(self.plan.get('instruments', []))} tracks"
        except Exception as e:
            self.state = "cancelled" if str(e) == "cancelled" else "error"
            self.message = str(e)
            self._proc = None
        finally:
            try:
                os.remove(os.path.join(COMPOSITIONS, self.name, "progress.json"))
            except Exception:
                pass


COMPOSERJOB = ComposerJob()


# ---- HTTP -------------------------------------------------------------------
POST_ROUTES = {
    "/api/load": lambda d: MGR.load(d.get("worker", ""), d),
    "/api/stop": lambda d: MGR.stop(),
    "/api/llm": lambda d: {"text": do_llm(d.get("task", "code"), d.get("prompt", ""),
                                          d.get("image"), d.get("unlocked"))},
    "/api/image": lambda d: {"image": do_image(d.get("prompt", ""), d.get("size", "1:1"),
                                               int(d.get("steps", 20)), int(d.get("seed", -1)),
                                               d.get("model", "base4b"))},
    "/api/edit": lambda d: {"image": do_edit(d.get("prompt", ""), d.get("images", []),
                                             d.get("model", "base4b"), d.get("size", ""),
                                             int(d.get("steps", 20)), int(d.get("seed", -1)),
                                             float(d.get("cfg", 0)))},
    "/api/music": lambda d: do_music(d),
    "/api/stt": lambda d: {"text": do_stt(d.get("audio", ""), d.get("filename", "a.wav"))},
    "/api/tts": lambda d: {"audio": do_tts(d.get("text", ""), d.get("voice", ""), _resolve_ref(d),
                                           _tts_controls(d))},
    "/api/voice": lambda d: {"audio": do_voice(d.get("mode", "text"), d.get("text", ""),
                                               d.get("source"), d.get("target"), d.get("method", "tts"))},
    "/api/voicemodel": lambda d: do_voicemodel(d.get("text", ""), d.get("audio"), _tts_controls(d)),
    "/api/voice_train": lambda d: TRAINJOB.start(d.get("name", ""), d.get("language", "en"),
                                                 int(d.get("epochs", 12)), d.get("samples", [])),
    "/api/voice_delete": lambda d: delete_voice(d.get("name", "")),
    # Story Maker
    "/api/story_save": lambda d: story_save(d.get("story", {})),
    "/api/story_delete": lambda d: story_delete(d.get("id", "")),
    "/api/story_generate": lambda d: story_generate(d.get("id", "")),
    "/api/story_regen_scene": lambda d: story_regen_scene(d.get("id", ""), d.get("key", "")),
    "/api/story_export": lambda d: story_export(d.get("id", "")),
    # Audiobook
    "/api/audiobook_start": lambda d: AUDIOBOOKJOB.start(d),
    # Lullaby Studio
    "/api/lullaby_start": lambda d: LULLABYJOB.start(d),
    "/api/lullaby_analyze": lambda d: LULLABYJOB.analyze(d),
    "/api/lullaby_render": lambda d: LULLABYJOB.render(d),
    # Track Splitter
    "/api/split_start": lambda d: SPLITJOB.start(d),
    # Sprite Studio
    "/api/sprite_start": lambda d: SPRITEJOB.start(d),
    "/api/sprite_cancel": lambda d: SPRITEJOB.cancel(),
    "/api/sprite_reroll": lambda d: SPRITEJOB.reroll(d),
    # Composer
    "/api/composer_start": lambda d: COMPOSERJOB.start(d),
    "/api/composer_cancel": lambda d: COMPOSERJOB.cancel(),
    "/api/composer_rerender": lambda d: COMPOSERJOB.rerender(d),
}


class Handler(BaseHTTPRequestHandler):
    def log_message(self, *a):
        pass

    def _send(self, code, body, ctype="application/json", extra: dict = None):
        data = body if isinstance(body, bytes) else body.encode("utf-8")
        self.send_response(code)
        self.send_header("Content-Type", ctype)
        self.send_header("Content-Length", str(len(data)))
        for k, v in (extra or {}).items():
            self.send_header(k, v)
        self.end_headers()
        self.wfile.write(data)

    def _serve_audiobook_file(self):
        from urllib.parse import unquote, urlparse
        rel = unquote(urlparse(self.path).path[len("/audiobooks/"):])
        base = os.path.normpath(AUDIOBOOKS)
        safe = os.path.normpath(os.path.join(base, rel))
        if not (safe == base or safe.startswith(base + os.sep)) or not os.path.isfile(safe):
            self._send(404, json.dumps({"error": "not found"}))
            return
        self.send_response(200)
        self.send_header("Content-Type", "audio/mpeg")
        self.send_header("Content-Length", str(os.path.getsize(safe)))
        self.send_header("Content-Disposition", 'inline; filename="' + os.path.basename(safe) + '"')
        self.end_headers()
        with open(safe, "rb") as f:
            shutil.copyfileobj(f, self.wfile)

    def _serve_range(self, path, ctype, disposition=None):
        """Serve a file with HTTP Range support (206 Partial Content) — without
        this, a browser's <audio> element can't seek ahead of what it has
        already downloaded, so scrubbing a multi-MB preview silently does
        nothing until the whole file finishes loading."""
        size = os.path.getsize(path)
        start, end, status = 0, size - 1, 200
        rng = self.headers.get("Range")
        if rng and rng.startswith("bytes="):
            try:
                a, _, b = rng[len("bytes="):].partition("-")
                if a:
                    start = int(a)
                if b:
                    end = min(int(b), size - 1)
                if 0 <= start <= end < size:
                    status = 206
                else:
                    start, end = 0, size - 1
            except ValueError:
                start, end = 0, size - 1
        length = end - start + 1
        self.send_response(status)
        self.send_header("Content-Type", ctype)
        self.send_header("Accept-Ranges", "bytes")
        self.send_header("Content-Length", str(length))
        if status == 206:
            self.send_header("Content-Range", f"bytes {start}-{end}/{size}")
        if disposition:
            self.send_header("Content-Disposition", disposition)
        self.end_headers()
        with open(path, "rb") as f:
            f.seek(start)
            remaining = length
            while remaining > 0:
                data = f.read(min(65536, remaining))
                if not data:
                    break
                self.wfile.write(data)
                remaining -= len(data)

    def _serve_lullaby_file(self):
        from urllib.parse import unquote, urlparse
        rel = unquote(urlparse(self.path).path[len("/lullabies/"):])
        base = os.path.normpath(LULLABIES)
        safe = os.path.normpath(os.path.join(base, rel))
        if not safe.startswith(base + os.sep) or not os.path.isfile(safe):
            self._send(404, json.dumps({"error": "not found"}))
            return
        ctype = "audio/mpeg" if safe.lower().endswith(".mp3") else "audio/wav"
        self._serve_range(safe, ctype, 'inline; filename="' + os.path.basename(safe) + '"')

    PREVIEW_FILES = {
        # /lullaby_preview/<name>/<key>.mp3 -> file in the song's work dir
        f"stem_{s}.mp3": f"stem_{s}_preview.mp3"
        for s in ("vocals", "guitar", "piano", "other", "bass", "drums")
    }

    def _serve_lullaby_preview(self):
        """Per-stem audition mp3s out of the lullabykit work dir:
        /lullaby_preview/<name>/stem_<key>.mp3"""
        from urllib.parse import unquote, urlparse
        parts = unquote(urlparse(self.path).path).split("/")
        if len(parts) != 4 or parts[3] not in self.PREVIEW_FILES:
            self._send(404, json.dumps({"error": "not found"}))
            return
        name = _safe_name(parts[2])
        safe = os.path.join(LULLABYKIT, "work", f"lullaby_src_{name}",
                            self.PREVIEW_FILES[parts[3]])
        if not os.path.isfile(safe):
            self._send(404, json.dumps({"error": "not found"}))
            return
        self._serve_range(safe, "audio/mpeg")

    COMPOSER_TYPES = {".mp3": "audio/mpeg", ".wav": "audio/wav",
                      ".flac": "audio/flac", ".mid": "audio/midi",
                      ".md": "text/markdown; charset=utf-8",
                      ".json": "application/json"}

    def _serve_composition_file(self):
        from urllib.parse import unquote, urlparse
        rel = unquote(urlparse(self.path).path[len("/compositions/"):])
        base = os.path.normpath(COMPOSITIONS)
        safe = os.path.normpath(os.path.join(base, rel))
        if not safe.startswith(base + os.sep) or not os.path.isfile(safe):
            self._send(404, json.dumps({"error": "not found"}))
            return
        ext = os.path.splitext(safe)[1].lower()
        ctype = self.COMPOSER_TYPES.get(ext)
        if not ctype:                       # never hand out _work scratch files
            self._send(404, json.dumps({"error": "not found"}))
            return
        # audio needs Range so the player can scrub; MIDI is a download
        disp = ("attachment" if ext == ".mid" else "inline") + \
               '; filename="' + os.path.basename(safe) + '"'
        self._serve_range(safe, ctype, disp)

    def _serve_split_file(self):
        from urllib.parse import unquote, urlparse
        rel = unquote(urlparse(self.path).path[len("/splits/"):])
        base = os.path.normpath(SPLITS)
        safe = os.path.normpath(os.path.join(base, rel))
        if not safe.startswith(base + os.sep) or not os.path.isfile(safe):
            self._send(404, json.dumps({"error": "not found"}))
            return
        ctype = "audio/mpeg" if safe.lower().endswith(".mp3") else "audio/wav"
        self._serve_range(safe, ctype, 'inline; filename="' + os.path.basename(safe) + '"')

    def _serve_sprite_file(self):
        from urllib.parse import unquote, urlparse
        rel = unquote(urlparse(self.path).path[len("/sprites/"):])
        base = os.path.normpath(SPRITES)
        safe = os.path.normpath(os.path.join(base, rel))
        if not safe.startswith(base + os.sep) or not os.path.isfile(safe):
            self._send(404, json.dumps({"error": "not found"}))
            return
        ctype = "image/png" if safe.lower().endswith(".png") else "application/json"
        self.send_response(200)
        self.send_header("Content-Type", ctype)
        self.send_header("Content-Length", str(os.path.getsize(safe)))
        self.end_headers()
        with open(safe, "rb") as f:
            shutil.copyfileobj(f, self.wfile)

    def do_GET(self):
        if self.path in ("/", "/index.html"):
            with open(os.path.join(HERE, "index.html"), "rb") as f:
                # no-store: the UI is a single file edited in place during
                # development, and without this the browser happily serves a
                # cached copy — new panels and controls silently don't appear
                self._send(200, f.read(), "text/html; charset=utf-8",
                           extra={"Cache-Control": "no-store, must-revalidate",
                                  "Pragma": "no-cache"})
        elif self.path == "/health":
            self._send(200, json.dumps({"ok": True}))
        elif self.path == "/api/status":
            self._send(200, json.dumps(MGR.status()))
        elif self.path == "/api/gpu":
            self._send(200, json.dumps(gpu_stats()))
        elif self.path == "/api/config":
            self._send(200, json.dumps({"unlocked_model": OLLAMA_UNLOCKED,
                                        "standard_model": "gpt-oss:20b",
                                        "vision_model": "qwen3-vl:8b"}))
        elif self.path == "/api/voices":
            self._send(200, json.dumps({"voices": list_voices()}))
        elif self.path == "/api/voice_train_status":
            self._send(200, json.dumps(TRAINJOB.status()))
        elif self.path == "/api/scripts":
            try:
                with open(SCRIPTS_JSON, encoding="utf-8") as f:
                    self._send(200, f.read())
            except Exception:
                self._send(200, json.dumps([]))
        elif self.path == "/api/story_list":
            self._send(200, json.dumps(story_list()))
        elif self.path == "/api/story_status":
            self._send(200, json.dumps(STORYJOB.status()))
        elif self.path == "/api/audiobook_status":
            self._send(200, json.dumps(AUDIOBOOKJOB.status()))
        elif self.path == "/api/lullaby_status":
            self._send(200, json.dumps(LULLABYJOB.status()))
        elif self.path.startswith("/api/lullaby_info"):
            from urllib.parse import urlparse, parse_qs
            name = (parse_qs(urlparse(self.path).query).get("name", [""]) or [""])[0]
            try:
                self._send(200, json.dumps(lullaby_info(name)))
            except Exception as e:
                self._send(404, json.dumps({"error": str(e)}))
        elif self.path.startswith("/lullaby_preview/"):
            self._serve_lullaby_preview()
        elif self.path.startswith("/lullabies/"):
            self._serve_lullaby_file()
        elif self.path == "/api/split_status":
            self._send(200, json.dumps(SPLITJOB.status()))
        elif self.path == "/api/split_list":
            self._send(200, json.dumps({"splits": split_list()}))
        elif self.path.startswith("/api/split_zip"):
            from urllib.parse import urlparse, parse_qs
            name = (parse_qs(urlparse(self.path).query).get("name", [""]) or [""])[0]
            try:
                data = split_zip_bytes(name)
                self.send_response(200)
                self.send_header("Content-Type", "application/zip")
                self.send_header("Content-Length", str(len(data)))
                self.send_header("Content-Disposition",
                                 'attachment; filename="%s.zip"' % (_safe_name(name) or "tracks"))
                self.end_headers()
                self.wfile.write(data)
            except Exception as e:
                self._send(404, json.dumps({"error": str(e)}))
        elif self.path.startswith("/splits/"):
            self._serve_split_file()
        elif self.path == "/api/refs":
            self._send(200, json.dumps({"refs": list_refs()}))
        elif self.path.startswith("/audiobooks/"):
            self._serve_audiobook_file()
        elif self.path == "/api/composer_status":
            self._send(200, json.dumps(COMPOSERJOB.status()))
        elif self.path == "/api/composer_library":
            try:
                self._send(200, json.dumps(composer_library()))
            except Exception as e:
                self._send(500, json.dumps({"error": str(e)}))
        elif self.path == "/api/composer_list":
            self._send(200, json.dumps({"songs": composer_list()}))
        elif self.path.startswith("/api/composer_info"):
            from urllib.parse import urlparse, parse_qs
            name = (parse_qs(urlparse(self.path).query).get("name", [""]) or [""])[0]
            try:
                self._send(200, json.dumps(composer_info(name)))
            except Exception as e:
                self._send(404, json.dumps({"error": str(e)}))
        elif self.path.startswith("/api/composer_notes"):
            from urllib.parse import urlparse, parse_qs
            name = (parse_qs(urlparse(self.path).query).get("name", [""]) or [""])[0]
            try:
                self._send(200, json.dumps(composer_notes(name)))
            except Exception as e:
                self._send(404, json.dumps({"error": str(e)}))
        elif self.path.startswith("/api/composer_zip"):
            from urllib.parse import urlparse, parse_qs
            name = (parse_qs(urlparse(self.path).query).get("name", [""]) or [""])[0]
            try:
                data = composer_zip_bytes(name)
                self.send_response(200)
                self.send_header("Content-Type", "application/zip")
                self.send_header("Content-Length", str(len(data)))
                self.send_header("Content-Disposition",
                                 'attachment; filename="%s.zip"' % (_safe_name(name) or "song"))
                self.end_headers()
                self.wfile.write(data)
            except Exception as e:
                self._send(404, json.dumps({"error": str(e)}))
        elif self.path.startswith("/compositions/"):
            self._serve_composition_file()
        elif self.path == "/api/sprite_status":
            self._send(200, json.dumps(SPRITEJOB.status()))
        elif self.path.startswith("/api/sprite_zip"):
            from urllib.parse import urlparse, parse_qs
            name = (parse_qs(urlparse(self.path).query).get("name", [""]) or [""])[0]
            try:
                data = sprite_zip_bytes(name)
                self.send_response(200)
                self.send_header("Content-Type", "application/zip")
                self.send_header("Content-Length", str(len(data)))
                self.send_header("Content-Disposition",
                                 'attachment; filename="%s.zip"' % (_safe_name(name) or "sprites"))
                self.end_headers()
                self.wfile.write(data)
            except Exception as e:
                self._send(404, json.dumps({"error": str(e)}))
        elif self.path.startswith("/sprites/"):
            self._serve_sprite_file()
        elif self.path.startswith("/api/story_load"):
            from urllib.parse import urlparse, parse_qs
            sid = (parse_qs(urlparse(self.path).query).get("id", [""]) or [""])[0]
            try:
                self._send(200, json.dumps(story_load(sid)))
            except Exception as e:
                self._send(404, json.dumps({"error": str(e)}))
        else:
            self._send(404, json.dumps({"error": "not found"}))

    def do_POST(self):
        fn = POST_ROUTES.get(self.path)
        if not fn:
            self._send(404, json.dumps({"error": "not found"}))
            return
        try:
            n = int(self.headers.get("Content-Length", 0))
            # errors="replace": browsers always send UTF-8, but a hand-rolled
            # client (curl on a cp1252 console, a script) sending a stray byte
            # should not fail with an opaque codec traceback
            d = json.loads(self.rfile.read(n).decode("utf-8", "replace") or "{}")
            self._send(200, json.dumps(fn(d)))
        except Exception as e:
            self._send(500, json.dumps({"error": str(e)}))


def lan_ip() -> str:
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    try:
        s.connect(("8.8.8.8", 80))  # no packets sent; just resolves the chosen iface
        return s.getsockname()[0]
    except Exception:
        return "127.0.0.1"
    finally:
        s.close()


if __name__ == "__main__":
    # Localhost-only by default — remote access is via Tailscale Serve
    # (https://<machine>.<tailnet>.ts.net -> 127.0.0.1:8800), so no open port.
    # Set STUDIO_HOST=0.0.0.0 only if you deliberately want LAN exposure.
    host = os.environ.get("STUDIO_HOST", "127.0.0.1")
    port = int(sys.argv[1]) if len(sys.argv) > 1 else 8800
    ip = lan_ip()
    print("Local AI Studio")
    print(f"  this machine : http://127.0.0.1:{port}")
    if host != "127.0.0.1":
        print(f"  on your LAN  : http://{ip}:{port}   (open this on phones/other PCs)")
    print(f"  (bound to {host}:{port})")
    ThreadingHTTPServer((host, port), Handler).serve_forever()
```

## File 2 of 27 — `%USERPROFILE%\local-ai-studio\spritekit.py`

```python
#!/usr/bin/env python3
"""spritekit: post-process generated sprite frames for the Sprite Studio tab.

Runs under ComfyUI's venv python (has Pillow; rembg optional for cutout).
Two subcommands, both driven by server.py's SpriteJob:

  action  --dir DIR --size N [--cutout] --sheet OUT.png
      Process one action folder in place: optional rembg background removal
      (true alpha), downscale each frame_*.png to N x N, then write a
      horizontal strip sheet of the processed frames to OUT.png.

  combine --project DIR --size N --order a,b,c --out spritesheet.png
      Build one combined sheet (one row per action, padded to the widest row)
      from the already-processed frames, plus OUT's .json metadata sidecar
      (cell size, row/frame-count per action) for game-engine import.

Exit codes: 0 ok · 1 bad args / IO error · 3 cutout requested but rembg missing.
"""
from __future__ import annotations

import argparse
import glob
import json
import os
import sys


def frames_in(d: str) -> list[str]:
    return sorted(glob.glob(os.path.join(d, "frame_*.png")))


def load_cutout(enabled: bool):
    """Return a cutout(img)->img callable, or None. Missing rembg -> exit 3 so the
    server can degrade gracefully (keep opaque frames) instead of failing the job."""
    if not enabled:
        return None
    try:
        from rembg import remove, new_session
    except ImportError:
        print("rembg not installed (pip install rembg onnxruntime into ComfyUI's venv)",
              file=sys.stderr)
        sys.exit(3)
    session = new_session(os.environ.get("REMBG_MODEL", "u2net"))
    return lambda img: remove(img, session=session, post_process_mask=True)


def cmd_action(a) -> int:
    from PIL import Image
    paths = frames_in(a.dir)
    if not paths:
        print(f"no frame_*.png in {a.dir}", file=sys.stderr)
        return 1
    cutout = load_cutout(a.cutout)
    frames = []
    for p in paths:
        img = Image.open(p).convert("RGBA")
        if cutout:
            img = cutout(img)
        if img.width != a.size or img.height != a.size:
            img = img.resize((a.size, a.size), Image.LANCZOS)
        img.save(p)          # processed in place — the UI serves these paths
        frames.append(img)
    sheet = Image.new("RGBA", (a.size * len(frames), a.size), (0, 0, 0, 0))
    for i, f in enumerate(frames):
        sheet.paste(f, (i * a.size, 0))
    sheet.save(a.sheet)
    print(json.dumps({"frames": len(frames), "sheet": a.sheet}))
    return 0


def cmd_frame(a) -> int:
    """Process ONE frame in place (re-rolls): cutout + resize, nothing else. Keeps
    already-processed neighbours untouched — re-running rembg on frames that
    already have clean alpha erodes their edges."""
    from PIL import Image
    if not os.path.isfile(a.file):
        print(f"not found: {a.file}", file=sys.stderr)
        return 1
    cutout = load_cutout(a.cutout)
    img = Image.open(a.file).convert("RGBA")
    if cutout:
        img = cutout(img)
    if img.size != (a.size, a.size):
        img = img.resize((a.size, a.size), Image.LANCZOS)
    img.save(a.file)
    print(json.dumps({"frame": a.file}))
    return 0


def cmd_strip(a) -> int:
    """Assemble an action's strip sheet from already-processed frames (no cutout,
    no resize) — used after a single-frame re-roll."""
    from PIL import Image
    paths = frames_in(a.dir)
    if not paths:
        print(f"no frame_*.png in {a.dir}", file=sys.stderr)
        return 1
    sheet = Image.new("RGBA", (a.size * len(paths), a.size), (0, 0, 0, 0))
    for i, p in enumerate(paths):
        sheet.paste(Image.open(p).convert("RGBA"), (i * a.size, 0))
    sheet.save(a.sheet)
    print(json.dumps({"frames": len(paths), "sheet": a.sheet}))
    return 0


def cmd_combine(a) -> int:
    from PIL import Image
    order = [x for x in (a.order or "").split(",") if x]
    actions = [x for x in order if frames_in(os.path.join(a.project, x))] or \
              sorted(d for d in os.listdir(a.project)
                     if frames_in(os.path.join(a.project, d)))
    if not actions:
        print(f"no action folders with frames in {a.project}", file=sys.stderr)
        return 1
    rows = [(act, frames_in(os.path.join(a.project, act))) for act in actions]
    cols = max(len(fr) for _, fr in rows)
    sheet = Image.new("RGBA", (a.size * cols, a.size * len(rows)), (0, 0, 0, 0))
    meta = {"cell": [a.size, a.size], "columns": cols, "actions": {}}
    for r, (act, fr) in enumerate(rows):
        for c, p in enumerate(fr):
            sheet.paste(Image.open(p).convert("RGBA"), (c * a.size, r * a.size))
        meta["actions"][act] = {"row": r, "frames": len(fr)}
    sheet.save(a.out)
    with open(os.path.splitext(a.out)[0] + ".json", "w", encoding="utf-8") as f:
        json.dump(meta, f, indent=2)
    print(json.dumps({"actions": len(rows), "columns": cols, "sheet": a.out}))
    return 0


def main() -> int:
    ap = argparse.ArgumentParser(description="Sprite frame post-processing for Sprite Studio.")
    sub = ap.add_subparsers(dest="cmd", required=True)
    p1 = sub.add_parser("action", help="cutout/resize one action folder + strip sheet")
    p1.add_argument("--dir", required=True)
    p1.add_argument("--size", type=int, default=256)
    p1.add_argument("--cutout", action="store_true")
    p1.add_argument("--sheet", required=True)
    p2 = sub.add_parser("combine", help="combined sheet + json metadata for a project")
    p2.add_argument("--project", required=True)
    p2.add_argument("--size", type=int, default=256)
    p2.add_argument("--order", default="")
    p2.add_argument("--out", required=True)
    p3 = sub.add_parser("frame", help="cutout/resize a single frame in place (re-roll)")
    p3.add_argument("--file", required=True)
    p3.add_argument("--size", type=int, default=256)
    p3.add_argument("--cutout", action="store_true")
    p4 = sub.add_parser("strip", help="rebuild an action strip from processed frames")
    p4.add_argument("--dir", required=True)
    p4.add_argument("--size", type=int, default=256)
    p4.add_argument("--sheet", required=True)
    a = ap.parse_args()
    return {"action": cmd_action, "combine": cmd_combine,
            "frame": cmd_frame, "strip": cmd_strip}[a.cmd](a)


if __name__ == "__main__":
    raise SystemExit(main())
```

## File 3 of 27 — `%USERPROFILE%\local-ai-studio\composerkit.py`

```python
#!/usr/bin/env python3
"""composerkit: turn a song PLAN into a finished, mixed instrumental track.

This is the Composer tab's engine — the local-studio answer to "pick the
instruments you need, find free plugins for them, then arrange, mix and
automate the song in a DAW":

  plugins     -> the General MIDI instrument set inside the bundled SoundFont.
                 Free, permissively licensed, already on this disk, and inside
                 the 800 MB budget — see soundfont_library(). 128 melodic
                 programs + drum kits, played by FluidSynth, which is the
                 sampler/"plugin host".
  arrangement -> the LLM decides the MUSICAL DIRECTION (key, tempo, structure,
                 which instrument plays which role, the chord progression per
                 section, the FX and automation moves). This module writes the
                 actual notes from that direction, because note-level output
                 from a 20B local model is not reliable enough to listen to —
                 same division of labour as lullabykit (theory in code).
  mixing      -> every track is rendered to its own stem, then run through its
                 own chain: saturation, tone shelves, a time-varying lowpass,
                 tempo-synced ping-pong delay and convolution reverb (both with
                 AUTOMATED send levels), tremolo, volume and pan automation.
  production  -> risers into a chorus, sub impacts on the downbeat, downlifters
                 out of a section, drop silences, and optional kick sidechain
                 ducking, all synthesized in the audio domain.

Everything is deterministic given the score's seed, so a render can be
reproduced exactly and a plan can be tweaked one field at a time.

Outputs (into --out's directory):
  <name>.mp3 / <name>.wav   the finished master
  <name>.mid                the whole arrangement as multitrack MIDI, with pan
                            and volume CCs — open it in a real DAW and swap in
                            your own plugins
  stems/<track>.flac        per-instrument stems (post-FX, pre-master)
  score.json                the normalized plan that was actually rendered
  arrangement.md            human-readable score notes

Usage:
  python composerkit.py render --score score.json --out compositions/x/x
  python composerkit.py library          # soundfonts + instrument catalog (JSON)
"""
from __future__ import annotations

import argparse
import json
import math
import os
import random
import shutil
import subprocess
import sys
import time
import zlib
from pathlib import Path

try:
    import numpy as np
except ImportError:     # server.py imports this module with the stdlib python
    np = None           # just for the catalogs below; rendering needs the venv

ROOT = Path(__file__).resolve().parent
LULLABYKIT = ROOT / "lullabykit"
FLUIDSYNTH = LULLABYKIT / "bin" / "fluidsynth" / "bin" / "fluidsynth.exe"
SOUNDFONTS = LULLABYKIT / "soundfonts"
SF_GM = SOUNDFONTS / "FluidR3_GM.sf2"

SR = 44100
# The plugin budget from the brief. FluidR3_GM (141 MB) passes comfortably.
SF_BUDGET_MB = 800


# ============================================================ instrument catalog

GM_INSTRUMENTS = [
    "Acoustic Grand Piano", "Bright Acoustic Piano", "Electric Grand Piano",
    "Honky-tonk Piano", "Electric Piano 1", "Electric Piano 2", "Harpsichord",
    "Clavinet", "Celesta", "Glockenspiel", "Music Box", "Vibraphone", "Marimba",
    "Xylophone", "Tubular Bells", "Dulcimer", "Drawbar Organ", "Percussive Organ",
    "Rock Organ", "Church Organ", "Reed Organ", "Accordion", "Harmonica",
    "Tango Accordion", "Acoustic Guitar (nylon)", "Acoustic Guitar (steel)",
    "Electric Guitar (jazz)", "Electric Guitar (clean)", "Electric Guitar (muted)",
    "Overdriven Guitar", "Distortion Guitar", "Guitar Harmonics", "Acoustic Bass",
    "Electric Bass (finger)", "Electric Bass (pick)", "Fretless Bass", "Slap Bass 1",
    "Slap Bass 2", "Synth Bass 1", "Synth Bass 2", "Violin", "Viola", "Cello",
    "Contrabass", "Tremolo Strings", "Pizzicato Strings", "Orchestral Harp",
    "Timpani", "String Ensemble 1", "String Ensemble 2", "Synth Strings 1",
    "Synth Strings 2", "Choir Aahs", "Voice Oohs", "Synth Voice", "Orchestra Hit",
    "Trumpet", "Trombone", "Tuba", "Muted Trumpet", "French Horn", "Brass Section",
    "Synth Brass 1", "Synth Brass 2", "Soprano Sax", "Alto Sax", "Tenor Sax",
    "Baritone Sax", "Oboe", "English Horn", "Bassoon", "Clarinet", "Piccolo",
    "Flute", "Recorder", "Pan Flute", "Blown Bottle", "Shakuhachi", "Whistle",
    "Ocarina", "Lead 1 (square)", "Lead 2 (sawtooth)", "Lead 3 (calliope)",
    "Lead 4 (chiff)", "Lead 5 (charang)", "Lead 6 (voice)", "Lead 7 (fifths)",
    "Lead 8 (bass + lead)", "Pad 1 (new age)", "Pad 2 (warm)", "Pad 3 (polysynth)",
    "Pad 4 (choir)", "Pad 5 (bowed)", "Pad 6 (metallic)", "Pad 7 (halo)",
    "Pad 8 (sweep)", "FX 1 (rain)", "FX 2 (soundtrack)", "FX 3 (crystal)",
    "FX 4 (atmosphere)", "FX 5 (brightness)", "FX 6 (goblins)", "FX 7 (echoes)",
    "FX 8 (sci-fi)", "Sitar", "Banjo", "Shamisen", "Koto", "Kalimba", "Bag pipe",
    "Fiddle", "Shanai", "Tinkle Bell", "Agogo", "Steel Drums", "Woodblock",
    "Taiko Drum", "Melodic Tom", "Synth Drum", "Reverse Cymbal",
    "Guitar Fret Noise", "Breath Noise", "Seashore", "Bird Tweet",
    "Telephone Ring", "Helicopter", "Applause", "Gunshot",
]

# The menu the planner actually chooses from — the full 128 include sound effects
# and novelty patches that wreck a mix. Grouped by the job they do so the model
# reasons about ROLES ("what does this song need?") instead of patch names.
INSTRUMENT_MENU = {
    "keys": [0, 1, 4, 5, 6, 11, 16, 18, 19, 21],
    "guitar": [24, 25, 26, 27, 28, 29, 30],
    "bass": [32, 33, 34, 35, 36, 38, 39],
    "strings": [40, 42, 45, 46, 48, 49, 50, 44],
    "brass": [56, 57, 58, 60, 61, 62, 63],
    "winds": [64, 65, 66, 68, 71, 73, 74, 75],
    "synth_lead": [80, 81, 82, 84, 85, 87],
    "synth_pad": [88, 89, 90, 91, 92, 94, 95],
    "bells_plucked": [8, 9, 10, 12, 13, 14, 46, 108],
    "voices": [52, 53, 54],
    "world": [104, 105, 106, 107, 114, 116],
}

# What this engine knows how to WRITE. The planner assigns one to each track;
# the program number only decides the timbre, the role decides the notes.
ROLES = {
    "drums": "full kit — kick/snare/hats/cymbals/fills (always GM channel 10)",
    "perc": "auxiliary percussion — shaker, tambourine, congas (channel 10)",
    "bass": "bass line following the chord roots",
    "chords": "rhythmic chord comping — stabs, strums, piano rhythm",
    "pad": "long sustained chords underneath everything",
    "arp": "arpeggio running through the chord tones",
    "lead": "the main melody / hook",
    "counter": "counter-melody answering the lead in its gaps",
}
MELODIC_ROLES = ("bass", "chords", "pad", "arp", "lead", "counter")

# role -> (lo, hi, target_center) MIDI register + default mix level
ROLE_REGISTER = {
    "bass": (28, 45, 36), "chords": (50, 76, 62), "pad": (46, 79, 60),
    "arp": (60, 89, 74), "lead": (65, 88, 75), "counter": (57, 79, 67),
}
ROLE_LEVEL = {"drums": 1.0, "perc": 0.32, "bass": 0.92, "chords": 0.55,
              "pad": 0.42, "arp": 0.48, "lead": 0.85, "counter": 0.45}

# GM percussion note numbers
DRUM = {"kick": 36, "kick2": 35, "snare": 38, "snare2": 40, "clap": 39,
        "stick": 37, "hat": 42, "pedalhat": 44, "openhat": 46, "crash": 49,
        "crash2": 57, "splash": 55, "ride": 51, "ridebell": 53, "tom_lo": 41,
        "tom_mid": 45, "tom_hi": 48, "tom_top": 50, "tamb": 54, "cowbell": 56,
        "shaker": 70, "cabasa": 69, "conga_hi": 63, "conga_lo": 64, "claves": 75}


def soundfont_library() -> dict:
    """The General MIDI banks this tab can play, and whether they fit the budget.

    Only GM banks are listed, because a GM bank is the only thing the Composer
    plays — every instrument is a program number in one. Specialist
    single-instrument libraries that happen to sit in the soundfonts folder (the
    Salamander grand, which the Lullaby kit uses) are deliberately not surfaced
    here: listing a font this tab never loads was confusing, and routing piano
    tracks to a 1.2 GB font just to gain one patch was not worth the size or the
    inconsistency with every other instrument."""
    fonts = []
    if SOUNDFONTS.is_dir():
        for hit in sorted(SOUNDFONTS.glob("**/*.sf[23]")):
            # "is this a full General MIDI bank?" — the bundled font is FluidR3,
            # but accept any drop-in GM bank so swapping the library works
            low = hit.name.lower()
            if not (low.startswith("fluidr3") or "_gm" in low
                    or "gm." in low or "general" in low):
                continue
            mb = hit.stat().st_size / (1024 * 1024)
            fonts.append({"name": hit.name,
                          "path": str(hit),
                          "mb": round(mb, 1),
                          "gm": True,
                          "within_budget": mb <= SF_BUDGET_MB,
                          "used_for": "every instrument"})
    return {
        "budget_mb": SF_BUDGET_MB,
        "soundfonts": fonts,
        "usable": [f for f in fonts if f["within_budget"] and f["gm"]],
        "fluidsynth": str(FLUIDSYNTH) if FLUIDSYNTH.is_file() else None,
        "instruments": {group: [{"program": p, "name": GM_INSTRUMENTS[p]}
                                for p in progs]
                        for group, progs in INSTRUMENT_MENU.items()},
        "roles": ROLES,
        # the whole GM map, for the editor's instrument dropdown — the curated
        # menu above is for steering the planner, but a human editing a track
        # should be able to reach any patch
        "gm_all": list(GM_INSTRUMENTS),
        # each style's home tempo/key, so the planner can be given a band to aim
        # at instead of inferring one from an example (which it just copies)
        "genre_hints": {g: {"bpm": t["bpm"], "key": t["key"]}
                        for g, t in GENRE_TEMPLATES.items()},
    }


# ============================================================ music theory

NOTE_PC = {"C": 0, "C#": 1, "DB": 1, "D": 2, "D#": 3, "EB": 3, "E": 4, "FB": 4,
           "F": 5, "F#": 6, "GB": 6, "G": 7, "G#": 8, "AB": 8, "A": 9, "A#": 10,
           "BB": 10, "B": 11, "CB": 11}
PC_NAME = ["C", "C#", "D", "Eb", "E", "F", "F#", "G", "Ab", "A", "Bb", "B"]
MAJOR = [0, 2, 4, 5, 7, 9, 11]
MINOR = [0, 2, 3, 5, 7, 8, 10]

# chord quality -> intervals from the root. Ordered longest-suffix-first when
# matched so "m7b5" doesn't get eaten by "m".
QUALITIES = [
    ("maj13", [0, 4, 7, 11, 14, 21]), ("maj11", [0, 4, 7, 11, 17]),
    ("maj9", [0, 4, 7, 11, 14]), ("maj7", [0, 4, 7, 11]),
    ("m7b5", [0, 3, 6, 10]), ("dim7", [0, 3, 6, 9]), ("m11", [0, 3, 7, 10, 17]),
    ("m9", [0, 3, 7, 10, 14]), ("m7", [0, 3, 7, 10]), ("m6", [0, 3, 7, 9]),
    ("madd9", [0, 3, 7, 14]), ("mmaj7", [0, 3, 7, 11]),
    ("sus2", [0, 2, 7]), ("sus4", [0, 5, 7]), ("7sus4", [0, 5, 7, 10]),
    ("add9", [0, 4, 7, 14]), ("13", [0, 4, 7, 10, 21]), ("11", [0, 4, 7, 10, 17]),
    ("9", [0, 4, 7, 10, 14]), ("7", [0, 4, 7, 10]), ("6", [0, 4, 7, 9]),
    ("dim", [0, 3, 6]), ("aug", [0, 4, 8]), ("5", [0, 7]),
    ("min", [0, 3, 7]), ("maj", [0, 4, 7]), ("m", [0, 3, 7]), ("", [0, 4, 7]),
]


def parse_key(text: str) -> tuple:
    """'F# minor' / 'Bb' / 'a minor' -> (tonic_pc, 'minor'|'major')."""
    s = (text or "C major").strip()
    mode = "minor" if ("min" in s.lower() or s.rstrip().endswith("m")) else "major"
    head = s.split()[0].rstrip("m") if " " in s else s.rstrip("m")
    pc = NOTE_PC.get(head.upper().replace("♯", "#").replace("♭", "B"), 0)
    return pc, mode


def parse_chord(symbol: str, key_pc: int = 0, mode: str = "major") -> dict:
    """Chord symbol -> {'symbol', 'root', 'ivs', 'bass'}.

    Also accepts roman numerals (I, ii, V7, iv) because planners reach for them
    constantly, and returns None for anything unparseable so the caller can
    substitute a diatonic chord instead of rendering nonsense."""
    s = (symbol or "").strip().replace("Δ", "maj7").replace("-", "m")
    if not s:
        return None
    bass = None
    if "/" in s:
        s, _, b = s.partition("/")
        bass = NOTE_PC.get(b.strip().upper(), None)

    roman = _parse_roman(s, key_pc, mode)
    if roman:
        root, tail = roman
    else:
        head = s[:2] if len(s) > 1 and s[1] in "#b♯♭" else s[:1]
        pc = NOTE_PC.get(head.upper().replace("♯", "#").replace("♭", "B"))
        if pc is None:
            return None
        root, tail = pc, s[len(head):]

    tail = tail.strip().lower().replace("min", "m").replace("major", "maj")
    for name, ivs in QUALITIES:
        if tail == name or (name and tail.startswith(name) and
                            tail[len(name):] in ("", "9", "11", "13")):
            return {"symbol": symbol.strip(), "root": root % 12,
                    "ivs": list(ivs), "bass": bass}
    return {"symbol": symbol.strip(), "root": root % 12,
            "ivs": [0, 3, 7] if tail.startswith("m") else [0, 4, 7], "bass": bass}


ROMAN = [("viii", 7), ("vii", 6), ("vi", 5), ("iv", 3), ("v", 4), ("iii", 2),
         ("ii", 1), ("i", 0)]


def _parse_roman(s: str, key_pc: int, mode: str):
    """'bVII', 'iv', 'V7' -> (root_pc, quality_tail) — or None if not a numeral."""
    body, flat = s, 0
    if body[:1] in ("b", "#") and len(body) > 1 and body[1] in "iIvV":
        flat = -1 if body[0] == "b" else 1
        body = body[1:]
    low = body.lower()
    for numeral, degree in ROMAN:
        if low.startswith(numeral):
            scale = MAJOR if mode == "major" else MINOR
            root = (key_pc + scale[degree % 7] + flat) % 12
            tail = body[len(numeral):]
            # lowercase numeral means minor unless the tail already says otherwise
            if body[:len(numeral)].islower() and not tail[:3].lower() in (
                    "dim", "aug", "maj", "sus"):
                tail = "m" + tail if not tail.startswith("m") else tail
            return root, tail
    return None


def diatonic_chords(key_pc: int, mode: str) -> list:
    """A safe progression in the key — the fallback when a plan's chords don't
    parse, and the source of template progressions."""
    scale = MAJOR if mode == "major" else MINOR
    quals = ([0, 3, 7], [0, 3, 7], [0, 4, 7], [0, 4, 7], [0, 4, 7], [0, 3, 7], [0, 3, 6]) \
        if mode == "minor" else \
        ([0, 4, 7], [0, 3, 7], [0, 3, 7], [0, 4, 7], [0, 4, 7], [0, 3, 7], [0, 3, 6])
    out = []
    for d in range(7):
        root = (key_pc + scale[d]) % 12
        ivs = list(quals[d])
        out.append({"symbol": PC_NAME[root] + ("m" if ivs[1] == 3 else ""),
                    "root": root, "ivs": ivs, "bass": None})
    return out


def scale_pitches(key_pc: int, mode: str) -> list:
    return [(key_pc + s) % 12 for s in (MAJOR if mode == "major" else MINOR)]


def detect_key_from_chords(sections: list) -> tuple:
    """Best-fitting key for the progression that was actually written.

    Planners routinely declare one key and then write a progression in another
    ("C minor" over Dm-Gm-A7-Bb). The melody writers snap to the declared key's
    scale, so a mismatch puts the tune a step off its own harmony — audible, and
    the kind of thing that makes generated music sound broken. The chords are the
    ground truth here, so they win. Returns (tonic_pc, mode, score, weight)."""
    weights = {}
    for sec in sections:
        w = float(sec.get("bars") or 8)
        for c in sec["chords"]:
            for iv in c["ivs"]:
                pc = (c["root"] + iv) % 12
                # roots and thirds identify a key far better than upper extensions
                weights[pc] = weights.get(pc, 0.0) + w * (2.0 if iv in (0, 3, 4) else 1.0)
    total = sum(weights.values())
    if not total:
        return 0, "major", 0.0, 0.0
    best = (0, "major", -1.0)
    for tonic in range(12):
        for mode in ("major", "minor"):
            pcs = set(scale_pitches(tonic, mode))
            fit = sum(w for pc, w in weights.items() if pc in pcs) / total
            # a tonic that the progression actually leans on beats one it never plays
            fit += 0.06 * (weights.get(tonic, 0.0) / total)
            if fit > best[2]:
                best = (tonic, mode, fit)
    return best[0], best[1], best[2], total


def key_fit(sections: list, key_pc: int, mode: str) -> float:
    """How well one key explains the written chords (0-1)."""
    weights = {}
    for sec in sections:
        w = float(sec.get("bars") or 8)
        for c in sec["chords"]:
            for iv in c["ivs"]:
                pc = (c["root"] + iv) % 12
                weights[pc] = weights.get(pc, 0.0) + w * (2.0 if iv in (0, 3, 4) else 1.0)
    total = sum(weights.values())
    if not total:
        return 1.0
    pcs = set(scale_pitches(key_pc, mode))
    return sum(w for pc, w in weights.items() if pc in pcs) / total


def degree_pitch(degree: int, key_pc: int, mode: str, base: int) -> int:
    """Scale degree (0 = tonic, may run past an octave or below zero) -> MIDI."""
    scale = MAJOR if mode == "major" else MINOR
    octv, idx = divmod(degree, 7)
    return base + 12 * octv + scale[idx]


def register_base(key_pc: int, lo: int) -> int:
    """Lowest MIDI tonic at or above `lo` — anchors a part in its register."""
    return lo + ((key_pc - lo) % 12)


def chord_pitches(chord: dict, lo: int, hi: int, size: int = 4,
                  prev: list = None) -> list:
    """Voice a chord inside [lo, hi], choosing the inversion that moves least
    from the previous voicing. Without this the comping jumps an octave every
    time the progression crosses C and the part stops sounding played."""
    root = register_base(chord["root"], lo)
    tones = [root + iv for iv in chord["ivs"]][:max(3, size)]
    best, best_cost = tones, 1e9
    for inv in range(len(tones)):
        cand = tones[inv:] + [p + 12 for p in tones[:inv]]
        cand = [p for p in cand if lo <= p <= hi]
        if len(cand) < 2:
            continue
        center = (sum(prev) / len(prev)) if prev else (lo + hi) / 2
        cost = abs(sum(cand) / len(cand) - center)
        if cost < best_cost:
            best, best_cost = cand, cost
    return sorted(p for p in best if lo <= p <= hi)


def nearest_chord_tone(pitch: int, chord: dict) -> int:
    pcs = {(chord["root"] + iv) % 12 for iv in chord["ivs"]}
    for delta in (0, -1, 1, -2, 2, -3, 3, 4, -4):
        if (pitch + delta) % 12 in pcs:
            return pitch + delta
    return pitch


# ============================================================ genre templates

# Each template is a complete, listenable plan — used verbatim when no planner
# model is loaded, and as the source of every default the planner leaves out.
GENRE_TEMPLATES = {
    "synth-pop": {
        "bpm": 118, "key": "A minor", "sidechain": True,
        "instruments": [
            {"track": "drums", "role": "drums", "program": 0, "pan": 0.0},
            {"track": "bass", "role": "bass", "program": 38, "pan": 0.0, "drive": 0.25},
            {"track": "pad", "role": "pad", "program": 89, "pan": -0.3, "reverb": 0.7,
             "automation": [{"param": "cutoff", "from": 0.25, "to": 1.0}]},
            {"track": "arp", "role": "arp", "program": 81, "pan": 0.35, "delay": 0.3,
             "automation": [{"param": "pan", "mode": "autopan", "depth": 0.45, "bars": 4}]},
            {"track": "lead", "role": "lead", "program": 80, "pan": 0.0, "reverb": 0.35,
             "delay": 0.25},
            {"track": "keys", "role": "chords", "program": 4, "pan": 0.22, "reverb": 0.3},
        ],
        "sections": [
            ("intro", 8, 0.28, ["pad", "arp"], ["riser"]),
            ("verse", 16, 0.5, ["drums", "bass", "pad", "arp", "keys"], []),
            ("pre-chorus", 8, 0.66, ["drums", "bass", "pad", "keys"], ["riser"]),
            ("chorus", 16, 0.95, ["drums", "bass", "pad", "arp", "lead", "keys"], ["impact"]),
            ("verse 2", 16, 0.55, ["drums", "bass", "pad", "arp", "keys"], ["downlifter"]),
            ("bridge", 8, 0.42, ["pad", "keys", "lead"], ["riser"]),
            ("chorus 2", 16, 1.0, ["drums", "bass", "pad", "arp", "lead", "keys"], ["drop"]),
            ("outro", 8, 0.35, ["pad", "arp", "lead"], []),
        ],
        "progressions": {"verse": ["Am", "F", "C", "G"], "chorus": ["F", "C", "G", "Am"],
                         "bridge": ["Dm", "Am", "Bb", "F"]},
    },
    "cinematic": {
        "bpm": 84, "key": "D minor", "sidechain": False,
        "instruments": [
            {"track": "strings", "role": "pad", "program": 48, "pan": -0.2, "reverb": 0.85,
             "automation": [{"param": "volume", "from": 0.4, "to": 1.0}]},
            {"track": "cello", "role": "bass", "program": 42, "pan": 0.15, "reverb": 0.6},
            {"track": "piano", "role": "chords", "program": 0, "pan": 0.25, "reverb": 0.55},
            {"track": "horns", "role": "lead", "program": 60, "pan": 0.0, "reverb": 0.6},
            {"track": "harp", "role": "arp", "program": 46, "pan": -0.4, "reverb": 0.5},
            {"track": "timpani", "role": "drums", "program": 0, "pan": 0.0, "reverb": 0.7},
        ],
        "sections": [
            ("intro", 8, 0.2, ["strings", "harp"], []),
            ("build", 12, 0.5, ["strings", "cello", "piano", "harp"], ["riser"]),
            ("theme", 16, 0.85, ["strings", "cello", "piano", "horns", "timpani"], ["impact"]),
            ("breakdown", 8, 0.35, ["piano", "harp", "strings"], []),
            ("climax", 16, 1.0, ["strings", "cello", "piano", "horns", "harp", "timpani"],
             ["impact"]),
            ("outro", 8, 0.25, ["strings", "harp"], ["downlifter"]),
        ],
        "progressions": {"verse": ["Dm", "Bb", "F", "C"], "chorus": ["Bb", "F", "Gm", "Dm"],
                         "bridge": ["Gm", "Dm", "Eb", "Bb"]},
    },
    "lo-fi": {
        "bpm": 82, "key": "F major", "sidechain": False,
        "instruments": [
            {"track": "drums", "role": "drums", "program": 0, "pan": 0.0, "tone": "warm"},
            {"track": "bass", "role": "bass", "program": 33, "pan": 0.0},
            {"track": "keys", "role": "chords", "program": 4, "pan": -0.25, "reverb": 0.45,
             "delay": 0.2},
            {"track": "vibes", "role": "lead", "program": 11, "pan": 0.3, "reverb": 0.5,
             "delay": 0.3},
            {"track": "pad", "role": "pad", "program": 89, "pan": 0.0, "reverb": 0.7},
            {"track": "shaker", "role": "perc", "program": 0, "pan": 0.4},
        ],
        "sections": [
            ("intro", 8, 0.3, ["keys", "pad"], []),
            ("loop A", 16, 0.55, ["drums", "bass", "keys", "pad", "shaker"], []),
            ("loop B", 16, 0.68, ["drums", "bass", "keys", "vibes", "pad", "shaker"], []),
            ("break", 8, 0.35, ["keys", "pad", "vibes"], []),
            ("loop C", 16, 0.7, ["drums", "bass", "keys", "vibes", "pad", "shaker"], []),
            ("outro", 8, 0.3, ["keys", "pad"], []),
        ],
        "progressions": {"verse": ["Fmaj7", "Dm7", "Gm7", "C7"],
                         "chorus": ["Bbmaj7", "Am7", "Dm7", "Gm7"],
                         "bridge": ["Dm7", "Gm7", "C7", "Fmaj7"]},
    },
    "rock": {
        "bpm": 138, "key": "E minor", "sidechain": False,
        "instruments": [
            {"track": "drums", "role": "drums", "program": 0, "pan": 0.0},
            {"track": "bass", "role": "bass", "program": 34, "pan": 0.0, "drive": 0.3},
            {"track": "rhythm", "role": "chords", "program": 29, "pan": -0.45, "drive": 0.45},
            {"track": "rhythm2", "role": "chords", "program": 29, "pan": 0.45, "drive": 0.45},
            {"track": "lead", "role": "lead", "program": 30, "pan": 0.1, "drive": 0.35,
             "delay": 0.2, "reverb": 0.3},
            {"track": "organ", "role": "pad", "program": 18, "pan": 0.0, "reverb": 0.35},
        ],
        "sections": [
            ("intro", 8, 0.55, ["drums", "bass", "rhythm", "rhythm2"], []),
            ("verse", 16, 0.6, ["drums", "bass", "rhythm", "organ"], []),
            ("chorus", 16, 0.95, ["drums", "bass", "rhythm", "rhythm2", "lead", "organ"],
             ["impact"]),
            ("verse 2", 16, 0.62, ["drums", "bass", "rhythm", "organ"], []),
            ("solo", 12, 0.9, ["drums", "bass", "rhythm", "rhythm2", "lead"], []),
            ("chorus 2", 16, 1.0, ["drums", "bass", "rhythm", "rhythm2", "lead", "organ"],
             ["impact"]),
            ("outro", 8, 0.5, ["drums", "bass", "rhythm", "organ"], ["downlifter"]),
        ],
        "progressions": {"verse": ["Em", "C", "G", "D"], "chorus": ["C", "G", "D", "Em"],
                         "bridge": ["Am", "Em", "C", "D"]},
    },
    "edm": {
        "bpm": 128, "key": "F minor", "sidechain": True,
        "instruments": [
            {"track": "drums", "role": "drums", "program": 0, "pan": 0.0},
            {"track": "bass", "role": "bass", "program": 38, "pan": 0.0, "drive": 0.35},
            {"track": "pluck", "role": "arp", "program": 81, "pan": 0.3, "delay": 0.35,
             "automation": [{"param": "pan", "mode": "autopan", "depth": 0.5, "bars": 2}]},
            {"track": "supersaw", "role": "chords", "program": 90, "pan": 0.0, "reverb": 0.5,
             "automation": [{"param": "cutoff", "from": 0.2, "to": 1.0}]},
            {"track": "lead", "role": "lead", "program": 81, "pan": 0.0, "reverb": 0.4,
             "delay": 0.3},
            {"track": "pad", "role": "pad", "program": 91, "pan": -0.35, "reverb": 0.8},
        ],
        "sections": [
            ("intro", 8, 0.3, ["pad", "pluck"], []),
            ("build", 8, 0.6, ["drums", "pluck", "supersaw", "pad"], ["riser"]),
            ("drop", 16, 1.0, ["drums", "bass", "supersaw", "lead", "pluck"], ["impact", "drop"]),
            ("breakdown", 12, 0.4, ["pad", "supersaw", "lead"], ["downlifter"]),
            ("build 2", 8, 0.7, ["drums", "pluck", "supersaw", "pad"], ["riser"]),
            ("drop 2", 16, 1.0, ["drums", "bass", "supersaw", "lead", "pluck"], ["impact", "drop"]),
            ("outro", 8, 0.35, ["pad", "pluck"], []),
        ],
        "progressions": {"verse": ["Fm", "Db", "Ab", "Eb"], "chorus": ["Db", "Ab", "Eb", "Fm"],
                         "bridge": ["Bbm", "Fm", "Db", "Ab"]},
    },
    "jazz": {
        "bpm": 124, "key": "Bb major", "sidechain": False,
        "instruments": [
            {"track": "drums", "role": "drums", "program": 0, "pan": 0.0, "reverb": 0.3},
            {"track": "bass", "role": "bass", "program": 32, "pan": -0.2},
            {"track": "piano", "role": "chords", "program": 0, "pan": 0.25, "reverb": 0.35},
            {"track": "sax", "role": "lead", "program": 66, "pan": 0.0, "reverb": 0.4},
            {"track": "trumpet", "role": "counter", "program": 56, "pan": 0.3, "reverb": 0.4},
            {"track": "vibes", "role": "arp", "program": 11, "pan": -0.35, "reverb": 0.45},
        ],
        "sections": [
            ("head", 16, 0.55, ["drums", "bass", "piano", "sax"], []),
            ("head 2", 16, 0.65, ["drums", "bass", "piano", "sax", "trumpet"], []),
            ("solo", 16, 0.75, ["drums", "bass", "piano", "vibes"], []),
            ("shout", 12, 0.9, ["drums", "bass", "piano", "sax", "trumpet"], ["impact"]),
            ("out head", 16, 0.6, ["drums", "bass", "piano", "sax", "trumpet"], []),
        ],
        "progressions": {"verse": ["Bbmaj7", "Gm7", "Cm7", "F7"],
                         "chorus": ["Ebmaj7", "Dm7", "Gm7", "Cm7"],
                         "bridge": ["Dm7", "G7", "Cm7", "F7"]},
    },
    "hip hop": {
        "bpm": 92, "key": "C minor", "sidechain": False,
        "instruments": [
            {"track": "drums", "role": "drums", "program": 0, "pan": 0.0},
            {"track": "808", "role": "bass", "program": 38, "pan": 0.0, "drive": 0.3},
            {"track": "keys", "role": "chords", "program": 4, "pan": -0.25, "reverb": 0.4},
            {"track": "bells", "role": "lead", "program": 9, "pan": 0.25, "delay": 0.3,
             "reverb": 0.4},
            {"track": "strings", "role": "pad", "program": 49, "pan": 0.0, "reverb": 0.7},
            {"track": "perc", "role": "perc", "program": 0, "pan": 0.35},
        ],
        "sections": [
            ("intro", 8, 0.35, ["keys", "strings"], []),
            ("verse", 16, 0.6, ["drums", "808", "keys", "strings", "perc"], ["impact"]),
            ("hook", 16, 0.9, ["drums", "808", "keys", "bells", "strings", "perc"], []),
            ("verse 2", 16, 0.62, ["drums", "808", "keys", "strings", "perc"], []),
            ("hook 2", 16, 0.95, ["drums", "808", "keys", "bells", "strings", "perc"], []),
            ("outro", 8, 0.3, ["keys", "strings"], ["downlifter"]),
        ],
        "progressions": {"verse": ["Cm", "Ab", "Eb", "Bb"], "chorus": ["Fm", "Cm", "Ab", "Eb"],
                         "bridge": ["Ab", "Eb", "Bb", "Cm"]},
    },
    "ambient": {
        "bpm": 70, "key": "C major", "sidechain": False,
        "instruments": [
            {"track": "pad", "role": "pad", "program": 88, "pan": -0.2, "reverb": 0.9,
             "automation": [{"param": "cutoff", "from": 0.2, "to": 0.9}]},
            {"track": "pad2", "role": "pad", "program": 95, "pan": 0.25, "reverb": 0.9},
            {"track": "bells", "role": "arp", "program": 10, "pan": 0.35, "delay": 0.45,
             "reverb": 0.7,
             "automation": [{"param": "pan", "mode": "autopan", "depth": 0.6, "bars": 8}]},
            {"track": "cello", "role": "bass", "program": 42, "pan": 0.0, "reverb": 0.75},
            {"track": "flute", "role": "lead", "program": 73, "pan": 0.1, "reverb": 0.8,
             "delay": 0.35},
        ],
        "sections": [
            ("emerge", 12, 0.2, ["pad", "cello"], []),
            ("drift", 16, 0.4, ["pad", "pad2", "bells", "cello"], []),
            ("bloom", 16, 0.65, ["pad", "pad2", "bells", "cello", "flute"], ["impact"]),
            ("recede", 12, 0.3, ["pad", "bells", "flute"], ["downlifter"]),
            ("rest", 8, 0.15, ["pad", "cello"], []),
        ],
        "progressions": {"verse": ["Cmaj7", "Am7", "Fmaj7", "G"],
                         "chorus": ["Fmaj7", "Cmaj7", "Dm7", "G"],
                         "bridge": ["Am7", "Em7", "Fmaj7", "Cmaj7"]},
    },
}
DEFAULT_GENRE = "synth-pop"


def template_for(genre: str) -> dict:
    g = (genre or "").strip().lower()
    if g in GENRE_TEMPLATES:
        return GENRE_TEMPLATES[g]
    for name, t in GENRE_TEMPLATES.items():          # "dark cinematic trailer"
        if name in g or g in name:
            return t
    for name, t in GENRE_TEMPLATES.items():
        if any(w and w in g for w in name.split("-")):
            return t
    return GENRE_TEMPLATES[DEFAULT_GENRE]


# ============================================================ score normalization

WORD_LEVELS = {"none": 0.0, "silent": 0.0, "min": 0.1, "low": 0.3, "quiet": 0.3,
               "soft": 0.35, "medium": 0.6, "mid": 0.6, "moderate": 0.6,
               "high": 0.85, "loud": 0.9, "full": 1.0, "max": 1.0, "maximum": 1.0}


def _clamp(v, lo, hi, default):
    """Coerce-and-clamp. Every plan field goes through this because the planner
    happily returns "118 bpm", "75%", "high", null, or "Pad 2 (warm)" where a
    number belongs — and a render must never die on one bad field."""
    if isinstance(v, str):
        s = v.strip().lower()
        if s in WORD_LEVELS:
            v = WORD_LEVELS[s]
        elif s.endswith("%"):
            try:
                v = float(s[:-1]) / 100.0
            except ValueError:
                return default
    try:
        n = float(str(v).strip().split()[0].rstrip("%"))
    except (TypeError, ValueError, IndexError, AttributeError):
        return default
    if isinstance(lo, int) and isinstance(hi, int):
        n = int(round(n))
    return max(lo, min(hi, n))


# A patch that suits each role, for when the plan's program is missing or bogus.
# Clamping a nonsense number instead would hand a pad the Gunshot patch.
ROLE_PROGRAM = {"drums": 0, "perc": 0, "bass": 33, "chords": 0, "pad": 89,
                "arp": 81, "lead": 80, "counter": 73}
GM_BY_NAME = {n.lower(): i for i, n in enumerate(GM_INSTRUMENTS)}
# words in a track name that give away what it's for, when the role is missing
# or unrecognized — far better than positional guessing
NAME_ROLE_HINTS = (
    (("drum", "kit", "beat", "percussion"), "drums"),
    (("shaker", "tamb", "conga", "perc"), "perc"),
    (("bass", "808", "sub"), "bass"),
    (("pad", "string", "atmos", "choir", "swell", "organ"), "pad"),
    (("arp", "pluck", "sequence", "seq"), "arp"),
    (("lead", "melody", "hook", "sax", "trumpet", "flute", "vocal", "solo"), "lead"),
    (("counter", "harmony", "answer"), "counter"),
    (("piano", "keys", "key", "guitar", "rhythm", "chord", "stab", "comp",
      "vibes", "bell", "harp", "synth"), "chords"),
)


def _resolve_program(value, role: str) -> tuple:
    """Program number from anything a planner might send: a number, a GM patch
    name ("Pad 2 (warm)"), or junk. Returns (program, warning)."""
    if role in ("drums", "perc"):
        return 0, None
    if isinstance(value, str) and not value.strip().lstrip("-").isdigit():
        key = value.strip().lower()
        if key in GM_BY_NAME:
            return GM_BY_NAME[key], None
        for name, idx in GM_BY_NAME.items():          # "warm pad" -> "Pad 2 (warm)"
            if key and (key in name or name in key):
                return idx, None
        return ROLE_PROGRAM[role], f"unknown instrument {value!r} -> " \
                                   f"{GM_INSTRUMENTS[ROLE_PROGRAM[role]]}"
    # out of range falls back to the role's patch rather than clamping: clamping
    # 200 would hand a pad the Gunshot sample at the top of the GM map
    n = _clamp(value, -9999, 9999, -1)
    if not 0 <= n <= 127:
        return ROLE_PROGRAM[role], (None if n == -1 else
                                    f"program {n} is outside GM 0-127 -> "
                                    f"{GM_INSTRUMENTS[ROLE_PROGRAM[role]]}")
    return n, None


def _infer_role(item: dict, name: str, index: int) -> tuple:
    """Best-effort role. Reads the declared role first, then the track name, and
    only then falls back to position."""
    raw = str(item.get("role") or "").strip().lower()
    alias = {"keys": "chords", "piano": "chords", "guitar": "chords",
             "rhythm": "chords", "comp": "chords", "stab": "chords",
             "harmony": "counter", "counter-melody": "counter",
             "countermelody": "counter", "melody": "lead", "hook": "lead",
             "solo": "lead", "percussion": "perc", "aux": "perc",
             "kit": "drums", "drum": "drums", "beat": "drums",
             "strings": "pad", "atmosphere": "pad", "texture": "pad"}
    role = alias.get(raw, raw)
    if role in ROLES:
        return role, None
    hay = f"{name} {raw}".lower()
    for words, guess in NAME_ROLE_HINTS:
        if any(w in hay for w in words):
            return guess, (f"track {index + 1} ({name}): role {raw!r} isn't one I "
                           f"can write — treated it as {guess}" if raw else None)
    guess = "lead" if index == 0 else "chords"
    return guess, (f"track {index + 1} ({name}): role {raw!r} isn't one I can "
                   f"write — treated it as {guess}")


def _progression_for(name: str, prog: dict) -> list:
    n = (name or "").lower()
    if any(k in n for k in ("chorus", "hook", "drop", "theme", "climax", "shout")):
        return prog.get("chorus") or prog["verse"]
    if any(k in n for k in ("bridge", "middle", "break", "solo", "breakdown", "recede")):
        return prog.get("bridge") or prog["verse"]
    return prog["verse"]


def normalize_score(raw: dict) -> dict:
    """Repair ANY plan into something renderable.

    The planner is a 20B local model, so every field is treated as a
    suggestion: unknown roles, out-of-range programs, chords that don't parse,
    sections referencing tracks that don't exist, missing instruments — all get
    replaced from the genre template rather than failing the render. A plan that
    arrives empty still produces a complete song."""
    raw = dict(raw or {})
    genre = str(raw.get("genre") or DEFAULT_GENRE).strip() or DEFAULT_GENRE
    tpl = template_for(genre)
    warnings = []
    # Falling back to the template is only worth reporting when a planner TRIED
    # and produced something unusable. When the caller asked for the template
    # outright, saying "no usable instrument list" reads as a failure.
    from_template = str(raw.get("planner") or "").lower() == "template"

    key = str(raw.get("key") or tpl["key"])
    key_pc, mode = parse_key(key)
    bpm = _clamp(raw.get("bpm"), 50, 200, tpl["bpm"])
    tsig = _clamp(raw.get("time_signature"), 3, 4, 4)
    seed = _clamp(raw.get("seed"), -1, 2 ** 31 - 1, -1)
    if seed < 0:
        seed = random.randint(1, 2 ** 31 - 1)

    # ---- instruments ----
    src = raw.get("instruments")
    if isinstance(src, dict):        # accept {"pad": {...}} as well as a list
        src = [{**v, "track": k} for k, v in src.items() if isinstance(v, dict)]
    if not isinstance(src, list) or not src:
        src = tpl["instruments"]
        if not from_template:
            warnings.append("no usable instrument list in the plan — used the "
                            f"{genre} template roster")
    insts, seen = [], set()
    for i, item in enumerate(src[:10]):
        if not isinstance(item, dict):
            continue
        name = str(item.get("track") or item.get("name") or
                   item.get("role") or f"track {i + 1}").strip()[:24] or f"track {i + 1}"
        role, warn = _infer_role(item, name, i)
        if warn:
            warnings.append(warn)
        while name.lower() in seen:
            name += "2"
        seen.add(name.lower())
        prog, warn = _resolve_program(item.get("program"), role)
        if warn:
            warnings.append(f"track {i + 1} ({name}): {warn}")
        # hand-edited notes from the piano roll: [[beat, dur_beats, pitch, vel]].
        # Present = this track replays exactly these instead of being re-composed,
        # so an edit survives every later re-render.
        override = item.get("notes_override")
        clean_override = None
        if isinstance(override, list) and override:
            clean_override = []
            for n in override[:20000]:
                if not isinstance(n, (list, tuple)) or len(n) < 4:
                    continue
                clean_override.append([
                    round(max(0.0, _clamp(n[0], 0.0, 100000.0, 0.0)), 4),
                    round(max(0.02, _clamp(n[1], 0.02, 64.0, 1.0)), 4),
                    int(_clamp(n[2], 0, 127, 60)), int(_clamp(n[3], 1, 127, 90))])
            clean_override = clean_override or None

        t = {
            "track": name, "role": role, "program": prog,
            "notes_override": clean_override,
            "muted": str(item.get("muted", False)).lower() in ("true", "1", "yes"),
            # drums/perc always play the GM percussion map on channel 10, so the
            # program number is meaningless for them — don't label it as a patch
            "instrument": ("GM Drum Kit" if role in ("drums", "perc")
                           else GM_INSTRUMENTS[prog]),
            "level": round(_clamp(item.get("level"), 0.05, 1.5, ROLE_LEVEL[role]), 3),
            "pan": round(_clamp(item.get("pan"), -1.0, 1.0, 0.0), 3),
            "reverb": round(_clamp(item.get("reverb"), 0.0, 1.0, 0.3), 3),
            "delay": round(_clamp(item.get("delay"), 0.0, 1.0, 0.0), 3),
            "drive": round(_clamp(item.get("drive"), 0.0, 1.0, 0.0), 3),
            "tone": str(item.get("tone") or "neutral").strip().lower()[:12],
            "automation": _norm_automation(item.get("automation")),
        }
        insts.append(t)
    if not any(t["role"] == "drums" for t in insts) and tsig == 4 and \
            "ambient" not in genre.lower():
        insts.append({"track": "drums", "role": "drums", "program": 0,
                      "instrument": "GM Drum Kit", "level": 1.0, "pan": 0.0,
                      "reverb": 0.25, "delay": 0.0, "drive": 0.0,
                      "tone": "neutral", "automation": []})
    if not any(t["role"] == "bass" for t in insts):
        insts.append({"track": "bass", "role": "bass", "program": 33,
                      "instrument": GM_INSTRUMENTS[33], "level": ROLE_LEVEL["bass"],
                      "pan": 0.0, "reverb": 0.2, "delay": 0.0, "drive": 0.15,
                      "tone": "neutral", "automation": []})
    want_tracks = _clamp(raw.get("target_tracks"), 1, 10, 0)
    if want_tracks and len(insts) > want_tracks:
        # honour the requested instrument count — the template rosters are a
        # fixed size, so without this the UI's slider does nothing whenever the
        # plan came from a template rather than the planner. Drop the least
        # load-bearing parts first: a song survives losing its counter-melody,
        # not its drums or bass.
        keep_order = ("drums", "bass", "lead", "chords", "pad", "arp", "perc", "counter")
        ranked = sorted(insts, key=lambda t: keep_order.index(t["role"])
                        if t["role"] in keep_order else 99)
        kept = {id(t) for t in ranked[:want_tracks]}
        insts = [t for t in insts if id(t) in kept]
    names = [t["track"] for t in insts]

    # ---- sections ----
    prog_bank = {k: list(v) for k, v in tpl["progressions"].items()}
    src_sec = raw.get("sections")
    if not isinstance(src_sec, list) or not src_sec:
        src_sec = [{"name": n, "bars": b, "energy": e, "tracks": tracks, "fx": fx}
                   for n, b, e, tracks, fx in tpl["sections"]]
        if not from_template:
            warnings.append(f"no usable section list — used the {genre} "
                            f"template structure")
    sections = []
    for item in src_sec[:16]:
        if not isinstance(item, dict):
            continue
        name = str(item.get("name") or f"section {len(sections) + 1}").strip()[:28]
        bars = _clamp(item.get("bars"), 2, 32, 8)
        energy = round(_clamp(item.get("energy"), 0.0, 1.0, 0.6), 3)
        tracks = [n for n in (item.get("tracks") or []) if n in names]
        if not tracks:
            # fall back to the template's own instrumentation for a section of
            # this energy: everything but the melody at low energy, all in at high
            tracks = [t["track"] for t in insts
                      if energy > 0.7 or t["role"] not in ("lead", "counter")]
            if energy < 0.35:
                keep = [t["track"] for t in insts if t["role"] in ("pad", "chords", "arp")]
                tracks = keep or tracks
        chords = []
        for sym in (item.get("chord_symbols") or item.get("chords")
                    or _progression_for(name, prog_bank)):
            # accept a normalized score as input as well as a raw plan: the
            # editor round-trips this module's own output, where chords are dicts
            # rather than symbol strings. Without this a saved score reparsed to
            # "{'symbol': ...}", failed, and silently fell back to a diatonic
            # progression — an edit would quietly rewrite the harmony.
            if isinstance(sym, dict):
                sym = sym.get("symbol") or ""
            c = parse_chord(str(sym), key_pc, mode)
            if c:
                chords.append(c)
        if not chords:
            dia = diatonic_chords(key_pc, mode)
            chords = [dia[0], dia[5], dia[3], dia[4]]
            warnings.append(f"'{name}': no parseable chords — used a diatonic progression")
        fx = [f for f in (item.get("fx") or [])
              if str(f).strip().lower() in ("riser", "impact", "downlifter", "drop",
                                            "filter_sweep")]
        sections.append({"name": name, "bars": bars, "energy": energy,
                         "tracks": tracks, "fx": [str(f).lower() for f in fx],
                         "chords": chords,
                         "chord_symbols": [c["symbol"] for c in chords]})
    if not sections:
        sections = [{"name": "main", "bars": 16, "energy": 0.7,
                     "tracks": names, "fx": [], "chords": diatonic_chords(key_pc, mode)[:4],
                     "chord_symbols": []}]

    # Trust the chords over the declared key when they disagree — see
    # detect_key_from_chords. Only overrides on a clear win, so a progression
    # with a couple of borrowed chords doesn't get its key relabelled.
    declared_fit = key_fit(sections, key_pc, mode)
    det_pc, det_mode, det_fit, _w = detect_key_from_chords(sections)
    if det_fit > declared_fit + 0.12 and (det_pc, det_mode) != (key_pc, mode):
        warnings.append(
            f"the plan said {PC_NAME[key_pc]} {mode} but the progression is in "
            f"{PC_NAME[det_pc]} {det_mode} ({det_fit:.0%} of chord tones fit vs "
            f"{declared_fit:.0%}) — used the chords' key so the melody agrees "
            f"with the harmony")
        key_pc, mode = det_pc, det_mode

    _fit_duration(sections, _clamp(raw.get("target_seconds"), 0, 900, 0),
                  tsig, bpm, warnings)
    total_bars = sum(s["bars"] for s in sections)
    return {
        "title": str(raw.get("title") or "Untitled").strip()[:80] or "Untitled",
        "genre": genre, "brief": str(raw.get("brief") or "")[:600],
        "notes": str(raw.get("notes") or "")[:1200],
        "bpm": bpm, "key": f"{PC_NAME[key_pc]} {mode}", "key_pc": key_pc, "mode": mode,
        "time_signature": tsig, "seed": seed,
        "sidechain": str(raw.get("sidechain", tpl["sidechain"])).lower()
                     not in ("false", "none", "0", "no", ""),
        "swing": round(_clamp(raw.get("swing"), 0.0, 0.6, 0.0), 3),
        # performance realism, on by default; exposed so a render can be A/B'd
        # against the raw quantized version
        "humanize": str(raw.get("humanize", True)).lower()
                    not in ("false", "none", "0", "no"),
        "instruments": insts, "sections": sections,
        "total_bars": total_bars,
        "duration": round(total_bars * tsig * 60.0 / bpm, 1),
        "planner": str(raw.get("planner") or "template")[:60],
        "warnings": warnings,
    }


def _fit_duration(sections: list, target_seconds: float, tsig: int, bpm: int,
                  warnings: list):
    """Scale section lengths in place so the song lands near the requested
    length.

    Runs here rather than on the incoming plan because the sections may have come
    from a genre template (fixed length) instead of the planner — fitting before
    that substitution silently did nothing, and a 1-minute request rendered 3:30.
    Planners are also just bad at this arithmetic, and it's the one part of a plan
    that's pure maths rather than taste."""
    if not target_seconds or not sections:
        return
    bar_seconds = tsig * 60.0 / bpm
    target_bars = max(4.0, target_seconds / bar_seconds)
    current = float(sum(s["bars"] for s in sections))
    f = target_bars / current
    if 0.88 <= f <= 1.12:
        return
    for s in sections:
        # keep even bar counts so phrases stay square
        s["bars"] = int(max(2, min(32, round(s["bars"] * f / 2) * 2)))
    got = sum(s["bars"] for s in sections) * bar_seconds
    if abs(got - target_seconds) > max(20.0, 0.25 * target_seconds):
        warnings.append(
            f"asked for {int(target_seconds // 60)}:{int(target_seconds % 60):02d} "
            f"but the structure fits {int(got // 60)}:{int(got % 60):02d} — "
            f"sections are capped at 32 bars, so add or remove sections to get closer")


AUTO_PARAMS = ("cutoff", "volume", "pan", "reverb", "delay")


def _norm_automation(src) -> list:
    out = []
    if not isinstance(src, list):
        return out
    for item in src[:4]:
        if not isinstance(item, dict):
            continue
        param = str(item.get("param") or "").strip().lower()
        if param not in AUTO_PARAMS:
            continue
        mode = str(item.get("mode") or "ramp").strip().lower()
        if mode not in ("ramp", "autopan", "tremolo"):
            mode = "ramp"
        out.append({
            "param": param, "mode": mode,
            "from": round(_clamp(item.get("from"), -1.0, 2.0, 0.0), 3),
            "to": round(_clamp(item.get("to"), -1.0, 2.0, 1.0), 3),
            "depth": round(_clamp(item.get("depth"), 0.0, 1.0, 0.4), 3),
            "bars": round(_clamp(item.get("bars"), 0.25, 32.0, 4.0), 3),
            "section": str(item.get("section") or "").strip()[:28],
        })
    return out


# ============================================================ arrangement -> notes

# One bar of 4/4 as 16 sixteenths. 'x' = accent, 'o' = normal, '-' = ghost.
DRUM_PATTERNS = {
    "pop": {"kick": "x..o..x...o.....", "snare": "....x.......x...",
            "hat": "..o...o...o...o.", "openhat": "............o...",
            "shaker": "o.o.o.o.o.o.o.o."},
    "rock": {"kick": "x...x..x..x.....", "snare": "....x.......x..o",
             "hat": "o.o.o.o.o.o.o.o.", "ride": "................"},
    "edm": {"kick": "x...x...x...x...", "snare": "....x.......x...",
            "hat": "..x...x...x...x.", "openhat": "..o...o...o...o.",
            "clap": "....x.......x..."},
    "house": {"kick": "x...x...x...x...", "clap": "....x.......x...",
              "hat": "..o.o.o.o.o.o.o.", "openhat": "..x...x...x...x."},
    "hip hop": {"kick": "x.....o.x.......", "snare": "....x.......x...",
                "hat": "o.o.o.o.o.o.o.o-", "openhat": "..........o....."},
    "trap": {"kick": "x.....x...x.....", "snare": "....x.......x...",
             "hat": "oooooooooooooooo", "openhat": "..........o....."},
    "funk": {"kick": "x..x..o.x...o...", "snare": "....x..o....x..o",
             "hat": "o-o-o-o-o-o-o-o-", "openhat": "......o........."},
    "jazz": {"kick": "x.......x.......", "snare": "....-.......-..o",
             "ride": "o..o.oo..o.oo..o", "pedalhat": "....o.......o..."},
    "latin": {"kick": "x.....x.x.....x.", "snare": "..o...o...o...o.",
              "hat": "o.o.o.o.o.o.o.o.", "conga_hi": "..o.o...o.o.o..o",
              "claves": "..x..x..x...x..."},
    "ballad": {"kick": "x.......x.......", "snare": "....x.......x...",
               "hat": "....o.......o..."},
    "lo-fi": {"kick": "x.....o.x.......", "snare": "....x.......x..-",
              "hat": "o..o.o..o..o.o..", "shaker": "..o...o...o...o."},
    "metal": {"kick": "x.x.x.x.x.x.x.x.", "snare": "....x.......x...",
              "hat": "o...o...o...o...", "crash2": "................"},
    "cinematic": {"kick": "x.......x.......", "tom_lo": "x.......x.......",
                  "tom_mid": "........x...x...", "crash": "................"},
    "ambient": {"kick": "x...............", "shaker": "....o.......o..."},
    # 3/4 patterns are 12 steps, not 16 — a 16-step pattern read modulo 12 puts
    # the backbeat somewhere different in every bar
    "waltz": {"kick": "x...........", "snare": "....x...x...",
              "hat": "o...o...o...", "shaker": "..o...o...o."},
}
GENRE_KIT = {"synth-pop": "pop", "edm": "edm", "rock": "rock", "jazz": "jazz",
             "hip hop": "hip hop", "lo-fi": "lo-fi", "cinematic": "cinematic",
             "ambient": "ambient"}


def _kit_for(genre: str, tsig: int) -> dict:
    if tsig == 3:
        return DRUM_PATTERNS["waltz"]
    g = (genre or "").lower()
    for key in sorted(DRUM_PATTERNS, key=len, reverse=True):
        if key in g:
            return DRUM_PATTERNS[key]
    return DRUM_PATTERNS[GENRE_KIT.get(g, "pop")]


class Timeline:
    """Bar-by-bar map of the whole song: which section, chord, energy and
    absolute beat each bar sits at. Every part writer reads this, so the parts
    agree on harmony and dynamics without talking to each other."""

    def __init__(self, score: dict):
        self.bpb = score["time_signature"]
        self.bpm = score["bpm"]
        self.spb = 60.0 / self.bpm                       # seconds per beat
        self.bar_seconds = self.bpb * self.spb
        self.bars = []
        beat = 0.0
        for si, sec in enumerate(score["sections"]):
            chords, n = sec["chords"], sec["bars"]
            per_bar = 2 if len(chords) >= 2 * n else 1
            for b in range(n):
                if per_bar == 2:
                    cs = [(0.0, chords[(2 * b) % len(chords)]),
                          (self.bpb / 2.0, chords[(2 * b + 1) % len(chords)])]
                else:
                    cs = [(0.0, chords[b % len(chords)])]
                self.bars.append({
                    "abs": len(self.bars), "section": si, "name": sec["name"],
                    "in_section": b, "of_section": n, "energy": sec["energy"],
                    "tracks": sec["tracks"], "fx": sec["fx"],
                    "chords": cs, "start_beat": beat,
                    "first": b == 0, "last": b == n - 1,
                })
                beat += self.bpb
        self.total_beats = beat
        self.duration = beat * self.spb

    def chord_at(self, bar, beat_in_bar):
        cur = bar["chords"][0][1]
        for off, c in bar["chords"]:
            if beat_in_bar + 1e-6 >= off:
                cur = c
        return cur

    def section_bounds(self, si: int) -> tuple:
        bars = [b for b in self.bars if b["section"] == si]
        return (bars[0]["start_beat"] * self.spb,
                (bars[-1]["start_beat"] + self.bpb) * self.spb)


def _hit_vel(mark: str, base: int, energy: float, rng) -> int:
    scale = {"x": 1.0, "o": 0.82, "-": 0.5}.get(mark, 0.0)
    v = base * scale * (0.62 + 0.45 * energy) + rng.uniform(-4, 4)
    return int(max(1, min(127, v)))


def write_drums(tl: Timeline, score: dict, track: dict, rng) -> list:
    """Kit part: the genre pattern, thinned at low energy, with 16th hats at
    high energy, an 8-bar fill and a crash on every section downbeat."""
    kit = _kit_for(score["genre"], tl.bpb)
    steps = 4 * tl.bpb
    notes = []
    for bar in tl.bars:
        if track["track"] not in bar["tracks"]:
            continue
        e = bar["energy"]
        for piece, pattern in kit.items():
            note = DRUM.get(piece)
            if note is None:
                continue
            base = {"kick": 108, "snare": 104, "clap": 96, "hat": 76,
                    "openhat": 82, "ride": 74, "shaker": 62}.get(piece, 88)
            for s in range(steps):
                mark = pattern[s % len(pattern)]
                if mark == ".":
                    continue
                # thin the kit out when the section is quiet, drive it when loud
                if e < 0.45 and piece in ("openhat", "shaker", "ride") and s % 4:
                    continue
                if e < 0.3 and piece == "hat" and s % 4:
                    continue
                if mark == "-" and e < 0.6:
                    continue
                beat = bar["start_beat"] + s / 4.0
                notes.append((beat, 0.22, note, _hit_vel(mark, base, e, rng)))
            # extra 16th hats up top when the section is really going
            if piece == "hat" and e > 0.8:
                for s in range(1, steps, 2):
                    if pattern[s % len(pattern)] == ".":
                        notes.append((bar["start_beat"] + s / 4.0, 0.15, DRUM["hat"],
                                      _hit_vel("-", 74, e, rng)))
        if bar["first"] and bar["energy"] > 0.45:
            notes.append((bar["start_beat"], 1.5, DRUM["crash"],
                          _hit_vel("x", 110, bar["energy"], rng)))
        # fill into the next 8-bar phrase (and into the next section)
        phrase_end = (bar["in_section"] % 8 == 7) or bar["last"]
        if phrase_end and bar["energy"] > 0.5 and tl.bpb == 4:
            for k, note in enumerate(("tom_top", "tom_hi", "tom_mid", "tom_lo")):
                for j in range(2):
                    beat = bar["start_beat"] + 2.0 + k * 0.5 + j * 0.25
                    notes.append((beat, 0.2, DRUM[note],
                                  _hit_vel("o", 96 + 4 * k, bar["energy"], rng)))
    return notes


def write_perc(tl: Timeline, score: dict, track: dict, rng) -> list:
    notes = []
    for bar in tl.bars:
        if track["track"] not in bar["tracks"]:
            continue
        e = bar["energy"]
        for s in range(0, 4 * tl.bpb, 2):
            notes.append((bar["start_beat"] + s / 4.0, 0.12, DRUM["shaker"],
                          _hit_vel("o" if s % 4 else "x", 58, e, rng)))
        if e > 0.6:
            for s in (4, 12):
                if s < 4 * tl.bpb:
                    notes.append((bar["start_beat"] + s / 4.0, 0.2, DRUM["tamb"],
                                  _hit_vel("o", 70, e, rng)))
    return notes


BASS_PATTERNS = {
    "sustain": [(0.0, 4.0)],
    "half": [(0.0, 2.0), (2.0, 2.0)],
    "eighths": [(i * 0.5, 0.45) for i in range(8)],
    "pump": [(i * 0.25, 0.22) for i in range(16)],
    "syncopated": [(0.0, 0.5), (0.75, 0.25), (1.5, 0.5), (2.5, 0.5), (3.0, 0.25),
                   (3.5, 0.5)],
    "walking": [(i * 1.0, 0.9) for i in range(4)],
    "root_five": [(0.0, 1.5), (1.5, 0.5), (2.0, 1.5), (3.5, 0.5)],
}


def _bass_style(genre: str, energy: float) -> str:
    g = (genre or "").lower()
    if "jazz" in g:
        return "walking"
    if "ambient" in g or "cinematic" in g:
        return "sustain" if energy < 0.6 else "half"
    if "edm" in g or "house" in g:
        return "pump" if energy > 0.75 else "eighths"
    if "funk" in g or "hip hop" in g or "lo-fi" in g:
        return "syncopated"
    if energy < 0.4:
        return "half"
    return "eighths" if energy > 0.6 else "root_five"


def write_bass(tl: Timeline, score: dict, track: dict, rng) -> list:
    lo, hi, _ = ROLE_REGISTER["bass"]
    scale = scale_pitches(score["key_pc"], score["mode"])
    notes, prev = [], None
    for bar in tl.bars:
        if track["track"] not in bar["tracks"]:
            continue
        style = _bass_style(score["genre"], bar["energy"])
        pattern = BASS_PATTERNS[style]
        for off, dur in pattern:
            if off >= tl.bpb:
                continue
            chord = tl.chord_at(bar, off)
            root = register_base(chord["root"], lo)
            pitch = root
            if style == "walking":
                # approach the next chord's root by step — what makes a walking
                # line walk instead of just restating the root four times
                step = int(off)
                nxt = tl.chord_at(bar, min(off + 1.0, tl.bpb - 0.01))
                if step == 3 and nxt["root"] != chord["root"]:
                    target = register_base(nxt["root"], lo)
                    pitch = target - 1 if target > root else target + 1
                elif step in (1, 2):
                    pitch = root + [0, 7, 3 if 3 in chord["ivs"] else 4][step % 3]
            elif style in ("root_five", "half"):
                pitch = root + (7 if off >= tl.bpb / 2 else 0)
            elif style in ("eighths", "pump") and bar["energy"] > 0.7:
                # octave pushes on the back half of the bar
                pitch = root + (12 if (off * 4) % 4 == 2 else 0)
            elif style == "syncopated" and off > 2.0:
                pitch = root + (12 if rng.random() < 0.35 else 0)
            pitch = max(lo, min(hi + 12, pitch))
            vel = int(76 + 34 * bar["energy"] + rng.uniform(-5, 5))
            notes.append((bar["start_beat"] + off, dur * 0.94,
                          pitch, max(30, min(120, vel))))
            prev = pitch
    return notes


CHORD_RHYTHMS = {
    "sustain": [(0.0, 4.0)],
    "half": [(0.0, 1.9), (2.0, 1.9)],
    "quarters": [(i * 1.0, 0.9) for i in range(4)],
    "eighths": [(i * 0.5, 0.42) for i in range(8)],
    "offbeat": [(0.5, 0.45), (1.5, 0.45), (2.5, 0.45), (3.5, 0.45)],
    "charleston": [(0.0, 0.9), (1.75, 0.7), (2.5, 0.4)],
    "piano_comp": [(0.0, 1.4), (1.5, 0.4), (2.0, 1.4), (3.5, 0.4)],
}


def _chord_rhythm(genre: str, energy: float, rng) -> str:
    g = (genre or "").lower()
    if "jazz" in g:
        return "charleston"
    if "reggae" in g or "funk" in g:
        return "offbeat"
    if "rock" in g or "metal" in g:
        return "eighths" if energy > 0.7 else "quarters"
    if "edm" in g or "house" in g:
        return "offbeat" if energy > 0.7 else "sustain"
    if "lo-fi" in g or "hip hop" in g:
        return "piano_comp"
    if energy < 0.4:
        return "sustain"
    return "quarters" if energy < 0.75 else "eighths"


def write_chords(tl: Timeline, score: dict, track: dict, rng) -> list:
    lo, hi, _ = ROLE_REGISTER["chords"]
    notes, prev = [], None
    for bar in tl.bars:
        if track["track"] not in bar["tracks"]:
            continue
        rhythm = CHORD_RHYTHMS[_chord_rhythm(score["genre"], bar["energy"], rng)]
        for off, dur in rhythm:
            if off >= tl.bpb:
                continue
            chord = tl.chord_at(bar, off)
            voicing = chord_pitches(chord, lo, hi, size=4, prev=prev)
            prev = voicing or prev
            vel = int(58 + 40 * bar["energy"] + rng.uniform(-4, 4))
            for k, p in enumerate(voicing):
                # voices of a chord start together here on purpose — the strum /
                # roll is articulated later by spread_chords, which needs to be
                # able to SEE the chord. Jittering each voice here instead made
                # every note its own group, so no chord was ever spread.
                notes.append((bar["start_beat"] + off,
                              min(dur, tl.bpb - off) * 0.95, p,
                              max(24, min(115, vel - 3 * k))))
    return notes


def write_pad(tl: Timeline, score: dict, track: dict, rng) -> list:
    lo, hi, _ = ROLE_REGISTER["pad"]
    notes, prev = [], None
    for bar in tl.bars:
        if track["track"] not in bar["tracks"]:
            continue
        for off, chord in bar["chords"]:
            span = tl.bpb - off if len(bar["chords"]) == 1 else tl.bpb / 2.0
            voicing = chord_pitches(chord, lo, hi, size=4, prev=prev)
            prev = voicing or prev
            vel = int(40 + 30 * bar["energy"])
            for k, p in enumerate(voicing):
                notes.append((bar["start_beat"] + off, span * 1.02, p,
                              max(18, min(90, vel - 2 * k))))
    return notes


ARP_SHAPES = {"up": [0, 1, 2, 3], "down": [3, 2, 1, 0], "updown": [0, 1, 2, 3, 2, 1],
              "wide": [0, 2, 1, 3], "trance": [0, 1, 2, 1, 3, 2, 1, 0]}


def write_arp(tl: Timeline, score: dict, track: dict, rng) -> list:
    lo, hi, _ = ROLE_REGISTER["arp"]
    shape = ARP_SHAPES[rng.choice(list(ARP_SHAPES))]
    notes = []
    for bar in tl.bars:
        if track["track"] not in bar["tracks"]:
            continue
        rate = 0.25 if bar["energy"] > 0.55 else 0.5      # 16ths vs 8ths
        n_steps = int(tl.bpb / rate)
        for s in range(n_steps):
            off = s * rate
            chord = tl.chord_at(bar, off)
            tones = chord_pitches(chord, lo, hi, size=4)
            if not tones:
                continue
            idx = shape[s % len(shape)]
            pitch = tones[idx % len(tones)] + (12 if idx >= len(tones) else 0)
            vel = int(52 + 34 * bar["energy"] + (8 if s % 4 == 0 else 0) + rng.uniform(-4, 4))
            notes.append((bar["start_beat"] + off, rate * 0.92,
                          min(hi + 12, pitch), max(24, min(112, vel))))
    return notes


def make_motif(rng, bars: int, tsig: int, density: float, span: int) -> list:
    """A reusable melodic idea as (beat_offset, duration, scale_degree).

    Rhythm comes from a small bank of singable cells rather than random onsets —
    random rhythm is the fastest way to make a generated melody sound like an
    exercise. Contour is a seeded walk that lands on a stable degree so the
    phrase resolves."""
    cells = [[(0.0, 1.0), (1.0, 0.5), (1.5, 0.5), (2.0, 2.0)],
             [(0.0, 0.5), (0.5, 0.5), (1.0, 1.0), (2.0, 1.0), (3.0, 1.0)],
             [(0.0, 1.5), (1.5, 0.5), (2.0, 1.0), (3.0, 1.0)],
             [(0.0, 2.0), (2.0, 1.0), (3.0, 1.0)],
             [(0.0, 0.75), (0.75, 0.75), (1.5, 0.5), (2.0, 1.5)],
             [(0.0, 1.0), (1.5, 0.5), (2.0, 0.5), (2.5, 0.5), (3.0, 1.0)]]
    motif, degree = [], rng.choice([0, 2, 4])
    for b in range(bars):
        cell = cells[rng.randrange(len(cells))]
        if density < 0.5:                    # thin the cell out for quiet parts
            cell = [c for i, c in enumerate(cell) if i % 2 == 0] or cell[:1]
        for off, dur in cell:
            if off >= tsig:
                continue
            motif.append((b * tsig + off, dur, degree))
            # mostly steps, occasional leap, kept inside the register span
            move = rng.choice([-2, -1, -1, 1, 1, 2, 3, -3]) if rng.random() < 0.85 \
                else rng.choice([-4, 4, 5])
            degree = max(-2, min(span, degree + move))
    if motif:                                # resolve the phrase
        last = motif[-1]
        stable = min((0, 2, 4, 7), key=lambda d: abs(d - last[2]))
        motif[-1] = (last[0], max(last[1], 1.5), stable)
    return motif


def _motif_key(name: str) -> str:
    n = (name or "").lower()
    if any(k in n for k in ("chorus", "hook", "drop", "theme", "climax", "shout", "bloom")):
        return "chorus"
    if any(k in n for k in ("bridge", "middle", "solo", "break")):
        return "bridge"
    return "verse"


def write_lead(tl: Timeline, score: dict, track: dict, rng, offset: int = 0) -> list:
    """The tune. One motif per section TYPE, so the chorus hook recurs and the
    verses share an identity — then chord-tone snapping on strong beats keeps it
    consonant as the harmony moves underneath."""
    role = track["role"]
    lo, hi, center = ROLE_REGISTER[role]
    key_pc, mode = score["key_pc"], score["mode"]
    base = register_base(key_pc, lo + 2)
    span = 9 if role == "lead" else 7
    motifs = {k: make_motif(rng, 2, tl.bpb, 0.8 if k == "chorus" else 0.6, span)
              for k in ("verse", "chorus", "bridge")}
    notes = []
    for si, sec in enumerate(score["sections"]):
        bars = [b for b in tl.bars if b["section"] == si]
        if not bars or track["track"] not in bars[0]["tracks"]:
            continue
        motif = motifs[_motif_key(sec["name"])]
        lift = 12 if (sec["energy"] > 0.85 and role == "lead") else 0
        # phrase the section in 4-bar groups: motif, motif, motif, variation
        for grp in range(0, len(bars), 4):
            chunk = bars[grp:grp + 4]
            for rep in range(0, len(chunk), 2):
                pair = chunk[rep:rep + 2]
                if not pair:
                    continue
                vary = (grp + rep) // 2 % 2 == 1
                for m_off, m_dur, degree in motif:
                    bar_i = int(m_off // tl.bpb)
                    if bar_i >= len(pair):
                        continue
                    bar = pair[bar_i]
                    beat_in_bar = m_off % tl.bpb
                    d = degree + (1 if vary and rng.random() < 0.4 else 0) + offset
                    pitch = degree_pitch(d, key_pc, mode, base) + lift
                    chord = tl.chord_at(bar, beat_in_bar)
                    if beat_in_bar % 2 < 0.01:           # strong beat -> chord tone
                        pitch = nearest_chord_tone(pitch, chord)
                    pitch = max(lo, min(hi, pitch))
                    vel = int(64 + 40 * bar["energy"] +
                              (6 if beat_in_bar < 0.01 else 0) + rng.uniform(-4, 4))
                    # no jitter here either: humanize_timing owns timing feel, and
                    # its correlated drift reads as a player where independent
                    # per-note noise reads as unsteadiness
                    notes.append((bar["start_beat"] + beat_in_bar,
                                  m_dur * 0.9, pitch, max(30, min(120, vel))))
    return notes


def write_counter(tl: Timeline, score: dict, track: dict, rng) -> list:
    """A third below the lead's idea, one bar later — lands in the lead's gaps
    rather than fighting it for the same beats."""
    notes = write_lead(tl, score, track, rng, offset=-2)
    shift = tl.bpb
    out = []
    for beat, dur, pitch, vel in notes:
        if beat + shift < tl.total_beats:
            out.append((beat + shift, dur, pitch, int(vel * 0.82)))
    return out


WRITERS = {"drums": write_drums, "perc": write_perc, "bass": write_bass,
           "chords": write_chords, "pad": write_pad, "arp": write_arp,
           "lead": write_lead, "counter": write_counter}


# ============================================================ performance realism
#
# The single biggest reason a rendered arrangement "sounds like MIDI" is not the
# samples — it is that nothing performs. Every sustained note is a flat block,
# every kick is the identical sample at the identical velocity, all voices of a
# chord land on the same millisecond, and timing error is white noise. These
# passes run over the finished parts, so the writers above stay about music and
# this stays about performance.

# Sustained roles get a within-note expression curve; struck/plucked ones don't
# (a piano note's loudness is decided at the hammer, not after it).
SUSTAINED_ROLES = ("pad", "lead", "counter")
# GM patches that hold and swell regardless of role: strings, brass, winds, organ,
# choir and the synth pads.
SUSTAINED_PROGRAMS = set(range(40, 55)) | set(range(56, 80)) | set(range(16, 21)) \
    | set(range(88, 96)) | {52, 53, 54}


def is_sustained(track: dict) -> bool:
    return track["role"] in SUSTAINED_ROLES or track["program"] in SUSTAINED_PROGRAMS


def expression_curve(notes: list, track: dict, tl: Timeline, rng) -> list:
    """CC11 points shaping loudness WITHIN each note — verified to modulate
    FluidR3 (a swelled note renders 0.05 -> 0.13 where a flat one sits at 0.17).

    CC11 is per-channel, and each track renders as its own MIDI file, so for a
    polyphonic pad the curve moves the whole chord together. That is still much
    closer to a played part than the dead-flat sustain it replaces.

    Returns [(beat, controller, value)]."""
    if not notes:
        return []
    # group notes that start together — one curve per chord, not per voice
    events, groups = [], {}
    for beat, dur, pitch, vel in notes:
        groups.setdefault(round(beat, 3), []).append((dur, vel))
    keys = sorted(groups)
    for gi, start in enumerate(keys):
        dur = max(d for d, _v in groups[start])
        vel = max(v for _d, v in groups[start])
        nxt = keys[gi + 1] if gi + 1 < len(keys) else start + dur
        span = max(0.12, min(dur, nxt - start))
        # a played swell: soft entry, growth into the note's body, slight decay
        peak = 0.72 + 0.28 * (vel / 127.0) + rng.uniform(-0.06, 0.06)
        entry = 0.34 + 0.3 * (vel / 127.0) + rng.uniform(-0.05, 0.05)
        shape = [(0.0, entry), (0.35, peak), (0.72, peak * 0.94),
                 (1.0, peak * 0.82)]
        steps = max(3, min(14, int(span * 6)))
        for k in range(steps + 1):
            f = k / steps
            # piecewise-linear through the shape points
            val = shape[-1][1]
            for (f0, v0), (f1, v1) in zip(shape, shape[1:]):
                if f0 <= f <= f1:
                    val = v0 + (v1 - v0) * ((f - f0) / max(f1 - f0, 1e-6))
                    break
            events.append((start + f * span, 11, int(max(8, min(127, val * 127)))))
    return events


# GM gives several usable takes of each core drum voice. Rotating them is what
# stops a hat pattern sounding like one sample retriggered 16 times a bar.
DRUM_ALTS = {
    DRUM["kick"]: (DRUM["kick"], DRUM["kick"], DRUM["kick2"]),
    DRUM["snare"]: (DRUM["snare"], DRUM["snare"], DRUM["snare2"]),
    DRUM["hat"]: (DRUM["hat"], DRUM["hat"], DRUM["hat"], DRUM["pedalhat"]),
}


def humanize_drums(notes: list, tl: Timeline, rng) -> list:
    """Round-robin the repeated voices and vary velocity within each one.

    Also nudges the backbeat a few ms late — snares landing exactly on the grid
    is the most recognisable drum-machine tell there is."""
    out, counts = [], {}
    for beat, dur, pitch, vel in notes:
        alts = DRUM_ALTS.get(pitch)
        if alts:
            n = counts.get(pitch, 0)
            counts[pitch] = n + 1
            pitch = alts[n % len(alts)]
        if pitch in (DRUM["snare"], DRUM["snare2"]):
            beat += rng.uniform(0.004, 0.014) / tl.spb      # 4-14 ms, laid back
        # velocity spread WITHIN a voice, so repeated hits aren't identical
        vel = int(max(1, min(127, vel * rng.uniform(0.88, 1.06))))
        out.append((beat, dur, pitch, vel))
    return out


# Per-role feel: where an instrument habitually sits against the click. A bass
# slightly ahead and a pad slightly behind is most of what "a band" sounds like.
ROLE_PUSH = {"drums": 0.0, "perc": 0.004, "bass": -0.006, "chords": 0.004,
             "pad": 0.014, "arp": 0.0, "lead": 0.006, "counter": 0.010}


def humanize_timing(notes: list, track: dict, tl: Timeline, rng) -> list:
    """Correlated timing drift instead of per-note white noise.

    A player's timing wanders slowly and then corrects; independent random
    offsets per note just sound unsteady. A slow random walk (bounded, gently
    pulled back to zero) reads as human where jitter reads as broken."""
    if not notes:
        return notes
    # everything here is defined in SECONDS and converted, because a player's
    # timing error doesn't shrink when the tempo rises — expressing it in beats
    # would make fast songs sloppy and slow ones stiff
    push = ROLE_PUSH.get(track["role"], 0.0) / tl.spb
    sigma, cap = 0.005 / tl.spb, 0.016 / tl.spb            # 5 ms step, 16 ms cap
    drift, out = 0.0, []
    for beat, dur, pitch, vel in sorted(notes, key=lambda n: n[0]):
        drift = drift * 0.82 + rng.gauss(0.0, sigma)
        drift = max(-cap, min(cap, drift))
        out.append((max(0.0, beat + push + drift), dur, pitch, vel))
    return out


def spread_chords(notes: list, track: dict, tl: Timeline, rng) -> list:
    """Offset the voices of a chord so it is played, not stamped.

    Guitars strum low-to-high over ~28ms; pianos roll ~10ms; strings and pads
    enter raggedly because the players don't breathe together. Same operation,
    three different spreads — in seconds, so a strum takes the same real time
    whatever the tempo."""
    guitar = 24 <= track["program"] <= 31
    if track["role"] not in ("chords", "pad") and not guitar:
        return notes
    seconds = 0.028 if guitar else (0.010 if track["role"] == "chords" else 0.040)
    spread = seconds / tl.spb
    groups = {}
    for n in notes:
        groups.setdefault(round(n[0], 3), []).append(n)
    out = []
    for start, grp in groups.items():
        if len(grp) < 2:
            out += grp
            continue
        # strums run bottom-up; ensemble entries are simply uneven
        order = sorted(grp, key=lambda n: n[2]) if guitar else \
            sorted(grp, key=lambda n: rng.random())
        for k, (beat, dur, pitch, vel) in enumerate(order):
            off = spread * (k / max(len(order) - 1, 1))
            if not guitar and track["role"] == "pad":
                off = rng.uniform(0.0, spread)
            # the later a voice enters, the softer it speaks
            out.append((beat + off, max(0.05, dur - off), pitch,
                        int(max(1, min(127, vel * (1.0 - 0.06 * k))))))
    return out


def shape_lengths(notes: list, track: dict, rng) -> list:
    """Note lengths a player would use: melodic lines overlap into the next note
    (legato), rhythmic parts breathe (staccato). Uniform durations are another
    reason a part reads as data rather than playing."""
    role = track["role"]
    if role in ("drums", "perc"):
        return notes
    legato = role in ("lead", "counter", "pad")
    out = []
    for beat, dur, pitch, vel in notes:
        if legato:
            dur *= rng.uniform(1.02, 1.12)          # run into the next note
        else:
            dur *= rng.uniform(0.82, 0.98)          # let go before the next
        out.append((beat, max(0.05, dur), pitch, vel))
    return out


def compose(score: dict) -> tuple:
    """score -> (Timeline, parts, ccs) where parts is
    {track: [(beat, dur_beats, pitch, vel)]} and ccs is
    {track: [(beat, controller, value)]}."""
    tl = Timeline(score)
    parts, ccs = {}, {}
    for track in score["instruments"]:
        # Seeded by track NAME, not position: editing the arrangement (adding a
        # track, reordering, renaming another part) must not reshuffle the notes
        # of every track after it. Stable seeds are what make an edit-and-
        # re-render loop usable.
        seed = (score["seed"] * 7919 + zlib.crc32(track["track"].encode())) % (2 ** 31)
        rng = random.Random(seed)
        if track.get("muted"):
            # kept in the score (so the editor still shows the track and its
            # settings) but contributes nothing to the render
            parts[track["track"]] = []
            continue
        notes = WRITERS[track["role"]](tl, score, track, rng)
        if score["swing"] > 0.01:
            notes = _apply_swing(notes, score["swing"])

        edited = bool(track.get("notes_override"))
        if edited:
            # hand-edited notes replace the generated part entirely
            notes = [tuple(n[:4]) for n in track["notes_override"]]

        if score.get("humanize", True):
            # An edited part is authoritative: the timing, spread and length
            # passes are SKIPPED for it, because the user placed those notes
            # deliberately and re-humanizing them both moves them off the grid
            # they were drawn on and compounds a little more drift on every
            # edit-and-re-render cycle. Expression still applies — respecting
            # the edit shouldn't mean the part goes back to sounding dead.
            if not edited:
                if track["role"] == "drums":
                    notes = humanize_drums(notes, tl, rng)
                notes = spread_chords(notes, track, tl, rng)
                notes = shape_lengths(notes, track, rng)
                notes = humanize_timing(notes, track, tl, rng)
            if is_sustained(track):
                ccs[track["track"]] = expression_curve(notes, track, tl, rng)
        parts[track["track"]] = notes
    return tl, parts, ccs


def _apply_swing(notes: list, amount: float) -> list:
    """Push every off-beat eighth later — the difference between a straight grid
    and a groove that breathes."""
    out = []
    for beat, dur, pitch, vel in notes:
        frac = beat % 1.0
        if abs(frac - 0.5) < 0.06:
            beat += 0.16 * amount
        out.append((beat, dur, pitch, vel))
    return out


# ============================================================ MIDI

def parts_to_midi(score: dict, tl: Timeline, parts: dict, path: Path,
                  only: str = None, tail: float = 3.0, ccs: dict = None):
    """Write MIDI. `only` renders a single track (for stems); otherwise the whole
    arrangement, with pan (CC10) and volume (CC7) so the file opens in a DAW
    already roughly balanced and placed in the stereo field."""
    import pretty_midi

    pm = pretty_midi.PrettyMIDI(initial_tempo=float(score["bpm"]))
    for track in score["instruments"]:
        if only and track["track"] != only:
            continue
        notes = parts.get(track["track"]) or []
        drum = track["role"] in ("drums", "perc")
        inst = pretty_midi.Instrument(program=0 if drum else track["program"],
                                      is_drum=drum, name=track["track"][:24])
        for beat, dur, pitch, vel in notes:
            t0 = beat * tl.spb
            inst.notes.append(pretty_midi.Note(velocity=int(vel), pitch=int(pitch),
                                               start=t0,
                                               end=t0 + max(0.05, dur * tl.spb)))
        # within-note expression (see expression_curve) — the difference between
        # a held chord and a played one
        for beat, cc, val in (ccs or {}).get(track["track"], ()):
            inst.control_changes.append(
                pretty_midi.ControlChange(int(cc), int(val), max(0.0, beat * tl.spb)))
        if not only:
            pan = int(round(64 + 63 * track["pan"]))
            inst.control_changes.append(pretty_midi.ControlChange(10, max(0, min(127, pan)), 0.0))
            inst.control_changes.append(
                pretty_midi.ControlChange(7, max(1, min(127, int(track["level"] * 100))), 0.0))
        # a silent controller past the last note so FluidSynth renders the full
        # release tail instead of cutting the file at the final note-off
        inst.control_changes.append(
            pretty_midi.ControlChange(11, 127, tl.duration + tail))
        pm.instruments.append(inst)
    pm.write(str(path))
    return path


# ============================================================ audio: FX + automation

def _biquad(x, kind: str, freq: float, q: float = 0.707, gain_db: float = 0.0):
    from scipy.signal import butter, iirfilter, sosfilt
    freq = max(20.0, min(SR * 0.45, freq))
    if kind in ("low", "high"):
        sos = butter(2, freq, btype=kind, fs=SR, output="sos")
    else:
        sos = iirfilter(2, freq, btype="low" if kind == "shelf_lo" else "high",
                        ftype="butter", fs=SR, output="sos")
        wet = sosfilt(sos, x, axis=0)
        return x + wet * (10 ** (gain_db / 20.0) - 1.0)
    return sosfilt(sos, x, axis=0)


def _sweep_lowpass(x, cutoff_env, block: int = 2048):
    """Time-varying lowpass. Filter state carries across blocks (sosfilt zi), so
    a full sweep from closed to open has no zipper noise at the block edges —
    this is the workhorse behind every filter-automation move.

    Cutoff is quantized to sixth-of-a-semitone steps and the designed filters
    are cached: a four-minute sweep is ~5000 blocks, and redesigning a Butterworth
    for each one costs more than the filtering itself."""
    from scipy.signal import butter, sosfilt, sosfilt_zi
    env = np.atleast_1d(np.asarray(cutoff_env, dtype=np.float32))
    out = np.zeros_like(x)
    design = {}
    zi = None
    for i in range(0, len(x), block):
        seg = x[i:i + block]
        if not len(seg):
            break
        frac = float(np.clip(env[min(i, len(env) - 1)], 0.0, 1.0))
        q = round(frac * 72) / 72.0                   # ~1/6-semitone resolution
        hz = 120.0 * (18000.0 / 120.0) ** q           # log-spaced: musical sweep
        if hz >= SR * 0.44:
            out[i:i + len(seg)] = seg
            zi = None
            continue
        sos = design.get(q)
        if sos is None:
            sos = design[q] = butter(2, hz, btype="low", fs=SR, output="sos")
        if zi is None:
            zi = sosfilt_zi(sos) * float(seg[0])
        y, zi = sosfilt(sos, seg, zi=zi)
        out[i:i + len(seg)] = y
    return out


def make_ir(seconds: float, decay: float, damp: float, rng) -> "np.ndarray":
    """Synthesized stereo reverb impulse: decaying noise, damped and predelayed.

    A convolution reverb built this way costs ~15 lines and one FFT per track,
    and sounds far better than a comb-filter box — which matters because six or
    seven tracks all sharing one space is what makes a mix sound like a record
    instead of a MIDI file."""
    n = max(256, int(SR * seconds))
    t = np.arange(n) / SR
    env = np.exp(-t * decay)
    ir = rng.standard_normal((n, 2)) * env[:, None]
    ir = _biquad(ir, "low", damp)
    pre = int(0.012 * SR)
    ir[:pre] = 0.0
    ir[:, 1] = np.roll(ir[:, 1], 37)               # decorrelate the channels
    peak = np.sqrt((ir ** 2).sum(axis=0)).max() or 1.0
    return ir / peak


def convolve_wet(mono, ir):
    # oaconvolve, not fftconvolve: overlap-add is the right algorithm for a
    # multi-minute signal against a few-second kernel, and avoids FFT-ing ten
    # million samples per channel per track.
    from scipy.signal import oaconvolve
    wet = np.zeros((len(mono), 2), dtype=np.float32)
    for c in range(2):
        wet[:, c] = oaconvolve(mono, ir[:, c])[:len(mono)]
    return wet


def tap_delay(mono, delay_samples: int, feedback: float, taps: int = 6,
              ping_pong: bool = True):
    """Tempo-synced echo as a sum of decaying taps — cheaper than a sample-wise
    feedback loop and indistinguishable at these settings."""
    out = np.zeros((len(mono), 2), dtype=np.float32)
    for k in range(1, taps + 1):
        d = delay_samples * k
        if d >= len(mono):
            break
        g = feedback ** k
        seg = mono[:len(mono) - d] * g
        if ping_pong:
            c = (k - 1) % 2
            out[d:, c] += seg
            out[d:, 1 - c] += seg * 0.35
        else:
            out[d:, 0] += seg
            out[d:, 1] += seg
    return _biquad(out, "low", 5200.0)


def envelope(n: int, spec: list, tl: Timeline, score: dict, param: str,
             default: float) -> "np.ndarray":
    """Build a per-sample automation curve for one parameter.

    'ramp' with a section name sweeps only inside that section (a build opening
    its filter); without one it sweeps across the whole song. 'autopan' and
    'tremolo' are LFOs with rates in bars, so they stay locked to the grid."""
    env = np.full(n, default, dtype=np.float32)
    for a in spec:
        if a["param"] != param:
            continue
        if a["mode"] in ("autopan", "tremolo"):
            period = max(0.05, a["bars"] * tl.bar_seconds)
            t = np.arange(n) / SR
            lfo = np.sin(2 * np.pi * t / period)
            if a["mode"] == "autopan":
                env = np.clip(default + a["depth"] * lfo, -1.0, 1.0).astype(np.float32)
            else:
                env = (default * (1.0 - a["depth"] * 0.5 * (1.0 - lfo))).astype(np.float32)
            continue
        s0, s1 = 0, n
        if a["section"]:
            want = a["section"].lower()
            match = next((si for si, sec in enumerate(score["sections"])
                          if sec["name"].lower() == want), None)
            if match is None:                # tolerate near-misses ("build" / "buildup")
                match = next((si for si, sec in enumerate(score["sections"])
                              if want in sec["name"].lower()
                              or sec["name"].lower() in want), None)
            if match is None:
                # a section name that matches nothing is skipped, NOT applied to
                # the whole song — one typo'd name shouldn't blanket-filter the
                # entire track for four minutes
                continue
            t0, t1 = tl.section_bounds(match)
            s0, s1 = int(t0 * SR), min(n, int(t1 * SR))
        if s1 <= s0:
            continue
        ramp = np.linspace(a["from"], a["to"], s1 - s0, dtype=np.float32)
        env[s0:s1] = ramp
        if s1 < n:
            env[s1:] = a["to"]
    return env


TONE_EQ = {"warm": (("shelf_lo", 220.0, 3.0), ("low", 7000.0, 0.0)),
           "bright": (("shelf_hi", 3500.0, 3.5),),
           "dark": (("low", 2600.0, 0.0),),
           "thin": (("high", 320.0, 0.0),),
           "neutral": ()}


def process_track(stem_wav: Path, track: dict, tl: Timeline, score: dict,
                  n_out: int, rng) -> "np.ndarray":
    """One track's whole channel strip: saturation -> tone -> automated filter
    -> automated delay & reverb sends -> volume & pan automation -> stereo."""
    import soundfile as sf

    audio, sr = sf.read(str(stem_wav), always_2d=True, dtype="float32")
    if sr != SR:
        raise RuntimeError(f"{stem_wav.name}: unexpected sample rate {sr}")
    mono = audio.mean(axis=1)
    if len(mono) < n_out:
        mono = np.pad(mono, (0, n_out - len(mono)))
    mono = mono[:n_out]
    auto = track["automation"]

    if track["drive"] > 0.01:
        k = 1.0 + 14.0 * track["drive"]
        mono = np.tanh(mono * k) / np.tanh(k) * (1.0 - 0.25 * track["drive"])
    for kind, freq, gain in TONE_EQ.get(track["tone"], ()):
        mono = _biquad(mono, kind, freq, gain_db=gain)
    if any(a["param"] == "cutoff" for a in auto):
        mono = _sweep_lowpass(mono, envelope(n_out, auto, tl, score, "cutoff", 1.0))

    wet = np.zeros((n_out, 2), dtype=np.float32)
    if track["delay"] > 0.01:
        # dotted-eighth on leads/arps (the classic wide echo), straight eighth
        # elsewhere — a quarter-note delay on a busy part just smears it
        beats = 0.75 if track["role"] in ("lead", "arp", "counter") else 0.5
        d = int(beats * tl.spb * SR)
        send = envelope(n_out, auto, tl, score, "delay", track["delay"])
        wet += tap_delay(mono, d, 0.5) * send[:, None]
    if track["reverb"] > 0.01:
        size = {"pad": 2.6, "lead": 1.8, "chords": 1.5, "arp": 1.6,
                "bass": 0.8, "drums": 1.0, "perc": 1.0, "counter": 1.8}[track["role"]]
        ir = make_ir(size * (0.7 + 0.6 * track["reverb"]),
                     decay=3.2 / size, damp=4200.0 + 3000.0 * track["reverb"], rng=rng)
        send = envelope(n_out, auto, tl, score, "reverb", track["reverb"])
        wet += convolve_wet(mono, ir) * (send * 0.85)[:, None]

    vol = envelope(n_out, auto, tl, score, "volume", 1.0) * track["level"]
    pan = envelope(n_out, auto, tl, score, "pan", track["pan"])
    # constant-power panning: a part swept across the field keeps its loudness
    ang = (np.clip(pan, -1.0, 1.0) + 1.0) * (math.pi / 4.0)
    gl, gr = np.cos(ang), np.sin(ang)
    out = np.empty((n_out, 2), dtype=np.float32)
    dry = mono * vol
    out[:, 0] = dry * gl + wet[:, 0] * vol
    out[:, 1] = dry * gr + wet[:, 1] * vol
    return out


# ---- production FX (synthesized, not sampled) --------------------------------

def _noise(n, rng):
    return rng.standard_normal(n).astype(np.float32)


def riser(seconds: float, rng) -> "np.ndarray":
    """Noise sweep + rising sine into a downbeat: the standard build-up cue."""
    n = int(seconds * SR)
    t = np.arange(n) / SR
    frac = (t / max(t[-1], 1e-6)) ** 1.6
    swept = _sweep_lowpass(_noise(n, rng) * 0.5, 0.25 + 0.72 * frac)
    tone = np.sin(2 * np.pi * np.cumsum(220.0 * (5.5 ** frac)) / SR) * 0.16
    amp = frac ** 1.4
    mono = (swept * 0.75 + tone) * amp
    out = np.stack([mono, np.roll(mono, 90)], axis=1)
    return _biquad(out, "high", 180.0) * 0.55


def impact(rng) -> "np.ndarray":
    """Sub boom + damped noise crack for a section downbeat."""
    n = int(2.2 * SR)
    t = np.arange(n) / SR
    sub = np.sin(2 * np.pi * np.cumsum(np.linspace(78.0, 32.0, n)) / SR) * np.exp(-t * 2.6)
    crack = _biquad(_noise(n, rng), "low", 2600.0) * np.exp(-t * 7.0) * 0.35
    mono = (sub * 0.85 + crack).astype(np.float32)
    return np.stack([mono, np.roll(mono, 23)], axis=1) * 0.8


def downlifter(seconds: float, rng) -> "np.ndarray":
    n = int(seconds * SR)
    t = np.arange(n) / SR
    frac = t / max(t[-1], 1e-6)
    swept = _sweep_lowpass(_noise(n, rng) * 0.45, 0.85 - 0.7 * frac)
    tone = np.sin(2 * np.pi * np.cumsum(np.linspace(900.0, 120.0, n)) / SR) * 0.14
    mono = (swept + tone) * (1.0 - frac) ** 1.2
    return np.stack([mono, np.roll(mono, -70)], axis=1) * 0.5


def kick_duck(parts: dict, tl: Timeline, score: dict, n: int) -> "np.ndarray":
    """Sidechain envelope from the actual kick hits, so pads/chords breathe with
    the beat. Built from the MIDI rather than by detecting the audio — exact,
    and free."""
    env = np.ones(n, dtype=np.float32)
    drums = [t["track"] for t in score["instruments"] if t["role"] == "drums"]
    hold, rel = int(0.045 * SR), int(0.17 * SR)
    shape = np.concatenate([np.full(hold, 0.35),
                            np.linspace(0.35, 1.0, rel)]).astype(np.float32)
    for name in drums:
        for beat, _dur, pitch, _vel in parts.get(name, []):
            if pitch not in (DRUM["kick"], DRUM["kick2"]):
                continue
            i = int(beat * tl.spb * SR)
            j = min(n, i + len(shape))
            if i < n:
                env[i:j] = np.minimum(env[i:j], shape[:j - i])
    return env


def add_production_fx(bus, tl: Timeline, score: dict, rng):
    """Risers, impacts, downlifters and drop silences, placed from each section's
    fx list. A 'riser' belongs to the section it builds OUT OF, so it lands
    exactly on the next section's downbeat."""
    n = len(bus)

    def mix_at(sig, t0):
        i = max(0, int(t0 * SR))
        j = min(n, i + len(sig))
        if j > i:
            bus[i:j] += sig[:j - i]

    for si, sec in enumerate(score["sections"]):
        t0, t1 = tl.section_bounds(si)
        for fx in sec["fx"]:
            if fx == "riser":
                length = min(2 * tl.bar_seconds, t1 - t0)
                mix_at(riser(length, rng), t1 - length)
            elif fx == "impact":
                mix_at(impact(rng), t0)
            elif fx == "downlifter":
                mix_at(downlifter(min(1.6, tl.bar_seconds), rng), t1)
            elif fx == "drop":
                # a beat of near-silence before the downbeat makes the drop hit
                i = max(0, int((t0 - tl.spb) * SR))
                j = min(n, int(t0 * SR))
                if j > i:
                    bus[i:j] *= np.linspace(0.6, 0.02, j - i)[:, None]
            elif fx == "filter_sweep":
                i, j = max(0, int(t0 * SR)), min(n, int(t1 * SR))
                if j > i + SR:
                    # opens from ~1.1 kHz, not ~470 Hz, and on a curve that
                    # clears the muffled zone early: this filters the WHOLE mix,
                    # and a linear sweep from near-closed left a 30-second
                    # section sounding like it was playing underwater
                    envl = (0.45 + 0.55 * np.linspace(0.0, 1.0, j - i,
                                                      dtype=np.float32) ** 0.55)
                    for c in range(2):
                        bus[i:j, c] = _sweep_lowpass(bus[i:j, c], envl)
    return bus


# ============================================================ render pipeline

def _run(cmd, **kw):
    p = subprocess.run(cmd, stdout=subprocess.DEVNULL, stderr=subprocess.PIPE,
                       text=True, errors="replace", **kw)
    if p.returncode != 0:
        raise RuntimeError(f"{Path(cmd[0]).name} failed: "
                           f"{(p.stderr or '').strip()[-400:]}")


def _ffmpeg_exe(name: str = "ffmpeg") -> str:
    import glob
    found = shutil.which(name)
    if found:
        return found
    hits = glob.glob(os.path.join(os.environ.get("LOCALAPPDATA", ""), "Microsoft",
                     "WinGet", "Packages", "Gyan.FFmpeg*", "**", f"{name}.exe"),
                     recursive=True)
    if hits:
        return hits[0]
    raise RuntimeError(f"{name} not found — install it: winget install Gyan.FFmpeg")


def _progress(outdir: Path, step: int, total: int, current: str):
    """Live progress for the web UI — the server polls this file while the
    subprocess runs, so a 60-second render isn't a blank spinner."""
    try:
        (outdir / "progress.json").write_text(json.dumps(
            {"step": step, "total": total, "current": current}), encoding="utf-8")
    except Exception:
        pass
    print(f"[{step}/{total}] {current}", flush=True)


def render_stem_wav(midi: Path, wav: Path, soundfont: Path, gain: float = 0.8):
    _run([str(FLUIDSYNTH), "-ni", "-F", str(wav), "-r", str(SR), "-g", str(gain),
          "-o", "synth.reverb.active=0", "-o", "synth.chorus.active=0",
          "-o", "synth.polyphony=512",
          str(soundfont), str(midi)])
    return wav


def master(mix_wav: Path, out_wav: Path, out_mp3: Path):
    """Bus glue + streaming loudness. ffmpeg does the dynamics here for the same
    reason lullabykit does: loudnorm's two-pass-in-one-filter behaviour is hard
    to beat by hand, and the chain is one process."""
    ff = _ffmpeg_exe()
    # loudnorm before the limiter, not after: normalizing last would undo the
    # limiting and put peaks back over the ceiling
    chain = ("highpass=f=28,"
             "acompressor=threshold=0.12:ratio=2.4:attack=12:release=220:makeup=1.4,"
             "loudnorm=I=-14:TP=-1.0:LRA=11,"
             "alimiter=limit=0.97:attack=5:release=60,"
             "afade=t=in:d=0.05")
    _run([ff, "-y", "-v", "error", "-i", str(mix_wav), "-af", chain,
          "-ar", str(SR), "-c:a", "pcm_s16le", str(out_wav)])
    _run([ff, "-y", "-v", "error", "-i", str(out_wav), "-c:a", "libmp3lame",
          "-q:a", "1", str(out_mp3)])


def arrangement_md(score: dict) -> str:
    lines = [f"# {score['title']}", "",
             f"**{score['genre']}** · {score['key']} · {score['bpm']} bpm · "
             f"{score['time_signature']}/4 · {score['total_bars']} bars · "
             f"{int(score['duration'] // 60)}:{int(score['duration'] % 60):02d}", ""]
    if score["brief"]:
        lines += [f"> {score['brief']}", ""]
    if score["notes"]:
        lines += ["## Producer's notes", "", score["notes"], ""]
    lines += ["## Instruments", "",
              "| Track | Role | Instrument (GM) | Level | Pan | Reverb | Delay | Drive | Automation |",
              "|---|---|---|---|---|---|---|---|---|"]
    for t in score["instruments"]:
        pan = "C" if abs(t["pan"]) < 0.05 else \
            f"{'L' if t['pan'] < 0 else 'R'}{abs(round(t['pan'] * 100))}"
        auto = ", ".join(
            f"{a['param']} {a['mode']}" + (f" ({a['section']})" if a["section"] else "")
            for a in t["automation"]) or "—"
        lines.append(f"| {t['track']} | {t['role']} | {t['instrument']} | "
                     f"{t['level']:.2f} | {pan} | {t['reverb']:.2f} | {t['delay']:.2f} | "
                     f"{t['drive']:.2f} | {auto} |")
    lines += ["", "## Structure", "", "| Section | Bars | Energy | Chords | Playing | FX |",
              "|---|---|---|---|---|---|"]
    for s in score["sections"]:
        lines.append(f"| {s['name']} | {s['bars']} | {s['energy']:.2f} | "
                     f"{' · '.join(s['chord_symbols']) or '—'} | "
                     f"{', '.join(s['tracks'])} | {', '.join(s['fx']) or '—'} |")
    lines += ["", "## Files", "",
              "- `<name>.mp3` / `.wav` — the master",
              "- `<name>.mid` — full multitrack MIDI with pan/volume CCs "
              "(open it in a DAW and swap in your own plugins)",
              "- `stems/*.flac` — per-track stems, post-FX and pre-master",
              "- `score.json` — the exact plan this render came from", ""]
    if score["warnings"]:
        lines += ["## Plan repairs", ""] + [f"- {w}" for w in score["warnings"]] + [""]
    return "\n".join(lines)


def render(score: dict, out_base: Path, keep_stems: bool = True) -> dict:
    import soundfile as sf

    if np is None:
        raise RuntimeError("numpy is required — run this with lullabykit's venv python")
    if not FLUIDSYNTH.is_file():
        raise RuntimeError(f"FluidSynth not found at {FLUIDSYNTH}")
    lib = soundfont_library()
    if not lib["usable"]:
        raise RuntimeError(f"no GM soundfont under {SF_BUDGET_MB} MB in {SOUNDFONTS}")
    soundfont = Path(lib["usable"][0]["path"])

    outdir = out_base.parent
    outdir.mkdir(parents=True, exist_ok=True)
    work = outdir / "_work"
    work.mkdir(exist_ok=True)
    stems_dir = outdir / "stems"
    # a re-render can have fewer/renamed tracks than the last one — stale stems
    # would otherwise linger on disk and keep appearing in the UI
    shutil.rmtree(stems_dir, ignore_errors=True)
    for old in outdir.glob("*_polished.mp3"):
        old.unlink(missing_ok=True)
    if keep_stems:
        stems_dir.mkdir(parents=True, exist_ok=True)

    tracks = score["instruments"]
    # stem/MIDI filenames come from slugged track names, and two distinct names
    # can slug the same ("pad 1" and "pad-1"), which would overwrite one stem
    # with the other's audio — assign unique slugs up front
    slugs, used = {}, set()
    for t in tracks:
        base = _slug(t["track"])
        s, k = base, 2
        while s in used:
            s, k = f"{base}_{k}", k + 1
        used.add(s)
        slugs[t["track"]] = s

    total = len(tracks) * 2 + 4
    step = 0
    _progress(outdir, step, total, "arranging")

    tl, parts, ccs = compose(score)
    (outdir / "score.json").write_text(json.dumps(score, indent=2), encoding="utf-8")
    (outdir / "arrangement.md").write_text(arrangement_md(score), encoding="utf-8")
    step += 1
    _progress(outdir, step, total, f"{score['total_bars']} bars written — rendering parts")

    # ---- one stem per track, so each gets its own channel strip ----
    raw = {}
    for t in tracks:
        step += 1
        _progress(outdir, step, total, f"playing {t['track']} · {t['instrument']}")
        mid = work / f"{slugs[t['track']]}.mid"
        parts_to_midi(score, tl, parts, mid, only=t["track"], ccs=ccs)
        wav = work / f"{slugs[t['track']]}.wav"
        render_stem_wav(mid, wav, soundfont)
        raw[t["track"]] = wav

    tail = 4.0
    n_out = int((tl.duration + tail) * SR)
    bus = np.zeros((n_out, 2), dtype=np.float32)
    duck = kick_duck(parts, tl, score, n_out) if score["sidechain"] else None

    for t in tracks:
        step += 1
        _progress(outdir, step, total, f"mixing {t['track']} — fx & automation")
        # zlib.crc32, not hash(): str hashing is salted per process, which would
        # make an identical seed render differently on every run
        rng = np.random.default_rng(score["seed"] + zlib.crc32(t["track"].encode()) % 99991)
        chan = process_track(raw[t["track"]], t, tl, score, n_out, rng)
        if duck is not None and t["role"] in ("pad", "chords", "arp", "counter"):
            chan *= duck[:, None]
        bus += chan
        if keep_stems:
            peak = float(np.abs(chan).max()) or 1.0
            # FLAC, not WAV: lossless, every DAW reads it, and it roughly halves
            # the download — a set of WAV stems for one song runs to ~160 MB
            sf.write(stems_dir / f"{slugs[t['track']]}.flac",
                     chan / max(1.0, peak / 0.98), SR)

    step += 1
    _progress(outdir, step, total, "risers, impacts & drops")
    add_production_fx(bus, tl, score, np.random.default_rng(score["seed"] + 99))

    step += 1
    _progress(outdir, step, total, "mastering")
    peak = float(np.abs(bus).max()) or 1.0
    if peak > 0.98:
        bus = bus / peak * 0.98
    fade = int(min(3.0, tl.bar_seconds) * SR)
    bus[-fade:] *= np.linspace(1.0, 0.0, fade)[:, None]
    premix = work / "mix.wav"
    sf.write(premix, bus, SR)

    out_wav, out_mp3 = out_base.with_suffix(".wav"), out_base.with_suffix(".mp3")
    master(premix, out_wav, out_mp3)
    parts_to_midi(score, tl, parts, out_base.with_suffix(".mid"), ccs=ccs)

    step += 1
    _progress(outdir, step, total, "done")
    shutil.rmtree(work, ignore_errors=True)
    files = [out_mp3, out_wav, out_base.with_suffix(".mid"),
             outdir / "arrangement.md", outdir / "score.json"]
    if keep_stems:
        files += sorted(stems_dir.glob("*.flac"))
    return {"files": [str(f) for f in files if f.exists()],
            "duration": round(tl.duration, 1), "score": score}


def notes_payload(score: dict) -> dict:
    """The composed parts in a form the piano-roll editor can draw and send back.

    Notes are given in BEATS (not seconds) because that is what the editor's grid
    is drawn in and what survives a later tempo change. Emitted post-humanization
    so what you see is what you heard; a track you then edit is stored as
    notes_override and replays verbatim."""
    tl, parts, _ccs = compose(score)
    out = {}
    for t in score["instruments"]:
        out[t["track"]] = [[round(b, 4), round(d, 4), int(p), int(v)]
                           for b, d, p, v in sorted(parts.get(t["track"], []),
                                                    key=lambda n: (n[0], n[2]))]
    return {"bpm": score["bpm"], "beats_per_bar": tl.bpb,
            "total_beats": round(tl.total_beats, 3),
            "sections": [{"name": s["name"], "bars": s["bars"]}
                         for s in score["sections"]],
            "notes": out}


def _slug(s: str) -> str:
    return "".join(c if (c.isalnum() or c in "-_") else "_" for c in s)[:40] or "track"


# ============================================================ CLI

def main() -> int:
    ap = argparse.ArgumentParser(description=__doc__,
                                 formatter_class=argparse.RawDescriptionHelpFormatter)
    sub = ap.add_subparsers(dest="cmd", required=True)

    r = sub.add_parser("render", help="render a score plan to a finished track")
    r.add_argument("--score", type=Path, required=True, help="score JSON (any subset)")
    r.add_argument("--out", type=Path, required=True,
                   help="output base path, e.g. compositions/x/x (no extension)")
    r.add_argument("--no-stems", action="store_true")

    sub.add_parser("library", help="print the instrument/soundfont catalog as JSON")

    n = sub.add_parser("normalize", help="repair a plan and print the final score")
    n.add_argument("--score", type=Path, required=True)

    p = sub.add_parser("notes", help="print the composed notes (for the editor)")
    p.add_argument("--score", type=Path, required=True)

    args = ap.parse_args()

    if args.cmd == "library":
        print(json.dumps(soundfont_library(), indent=2))
        return 0
    raw = json.loads(args.score.read_text(encoding="utf-8"))
    score = normalize_score(raw)
    if args.cmd == "normalize":
        print(json.dumps(score, indent=2))
        return 0
    if args.cmd == "notes":
        print(json.dumps(notes_payload(score)))
        return 0

    t0 = time.time()
    result = render(score, args.out, keep_stems=not args.no_stems)
    result["seconds"] = round(time.time() - t0, 1)
    print(json.dumps(result))
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## File 4 of 27 — `%USERPROFILE%\local-ai-studio\sf3convert.py`

```python
#!/usr/bin/env python3
"""sf3convert: shrink a SoundFont by storing its samples as Ogg Vorbis (SF2 -> SF3).

Why this exists: the Salamander Grand Piano ships as a 1.2 GB .sf2 whose sample
chunk is 100% uncompressed 16-bit PCM — 960 individual samples (16 velocity
layers x 30 stereo key zones). Nothing about the *instrument* is 1.2 GB; the
format simply has no compression. SF3 is the same SoundFont container with each
sample replaced by an Ogg Vorbis stream, and the bundled FluidSynth 2.4.7 reads
it (probed: renders at 0.9963 waveform correlation against the raw PCM original,
at ~16% of the size).

What changes, per the SF3 convention FluidSynth implements:
  * the `smpl` chunk becomes concatenated Ogg Vorbis streams
  * each `shdr` record's dwStart/dwEnd become BYTE offsets into that chunk
    (they are sample-frame indices in SF2)
  * loop points become frame offsets RELATIVE to the sample start
  * sfSampleType gets bit 0x10 set, marking the sample compressed
  * the `ifil` version chunk is bumped to 3.x
Every other chunk is copied through byte-for-byte.

The source file is never modified — the SF3 is written alongside it.

STATUS — NOT YET WORKING, kept because it is close and the win is large.
Converting Salamander produces a 94.3 MB font (7.8% of 1207.8 MB) in 43s that
loads, decays correctly, keeps its stereo image, and correlates 0.99 against the
original on a single note. But a per-note sweep shows its peak level depends
only on VELOCITY and not on pitch (0.049 / 0.197 / 0.442 at vel 40/80/120,
identical for notes 36 through 84), i.e. every key ends up playing the same
sample data. Something about the rewritten dwStart/dwEnd byte offsets or the
loop-point convention is not what FluidSynth's SF3 reader expects. Do not ship
output from this until verify() passes — see the note in verify() about why a
correlation check alone is not enough to catch this.

Usage:
  python sf3convert.py in.sf2 [--out out.sf3] [--quality 0.6] [--verify]
"""
from __future__ import annotations

import argparse
import io
import struct
import sys
import time
from pathlib import Path

import numpy as np
import soundfile as sf

VORBIS_FLAG = 0x10       # sfSampleType bit: this sample is Ogg Vorbis
SHDR_LEN = 46            # bytes per sample header record
WRITE_BLOCK = 65536      # frames per write — see the note in convert()


def _chunks(data: bytes, start: int, end: int):
    """Walk the RIFF chunks in [start, end) -> (tag, payload_start, payload_len)."""
    i = start
    while i + 8 <= end:
        tag = data[i:i + 4]
        n = struct.unpack("<I", data[i + 4:i + 8])[0]
        yield tag, i + 8, n
        i += 8 + n + (n & 1)          # chunks are word-aligned


def _find(data: bytes):
    """Locate the sdta/pdta lists and the chunks we need to rewrite."""
    if data[:4] != b"RIFF" or data[8:12] != b"sfbk":
        raise SystemExit("not a SoundFont (missing RIFF/sfbk header)")
    total = struct.unpack("<I", data[4:8])[0]
    found = {}
    for tag, off, n in _chunks(data, 12, min(len(data), 8 + total)):
        if tag == b"LIST":
            kind = data[off:off + 4]
            found[kind] = (off + 4, n - 4)
            for sub, soff, sn in _chunks(data, off + 4, off + n):
                found[sub] = (soff, sn)
        else:
            found[tag] = (off, n)
    for need in (b"smpl", b"shdr", b"ifil"):
        if need not in found:
            raise SystemExit(f"SoundFont is missing its {need.decode()} chunk")
    return found


def _pad(b: bytes) -> bytes:
    return b + (b"\0" if len(b) % 2 else b"")


def _chunk(tag: bytes, payload: bytes) -> bytes:
    return tag + struct.pack("<I", len(payload)) + _pad(payload)


def convert(src: Path, dst: Path, quality: float = 0.6,
            progress_every: int = 60) -> dict:
    data = src.read_bytes()
    at = _find(data)
    smpl_off, smpl_len = at[b"smpl"]
    shdr_off, shdr_len = at[b"shdr"]
    n_rec = shdr_len // SHDR_LEN

    pcm = np.frombuffer(data, dtype="<i2", count=smpl_len // 2, offset=smpl_off)

    new_smpl = io.BytesIO()
    new_shdr = bytearray()
    t0 = time.time()
    converted = 0

    for k in range(n_rec):
        rec = bytearray(data[shdr_off + SHDR_LEN * k:shdr_off + SHDR_LEN * (k + 1)])
        name = bytes(rec[:20]).split(b"\0")[0]
        start, end, loop_s, loop_e, rate = struct.unpack("<IIIII", rec[20:40])
        orig_pitch, pitch_corr, link, stype = struct.unpack("<BbHH", rec[40:46])

        # the terminal record (and anything degenerate) is copied as-is
        if name == b"EOS" or end <= start or k == n_rec - 1:
            new_shdr += rec
            continue

        frames = pcm[start:end]
        if not len(frames):
            new_shdr += rec
            continue

        buf = io.BytesIO()
        with sf.SoundFile(buf, "w", samplerate=int(rate) or 44100, channels=1,
                          format="OGG", subtype="VORBIS") as f:
            # Write in blocks. Handing libsndfile's Vorbis encoder a single
            # million-frame buffer crashes the process outright (no exception —
            # the interpreter dies), and Salamander's samples are ~21 s each.
            fl = frames.astype(np.float32) / 32768.0
            for i in range(0, len(fl), WRITE_BLOCK):
                f.write(fl[i:i + WRITE_BLOCK])
        ogg = buf.getvalue()

        byte_start = new_smpl.tell()
        new_smpl.write(ogg)
        byte_end = new_smpl.tell()

        struct.pack_into("<IIIII", rec, 20, byte_start, byte_end,
                         max(0, loop_s - start), max(0, loop_e - start), rate)
        struct.pack_into("<BbHH", rec, 40, orig_pitch, pitch_corr, link,
                         stype | VORBIS_FLAG)
        new_shdr += rec
        converted += 1
        if progress_every and converted % progress_every == 0:
            done = new_smpl.tell()
            print(f"   {converted}/{n_rec - 1} samples · "
                  f"{done / 1048576:.0f} MB written · {time.time() - t0:.0f}s",
                  flush=True)

    smpl_new = new_smpl.getvalue()

    # rebuild: sdta gets the compressed smpl, pdta gets the rewritten shdr,
    # everything else is passed through untouched
    out_info = bytearray(b"INFO")
    for tag, off, n in _chunks(data, *(at[b"INFO"][0], at[b"INFO"][0] + at[b"INFO"][1])):
        payload = data[off:off + n]
        if tag == b"ifil":
            payload = struct.pack("<HH", 3, 0)       # declare SF3
        out_info += _chunk(tag, payload)

    out_pdta = bytearray(b"pdta")
    for tag, off, n in _chunks(data, *(at[b"pdta"][0], at[b"pdta"][0] + at[b"pdta"][1])):
        payload = bytes(new_shdr) if tag == b"shdr" else data[off:off + n]
        out_pdta += _chunk(tag, payload)

    body = (b"sfbk"
            + _chunk(b"LIST", bytes(out_info))
            + _chunk(b"LIST", b"sdta" + _chunk(b"smpl", smpl_new))
            + _chunk(b"LIST", bytes(out_pdta)))
    dst.write_bytes(b"RIFF" + struct.pack("<I", len(body)) + body)

    return {"samples": converted,
            "src_mb": round(len(data) / 1048576, 1),
            "dst_mb": round(dst.stat().st_size / 1048576, 1),
            "seconds": round(time.time() - t0, 1)}


def _render_note(font: Path, fluidsynth: Path, work: Path, note: int, vel: int):
    import mido
    import subprocess
    mid = work / "n.mid"
    m = mido.MidiFile(ticks_per_beat=480)
    tr = mido.MidiTrack()
    m.tracks.append(tr)
    tr.append(mido.Message("program_change", program=0, channel=0, time=0))
    tr.append(mido.Message("note_on", note=note, velocity=vel, channel=0, time=0))
    tr.append(mido.Message("note_off", note=note, velocity=0, channel=0, time=480 * 3))
    m.save(str(mid))
    wav = work / "n.wav"
    r = subprocess.run([str(fluidsynth), "-ni", "-F", str(wav), "-r", "48000",
                        "-g", "0.7", "-o", "synth.reverb.active=0",
                        "-o", "synth.chorus.active=0", str(font), str(mid)],
                       capture_output=True, text=True)
    if r.returncode != 0 or not wav.is_file():
        return None
    a, _ = sf.read(str(wav), always_2d=True)
    return a.mean(axis=1)


def verify(sf2: Path, sf3: Path, fluidsynth: Path, work: Path) -> dict:
    """Prove the converted font still maps NOTE -> SAMPLE correctly.

    A whole-song correlation is not enough: an earlier version of this check
    passed a font whose every key played the same sample, because the two
    renders still correlated at 0.94 overall. The tell was that the SF3's peak
    level depended only on velocity and not at all on pitch. So the real test
    sweeps notes and asks whether the per-note level PATTERN survives — if the
    converted font's levels no longer vary with pitch the way the original's do,
    the sample offsets are wrong no matter how good the correlation looks."""
    work.mkdir(parents=True, exist_ok=True)
    notes, vel = (36, 48, 60, 72, 84), 80
    peaks = {"sf2": [], "sf3": []}
    corrs = []
    for note in notes:
        a = _render_note(sf2, fluidsynth, work, note, vel)
        b = _render_note(sf3, fluidsynth, work, note, vel)
        if a is None or b is None or len(a) < 1000 or len(b) < 1000:
            return {"ok": False, "error": f"note {note} failed to render"}
        peaks["sf2"].append(float(np.abs(a).max()))
        peaks["sf3"].append(float(np.abs(b).max()))
        n = min(len(a), len(b))
        corrs.append(float(np.corrcoef(a[:n], b[:n])[0, 1]))

    p2, p3 = np.array(peaks["sf2"]), np.array(peaks["sf3"])
    # relative variation of level across pitch, in each font
    spread2 = float(p2.std() / max(p2.mean(), 1e-9))
    spread3 = float(p3.std() / max(p3.mean(), 1e-9))
    pitch_ok = spread3 > 0.35 * spread2          # SF3 must still vary with pitch
    corr_ok = min(corrs) > 0.9
    return {"ok": bool(pitch_ok and corr_ok),
            "min_correlation": round(min(corrs), 4),
            "pitch_level_spread_sf2": round(spread2, 4),
            "pitch_level_spread_sf3": round(spread3, 4),
            "pitch_mapping_ok": bool(pitch_ok),
            "peaks_sf2": [round(x, 5) for x in p2],
            "peaks_sf3": [round(x, 5) for x in p3]}


def main() -> int:
    ap = argparse.ArgumentParser(description=__doc__,
                                 formatter_class=argparse.RawDescriptionHelpFormatter)
    ap.add_argument("src", type=Path)
    ap.add_argument("--out", type=Path, default=None)
    ap.add_argument("--quality", type=float, default=0.6)
    ap.add_argument("--verify", action="store_true",
                    help="render both fonts and correlate (recommended)")
    args = ap.parse_args()

    src = args.src
    if not src.is_file():
        raise SystemExit(f"not found: {src}")
    dst = args.out or src.with_suffix(".sf3")
    print(f"converting {src.name} -> {dst.name}", flush=True)
    info = convert(src, dst, quality=args.quality)
    print(f"   {info['samples']} samples · {info['src_mb']} MB -> {info['dst_mb']} MB "
          f"({info['dst_mb'] / max(info['src_mb'], 1e-9):.1%}) in {info['seconds']}s",
          flush=True)

    if args.verify:
        fs = Path(__file__).resolve().parent / "lullabykit" / "bin" / "fluidsynth" / "bin" / "fluidsynth.exe"
        v = verify(src, dst, fs, dst.parent / "_sf3verify")
        print(f"   verify: {v}", flush=True)
        if not v.get("ok"):
            print("   FAILED — leaving the original in place; delete the .sf3",
                  file=sys.stderr)
            return 1
    print(f"done: {dst}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## File 5 of 27 — `%USERPROFILE%\local-ai-studio\lullabykit\pipeline.py`

```python
"""
lullabykit: turn a song into a soft piano + music-box lullaby instrumental.

v2 — musical arrangement engine. The v1 approach (transcribe the whole mix,
top-note-is-melody, pitch-class chord clusters) produced random-sounding notes;
v2 fixes that with actual music analysis:

  analysis   : beat grid + bar phase, key detection (Krumhansl-Schmuckler),
               per-bar diatonic chord recognition (chroma template matching with
               a bass-stem root prior)
  melody     : transcribed from the ISOLATED VOCAL stem only (basic-pitch),
               then cleaned — monophony enforcement, octave-jump repair,
               scale snapping, eighth-note grid quantization, phrase-shaped
               velocities (falls back to the 'other' stem for instrumentals)
  arrangement: rebuilt from scratch on a clean grid at a true lullaby tempo
               (55-88 bpm): rocking broken-chord piano left hand, legato melody
               right hand, music-box doubling an octave up, optional soft string
               pad, sustain pedal per bar
  sound      : piano rendered with the Salamander Grand (real sampled Yamaha C5)
               when present, layers with FluidR3 GM; large-room FluidSynth
               reverb; warm mastering chain (shelf EQ, gentle compression,
               loudness normalization, long fades)

melody-match engine (--engine melody-match) — an alternative to the above for
when the arranged engine's scale-snap + eighth-grid quantization mangles a
florid or heavily-ornamented vocal line into something unrecognizable. Traces
the singer's actual continuous pitch curve with FCPE (10ms resolution, no
scale/grid quantization) and plays it back on a single portamento-capable
instrument via per-frame MIDI pitch-bend + expression. Validated by
round-tripping FCPE over the rendered output and correlating against the
original vocal's F0: 0.94-0.99 correlation across two real songs, vs. the
arranged engine's -0.03 to ~0.6 on material it struggles with.

Usage:
  .venv\\Scripts\\python.exe pipeline.py "song.mp3" [--tempo-scale 0.72]
      [--out output\\name] [--from-stage arrange] [--no-celesta] [--no-pad]
  .venv\\Scripts\\python.exe pipeline.py "song.mp3" --engine melody-match
      --continuous-instrument cello [--melody-stems '{"vocals":1}']
"""

import argparse
import glob
import json
import os
import shutil
import subprocess
import sys
from pathlib import Path

import numpy as np
import pretty_midi
import mido

ROOT = Path(__file__).resolve().parent
FLUIDSYNTH = ROOT / "bin" / "fluidsynth" / "bin" / "fluidsynth.exe"
SF_FLUIDR3 = ROOT / "soundfonts" / "FluidR3_GM.sf2"
WORK = ROOT / "work"
OUTPUT = ROOT / "output"

PROG_PIANO = 0        # GM Acoustic Grand
PROG_MUSIC_BOX = 10   # GM Music Box
PROG_STRINGS = 48     # GM String Ensemble 1

# melody-match engine: portamento-capable instruments, picked for how closely
# they can trace a continuous vocal pitch curve (a fixed-pitch instrument like
# piano can't glide between notes at all). Cello is the default — warm, clearly
# instrumental (not uncanny like a "voice" patch), and scored joint-best with
# synth_voice in round-trip pitch-fidelity testing across two real songs.
CONTINUOUS_INSTRUMENTS = {
    "cello": 42, "violin": 40, "flute": 73, "synth_voice": 54,
    "music_box": PROG_MUSIC_BOX,
}
# each note's own base pitch already sits at its true semitone (_segment_notes
# anchors it), so the residual bend only needs to cover vibrato/expression —
# not a whole phrase's range like an earlier, buggier version of this engine
# that used one note per phrase and bent through the entire tune (sounded
# like a siren/theremin wail, not a melody). Narrower range = finer per-cent
# resolution from the same 14-bit MIDI bend value.
BEND_RANGE_SEMITONES = 4

# scale degrees (semitones from tonic) of the diatonic triads we allow
MAJOR_SCALE = [0, 2, 4, 5, 7, 9, 11]
MINOR_SCALE = [0, 2, 3, 5, 7, 8, 10]
MAJOR_TRIADS = [(0, "maj"), (2, "min"), (4, "min"), (5, "maj"), (7, "maj"), (9, "min")]
MINOR_TRIADS = [(0, "min"), (3, "maj"), (5, "min"), (7, "min"), (8, "maj"), (10, "maj")]

# Krumhansl-Schmuckler key profiles
KS_MAJOR = np.array([6.35, 2.23, 3.48, 2.33, 4.38, 4.09, 2.52, 5.19, 2.39, 3.66, 2.29, 2.88])
KS_MINOR = np.array([6.33, 2.68, 3.52, 5.38, 2.60, 3.53, 2.54, 4.75, 3.98, 2.69, 3.34, 3.17])

NOTE_NAMES = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"]


def _ffmpeg_exe(name: str = "ffmpeg") -> str:
    found = shutil.which(name)
    if found:
        return found
    hits = glob.glob(os.path.join(os.environ.get("LOCALAPPDATA", ""), "Microsoft",
                     "WinGet", "Packages", "Gyan.FFmpeg*", "**", f"{name}.exe"),
                     recursive=True)
    if hits:
        return hits[0]
    raise RuntimeError(f"{name} not found — install it: winget install Gyan.FFmpeg")


def piano_soundfont() -> Path:
    """Salamander Grand if installed (much better piano), else FluidR3 GM."""
    for hit in ROOT.joinpath("soundfonts").glob("**/*.sf2"):
        if "salamander" in hit.name.lower():
            return hit
    return SF_FLUIDR3


def run(cmd, **kw):
    print(f"$ {' '.join(str(c) for c in cmd)}", flush=True)
    subprocess.run(cmd, check=True, **kw)


# ---------------------------------------------------------------- stages 1-3

def extract_audio(input_path: Path, out_wav: Path, sr: int = 44100) -> Path:
    run([_ffmpeg_exe(), "-y", "-i", str(input_path),
         "-vn", "-ac", "2", "-ar", str(sr), str(out_wav)])
    return out_wav


# htdemucs_ft: Demucs's own recommended production model for vocals/drums/
# bass/other — a 4-model ensemble ("fine-tuned"), ~4x slower than the base
# model but no cross-source bleed. htdemucs_6s (the old default here) adds
# guitar/piano stems but Demucs's own docs call it out as NOT production
# quality: its piano/guitar separation heads bleed into and steal energy from
# the other stems — the most likely cause of a sustained instrument (sax,
# brass, strings, anything living in "other") intermittently dropping out.
# So: htdemucs_ft supplies vocals/drums/bass/other (the stems that matter for
# every song), htdemucs_6s runs only as a second pass to ALSO offer guitar/
# piano as extra, best-effort tracks — its lower-quality other/vocals/drums/
# bass from that pass are discarded.
DEMUCS_MODEL = "htdemucs_ft"
DEMUCS_MODEL_EXTRA = "htdemucs_6s"
STEM_NAMES = ["vocals", "guitar", "piano", "other", "bass", "drums"]
# what carries into the lullaby by default when the user hasn't chosen
DEFAULT_STEM_WEIGHTS = {"vocals": 1.0, "guitar": 1.0, "piano": 1.0, "other": 1.0,
                        "bass": 0.0, "drums": 0.0}


def separate_stems(wav_path: Path, stems_dir: Path) -> Path:
    stems_dir.mkdir(parents=True, exist_ok=True)
    run([sys.executable, "-m", "demucs.separate", "-n", DEMUCS_MODEL,
         "-o", str(stems_dir), str(wav_path)])
    out_dir = stems_dir / DEMUCS_MODEL / wav_path.stem

    run([sys.executable, "-m", "demucs.separate", "-n", DEMUCS_MODEL_EXTRA,
         "-o", str(stems_dir), str(wav_path)])
    extra_dir = stems_dir / DEMUCS_MODEL_EXTRA / wav_path.stem
    for name in ("guitar", "piano"):
        src = extra_dir / f"{name}.wav"
        if src.is_file():
            shutil.copyfile(src, out_dir / f"{name}.wav")
    shutil.rmtree(stems_dir / DEMUCS_MODEL_EXTRA, ignore_errors=True)
    return out_dir


REMIX_FOCUS_WEIGHTS = {
    # legacy focus presets (the multitrack workbench passes explicit weights)
    "both": DEFAULT_STEM_WEIGHTS,
    "vocals": {**DEFAULT_STEM_WEIGHTS, "guitar": 0.35, "piano": 0.35, "other": 0.35},
    "instruments": {**DEFAULT_STEM_WEIGHTS, "vocals": 0.35},
}


def prep_remix_input(stem_dir: Path, out_wav: Path, focus: str = "both",
                     stem_weights: dict = None) -> Path:
    """Build the cleanest possible input for an ACE-Step lullaby remix from the
    USER-CHOSEN stems at their chosen levels (multitrack workbench), then: HPSS
    harmonic component only (drops riser sweeps / cymbal swells / noise beds
    that the model renders as ugly artifacts in buildups), de-vocalize the
    vocal stem if present, tail trimmed at the last real musical energy with a
    gentle fade so the model never sees the messy outro."""
    import librosa
    import soundfile as sf

    from scipy.signal import butter, sosfiltfilt

    weights = dict(stem_weights) if stem_weights else \
        dict(REMIX_FOCUS_WEIGHTS.get(focus, REMIX_FOCUS_WEIGHTS["both"]))
    weights = {k: min(1.5, max(0.0, float(v))) for k, v in weights.items()
               if k in STEM_NAMES}
    if not any(v > 0 for v in weights.values()):
        raise RuntimeError("no stems selected — include at least one track")
    print("   remix stems: " + ", ".join(f"{k}={v:g}" for k, v in weights.items()
                                         if v > 0), flush=True)
    sr = 44100
    chans = []
    for name, w in weights.items():
        if w <= 0:
            continue
        audio, file_sr = sf.read(stem_dir / f"{name}.wav", always_2d=True)
        if file_sr != sr:
            raise RuntimeError(f"unexpected sample rate in {name}.wav")
        audio = audio.astype(np.float32)
        if name == "vocals":
            # de-vocalize: keep the sung melody (fundamentals < ~1kHz) but strip
            # the speech cues (consonants / sibilance / formant edge > ~3kHz)
            # that survive ACE's regeneration as ghost-vocal artifacts — the
            # stem becomes a soft melodic pad instead of a recognizable voice
            sos = butter(4, 3200, btype="low", fs=sr, output="sos")
            audio = sosfiltfilt(sos, audio, axis=0).astype(np.float32)
        chans.append(audio * w)
    n = max(a.shape[0] for a in chans)
    mix = np.zeros((n, 2), dtype=np.float32)
    for a in chans:
        mix[: a.shape[0]] += a[:, :2] if a.shape[1] >= 2 else np.repeat(a, 2, axis=1)

    # per-channel HPSS, keep harmonic only — ADAPTIVELY. Songs whose identity is
    # transient (rapid patter vocals, staccato stabs) read as "percussive" to
    # HPSS; a strict filter throws their melody away (measured: 23% energy
    # retention on ensemble musical theater -> output stopped resembling the
    # song). Relax the margin, then blend the mix back in, until enough of the
    # song survives.
    def hpss(margin):
        out = np.zeros_like(mix)
        for c in range(2):
            out[:, c] = librosa.effects.harmonic(np.ascontiguousarray(mix[:, c]),
                                                 margin=margin)
        return out

    e_mix = float(np.mean(mix ** 2)) + 1e-12
    harm = hpss(3.0)
    retention = float(np.mean(harm ** 2)) / e_mix
    if retention < 0.45:
        harm = hpss(1.5)
        retention = float(np.mean(harm ** 2)) / e_mix
        print(f"   transient-heavy song: relaxed HPSS (retention {retention:.0%})", flush=True)
    if retention < 0.45:
        harm = 0.5 * harm + 0.5 * mix
        retention = float(np.mean(harm ** 2)) / e_mix
        print(f"   still thin: blended original back (retention {retention:.0%})", flush=True)

    # tail trim: last 0.5s window with real energy, +1.5s of room, 2.5s fade
    win = sr // 2
    mono = np.abs(harm).mean(axis=1)
    rms = np.array([np.sqrt(np.mean(mono[i:i + win] ** 2)) for i in range(0, len(mono), win)])
    active = np.where(rms > 0.004)[0]
    if len(active):
        end = min(len(harm), (active[-1] + 1) * win + int(1.5 * sr))
        harm = harm[:end]
        fade = min(int(2.5 * sr), len(harm))
        harm[-fade:] *= np.linspace(1.0, 0.0, fade)[:, None]

    peak = np.max(np.abs(harm)) or 1.0
    if peak > 0.98:
        harm = harm / peak * 0.98
    sf.write(out_wav, harm, sr)
    print(f"   remix input: harmonic-only, {len(harm)/sr:.0f}s", flush=True)
    return out_wav


def mix_non_drum_stems(stem_dir: Path, out_wav: Path) -> Path:
    import soundfile as sf
    parts, sr_ref = [], None
    for name in (n for n in STEM_NAMES if n != "drums"):
        audio, sr = sf.read(stem_dir / f"{name}.wav", always_2d=True)
        sr_ref = sr_ref or sr
        parts.append(audio)
    n = max(p.shape[0] for p in parts)
    mix = np.zeros((n, parts[0].shape[1]), dtype=np.float32)
    for p in parts:
        mix[: p.shape[0]] += p.astype(np.float32)
    peak = np.max(np.abs(mix)) or 1.0
    if peak > 0.98:
        mix = mix / peak * 0.98
    sf.write(out_wav, mix, sr_ref)
    return out_wav


# ---------------------------------------------------------------- stage 4: analysis + transcription

def detect_key(chroma_mean: np.ndarray) -> tuple:
    """(tonic_pc, 'major'|'minor') by Krumhansl-Schmuckler correlation."""
    best = (-2.0, 0, "major")
    for tonic in range(12):
        rolled = np.roll(chroma_mean, -tonic)
        for profile, mode in ((KS_MAJOR, "major"), (KS_MINOR, "minor")):
            r = np.corrcoef(rolled, profile)[0, 1]
            if r > best[0]:
                best = (r, tonic, mode)
    return best[1], best[2]


def analyze(raw_wav: Path, stem_dir: Path, out_json: Path) -> dict:
    """Beat grid, bar phase, key, and per-bar chords. Saved as JSON so the
    arrange stage can be re-run instantly while tuning."""
    import librosa

    y_mix, sr = librosa.load(str(raw_wav), sr=22050, mono=True)
    y_harm = librosa.effects.harmonic(y_mix, margin=4)

    tempo, beat_frames = librosa.beat.beat_track(y=y_mix, sr=sr, trim=False)
    tempo = float(np.atleast_1d(tempo)[0])
    beat_times = librosa.frames_to_time(beat_frames, sr=sr).tolist()
    # normalize half/double-time lock-ins — the BEAT GRID must move with the
    # tempo number, or every downstream duration is off by 2x (a 152->76
    # halving without decimating beats once produced a 14-minute lullaby)
    while tempo > 150 and len(beat_times) > 8:
        tempo /= 2
        beat_times = beat_times[::2]
    while tempo < 60:
        tempo *= 2
        mids = [(a + b) / 2 for a, b in zip(beat_times, beat_times[1:])]
        merged = []
        for b, m in zip(beat_times, mids + [None]):
            merged.append(b)
            if m is not None:
                merged.append(m)
        beat_times = merged
    if len(beat_times) < 8:
        raise RuntimeError("could not find a beat grid in this track")

    chroma = librosa.feature.chroma_cqt(y=y_harm, sr=sr)
    key_pc, key_mode = detect_key(chroma.mean(axis=1))

    # per-beat mean chroma
    frames = librosa.time_to_frames(beat_times, sr=sr)
    beat_chroma = []
    for i in range(len(frames) - 1):
        seg = chroma[:, frames[i]:max(frames[i] + 1, frames[i + 1])]
        beat_chroma.append(seg.mean(axis=1))
    beat_chroma.append(beat_chroma[-1])
    beat_chroma = np.array(beat_chroma)

    # bar phase (4/4 assumed): the phase whose bar boundaries see the most
    # harmonic change is where the chords move
    def phase_score(ph):
        idx = list(range(ph, len(beat_chroma) - 4, 4))
        return sum(float(np.linalg.norm(beat_chroma[i + 4] - beat_chroma[i])) for i in idx) / max(len(idx), 1)
    phase = int(np.argmax([phase_score(p) for p in range(4)]))

    # bass-stem chroma as a root prior
    y_bass, _ = librosa.load(str(stem_dir / "bass.wav"), sr=22050, mono=True)
    bass_chroma = librosa.feature.chroma_cqt(y=y_bass, sr=22050)

    triads = MAJOR_TRIADS if key_mode == "major" else MINOR_TRIADS
    candidates = []
    for deg, qual in triads:
        root = (key_pc + deg) % 12
        third = (root + (4 if qual == "maj" else 3)) % 12
        fifth = (root + 7) % 12
        tpl = np.zeros(12)
        tpl[root], tpl[third], tpl[fifth] = 1.0, 0.8, 0.9
        candidates.append({"root": root, "qual": qual, "tpl": tpl})

    bars = []
    prev_pick = None
    for b0 in range(phase, len(beat_times) - 1, 4):
        t0 = beat_times[b0]
        t1 = beat_times[min(b0 + 4, len(beat_times) - 1)]
        seg = beat_chroma[b0:min(b0 + 4, len(beat_chroma))].mean(axis=0)
        f0, f1 = librosa.time_to_frames([t0, t1], sr=22050)
        bseg = bass_chroma[:, f0:max(f0 + 1, f1)].mean(axis=1)
        bseg = bseg / (bseg.max() + 1e-9)
        scores = [float(seg @ c["tpl"] + 0.6 * bseg[c["root"]]) for c in candidates]
        pick = int(np.argmax(scores))
        # hysteresis: stay on the previous chord unless clearly beaten
        if prev_pick is not None and scores[pick] < scores[prev_pick] * 1.08:
            pick = prev_pick
        prev_pick = pick
        bars.append({"start": t0, "end": t1,
                     "root": candidates[pick]["root"], "qual": candidates[pick]["qual"]})

    info = {"tempo": tempo, "beat_times": beat_times, "bar_phase": phase,
            "key_pc": key_pc, "key_mode": key_mode, "bars": bars}
    out_json.write_text(json.dumps(info))
    print(f"   key: {NOTE_NAMES[key_pc]} {key_mode}   tempo: {tempo:.0f} bpm   "
          f"bars: {len(bars)}", flush=True)
    return info


def _transcribe_stem(wav: Path) -> "pretty_midi.PrettyMIDI":
    from basic_pitch.inference import predict
    from basic_pitch import ICASSP_2022_MODEL_PATH

    _, midi_data, _ = predict(
        str(wav),
        model_or_model_path=ICASSP_2022_MODEL_PATH,
        onset_threshold=0.55,
        frame_threshold=0.35,
        minimum_note_length=120,
        melodia_trick=True,
    )
    return midi_data


def _melody_score(midi_data) -> float:
    """How much does this transcription look like a LEAD MELODY (vs accompaniment
    chords, ad-libs, or silence)? Higher is better.

    A real melody: covers a good share of the song, is mostly one note at a
    time, moves stepwise more than it leaps, holds notes rather than stabbing,
    and lives in a sane pitch band."""
    notes = sorted((n for inst in midi_data.instruments if not inst.is_drum
                    for n in inst.notes if 43 <= n.pitch <= 84),
                   key=lambda n: n.start)
    if len(notes) < 24:
        return 0.0
    song_end = max(n.end for n in notes)

    # coverage: merged voiced time / duration
    merged, cur = [], [notes[0].start, notes[0].end]
    for n in notes[1:]:
        if n.start <= cur[1] + 0.1:
            cur[1] = max(cur[1], n.end)
        else:
            merged.append(cur)
            cur = [n.start, n.end]
    merged.append(cur)
    coverage = sum(e - s for s, e in merged) / max(song_end, 1e-6)

    # monophony: fraction of notes that don't overlap their successor
    overlaps = sum(1 for a, b in zip(notes, notes[1:]) if b.start < a.end - 0.06)
    mono = 1.0 - overlaps / len(notes)

    # stepwise motion on the top-line reduction
    top, last_start = [], -1.0
    for n in notes:
        if n.start - last_start > 0.06:
            top.append(n.pitch)
            last_start = n.start
        elif top and n.pitch > top[-1]:
            top[-1] = n.pitch
    steps = [abs(b - a) for a, b in zip(top, top[1:])]
    stepwise = (sum(1 for s in steps if s <= 2) / len(steps)) if steps else 0.0

    med_dur = float(np.median([n.end - n.start for n in notes]))
    iqr = float(np.percentile([n.pitch for n in notes], 75)
                - np.percentile([n.pitch for n in notes], 25))

    return (2.0 * min(coverage, 0.9)
            + 1.5 * mono
            + 1.5 * stepwise
            + 1.0 * min(med_dur / 0.30, 1.0)
            + (1.0 if 4 <= iqr <= 14 else 0.4))


def _mix_stems(stem_dir: Path, names: tuple, out_path: Path) -> Path:
    import soundfile as sf
    parts, sr_ref = [], None
    for n in names:
        audio, sr = sf.read(stem_dir / f"{n}.wav", always_2d=True)
        sr_ref = sr_ref or sr
        parts.append(audio.astype(np.float32))
    ln = max(p.shape[0] for p in parts)
    mix = np.zeros((ln, parts[0].shape[1]), dtype=np.float32)
    for p in parts:
        mix[: p.shape[0]] += p
    peak = np.max(np.abs(mix)) or 1.0
    if peak > 0.98:
        mix = mix / peak * 0.98
    sf.write(out_path, mix, sr_ref)
    return out_path


def _mix_stems_weighted(stem_dir: Path, weights: dict, out_path: Path) -> Path:
    import soundfile as sf
    parts, sr_ref = [], None
    for name, w in weights.items():
        if w <= 0:
            continue
        audio, sr = sf.read(stem_dir / f"{name}.wav", always_2d=True)
        sr_ref = sr_ref or sr
        parts.append(audio.astype(np.float32) * w)
    if not parts:
        raise RuntimeError("no stems selected for melody — tick at least one track")
    ln = max(p.shape[0] for p in parts)
    mix = np.zeros((ln, parts[0].shape[1]), dtype=np.float32)
    for p in parts:
        mix[: p.shape[0]] += p
    peak = np.max(np.abs(mix)) or 1.0
    if peak > 0.98:
        mix = mix / peak * 0.98
    sf.write(out_path, mix, sr_ref)
    return out_path


def transcribe_melody(stem_dir: Path, out_midi: Path, source: str = "auto",
                      stem_weights: dict = None) -> Path:
    """Transcribe the lead melody.

    stem_weights (the multitrack workbench path — the normal case now): a
    {stem: level} dict straight from the Tracks panel. Whatever the user
    ticked/weighted there IS the melody source, transcribed directly. No
    further picking, no candidates, no auto-scoring.

    source (legacy path, used only when stem_weights is None — plain CLI use
    without a workbench selection): 'vocals' | 'instruments' | 'vocals_other'
    | 'auto', where auto scores each against a melody-likeness heuristic and
    keeps the best."""
    work = out_midi.parent

    if stem_weights:
        weights = {k: v for k, v in stem_weights.items() if k in STEM_NAMES and v > 0}
        print("   melody stems: " + ", ".join(f"{k}={v:g}" for k, v in weights.items()),
              flush=True)
        mix_path = _mix_stems_weighted(stem_dir, weights, work / "melody_mix.wav")
        midi_data = _transcribe_stem(mix_path)
        midi_data.write(str(out_midi))
        return out_midi

    candidates = {
        "vocals": stem_dir / "vocals.wav",
        "instruments": _mix_stems(stem_dir, ("guitar", "piano", "other"),
                                  work / "instruments_bus.wav"),
        "vocals_other": _mix_stems(stem_dir, ("vocals", "other"),
                                   work / "vocals_other_bus.wav"),
    }
    if source in candidates:
        midi_data = _transcribe_stem(candidates[source])
        print(f"   melody source: {source} (manual)", flush=True)
        midi_data.write(str(out_midi))
        return out_midi

    scored = {}
    for name, wav in candidates.items():
        md = _transcribe_stem(wav)
        scored[name] = (_melody_score(md), md)
        print(f"   melody candidate {name}: score {scored[name][0]:.2f}", flush=True)
    pick = max(scored, key=lambda k: scored[k][0])
    if scored[pick][0] <= 0.0:
        raise RuntimeError("no melody found in vocals/instruments/vocals_other")
    print(f"   melody source: {pick} (auto)", flush=True)
    scored[pick][1].write(str(out_midi))
    return out_midi


def write_stem_previews(stem_dir: Path, work: Path, buckets: int = 200):
    """Multitrack workbench assets: per-stem audition mp3 + peak envelope."""
    import soundfile as sf
    info = {}
    for name in STEM_NAMES:
        wav = stem_dir / f"{name}.wav"
        if not wav.is_file():
            continue
        run([_ffmpeg_exe(), "-y", "-v", "quiet", "-i", str(wav),
             "-c:a", "libmp3lame", "-b:a", "128k",
             str(work / f"stem_{name}_preview.mp3")])
        audio, sr = sf.read(wav, always_2d=True)
        mono = np.abs(audio).mean(axis=1)
        step = max(1, len(mono) // buckets)
        peaks = [round(float(mono[i:i + step].max()), 4)
                 for i in range(0, len(mono) - 1, step)][:buckets]
        info[name] = {"peaks": peaks,
                      "rms": round(float(np.sqrt(np.mean(mono ** 2))), 5)}
    (work / "stems.json").write_text(json.dumps(info))
    print(f"   stem previews: {', '.join(info)}", flush=True)


def write_waveform_json(raw_wav: Path, out_json: Path, buckets: int = 600):
    """Peak-per-bucket envelope of the song for the workbench waveform strip."""
    import soundfile as sf
    audio, sr = sf.read(raw_wav, always_2d=True)
    mono = np.abs(audio).mean(axis=1)
    step = max(1, len(mono) // buckets)
    peaks = [round(float(mono[i:i + step].max()), 4)
             for i in range(0, len(mono) - 1, step)][:buckets]
    out_json.write_text(json.dumps({"duration": len(mono) / sr, "peaks": peaks}))


# ---------------------------------------------------------------- stage 5: arrangement

def _clean_melody(raw_midi: Path, info: dict) -> list:
    """-> [(eighth_index, pitch, n_eighths)] on the source-song eighth grid."""
    beat_times = info["beat_times"]
    scale = MAJOR_SCALE if info["key_mode"] == "major" else MINOR_SCALE
    scale_pcs = {(info["key_pc"] + d) % 12 for d in scale}

    # eighth-note grid over the source timeline
    grid = []
    for i in range(len(beat_times) - 1):
        grid.append(beat_times[i])
        grid.append((beat_times[i] + beat_times[i + 1]) / 2)
    grid.append(beat_times[-1])
    grid = np.array(grid)

    def to_grid(t):
        return int(np.argmin(np.abs(grid - t)))

    src = pretty_midi.PrettyMIDI(str(raw_midi))
    notes = sorted((n for inst in src.instruments if not inst.is_drum
                    for n in inst.notes if 43 <= n.pitch <= 84),
                   key=lambda n: (n.start, -(n.end - n.start)))
    if not notes:
        return []
    span = max(n.end for n in notes) - notes[0].start

    # dense polyphonic sources (instrument stems: chords, arpeggios, doublings)
    # need real lead-line extraction, not just overlap pruning — otherwise the
    # accompaniment becomes "extra key presses that make no sense"
    if len(notes) / max(span, 1.0) > 3.5:
        med_dur = float(np.median([n.end - n.start for n in notes]))
        clusters, cur = [], [notes[0]]
        for n in notes[1:]:
            if n.start - cur[-1].start < 0.08:
                cur.append(n)          # same strum/chord/arp onset
            else:
                clusters.append(cur)
                cur = [n]
        clusters.append(cur)
        skyline, register = [], None
        for cl in clusters:
            def salience(n):
                s = (n.end - n.start) / med_dur + 0.02 * n.pitch
                if register is not None:
                    s -= abs(n.pitch - register) / 6.0   # stay near the melody's register
                return s
            best = max(cl, key=salience)
            if best.end - best.start < 0.5 * med_dur:
                continue               # short stab, not melody
            skyline.append(best)
            register = best.pitch if register is None else 0.7 * register + 0.3 * best.pitch
        if len(skyline) >= 24:
            notes = skyline

    # monophony: drop notes fully inside the previous kept note (unless much longer)
    kept = []
    for n in notes:
        if kept and n.start < kept[-1].end - 0.06:
            if n.end - n.start > 1.8 * (kept[-1].end - kept[-1].start):
                kept[-1] = n
            continue
        kept.append(n)

    # ghost-note filter: a short note leaping far from BOTH neighbours is
    # transcription junk, not melody
    kept = [n for i, n in enumerate(kept)
            if not (0 < i < len(kept) - 1
                    and abs(n.pitch - kept[i - 1].pitch) > 9
                    and abs(n.pitch - kept[i + 1].pitch) > 9
                    and (n.end - n.start) < 0.3)]

    # octave-jump repair against a running median
    pitches = []
    for n in kept:
        p = n.pitch
        if len(pitches) >= 3:
            med = float(np.median(pitches[-5:]))
            for shift in (-12, 12):
                if abs(p + shift - med) + 4 < abs(p - med):
                    p += shift
                    break
        pitches.append(p)

    # scale snap + grid quantize
    out = []
    for n, p in zip(kept, pitches):
        if p % 12 not in scale_pcs:
            up, down = p + 1, p - 1
            p = up if up % 12 in scale_pcs else (down if down % 12 in scale_pcs else p)
        g0, g1 = to_grid(n.start), to_grid(n.end)
        if g1 <= g0:
            g1 = g0 + 1
        if out and out[-1][0] == g0:
            continue
        out.append([g0, int(p), g1 - g0])

    # legato: extend each note to the next onset (capped at 2 bars = 16 eighths)
    for i in range(len(out) - 1):
        gap = out[i + 1][0] - out[i][0]
        out[i][2] = min(max(out[i][2], gap), 16)

    # register: put the melody's median around G4 (67)
    if out:
        med = float(np.median([p for _, p, _ in out]))
        shift = int(round((67 - med) / 12)) * 12
        out = [[g, p + shift, d] for g, p, d in out]
    return out


def _phrases(melody: list) -> list:
    """Split into phrases at gaps >= 4 eighths (half a bar of silence)."""
    phrases, cur = [], []
    for note in melody:
        if cur and note[0] - (cur[-1][0] + cur[-1][2]) >= 4:
            phrases.append(cur)
            cur = []
        cur.append(note)
    if cur:
        phrases.append(cur)
    return phrases


def humanize_part(inst, rng, *, spread: float = 0.0, drift_ms: float = 0.0,
                  expression: bool = False):
    """Make a rendered part sound played rather than stamped.

    Ported from composerkit's realism pass, which measurably cut "sounds like
    MIDI" on the Composer tab. Three things matter here:

      spread     — voices of a chord entering together on the exact same
                   millisecond is the loudest artificial tell there is. The
                   string pad was built with jitter=0.0, so every note of every
                   chord landed on the same sample. Real players don't.
      drift      — a slow correlated wander reads as a human keeping time;
                   independent per-note noise (what add()'s uniform jitter did)
                   reads as unsteadiness.
      expression — CC11 within each held note. FluidR3 responds to it (verified:
                   a swelled note renders 0.05 -> 0.13 where a flat one sits at
                   0.17), and a dead-flat sustained pad is the other big tell.

    Everything is in SECONDS, so the feel doesn't change with tempo.
    """
    if not inst.notes:
        return
    notes = sorted(inst.notes, key=lambda n: (n.start, n.pitch))

    if spread > 0:
        groups = {}
        for n in notes:
            groups.setdefault(round(n.start, 3), []).append(n)
        for _t, grp in groups.items():
            if len(grp) < 2:
                continue
            order = sorted(grp, key=lambda n: rng.random())
            for k, n in enumerate(order):
                off = rng.uniform(0.0, spread)
                n.start += off
                n.velocity = max(1, min(127, int(n.velocity * (1.0 - 0.05 * k))))

    if drift_ms > 0:
        sigma, cap = drift_ms / 1000.0, 2.5 * drift_ms / 1000.0
        d = 0.0
        for n in notes:
            d = max(-cap, min(cap, d * 0.82 + rng.gauss(0.0, sigma)))
            n.start = max(0.0, n.start + d)
            n.end = max(n.start + 0.05, n.end + d)

    if expression:
        # one curve per chord entry: soft in, grow into the body, ease off
        starts = sorted({round(n.start, 3) for n in notes})
        for i, t0 in enumerate(starts):
            grp = [n for n in notes if abs(n.start - t0) < 0.05]
            if not grp:
                continue
            t1 = starts[i + 1] if i + 1 < len(starts) else max(n.end for n in grp)
            span = max(0.25, min(max(n.end for n in grp) - t0, t1 - t0))
            peak = 0.70 + 0.30 * (max(n.velocity for n in grp) / 127.0)
            for k in range(7):
                f = k / 6.0
                val = (0.36 + (peak - 0.36) * (f / 0.4) if f < 0.4
                       else peak - (peak * 0.22) * ((f - 0.4) / 0.6))
                inst.control_changes.append(pretty_midi.ControlChange(
                    11, int(max(8, min(127, val * 127))), t0 + f * span))


def arrange(raw_midi: Path, info: dict, out_piano: Path, out_layers: Path,
            tempo_scale: float = 0.72, celesta: bool = True, pad: bool = True,
            clamp_bpm: bool = True, humanize: bool = True) -> float:
    """Build the lullaby on a clean output grid. Returns output bpm.

    clamp_bpm=False skips the 55-88bpm clamp, keeping out_bpm a literal
    tempo_scale multiple of the source tempo instead — used when this runs
    alongside the melody-match engine in a hybrid render (Melody Match tab)
    so the two tracks' timelines scale by the same factor rather than
    drifting apart over a long song (melody-match's timeline is a plain
    linear stretch by tempo_scale; the clamp exists only for the standalone
    Piano engine's own "true lullaby tempo" target)."""
    rng = np.random.default_rng(7)
    out_bpm = info["tempo"] * tempo_scale
    if clamp_bpm:
        out_bpm = float(np.clip(out_bpm, 55, 88))
    e = 60.0 / out_bpm / 2          # seconds per eighth note at the output tempo
    bar_len = 8                      # eighths per 4/4 bar

    melody = _clean_melody(raw_midi, info)
    if not melody:
        raise RuntimeError("no melody could be transcribed from this track")

    # map source bars onto the output eighth grid
    beat_times = np.array(info["beat_times"])
    phase = info["bar_phase"]

    def src_time_to_eighth(t):
        b = int(np.searchsorted(beat_times, t)) - 1
        b = max(0, min(b, len(beat_times) - 2))
        frac = (t - beat_times[b]) / max(beat_times[b + 1] - beat_times[b], 1e-6)
        return (b - phase) * 2 + frac * 2

    bars = []
    for bar in info["bars"]:
        g0 = int(round(src_time_to_eighth(bar["start"])))
        bars.append({"g0": g0, "root": bar["root"], "qual": bar["qual"]})

    intro_e = bar_len                # one bar of accompaniment before the song
    total_e = max(m[0] + m[2] for m in melody) + intro_e + 2 * bar_len

    pm = pretty_midi.PrettyMIDI(initial_tempo=out_bpm)
    piano = pretty_midi.Instrument(program=PROG_PIANO, name="piano")
    box = pretty_midi.Instrument(program=PROG_MUSIC_BOX, name="music box")
    strings = pretty_midi.Instrument(program=PROG_STRINGS, name="pad")

    def add(inst, pitch, g_start, g_len, vel, jitter=0.012):
        t0 = (g_start + intro_e) * e + float(rng.uniform(-jitter, jitter))
        t0 = max(0.0, t0)
        inst.notes.append(pretty_midi.Note(velocity=int(vel), pitch=int(pitch),
                                           start=t0, end=t0 + g_len * e + 0.06))

    # ---- accompaniment: rocking broken chord (R 5 10 5 | 3' 5 10 5) per bar ----
    def chord_tones(root_pc, qual):
        root = 36 + ((root_pc - 0) % 12)          # C2..B2 register
        if root > 45:
            root -= 12
        third = root + (4 if qual == "maj" else 3)
        return root, third, root + 7, third + 12   # root, third, fifth, tenth

    last_g = -1
    for bi, bar in enumerate(bars):
        g0 = bar["g0"]
        if g0 <= last_g or g0 < -bar_len or g0 > total_e:
            continue
        last_g = g0
        r, t3, f5, t10 = chord_tones(bar["root"], bar["qual"])
        pattern = [r, f5, t10, f5, t3 + 12, f5, t10, f5]
        for k, pitch in enumerate(pattern):
            vel = 40 + (4 if k == 0 else 0) + int(rng.integers(-3, 4))
            add(piano, pitch, g0 + k, 2, vel)      # each arp note rings 2 eighths
        # sustain pedal per bar
        t_dn = (g0 + intro_e) * e + 0.01
        t_up = (g0 + intro_e + bar_len) * e - 0.04
        piano.control_changes.append(pretty_midi.ControlChange(64, 100, max(0, t_dn)))
        piano.control_changes.append(pretty_midi.ControlChange(64, 0, max(0, t_up)))
        if pad:
            for pitch, vel in ((r + 12, 25), (f5 + 12, 22), (t3 + 12, 20)):
                add(strings, pitch, g0, bar_len, vel, jitter=0.0)

    # intro bar: first chord's pattern, softer
    if bars:
        r, t3, f5, t10 = chord_tones(bars[0]["root"], bars[0]["qual"])
        for k, pitch in enumerate([r, f5, t10, f5, t3 + 12, f5, t10, f5]):
            add(piano, pitch, k - intro_e, 2, 32 + int(rng.integers(-2, 3)))

    # ---- melody: phrase-shaped velocities, music-box doubling ----
    for phrase in _phrases(melody):
        n = len(phrase)
        for i, (g0, pitch, d) in enumerate(phrase):
            arch = np.sin(np.pi * (i / max(n - 1, 1))) if n > 1 else 0.6
            vel = int(54 + 16 * arch + rng.integers(-3, 4))
            add(piano, pitch, g0, d, vel)
            if celesta:
                add(box, pitch + 12, g0, min(d, 8), max(24, vel - 26), jitter=0.02)

    # ---- ending: gentle I chord ----
    if bars:
        tonic = info["key_pc"]
        qual = "maj" if info["key_mode"] == "major" else "min"
        r, t3, f5, t10 = chord_tones(tonic, qual)
        g_end = last_g + bar_len
        for pitch, vel in ((r, 40), (f5, 34), (t3 + 12, 32), (r + 24, 30)):
            add(piano, pitch, g_end, bar_len * 2, vel, jitter=0.03)
        t_dn = (g_end + intro_e) * e
        piano.control_changes.append(pretty_midi.ControlChange(64, 100, t_dn))

    if humanize:
        # A lullaby lives or dies on feel, so this matters more here than
        # anywhere: the piano gets a gentle wander and a hand-roll on its
        # chords, the string pad gets a proper ragged ensemble entry plus
        # within-note swell (it was previously jitter=0.0 and dead flat), and
        # the music box gets the lightest touch since it's a struck sound.
        hr = np.random.default_rng(11)

        class _R:                      # tiny shim: pretty_midi wants python rng
            random = staticmethod(lambda: float(hr.random()))
            uniform = staticmethod(lambda a, b: float(hr.uniform(a, b)))
            gauss = staticmethod(lambda m, s: float(hr.normal(m, s)))
        humanize_part(piano, _R, spread=0.012, drift_ms=5.0)
        if pad:
            humanize_part(strings, _R, spread=0.045, drift_ms=7.0, expression=True)
        if celesta:
            humanize_part(box, _R, spread=0.006, drift_ms=4.0)

    pm.instruments.append(piano)
    pm.write(str(out_piano))

    pm2 = pretty_midi.PrettyMIDI(initial_tempo=out_bpm)
    if celesta and box.notes:
        pm2.instruments.append(box)
    if pad and strings.notes:
        pm2.instruments.append(strings)
    if pm2.instruments:
        pm2.write(str(out_layers))
    elif out_layers.exists():
        out_layers.unlink()

    print(f"   out tempo: {out_bpm:.0f} bpm   melody notes: {len(melody)}   "
          f"piano notes: {len(piano.notes)}", flush=True)
    return out_bpm


# ---------------------------------------------------------------- melody-match engine
def _hz_to_midi(f0: np.ndarray) -> np.ndarray:
    out = np.zeros_like(f0)
    voiced = f0 > 0
    out[voiced] = 69 + 12 * np.log2(f0[voiced] / 440.0)
    return out


def extract_f0_curve(wav_path: Path, f0_min: float = 65, f0_max: float = 1100):
    """Continuous per-~10ms pitch + loudness curve via FCPE — the melody-match
    engine's source of truth, in place of basic-pitch's quantized transcription."""
    import soundfile as sf
    import torch
    try:
        from torchfcpe import spawn_bundled_infer_model
    except ImportError:
        raise RuntimeError("melody-match needs torchfcpe: "
                           "'.venv\\Scripts\\python.exe -m pip install torchfcpe'")

    device = "cuda" if torch.cuda.is_available() else "cpu"
    model = spawn_bundled_infer_model(device=device)
    audio, sr = sf.read(str(wav_path), always_2d=True)
    mono = audio.mean(axis=1).astype(np.float32)
    t = torch.from_numpy(mono).float().to(device).unsqueeze(0).unsqueeze(-1)
    f0 = model.infer(t, sr=sr, decoder_mode="local_argmax", threshold=0.006,
                      f0_min=f0_min, f0_max=f0_max)
    f0 = f0.squeeze().cpu().numpy()
    hop = len(mono) / sr / len(f0)

    hop_samples = int(round(hop * sr))
    rms = np.zeros(len(f0), dtype=np.float32)
    for i in range(len(f0)):
        s, e = i * hop_samples, min(len(mono), i * hop_samples + hop_samples)
        if e > s:
            rms[i] = np.sqrt(np.mean(mono[s:e] ** 2))
    return f0, rms, hop


def _voiced_segments(f0_midi: np.ndarray, hop: float,
                     merge_gap_s: float = 0.08, min_note_s: float = 0.05) -> list:
    """Contiguous voiced runs, bridging short unvoiced gaps (consonants,
    breaths) so brief dropouts don't retrigger a new note mid-phrase."""
    n = len(f0_midi)
    voiced = f0_midi > 0
    merge_gap_frames = max(1, int(round(merge_gap_s / hop)))
    bridged = voiced.copy()
    i = 0
    while i < n:
        if not voiced[i]:
            j = i
            while j < n and not voiced[j]:
                j += 1
            if (j - i) <= merge_gap_frames and i > 0 and j < n:
                bridged[i:j] = True
            i = j
        else:
            i += 1
    segments = []
    i = 0
    while i < n:
        if bridged[i]:
            j = i
            while j < n and bridged[j]:
                j += 1
            if (j - i) * hop >= min_note_s:
                segments.append((i, j))
            i = j
        else:
            i += 1
    return segments


def _fill_gaps(f0_midi: np.ndarray, s: int, e: int) -> np.ndarray:
    """Bridged (unvoiced) frames inside a segment get linearly interpolated
    pitch from the surrounding voiced frames instead of a 0 dropout."""
    seg = f0_midi[s:e].copy()
    idx = np.arange(len(seg))
    voiced_idx = idx[seg > 0]
    if len(voiced_idx) == 0:
        return seg
    return np.interp(idx, voiced_idx, seg[voiced_idx])


def _segment_notes(pitch_curve: np.ndarray, hop: float,
                   min_note_s: float = 0.08, smooth_s: float = 0.08) -> list:
    """Split one continuous (gap-filled) pitch curve into distinct sung
    NOTES, rather than treating the whole curve as a single pitch-bent tone
    — a phrase-long glide sounds like a siren/theremin wail, not a melody.

    A rolling median (smooth_s wide) absorbs vibrato/jitter before rounding
    to the nearest semitone, then consecutive same-semitone frames become one
    run; any run shorter than min_note_s (a vibrato wobble that briefly
    crossed a semitone boundary, or transcription noise) is merged into
    whichever neighboring run is closer in pitch. Returns [(start, end,
    base_pitch)] — base_pitch is the median of the RAW curve over that span,
    so the note itself still carries its true tuning, only the boundaries
    are decided from the smoothed version."""
    n = len(pitch_curve)
    if n == 0:
        return []
    win = max(1, int(round(smooth_s / hop)))
    smoothed = np.empty(n)
    for i in range(n):
        lo, hi = max(0, i - win), min(n, i + win + 1)
        smoothed[i] = np.median(pitch_curve[lo:hi])
    rounded = np.round(smoothed).astype(int)

    runs = []
    i = 0
    while i < n:
        j = i
        while j < n and rounded[j] == rounded[i]:
            j += 1
        runs.append([i, j, rounded[i]])
        i = j

    min_frames = max(1, int(round(min_note_s / hop)))
    changed = True
    while changed and len(runs) > 1:
        changed = False
        for k, (s, e, p) in enumerate(runs):
            if (e - s) < min_frames:
                left = runs[k - 1] if k > 0 else None
                right = runs[k + 1] if k + 1 < len(runs) else None
                if left and (not right or abs(p - left[2]) <= abs(p - right[2])):
                    left[1] = e
                elif right:
                    right[0] = s
                else:
                    break
                del runs[k]
                changed = True
                break

    return [(s, e, int(np.clip(round(float(np.median(pitch_curve[s:e]))), 1, 126)))
            for s, e, _ in runs]


def build_melody_match_midi(f0_midi: np.ndarray, rms: np.ndarray, hop: float,
                            out_midi: Path, program: int, tempo_scale: float = 1.0,
                            bend_range: int = BEND_RANGE_SEMITONES) -> int:
    """MIDI tracing the actual sung notes (no scale-snap, no rhythm grid) —
    phrases are split into distinct notes by _segment_notes, each with its
    own note-on/off, and only the small residual deviation within a note
    (vibrato, slight scoop) is carried as pitch-bend. tempo_scale stretches
    time only (pitch is untouched); <1.0 slows the piece, matching the
    arranged engine's slider."""
    phrases = _voiced_segments(f0_midi, hop)
    ticks_per_beat = 960
    tempo_us = 500000  # fixed 120bpm clock, used only as the tick<->seconds basis
    ticks_per_sec = ticks_per_beat / (tempo_us / 1e6)

    mid = mido.MidiFile(ticks_per_beat=ticks_per_beat)
    track = mido.MidiTrack()
    mid.tracks.append(track)
    track.append(mido.MetaMessage("set_tempo", tempo=tempo_us, time=0))
    track.append(mido.Message("program_change", program=program, channel=0, time=0))
    for cc, val in ((101, 0), (100, 0), (6, bend_range), (38, 0)):  # RPN 0: bend range
        track.append(mido.Message("control_change", control=cc, value=val, channel=0, time=0))

    rms_max = max(float(rms.max()), 1e-6)
    events = []
    n_notes = 0
    for ps, pe in phrases:
        phrase_pitch = _fill_gaps(f0_midi, ps, pe)
        for ns, ne, base in _segment_notes(phrase_pitch, hop):
            s, e = ps + ns, ps + ne
            n_notes += 1
            vel = int(np.clip(30 + 90 * (rms[s:e].mean() / rms_max), 30, 110))
            events.append((s * hop, mido.Message("note_on", note=base, velocity=vel, channel=0)))
            for k in range(s, e):
                frac = phrase_pitch[k - ps] - base
                bend = int(np.clip(frac / bend_range, -1, 1) * 8191)
                expr = int(np.clip(30 + 97 * (rms[k] / rms_max), 0, 127))
                t = k * hop
                events.append((t, mido.Message("pitchwheel", pitch=bend, channel=0)))
                events.append((t, mido.Message("control_change", control=11, value=expr, channel=0)))
            events.append((e * hop, mido.Message("note_off", note=base, velocity=0, channel=0)))

    events.sort(key=lambda x: x[0])
    last_tick = 0
    for t_sec, msg in events:
        tick = int(round((t_sec / max(tempo_scale, 0.01)) * ticks_per_sec))
        msg.time = max(0, tick - last_tick)
        track.append(msg)
        last_tick = tick
    track.append(mido.MetaMessage("end_of_track", time=0))
    mid.save(str(out_midi))
    return n_notes


def render_melody_match(stem_dir: Path, work: Path, out_base: Path,
                        analysis_json: Path = None, solo_weights: dict = None,
                        arranged_weights: dict = None, instrument: str = "cello",
                        tempo_scale: float = 0.85, celesta: bool = True,
                        pad: bool = True) -> list:
    """Melody Match end to end — and, unlike the other two engines, able to
    route DIFFERENT stems to DIFFERENT treatments in one render rather than
    forcing everything selected through the same pipeline:

    solo_weights stems are mixed together and traced with continuous F0
    (FCPE, no scale-snap) onto a single portamento-capable instrument —
    for whichever stem you want the exact melody preserved from (typically
    vocals). arranged_weights stems instead go through the Piano engine's own
    transcribe+arrange pipeline (quantized to a scale/eighth-grid, rebuilt as
    a full piano+music-box arrangement) — for a track you'd rather hear as a
    proper backing arrangement than a literal pitch trace (e.g. an existing
    piano/guitar part in the song). Whichever of the two are used get mixed
    together before mastering. Returns [mp3_path, wav_path]."""
    parts = []

    if solo_weights and any(v > 0 for v in solo_weights.values()):
        program = CONTINUOUS_INSTRUMENTS.get(instrument, CONTINUOUS_INSTRUMENTS["cello"])
        mix_path = _mix_stems_weighted(stem_dir, solo_weights, work / "melody_match_mix.wav")
        f0, rms, hop = extract_f0_curve(mix_path)
        f0_midi = _hz_to_midi(f0)
        midi_path = work / "melody_match.mid"
        n_notes = build_melody_match_midi(f0_midi, rms, hop, midi_path, program, tempo_scale)
        solo_wav = work / "melody_match_raw.wav"
        sf_choice = piano_soundfont() if instrument == "music_box" else SF_FLUIDR3
        render_midi(midi_path, solo_wav, sf_choice, gain=0.6)
        print(f"   melody-match: {instrument} (GM program {program}), {n_notes} notes, "
              f"tempo x{tempo_scale:g}", flush=True)
        parts.append(solo_wav)

    if arranged_weights and any(v > 0 for v in arranged_weights.values()):
        if not analysis_json or not analysis_json.is_file():
            raise RuntimeError("arranged-stems needs the song analyzed first (no analysis.json)")
        info = json.loads(analysis_json.read_text())
        melody_midi = work / "melody.mid"
        transcribe_melody(stem_dir, melody_midi, stem_weights=arranged_weights)
        piano_midi, layers_midi = work / "piano.mid", work / "layers.mid"
        # clamp_bpm=False: keep this in the same linear tempo_scale as the
        # melody-match track above, so the two don't drift apart over the song
        arrange(melody_midi, info, piano_midi, layers_midi, tempo_scale=tempo_scale,
               celesta=celesta, pad=pad, clamp_bpm=False)
        piano_wav = work / "piano.wav"
        render_midi(piano_midi, piano_wav, piano_soundfont(), gain=0.55)
        parts.append(piano_wav)
        if layers_midi.exists():
            layers_wav = work / "layers.wav"
            render_midi(layers_midi, layers_wav, SF_FLUIDR3, gain=0.45)
            parts.append(layers_wav)
        print("   arranged: piano+music-box render added", flush=True)

    if not parts:
        raise RuntimeError("no stems routed to either engine — tick at least one track")

    if len(parts) == 1:
        premix = parts[0]
    else:
        premix = work / "hybrid_premix.wav"
        inputs = []
        for p in parts:
            inputs += ["-i", str(p)]
        filt = "".join(f"[{i}:a]" for i in range(len(parts))) + f"amix=inputs={len(parts)}:normalize=0"
        run([_ffmpeg_exe(), "-y", *inputs, "-filter_complex", filt, "-ar", "44100", str(premix)])

    wav_path = out_base.with_suffix(".wav")
    master(premix, None, wav_path)
    mp3_path = out_base.with_suffix(".mp3")
    run([_ffmpeg_exe(), "-y", "-i", str(wav_path), "-codec:a", "libmp3lame",
         "-b:a", "192k", str(mp3_path)])
    return [mp3_path, wav_path]


# ---------------------------------------------------------------- stage 6: render + master

REVERB = ["-o", "synth.reverb.active=1", "-o", "synth.reverb.room-size=0.78",
          "-o", "synth.reverb.damp=0.25", "-o", "synth.reverb.width=0.9",
          "-o", "synth.reverb.level=0.72", "-o", "synth.chorus.active=0"]


def render_midi(midi_path: Path, wav_path: Path, soundfont: Path,
                gain: float, sr: int = 44100) -> Path:
    run([str(FLUIDSYNTH), "-ni", *REVERB, "-F", str(wav_path), "-r", str(sr),
         "-g", str(gain), str(soundfont), str(midi_path)])
    return wav_path


def master(piano_wav: Path, layers_wav, out_path: Path, fade_in: float = 3.0,
           fade_out: float = 7.0) -> Path:
    ff = _ffmpeg_exe()
    probe = subprocess.run(
        [_ffmpeg_exe("ffprobe"), "-v", "quiet", "-show_entries", "format=duration",
         "-of", "default=noprint_wrappers=1:nokey=1", str(piano_wav)],
        check=True, capture_output=True, text=True)
    duration = float(probe.stdout.strip())
    fade_start = max(0.0, duration - fade_out)

    chain = (
        "highshelf=f=6500:g=-3.5,"
        "lowshelf=f=170:g=1.5,"
        "lowpass=f=11000,"
        "acompressor=threshold=0.06:ratio=2:attack=40:release=400:makeup=1.5,"
        "loudnorm=I=-16:TP=-1.5:LRA=9,"
        f"afade=t=in:d={fade_in},afade=t=out:st={fade_start}:d={fade_out}:curve=exp"
    )
    if layers_wav and Path(layers_wav).is_file():
        run([ff, "-y", "-i", str(piano_wav), "-i", str(layers_wav),
             "-filter_complex",
             f"[1:a]volume=0.5[l];[0:a][l]amix=inputs=2:normalize=0,{chain}",
             "-ar", "44100", str(out_path)])
    else:
        run([ff, "-y", "-i", str(piano_wav), "-af", chain, "-ar", "44100", str(out_path)])
    return out_path


# ---------------------------------------------------------------- main

STAGES = ["extract", "separate", "mix", "transcribe", "arrange", "render"]


def main():
    ap = argparse.ArgumentParser(description=__doc__)
    ap.add_argument("input", type=Path)
    ap.add_argument("--tempo-scale", type=float, default=0.72,
                     help="<1.0 slows the piece (output clamped to 55-88 bpm)")
    ap.add_argument("--out", type=Path, default=None)
    ap.add_argument("--from-stage", choices=STAGES, default="extract")
    ap.add_argument("--until-stage", choices=STAGES, default="render",
                     help="stop after this stage (e.g. 'mix' when another engine "
                          "consumes the drum-free stem mix)")
    ap.add_argument("--remix-prep", action="store_true",
                     help="also write remix_input.wav during the mix stage "
                          "(vocals+other, harmonic-only, tail-trimmed) for the "
                          "ACE-Step remix engine")
    ap.add_argument("--melody-source",
                     choices=["auto", "vocals", "instruments", "vocals_other"],
                     default="auto",
                     help="legacy: which stem(s) carry the tune when no "
                          "--melody-stems selection is given (plain CLI use)")
    ap.add_argument("--melody-stems", default=None,
                     help="JSON dict of per-stem levels from the multitrack "
                          "workbench Tracks panel, e.g. "
                          "'{\"vocals\":1,\"other\":1,\"guitar\":0}' — the "
                          "melody is transcribed directly from this mix, no "
                          "further picking. Overrides --melody-source.")
    ap.add_argument("--analysis-extras", action="store_true",
                     help="workbench analyze phase: write the waveform envelope "
                          "and per-stem preview/audio-level JSON for the "
                          "multitrack Tracks panel")
    ap.add_argument("--remix-focus", choices=["both", "vocals", "instruments"],
                     default="both",
                     help="with --remix-prep: which stems dominate the remix "
                          "input (the other is attenuated, not removed)")
    ap.add_argument("--stem-weights", default=None,
                     help="with --remix-prep: JSON dict of per-stem levels from "
                          "the multitrack workbench, e.g. "
                          "'{\"vocals\":1,\"piano\":1,\"bass\":0}' — overrides "
                          "--remix-focus")
    ap.add_argument("--no-celesta", action="store_true", help="skip the music-box layer")
    ap.add_argument("--no-pad", action="store_true", help="skip the soft string pad")
    ap.add_argument("--no-humanize", action="store_true",
                     help="skip the performance pass (chord spread, correlated "
                          "timing drift, CC11 swells) and render the raw "
                          "quantized arrangement — for A/B comparison")
    ap.add_argument("--engine", choices=["arranged", "melody-match"], default="arranged",
                     help="'arranged' (default): transcribe+quantize+rebuild as a full "
                          "piano arrangement. 'melody-match': trace the actual continuous "
                          "pitch curve (FCPE) on a single portamento-capable instrument — "
                          "no scale-snap or grid quantization, closest to the sung tune.")
    ap.add_argument("--continuous-instrument", choices=list(CONTINUOUS_INSTRUMENTS),
                     default="cello", help="melody-match engine instrument")
    ap.add_argument("--arranged-stems", default=None,
                     help="melody-match engine only: JSON dict of per-stem levels to ALSO "
                          "run through the arranged (transcribe+quantize+piano) pipeline and "
                          "mix in — e.g. trace vocals on cello via --melody-stems while an "
                          "existing piano part is rebuilt as a full arrangement via this flag, "
                          "instead of forcing every ticked stem through the same engine")
    args = ap.parse_args()

    if not FLUIDSYNTH.exists():
        sys.exit(f"FluidSynth not found at {FLUIDSYNTH}")

    WORK.mkdir(exist_ok=True)
    OUTPUT.mkdir(exist_ok=True)
    work = WORK / args.input.stem
    work.mkdir(parents=True, exist_ok=True)

    raw_wav = work / "raw.wav"
    stem_dir = work / "stems" / DEMUCS_MODEL / "raw"
    mix_wav = work / "no_drums.wav"
    melody_midi = work / "melody.mid"
    analysis_json = work / "analysis.json"
    piano_midi = work / "piano.mid"
    layers_midi = work / "layers.mid"
    piano_wav = work / "piano.wav"
    layers_wav = work / "layers.wav"

    start = STAGES.index(args.from_stage)
    until = STAGES.index(args.until_stage)

    def stop_after(stage):
        if until <= STAGES.index(stage):
            print(f"\nStopped after stage: {stage}", flush=True)
            sys.exit(0)

    if start <= STAGES.index("extract"):
        print("== 1/6 extracting audio ==", flush=True)
        extract_audio(args.input, raw_wav)
    stop_after("extract")

    if start <= STAGES.index("separate"):
        # separation is expensive (two Demucs passes) and its result depends
        # only on the audio, not on any downstream choice — safe to skip if a
        # previous run (from either the Lullaby tab or the Track Splitter tab;
        # they share this same work-dir cache by song name) already has it
        if all((stem_dir / f"{s}.wav").is_file() for s in STEM_NAMES):
            print("== 2/6 stems already separated (cached) ==", flush=True)
        else:
            print("== 2/6 separating stems (demucs) ==", flush=True)
            separate_stems(raw_wav, work / "stems")
    stop_after("separate")

    if start <= STAGES.index("mix"):
        print("== 3/6 mixing non-drum stems ==", flush=True)
        mix_non_drum_stems(stem_dir, mix_wav)
        if args.remix_prep:
            weights = json.loads(args.stem_weights) if args.stem_weights else None
            prep_remix_input(stem_dir, work / "remix_input.wav",
                             focus=args.remix_focus, stem_weights=weights)
    stop_after("mix")

    if args.engine == "melody-match":
        print("== 4/6 tracking pitch (FCPE) + rendering "
              f"{args.continuous_instrument} ==", flush=True)
        solo_weights = json.loads(args.melody_stems) if args.melody_stems else DEFAULT_STEM_WEIGHTS
        arranged_weights = json.loads(args.arranged_stems) if args.arranged_stems else None
        out_base = args.out or (OUTPUT / f"{args.input.stem}_lullaby")
        out_base.parent.mkdir(parents=True, exist_ok=True)
        final = render_melody_match(stem_dir, work, out_base, analysis_json=analysis_json,
                                    solo_weights=solo_weights, arranged_weights=arranged_weights,
                                    instrument=args.continuous_instrument,
                                    tempo_scale=args.tempo_scale,
                                    celesta=not args.no_celesta, pad=not args.no_pad)
        print(f"\nDone: {final}", flush=True)
        return

    if start <= STAGES.index("transcribe"):
        # key/tempo/chords are properties of the song, not of any stem
        # selection — computed once and reused by every later re-render
        if not analysis_json.is_file():
            print("== 4/6 analyzing key/beats/chords ==", flush=True)
            analyze(raw_wav, stem_dir, analysis_json)
        if args.analysis_extras:
            print("== 4/6 building workbench previews ==", flush=True)
            write_waveform_json(raw_wav, work / "waveform.json")
            write_stem_previews(stem_dir, work)
        else:
            print("== 4/6 transcribing melody ==", flush=True)
            melody_stems = json.loads(args.melody_stems) if args.melody_stems else None
            transcribe_melody(stem_dir, melody_midi, source=args.melody_source,
                              stem_weights=melody_stems)
    stop_after("transcribe")

    info = json.loads(analysis_json.read_text())

    if start <= STAGES.index("arrange"):
        print("== 5/6 arranging ==", flush=True)
        arrange(melody_midi, info, piano_midi, layers_midi,
                tempo_scale=args.tempo_scale,
                celesta=not args.no_celesta, pad=not args.no_pad,
                humanize=not args.no_humanize)

    print("== 6/6 rendering + mastering ==", flush=True)
    sf_piano = piano_soundfont()
    print(f"   piano soundfont: {sf_piano.name}", flush=True)
    render_midi(piano_midi, piano_wav, sf_piano, gain=0.55)
    lw = None
    if layers_midi.exists():
        render_midi(layers_midi, layers_wav, SF_FLUIDR3, gain=0.45)
        lw = layers_wav

    out_base = args.out or (OUTPUT / f"{args.input.stem}_lullaby")
    out_base.parent.mkdir(parents=True, exist_ok=True)
    final = master(piano_wav, lw, out_base.with_suffix(".wav"))
    print(f"\nDone: {final}", flush=True)


if __name__ == "__main__":
    main()
```

## File 6 of 27 — `%USERPROFILE%\local-ai-studio\index.html`

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Local AI Studio</title>
<link rel="icon" href="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 32 32'%3E%3Crect width='32' height='32' rx='8' fill='%237b6cf6'/%3E%3Cg fill='none' stroke='%23fff' stroke-width='2.2' stroke-linecap='round'%3E%3Cpath d='M16 7v3M16 22v3M7 16h3M22 16h3M9.6 9.6l2.1 2.1M20.3 20.3l2.1 2.1M22.4 9.6l-2.1 2.1M11.7 20.3l-2.1 2.1'/%3E%3Ccircle cx='16' cy='16' r='3.4'/%3E%3C/g%3E%3C/svg%3E">
<script>/* set theme before paint to avoid flash */try{if(localStorage.getItem('lais-theme')==='light')document.documentElement.classList.add('light');}catch(e){}</script>
<style>
  /* ============================ DESIGN TOKENS ============================ */
  :root{
    /* neutrals / surfaces — GLASS: translucent panels over an ambient backdrop */
    --bg:#07080d;
    --surface:rgba(22,25,37,.52); --surface-2:rgba(255,255,255,.045); --surface-3:rgba(255,255,255,.09);
    --surface-solid:#161925;                     /* for masks/rings that must be opaque */
    --border:rgba(255,255,255,.09); --border-soft:rgba(255,255,255,.055);
    --edge-light:rgba(255,255,255,.06);          /* top-edge glass highlight */
    --glass:blur(18px) saturate(1.35);
    --ink:#eceef6; --mut:#9aa2b6; --mut-2:#6d7590;
    /* accent (indigo/violet) */
    --accent:#7b6cf6; --accent-hover:#8e80ff; --accent-press:#6a5bf0;
    --accent-soft:rgba(123,108,246,.14); --accent-ring:rgba(123,108,246,.45);
    /* semantic */
    --success:#3fcf7f; --warn:#fbbf24; --danger:#f4685f; --info:#5aa7ff;
    --success-soft:rgba(63,207,127,.13); --danger-soft:rgba(244,104,95,.13);
    /* domain hues (icons / active indicator only) */
    --hue-text:#5aa7ff; --hue-image:#a78bfa; --hue-voice:#34d399; --hue-story:#fbbf24; --hue-home:#7b6cf6; --hue-music:#f472b6;
    /* spacing (8pt) */
    --s1:4px; --s2:8px; --s3:12px; --s4:16px; --s5:24px; --s6:32px; --s7:48px;
    /* radius */
    --r-sm:8px; --r-md:12px; --r-lg:18px; --r-full:999px;
    /* elevation */
    --e1:0 1px 2px rgba(0,0,0,.35); --e2:0 10px 32px rgba(0,0,0,.42); --e3:0 26px 70px rgba(0,0,0,.6);
    /* motion */
    --dur:.16s; --dur-2:.28s; --ease:cubic-bezier(.4,0,.2,1);
    --font:'Inter',ui-sans-serif,system-ui,'Segoe UI',Roboto,sans-serif;
    --mono:ui-monospace,'Cascadia Code',Consolas,monospace;
    /* legacy aliases so the Story Maker block + status utils keep working */
    --card:var(--surface); --edge:var(--border); --acc:var(--accent); --ok:var(--success);
  }
  :root.light{
    --bg:#e9ebf3;
    --surface:rgba(255,255,255,.62); --surface-2:rgba(96,108,150,.07); --surface-3:rgba(96,108,150,.13);
    --surface-solid:#ffffff;
    --border:rgba(24,30,60,.10); --border-soft:rgba(24,30,60,.06);
    --edge-light:rgba(255,255,255,.8);
    --ink:#1b1e27; --mut:#5c6473; --mut-2:#8a92a3;
    --accent-soft:rgba(123,108,246,.12);
    --success-soft:rgba(63,207,127,.14); --danger-soft:rgba(244,104,95,.12);
    --e1:0 1px 2px rgba(20,22,40,.05); --e2:0 10px 28px rgba(20,22,40,.10); --e3:0 22px 60px rgba(20,22,40,.16);
  }

  /* ============================ RESET / BASE ============================ */
  *{box-sizing:border-box}
  html,body{height:100%}
  body{margin:0;background:var(--bg);color:var(--ink);
       font:14.5px/1.55 var(--font);-webkit-font-smoothing:antialiased;overflow:hidden}
  /* ambient backdrop the glass panels sit over — fixed, so blur has depth to refract */
  body::before{content:"";position:fixed;inset:0;z-index:-1;pointer-events:none;background:
    radial-gradient(640px 420px at 6% -6%,  rgba(123,108,246,.17), transparent 62%),
    radial-gradient(760px 520px at 104% 12%, rgba(244,114,182,.11), transparent 62%),
    radial-gradient(900px 640px at 46% 118%, rgba(90,167,255,.11),  transparent 62%),
    var(--bg)}
  :root.light body::before{background:
    radial-gradient(640px 420px at 6% -6%,  rgba(123,108,246,.14), transparent 62%),
    radial-gradient(760px 520px at 104% 12%, rgba(244,114,182,.10), transparent 62%),
    radial-gradient(900px 640px at 46% 118%, rgba(90,167,255,.12),  transparent 62%),
    var(--bg)}
  h1,h2,h3,h4{margin:0;font-weight:650;letter-spacing:-.01em}
  button{font:inherit;cursor:pointer}
  ::selection{background:var(--accent-soft)}
  *:focus-visible{outline:2px solid var(--accent-ring);outline-offset:2px;border-radius:6px}
  /* scrollbars */
  *::-webkit-scrollbar{width:10px;height:10px}
  *::-webkit-scrollbar-thumb{background:var(--surface-3);border-radius:8px;border:2px solid transparent;background-clip:content-box}
  *::-webkit-scrollbar-thumb:hover{background:var(--border)}

  /* ============================ APP SHELL ============================ */
  .app{display:flex;height:100vh;overflow:hidden}
  /* sidebar */
  .sidebar{width:248px;flex:0 0 248px;background:var(--surface);border-right:1px solid var(--border);
           display:flex;flex-direction:column;overflow-y:auto;z-index:40}
  .brand{display:flex;align-items:center;gap:10px;padding:18px 18px 14px}
  .brand .logo{width:30px;height:30px;border-radius:9px;flex:0 0 auto;
    background:linear-gradient(135deg,var(--accent),#a78bfa);display:grid;place-items:center;color:#fff;box-shadow:var(--e1)}
  .brand .logo svg{width:17px;height:17px}
  .brand b{font-size:15px;letter-spacing:-.02em}
  .brand small{display:block;color:var(--mut-2);font-size:11px;font-weight:500;letter-spacing:.01em}
  .nav{padding:6px 10px 14px;flex:1}
  .navgroup{margin-top:12px}
  .navlabel{font-size:10.5px;font-weight:700;letter-spacing:.09em;text-transform:uppercase;color:var(--mut-2);padding:6px 10px 5px}
  .navitem{display:flex;align-items:center;gap:11px;padding:9px 10px;border-radius:var(--r-sm);color:var(--mut);
           cursor:pointer;font-weight:550;position:relative;transition:background var(--dur) var(--ease),color var(--dur) var(--ease)}
  .navitem svg{width:17px;height:17px;flex:0 0 auto;color:var(--ico,var(--mut));transition:color var(--dur)}
  .navitem:hover{background:var(--surface-2);color:var(--ink)}
  .navitem.active{background:var(--accent-soft);color:var(--ink)}
  .navitem.active svg{color:var(--ico,var(--accent))}
  .navitem.active::before{content:"";position:absolute;left:-10px;top:8px;bottom:8px;width:3px;border-radius:3px;background:var(--ico,var(--accent))}
  .sidebar-foot{padding:12px 16px 16px;border-top:1px solid var(--border-soft);color:var(--mut-2);font-size:11.5px;line-height:1.5}
  .sidebar-foot a{color:var(--mut);text-decoration:none}
  .sidebar-foot a:hover{color:var(--accent)}

  /* main */
  .main{flex:1;min-width:0;display:flex;flex-direction:column;overflow:hidden}
  .topbar{height:58px;flex:0 0 58px;display:flex;align-items:center;gap:12px;
          padding:0 22px;border-bottom:1px solid var(--border);background:color-mix(in srgb,var(--surface) 70%,transparent);backdrop-filter:blur(8px)}
  .hamb{display:grid;place-items:center;background:transparent;border:0;color:var(--mut);padding:6px;border-radius:8px}
  .hamb:hover{background:var(--surface-2);color:var(--ink)}
  /* collapsed sidebar (desktop): icon-only rail for extra workspace */
  .sidebar{transition:width var(--dur-2) var(--ease),flex-basis var(--dur-2) var(--ease)}
  @media(min-width:1001px){
    body.nav-min .sidebar{width:70px;flex:0 0 70px}
    body.nav-min .brand{justify-content:center;padding:18px 0 10px}
    body.nav-min .brand > div,body.nav-min .navlabel,body.nav-min .navitem span,body.nav-min .sidebar-foot{display:none}
    body.nav-min .nav{padding:6px 8px 14px}
    body.nav-min .navitem{justify-content:center;padding:11px 0}
    body.nav-min .navitem.active::before{left:-8px}
    body.nav-min .navgroup{margin-top:8px;border-top:1px solid var(--border-soft);padding-top:8px}
  }
  /* GPU pill (topbar): live VRAM bar, control-panel style */
  .gpupill{display:flex;align-items:center;gap:8px;background:var(--surface-2);border:1px solid var(--border);
           border-radius:var(--r-full);padding:6px 13px;backdrop-filter:var(--glass);-webkit-backdrop-filter:var(--glass)}
  .gpupill .lab{font-size:10px;color:var(--mut-2);font-weight:700;letter-spacing:.06em;text-transform:uppercase}
  .gpubar{width:84px;height:8px;border-radius:var(--r-full);background:rgba(0,0,0,.3);overflow:hidden;
          box-shadow:inset 0 1px 2px rgba(0,0,0,.4)}
  :root.light .gpubar{background:rgba(20,25,50,.12)}
  .gpubar > div{height:100%;width:0;border-radius:var(--r-full);background:var(--success);
                transition:width .6s var(--ease),background .3s}
  .gpubar > div.warn{background:var(--warn)}
  .gpubar > div.danger{background:var(--danger)}
  #gpuText{font-size:11.5px;color:var(--mut);font-weight:600;min-width:52px;text-align:right;font-variant-numeric:tabular-nums}
  @media(max-width:860px){.gpupill{display:none}}
  #viewTitle{font-size:15.5px;font-weight:650}
  .topbar .spacer{flex:1}
  /* model status pill */
  .modelpill{display:flex;align-items:center;gap:9px;background:var(--surface-2);border:1px solid var(--border);
             border-radius:var(--r-full);padding:6px 7px 6px 13px;max-width:340px}
  .gdot{width:8px;height:8px;border-radius:50%;background:var(--mut-2);flex:0 0 auto;transition:background var(--dur)}
  .gdot.on{background:var(--success);box-shadow:0 0 0 3px var(--success-soft)}
  #gModelText{font-size:12.5px;color:var(--ink);white-space:nowrap;overflow:hidden;text-overflow:ellipsis;font-weight:550}
  .modelpill .lab{font-size:10px;color:var(--mut-2);font-weight:700;letter-spacing:.06em;text-transform:uppercase;margin-right:-3px}
  .pill-stop{background:var(--danger-soft);color:var(--danger);border:0;border-radius:var(--r-full);
             padding:4px 11px;font-size:11.5px;font-weight:650}
  .pill-stop:hover{background:var(--danger);color:#fff}
  .pill-stop:disabled{opacity:.5;cursor:default}
  .topbtn{display:flex;align-items:center;gap:7px;background:var(--surface-2);border:1px solid var(--border);
          color:var(--mut);border-radius:var(--r-sm);padding:7px 11px;font-size:12.5px;font-weight:550}
  .topbtn:hover{color:var(--ink);border-color:var(--accent)}
  .topbtn svg{width:15px;height:15px}
  .kbd{font:600 10.5px var(--mono);background:var(--surface-3);border:1px solid var(--border);border-radius:5px;padding:1px 5px;color:var(--mut)}
  .icobtn{background:var(--surface-2);border:1px solid var(--border);color:var(--mut);width:34px;height:34px;border-radius:var(--r-sm);display:grid;place-items:center}
  .icobtn:hover{color:var(--ink);border-color:var(--accent)}
  .icobtn svg{width:16px;height:16px}

  /* views */
  .views{flex:1;min-height:0;position:relative;overflow:hidden}
  .panel{display:none;height:100%;overflow:auto}
  .panel.active{display:block;animation:fade var(--dur-2) var(--ease)}
  @keyframes fade{from{opacity:0;transform:translateY(4px)}to{opacity:1;transform:none}}
  .view{padding:30px 34px 56px}
  .view-narrow{max-width:820px;margin:0 auto}
  .view-head{margin-bottom:20px}
  .view-head h1{font-size:21px}
  .view-head p{margin:5px 0 0;color:var(--mut);font-size:13.5px;max-width:60ch}

  /* ============================ COMPONENTS ============================ */
  .card{background:var(--surface);border:1px solid var(--border);border-radius:var(--r-lg);
        padding:22px;box-shadow:var(--e1);display:flex;flex-direction:column;gap:var(--s4)}
  .card + .card{margin-top:var(--s4)}
  .card-head{display:flex;align-items:center;gap:11px;margin-bottom:2px}
  .card-head .cico{width:32px;height:32px;border-radius:9px;display:grid;place-items:center;flex:0 0 auto;
                   background:var(--accent-soft);color:var(--ico,var(--accent))}
  .card-head .cico svg{width:17px;height:17px}
  .card-head h2{font-size:15.5px}
  .card-head .sub{color:var(--mut-2);font-size:12px;font-weight:500;margin-top:1px}

  .field{display:flex;flex-direction:column;gap:6px}
  .field > label,.flabel{font-size:12px;font-weight:600;color:var(--mut);letter-spacing:.005em}
  .field .note,.note{font-size:11.5px;color:var(--mut-2);font-weight:400}
  .row{display:flex;gap:12px}.row>*{flex:1;min-width:0}

  textarea,input[type=text],input[type=number],input:not([type]),select{
    width:100%;background:var(--surface-2);color:var(--ink);border:1px solid var(--border);
    border-radius:var(--r-sm);padding:10px 12px;font:inherit;transition:border-color var(--dur),box-shadow var(--dur)}
  textarea:focus,input:focus,select:focus{outline:none;border-color:var(--accent);box-shadow:0 0 0 3px var(--accent-soft)}
  textarea{resize:vertical;min-height:92px;line-height:1.6}
  select{appearance:none;background-image:url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 24 24' fill='none' stroke='%2399a1b3' stroke-width='2.2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpath d='M6 9l6 6 6-6'/%3E%3C/svg%3E");
          background-repeat:no-repeat;background-position:right 11px center;padding-right:34px;cursor:pointer}
  input[type=file]{padding:9px 12px;color:var(--mut);cursor:pointer}
  input[type=file]::file-selector-button{background:var(--surface-3);color:var(--ink);border:1px solid var(--border);
    border-radius:6px;padding:6px 12px;margin-right:11px;font:inherit;font-weight:600;cursor:pointer}
  input[type=file]::file-selector-button:hover{border-color:var(--accent);color:var(--accent)}

  /* file drop zone wrapper */
  .filedrop{display:flex;align-items:center;gap:10px;background:var(--surface-2);border:1.5px dashed var(--border);
            border-radius:var(--r-md);padding:6px;transition:border-color var(--dur),background var(--dur)}
  .filedrop:hover{border-color:var(--accent)}
  .filedrop.over{border-color:var(--accent);background:var(--accent-soft)}
  /* flex:1/width:0/min-width:0 — a long chosen-filename must never widen the input
     past its column (it was pushing the whole card under the sticky output stage) */
  .filedrop input[type=file]{background:transparent;border:0;padding:4px;flex:1 1 0;width:0;min-width:0}
  .filedrop-name{font-size:11.5px;color:var(--mut-2);margin-left:auto;padding-right:8px;max-width:45%;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}

  fieldset{border:0;padding:0;margin:0;display:flex;flex-direction:column;gap:var(--s4);transition:opacity var(--dur-2)}
  fieldset:disabled{opacity:.45;filter:saturate(.6)}

  /* buttons */
  .go{background:var(--accent);color:#fff;border:0;border-radius:var(--r-sm);padding:11px 18px;font-weight:650;
      box-shadow:var(--e1);transition:background var(--dur),transform var(--dur)}
  .go:hover{background:var(--accent-hover)}
  .go:active{transform:translateY(1px);background:var(--accent-press)}
  .go:disabled{opacity:.5;cursor:default;box-shadow:none}
  .load{background:var(--success-soft);color:var(--success);border:1px solid color-mix(in srgb,var(--success) 40%,transparent);
        border-radius:var(--r-sm);padding:8px 15px;font-weight:650;transition:background var(--dur)}
  .load:hover{background:var(--success);color:#06210f}
  .stop{background:var(--danger-soft);color:var(--danger);border:1px solid color-mix(in srgb,var(--danger) 38%,transparent);
        border-radius:var(--r-sm);padding:8px 15px;font-weight:650;transition:background var(--dur)}
  .stop:hover{background:var(--danger);color:#fff}
  .load:disabled,.stop:disabled,.go:disabled{opacity:.5;cursor:default}
  .rec{background:var(--surface-3);color:var(--ink);border:1px solid var(--border);border-radius:var(--r-sm);
       padding:9px 15px;font-weight:600;transition:background var(--dur)}
  .rec:hover{border-color:var(--accent)}
  .rec.on{background:var(--danger);color:#fff;border-color:var(--danger);animation:pulse 1.4s infinite}
  @keyframes pulse{0%,100%{box-shadow:0 0 0 0 var(--danger-soft)}50%{box-shadow:0 0 0 6px transparent}}
  a.dlbtn{display:inline-flex;align-items:center;gap:7px;background:var(--success-soft);color:var(--success);text-decoration:none;
          border:1px solid color-mix(in srgb,var(--success) 38%,transparent);border-radius:var(--r-sm);padding:9px 15px;font-weight:650;width:max-content}
  a.dlbtn:hover{background:var(--success);color:#06210f}

  /* unlocked switch (restyled checkbox) */
  .chk{display:inline-flex;align-items:center;gap:8px;font-size:12.5px;color:var(--mut);cursor:pointer;user-select:none;font-weight:550}
  /* stacked variant: a checkbox whose explanation is a sentence, not a few words.
     Plain .chk is align-items:center inline-flex, which squeezes a long .note
     into its own narrow column beside the label. */
  .chk.chk-stack{display:flex;align-items:flex-start;margin-bottom:7px}
  .chk.chk-stack > input[type=checkbox]{margin-top:1px;flex:0 0 auto}
  .chk.chk-stack > span{display:block;min-width:0}
  .chk.chk-stack .note{display:block;margin-top:1px;font-weight:400;line-height:1.45}
  .chk input[type=checkbox]{appearance:none;width:34px;height:20px;border-radius:var(--r-full);background:var(--surface-3);
    border:1px solid var(--border);position:relative;flex:0 0 auto;transition:background var(--dur);cursor:pointer}
  .chk input[type=checkbox]::after{content:"";position:absolute;top:2px;left:2px;width:14px;height:14px;border-radius:50%;
    background:var(--mut);transition:transform var(--dur),background var(--dur)}
  .chk input[type=checkbox]:checked{background:var(--accent-soft);border-color:var(--accent)}
  .chk input[type=checkbox]:checked::after{transform:translateX(14px);background:var(--accent)}
  .chk input[type=checkbox]:disabled{opacity:.4;cursor:default}

  /* load bar */
  .loadbar{display:flex;gap:10px;align-items:center;padding:10px 12px;background:var(--surface-2);
           border:1px solid var(--border);border-radius:var(--r-md)}
  .loadstat{font-size:12px;color:var(--mut);margin-left:auto;display:flex;align-items:center;gap:7px}
  .loadstat.on{color:var(--success)} .loadstat.run{color:var(--accent)} .loadstat.err{color:var(--danger)}
  .dot{display:inline-block;width:8px;height:8px;border-radius:50%;background:var(--mut-2);flex:0 0 auto}
  .dot.on{background:var(--success)} .dot.run{background:var(--accent);animation:blink 1s infinite}
  @keyframes blink{50%{opacity:.35}}

  /* advanced (collapsible) */
  details.adv{background:var(--surface-2);border:1px solid var(--border);border-radius:var(--r-md);padding:0}
  details.adv summary{list-style:none;cursor:pointer;padding:11px 14px;font-size:12.5px;font-weight:600;color:var(--mut);
    display:flex;align-items:center;gap:8px}
  details.adv summary::-webkit-details-marker{display:none}
  details.adv summary::before{content:"";width:7px;height:7px;border-right:2px solid var(--mut-2);border-bottom:2px solid var(--mut-2);
    transform:rotate(-45deg);transition:transform var(--dur)}
  details.adv[open] summary::before{transform:rotate(45deg)}
  details.adv .adv-body{padding:0 14px 14px;display:flex;flex-direction:column;gap:var(--s4)}

  /* info chip */
  .chip-info{display:flex;align-items:flex-start;gap:8px;background:var(--accent-soft);border:1px solid color-mix(in srgb,var(--accent) 30%,transparent);
    border-radius:var(--r-sm);padding:9px 12px;font-size:12px;color:var(--ink);line-height:1.45}
  .chip-info svg{width:15px;height:15px;flex:0 0 auto;margin-top:1px;color:var(--accent)}
  .chip-info.warn{background:rgba(251,191,36,.12);border-color:rgba(251,191,36,.3)}
  .chip-info.warn svg{color:var(--warn)}

  /* status + output */
  .status{font-size:12px;color:var(--mut);min-height:16px}
  .status.run{color:var(--accent)} .status.err{color:var(--danger)} .status.ok{color:var(--success)}
  .out{background:var(--surface-2);border:1px solid var(--border);border-radius:var(--r-md);padding:13px;min-height:44px;
       white-space:pre-wrap;word-break:break-word;font-size:14px;line-height:1.6}
  .out.code{font-family:var(--mono);font-size:13px}
  .out:empty{display:none}
  .scripttext{background:var(--surface-2);border:1px solid var(--border);border-left:3px solid var(--accent);
    border-radius:var(--r-sm);padding:12px 14px;font-size:15px;line-height:1.6}
  img.result{max-width:100%;border-radius:var(--r-md);border:1px solid var(--border);box-shadow:var(--e1)}
  .thumbs{display:flex;gap:8px;flex-wrap:wrap}.thumbs img{height:70px;border-radius:8px;border:1px solid var(--border)}
  /* Sprite Studio — checkerboard behind frames so transparency reads at a glance */
  /* ---- Composer wizard ---- */
  .wiz{display:flex;align-items:center;gap:6px;margin-bottom:12px;flex-wrap:wrap}
  .wiz-step{display:flex;align-items:center;gap:9px;padding:7px 14px 7px 9px;border-radius:var(--r-sm);
            border:1px solid var(--edge);background:var(--surface-2);cursor:pointer;text-align:left;color:var(--mut)}
  .wiz-step:hover:not(.lock){border-color:var(--accent)}
  .wiz-step.on{border-color:var(--accent);background:color-mix(in srgb,var(--accent) 13%,transparent);color:var(--text)}
  .wiz-step.lock{opacity:.45;cursor:not-allowed}
  .wiz-step.done .wiz-n{background:var(--ok,#22c55e);color:#0e0e12}
  .wiz-n{flex:0 0 auto;width:23px;height:23px;border-radius:50%;display:grid;place-items:center;
         background:var(--surface-3);font-size:12px;font-weight:700}
  .wiz-step.on .wiz-n{background:var(--accent);color:#fff}
  .wiz-step b{display:block;font-size:12.5px;line-height:1.2}
  .wiz-step small{display:block;font-size:10.5px;color:var(--muted)}
  .wiz-sep{flex:0 0 18px;height:1px;background:var(--edge)}
  /* Step 1 is deliberately two columns so the whole setup fits one screen —
     it used to be a single tall column that always needed scrolling. */
  .wiz-cols{display:grid;grid-template-columns:1.25fr 1fr;gap:14px;align-items:start}
  .wiz-cols > .card{display:flex;flex-direction:column;gap:9px;margin:0}
  @media (max-width:1100px){ .wiz-cols{grid-template-columns:1fr} }
  .coed-player{display:block;margin:0 0 12px;padding:10px 12px}
  .coed-player .pl-top{display:flex;align-items:baseline;gap:10px;flex-wrap:wrap;margin-bottom:6px}
  .coed-player b{font-size:13px}
  .coed-player audio{display:block;width:100%;height:36px}
  /* ---- Composer arrangement editor (DAW-style clip grid + channel strips) ---- */
  .coed{width:100%;border:1px solid var(--edge);border-radius:var(--r-sm);overflow:hidden;background:var(--surface-2)}
  .coed-bar{display:flex;align-items:center;gap:8px;padding:7px 10px;background:var(--surface-3);border-bottom:1px solid var(--edge);flex-wrap:wrap}
  .coed-bar b{font-size:12px}
  .coed-scroll{overflow-x:auto;overflow-y:visible}
  .coed-grid{min-width:max-content}
  .coed-row{display:flex;align-items:stretch;border-bottom:1px solid var(--edge);min-height:30px}
  .coed-row:last-child{border-bottom:0}
  .coed-row.on{background:color-mix(in srgb,var(--accent) 7%,transparent)}
  .coed-head{flex:0 0 168px;position:sticky;left:0;z-index:2;background:var(--surface-2);
             border-right:1px solid var(--edge);display:flex;align-items:center;gap:5px;padding:3px 6px}
  .coed-row.on .coed-head{background:color-mix(in srgb,var(--accent) 12%,var(--surface-2))}
  .coed-name{flex:1;min-width:0;font-size:12px;font-weight:600;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;cursor:pointer}
  .coed-mini{flex:0 0 auto;width:17px;height:17px;border-radius:3px;border:1px solid var(--edge);
             background:var(--surface-3);color:var(--muted);font-size:9px;font-weight:700;cursor:pointer;padding:0;line-height:15px}
  .coed-mini.act{background:var(--warn);color:#1a1a1a;border-color:var(--warn)}
  .coed-lane{flex:1;display:flex;align-items:stretch;gap:2px;padding:3px 2px}
  .coed-clip{border-radius:3px;cursor:pointer;position:relative;overflow:hidden;
             background:var(--surface-3);border:1px solid var(--edge);transition:filter .12s}
  .coed-clip:hover{filter:brightness(1.35)}
  .coed-clip.on{border-color:transparent}
  .coed-clip canvas{display:block;width:100%;height:100%;opacity:.55}
  .coed-clip .lbl{position:absolute;left:3px;top:1px;font-size:9px;font-weight:700;color:#0e0e12;opacity:.75;pointer-events:none}
  .coed-secrow{display:flex;background:var(--surface-3);border-bottom:1px solid var(--edge);position:sticky;top:0;z-index:3}
  .coed-secrow .coed-head{background:var(--surface-3)}
  .coed-sec{border-left:1px solid var(--edge);padding:4px 5px;font-size:11px;cursor:pointer;overflow:hidden}
  .coed-sec:first-child{border-left:0}
  .coed-sec:hover{background:color-mix(in srgb,var(--accent) 14%,transparent)}
  .coed-sec.sel{background:color-mix(in srgb,var(--accent) 22%,transparent)}
  .coed-sec b{display:block;font-size:11px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
  .coed-sec span{font-size:10px;color:var(--muted);white-space:nowrap}
  .coed-insp{border-top:1px solid var(--edge);padding:10px;background:var(--surface-2)}
  .coed-insp .row{gap:10px}
  .coed-knobs{display:grid;grid-template-columns:repeat(auto-fit,minmax(118px,1fr));gap:8px 12px}
  .coed-knob label{display:flex;justify-content:space-between;font-size:11px;color:var(--muted);margin-bottom:1px}
  .coed-knob input[type=range]{width:100%}
  .coed-chips{display:flex;flex-wrap:wrap;gap:4px}
  .coed-chip{font-size:11px;padding:2px 8px;border-radius:var(--r-full);border:1px solid var(--edge);
             background:var(--surface-3);cursor:pointer;color:var(--muted)}
  .coed-chip.on{background:var(--accent);border-color:var(--accent);color:#fff}
  .coed-dirty{color:var(--warn);font-size:11px}
  /* piano roll */
  .coroll{border:1px solid var(--edge);border-radius:var(--r-sm);background:var(--surface-3);overflow:auto;max-height:340px}
  .coroll canvas{display:block;cursor:crosshair}
  .coed-note{font-size:11px;color:var(--muted)}
  .spsec{width:100%;margin-bottom:16px}
  .sphead{display:flex;align-items:center;gap:10px;margin:0 0 6px}
  .sphead b{font-size:13px;text-transform:capitalize}
  .sphead a.dlbtn{padding:4px 10px;font-size:12px;margin-left:auto}
  .sphead a.dlbtn + a.dlbtn{margin-left:0}
  .spframes{display:flex;flex-wrap:wrap;gap:6px}
  .spframes img,.spsheet{border:1px solid var(--border);border-radius:8px;cursor:zoom-in;
    background:repeating-conic-gradient(rgba(127,127,127,.16) 0% 25%, transparent 0% 50%) 0 0/16px 16px}
  .spframes img{width:84px;height:84px;object-fit:contain}
  .spf{position:relative}
  .spf .rr{position:absolute;top:2px;right:2px;padding:1px 6px;font-size:11px;border-radius:6px;
    border:1px solid var(--border);background:var(--surface-2);color:var(--mut);cursor:pointer;
    opacity:0;transition:opacity .15s}
  .spf:hover .rr{opacity:1}
  .spf .rr:hover{color:var(--ink);background:var(--surface-3)}
  .spsheet{max-width:100%}
  .spchecks{display:grid;grid-template-columns:repeat(3,1fr);gap:6px 10px}
  .spchecks .chk{font-size:13px}
  .tagchips{display:flex;gap:6px;flex-wrap:wrap;align-items:center}
  audio{width:100%;margin-top:2px}
  .hide{display:none !important}

  /* progress bar */
  .prog{background:var(--surface-3);border-radius:var(--r-full);height:10px;overflow:hidden}
  .prog > div{height:100%;width:0;background:linear-gradient(90deg,var(--accent),#a78bfa);transition:width .35s var(--ease)}

  /* output preview wrapper */
  .resultwrap{display:flex;flex-direction:column;gap:10px}
  .copybtn{align-self:flex-start;background:var(--surface-2);border:1px solid var(--border);color:var(--mut);
    border-radius:var(--r-sm);padding:5px 11px;font-size:11.5px;font-weight:600}
  .copybtn:hover{color:var(--accent);border-color:var(--accent)}

  /* segmented control (generic) */
  .seg{display:inline-flex;background:var(--surface-2);border:1px solid var(--border);border-radius:var(--r-sm);padding:3px;gap:2px}
  .seg button{background:transparent;border:0;color:var(--mut);border-radius:6px;padding:6px 13px;font-weight:600;font-size:12.5px}
  .seg button.active{background:var(--accent);color:#fff}

  /* home */
  .home-hero{display:flex;align-items:center;gap:16px;margin-bottom:22px}
  .home-hero .logo{width:52px;height:52px;border-radius:14px;background:linear-gradient(135deg,var(--accent),#a78bfa);
    display:grid;place-items:center;color:#fff;box-shadow:var(--e2)}
  .home-hero .logo svg{width:28px;height:28px}
  .home-hero h1{font-size:24px}
  .home-hero p{margin:3px 0 0;color:var(--mut);font-size:13.5px}
  .home-status{display:flex;align-items:center;gap:14px;flex-wrap:wrap}
  .home-status .big{font-size:17px;font-weight:650;display:flex;align-items:center;gap:10px}
  .tiles{display:grid;grid-template-columns:repeat(auto-fill,minmax(178px,1fr));gap:13px;margin-top:6px}
  .tile{background:var(--surface);border:1px solid var(--border);border-radius:var(--r-md);padding:16px;cursor:pointer;
    display:flex;flex-direction:column;gap:9px;transition:border-color var(--dur),transform var(--dur),box-shadow var(--dur)}
  .tile:hover{border-color:var(--accent);transform:translateY(-2px);box-shadow:var(--e2)}
  .tile .tico{width:34px;height:34px;border-radius:9px;display:grid;place-items:center;background:var(--accent-soft);color:var(--ico,var(--accent))}
  .tile .tico svg{width:18px;height:18px}
  .tile b{font-size:13.5px}
  .tile span{font-size:11.5px;color:var(--mut-2);line-height:1.4}

  /* toasts */
  #toasts{position:fixed;right:18px;bottom:18px;z-index:200;display:flex;flex-direction:column;gap:9px}
  .toast{background:var(--surface-3);border:1px solid var(--border);border-left:3px solid var(--mut);
    border-radius:var(--r-sm);padding:11px 15px;box-shadow:var(--e2);font-size:13px;max-width:340px;
    opacity:0;transform:translateX(16px);transition:opacity var(--dur-2),transform var(--dur-2)}
  .toast.show{opacity:1;transform:none}
  .toast.err{border-left-color:var(--danger)} .toast.ok{border-left-color:var(--success)}

  /* command palette */
  .cmdk{position:fixed;inset:0;background:rgba(4,5,10,.55);backdrop-filter:blur(3px);z-index:150;display:flex;
    justify-content:center;align-items:flex-start;padding-top:14vh}
  .cmdk-box{width:min(540px,92vw);background:var(--surface);border:1px solid var(--border);border-radius:var(--r-lg);
    box-shadow:var(--e3);overflow:hidden;animation:pop var(--dur-2) var(--ease)}
  @keyframes pop{from{opacity:0;transform:scale(.97) translateY(-6px)}to{opacity:1;transform:none}}
  .cmdk-box input{border:0;border-bottom:1px solid var(--border);border-radius:0;background:transparent;
    padding:15px 18px;font-size:15px}
  .cmdk-box input:focus{box-shadow:none}
  .cmdk-list{max-height:50vh;overflow:auto;padding:7px}
  .cmdk-item{padding:10px 13px;border-radius:var(--r-sm);font-size:13.5px;color:var(--mut);cursor:pointer}
  .cmdk-item.on,.cmdk-item:hover{background:var(--accent-soft);color:var(--ink)}
  .cmdk-empty{padding:18px;text-align:center;color:var(--mut-2);font-size:13px}

  /* ============================ STORY MAKER ============================ */
  .panel[data-panel="story"].active{display:flex;flex-direction:column;overflow:hidden}
  .storypanel{padding:0;max-width:none;min-height:0}
  .sttoolbar{display:flex;align-items:center;gap:9px;padding:12px 18px;border-bottom:1px solid var(--edge);background:var(--card);flex-wrap:wrap;flex:0 0 auto}
  .sttitle{max-width:260px;background:var(--surface-2);font-weight:600}
  .stseg{display:flex;border:1px solid var(--edge);border-radius:var(--r-sm);overflow:hidden;padding:0}
  .segbtn{background:transparent;color:var(--mut);border:0;padding:8px 14px;font:inherit;font-weight:600;cursor:pointer}
  .segbtn.active{background:var(--accent);color:#fff}
  .sttoolbar .spacer{flex:1}
  .stload{max-width:180px}
  .stplan{display:flex;flex:1;min-height:0}
  .stread{flex:1;min-height:0;display:flex;flex-direction:column}
  .stpalette{width:266px;flex:0 0 266px;border-right:1px solid var(--edge);overflow:auto;padding:14px 13px 40px}
  .stinspector{width:316px;flex:0 0 316px;border-left:1px solid var(--edge);overflow:auto;padding:16px 15px 40px}
  .stcanvaswrap{flex:1;min-width:0;overflow:auto;background:
     radial-gradient(circle at 1px 1px,var(--surface-2) 1.5px,transparent 0);background-size:24px 24px}
  .stcanvas{padding:22px;min-width:max-content;position:relative}
  .palsec{margin-bottom:17px}
  .palsec h3{margin:0 0 8px;font-size:10.5px;letter-spacing:.07em;text-transform:uppercase;color:var(--mut);display:flex;align-items:center;gap:6px}
  .palsec h3 .miniadd{margin-left:auto}
  .miniadd{background:var(--accent);color:#fff;border:0;border-radius:6px;padding:3px 10px;font:inherit;font-size:11px;font-weight:650;cursor:pointer}
  .miniadd:hover{background:var(--accent-hover)}
  .chip{display:flex;align-items:center;gap:8px;background:var(--surface-2);border:1px solid var(--edge);border-radius:var(--r-sm);padding:8px 10px;margin-bottom:6px;cursor:grab;font-size:13px}
  .chip:hover{border-color:var(--acc)}
  .chip.sel{border-color:var(--acc);box-shadow:0 0 0 1px var(--acc) inset}
  .cdot{width:11px;height:11px;border-radius:50%;flex:0 0 auto}
  .tool{cursor:grab;border-style:dashed;color:var(--mut)}
  .palhint{font-size:11px;color:var(--mut-2);margin:2px 0 4px}
  .lane{margin-bottom:14px}
  .lanehead{display:flex;align-items:center;gap:8px;font-size:12px;color:var(--mut);margin-bottom:9px}
  .lanehead .ltag{background:var(--surface-2);border:1px solid var(--edge);border-radius:20px;padding:3px 12px;font-weight:600;color:var(--ink)}
  .lanehead.parallel .ltag{border-color:#7d5bbe;color:#d9c4ff}
  .track{display:flex;align-items:stretch}
  .beat{position:relative;width:194px;flex:0 0 194px;background:var(--card);border:1px solid var(--edge);border-radius:var(--r-md);padding:11px 12px;margin-right:28px;cursor:pointer;transition:border-color var(--dur),box-shadow var(--dur)}
  .beat:hover{border-color:#3a4150;box-shadow:var(--e1)}
  .beat.sel{border-color:var(--acc);box-shadow:0 0 0 1px var(--acc) inset}
  .beat::after{content:"";position:absolute;right:-28px;top:50%;width:28px;height:2px;background:var(--edge)}
  .beat:last-child::after{display:none}
  .beat .bt{font-weight:600;font-size:13.5px;margin-bottom:3px;word-break:break-word}
  .beat .bs{font-size:11.5px;color:var(--mut);line-height:1.4;max-height:30px;overflow:hidden}
  .beat .brow{display:flex;align-items:center;gap:5px;margin-top:8px;flex-wrap:wrap}
  .beat .bloc{font-size:11px;color:var(--mut)}
  .beat .pip{font-size:10.5px;background:#2a1d12;border:1px solid var(--acc);color:#f0a884;border-radius:10px;padding:1px 7px}
  .dropadd{flex:0 0 auto;align-self:center;border:1px dashed var(--edge);color:var(--mut);border-radius:var(--r-sm);padding:11px 13px;font-size:12px;cursor:pointer;background:transparent;white-space:nowrap}
  .dropadd:hover{border-color:var(--acc);color:var(--ink)}
  .dropadd.over,.beat.over{outline:2px dashed var(--acc);outline-offset:2px}
  .stfblane{position:relative;height:30px;margin:2px 0 12px}
  .stfb{position:absolute;top:3px;height:24px;min-width:26px;background:rgba(141,107,201,.16);border:1px solid #7d5bbe;border-radius:13px;display:flex;align-items:center;padding:0 12px;font-size:11px;color:#d9c4ff;cursor:pointer;white-space:nowrap}
  .stfbh{position:absolute;top:-2px;width:12px;height:28px;cursor:ew-resize}
  .insp h3{margin:0 0 12px;font-size:14px}
  .insp .fld{margin-bottom:11px}
  .insp label{margin-bottom:4px;display:block;font-size:12px;font-weight:600;color:var(--mut)}
  .traitrow{display:flex;gap:5px;margin-bottom:5px}
  .traitrow input{flex:2;min-width:0}.traitrow select{flex:1.3;min-width:0}
  .iconbtn{background:var(--surface-2);border:1px solid var(--edge);color:var(--mut);border-radius:6px;cursor:pointer;padding:6px 9px;font:inherit}
  .iconbtn:hover{color:var(--danger);border-color:var(--danger)}
  .del{background:var(--danger-soft);border:1px solid color-mix(in srgb,var(--danger) 38%,transparent);color:var(--danger);border-radius:var(--r-sm);padding:8px 12px;cursor:pointer;font:inherit;font-weight:650;margin-top:8px}
  .del:hover{background:var(--danger);color:#fff}
  .cbrow{display:flex;align-items:center;gap:7px;font-size:13px;color:var(--ink);margin-bottom:5px}
  .cbrow input{width:auto;flex:0 0 auto}
  .insp .swatches{display:flex;gap:6px;flex-wrap:wrap}
  .sw{width:21px;height:21px;border-radius:50%;cursor:pointer;border:2px solid transparent}
  .sw.on{border-color:var(--ink)}
  .streadbar{display:flex;align-items:center;gap:9px;padding:11px 18px;border-bottom:1px solid var(--edge);flex:0 0 auto}
  .streader{flex:1;overflow:auto;padding:30px 26px 70px;max-width:840px;margin:0 auto;width:100%}
  .streader h1{font-size:28px;margin:0 0 24px}
  .scene{margin-bottom:26px;border-bottom:1px solid var(--edge);padding-bottom:20px}
  .scenehd{display:flex;align-items:center;gap:8px;font-size:11px;letter-spacing:.06em;text-transform:uppercase;color:var(--mut);margin-bottom:10px}
  .scene.par .scenehd{color:#bfa3ec}
  .scene.fb .scenehd{color:#d9c4ff;font-style:italic}
  .scene p{margin:0 0 15px;line-height:1.85;font-size:16.5px;font-family:Georgia,'Times New Roman',serif}
  .scenehd .regen{margin-left:auto;cursor:pointer;background:transparent;border:1px solid var(--edge);color:var(--mut);border-radius:6px;padding:4px 10px;font:inherit;font-size:11px}
  .scenehd .regen:hover{border-color:var(--acc);color:var(--ink)}
  .emptyhint{color:var(--mut-2);font-size:13px;padding:24px;text-align:center}

  /* ============================ GLASS ============================ */
  /* Frosted panels: translucent surface + backdrop blur + a light top edge. */
  .card,.sidebar,.topbar,.stage,.cmdk-box,.toast,.loadbar,
  .stpalette,.stinspector,.sttoolbar{
    backdrop-filter:var(--glass);-webkit-backdrop-filter:var(--glass)}
  .card{box-shadow:var(--e1),inset 0 1px 0 var(--edge-light)}
  .tile{box-shadow:inset 0 1px 0 var(--edge-light)}
  .stage{box-shadow:inset 0 1px 0 var(--edge-light)}
  .sidebar{background:rgba(13,15,23,.42);border-right-color:var(--border-soft)}
  :root.light .sidebar{background:rgba(255,255,255,.5)}
  .topbar{background:rgba(10,12,18,.38)}
  :root.light .topbar{background:rgba(255,255,255,.45)}
  .wizring span{background:var(--surface-solid)}
  .cmdk-box{background:rgba(18,21,32,.72)}
  :root.light .cmdk-box{background:rgba(255,255,255,.8)}
  .toast{background:rgba(24,27,40,.72)}
  :root.light .toast{background:rgba(255,255,255,.85)}
  /* inputs: darker glass wells so they read as recessed */
  textarea,input[type=text],input[type=number],input:not([type]),select{
    background:rgba(0,0,0,.22);border-color:var(--border)}
  :root.light textarea,:root.light input[type=text],:root.light input[type=number],
  :root.light input:not([type]),:root.light select{background:rgba(255,255,255,.55)}
  /* primary action gets a soft tone glow */
  .go{box-shadow:0 4px 16px color-mix(in srgb,var(--accent) 28%,transparent),inset 0 1px 0 rgba(255,255,255,.22)}
  .sendbtn{box-shadow:0 4px 16px color-mix(in srgb,var(--accent) 30%,transparent)}
  /* scrollbars on glass */
  *::-webkit-scrollbar-thumb{background:rgba(255,255,255,.10);background-clip:content-box}
  :root.light *::-webkit-scrollbar-thumb{background:rgba(20,25,50,.18);background-clip:content-box}

  /* ============================ TONE SYSTEM ============================ */
  /* Each tool panel scopes its own accent; every component that already uses
     var(--accent) (buttons, focus rings, load dots, chips) tints automatically. */
  .panel[data-panel="lang"]{--tone:var(--hue-text)}
  .panel[data-panel="gen"],.panel[data-panel="edit"]{--tone:var(--hue-image)}
  .panel[data-panel="music"],.panel[data-panel="lullaby"]{--tone:var(--hue-music)}
  .panel[data-panel="stt"],.panel[data-panel="tts"],.panel[data-panel="voicestudio"],.panel[data-panel="audiobook"]{--tone:var(--hue-voice)}
  .panel[data-panel="story"]{--tone:var(--hue-story)}
  .panel[data-panel="home"]{--tone:var(--hue-home)}
  .panel{
    --accent:var(--tone,#7b6cf6);
    --accent-hover:color-mix(in srgb,var(--tone,#7b6cf6) 82%,#fff);
    --accent-press:color-mix(in srgb,var(--tone,#7b6cf6) 82%,#000);
    --accent-soft:color-mix(in srgb,var(--tone,#7b6cf6) 13%,transparent);
    --accent-ring:color-mix(in srgb,var(--tone,#7b6cf6) 45%,transparent);
    --on-accent:#0d0f16}
  .panel .go{color:var(--on-accent)}
  .panel .seg button.active,.panel .segbtn.active,.panel .miniadd{color:var(--on-accent)}
  /* topbar underline picks up the active tool's hue */
  .topbar{position:relative}
  .topbar::after{content:"";position:absolute;left:0;right:0;bottom:-1px;height:2px;
    background:var(--viewtone,transparent);opacity:.75;transition:background var(--dur-2)}

  /* ============================ HERO (per-tool header) ============================ */
  .hero{display:flex;align-items:center;gap:14px;margin-bottom:18px;flex-wrap:wrap}
  .hero .hico{width:44px;height:44px;border-radius:13px;display:grid;place-items:center;flex:0 0 auto;color:var(--tone);
    background:linear-gradient(135deg,color-mix(in srgb,var(--tone) 26%,var(--surface-2)),var(--surface-2));
    border:1px solid color-mix(in srgb,var(--tone) 34%,var(--border))}
  .hero .hico svg{width:22px;height:22px}
  .hero h1{font-size:20px;letter-spacing:-.015em}
  .hero .tag{color:var(--mut);font-size:12.5px;margin-top:2px}
  .hero .loadbar{margin-left:auto;flex:0 0 auto;background:var(--surface)}

  /* ============================ WORKSPACE (controls | output) ============================ */
  .view-wide{max-width:1360px;margin:0 auto}
  fieldset.wsfs{display:block}
  .ws{display:grid;grid-template-columns:minmax(330px,410px) minmax(0,1fr);gap:18px;align-items:start}
  .ws-ctl{display:flex;flex-direction:column;gap:var(--s4);min-width:0}
  .ws-out{position:sticky;top:0;display:flex;flex-direction:column;gap:11px}
  .stage{position:relative;background:var(--surface-2);border:1px solid var(--border);border-radius:var(--r-lg);
    min-height:400px;display:flex;flex-direction:column;align-items:center;justify-content:center;overflow:hidden;padding:14px;gap:12px}
  .stage img.result{max-width:100%;max-height:68vh;margin:0;cursor:zoom-in}
  .stage .empty{display:flex;flex-direction:column;align-items:center;gap:11px;color:var(--mut-2);
    font-size:12.5px;text-align:center;padding:26px;max-width:34ch;line-height:1.6}
  .stage .empty svg{width:44px;height:44px;opacity:.55;color:var(--tone)}
  .stage .empty b{color:var(--mut);font-size:13px}
  .stage.loading::after{content:"";position:absolute;inset:0;pointer-events:none;
    background:linear-gradient(100deg,transparent 32%,var(--accent-soft) 50%,transparent 68%);
    background-size:250% 100%;animation:shimmer 1.5s infinite}
  @keyframes shimmer{from{background-position:210% 0}to{background-position:-60% 0}}
  .stage.loading .empty{opacity:.4}
  .stage.hasresult{justify-content:flex-start}
  .meta{display:flex;gap:16px;flex-wrap:wrap;font-size:11.5px;color:var(--mut-2);align-items:center}
  .meta b{color:var(--mut);font-weight:600}
  .outbar{display:flex;gap:9px;align-items:center;flex-wrap:wrap}
  .outbar .spacer{flex:1}

  /* aspect-ratio picker */
  .aspects{display:flex;gap:6px;flex-wrap:wrap}
  .aspects button{background:var(--surface-2);border:1px solid var(--border);color:var(--mut);border-radius:8px;
    padding:7px 0;width:52px;font-size:11px;font-weight:650;display:flex;flex-direction:column;align-items:center;gap:5px}
  .aspects button i{display:block;border:1.6px solid currentColor;border-radius:3px;opacity:.75}
  .aspects button.active{background:var(--accent-soft);border-color:var(--tone);color:var(--ink)}
  .aspects button:hover{border-color:var(--tone)}

  /* lightbox */
  .lightbox{position:fixed;inset:0;z-index:180;background:rgba(3,4,8,.9);display:grid;place-items:center;cursor:zoom-out}
  .lightbox img{max-width:94vw;max-height:92vh;border-radius:8px;box-shadow:var(--e3)}

  /* music result cards (renamed from .track — that name belongs to Story Maker lanes) */
  .mutrack{display:flex;flex-direction:column;gap:8px;background:var(--surface);border:1px solid var(--border);
           border-radius:var(--r-md);padding:12px;width:100%}
  .mutrack .tr-head{display:flex;align-items:center;gap:10px;font-size:12.5px;font-weight:600;color:var(--mut)}

  /* ============================ CHAT (Language) ============================ */
  .panel[data-panel="lang"].active{display:flex;flex-direction:column;overflow:hidden}
  .chatwrap{flex:1;min-height:0;display:flex;flex-direction:column;max-width:880px;margin:0 auto;width:100%;padding:0 26px}
  .chathead{flex:0 0 auto;padding-top:22px}
  .chatlog{flex:1;overflow-y:auto;padding:8px 2px 14px;display:flex;flex-direction:column;gap:13px}
  .msg{position:relative;max-width:80%;padding:11px 15px;border-radius:15px;font-size:14px;line-height:1.62;
       white-space:pre-wrap;word-break:break-word}
  .msg.user{align-self:flex-end;background:var(--accent-soft);
    border:1px solid color-mix(in srgb,var(--tone) 28%,transparent);border-bottom-right-radius:5px}
  .msg.ai{align-self:flex-start;background:var(--surface);border:1px solid var(--border);border-bottom-left-radius:5px}
  .msg.ai.thinking{color:var(--mut-2);font-style:italic}
  .msg .mimg{max-width:200px;border-radius:9px;display:block;margin-bottom:7px}
  .msg .mcopy{position:absolute;top:6px;right:-30px;opacity:0;transition:opacity var(--dur);background:transparent;
    border:0;color:var(--mut-2);padding:4px;border-radius:6px;cursor:pointer}
  .msg.ai:hover .mcopy{opacity:1}
  .msg .mcopy:hover{color:var(--accent)}
  .chatempty{flex:1;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:12px;
    color:var(--mut-2);font-size:13px;text-align:center;padding:30px}
  .chatempty svg{width:44px;height:44px;opacity:.5;color:var(--tone)}
  .suggest{display:flex;gap:7px;flex-wrap:wrap;justify-content:center}
  .composer{flex:0 0 auto;padding:10px 2px 20px;display:flex;flex-direction:column;gap:9px}
  .composer .crow{display:flex;gap:9px;align-items:flex-end}
  .composer textarea{flex:1;min-height:52px;max-height:180px;border-radius:14px;padding:13px 15px}
  .sendbtn{width:48px;height:48px;border-radius:13px;background:var(--accent);color:var(--on-accent);border:0;
    display:grid;place-items:center;flex:0 0 auto;box-shadow:var(--e1)}
  .sendbtn:hover{background:var(--accent-hover)}
  .sendbtn:disabled{opacity:.5}
  .sendbtn svg{width:20px;height:20px}
  .composer .copts{display:flex;gap:9px;align-items:center;flex-wrap:wrap}
  .composer .copts select{width:auto;padding:6px 30px 6px 10px;font-size:12.5px}
  .imgchip{display:inline-flex;align-items:center;gap:7px;background:var(--accent-soft);border:1px solid color-mix(in srgb,var(--tone) 30%,transparent);
    border-radius:var(--r-full);padding:4px 11px;font-size:12px;color:var(--ink)}
  .imgchip button{background:transparent;border:0;color:var(--mut);padding:0 2px}
  .imgchip button:hover{color:var(--danger)}

  /* ============================ WIZARD (Voice Studio — create) ============================ */
  .wizhead{display:flex;align-items:center;gap:13px;margin-bottom:4px}
  .wizring{width:46px;height:46px;border-radius:50%;flex:0 0 auto;display:grid;place-items:center;
    font-size:11px;font-weight:700;color:var(--ink);
    background:conic-gradient(var(--tone) calc(var(--p,0)*1%),var(--surface-3) 0)}
  .wizring span{width:36px;height:36px;border-radius:50%;background:var(--surface);display:grid;place-items:center}
  .scriptcard{background:var(--surface-2);border:1px solid var(--border);border-left:3px solid var(--tone);
    border-radius:var(--r-md);padding:18px 20px;font-size:16.5px;line-height:1.65;min-height:64px}

  /* ============================ RESPONSIVE ============================ */
  @media(max-width:1000px){
    .sidebar{position:fixed;inset:0 auto 0 0;transform:translateX(-100%);transition:transform var(--dur-2) var(--ease);box-shadow:var(--e3)}
    body.nav-open .sidebar{transform:none}
    body.nav-open::after{content:"";position:fixed;inset:0;background:rgba(0,0,0,.5);z-index:35}
    .hamb{display:grid;place-items:center}
    .topbtn .kbd,.topbtn span{display:none}
    .ws{grid-template-columns:1fr}
    .ws-out{position:static}
    .stage{min-height:240px}
    .hero .loadbar{margin-left:0;width:100%}
    .msg{max-width:92%}
    .msg .mcopy{position:static;opacity:1;align-self:flex-end}
  }
  @media(max-width:720px){
    .view{padding:20px 16px 48px}
    .modelpill{max-width:190px;padding-left:10px}
    .modelpill .lab{display:none}
    .stpalette{width:210px;flex-basis:210px}.stinspector{width:240px;flex-basis:240px}
  }
  @media(prefers-reduced-motion:reduce){*{animation:none !important;transition:none !important}}
</style>
</head>
<body>

<div id="toasts" aria-live="polite"></div>

<!-- command palette -->
<div id="cmdk" class="cmdk hide" role="dialog" aria-modal="true" aria-label="Command palette">
  <div class="cmdk-box">
    <input id="cmdkInput" placeholder="Jump to a view or run an action…" autocomplete="off" spellcheck="false">
    <div id="cmdkList" class="cmdk-list"></div>
  </div>
</div>

<div class="app">
  <!-- ===================== SIDEBAR ===================== -->
  <aside class="sidebar">
    <div class="brand">
      <span class="logo"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 2v4M12 18v4M2 12h4M18 12h4M5 5l2.5 2.5M16.5 16.5L19 19M19 5l-2.5 2.5M7.5 16.5L5 19"/><circle cx="12" cy="12" r="3.2"/></svg></span>
      <div><b>Local AI Studio</b><small>RTX 4080 · zero API cost</small></div>
    </div>
    <nav class="nav" role="navigation" aria-label="Primary">
      <div class="navitem active" data-tab="home" data-title="Home" style="--ico:var(--hue-home)">
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 10.5 12 3l9 7.5"/><path d="M5 9.5V21h14V9.5"/><path d="M9 21v-6h6v6"/></svg>
        <span>Home</span>
      </div>

      <div class="navgroup">
        <div class="navlabel">Text</div>
        <div class="navitem" data-tab="lang" data-title="Language" style="--ico:var(--hue-text)">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M4 7V5h16v2"/><path d="M9 19h6"/><path d="M12 5v14"/></svg>
          <span>Language</span>
        </div>
        <div class="navitem" data-tab="story" data-title="Story Maker" style="--ico:var(--hue-story)">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M4 5a2 2 0 0 1 2-2h6v18H6a2 2 0 0 0-2 2z"/><path d="M20 5a2 2 0 0 0-2-2h-6v18h6a2 2 0 0 1 2 2z"/></svg>
          <span>Story Maker</span>
        </div>
      </div>

      <div class="navgroup">
        <div class="navlabel">Image</div>
        <div class="navitem" data-tab="gen" data-title="Image — Generate" style="--ico:var(--hue-image)">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8.5" cy="8.5" r="1.6"/><path d="M21 15l-5-5L5 21"/></svg>
          <span>Generate</span>
        </div>
        <div class="navitem" data-tab="edit" data-title="Image — Edit" style="--ico:var(--hue-image)">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 20h9"/><path d="M16.5 3.5a2.1 2.1 0 0 1 3 3L7 19l-4 1 1-4z"/></svg>
          <span>Edit</span>
        </div>
        <div class="navitem" data-tab="sprite" data-title="Sprite Studio" style="--ico:var(--hue-image)">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="7" height="7" rx="1"/><rect x="14" y="3" width="7" height="7" rx="1"/><rect x="3" y="14" width="7" height="7" rx="1"/><rect x="14" y="14" width="7" height="7" rx="1"/></svg>
          <span>Sprites</span>
        </div>
      </div>

      <div class="navgroup">
        <div class="navlabel">Music</div>
        <div class="navitem" data-tab="composer" data-title="Composer — arrange &amp; mix a song" style="--ico:var(--hue-music)">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M4 20V9M9 20V4M14 20v-7M19 20v-11"/><circle cx="4" cy="20" r="1.4"/><circle cx="9" cy="20" r="1.4"/><circle cx="14" cy="20" r="1.4"/><circle cx="19" cy="20" r="1.4"/></svg>
          <span>Composer</span>
        </div>
        <div class="navitem" data-tab="music" data-title="Music Generation — ACE-Step" style="--ico:var(--hue-music)">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M9 18V5l12-2v13"/><circle cx="6" cy="18" r="3"/><circle cx="18" cy="16" r="3"/></svg>
          <span>Music Generation</span>
        </div>
        <div class="navitem" data-tab="lullaby" data-title="Lullaby — Piano Cover" style="--ico:var(--hue-music)">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 21a9 9 0 1 1 9-9c0 1-1 2-2.5 2S16 13 16 12a4 4 0 1 0-4 4"/><path d="M12 21c1.5 0 2.5-1 2.5-2.5S13.5 16 12 16"/></svg>
          <span>Lullaby</span>
        </div>
        <div class="navitem" data-tab="split" data-title="Track Splitter" style="--ico:var(--hue-music)">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M6 3v18M18 3v18"/><path d="M9 7h6M9 12h6M9 17h6"/></svg>
          <span>Track Splitter</span>
        </div>
      </div>

      <div class="navgroup">
        <div class="navlabel">Voice</div>
        <div class="navitem" data-tab="stt" data-title="Speech → Text" style="--ico:var(--hue-voice)">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="9" y="2" width="6" height="12" rx="3"/><path d="M5 11a7 7 0 0 0 14 0"/><path d="M12 18v3"/></svg>
          <span>Speech → Text</span>
        </div>
        <div class="navitem" data-tab="tts" data-title="Text → Speech" style="--ico:var(--hue-voice)">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M11 5 6 9H2v6h4l5 4z"/><path d="M15.5 8.5a5 5 0 0 1 0 7"/><path d="M19 5a9 9 0 0 1 0 14"/></svg>
          <span>Text → Speech</span>
        </div>
        <div class="navitem" data-tab="voicestudio" data-title="Voice Studio" style="--ico:var(--hue-voice)">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 2a3 3 0 0 1 3 3v6a3 3 0 0 1-6 0V5a3 3 0 0 1 3-3z"/><path d="M19 10v1a7 7 0 0 1-14 0v-1"/><path d="M12 18v4M8 22h8"/></svg>
          <span>Voice Studio</span>
        </div>
        <div class="navitem" data-tab="audiobook" data-title="Audiobook" style="--ico:var(--hue-voice)">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M4 5a2 2 0 0 1 2-2h13v18H6a2 2 0 0 0-2 2z"/><path d="M19 3v18"/><path d="M8 8h7M8 12h7"/></svg>
          <span>Audiobook</span>
        </div>
      </div>
    </nav>
    <div class="sidebar-foot">
      Everything runs on this PC — private &amp; free.<br>
      One model in VRAM at a time · mic needs the Tailscale HTTPS URL or localhost.
    </div>
  </aside>

  <!-- ===================== MAIN ===================== -->
  <div class="main">
    <header class="topbar">
      <button class="hamb" id="hamb" aria-label="Toggle navigation"><svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round"><line x1="3" y1="6" x2="21" y2="6"/><line x1="3" y1="12" x2="21" y2="12"/><line x1="3" y1="18" x2="21" y2="18"/></svg></button>
      <span id="viewTitle">Home</span>
      <span class="spacer"></span>
      <div class="gpupill" id="gpuPill" title="GPU VRAM">
        <span class="lab">GPU</span>
        <div class="gpubar"><div id="gpuFill"></div></div>
        <span id="gpuText">—</span>
      </div>
      <div class="modelpill" title="The model currently resident in VRAM">
        <span class="lab">Loaded</span>
        <span class="gdot" id="gModelDot"></span>
        <span id="gModelText">No model loaded</span>
        <button class="pill-stop hide" id="gStop" onclick="globalStop()">Stop</button>
      </div>
      <button class="topbtn" id="cmdkBtn" title="Command palette"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="7"/><path d="M21 21l-4.3-4.3"/></svg><span>Search</span><span class="kbd">Ctrl K</span></button>
      <button class="icobtn" id="themeBtn" title="Toggle light / dark" onclick="toggleTheme()" aria-label="Toggle theme">
        <svg id="themeIcon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 12.8A9 9 0 1 1 11.2 3 7 7 0 0 0 21 12.8z"/></svg>
      </button>
    </header>

    <div class="views">

      <!-- ===================== HOME ===================== -->
      <section class="panel active" data-panel="home">
        <div class="view view-narrow">
          <div class="home-hero">
            <span class="logo"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 2v4M12 18v4M2 12h4M18 12h4M5 5l2.5 2.5M16.5 16.5L19 19M19 5l-2.5 2.5M7.5 16.5L5 19"/><circle cx="12" cy="12" r="3.2"/></svg></span>
            <div>
              <h1>Local AI Studio</h1>
              <p>A private creative suite — language, images, music, speech &amp; stories, all running on your own GPU. Nothing leaves this machine; nothing is billed.</p>
            </div>
          </div>

          <div class="card">
            <div class="card-head"><span class="cico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="4" width="18" height="12" rx="2"/><path d="M7 20h10M9 16v4M15 16v4"/></svg></span><div><h2>Model status</h2><div class="sub">One model fits in 16 GB at a time</div></div></div>
            <div class="home-status">
              <span class="big"><span class="gdot" id="homeDot"></span><span id="homeModelText">No model loaded</span></span>
              <button class="stop hide" id="homeStop" onclick="globalStop()">Stop &amp; free VRAM</button>
            </div>
          </div>

          <div class="view-head" style="margin:24px 0 4px"><h1 style="font-size:15px">Jump in</h1></div>
          <div class="tiles" id="homeTiles"></div>

          <div class="view-head" style="margin:26px 0 4px"><h1 style="font-size:15px">How the studio works</h1></div>
          <div class="tiles">
            <div class="tile" style="cursor:default;--ico:var(--hue-home)">
              <span class="tico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="4" width="18" height="12" rx="2"/><path d="M7 20h10"/></svg></span>
              <b>One model at a time</b><span>Each tool loads its model on demand and frees whatever was loaded before — that's how everything fits in 16&nbsp;GB.</span>
            </div>
            <div class="tile" style="cursor:default;--ico:var(--hue-voice)">
              <span class="tico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="11" width="18" height="10" rx="2"/><path d="M7 11V7a5 5 0 0 1 10 0v4"/></svg></span>
              <b>Private by design</b><span>Generations live in memory only. The Save button is the single way anything is written to disk.</span>
            </div>
            <div class="tile" style="cursor:default;--ico:var(--hue-text)">
              <span class="tico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="9"/><path d="M2.5 12h19M12 3a14 14 0 0 1 0 18M12 3a14 14 0 0 0 0 18"/></svg></span>
              <b>Anywhere on your tailnet</b><span>Open the studio from any of your devices over Tailscale HTTPS — mic features need that secure URL.</span>
            </div>
          </div>
        </div>
      </section>

      <!-- ===================== LANGUAGE (chat) ===================== -->
      <section class="panel" data-panel="lang">
        <div class="chatwrap">
          <div class="hero chathead">
            <span class="hico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M4 7V5h16v2"/><path d="M9 19h6"/><path d="M12 5v14"/></svg></span>
            <div><h1>Language</h1><div class="tag">Chat with your local model — code, research &amp; vision</div></div>
            <div class="loadbar">
              <button class="load" id="langLoad" onclick="loadModel('lang')">Load model</button>
              <button class="stop hide" id="langStop" onclick="stopModel('lang')">Stop model</button>
              <span class="loadstat" id="langStat"><span class="dot"></span>not loaded</span>
            </div>
          </div>
          <div class="chatlog" id="llmOut">
            <div class="chatempty" id="llmEmpty">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="M21 11.5a8.4 8.4 0 0 1-8.5 8.3 8.9 8.9 0 0 1-3.7-.8L3 20l1.1-4.4a8.1 8.1 0 0 1-1.1-4.1A8.4 8.4 0 0 1 11.5 3a8.4 8.4 0 0 1 9.5 8.5z"/></svg>
              <b>Ask your local model anything</b>
              Replies never leave this machine. Recent turns are folded back in, so follow-ups keep their context.
              <div class="suggest">
                <button type="button" class="copybtn" onclick="chatSuggest(this)">Write a regex that matches UK postcodes</button>
                <button type="button" class="copybtn" onclick="chatSuggest(this)">Summarise the plot of Hamlet in 5 lines</button>
                <button type="button" class="copybtn" onclick="chatSuggest(this)">Explain VRAM like I'm five</button>
              </div>
            </div>
          </div>
          <fieldset id="langFs" disabled>
            <div class="composer">
              <div class="copts">
                <select id="llmTask" title="Changing task needs a reload — different model">
                  <option value="code">Code</option>
                  <option value="research">Research / writing</option>
                  <option value="vision">Vision (describe an image)</option>
                </select>
                <label class="chk" title="Use the uncensored / abliterated model (text only; vision is unaffected)"><input type="checkbox" id="llmUnlocked" onchange="reeval()"> 🔓 Unlocked</label>
                <span class="hide" id="llmImgWrap">
                  <label class="imgchip" style="cursor:pointer" title="Attach the image to describe">📎 <span id="llmImgName">attach image…</span>
                    <input type="file" id="llmImg" accept="image/*" style="display:none" onchange="chatImgPick()"></label>
                </span>
                <button type="button" class="copybtn" onclick="chatClear()" title="Start a fresh conversation">✚ New chat</button>
                <span class="status" id="llmStatus" style="margin-left:auto"></span>
              </div>
              <div class="crow">
                <textarea id="llmPrompt" placeholder="Ask for code, a summary, or (vision) a question about the attached image…"
                          onkeydown="if(event.key==='Enter'&&!event.shiftKey){event.preventDefault();runLLM();}"></textarea>
                <button class="sendbtn" id="llmSend" onclick="runLLM()" title="Send (Enter)">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round"><path d="M5 12h14M13 6l6 6-6 6"/></svg>
                </button>
              </div>
            </div>
          </fieldset>
        </div>
      </section>

      <!-- ===================== IMAGE: GENERATE ===================== -->
      <section class="panel" data-panel="gen">
        <div class="view view-wide">
          <div class="hero">
            <span class="hico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8.5" cy="8.5" r="1.6"/><path d="M21 15l-5-5L5 21"/></svg></span>
            <div><h1>Generate</h1><div class="tag">Text → image · FLUX.2 Klein, entirely on your GPU</div></div>
            <div class="loadbar">
              <button class="load" id="genLoad" onclick="loadModel('gen')">Load model</button>
              <button class="stop hide" id="genStop" onclick="stopModel('gen')">Stop model</button>
              <span class="loadstat" id="genStat"><span class="dot"></span>not loaded</span>
            </div>
          </div>
          <div class="ws">
            <div class="ws-ctl">
              <div class="card">
                <div class="field"><label>Model <span class="note">(load applies to Generate &amp; Edit)</span></label>
                  <select id="imgModel" onchange="reeval()"><option value="base4b">Base 4B (Apache, commercial-safe)</option><option value="miraclein9b">Miraclein 9B (uncensored · personal art)</option></select>
                </div>
                <fieldset id="genFs" disabled>
                  <div class="field"><label>Prompt</label>
                    <textarea id="imgPrompt" style="min-height:120px" placeholder="a renaissance oil painting of … , chiaroscuro, museum quality"></textarea>
                  </div>
                  <div class="field"><label>Aspect ratio</label>
                    <input type="hidden" id="imgSize" value="1:1">
                    <div class="aspects" id="imgAspects">
                      <button type="button" class="active" data-ar="1:1"><i style="width:18px;height:18px"></i>1:1</button>
                      <button type="button" data-ar="3:4"><i style="width:15px;height:20px"></i>3:4</button>
                      <button type="button" data-ar="4:3"><i style="width:20px;height:15px"></i>4:3</button>
                      <button type="button" data-ar="9:16"><i style="width:12px;height:21px"></i>9:16</button>
                      <button type="button" data-ar="16:9"><i style="width:22px;height:12px"></i>16:9</button>
                    </div>
                  </div>
                  <details class="adv"><summary>Advanced — steps, seed</summary>
                    <div class="adv-body"><div class="row">
                      <div class="field"><label>Steps</label><input type="number" id="imgSteps" value="20" min="1" max="50"></div>
                      <div class="field"><label>Seed (-1=rand)</label><input type="number" id="imgSeed" value="-1"></div>
                    </div></div>
                  </details>
                  <button class="go" id="imgGo" onclick="runImage()">✦ Generate (~25s)</button>
                  <div class="status" id="imgStatus"></div>
                </fieldset>
              </div>
            </div>
            <div class="ws-out">
              <div class="stage" id="imgStage">
                <div class="empty" id="imgEmpty">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8.5" cy="8.5" r="1.6"/><path d="M21 15l-5-5L5 21"/></svg>
                  <b>Your image appears here</b>
                  Describe what you want on the left, then Generate. Nothing is saved to disk until you click Save.
                </div>
                <img class="result hide" id="imgOut" alt="Generated image" onclick="lightbox(this)">
              </div>
              <div class="outbar">
                <div class="meta hide" id="imgMeta"></div>
                <span class="spacer"></span>
                <a id="imgDl" class="hide dlbtn" download="image.png">⬇ Save image</a>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- ===================== MUSIC: ACE-STEP ===================== -->
      <!-- ===================== COMPOSER (brief -> arranged & mixed song) ===================== -->
      <section class="panel" data-panel="composer">
        <div class="view view-wide">
          <div class="hero">
            <span class="hico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M4 20V9M9 20V4M14 20v-7M19 20v-11"/><circle cx="4" cy="20" r="1.4"/><circle cx="9" cy="20" r="1.4"/><circle cx="14" cy="20" r="1.4"/><circle cx="19" cy="20" r="1.4"/></svg></span>
            <div><h1>Composer</h1><div class="tag">Set it up, arrange it, export it — planned by your GPU's LLM, played by your CPU's sampler</div></div>
            <div class="loadbar">
              <button class="load" id="coLoad" onclick="loadModel('composer')">Load planner</button>
              <button class="stop hide" id="coStop" onclick="stopModel('composer')">Stop planner</button>
              <span class="loadstat" id="coStat"><span class="dot"></span>not loaded</span>
            </div>
          </div>

          <!-- steps: 1 set up -> 2 arrange -> 3 export. 2 and 3 unlock once a song
               exists, and stay freely clickable after that because editing is a
               loop between them, not a one-way march. -->
          <div class="wiz" id="coWiz">
            <button class="wiz-step on" id="coStepBtn1" onclick="coGoStep(1)">
              <span class="wiz-n">1</span><span><b>Set up</b><small>brief, style, sound</small></span></button>
            <span class="wiz-sep"></span>
            <button class="wiz-step lock" id="coStepBtn2" onclick="coGoStep(2)">
              <span class="wiz-n">2</span><span><b>Arrange</b><small>tracks, clips, notes</small></span></button>
            <span class="wiz-sep"></span>
            <button class="wiz-step lock" id="coStepBtn3" onclick="coGoStep(3)">
              <span class="wiz-n">3</span><span><b>Export</b><small>master, stems, MIDI</small></span></button>
            <span class="spacer" style="flex:1"></span>
            <span class="status" id="coStatus"></span>
          </div>
          <div id="coBarWrap" class="hide prog" style="width:100%;margin-bottom:12px"><div id="coBar"></div></div>

          <!-- ============ STEP 1 — set up ============ -->
          <div class="costep" id="coStep1">
            <div class="wiz-cols">
              <div class="card">
                <div class="field"><label>Brief <span class="note">— what the song should be. The planner picks instruments, key, tempo, structure, chords and mix from this.</span></label>
                  <textarea id="coBrief" style="min-height:70px" placeholder="a hopeful, cinematic instrumental about leaving a city at dawn — fragile piano start, huge strings-and-brass finish"></textarea>
                  <div class="tagchips" style="margin-top:5px">
                    <button type="button" class="copybtn" onclick="coSetBrief('a hopeful, cinematic instrumental about leaving a city at dawn — fragile start, huge finish','cinematic')">city at dawn</button>
                    <button type="button" class="copybtn" onclick="coSetBrief('a neon-lit late-night drive — pulsing bass, wide arpeggios, a chorus that opens right up','synth-pop')">neon drive</button>
                    <button type="button" class="copybtn" onclick="coSetBrief('a rainy-afternoon study loop — dusty keys, brushed drums, warm and unhurried','lo-fi')">rainy study</button>
                    <button type="button" class="copybtn" onclick="coSetBrief('a festival main-stage banger — huge build, hard drop, relentless energy','edm')">main stage</button>
                    <button type="button" class="copybtn" onclick="coSetBrief('a smoky late-set jazz number — walking bass, comping piano, sax taking the tune','jazz')">smoky jazz</button>
                    <button type="button" class="copybtn" onclick="coSetBrief('a slow-burn stadium rock instrumental — clean verses, twin guitars, soaring solo','rock')">stadium rock</button>
                  </div>
                </div>
                <div class="row">
                  <div class="field"><label>Style</label><select id="coGenre"></select></div>
                  <div class="field"><label>Length — <span id="coSecsLab">2:30</span></label>
                    <input type="range" id="coSecs" min="45" max="360" step="15" value="150" oninput="coSecsLabel()"></div>
                </div>
                <div class="row">
                  <div class="field"><label>Instruments — <span id="coTracksLab">6</span> <span class="note">tracks</span></label>
                    <input type="range" id="coTracks" min="3" max="9" step="1" value="6" oninput="$('coTracksLab').textContent=this.value"></div>
                  <div class="field"><label>Song name <span class="note">— folder under compositions/</span></label>
                    <input type="text" id="coName" placeholder="auto from the date"></div>
                </div>
                <details class="adv"><summary>Advanced — key, tempo, seed, planner</summary>
                  <div class="adv-body">
                    <div class="row">
                      <div class="field"><label>Key</label><select id="coKey">
                        <option value="">Planner decides</option>
                        <option>C major</option><option>C minor</option><option>D major</option><option>D minor</option><option>E major</option><option>E minor</option><option>F major</option><option>F minor</option><option>F# minor</option><option>G major</option><option>G minor</option><option>A major</option><option>A minor</option><option>Bb major</option><option>B minor</option><option>Eb major</option><option>Ab major</option>
                      </select></div>
                      <div class="field"><label>Tempo — <span id="coBpmLab">Planner decides</span></label>
                        <input type="range" id="coBpm" min="0" max="200" step="2" value="0" oninput="$('coBpmLab').textContent=(+this.value?this.value+' bpm':'Planner decides')"></div>
                      <div class="field"><label>Seed <span class="note">— -1 random</span></label>
                        <input type="number" id="coSeed" value="-1"></div>
                    </div>
                    <label class="chk"><input type="checkbox" id="coStems" checked> 🎚 Export per-instrument stems</label>
                    <label class="chk"><input type="checkbox" id="coUseLlm" checked> 🧠 Let the LLM plan it <span class="note">(off = style template, no model needed)</span></label>
                  </div>
                </details>
              </div>

              <div class="card">
                <div class="field" style="margin-bottom:2px"><label>Sound</label></div>
                <label class="chk chk-stack"><input type="checkbox" id="coHumanize" checked><span>🎻 Perform it, don't stamp it<span class="note">Within-note swells, strums &amp; rolls, drifting timing, drum round-robin. This is what stops it sounding like MIDI — untick to hear the raw quantized version.</span></span></label>
                <label class="chk chk-stack"><input type="checkbox" id="coPolish" onchange="coPolishToggle()"><span>✨ ACE-Step re-texture pass<span class="note">Re-renders the master with learned timbres. Less MIDI-sounding but softens transients; ships <b>alongside</b> the clean master. Swaps the GPU model.</span></span></label>
                <div class="field hide" id="coPolishWrap"><label>Polish strength — <span id="coPolLab">0.35</span></label>
                  <input type="range" id="coPolishDen" min="0.15" max="0.6" step="0.05" value="0.35" oninput="$('coPolLab').textContent=(+this.value).toFixed(2)"></div>

                <details class="adv" id="coLibDetails"><summary id="coLibSummary">Instrument library</summary>
                  <div class="adv-body" style="max-height:210px;overflow:auto">
                    <div id="coLibList" style="font-size:12px;line-height:1.65"></div>
                  </div>
                </details>

                <button class="go" id="coGo" onclick="coStart()" style="margin-top:auto">🎼 Compose song</button>
                <button class="rec hide" id="coCancel" onclick="coCancel()">■ Cancel</button>
              </div>
            </div>
          </div>

          <!-- ============ STEP 2 — arrange ============ -->
          <div class="costep hide" id="coStep2">
            <div class="coed-player card" id="coEdPlayer">
              <div class="pl-top">
                <b id="coEdNowPlaying">—</b>
                <span class="coed-note" style="margin-left:auto">edit below, then re-render to hear the change</span>
              </div>
              <audio controls id="coEdAudio"></audio>
            </div>
            <div class="coed" id="coEd">
              <div class="coed-bar">
                <b id="coEdTitle">Arrangement</b>
                <span class="coed-note" id="coEdInfo"></span>
                <span class="coed-dirty hide" id="coEdDirty">● unsaved edits</span>
                <span class="spacer" style="flex:1"></span>
                <button class="copybtn" onclick="coEdAddTrack()">+ track</button>
                <button class="copybtn" onclick="coEdAddSection()">+ section</button>
                <button class="copybtn" onclick="coEdReset()" title="Discard edits and reload the saved score">↺ revert</button>
                <button class="go" id="coEdRender" style="width:auto;padding:6px 14px" onclick="coEdRerender()">▶ Re-render</button>
              </div>
              <div class="coed-scroll"><div class="coed-grid" id="coEdGrid"></div></div>
              <div class="coed-insp" id="coEdInsp"></div>
            </div>
          </div>

          <!-- ============ STEP 3 — export ============ -->
          <div class="costep hide" id="coStep3">
            <div class="stage" id="coStage">
              <div class="empty" id="coEmpty">
                <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="M4 20V9M9 20V4M14 20v-7M19 20v-11"/></svg>
                <b>Your song appears here</b>
                Compose one in step 1 — you get the master, the producer's reasoning, every instrument's mix settings, the arrangement, the stems and the multitrack MIDI.
              </div>
              <div class="resultwrap hide" id="coOut" style="width:100%"></div>
            </div>
            <div class="outbar">
              <div class="meta" id="coMeta">Saved under <b>compositions/</b> — nothing leaves this machine.</div>
              <a class="dlbtn hide" id="coZip" href="#">⬇ Download everything</a>
            </div>
          </div>
        </div>
      </section>

      <section class="panel" data-panel="music">
        <div class="view view-wide">
          <div class="hero">
            <span class="hico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M9 18V5l12-2v13"/><circle cx="6" cy="18" r="3"/><circle cx="18" cy="16" r="3"/></svg></span>
            <div><h1>Music Generation</h1><div class="tag">Full songs &amp; instrumentals · ACE-Step 3.5B on your GPU</div></div>
            <div class="loadbar">
              <button class="load" id="muLoad" onclick="loadModel('music')">Load model</button>
              <button class="stop hide" id="muStop" onclick="stopModel('music')">Stop model</button>
              <span class="loadstat" id="muStat"><span class="dot"></span>not loaded</span>
            </div>
          </div>
          <div class="ws">
            <div class="ws-ctl">
              <div class="card">
            <fieldset id="muFs" disabled>
              <div class="field"><label>Style tags <span class="note">— genres, moods, instruments, tempo (comma separated)</span></label>
                <textarea id="muTags" style="min-height:64px" placeholder="synth-pop, electronic, 128 bpm, energetic, female vocals, catchy chorus"></textarea>
                <div class="tagchips">
                  <button type="button" class="copybtn" onclick="muAddTags('pop, upbeat, catchy melody, polished production')">pop</button>
                  <button type="button" class="copybtn" onclick="muAddTags('rock, electric guitar, driving drums, anthemic')">rock</button>
                  <button type="button" class="copybtn" onclick="muAddTags('edm, electronic, club, four on the floor, synth lead')">edm</button>
                  <button type="button" class="copybtn" onclick="muAddTags('hip hop, boom bap, punchy 808s, rap vocals')">hip hop</button>
                  <button type="button" class="copybtn" onclick="muAddTags('jazz, swing, saxophone, walking bass, smoky')">jazz</button>
                  <button type="button" class="copybtn" onclick="muAddTags('orchestral, cinematic, epic, strings, brass, percussion')">cinematic</button>
                  <button type="button" class="copybtn" onclick="muAddTags('acoustic folk, fingerpicked guitar, warm, intimate')">folk</button>
                  <button type="button" class="copybtn" onclick="muAddTags('heavy metal, distorted guitars, double kick, aggressive')">metal</button>
                  <button type="button" class="copybtn" onclick="muAddTags('lo-fi, chillhop, mellow, vinyl crackle, relaxed')">lo-fi</button>
                </div>
              </div>
              <label class="chk"><input type="checkbox" id="muInst" onchange="muInstChange()"> 🎹 Instrumental (no vocals / lyrics)</label>
              <div class="field" id="muLyricsWrap"><label>Lyrics <span class="note">— section tags steer the song; add a style with a hyphen, e.g. [chorus - anthemic] or [verse 2 - whispered] (keep modifiers short)</span></label>
                <textarea id="muLyrics" style="min-height:130px" placeholder="[verse]&#10;Neon lights across the rain-slick street&#10;…&#10;&#10;[chorus - anthemic]&#10;We run, we run, all night long"></textarea>
                <div class="tagchips">
                  <button type="button" class="copybtn" onclick="muInsert('[intro]')">+ [intro]</button>
                  <button type="button" class="copybtn" onclick="muInsert('[verse]')">+ [verse]</button>
                  <button type="button" class="copybtn" onclick="muInsert('[pre-chorus]')">+ [pre-chorus]</button>
                  <button type="button" class="copybtn" onclick="muInsert('[chorus]')">+ [chorus]</button>
                  <button type="button" class="copybtn" onclick="muInsert('[bridge]')">+ [bridge]</button>
                  <button type="button" class="copybtn" onclick="muInsert('[outro]')">+ [outro]</button>
                  <select id="muTagIns" style="width:150px;padding:4px 8px" onchange="if(this.value){muInsert(this.value);this.selectedIndex=0}">
                    <option value="">More tags…</option>
                    <optgroup label="Dynamics">
                      <option value="[build]">[build] — rising energy</option>
                      <option value="[drop]">[drop] — energy release</option>
                      <option value="[breakdown]">[breakdown] — stripped back</option>
                      <option value="[fade out]">[fade out] — fading ending</option>
                      <option value="[silence]">[silence] — silent beat</option>
                    </optgroup>
                    <optgroup label="Instrumental">
                      <option value="[instrumental]">[instrumental] — no vocals</option>
                      <option value="[guitar solo]">[guitar solo]</option>
                      <option value="[piano interlude]">[piano interlude]</option>
                      <option value="[drum break]">[drum break]</option>
                    </optgroup>
                    <optgroup label="Vocal style">
                      <option value="[whispered]">[whispered]</option>
                      <option value="[falsetto]">[falsetto]</option>
                      <option value="[raspy vocal]">[raspy vocal]</option>
                      <option value="[powerful belting]">[powerful belting]</option>
                      <option value="[spoken word]">[spoken word]</option>
                      <option value="[harmonies]">[harmonies]</option>
                      <option value="[call and response]">[call and response]</option>
                      <option value="[ad-lib]">[ad-lib]</option>
                    </optgroup>
                    <optgroup label="Energy / mood">
                      <option value="[high energy]">[high energy]</option>
                      <option value="[low energy]">[low energy]</option>
                      <option value="[building energy]">[building energy]</option>
                      <option value="[explosive]">[explosive]</option>
                    </optgroup>
                  </select>
                  <span class="note" style="margin-left:auto">Language <select id="muLang" style="width:110px;padding:4px 8px"><option value="en">English</option><option value="es">Spanish</option><option value="fr">French</option><option value="de">German</option><option value="it">Italian</option><option value="pt">Portuguese</option><option value="ja">Japanese</option><option value="ko">Korean</option><option value="zh">Chinese</option><option value="ru">Russian</option><option value="hi">Hindi</option><option value="ar">Arabic</option></select></span>
                </div>
              </div>
              <div class="field"><label>Duration — <span id="muSecsLab">2:00</span> <span class="note">(30 s – 4 min are the model's sweet spot)</span></label>
                <input type="range" id="muSecs" min="10" max="300" step="5" value="120" oninput="muSecsLabel()">
              </div>
              <div class="row">
                <div class="field"><label>BPM — <span id="muBpmLab">Auto</span> <span class="note">(slow 60–80 · mid 90–120 · fast 130–180)</span></label>
                  <input type="range" id="muBpm" min="0" max="220" step="5" value="0" oninput="$('muBpmLab').textContent=(+this.value?this.value:'Auto')">
                </div>
                <div class="field"><label>Key <span class="note">— common keys (C, G, D, Am, Em) are the most stable</span></label><select id="muKey">
                  <option value="">Auto (model decides)</option>
                  <option>C major</option><option>C minor</option><option>C# major</option><option>C# minor</option><option>D major</option><option>D minor</option><option>Eb major</option><option>Eb minor</option><option>E major</option><option>E minor</option><option>F major</option><option>F minor</option><option>F# major</option><option>F# minor</option><option>G major</option><option>G minor</option><option>Ab major</option><option>Ab minor</option><option>A major</option><option>A minor</option><option>Bb major</option><option>Bb minor</option><option>B major</option><option>B minor</option>
                </select></div>
              </div>
              <div class="row">
                <div class="field"><label>Variations</label><select id="muBatch"><option value="1">1 track</option><option value="2">2 tracks</option><option value="3">3 tracks</option><option value="4">4 tracks</option></select></div>
                <div class="field"><label>Format</label><select id="muFormat" onchange="muFormatChange()"><option value="mp3">MP3</option><option value="flac">FLAC (lossless)</option><option value="opus">Opus</option></select></div>
                <div class="field" id="muQualityWrap"><label>Quality</label><select id="muQuality"><option value="V0">V0 (best VBR)</option><option value="320k">320k</option><option value="128k">128k</option></select></div>
              </div>
              <details class="adv"><summary>Advanced — model, diffusion, sampler, song-planner, seed</summary>
                <div class="adv-body">
                  <div class="row">
                    <div class="field"><label>Model variant <span class="note">— turbo: cleaner vocals, no CFG harshness, ~6× faster (recommended). sft: follows the tags hardest, but CFG can roughen loud vocals</span></label><select id="muModel" onchange="muModelChange()">
                      <option value="turbo">turbo — smooth &amp; fast (recommended)</option>
                      <option value="sft">sft — max prompt adherence</option>
                    </select></div>
                  </div>
                  <div class="row">
                    <div class="field"><label>Steps — <span id="muStepsLab">8</span> <span class="note">refinement passes. turbo is distilled for 8 (more won't help); sft wants 50, under ~30 loses detail</span></label>
                      <input type="range" id="muSteps" min="4" max="100" step="1" value="8" oninput="$('muStepsLab').textContent=this.value">
                    </div>
                    <div class="field"><label>Guidance (CFG) — <span id="muCfgLab">1.0</span> <span class="note">sft only: how strictly it follows your tags (6–8 balanced). turbo ignores guidance — leave at 1</span></label>
                      <input type="range" id="muCfg" min="1" max="15" step="0.5" value="1" oninput="$('muCfgLab').textContent=(+this.value).toFixed(1)">
                    </div>
                  </div>
                  <div class="row">
                    <div class="field"><label>Shift — <span id="muShiftLab">3.0</span> <span class="note">denoising focus: higher = stronger song structure, lower = finer sonic detail. 3–6 is the stable band</span></label>
                      <input type="range" id="muShift" min="1" max="6" step="0.5" value="3" oninput="$('muShiftLab').textContent=(+this.value).toFixed(1)">
                    </div>
                    <div class="field"><label>Seed <span class="note">— -1 = random. Reuse a seed (shown after generating) to iterate on one song while tweaking settings</span></label>
                      <input type="number" id="muSeed" value="-1">
                    </div>
                  </div>
                  <div class="row">
                    <div class="field"><label>Sampler <span class="note">— how noise is removed each step</span></label><select id="muSampler">
                      <option value="euler">euler — official default, clean &amp; reliable</option>
                      <option value="er_sde">er_sde — smoother, fewer artifacts (community pick)</option>
                      <option value="euler_ancestral">euler_ancestral — adds variation per step</option>
                      <option value="heun">heun — slower, slightly more precise</option>
                      <option value="dpmpp_2m">dpmpp_2m</option><option value="dpmpp_2m_sde">dpmpp_2m_sde</option><option value="dpmpp_3m_sde">dpmpp_3m_sde</option><option value="uni_pc">uni_pc</option><option value="res_multistep">res_multistep</option><option value="deis">deis</option><option value="ddim">ddim</option>
                    </select></div>
                    <div class="field"><label>Scheduler <span class="note">— noise-removal timing curve</span></label><select id="muScheduler">
                      <option value="simple">simple — official default</option>
                      <option value="linear_quadratic">linear_quadratic — steadier rhythm, fewer crackles (community pick)</option>
                      <option value="normal">normal</option><option value="karras">karras</option><option value="exponential">exponential</option><option value="sgm_uniform">sgm_uniform</option><option value="beta">beta</option><option value="kl_optimal">kl_optimal</option>
                    </select></div>
                  </div>
                  <div class="row">
                    <div class="field"><label>Planner creativity — <span id="muLmTempLab">0.85</span> <span class="note">the song-planner LM's temperature. 0.85 official; lower = safer/more predictable songs, higher = more adventurous (can get messy past ~1.1)</span></label>
                      <input type="range" id="muLmTemp" min="0.5" max="1.2" step="0.05" value="0.85" oninput="$('muLmTempLab').textContent=(+this.value).toFixed(2)">
                    </div>
                    <div class="field"><label>Planner guidance — <span id="muLmCfgLab">2.0</span> <span class="note">how strongly the planner sticks to your tags/lyrics. 2.0 official; 1 = off (faster, looser), 3–4 = very literal</span></label>
                      <input type="range" id="muLmCfg" min="1" max="4" step="0.25" value="2" oninput="$('muLmCfgLab').textContent=(+this.value).toFixed(2)">
                    </div>
                  </div>
                  <div class="row">
                    <div class="field"><label>Time signature <span class="note">— 4/4 is the most reliable; 3/4 waltz and 6/8 swing are generally stable</span></label><select id="muTimeSig">
                      <option value="">Auto (model decides)</option>
                      <option value="4">4/4</option><option value="3">3/4</option><option value="6">6/8</option><option value="2">2/4</option>
                    </select></div>
                  </div>
                </div>
              </details>
              <details class="adv"><summary>Remix an existing track (audio → audio)</summary>
                <div class="adv-body">
                  <div class="field"><label>Source track <span class="note">— its length sets the output length</span></label>
                    <label class="filedrop"><input type="file" id="muSrc" accept="audio/*"><span class="filedrop-name"></span></label>
                  </div>
                  <div class="field"><label>Denoise — <span id="muDenLab">0.70</span> <span class="note">(lower = keep more of the original)</span></label>
                    <input type="range" id="muDenoise" min="0.1" max="1" step="0.05" value="0.7" oninput="$('muDenLab').textContent=(+this.value).toFixed(2)">
                  </div>
                </div>
              </details>
              <button class="go" id="muGo" onclick="runMusic()">♪ Generate music</button>
              <div class="status" id="muStatus"></div>
            </fieldset>
              </div>
            </div>
            <div class="ws-out">
              <div class="stage" id="muStage">
                <div class="empty" id="muEmpty">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="M9 18V5l12-2v13"/><circle cx="6" cy="18" r="3"/><circle cx="18" cy="16" r="3"/></svg>
                  <b>Your tracks appear here</b>
                  Set a style, add lyrics or go instrumental, and Generate. Each track streams straight from your GPU — nothing touches disk unless you Save.
                </div>
                <div class="resultwrap" id="muOut" style="width:100%"></div>
              </div>
              <div class="outbar">
                <div class="meta" id="muNote">Tracks aren't saved to disk — use <b>Save track</b> to keep one.</div>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- ===================== LULLABY (song -> piano lullaby) ===================== -->
      <section class="panel" data-panel="lullaby">
        <div class="view view-wide">
          <div class="hero">
            <span class="hico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 21a9 9 0 1 1 9-9c0 1-1 2-2.5 2S16 13 16 12a4 4 0 1 0-4 4"/></svg></span>
            <div><h1>Lullaby</h1><div class="tag">Turn any song into a soft solo-piano lullaby instrumental</div></div>
          </div>
          <div class="ws">
            <div class="ws-ctl">
              <div class="card">
                <div class="field"><label>Song file <span class="note">(mp3 / wav / mp4 — audio is extracted automatically)</span></label>
                  <label class="filedrop"><input type="file" id="lbFile" accept="audio/*,video/mp4" onchange="lbFileChange()"><span class="filedrop-name"></span></label>
                </div>
                <button class="go" id="lbAnalyzeBtn" onclick="lbAnalyze()">🔍 Analyze song</button>
                <div id="lbWorkbench" class="hide">
                  <div class="field"><label>Song map <span class="note" id="lbSummary"></span></label>
                    <canvas id="lbWave" height="86" style="width:100%;border-radius:8px;background:rgba(255,255,255,.04)"></canvas>
                  </div>
                  <div class="field"><label>Tracks <span class="note">(listen to each — ticked tracks carry into the lullaby at their slider level; the Piano engine's melody is transcribed straight from this selection too)</span></label>
                    <button type="button" class="rec" style="align-self:flex-start" onclick="playAllSelected('#lbStems', this)">▶ Play all selected</button>
                    <div id="lbStems" style="display:flex;flex-direction:column;gap:8px"></div>
                  </div>
                </div>
                <div class="field"><label>Engine</label>
                  <div class="seg" id="lbModeSeg">
                    <button type="button" class="active" data-mode="remix" onclick="lbSetMode('remix')">Remix — close to the original</button>
                    <button type="button" data-mode="piano" onclick="lbSetMode('piano')">Piano — rebuilt arrangement</button>
                    <button type="button" data-mode="melody-match" onclick="lbSetMode('melody-match')">Melody Match — closest to the sung tune</button>
                  </div>
                </div>
                <div id="lbRemixCtl">
                  <div class="field"><label>Follow</label>
                    <select id="lbFocus">
                      <option value="both">Whole song — vocals + instruments equally (recommended)</option>
                      <option value="vocals">Vocal melody — instruments turned down in what ACE hears</option>
                      <option value="instruments">Instruments — vocals turned down in what ACE hears</option>
                    </select>
                  </div>
                  <div class="field"><label>Re-imagination</label>
                    <div class="row">
                      <div style="flex:1"><span class="note">Denoise <b id="lbSoftVal">0.60</b> — higher drifts further from the original but gets dreamier</span>
                        <input type="range" id="lbSoft" min="0.45" max="0.75" step="0.01" value="0.60" style="width:100%"
                               oninput="document.getElementById('lbSoftVal').textContent=(+this.value).toFixed(2)"></div>
                    </div>
                  </div>
                  <div class="field"><label>Slowdown</label>
                    <div class="row">
                      <div style="flex:1"><span class="note">Tempo <b id="lbSlowVal">100%</b> of original (85-90% = sleepier)</span>
                        <input type="range" id="lbSlow" min="0.8" max="1" step="0.01" value="1" style="width:100%"
                               oninput="document.getElementById('lbSlowVal').textContent=Math.round(this.value*100)+'%'"></div>
                    </div>
                  </div>
                  <div class="meta">Drums and bass are removed and the song's dynamics are deeply flattened so it
                    stays soft the whole way through, then ACE-Step re-imagines it as a hushed felt-piano
                    lullaby and a dark, quiet mastering chain finishes it. Sounds closest to the original song.</div>
                </div>
                <div id="lbPianoCtl" class="hide">
                  <div class="field"><label>Slowdown</label>
                    <div class="row">
                      <div style="flex:1"><span class="note">Tempo <b id="lbTempoVal">72%</b> of original</span>
                        <input type="range" id="lbTempo" min="0.55" max="0.95" step="0.01" value="0.72" style="width:100%"
                               oninput="document.getElementById('lbTempoVal').textContent=Math.round(this.value*100)+'%'"></div>
                    </div>
                  </div>
                  <div class="field"><label style="display:flex;align-items:center;gap:8px;cursor:pointer">
                    <input type="checkbox" id="lbPolish" style="width:auto">
                    ACE-Step polish <span class="note">(experimental — can wander off the arrangement)</span></label>
                  </div>
                  <div class="meta">The vocal melody is transcribed, key &amp; chords are detected, and the song is
                    rebuilt as a rocking piano + music-box arrangement at true lullaby tempo, rendered with a
                    real sampled grand piano. A different, more storybook sound.</div>
                </div>
                <div id="lbMelodyCtl" class="hide">
                  <div class="field"><label>Instrument</label>
                    <select id="lbInstrument">
                      <option value="cello" selected>Cello — warm, closest overall match</option>
                      <option value="violin">Violin</option>
                      <option value="flute">Flute</option>
                      <option value="synth_voice">Synth Voice — most precise, less "instrumental"</option>
                      <option value="music_box">Music Box</option>
                    </select>
                  </div>
                  <div class="field"><label>Slowdown</label>
                    <div class="row">
                      <div style="flex:1"><span class="note">Tempo <b id="lbMelodyTempoVal">85%</b> of original</span>
                        <input type="range" id="lbMelodyTempo" min="0.55" max="1" step="0.01" value="0.85" style="width:100%"
                               oninput="document.getElementById('lbMelodyTempoVal').textContent=Math.round(this.value*100)+'%'"></div>
                    </div>
                  </div>
                  <div class="field"><label style="display:flex;align-items:center;gap:8px;cursor:pointer">
                    <input type="checkbox" id="lbMelodyPolish" style="width:auto">
                    ACE-Step polish <span class="note">(experimental — re-textures the render, can wander off the melody)</span></label>
                  </div>
                  <div class="meta">Traces the singer's actual pitch curve directly (no scale-snapping or
                    rhythm quantization — the tune it plays is the tune that was sung, glides and all) and
                    plays it on one instrument capable of sliding between notes like a voice does. Each
                    ticked track above can be routed independently via its <b>Route</b> selector — traced
                    onto the instrument, or rebuilt as a full Piano-style arrangement instead — and both are
                    mixed together, so e.g. vocals can follow the instrument while an existing piano part
                    gets its own rebuilt accompaniment.</div>
                </div>
                <button class="go hide" id="lbRenderBtn" onclick="lbRender()">🎹 Render lullaby</button>
                <div class="status" id="lbStatus"></div>
              </div>
            </div>
            <div class="ws-out">
              <div class="stage" id="lbStage" style="min-height:300px;justify-content:flex-start">
                <div class="empty" id="lbEmpty">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="M12 21a9 9 0 1 1 9-9c0 1-1 2-2.5 2S16 13 16 12a4 4 0 1 0-4 4"/></svg>
                  <b>Your lullaby appears here</b>
                  Upload a song and press Make lullaby — you'll get an MP3 and a lossless WAV.
                </div>
                <div id="lbBarWrap" class="hide prog" style="width:100%"><div id="lbBar"></div></div>
                <div id="lbFiles" style="width:100%;display:flex;flex-direction:column;gap:12px"></div>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- ===================== TRACK SPLITTER ===================== -->
      <section class="panel" data-panel="split">
        <div class="view view-wide">
          <div class="hero">
            <span class="hico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M6 3v18M18 3v18"/><path d="M9 7h6M9 12h6M9 17h6"/></svg></span>
            <div><h1>Track Splitter</h1><div class="tag">Split any song into vocals, guitar, piano, bass, drums &amp; other — and save them</div></div>
          </div>
          <div class="ws">
            <div class="ws-ctl">
              <div class="card">
                <div class="field"><label>Song file <span class="note">(mp3 / wav / mp4 — audio is extracted automatically)</span></label>
                  <label class="filedrop"><input type="file" id="spFile" accept="audio/*,video/mp4" onchange="spFileChange()"><span class="filedrop-name"></span></label>
                </div>
                <div class="field"><label>Save as</label>
                  <div class="seg" id="spFormatSeg">
                    <button type="button" class="active" data-fmt="mp3" onclick="spSetFormat('mp3')">MP3 — smaller files</button>
                    <button type="button" data-fmt="wav" onclick="spSetFormat('wav')">WAV — lossless</button>
                  </div>
                </div>
                <button class="go" id="spGoBtn" onclick="spSplit()">✂️ Split into tracks</button>
                <div class="status" id="splStatus"></div>
                <div class="field" style="margin-top:4px"><label>Previously split</label>
                  <div class="row" style="align-items:flex-end">
                    <select id="spLibrary" onchange="spOpenLibrary()"><option value="">— none yet —</option></select>
                    <button type="button" class="rec" style="flex:0 0 auto" onclick="spRefreshLibrary()" title="Refresh">↻</button>
                  </div>
                </div>
                <div class="meta">Runs the same two-pass separation as the Lullaby tab (Demucs htdemucs_ft for
                  clean vocals/drums/bass/other, htdemucs_6s for guitar/piano) — no arrangement, just the raw
                  tracks, saved to disk for you to use anywhere.</div>
              </div>
            </div>
            <div class="ws-out">
              <div class="stage" id="splStage" style="min-height:300px;justify-content:flex-start">
                <div class="empty" id="splEmpty">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="M6 3v18M18 3v18"/><path d="M9 7h6M9 12h6M9 17h6"/></svg>
                  <b>Your split tracks appear here</b>
                  Upload a song and press Split — each track gets its own scrubbable player and download link.
                </div>
                <div id="splBarWrap" class="hide prog" style="width:100%"><div id="splBar"></div></div>
                <a id="spZipBtn" class="go hide" style="text-decoration:none;text-align:center" download>⬇ Download all (.zip)</a>
                <button type="button" id="spPlayAllBtn" class="rec hide" onclick="playAllSelected('#spTracks', this)">▶ Play all selected</button>
                <div id="spTracks" style="width:100%;display:flex;flex-direction:column;gap:10px"></div>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- ===================== IMAGE: EDIT ===================== -->
      <section class="panel" data-panel="edit">
        <div class="view view-wide">
          <div class="hero">
            <span class="hico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 20h9"/><path d="M16.5 3.5a2.1 2.1 0 0 1 3 3L7 19l-4 1 1-4z"/></svg></span>
            <div><h1>Edit</h1><div class="tag">Remove · reframe · recompose — reference-guided editing</div></div>
            <div class="loadbar">
              <button class="load" id="editLoad" onclick="loadModel('edit')">Load model</button>
              <button class="stop hide" id="editStop" onclick="stopModel('edit')">Stop model</button>
              <span class="loadstat" id="editStat"><span class="dot"></span>not loaded</span>
            </div>
          </div>
          <div class="ws">
            <div class="ws-ctl">
              <div class="card">
                <div class="field"><label>Model</label>
                  <select id="edModel" onchange="reeval()"><option value="base4b">Base 4B (Apache, commercial-safe)</option><option value="miraclein9b">Miraclein 9B (uncensored · personal art)</option></select>
                </div>
                <fieldset id="editFs" disabled>
                  <div class="field"><label>Source image(s) — up to 4 (multi-reference)</label>
                    <label class="filedrop"><input type="file" id="edImgs" accept="image/*" multiple onchange="edPreview()"><span class="filedrop-name"></span></label>
                    <div class="thumbs" id="edThumbs"></div>
                  </div>
                  <div class="field"><label>Instruction — describe the change</label>
                    <textarea id="edPrompt" placeholder="e.g. remove the person on the left and fill the background · widen the shot · replace the background with a forest"></textarea>
                  </div>
                  <div class="row">
                    <div class="field"><label>Reframe</label><select id="edSize"><option value="">Keep original size</option><option value="1:1">1:1</option><option value="3:4">3:4</option><option value="4:3">4:3</option><option value="9:16">9:16</option><option value="16:9">16:9</option></select></div>
                    <div class="field"><label>Edit strength</label><select id="edCfg"><option value="1.5">High fidelity</option><option value="2.5">Balanced</option><option value="3.5">Strong</option></select></div>
                  </div>
                  <details class="adv"><summary>Advanced — steps, seed</summary>
                    <div class="adv-body"><div class="row">
                      <div class="field"><label>Steps</label><input type="number" id="edSteps" value="24" min="1" max="50"></div>
                      <div class="field"><label>Seed (-1=rand)</label><input type="number" id="edSeed" value="-1"></div>
                    </div></div>
                  </details>
                  <div class="note">High fidelity for removal/reframe; raise strength for bigger changes.</div>
                  <button class="go" id="edGo" onclick="runEdit()">✦ Apply edit (~30s)</button>
                  <div class="status" id="edStatus"></div>
                </fieldset>
              </div>
            </div>
            <div class="ws-out">
              <div class="stage" id="edStage">
                <div class="empty" id="edEmpty">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="M12 20h9"/><path d="M16.5 3.5a2.1 2.1 0 0 1 3 3L7 19l-4 1 1-4z"/></svg>
                  <b>Your edit appears here</b>
                  Drop in a photo, describe the change, and apply. Compare Before / After when it's done.
                </div>
                <img class="result hide" id="edOut" alt="Edited image" onclick="lightbox(this)">
              </div>
              <div class="outbar">
                <div class="seg hide" id="edBA">
                  <button type="button" onclick="edShow('before')">Before</button>
                  <button type="button" class="active" onclick="edShow('after')">After</button>
                </div>
                <div class="meta hide" id="edMeta"></div>
                <span class="spacer"></span>
                <a id="edDl" class="hide dlbtn" download="image.png">⬇ Save image</a>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- ===================== SPRITE STUDIO ===================== -->
      <section class="panel" data-panel="sprite">
        <div class="view view-wide">
          <div class="hero">
            <span class="hico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="7" height="7" rx="1"/><rect x="14" y="3" width="7" height="7" rx="1"/><rect x="3" y="14" width="7" height="7" rx="1"/><rect x="14" y="14" width="7" height="7" rx="1"/></svg></span>
            <div><h1>Sprite Studio</h1><div class="tag">One reference image → style-matched 2D game sprites &amp; sheets</div></div>
            <div class="loadbar">
              <button class="load" id="spLoad" onclick="loadModel('sprite')">Load model</button>
              <button class="stop hide" id="spStop" onclick="stopModel('sprite')">Stop model</button>
              <span class="loadstat" id="spStat"><span class="dot"></span>not loaded</span>
            </div>
          </div>
          <div class="ws">
            <div class="ws-ctl">
              <div class="card">
                <div class="field"><label>Model <span class="note">(shared with Generate &amp; Edit)</span></label>
                  <select id="spModel" onchange="reeval()"><option value="base4b">Base 4B (Apache, commercial-safe)</option><option value="miraclein9b">Miraclein 9B (uncensored · personal art)</option></select>
                </div>
                <fieldset id="spFs" disabled>
                  <div class="field"><label>Reference image — defines the character &amp; style</label>
                    <label class="filedrop"><input type="file" id="spRef" accept="image/*" onchange="spRefPreview()"><span class="filedrop-name"></span></label>
                    <div class="thumbs" id="spRefThumb"></div>
                  </div>
                  <div class="field"><label>Character description <span class="note">(optional — improves consistency)</span></label>
                    <input type="text" id="spDesc" placeholder="e.g. a small knight in blue armor with a round wooden shield">
                  </div>
                  <div class="field"><label>What to generate</label><br>
                    <div class="seg" id="spModeSeg">
                      <button type="button" class="active" data-mode="single" onclick="spSetMode('single')">Single action</button>
                      <button type="button" data-mode="set" onclick="spSetMode('set')">Full sprite set</button>
                    </div>
                  </div>
                  <div id="spSingle">
                    <div class="row">
                      <div class="field"><label>Action</label><select id="spAction" onchange="spActionChange()"></select></div>
                      <div class="field"><label>Frames <span class="note">(blank = default)</span></label>
                        <input type="number" id="spFrames" min="2" max="12" placeholder="auto" oninput="spEstimate()"></div>
                    </div>
                    <div class="field hide" id="spCustomWrap"><label>Custom action — describe the motion</label>
                      <input type="text" id="spCustom" placeholder="e.g. swinging a fishing rod · climbing a ladder · casting a spell"></div>
                  </div>
                  <div id="spSet" class="hide">
                    <div class="field"><label>Actions in the set</label>
                      <div class="spchecks" id="spSetActions"></div>
                    </div>
                  </div>
                  <div class="row">
                    <div class="field"><label>Perspective</label>
                      <select id="spView">
                        <option value="side">Side view (platformer)</option>
                        <option value="front">Front view</option>
                        <option value="topdown">Top-down (RPG)</option>
                        <option value="34">3/4 view</option>
                      </select></div>
                    <div class="field"><label>Frame size</label>
                      <select id="spSize"><option value="128">128 px</option><option value="256" selected>256 px</option><option value="512">512 px</option><option value="1024">1024 px (native)</option></select></div>
                  </div>
                  <div class="field"><label class="chk"><input type="checkbox" id="spCut" checked> Transparent background (rembg cutout)</label></div>
                  <div class="field"><label>Set name <span class="note">(saved under sprites/ — blank = auto)</span></label>
                    <input type="text" id="spName" placeholder="e.g. blue-knight"></div>
                  <details class="adv"><summary>Advanced — steps, style fidelity, seed</summary>
                    <div class="adv-body">
                      <div class="row">
                        <div class="field"><label>Steps</label><input type="number" id="spSteps" value="24" min="4" max="50"></div>
                        <div class="field"><label>Seed (-1=rand)</label><input type="number" id="spSeed" value="-1"></div>
                      </div>
                      <div class="field"><label>Style fidelity</label>
                        <select id="spCfg"><option value="2">Closest to reference</option><option value="2.5" selected>Balanced</option><option value="3.5">Strong poses</option></select></div>
                    </div>
                  </details>
                  <div class="note" id="spEstimate"></div>
                  <button class="go" id="spGo" onclick="runSprites()">✦ Generate sprites</button>
                  <button class="stop hide" id="spCancel" type="button" onclick="cancelSprites()">✕ Cancel after this frame</button>
                  <div id="spBarWrap" class="hide prog" style="width:100%"><div id="spBar"></div></div>
                  <div class="status" id="spStatus"></div>
                </fieldset>
              </div>
            </div>
            <div class="ws-out">
              <div class="stage" id="spStage" style="min-height:300px">
                <div class="empty" id="spEmpty">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="7" height="7" rx="1"/><rect x="14" y="3" width="7" height="7" rx="1"/><rect x="3" y="14" width="7" height="7" rx="1"/><rect x="14" y="14" width="7" height="7" rx="1"/></svg>
                  <b>Your sprites appear here</b>
                  Drop in one character image, pick an action or a full set, and Generate. Frames appear as they finish; the checkerboard shows transparency.
                </div>
                <div class="out hide" id="spOut" style="width:100%;border:0;background:transparent"></div>
              </div>
              <div class="outbar">
                <div class="meta hide" id="spMeta"></div>
                <span class="spacer"></span>
                <a id="spZip" class="hide dlbtn">⬇ Download set (.zip)</a>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- ===================== STT ===================== -->
      <section class="panel" data-panel="stt">
        <div class="view view-wide">
          <div class="hero" style="--tone:var(--hue-voice)">
            <span class="hico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="9" y="2" width="6" height="12" rx="3"/><path d="M5 11a7 7 0 0 0 14 0"/><path d="M12 18v3"/></svg></span>
            <div><h1>Speech → Text</h1><div class="tag">Transcribe audio or your mic · Parakeet-TDT</div></div>
            <div class="loadbar">
              <button class="load" id="sttLoad" onclick="loadModel('stt')">Load model</button>
              <button class="stop hide" id="sttStop" onclick="stopModel('stt')">Stop model</button>
              <span class="loadstat" id="sttStat"><span class="dot"></span>not loaded</span>
            </div>
          </div>
          <div class="ws">
            <div class="ws-ctl">
              <div class="card">
                <fieldset id="sttFs" disabled>
                  <div class="field"><label>Audio file (wav / mp3 / m4a)</label>
                    <label class="filedrop"><input type="file" id="sttFile" accept="audio/*" onchange="sttFilePick()"><span class="filedrop-name"></span></label>
                  </div>
                  <div class="field"><label>…or record your mic <span class="note">(needs HTTPS / localhost)</span></label>
                    <button type="button" class="rec" id="sttRec" onclick="vsRec('stt')">🎙 Record</button>
                    <audio id="sttPrev" class="hide" controls></audio>
                  </div>
                  <button class="go" id="sttGo" onclick="runSTT()">Transcribe</button>
                  <div class="status" id="sttStatus"></div>
                </fieldset>
              </div>
            </div>
            <div class="ws-out">
              <div class="stage" id="sttStage" style="--tone:var(--hue-voice);min-height:300px">
                <div class="empty" id="sttEmpty">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><rect x="9" y="2" width="6" height="12" rx="3"/><path d="M5 11a7 7 0 0 0 14 0"/><path d="M12 18v3"/></svg>
                  <b>Your transcript appears here</b>
                  Drop in a recording or capture your mic, then Transcribe.
                </div>
                <div class="out hide" id="sttOut" style="width:100%;border:0;background:transparent;font-size:15px;line-height:1.75"></div>
              </div>
              <div class="outbar">
                <div class="meta hide" id="sttMeta"></div>
                <span class="spacer"></span>
                <button type="button" class="copybtn hide" id="sttCopy" onclick="copyText($('sttOut').textContent,this)">⧉ Copy transcript</button>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- ===================== TTS ===================== -->
      <section class="panel" data-panel="tts">
        <div class="view view-wide">
          <div class="hero">
            <span class="hico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M11 5 6 9H2v6h4l5 4z"/><path d="M15.5 8.5a5 5 0 0 1 0 7"/><path d="M19 5a9 9 0 0 1 0 14"/></svg></span>
            <div><h1>Text → Speech</h1><div class="tag">54 Kokoro narrators · Chatterbox voice cloning</div></div>
            <div class="loadbar">
              <button class="load" id="ttsLoad" onclick="loadModel('tts')">Load model</button>
              <button class="stop hide" id="ttsStop" onclick="stopModel('tts')">Stop model</button>
              <span class="loadstat" id="ttsStat"><span class="dot"></span>not loaded</span>
            </div>
          </div>
          <div class="ws">
            <div class="ws-ctl">
              <div class="card">
                <div class="field"><label>Engine <span class="note">(changing this needs a reload)</span></label>
                  <select id="ttsEngine" onchange="ttsEngineChange();reeval()"><option value="kokoro">Kokoro (fast)</option><option value="chatterbox">Chatterbox (cloning)</option></select>
                </div>
                <fieldset id="ttsFs" disabled>
                  <div class="field"><label>Text</label><textarea id="ttsText" style="min-height:110px" placeholder="The line to speak…"></textarea></div>
                  <div class="field" id="ttsVoiceWrap"><label>Voice</label><select id="ttsVoice"></select></div>
                  <div class="field hide" id="ttsRefWrap"><label>Reference voice clip (5–15s wav) — required for Chatterbox</label>
                    <label class="filedrop"><input type="file" id="ttsRef" accept="audio/*"><span class="filedrop-name"></span></label>
                  </div>
                  <button class="go" id="ttsGo" onclick="runTTS()">Synthesize</button>
                  <div class="status" id="ttsStatus"></div>
                </fieldset>
              </div>
            </div>
            <div class="ws-out">
              <div class="stage" id="ttsStage" style="min-height:300px">
                <div class="empty" id="ttsEmpty">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="M11 5 6 9H2v6h4l5 4z"/><path d="M15.5 8.5a5 5 0 0 1 0 7"/><path d="M19 5a9 9 0 0 1 0 14"/></svg>
                  <b>Your audio appears here</b>
                  Type a line, pick a voice, and Synthesize.
                </div>
                <audio class="hide" id="ttsOut" controls style="width:92%"></audio>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- ===================== VOICE STUDIO ===================== -->
      <section class="panel" data-panel="voicestudio">
        <div class="view view-wide">
          <div class="hero">
            <span class="hico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 2a3 3 0 0 1 3 3v6a3 3 0 0 1-6 0V5a3 3 0 0 1 3-3z"/><path d="M19 10v1a7 7 0 0 1-14 0v-1"/><path d="M12 18v4M8 22h8"/></svg></span>
            <div><h1>Voice Studio</h1><div class="tag">Train a model of your own voice, then make it say anything</div></div>
            <div class="seg" style="margin-left:auto">
              <button type="button" class="active" id="vsModeUse" onclick="$('vsMode').value='use';vsModeChange()">Use a voice</button>
              <button type="button" id="vsModeCreate" onclick="$('vsMode').value='create';vsModeChange()">＋ Create a voice</button>
            </div>
            <select id="vsMode" class="hide" onchange="vsModeChange()">
              <option value="use">Use a voice</option>
              <option value="create">Create a new voice (fine-tune)</option>
            </select>
          </div>

          <!-- USE A VOICE -->
          <div id="vsUse">
            <div class="ws">
              <div class="ws-ctl">
                <div class="card">
                  <div class="field"><label>Voice model</label>
                    <div class="row" style="align-items:flex-end">
                      <select id="vsVoice" onchange="reeval()"></select>
                      <button type="button" class="rec" style="flex:0 0 auto" onclick="loadVoices()" title="Refresh voice list">↻</button>
                    </div>
                  </div>
                  <div class="loadbar">
                    <button class="load" id="vmLoad" onclick="loadModel('voicemodel')">Load model</button>
                    <button class="stop hide" id="vmStop" onclick="stopModel('voicemodel')">Stop model</button>
                    <span class="loadstat" id="vmStat"><span class="dot"></span>not loaded</span>
                  </div>
                  <fieldset id="vmFs" disabled>
                    <div class="field"><label>Input</label>
                      <select id="vmInput" onchange="vmInputChange()">
                        <option value="text">Type text</option>
                        <option value="voice">Speak / upload audio (auto-transcribed, re-voiced)</option>
                      </select>
                    </div>
                    <div class="field" id="vmTextWrap"><label>Text</label><textarea id="vmText" placeholder="What should your voice say…"></textarea></div>
                    <div class="field hide" id="vmVoiceWrap">
                      <label>Audio in — record or upload (your words, spoken back in this voice)</label>
                      <div class="row">
                        <button type="button" class="rec" id="vmRec" onclick="vsRec('vm')">🎙 Record</button>
                        <label class="filedrop"><input type="file" id="vmFile" accept="audio/*" onchange="vsPick('vm')"><span class="filedrop-name"></span></label>
                      </div>
                      <audio id="vmPrev" class="hide" controls></audio>
                    </div>
                    <button class="go" onclick="runVoiceModel()">Speak</button>
                    <div class="status" id="vmRunStat"></div>
                  </fieldset>
                </div>
              </div>
              <div class="ws-out">
                <div class="stage" style="min-height:300px">
                  <div class="empty" id="vmEmpty">
                    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="M12 2a3 3 0 0 1 3 3v6a3 3 0 0 1-6 0V5a3 3 0 0 1 3-3z"/><path d="M19 10v1a7 7 0 0 1-14 0v-1"/><path d="M12 18v4M8 22h8"/></svg>
                    <b>Your voice appears here</b>
                    Pick a trained voice, load it, and give it words — typed or spoken.
                  </div>
                  <div class="out hide" id="vmHeard" style="border:0;background:transparent"></div>
                  <audio id="vmOut" class="hide" controls style="width:92%"></audio>
                </div>
                <div class="outbar"><span class="spacer"></span><a id="vmDl" class="hide dlbtn" download="voice.wav">⬇ Download .wav</a></div>
              </div>
            </div>
          </div>

          <!-- CREATE A VOICE (wizard) -->
          <div id="vsCreate" class="hide">
            <div class="view-narrow">
              <div class="card">
                <div class="wizhead">
                  <div class="wizring" id="vcRing" style="--p:0"><span id="vcRingText">0%</span></div>
                  <div><h2 style="font-size:15.5px">Record your training set</h2>
                    <div class="note">Read each short sentence aloud. Quiet room, steady mic ~15&nbsp;cm away, normal voice. Mic needs the HTTPS (Tailscale) or localhost URL.</div></div>
                </div>
                <div class="row">
                  <div class="field"><label>Voice name</label><input id="vcName" placeholder="e.g. my-voice"></div>
                  <div class="field"><label>How much to record</label>
                    <select id="vcAmount" onchange="vcAmountChange()">
                      <option value="15">Minimal — 15 sentences (~3 min)</option>
                      <option value="30" selected>Solid — 30 sentences (~6 min, recommended)</option>
                      <option value="40">Maximum — 40 sentences (best quality)</option>
                    </select>
                  </div>
                </div>
                <div id="vcRecordArea">
                  <div class="field"><label>📖 Read this aloud</label><div class="scriptcard" id="vcSentence"></div></div>
                  <div class="row" style="margin-top:12px">
                    <button type="button" class="rec" id="vcRecBtn" onclick="vsRec('train')">🎙 Record</button>
                    <button type="button" class="rec" id="vcRedoBtn" onclick="vcShow()" disabled>↺ Redo</button>
                    <button type="button" class="go" id="vcNextBtn" onclick="vcAccept()" disabled>Save &amp; next ▶</button>
                  </div>
                  <audio id="vcPrev" class="hide" controls></audio>
                  <div class="status" id="vcProgress" style="margin-top:8px">0 recorded</div>
                </div>
                <button class="go" id="vcTrainBtn" onclick="vcTrain()" disabled>Train voice</button>
                <div class="status" id="vcTrainStat"></div>
                <div id="vcBarWrap" class="hide prog"><div id="vcBar"></div></div>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- ===================== AUDIOBOOK ===================== -->
      <section class="panel" data-panel="audiobook">
        <div class="view view-wide">
          <div class="hero">
            <span class="hico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M4 5a2 2 0 0 1 2-2h13v18H6a2 2 0 0 0-2 2z"/><path d="M19 3v18"/><path d="M8 8h7M8 12h7"/></svg></span>
            <div><h1>Audiobook</h1><div class="tag">Turn a story or any text into chaptered MP3s, in any voice you have</div></div>
            <div class="loadbar">
              <button class="load" id="abLoad" onclick="loadModel('audiobook')">Load voice</button>
              <button class="stop hide" id="abStop" onclick="stopModel('audiobook')">Stop voice</button>
              <span class="loadstat" id="abStat"><span class="dot"></span>not loaded</span>
            </div>
          </div>
          <div class="ws">
            <div class="ws-ctl">
              <div class="card">

            <div class="field"><label>Narration voice</label>
              <select id="abEngine" onchange="abEngineChange()">
                <option value="kokoro">Kokoro — fast, clean narrator</option>
                <option value="chatterbox">Chatterbox — expressive / cloning</option>
                <option value="zonos">Zonos — max quality (if installed)</option>
                <option value="voice">My fine-tuned voice (XTTS)</option>
              </select>
            </div>
            <div class="field" id="abVoiceWrap"><label>Kokoro voice</label><select id="abVoice"></select></div>
            <div class="field hide" id="abVoiceModelWrap"><label>Voice model</label>
              <div class="row" style="align-items:flex-end"><select id="abVoiceModel"></select>
                <button type="button" class="rec" style="flex:0 0 auto" onclick="loadVoices()" title="Refresh voices">↻</button></div>
            </div>
            <div class="field hide" id="abRefWrap"><label>Reference voice <span class="note">(the clip this engine clones)</span></label>
              <select id="abRef" onchange="abRefChange()"></select>
              <label class="filedrop hide" id="abRefUpWrap" style="margin-top:6px"><input type="file" id="abRefFile" accept="audio/*"><span class="filedrop-name"></span></label>
            </div>
            <div class="row" style="align-items:center">
              <button type="button" class="rec" id="abPreviewBtn" style="flex:0 0 auto" onclick="abPreview()" title="Hear one sample line in this voice before rendering">🔊 Preview voice</button>
              <audio id="abPreviewAudio" class="hide" controls></audio>
            </div>

            <div class="field" style="margin-top:4px"><label>Source</label>
              <div class="seg" id="abSourceSeg">
                <button type="button" class="active" data-src="story" onclick="abSetSource('story')">Story project</button>
                <button type="button" data-src="text" onclick="abSetSource('text')">Paste / upload</button>
              </div>
            </div>
            <div class="field" id="abStoryWrap"><label>Story</label>
              <div class="row" style="align-items:flex-end"><select id="abStorySel"></select>
                <button type="button" class="rec" style="flex:0 0 auto" onclick="abRefreshStories()" title="Refresh stories">↻</button></div>
            </div>
            <div id="abTextWrap" class="hide">
              <div class="field"><label>Title</label><input id="abTitle" placeholder="My Audiobook"></div>
              <div class="field"><label>Text — or upload .txt / .md <span class="note">(Markdown headings or “* * *” become chapters)</span></label>
                <label class="filedrop"><input type="file" id="abFile" accept=".txt,.md,text/plain" onchange="abLoadFile()"><span class="filedrop-name"></span></label>
                <textarea id="abText" placeholder="Paste your text here…" style="min-height:150px;margin-top:8px"></textarea>
              </div>
            </div>

            <div class="field" style="margin-top:4px"><label>Narration controls</label>
              <div class="row">
                <div style="flex:1"><span class="note">Speed <b id="abSpeedVal">1.00×</b></span>
                  <input type="range" id="abSpeed" min="0.5" max="2" step="0.05" value="1" style="width:100%"
                         oninput="document.getElementById('abSpeedVal').textContent=(+this.value).toFixed(2)+'×'"></div>
              </div>
              <div class="row hide" id="abZonosCtl" style="margin-top:8px">
                <div style="flex:1"><span class="note">Expressiveness <b id="abPitchVal">20</b></span>
                  <input type="range" id="abPitch" min="10" max="45" step="1" value="20" style="width:100%"
                         oninput="document.getElementById('abPitchVal').textContent=this.value"></div>
                <div style="flex:1"><span class="note">Emotion</span>
                  <select id="abEmotion"><option value="">Auto</option><option value="neutral">Neutral</option><option value="warm">Warm</option><option value="somber">Somber</option><option value="tense">Tense</option><option value="intense">Intense</option></select></div>
              </div>
              <div class="row hide" id="abChatterCtl" style="margin-top:8px">
                <div style="flex:1"><span class="note">Exaggeration <b id="abExagVal">0.50</b></span>
                  <input type="range" id="abExag" min="0" max="1" step="0.05" value="0.5" style="width:100%"
                         oninput="document.getElementById('abExagVal').textContent=(+this.value).toFixed(2)"></div>
              </div>
            </div>

            <button class="go" id="abGenBtn" onclick="abGenerate()">🎧 Render audiobook</button>
            <div class="status" id="abStatus"></div>
              </div>
            </div>
            <div class="ws-out">
              <div class="stage" id="abStage" style="min-height:300px;justify-content:flex-start">
                <div class="empty" id="abEmpty">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"><path d="M4 5a2 2 0 0 1 2-2h13v18H6a2 2 0 0 0-2 2z"/><path d="M19 3v18"/><path d="M8 8h7M8 12h7"/></svg>
                  <b>Your audiobook appears here</b>
                  One MP3 per chapter plus the combined book. A full novel takes a while — progress shows here.
                </div>
                <div id="abBarWrap" class="hide prog" style="width:100%"><div id="abBar"></div></div>
                <div id="abFiles" style="width:100%;display:flex;flex-direction:column;gap:12px"></div>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- ===================== STORY MAKER ===================== -->
      <section class="panel storypanel" data-panel="story" id="storyPanel">
        <div class="sttoolbar">
          <input id="stTitle" class="sttitle" placeholder="Story title" oninput="stTitleChange()">
          <div class="stseg">
            <button id="stModePlan" class="segbtn active" onclick="stMode('plan')">✎ Plan</button>
            <button id="stModeRead" class="segbtn" onclick="stMode('read')">📖 Read</button>
          </div>
          <span class="spacer"></span>
          <span id="stStatus" class="status"></span>
          <button class="rec" onclick="stNew()">New</button>
          <select id="stLoadSel" class="stload" title="Saved stories"></select>
          <button class="rec" onclick="stOpen()">Open</button>
          <button class="rec" onclick="stDelete()">Delete</button>
          <span class="stmodel" style="display:inline-flex;align-items:center;gap:8px" title="The fiction model (koboldcpp · Cydonia-24B) — loading it frees other models from VRAM">
            <button class="load" id="storyLoad" onclick="loadModel('story')">Load story model</button>
            <button class="stop hide" id="storyStop" onclick="stopModel('story')">Stop model</button>
            <span class="loadstat" id="storyStat"><span class="dot"></span>not loaded</span>
          </span>
          <button class="go" id="stGenBtn" onclick="stGenerate()">✨ Generate story</button>
        </div>

        <div id="stPlan" class="stplan">
          <aside class="stpalette" id="stPalette"></aside>
          <section class="stcanvaswrap"><div class="stcanvas" id="stCanvas"></div></section>
          <aside class="stinspector insp" id="stInspector"></aside>
        </div>

        <div id="stRead" class="stread hide">
          <div class="streadbar">
            <span id="stGenProg" class="status"></span>
            <span class="spacer"></span>
            <button class="rec" onclick="stGenerate()">✨ Regenerate all</button>
            <button class="go" onclick="stExport()">⬇ Export .md</button>
          </div>
          <div class="streader" id="stReader"></div>
        </div>
      </section>

    </div><!-- /views -->
  </div><!-- /main -->
</div><!-- /app -->

<script>
const $ = id => document.getElementById(id);

/* ===================== SHELL: nav, theme, palette, toasts ===================== */
const VIEW_TONES={home:'--hue-home',lang:'--hue-text',story:'--hue-story',gen:'--hue-image',edit:'--hue-image',
                  sprite:'--hue-image',composer:'--hue-music',music:'--hue-music',lullaby:'--hue-music',split:'--hue-music',
                  stt:'--hue-voice',tts:'--hue-voice',voicestudio:'--hue-voice',audiobook:'--hue-voice'};
function setActiveTab(name){
  document.querySelectorAll('.navitem').forEach(x=>x.classList.toggle('active',x.dataset.tab===name));
  document.querySelectorAll('.panel').forEach(x=>x.classList.toggle('active',x.dataset.panel===name));
  const ni=document.querySelector('.navitem[data-tab="'+name+'"]');
  if(ni) $('viewTitle').textContent=ni.dataset.title||name;
  document.documentElement.style.setProperty('--viewtone',`var(${VIEW_TONES[name]||'--hue-home'})`);
  document.body.classList.remove('nav-open');
  const v=document.querySelector('.panel[data-panel="'+name+'"]'); if(v) v.scrollTop=0;
}
document.querySelectorAll('.navitem').forEach(t=>t.addEventListener('click',()=>setActiveTab(t.dataset.tab)));
$('hamb').addEventListener('click',()=>{
  if(window.innerWidth<=1000){ document.body.classList.toggle('nav-open'); return; }
  const min=document.body.classList.toggle('nav-min');
  try{localStorage.setItem('lais-nav',min?'min':'');}catch(e){}
});
try{ if(localStorage.getItem('lais-nav')==='min') document.body.classList.add('nav-min'); }catch(e){}
/* tab deep-links: open http://127.0.0.1:8800/#sprite (any data-tab name) to land on that tab */
try{ const h=location.hash.slice(1); if(h&&document.querySelector('.navitem[data-tab="'+h+'"]')) setActiveTab(h); }catch(e){}
document.querySelectorAll('.navitem').forEach(n=>n.title=n.dataset.title||'');  // tooltips for the icon rail
document.addEventListener('click',e=>{ if(document.body.classList.contains('nav-open') && !e.target.closest('.sidebar') && !e.target.closest('#hamb')) document.body.classList.remove('nav-open'); });

function applyTheme(t){ const light=t==='light'; document.documentElement.classList.toggle('light',light);
  const i=$('themeIcon'); if(i) i.innerHTML = light
    ? '<circle cx="12" cy="12" r="4.2"/><path d="M12 2v2.5M12 19.5V22M2 12h2.5M19.5 12H22M4.9 4.9l1.8 1.8M17.3 17.3l1.8 1.8M19.1 4.9l-1.8 1.8M6.7 17.3l-1.8 1.8"/>'
    : '<path d="M21 12.8A9 9 0 1 1 11.2 3 7 7 0 0 0 21 12.8z"/>'; }
function toggleTheme(){ const next=document.documentElement.classList.contains('light')?'dark':'light';
  try{localStorage.setItem('lais-theme',next);}catch(e){} applyTheme(next); }
applyTheme((function(){try{return localStorage.getItem('lais-theme');}catch(e){return null;}})()||'dark');

function toast(msg,type){ if(!msg) return; const c=$('toasts'); const t=document.createElement('div');
  t.className='toast '+(type||''); t.textContent=msg; c.appendChild(t);
  requestAnimationFrame(()=>t.classList.add('show'));
  setTimeout(()=>{t.classList.remove('show'); setTimeout(()=>t.remove(),320);}, 3400); }

/* command palette */
const CMDK_ITEMS=[
  {t:'Go to Home',f:()=>setActiveTab('home')},
  {t:'Language',f:()=>setActiveTab('lang')},
  {t:'Story Maker',f:()=>setActiveTab('story')},
  {t:'Image — Generate',f:()=>setActiveTab('gen')},
  {t:'Image — Edit',f:()=>setActiveTab('edit')},
  {t:'Sprite Studio',f:()=>setActiveTab('sprite')},
  {t:'Composer — arrange & mix a song',f:()=>setActiveTab('composer')},
  {t:'Music — ACE-Step',f:()=>setActiveTab('music')},
  {t:'Speech → Text',f:()=>setActiveTab('stt')},
  {t:'Text → Speech',f:()=>setActiveTab('tts')},
  {t:'Voice Studio',f:()=>setActiveTab('voicestudio')},
  {t:'Audiobook',f:()=>setActiveTab('audiobook')},
  {t:'New chat (Language)',f:()=>{setActiveTab('lang');chatClear();}},
  {t:'Stop / free the current model',f:()=>globalStop()},
  {t:'Toggle light / dark theme',f:()=>toggleTheme()},
];
let cmdkSel=0, cmdkFiltered=CMDK_ITEMS.slice();
function cmdkRender(q){ q=(q||'').toLowerCase().trim(); cmdkFiltered=CMDK_ITEMS.filter(i=>i.t.toLowerCase().includes(q)); cmdkSel=0;
  $('cmdkList').innerHTML = cmdkFiltered.length
    ? cmdkFiltered.map((i,idx)=>`<div class="cmdk-item ${idx===0?'on':''}" data-i="${idx}">${i.t}</div>`).join('')
    : '<div class="cmdk-empty">No matches</div>'; }
function cmdkHi(){ [...$('cmdkList').children].forEach((c,i)=>c.classList.toggle('on',i===cmdkSel)); const on=$('cmdkList').children[cmdkSel]; if(on&&on.scrollIntoView) on.scrollIntoView({block:'nearest'}); }
function cmdkOpen(){ $('cmdk').classList.remove('hide'); $('cmdkInput').value=''; cmdkRender(''); $('cmdkInput').focus(); }
function cmdkClose(){ $('cmdk').classList.add('hide'); }
$('cmdkBtn').addEventListener('click',cmdkOpen);
$('cmdkInput').addEventListener('input',e=>cmdkRender(e.target.value));
$('cmdkList').addEventListener('click',e=>{ const d=e.target.closest('.cmdk-item'); if(!d) return; const it=cmdkFiltered[+d.dataset.i]; cmdkClose(); if(it) it.f(); });
$('cmdk').addEventListener('click',e=>{ if(e.target.id==='cmdk') cmdkClose(); });
document.addEventListener('keydown',e=>{
  if((e.metaKey||e.ctrlKey)&&e.key.toLowerCase()==='k'){ e.preventDefault(); $('cmdk').classList.contains('hide')?cmdkOpen():cmdkClose(); return; }
  if($('cmdk').classList.contains('hide')) return;
  if(e.key==='Escape'){ cmdkClose(); }
  else if(e.key==='ArrowDown'){ e.preventDefault(); cmdkSel=Math.min(cmdkSel+1,cmdkFiltered.length-1); cmdkHi(); }
  else if(e.key==='ArrowUp'){ e.preventDefault(); cmdkSel=Math.max(cmdkSel-1,0); cmdkHi(); }
  else if(e.key==='Enter'){ const it=cmdkFiltered[cmdkSel]; cmdkClose(); if(it) it.f(); }
});

/* home tiles */
const HOME_TILES=[
  {tab:'lang',t:'Language',d:'Code, research & vision',hue:'--hue-text',ic:'<path d="M4 7V5h16v2"/><path d="M9 19h6"/><path d="M12 5v14"/>'},
  {tab:'story',t:'Story Maker',d:'Plan & generate prose',hue:'--hue-story',ic:'<path d="M4 5a2 2 0 0 1 2-2h6v18H6a2 2 0 0 0-2 2z"/><path d="M20 5a2 2 0 0 0-2-2h-6v18h6a2 2 0 0 1 2 2z"/>'},
  {tab:'gen',t:'Generate image',d:'FLUX.2 Klein',hue:'--hue-image',ic:'<rect x="3" y="3" width="18" height="18" rx="2"/><circle cx="8.5" cy="8.5" r="1.6"/><path d="M21 15l-5-5L5 21"/>'},
  {tab:'edit',t:'Edit image',d:'Remove · reframe · recompose',hue:'--hue-image',ic:'<path d="M12 20h9"/><path d="M16.5 3.5a2.1 2.1 0 0 1 3 3L7 19l-4 1 1-4z"/>'},
  {tab:'sprite',t:'Sprite Studio',d:'2D game sprites & sheets',hue:'--hue-image',ic:'<rect x="3" y="3" width="7" height="7" rx="1"/><rect x="14" y="3" width="7" height="7" rx="1"/><rect x="3" y="14" width="7" height="7" rx="1"/><rect x="14" y="14" width="7" height="7" rx="1"/>'},
  {tab:'composer',t:'Composer',d:'Arrange, mix & automate a song',hue:'--hue-music',ic:'<path d="M4 20V9M9 20V4M14 20v-7M19 20v-11"/><circle cx="4" cy="20" r="1.4"/><circle cx="9" cy="20" r="1.4"/><circle cx="14" cy="20" r="1.4"/><circle cx="19" cy="20" r="1.4"/>'},
  {tab:'music',t:'Music',d:'Songs & instrumentals · ACE-Step',hue:'--hue-music',ic:'<path d="M9 18V5l12-2v13"/><circle cx="6" cy="18" r="3"/><circle cx="18" cy="16" r="3"/>'},
  {tab:'stt',t:'Speech → Text',d:'Parakeet transcription',hue:'--hue-voice',ic:'<rect x="9" y="2" width="6" height="12" rx="3"/><path d="M5 11a7 7 0 0 0 14 0"/><path d="M12 18v3"/>'},
  {tab:'tts',t:'Text → Speech',d:'Kokoro · Chatterbox',hue:'--hue-voice',ic:'<path d="M11 5 6 9H2v6h4l5 4z"/><path d="M15.5 8.5a5 5 0 0 1 0 7"/>'},
  {tab:'voicestudio',t:'Voice Studio',d:'Train & use your voice',hue:'--hue-voice',ic:'<path d="M12 2a3 3 0 0 1 3 3v6a3 3 0 0 1-6 0V5a3 3 0 0 1 3-3z"/><path d="M19 10v1a7 7 0 0 1-14 0v-1"/><path d="M12 18v4M8 22h8"/>'},
  {tab:'audiobook',t:'Audiobook',d:'Text → chaptered MP3s',hue:'--hue-voice',ic:'<path d="M4 5a2 2 0 0 1 2-2h13v18H6a2 2 0 0 0-2 2z"/><path d="M19 3v18"/><path d="M8 8h7M8 12h7"/>'},
];
$('homeTiles').innerHTML = HOME_TILES.map(x=>`<div class="tile" data-tab="${x.tab}" style="--ico:var(${x.hue})">
  <span class="tico"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">${x.ic}</svg></span>
  <b>${x.t}</b><span>${x.d}</span></div>`).join('');
$('homeTiles').querySelectorAll('.tile').forEach(t=>t.addEventListener('click',()=>setActiveTab(t.dataset.tab)));

/* file drop zones */
function wireDrops(){
  document.querySelectorAll('.filedrop').forEach(z=>{
    const inp=z.querySelector('input[type=file]'); if(!inp) return;
    ['dragenter','dragover'].forEach(ev=>z.addEventListener(ev,e=>{e.preventDefault();z.classList.add('over');}));
    ['dragleave','dragend'].forEach(ev=>z.addEventListener(ev,e=>{z.classList.remove('over');}));
    z.addEventListener('drop',e=>{ e.preventDefault(); z.classList.remove('over');
      if(!e.dataTransfer.files.length) return;
      try{ inp.files=e.dataTransfer.files; }
      catch(_){ const dt=new DataTransfer(); [...e.dataTransfer.files].forEach(f=>dt.items.add(f)); inp.files=dt.files; }
      inp.dispatchEvent(new Event('change',{bubbles:true})); });
  });
}
document.addEventListener('change',e=>{
  const inp=e.target; if(inp.matches&&inp.matches('.filedrop input[type=file]')){
    const cue=inp.closest('.filedrop').querySelector('.filedrop-name');
    if(cue){ const fs=inp.files; cue.textContent = fs&&fs.length ? (fs.length>1?fs.length+' files':fs[0].name) : ''; }
  }
});

/* ===================== helpers ===================== */
function fileB64(input){return new Promise((res,rej)=>{const f=input.files[0]; if(!f)return res(null); const r=new FileReader(); r.onload=()=>res(r.result); r.onerror=rej; r.readAsDataURL(f);});}
function filesB64(input){return Promise.all([...input.files].slice(0,4).map(f=>new Promise((res,rej)=>{const r=new FileReader(); r.onload=()=>res(r.result); r.onerror=rej; r.readAsDataURL(f);})));}
async function post(path,body){const r=await fetch(path,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(body)}); const j=await r.json(); if(!r.ok||j.error) throw new Error(j.error||('HTTP '+r.status)); return j;}
async function get(path){const r=await fetch(path); return r.json();}
function setStatus(id,msg,cls){const e=$(id); if(e){e.textContent=msg; e.className='status '+(cls||'');} if(cls==='err') toast(msg,'err');}
function stageBusy(id,on){const s=$(id); if(s) s.classList.toggle('loading',!!on);}
function copyText(t,btn){ if(!navigator.clipboard) return; navigator.clipboard.writeText(t).then(()=>{ if(btn){const o=btn.textContent; btn.textContent='✓ copied'; setTimeout(()=>btn.textContent=o,1200);} }).catch(()=>{}); }
function lightbox(img){ if(!img.src) return; const lb=document.createElement('div'); lb.className='lightbox'; const i=new Image(); i.src=img.src; lb.appendChild(i); lb.onclick=()=>lb.remove(); document.body.appendChild(lb); }
function seenStage(id){ const s=$(id); if(s&&window.innerWidth<1000) s.scrollIntoView({behavior:'smooth',block:'nearest'}); }

// ---- runtime config (model tags, incl. the configurable Unlocked tag) ------
let CFG = {unlocked_model:'huihui_ai/gpt-oss-abliterated:20b', standard_model:'gpt-oss:20b', vision_model:'qwen3-vl:8b'};
get('/api/config').then(c=>{CFG=c; paint();}).catch(()=>{});
// Which Ollama tag the Language tab will load/run, given task + Unlocked toggle.
function llmModelTag(){
  if($('llmTask').value==='vision') return CFG.vision_model;
  return $('llmUnlocked').checked ? CFG.unlocked_model : CFG.standard_model;
}

// ---- per-tab load/stop -----------------------------------------------------
const TABS = {
  lang:{fs:'langFs',load:'langLoad',stop:'langStop',stat:'langStat',
        key:()=>'llm:'+llmModelTag(),
        params:()=>({worker:'llm',task:$('llmTask').value,unlocked:$('llmUnlocked').checked})},
  gen:{fs:'genFs',load:'genLoad',stop:'genStop',stat:'genStat',
       key:()=>'image:'+$('imgModel').value, params:()=>({worker:'image',model:$('imgModel').value})},
  edit:{fs:'editFs',load:'editLoad',stop:'editStop',stat:'editStat',
        key:()=>'image:'+$('edModel').value, params:()=>({worker:'image',model:$('edModel').value})},
  sprite:{fs:'spFs',load:'spLoad',stop:'spStop',stat:'spStat',
        key:()=>'image:'+$('spModel').value, params:()=>({worker:'image',model:$('spModel').value})},
  music:{fs:'muFs',load:'muLoad',stop:'muStop',stat:'muStat',
         key:()=>'music', params:()=>({worker:'music'})},
  // no fieldset: the Composer still works with no model loaded (it arranges from
  // the style template), so the panel is never gated — only the planner is
  composer:{load:'coLoad',stop:'coStop',stat:'coStat',
         key:()=>'llm:'+CFG.standard_model, params:()=>({worker:'llm',task:'research'})},
  stt:{fs:'sttFs',load:'sttLoad',stop:'sttStop',stat:'sttStat',
       key:()=>'stt', params:()=>({worker:'stt'})},
  tts:{fs:'ttsFs',load:'ttsLoad',stop:'ttsStop',stat:'ttsStat',
       key:()=>'tts:'+$('ttsEngine').value, params:()=>({worker:'tts',engine:$('ttsEngine').value})},
  voicemodel:{fs:'vmFs',load:'vmLoad',stop:'vmStop',stat:'vmStat',
              key:()=>'voicemodel:'+($('vsVoice').value||''),
              params:()=>({worker:'voicemodel',name:$('vsVoice').value})},
  story:{load:'storyLoad',stop:'storyStop',stat:'storyStat',   // no fieldset — Story Maker isn't gated
         key:()=>'story:Cydonia-24B', params:()=>({worker:'story'})},
  audiobook:{load:'abLoad',stop:'abStop',stat:'abStat',        // narration voice for the Audiobook tab
         key:()=>{const e=$('abEngine').value; return e==='voice' ? 'voicemodel:'+($('abVoiceModel').value||'') : 'tts:'+e;},
         params:()=>{const e=$('abEngine').value; return e==='voice' ? {worker:'voicemodel',name:$('abVoiceModel').value} : {worker:'tts',engine:e};}},
};
let LOADED = null;   // current backend key
let BUSY = false;

function prettyKey(k){
  if(!k) return 'No model loaded';
  if(k.indexOf('llm:')===0) return k.slice(4);
  if(k.indexOf('image:')===0) return 'Image · '+k.slice(6);
  if(k.indexOf('tts:')===0) return 'TTS · '+k.slice(4);
  if(k.indexOf('voicemodel:')===0) return 'Voice · '+(k.slice(11)||'?');
  if(k.indexOf('story:')===0) return 'Story · '+k.slice(6);
  if(k==='music') return 'Music · ACE-Step';
  if(k==='stt') return 'Speech → Text';
  return k;
}
function paintGlobal(){
  const on=!!LOADED;
  $('gModelText').textContent = prettyKey(LOADED);
  $('gModelDot').className = 'gdot'+(on?' on':'');
  $('gStop').classList.toggle('hide',!on); $('gStop').disabled=BUSY;
  $('homeModelText').textContent = prettyKey(LOADED);
  $('homeDot').className = 'gdot'+(on?' on':'');
  $('homeStop').classList.toggle('hide',!on); $('homeStop').disabled=BUSY;
}
function paint(){
  for(const [id,t] of Object.entries(TABS)){
    const on = LOADED && LOADED===t.key();
    if(t.fs){ const fsEl=$(t.fs); if(fsEl) fsEl.disabled = !on; }
    $(t.load).classList.toggle('hide', on);
    $(t.stop).classList.toggle('hide', !on);
    $(t.load).disabled = BUSY; $(t.stop).disabled = BUSY;
    const s=$(t.stat);
    if(on){ s.className='loadstat on'; s.innerHTML='<span class="dot on"></span>loaded'; }
    else if(LOADED){ s.className='loadstat'; s.innerHTML='<span class="dot"></span>another model is loaded'; }
    else { s.className='loadstat'; s.innerHTML='<span class="dot"></span>not loaded'; }
  }
  paintGlobal();
}
function reeval(){ paint(); }

async function loadModel(id){
  if(BUSY) return; BUSY=true;
  const t=TABS[id]; const s=$(t.stat);
  s.className='loadstat run'; s.innerHTML='<span class="dot run"></span>loading… (frees other models first)';
  $(t.load).disabled=true; paintGlobal();
  try{ const r=await post('/api/load',t.params()); LOADED=r.key; toast(prettyKey(LOADED)+' loaded','ok'); }
  catch(e){ s.className='loadstat err'; s.innerHTML='<span class="dot"></span>'+e.message; toast(e.message,'err'); }
  finally{ BUSY=false; paint(); }
}
async function stopModel(id){
  if(BUSY) return; BUSY=true; paint();
  try{ const r=await post('/api/stop',{}); LOADED=r.key; }
  catch(e){ /* ignore */ }
  finally{ BUSY=false; paint(); }
}
async function globalStop(){
  if(BUSY||!LOADED) return; BUSY=true; paint();
  try{ const r=await post('/api/stop',{}); LOADED=r.key; toast('Model stopped — VRAM freed','ok'); }
  catch(e){ toast('Could not stop model','err'); }
  finally{ BUSY=false; paint(); }
}

// ---- runs (only reachable when the fieldset is enabled) --------------------
$('llmTask').onchange=()=>{const v=$('llmTask').value==='vision';$('llmImgWrap').classList.toggle('hide',!v);$('llmUnlocked').disabled=v; reeval();};

// ---- Language chat ---------------------------------------------------------
let CHAT=[];
function chatSuggest(b){ $('llmPrompt').value=b.textContent; $('llmPrompt').focus(); }
function chatClear(){
  CHAT=[]; try{$('llmImg').value='';}catch(_){} const n=$('llmImgName'); if(n) n.textContent='attach image…';
  document.querySelectorAll('#llmOut .msg').forEach(m=>m.remove());
  $('llmEmpty').classList.remove('hide'); setStatus('llmStatus','','');
}
function chatImgPick(){ const f=$('llmImg').files[0]; $('llmImgName').textContent=f?f.name:'attach image…'; }
function addMsg(role,text,imgSrc){
  $('llmEmpty').classList.add('hide');
  const log=$('llmOut'), d=document.createElement('div'); d.className='msg '+role;
  if(imgSrc){ const im=new Image(); im.className='mimg'; im.src=imgSrc; d.appendChild(im); }
  d.appendChild(document.createTextNode(text));
  if(role==='ai'&&text){ const c=document.createElement('button'); c.className='mcopy'; c.title='Copy reply';
    c.textContent='⧉'; c.onclick=()=>copyText(text,c); d.appendChild(c); }
  log.appendChild(d); log.scrollTop=log.scrollHeight; return d;
}
async function runLLM(){
  const task=$('llmTask').value, q=$('llmPrompt').value.trim();
  if(!q) return;
  const img = task==='vision' ? await fileB64($('llmImg')) : null;
  if(task==='vision' && !img){ setStatus('llmStatus','attach an image for vision','err'); return; }
  addMsg('user',q,img);
  $('llmPrompt').value='';
  const think=addMsg('ai','thinking…'); think.classList.add('thinking');
  $('llmSend').disabled=true; setStatus('llmStatus','running on local model…','run');
  // Fold recent turns back into the prompt so follow-ups keep context (the backend
  // /api/llm is stateless; vision stays single-shot — image + question only).
  let prompt=q;
  const hist=CHAT.slice(-8);
  if(task!=='vision' && hist.length){
    prompt='Continue this conversation naturally. Recent exchange:\n'
      + hist.map(m=>(m.role==='user'?'User: ':'Assistant: ')+m.text).join('\n')
      + '\nUser: '+q+'\nAssistant:';
  }
  try{
    const j=await post('/api/llm',{task,prompt,image:img,unlocked:$('llmUnlocked').checked});
    think.remove(); addMsg('ai',j.text||'(empty response)');
    CHAT.push({role:'user',text:q},{role:'ai',text:j.text||''});
    setStatus('llmStatus','','');
  }catch(e){ think.remove(); addMsg('ai','⚠ '+e.message); setStatus('llmStatus',e.message,'err'); }
  finally{ $('llmSend').disabled=false; $('llmPrompt').focus(); }
}
async function runImage(){
  setStatus('imgStatus','generating…','run'); stageBusy('imgStage',true);
  $('imgDl').classList.add('hide'); $('imgMeta').classList.add('hide');
  const t0=Date.now();
  try{
    const j=await post('/api/image',{prompt:$('imgPrompt').value.trim(),model:$('imgModel').value,size:$('imgSize').value,steps:+$('imgSteps').value,seed:+$('imgSeed').value});
    $('imgEmpty').classList.add('hide'); $('imgStage').classList.add('hasresult');
    $('imgOut').src=j.image; $('imgOut').classList.remove('hide');
    $('imgDl').href=j.image; $('imgDl').classList.remove('hide');
    $('imgMeta').innerHTML=`<span><b>${$('imgSize').value}</b> aspect</span><span><b>${$('imgSteps').value}</b> steps</span>`
      +`<span><b>${((Date.now()-t0)/1000).toFixed(1)}s</b></span><span>in memory only — Save to keep</span>`;
    $('imgMeta').classList.remove('hide');
    setStatus('imgStatus','done','ok'); seenStage('imgStage');
  }catch(e){setStatus('imgStatus',e.message,'err');}
  finally{ stageBusy('imgStage',false); }
}
let ED_BA={before:null,after:null};
function edPreview(){const box=$('edThumbs'); box.innerHTML=''; const fs=[...$('edImgs').files].slice(0,4);
  fs.forEach(f=>{const i=new Image(); i.src=URL.createObjectURL(f); box.appendChild(i);});
  ED_BA.before=fs.length?URL.createObjectURL(fs[0]):null;}
function edShow(w){ if(!ED_BA.after) return;
  $('edOut').src = (w==='before'&&ED_BA.before) ? ED_BA.before : ED_BA.after;
  document.querySelectorAll('#edBA button').forEach(b=>b.classList.toggle('active',b.textContent.toLowerCase()===w)); }
async function runEdit(){
  const imgs=await filesB64($('edImgs'));
  if(!imgs.length){setStatus('edStatus','add at least one source image','err');return;}
  setStatus('edStatus','editing…','run'); stageBusy('edStage',true);
  $('edDl').classList.add('hide'); $('edBA').classList.add('hide'); $('edMeta').classList.add('hide');
  const t0=Date.now();
  try{
    const j=await post('/api/edit',{prompt:$('edPrompt').value.trim(),images:imgs,model:$('edModel').value,size:$('edSize').value,cfg:+$('edCfg').value,steps:+$('edSteps').value,seed:+$('edSeed').value});
    ED_BA.after=j.image;
    $('edEmpty').classList.add('hide'); $('edStage').classList.add('hasresult');
    $('edOut').src=j.image; $('edOut').classList.remove('hide');
    $('edDl').href=j.image; $('edDl').classList.remove('hide');
    if(ED_BA.before) $('edBA').classList.remove('hide');
    edShow('after');
    $('edMeta').innerHTML=`<span>strength <b>${$('edCfg').value}</b></span>`
      +`<span><b>${((Date.now()-t0)/1000).toFixed(1)}s</b></span><span>in memory only — Save to keep</span>`;
    $('edMeta').classList.remove('hide');
    setStatus('edStatus','done','ok'); seenStage('edStage');
  }catch(e){setStatus('edStatus',e.message,'err');}
  finally{ stageBusy('edStage',false); }
}
/* ---- Sprite Studio ---- */
// Mirror of server-side SPRITE_ACTIONS: action -> default frame count.
const SP_ACTIONS={idle:4,walk:6,run:6,jump:4,fall:2,crouch:2,attack:5,hurt:2,death:5};
$('spAction').innerHTML=Object.entries(SP_ACTIONS).map(([a,n])=>`<option value="${a}">${a} (${n} frames)</option>`).join('')+'<option value="custom">Custom…</option>';
$('spSetActions').innerHTML=Object.entries(SP_ACTIONS).map(([a,n])=>`<label class="chk"><input type="checkbox" data-act="${a}" checked> ${a} <span class="note">(${n})</span></label>`).join('');
$('spSetActions').addEventListener('change',()=>spEstimate());
let SP_MODE='single', spTimer=null;
function spSetMode(m){SP_MODE=m; document.querySelectorAll('#spModeSeg button').forEach(b=>b.classList.toggle('active',b.dataset.mode===m));
  $('spSingle').classList.toggle('hide',m!=='single'); $('spSet').classList.toggle('hide',m!=='set'); spEstimate();}
function spActionChange(){ $('spCustomWrap').classList.toggle('hide',$('spAction').value!=='custom'); spEstimate(); }
function spRefPreview(){ const box=$('spRefThumb'); box.innerHTML=''; const f=$('spRef').files[0];
  if(f){ const i=new Image(); i.src=URL.createObjectURL(f); box.appendChild(i); } }
function spFrameCount(){
  if(SP_MODE==='set') return [...document.querySelectorAll('#spSetActions input:checked')].reduce((s,c)=>s+SP_ACTIONS[c.dataset.act],0);
  return Math.min(12,Math.max(2,+$('spFrames').value||SP_ACTIONS[$('spAction').value]||4)); }
function spEstimate(){ const n=spFrameCount();
  $('spEstimate').textContent=`${n} frames ≈ ${Math.max(1,Math.round(n*0.6))}–${Math.round(n*1.1)} min — frames appear as they finish; everything is saved to sprites/.`; }
function spDone(){ $('spGo').classList.remove('hide'); $('spCancel').classList.add('hide'); stageBusy('spStage',false); }
async function cancelSprites(){ try{ await post('/api/sprite_cancel',{}); setStatus('spStatus','cancelling after this frame…','run'); }catch(e){} }
async function runSprites(){
  const img=await fileB64($('spRef'));
  if(!img){ setStatus('spStatus','add a reference image first','err'); return; }
  const body={image:img,desc:$('spDesc').value.trim(),mode:SP_MODE,view:$('spView').value,
    size:+$('spSize').value,cutout:$('spCut').checked,steps:+$('spSteps').value,
    cfg:+$('spCfg').value,seed:+$('spSeed').value,name:$('spName').value.trim()};
  if(SP_MODE==='set'){
    body.actions=[...document.querySelectorAll('#spSetActions input:checked')].map(c=>c.dataset.act);
    if(!body.actions.length){ setStatus('spStatus','pick at least one action','err'); return; }
  }else{
    body.action=$('spAction').value; body.custom=$('spCustom').value.trim();
    if(body.action==='custom'&&!body.custom){ setStatus('spStatus','describe the custom action','err'); return; }
    const f=+$('spFrames').value; if(f) body.frames=f;
  }
  $('spGo').classList.add('hide'); $('spCancel').classList.remove('hide');
  $('spBarWrap').classList.remove('hide'); $('spBar').style.width='0%';
  $('spOut').innerHTML=''; $('spZip').classList.add('hide'); $('spMeta').classList.add('hide');
  SP_FILES=new Set();
  setStatus('spStatus','starting…','run'); stageBusy('spStage',true);
  try{ await post('/api/sprite_start',body); spPoll(); }
  catch(e){ setStatus('spStatus',e.message,'err'); spDone(); }
}
async function spPoll(){
  clearTimeout(spTimer);
  try{
    const s=await get('/api/sprite_status');
    const pct=s.total?Math.round((s.step/s.total)*100):0; $('spBar').style.width=pct+'%';
    spRender(s);
    if(s.state==='running'){
      setStatus('spStatus',`Rendering ${s.step}/${s.total} — ${s.current||''}`,'run');
      $('spGo').classList.add('hide'); $('spCancel').classList.remove('hide');
      $('spBarWrap').classList.remove('hide'); stageBusy('spStage',true);
      spTimer=setTimeout(spPoll,2500); return;
    }
    if(s.state==='done'){
      setStatus('spStatus','✓ '+(s.message||'sprite set complete'),'ok'); $('spBar').style.width='100%';
      $('spZip').href='/api/sprite_zip?name='+encodeURIComponent(s.name);
      $('spZip').setAttribute('download',s.name+'.zip'); $('spZip').classList.remove('hide');
      $('spMeta').innerHTML=`<span>saved to <b>sprites/${s.name}</b></span>`; $('spMeta').classList.remove('hide');
    }
    else if(s.state==='error'){ setStatus('spStatus','Error: '+s.message,'err'); }
    else if(s.state==='cancelled'){ setStatus('spStatus','Cancelled — finished frames kept in sprites/'+s.name,'ok'); }
    spDone();
  }catch(e){ spTimer=setTimeout(spPoll,3000); }
}
let SP_NAME=null, SP_FILES=new Set();
function spRender(s){
  if(!(s.files||[]).length) return;
  if(s.name&&s.name!==SP_NAME){ SP_NAME=s.name; SP_FILES=new Set(); }
  // A re-roll's status lists only the rebuilt files — merge so the rest stay visible.
  (s.files||[]).forEach(f=>SP_FILES.add(f));
  const files=[...SP_FILES];
  $('spEmpty').classList.add('hide'); $('spStage').classList.add('hasresult'); $('spOut').classList.remove('hide');
  const url=f=>'/sprites/'+f.split('/').map(encodeURIComponent).join('/');
  const groups={}; let combined=null, meta=null;
  files.forEach(f=>{ const parts=f.split('/');
    if(parts.length===3){ const g=groups[parts[1]]=groups[parts[1]]||{frames:[]}; g.frames.push(f); }
    else if(f.endsWith('_sheet.png')){ const a=parts[1].replace(/_sheet\.png$/,''); const g=groups[a]=groups[a]||{frames:[]}; g.sheet=f; }
    else if(f.endsWith('spritesheet.png')) combined=f;
    else if(f.endsWith('spritesheet.json')) meta=f; });
  // Cache-busting: normal run flips v once when an action's strip lands (frames were
  // rewritten in place by the cutout pass); a re-roll (total===1) busts with a timestamp.
  const rr=s.total===1, stamp=Date.now();
  let h='';
  if(combined) h+=`<div class="spsec"><div class="sphead"><b>Combined sheet</b>
      <a class="dlbtn" href="${url(combined)}" download>⬇ spritesheet.png</a>${meta?`<a class="dlbtn" href="${url(meta)}" download>⬇ metadata</a>`:''}</div>
      <img class="spsheet" src="${url(combined)}${rr?'?v='+stamp:''}" onclick="lightbox(this)"></div>`;
  for(const [a,g] of Object.entries(groups)){
    const v=rr?stamp:(g.sheet?2:1);
    h+=`<div class="spsec"><div class="sphead"><b>${a}</b><span class="note">${g.frames.length} frames</span>
        ${g.sheet?`<a class="dlbtn" href="${url(g.sheet)}${rr?'?v='+stamp:''}" download>⬇ strip sheet</a>`:''}</div>
        <div class="spframes">${g.frames.map((f,i)=>`<span class="spf"><img src="${url(f)}?v=${v}" loading="lazy" onclick="lightbox(this)">
          <button class="rr" title="Re-roll this frame (new seed, stronger pose guidance)" onclick="spReroll('${a}',${i+1})">↻</button></span>`).join('')}</div></div>`;
  }
  $('spOut').innerHTML=h;
}
async function spReroll(action,frame){
  if(!SP_NAME){ setStatus('spStatus','no sprite set loaded','err'); return; }
  setStatus('spStatus',`re-rolling ${action} frame ${frame}…`,'run'); stageBusy('spStage',true);
  $('spGo').classList.add('hide'); $('spBarWrap').classList.remove('hide'); $('spBar').style.width='0%';
  try{ await post('/api/sprite_reroll',{name:SP_NAME,action:action,frame:frame}); spPoll(); }
  catch(e){ setStatus('spStatus',e.message,'err'); spDone(); }
}
spEstimate();
// A sprite set survives a page refresh: reattach to a running/finished job on load.
get('/api/sprite_status').then(s=>{ if(s.state==='running'||(s.files||[]).length) spPoll(); }).catch(()=>{});

/* ---- Composer (brief -> arranged, mixed, automated song) ---- */
let coTimer=null, CO_LIB=null;

function coSecsLabel(){ const v=+$('coSecs').value; $('coSecsLab').textContent=Math.floor(v/60)+':'+String(v%60).padStart(2,'0'); }
function coPolishToggle(){ $('coPolishWrap').classList.toggle('hide',!$('coPolish').checked); }

/* ---- wizard steps ----
   Steps 2 and 3 need a rendered song, so they stay locked until one exists.
   After that all three are freely clickable: arranging and exporting is a loop
   (edit -> re-render -> listen -> edit), not a one-way sequence. */
let CO_STEP=1, CO_HAS_SONG=false;
function coGoStep(n){
  if(n>1&&!CO_HAS_SONG) return;
  CO_STEP=n;
  for(let i=1;i<=3;i++){
    $('coStep'+i).classList.toggle('hide',i!==n);
    const b=$('coStepBtn'+i);
    b.classList.toggle('on',i===n);
    b.classList.toggle('lock',i>1&&!CO_HAS_SONG);
    b.classList.toggle('done',CO_HAS_SONG&&i<n);
  }
  if(n===2&&ED.score) coEdPaint();
}
function coUnlock(){
  CO_HAS_SONG=true;
  for(let i=2;i<=3;i++) $('coStepBtn'+i).classList.remove('lock');
}
function coSetBrief(text,genre){ $('coBrief').value=text; if(genre) $('coGenre').value=genre; $('coBrief').focus(); }

/* The instrument library IS the "free plugins under 800 MB" step: a scan of the
   bundled SoundFonts with the budget applied, shown so it's clear what the
   planner is choosing from and what got excluded. */
async function coLoadLibrary(){
  try{
    const lib=await get('/api/composer_library'); CO_LIB=lib;
    $('coGenre').innerHTML=(lib.genres||[]).map(g=>`<option value="${g}">${g}</option>`).join('');
    const nInst=Object.values(lib.instruments||{}).reduce((a,v)=>a+v.length,0);
    $('coLibSummary').textContent=`Instrument library — ${nInst} instruments, ${(lib.soundfonts||[]).length} SoundFonts`;
    // Each font is described by the JOB it does. The 800 MB figure governs the
    // general bank only; a specialist piano library being bigger than that is
    // not a failure, and calling it "excluded" while the renderer used it was
    // just wrong.
    $('coLibList').innerHTML=(lib.soundfonts||[]).map(f=>
      `<div style="margin-bottom:7px">
        <div style="display:flex;gap:7px;align-items:baseline">
          <span>${f.within_budget?'✅':'⛔'}</span><b style="font-weight:600">${coEsc(f.name)}</b>
          <span class="note" style="margin-left:auto">${f.mb} MB</span></div>
        <div class="note" style="padding-left:20px">General MIDI bank · used for ${coEsc(f.used_for)} ·
          ${f.within_budget?'within':'over'} the ${lib.budget_mb} MB budget</div>
      </div>`).join('') +
      Object.entries(lib.instruments||{}).map(([g,items])=>
        `<div style="margin-top:7px"><b style="font-weight:600;text-transform:capitalize">${g.replace(/_/g,' ')}</b>
         <div class="note">${items.map(i=>coEsc(i.name)).join(' · ')}</div></div>`).join('');
  }catch(e){
    $('coLibSummary').textContent='Instrument library — could not read it: '+e.message;
  }
}

async function coStart(){
  const brief=$('coBrief').value.trim();
  if(!brief && !confirm('No brief given — compose a generic '+$('coGenre').value+' track?')) return;
  $('coGo').classList.add('hide'); $('coCancel').classList.remove('hide');
  $('coBarWrap').classList.remove('hide'); $('coBar').style.width='0%';
  $('coZip').classList.add('hide'); stageBusy('coStage',true);
  setStatus('coStatus','starting…','run');
  try{
    await post('/api/composer_start',{
      brief, genre:$('coGenre').value, seconds:+$('coSecs').value,
      tracks:+$('coTracks').value, name:$('coName').value.trim(),
      key:$('coKey').value, bpm:+$('coBpm').value, seed:+$('coSeed').value,
      stems:$('coStems').checked, use_llm:$('coUseLlm').checked,
      humanize:$('coHumanize').checked,
      polish:$('coPolish').checked, polish_denoise:+$('coPolishDen').value});
    coPoll();
  }catch(e){ setStatus('coStatus',e.message,'err'); coDone(); }
}
async function coCancel(){ try{ await post('/api/composer_cancel',{}); }catch(e){} }
function coDone(){
  $('coGo').classList.remove('hide'); $('coCancel').classList.add('hide');
  stageBusy('coStage',false);
}
async function coPoll(){
  clearTimeout(coTimer);
  try{
    const s=await get('/api/composer_status');
    const pct=s.total?Math.round((s.step/s.total)*100):0; $('coBar').style.width=pct+'%';
    if(s.state==='running'){
      setStatus('coStatus',`${s.step}/${s.total} — ${s.current||'working'}`,'run');
      coTimer=setTimeout(coPoll,1500); return;
    }
    if(s.state==='done'){
      setStatus('coStatus','✓ '+(s.message||'song complete'),'ok'); $('coBar').style.width='100%';
      coRender(s);
      $('coZip').href='/api/composer_zip?name='+encodeURIComponent(s.name);
      $('coZip').setAttribute('download',s.name+'.zip'); $('coZip').classList.remove('hide');
      $('coMeta').innerHTML=`<span>saved to <b>compositions/${s.name}</b></span>`;
      // the finished score becomes the editor's document; notes reload since the
      // arrangement (and so the generated parts) may have changed
      ED.notes=null;
      coEdOpen(s.plan,s.name);
      coUnlock();
      coEdSetPlayer(s);
      // land on Arrange after a fresh compose (you want to shape it next), but
      // don't yank the user off a step they deliberately navigated to
      if(CO_STEP===1) coGoStep(2);
    }
    else if(s.state==='error'){ setStatus('coStatus','Error: '+s.message,'err'); }
    else if(s.state==='cancelled'){ setStatus('coStatus','Cancelled','ok'); }
    coDone();
  }catch(e){ coTimer=setTimeout(coPoll,3000); }
}

const CO_ROLE_ICON={drums:'🥁',perc:'🔔',bass:'🎸',chords:'🎹',pad:'🌫',arp:'✨',lead:'🎺',counter:'🎻'};
function coPanLabel(p){ if(Math.abs(p)<0.05) return 'C'; return (p<0?'L':'R')+Math.round(Math.abs(p)*100); }

function coRender(s){
  const sc=s.plan; if(!sc) return;
  $('coEmpty').classList.add('hide'); $('coStage').classList.add('hasresult');
  $('coOut').classList.remove('hide');
  const url=f=>'/compositions/'+f.split('/').map(encodeURIComponent).join('/');
  const files=s.files||[];
  const polished=files.find(f=>/_polished\.mp3$/i.test(f));
  const mp3=files.find(f=>/\.mp3$/i.test(f)&&!/_polished\.mp3$/i.test(f));
  const wav=files.find(f=>/\.wav$/i.test(f)&&!/\/stems\//.test(f));
  const mid=files.find(f=>/\.mid$/i.test(f)), stems=files.filter(f=>/\/stems\/.+\.(flac|wav)$/i.test(f));
  const mins=Math.floor((sc.duration||0)/60), secs=Math.round((sc.duration||0)%60);
  let h='';

  h+=`<div class="spsec"><div class="sphead"><b>${sc.title||'Untitled'}</b>
      <span class="note">${sc.genre} · ${sc.key} · ${sc.bpm} bpm · ${sc.time_signature}/4 · ${sc.total_bars} bars · ${mins}:${String(secs).padStart(2,'0')}</span></div>
      ${polished?'<div class="note" style="margin-bottom:3px">clean master — sampled instruments, stems intact</div>':''}
      ${mp3?`<audio controls style="width:100%" src="${url(mp3)}"></audio>`:''}
      <div class="sphead" style="margin-top:8px">
        ${mp3?`<a class="dlbtn" href="${url(mp3)}" download>⬇ MP3</a>`:''}
        ${wav?`<a class="dlbtn" href="${url(wav)}" download>⬇ WAV</a>`:''}
        ${mid?`<a class="dlbtn" href="${url(mid)}" download>⬇ Multitrack MIDI</a>`:''}
      </div>
      ${polished?`<div style="margin-top:12px">
        <div class="note" style="margin-bottom:3px">✨ ACE-Step re-texture — matched to the same loudness, so it's a fair A/B</div>
        <audio controls style="width:100%" src="${url(polished)}"></audio>
        <div class="sphead" style="margin-top:6px"><a class="dlbtn" href="${url(polished)}" download>⬇ Polished MP3</a></div>
      </div>`:''}</div>`;

  if(sc.notes) h+=`<div class="spsec"><div class="sphead"><b>Producer's notes</b>
      <span class="note" style="margin-left:auto">planned by ${sc.planner}</span></div>
      <div class="card" style="padding:12px;font-size:13px;line-height:1.6">${coEsc(sc.notes)}</div></div>`;

  h+=`<div class="spsec"><div class="sphead"><b>Instruments &amp; mix</b>
      <span class="note">${(sc.instruments||[]).length} tracks · GM patches from the local SoundFont</span></div>
      <div style="overflow-x:auto"><table style="width:100%;border-collapse:collapse;font-size:12px">
      <thead><tr>${['Track','Role','Instrument (GM)','Level','Pan','Rev','Dly','Drv','Automation']
        .map(x=>`<th style="text-align:left;padding:5px 8px;border-bottom:1px solid var(--edge);white-space:nowrap">${x}</th>`).join('')}</tr></thead><tbody>`;
  (sc.instruments||[]).forEach(t=>{
    const auto=(t.automation||[]).map(a=>`${a.param} ${a.mode}${a.section?' ('+coEsc(a.section)+')':''}`).join(', ')||'—';
    h+=`<tr>
      <td style="padding:5px 8px;border-bottom:1px solid var(--edge)"><b>${CO_ROLE_ICON[t.role]||'🎵'} ${coEsc(t.track)}</b></td>
      <td style="padding:5px 8px;border-bottom:1px solid var(--edge)">${t.role}</td>
      <td style="padding:5px 8px;border-bottom:1px solid var(--edge);white-space:nowrap">${coEsc(t.instrument)}</td>
      <td style="padding:5px 8px;border-bottom:1px solid var(--edge)">${(+t.level).toFixed(2)}</td>
      <td style="padding:5px 8px;border-bottom:1px solid var(--edge)">${coPanLabel(+t.pan)}</td>
      <td style="padding:5px 8px;border-bottom:1px solid var(--edge)">${(+t.reverb).toFixed(2)}</td>
      <td style="padding:5px 8px;border-bottom:1px solid var(--edge)">${(+t.delay).toFixed(2)}</td>
      <td style="padding:5px 8px;border-bottom:1px solid var(--edge)">${(+t.drive).toFixed(2)}</td>
      <td style="padding:5px 8px;border-bottom:1px solid var(--edge)">${auto}</td></tr>`;
  });
  h+='</tbody></table></div></div>';

  // structure: one row per section, width proportional to its bars, so the
  // arrangement reads as a timeline rather than a list
  h+=`<div class="spsec"><div class="sphead"><b>Arrangement</b><span class="note">energy bar = how loud &amp; busy the mix gets</span></div>`;
  (sc.sections||[]).forEach(x=>{
    const pct=Math.round((x.bars/Math.max(sc.total_bars,1))*100);
    h+=`<div style="margin-bottom:7px">
      <div style="display:flex;gap:8px;align-items:baseline;font-size:12px">
        <b style="text-transform:capitalize">${coEsc(x.name)}</b>
        <span class="note">${x.bars} bars</span>
        ${(x.fx||[]).map(f=>`<span class="copybtn" style="cursor:default;padding:1px 7px;font-size:11px">${f}</span>`).join('')}
        <span class="note" style="margin-left:auto">${(x.chord_symbols||[]).join(' · ')}</span></div>
      <div class="prog" style="height:7px;width:${Math.max(pct,4)}%;min-width:40px;margin:3px 0">
        <div style="width:${Math.round((x.energy||0)*100)}%"></div></div>
      <div class="note" style="font-size:11px">${(x.tracks||[]).join(', ')}</div></div>`;
  });
  h+='</div>';

  if(stems.length){
    h+=`<div class="spsec"><div class="sphead"><b>Stems</b><span class="note">post-FX, pre-master — one per instrument</span></div>`;
    stems.forEach(f=>{ const nm=f.split('/').pop().replace(/\.(flac|wav)$/i,'');
      h+=`<div style="display:flex;gap:8px;align-items:center;margin-bottom:5px">
        <b style="font-size:12px;min-width:78px">${coEsc(nm)}</b>
        <audio controls style="flex:1;height:32px" src="${url(f)}"></audio>
        <a class="dlbtn" href="${url(f)}" download>⬇</a></div>`; });
    h+='</div>';
  }

  if((sc.warnings||[]).length)
    h+=`<div class="spsec"><div class="chip-info warn">
      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 9v4M12 17v.01"/><path d="M10.3 3.9 2.4 18a2 2 0 0 0 1.7 3h15.8a2 2 0 0 0 1.7-3L13.7 3.9a2 2 0 0 0-3.4 0z"/></svg>
      <span><b>Repaired in the plan:</b><br>${(sc.warnings||[]).map(coEsc).join('<br>')}</span></div></div>`;

  $('coOut').innerHTML=h;
}
function coEsc(s){ return String(s==null?'':s).replace(/[&<>"']/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c])); }

/* ================= arrangement editor =================
   The score IS the project file, so the editor edits it directly and re-renders.
   ED.score is the working copy; ED.saved is what's on disk (for revert). Notes
   are lazily fetched per track and only become notes_override once you edit
   them, so an untouched track keeps being freshly composed. */
let ED={score:null,saved:null,name:null,notes:null,sel:null,selSec:null,dirty:false,roll:null};
const ED_ROLE_COLOR={drums:'#ef4444',perc:'#f59e0b',bass:'#3b82f6',chords:'#22d3ee',
                     pad:'#8b5cf6',arp:'#22c55e',lead:'#ec4899',counter:'#a78bfa'};

function coEdOpen(score,name){
  if(!score||!score.instruments) return;
  ED.score=JSON.parse(JSON.stringify(score));
  ED.saved=JSON.parse(JSON.stringify(score));
  ED.name=name; ED.notes=null; ED.sel=null; ED.selSec=null; ED.dirty=false;
  $('coEdDirty').classList.add('hide');
  coEdPaint();
}
function coEdMark(){ ED.dirty=true; $('coEdDirty').classList.remove('hide'); }
/* Step 2 carries its own player: the audio exists as soon as step 1 finishes, and
   you edit in response to hearing it — waiting until step 3 to listen would be
   backwards. */
function coEdSetPlayer(s){
  const mp3=(s.files||[]).find(f=>/\.mp3$/i.test(f)&&!/_polished\.mp3$/i.test(f));
  const a=$('coEdAudio');
  if(mp3){ a.src='/compositions/'+mp3.split('/').map(encodeURIComponent).join('/'); }
  const sc=s.plan||{};
  $('coEdNowPlaying').textContent=(sc.title||s.name||'—')+
    (sc.key?`  ·  ${sc.key} · ${sc.bpm} bpm`:'');
}
function coEdReset(){
  if(ED.dirty&&!confirm('Discard your edits and reload the saved arrangement?')) return;
  coEdOpen(ED.saved,ED.name);
}
function coEdBars(){ return ED.score.sections.reduce((a,s)=>a+s.bars,0)||1; }

function coEdPaint(){
  const sc=ED.score, total=coEdBars();
  const secs=sc.sections;
  $('coEdTitle').textContent=sc.title||'Arrangement';
  $('coEdInfo').textContent=`${sc.key} · ${sc.bpm} bpm · ${sc.time_signature}/4 · ${total} bars · `+
    `${sc.instruments.length} tracks`;
  // section header: width proportional to bars, matching the clip lanes below
  const w=b=>`flex:0 0 ${Math.max(58,Math.round(b/total*920))}px`;
  let h=`<div class="coed-secrow"><div class="coed-head"><span class="coed-note">sections →</span></div>`;
  secs.forEach((s,i)=>{ h+=`<div class="coed-sec ${ED.selSec===i?'sel':''}" style="${w(s.bars)}" onclick="coEdSelSec(${i})">
      <b>${coEsc(s.name)}</b><span>${s.bars} bars · e${(+s.energy).toFixed(2)}${(s.fx||[]).length?' · '+s.fx.join(','):''}</span></div>`; });
  h+=`</div>`;
  sc.instruments.forEach((t,ti)=>{
    const col=ED_ROLE_COLOR[t.role]||'#6b7280';
    h+=`<div class="coed-row ${ED.sel===ti?'on':''}">
      <div class="coed-head">
        <span class="coed-name" title="${coEsc(t.instrument)}" onclick="coEdSel(${ti})">${coEsc(t.track)}</span>
        <button class="coed-mini ${t.muted?'act':''}" title="Mute" onclick="coEdMute(${ti})">M</button>
        <button class="coed-mini" title="Remove track" onclick="coEdDelTrack(${ti})">✕</button>
      </div><div class="coed-lane">`;
    secs.forEach((s,si)=>{
      const on=(s.tracks||[]).indexOf(t.track)>=0;
      h+=`<div class="coed-clip ${on?'on':''}" style="${w(s.bars)};${on?'background:'+col:''}"
            title="${coEsc(t.track)} in ${coEsc(s.name)} — click to toggle"
            onclick="coEdToggle(${ti},${si})">${on?`<span class="lbl">${coEsc(t.track)}</span>`:''}</div>`;
    });
    h+=`</div></div>`;
  });
  $('coEdGrid').innerHTML=h;
  coEdInsp();
}

function coEdToggle(ti,si){
  const name=ED.score.instruments[ti].track, s=ED.score.sections[si];
  s.tracks=s.tracks||[];
  const k=s.tracks.indexOf(name);
  if(k>=0) s.tracks.splice(k,1); else s.tracks.push(name);
  coEdMark(); coEdPaint();
}
function coEdMute(ti){ const t=ED.score.instruments[ti]; t.muted=!t.muted; coEdMark(); coEdPaint(); }
function coEdDelTrack(ti){
  const t=ED.score.instruments[ti];
  if(!confirm(`Remove the "${t.track}" track from the arrangement?`)) return;
  ED.score.instruments.splice(ti,1);
  ED.score.sections.forEach(s=>{ const k=(s.tracks||[]).indexOf(t.track); if(k>=0) s.tracks.splice(k,1); });
  if(ED.sel===ti) ED.sel=null;
  coEdMark(); coEdPaint();
}
function coEdAddTrack(){
  const n=prompt('Track name?','new track'); if(!n) return;
  let name=n.trim().slice(0,24)||'track';
  while(ED.score.instruments.some(t=>t.track.toLowerCase()===name.toLowerCase())) name+='2';
  ED.score.instruments.push({track:name,role:'chords',program:0,instrument:'Acoustic Grand Piano',
    level:0.55,pan:0,reverb:0.3,delay:0,drive:0,tone:'neutral',automation:[],muted:false});
  ED.score.sections.forEach(s=>{ (s.tracks=s.tracks||[]).push(name); });
  ED.sel=ED.score.instruments.length-1; coEdMark(); coEdPaint();
}
function coEdAddSection(){
  const src=ED.score.sections[ED.score.sections.length-1]||{bars:8,energy:0.6,chords:[],chord_symbols:[]};
  ED.score.sections.push({name:'section '+(ED.score.sections.length+1),bars:src.bars,
    energy:src.energy,tracks:ED.score.instruments.map(t=>t.track),fx:[],
    chord_symbols:(src.chord_symbols||[]).slice()});
  ED.selSec=ED.score.sections.length-1; coEdMark(); coEdPaint();
}
function coEdSel(ti){ ED.sel=(ED.sel===ti?null:ti); ED.selSec=null; coEdPaint();
  if(ED.sel!==null) coEdLoadNotes(); }
function coEdSelSec(si){ ED.selSec=(ED.selSec===si?null:si); ED.sel=null; coEdPaint(); }

const ED_FX=['riser','impact','downlifter','drop','filter_sweep'];
const ED_ROLES=['drums','perc','bass','chords','pad','arp','lead','counter'];
const ED_TONES=['neutral','warm','bright','dark','thin'];

function coEdInsp(){
  const box=$('coEdInsp');
  if(ED.selSec!==null){ box.innerHTML=coEdSecForm(ED.score.sections[ED.selSec],ED.selSec); return; }
  if(ED.sel===null){ box.innerHTML=`<span class="coed-note">Click a <b>track name</b> to edit its instrument, mix and notes — or a <b>section</b> to change its bars, energy, chords and FX. Click any clip to toggle whether that track plays there.</span>`; return; }
  const t=ED.score.instruments[ED.sel];
  const gm=(CO_LIB&&CO_LIB.gm_all)||null;
  const instSel=gm?`<select onchange="coEdSetProgram(this.value)">${gm.map((n,i)=>
      `<option value="${i}" ${i===t.program?'selected':''}>${i} — ${coEsc(n)}</option>`).join('')}</select>`
    :`<input type="number" min="0" max="127" value="${t.program}" onchange="coEdSetProgram(this.value)">`;
  box.innerHTML=`
    <div class="row">
      <div class="field"><label>Instrument <span class="note">GM patch (drums ignore this)</span></label>${instSel}</div>
      <div class="field"><label>Role <span class="note">— what the arranger writes for it</span></label>
        <select onchange="coEdSetField('role',this.value)">${ED_ROLES.map(r=>
          `<option ${r===t.role?'selected':''}>${r}</option>`).join('')}</select></div>
      <div class="field"><label>Tone</label>
        <select onchange="coEdSetField('tone',this.value)">${ED_TONES.map(r=>
          `<option ${r===t.tone?'selected':''}>${r}</option>`).join('')}</select></div>
    </div>
    <div class="coed-knobs">
      ${coEdKnob('level','Level',t.level,0.05,1.5,0.05)}
      ${coEdKnob('pan','Pan',t.pan,-1,1,0.05)}
      ${coEdKnob('reverb','Reverb',t.reverb,0,1,0.05)}
      ${coEdKnob('delay','Delay',t.delay,0,1,0.05)}
      ${coEdKnob('drive','Drive',t.drive,0,1,0.05)}
    </div>
    <div style="margin-top:9px">
      <div class="sphead"><b>Automation</b>
        <button class="copybtn" style="margin-left:auto" onclick="coEdAddAuto()">+ lane</button></div>
      ${(t.automation||[]).length?(t.automation||[]).map((a,ai)=>coEdAutoRow(a,ai)).join(''):'<span class="coed-note">no automation on this track</span>'}
    </div>
    <div style="margin-top:11px">
      <div class="sphead"><b>Notes</b>
        <span class="note" id="coRollHint">drag to add · click a note to delete · shift-drag to move</span>
        ${t.notes_override?'<span class="coed-dirty" style="margin-left:auto">● hand-edited</span>'
          :'<span class="note" style="margin-left:auto">generated</span>'}
        ${t.notes_override?'<button class="copybtn" onclick="coEdClearNotes()">↺ regenerate</button>':''}
      </div>
      <div class="coroll" id="coRollWrap"><canvas id="coRoll" width="900" height="240"></canvas></div>
    </div>`;
  coEdDrawRoll();
}
function coEdKnob(k,lab,v,lo,hi,st){
  return `<div class="coed-knob"><label><span>${lab}</span><span id="coK_${k}">${(+v).toFixed(2)}</span></label>
    <input type="range" min="${lo}" max="${hi}" step="${st}" value="${v}"
      oninput="$('coK_${k}').textContent=(+this.value).toFixed(2);coEdSetField('${k}',+this.value,true)"></div>`;
}
function coEdAutoRow(a,ai){
  const params=['cutoff','volume','pan','reverb','delay'], modes=['ramp','autopan','tremolo'];
  const names=ED.score.sections.map(s=>s.name);
  return `<div class="row" style="gap:7px;align-items:flex-end;margin-bottom:5px">
    <div class="field" style="flex:0 0 108px"><label>Param</label><select onchange="coEdSetAuto(${ai},'param',this.value)">
      ${params.map(p=>`<option ${p===a.param?'selected':''}>${p}</option>`).join('')}</select></div>
    <div class="field" style="flex:0 0 108px"><label>Mode</label><select onchange="coEdSetAuto(${ai},'mode',this.value)">
      ${modes.map(m=>`<option ${m===a.mode?'selected':''}>${m}</option>`).join('')}</select></div>
    ${a.mode==='ramp'?`
      <div class="field" style="flex:0 0 84px"><label>From</label><input type="number" step="0.05" value="${a.from}" onchange="coEdSetAuto(${ai},'from',+this.value)"></div>
      <div class="field" style="flex:0 0 84px"><label>To</label><input type="number" step="0.05" value="${a.to}" onchange="coEdSetAuto(${ai},'to',+this.value)"></div>
      <div class="field" style="flex:1;min-width:120px"><label>Section <span class="note">blank = whole song</span></label>
        <select onchange="coEdSetAuto(${ai},'section',this.value)"><option value="">(whole song)</option>
        ${names.map(n=>`<option ${n===a.section?'selected':''}>${coEsc(n)}</option>`).join('')}</select></div>`
    :`<div class="field" style="flex:0 0 92px"><label>Depth</label><input type="number" step="0.05" value="${a.depth}" onchange="coEdSetAuto(${ai},'depth',+this.value)"></div>
      <div class="field" style="flex:0 0 92px"><label>Bars</label><input type="number" step="0.5" value="${a.bars}" onchange="coEdSetAuto(${ai},'bars',+this.value)"></div>`}
    <button class="coed-mini" style="margin-bottom:6px" onclick="coEdDelAuto(${ai})">✕</button></div>`;
}
function coEdSecForm(s,si){
  return `<div class="row">
      <div class="field"><label>Section name</label><input type="text" value="${coEsc(s.name)}" onchange="coEdSetSec('name',this.value)"></div>
      <div class="field"><label>Bars — <span id="coS_bars">${s.bars}</span></label>
        <input type="range" min="2" max="32" step="2" value="${s.bars}" oninput="$('coS_bars').textContent=this.value;coEdSetSec('bars',+this.value,true)"></div>
      <div class="field"><label>Energy — <span id="coS_en">${(+s.energy).toFixed(2)}</span> <span class="note">drives loudness, kit density &amp; velocity</span></label>
        <input type="range" min="0" max="1" step="0.05" value="${s.energy}" oninput="$('coS_en').textContent=(+this.value).toFixed(2);coEdSetSec('energy',+this.value,true)"></div>
    </div>
    <div class="field"><label>Chords <span class="note">— space or comma separated; loops to fill the section (Am F C G, Cmaj7, Bb/D, V7 …)</span></label>
      <input type="text" value="${coEsc((s.chord_symbols||[]).join(' '))}" onchange="coEdSetChords(this.value)"></div>
    <div class="field"><label>Section FX</label><div class="coed-chips">${ED_FX.map(f=>
      `<span class="coed-chip ${(s.fx||[]).indexOf(f)>=0?'on':''}" onclick="coEdToggleFx('${f}')">${f}</span>`).join('')}</div></div>
    <div class="sphead" style="margin-top:8px"><span class="coed-note">${(s.tracks||[]).length} of ${ED.score.instruments.length} tracks playing here</span>
      <button class="copybtn" style="margin-left:auto" onclick="coEdDelSection(${si})">✕ delete section</button></div>`;
}
function coEdSetField(k,v,quiet){
  const t=ED.score.instruments[ED.sel]; t[k]=v;
  if(k==='program'&&CO_LIB&&CO_LIB.gm_all) t.instrument=CO_LIB.gm_all[v]||t.instrument;
  coEdMark(); if(!quiet) coEdPaint();
}
function coEdSetProgram(v){ const t=ED.score.instruments[ED.sel]; t.program=+v;
  if(CO_LIB&&CO_LIB.gm_all) t.instrument=CO_LIB.gm_all[+v]||t.instrument; coEdMark(); coEdPaint(); }
function coEdAddAuto(){ const t=ED.score.instruments[ED.sel];
  (t.automation=t.automation||[]).push({param:'cutoff',mode:'ramp',from:0.2,to:1.0,depth:0.4,bars:4,section:''});
  coEdMark(); coEdPaint(); }
function coEdSetAuto(ai,k,v){ ED.score.instruments[ED.sel].automation[ai][k]=v; coEdMark(); coEdPaint(); }
function coEdDelAuto(ai){ ED.score.instruments[ED.sel].automation.splice(ai,1); coEdMark(); coEdPaint(); }
function coEdSetSec(k,v,quiet){ const s=ED.score.sections[ED.selSec]; s[k]=v; coEdMark(); if(!quiet) coEdPaint(); }
function coEdSetChords(v){
  const s=ED.score.sections[ED.selSec];
  s.chord_symbols=v.split(/[\s,]+/).filter(Boolean);
  delete s.chords;   // let the engine re-parse from the symbols
  coEdMark(); coEdPaint();
}
function coEdToggleFx(f){
  const s=ED.score.sections[ED.selSec]; s.fx=s.fx||[];
  const k=s.fx.indexOf(f); if(k>=0) s.fx.splice(k,1); else s.fx.push(f);
  coEdMark(); coEdPaint();
}
function coEdDelSection(si){
  if(ED.score.sections.length<2){ toast('a song needs at least one section','err'); return; }
  if(!confirm(`Delete the "${ED.score.sections[si].name}" section?`)) return;
  ED.score.sections.splice(si,1); ED.selSec=null; coEdMark(); coEdPaint();
}

/* ---- piano roll ---- */
async function coEdLoadNotes(){
  if(ED.notes) { coEdDrawRoll(); return; }
  try{
    const j=await get('/api/composer_notes?name='+encodeURIComponent(ED.name));
    if(j.error) throw new Error(j.error);
    ED.notes=j; coEdDrawRoll();
  }catch(e){ const w=$('coRollWrap'); if(w) w.innerHTML='<span class="coed-note" style="padding:8px;display:block">could not load notes: '+coEsc(e.message)+'</span>'; }
}
function coEdTrackNotes(){
  const t=ED.score.instruments[ED.sel]; if(!t) return null;
  if(t.notes_override) return t.notes_override;
  return (ED.notes&&ED.notes.notes&&ED.notes.notes[t.track])||null;
}
const ROLL={px:9,py:4,lo:24,hi:96};   // px per beat, px per semitone
function coEdDrawRoll(){
  const cv=$('coRoll'); if(!cv) return;
  const t=ED.score.instruments[ED.sel]; const notes=coEdTrackNotes();
  const bpb=ED.score.time_signature, bars=coEdBars(), beats=bars*bpb;
  const W=Math.max(900,Math.round(beats*ROLL.px)), rows=ROLL.hi-ROLL.lo, H=rows*ROLL.py;
  cv.width=W; cv.height=H; cv.style.width=W+'px'; cv.style.height=H+'px';
  const g=cv.getContext('2d');
  const css=k=>getComputedStyle(document.documentElement).getPropertyValue(k).trim();
  g.fillStyle=css('--surface-3')||'#1a1a20'; g.fillRect(0,0,W,H);
  // black-key rows
  g.fillStyle='rgba(0,0,0,.22)';
  for(let p=ROLL.lo;p<ROLL.hi;p++) if([1,3,6,8,10].indexOf(p%12)>=0)
    g.fillRect(0,(ROLL.hi-1-p)*ROLL.py,W,ROLL.py);
  // bar lines + section boundaries
  let acc=0;
  g.strokeStyle='rgba(255,255,255,.07)'; g.lineWidth=1;
  for(let b=0;b<=beats;b+=bpb){ g.beginPath(); g.moveTo(b*ROLL.px,0); g.lineTo(b*ROLL.px,H); g.stroke(); }
  g.strokeStyle='rgba(255,255,255,.30)';
  ED.score.sections.forEach(s=>{ acc+=s.bars; const x=acc*bpb*ROLL.px;
    g.beginPath(); g.moveTo(x,0); g.lineTo(x,H); g.stroke(); });
  if(!notes||!notes.length){
    g.fillStyle=css('--muted')||'#888'; g.font='12px system-ui';
    g.fillText(t&&t.muted?'track is muted':'no notes here — drag to draw some',10,18);
    return;
  }
  const col=ED_ROLE_COLOR[t.role]||'#6b7280';
  notes.forEach(n=>{
    const [b,d,p,v]=n; if(p<ROLL.lo||p>=ROLL.hi) return;
    g.fillStyle=col; g.globalAlpha=0.35+0.65*(v/127);
    g.fillRect(b*ROLL.px,(ROLL.hi-1-p)*ROLL.py,Math.max(2,d*ROLL.px-1),ROLL.py-1);
  });
  g.globalAlpha=1;
}
function coEdRollXY(ev){
  const cv=$('coRoll'), r=cv.getBoundingClientRect();
  return {beat:Math.max(0,(ev.clientX-r.left)/ROLL.px),
          pitch:Math.round(ROLL.hi-1-(ev.clientY-r.top)/ROLL.py)};
}
/* Editing a track's notes freezes them as notes_override — from then on that
   track replays exactly what you drew instead of being re-composed. */
function coEdEnsureOverride(){
  const t=ED.score.instruments[ED.sel];
  if(!t.notes_override) t.notes_override=JSON.parse(JSON.stringify(coEdTrackNotes()||[]));
  return t.notes_override;
}
function coEdClearNotes(){
  const t=ED.score.instruments[ED.sel];
  if(!confirm('Throw away your note edits and let the arranger re-compose this track?')) return;
  delete t.notes_override; coEdMark(); coEdPaint();
}
let ROLLDRAG=null;
document.addEventListener('mousedown',e=>{
  if(!e.target||e.target.id!=='coRoll') return;
  const p=coEdRollXY(e); const notes=coEdEnsureOverride();
  const hit=notes.findIndex(n=>n[2]===p.pitch&&p.beat>=n[0]&&p.beat<=n[0]+n[1]);
  if(hit>=0&&!e.shiftKey){ notes.splice(hit,1); coEdMark(); coEdDrawRoll(); return; }
  const snap=Math.floor(p.beat*4)/4;                       // 16th grid
  if(hit>=0&&e.shiftKey){ ROLLDRAG={mode:'move',i:hit,ox:p.beat-notes[hit][0],op:p.pitch}; return; }
  notes.push([snap,0.25,p.pitch,96]);
  ROLLDRAG={mode:'len',i:notes.length-1,start:snap};
  coEdMark(); coEdDrawRoll(); e.preventDefault();
});
document.addEventListener('mousemove',e=>{
  if(!ROLLDRAG) return;
  const notes=ED.score.instruments[ED.sel].notes_override; if(!notes) return;
  const p=coEdRollXY(e), n=notes[ROLLDRAG.i]; if(!n) return;
  if(ROLLDRAG.mode==='len') n[1]=Math.max(0.0625,Math.round((p.beat-ROLLDRAG.start)*4)/4||0.25);
  else { n[0]=Math.max(0,Math.round((p.beat-ROLLDRAG.ox)*4)/4); n[2]=p.pitch; }
  coEdDrawRoll();
});
document.addEventListener('mouseup',()=>{ if(ROLLDRAG){ ROLLDRAG=null; coEdPaint(); } });

async function coEdRerender(){
  if(!ED.score) return;
  $('coEdRender').disabled=true;
  setStatus('coStatus','re-rendering your edits…','run');
  $('coBarWrap').classList.remove('hide'); $('coBar').style.width='0%';
  stageBusy('coStage',true);
  try{
    await post('/api/composer_rerender',{name:ED.name,score:ED.score,
      stems:$('coStems').checked,polish:$('coPolish').checked,
      polish_denoise:+$('coPolishDen').value});
    ED.dirty=false; $('coEdDirty').classList.add('hide');
    coPoll();
  }catch(e){ setStatus('coStatus',e.message,'err'); }
  finally{ $('coEdRender').disabled=false; }
}

coSecsLabel();
coLoadLibrary();
/* a composition survives a page refresh: reattach to a running/finished job */
get('/api/composer_status').then(s=>{ if(s.state==='running'||s.plan) coPoll(); }).catch(()=>{});

/* ---- Music (ACE-Step) ---- */
function muAddTags(t){ const el=$('muTags'); const cur=el.value.trim(); el.value = cur ? (cur.replace(/,\s*$/,'')+', '+t) : t; el.focus(); }
function muInsert(marker){ const el=$('muLyrics'); const p=el.selectionStart??el.value.length;
  const ins=(p>0&&el.value[p-1]!=='\n'?'\n':'')+marker+'\n';
  el.value=el.value.slice(0,p)+ins+el.value.slice(p); el.focus(); el.selectionStart=el.selectionEnd=p+ins.length; }
function muInstChange(){ $('muLyricsWrap').classList.toggle('hide', $('muInst').checked); }
function muSecsLabel(){ const s=+$('muSecs').value; $('muSecsLab').textContent=Math.floor(s/60)+':'+String(s%60).padStart(2,'0'); }
function muModelChange(){
  const turbo = $('muModel').value==='turbo';
  $('muSteps').value = turbo ? 8 : 50;   $('muStepsLab').textContent = $('muSteps').value;
  $('muCfg').value   = turbo ? 1 : 7;    $('muCfgLab').textContent   = (+$('muCfg').value).toFixed(1);
}
function muFormatChange(){
  const f=$('muFormat').value, q=$('muQuality');
  $('muQualityWrap').classList.toggle('hide', f==='flac');
  q.innerHTML = f==='mp3' ? '<option value="V0">V0 (best VBR)</option><option value="320k">320k</option><option value="128k">128k</option>'
                          : '<option value="128k">128k</option><option value="192k">192k</option><option value="320k">320k</option><option value="96k">96k</option><option value="64k">64k</option>';
}
async function runMusic(){
  if(!$('muTags').value.trim()){ setStatus('muStatus','add some style tags first','err'); return; }
  setStatus('muStatus','composing… (turbo: ~10–40 s · sft: ~30–90 s)','run'); $('muOut').innerHTML='';
  stageBusy('muStage',true);
  try{
    const audio=await fileB64($('muSrc'));
    const fmt=$('muFormat').value;
    const j=await post('/api/music',{
      tags:$('muTags').value.trim(),
      lyrics: $('muInst').checked ? '' : $('muLyrics').value,
      model:$('muModel').value,
      seconds:+$('muSecs').value, steps:+$('muSteps').value, cfg:+$('muCfg').value,
      seed:+$('muSeed').value, shift:+$('muShift').value,
      sampler:$('muSampler').value, scheduler:$('muScheduler').value,
      bpm:+$('muBpm').value, keyscale:$('muKey').value, language:$('muLang').value,
      timesignature:$('muTimeSig').value,
      lm_temperature:+$('muLmTemp').value, lm_cfg:+$('muLmCfg').value,
      batch:+$('muBatch').value,
      format:fmt, quality: fmt==='flac' ? '' : $('muQuality').value,
      audio, denoise: audio ? +$('muDenoise').value : 1.0,
      filename: $('muSrc').files[0] ? $('muSrc').files[0].name : ''
    });
    $('muEmpty').classList.add('hide'); $('muStage').classList.add('hasresult');
    j.audios.forEach((u,i)=>{
      const d=document.createElement('div'); d.className='mutrack';
      const name=`acestep_${j.seed}${j.audios.length>1?'_'+(i+1):''}.${fmt==='opus'?'opus':fmt}`;
      d.innerHTML=`<div class="tr-head">🎵 Track ${i+1} · seed ${j.seed}<a class="dlbtn" style="margin-left:auto" download="${name}">⬇ Save track</a></div><audio controls></audio>`;
      d.querySelector('a').href=u; d.querySelector('audio').src=u;
      $('muOut').appendChild(d);
    });
    setStatus('muStatus',`done — seed ${j.seed} (enter it under Advanced to rerun a variation)`,'ok');
    seenStage('muStage');
  }catch(e){setStatus('muStatus',e.message,'err');}
  finally{ stageBusy('muStage',false); }
}

let STT_AUDIO=null;   // mic capture for Speech→Text (file input takes priority)
function sttFilePick(){ STT_AUDIO=null; $('sttPrev').classList.add('hide'); }
async function runSTT(){
  const fileA=await fileB64($('sttFile'));
  const a=fileA||STT_AUDIO;
  if(!a){setStatus('sttStatus','pick an audio file or record your mic','err');return;}
  setStatus('sttStatus','transcribing…','run'); stageBusy('sttStage',true);
  $('sttCopy').classList.add('hide'); $('sttMeta').classList.add('hide');
  try{
    const j=await post('/api/stt',{audio:a,filename:fileA?$('sttFile').files[0].name:'mic.wav'});
    $('sttEmpty').classList.add('hide'); $('sttStage').classList.add('hasresult');
    $('sttOut').textContent=j.text; $('sttOut').classList.remove('hide'); $('sttCopy').classList.remove('hide');
    const words=(j.text.trim().match(/\S+/g)||[]).length;
    $('sttMeta').innerHTML=`<span><b>${words}</b> words</span>`; $('sttMeta').classList.remove('hide');
    setStatus('sttStatus','done','ok'); seenStage('sttStage');
  }catch(e){setStatus('sttStatus',e.message,'err');}
  finally{ stageBusy('sttStage',false); }
}
function ttsEngineChange(){const cb=$('ttsEngine').value==='chatterbox'; $('ttsRefWrap').classList.toggle('hide',!cb); $('ttsVoiceWrap').classList.toggle('hide',cb);}
async function runTTS(){
  setStatus('ttsStatus','synthesizing…','run'); stageBusy('ttsStage',true); $('ttsOut').classList.add('hide');
  try{
    const engine=$('ttsEngine').value;
    const ref = engine==='chatterbox' ? await fileB64($('ttsRef')) : null;
    const j=await post('/api/tts',{text:$('ttsText').value.trim(),voice:$('ttsVoice').value.trim(),ref});
    $('ttsEmpty').classList.add('hide');
    $('ttsOut').src=j.audio; $('ttsOut').classList.remove('hide'); setStatus('ttsStatus','done','ok'); seenStage('ttsStage');
  }catch(e){setStatus('ttsStatus',e.message,'err');}
  finally{ stageBusy('ttsStage',false); }
}

// ---- generic audio helpers: mic + file -> WAV (decoded client-side) --------
function blobToDataURL(b){return new Promise((res,rej)=>{const r=new FileReader(); r.onload=()=>res(r.result); r.onerror=rej; r.readAsDataURL(b);});}
function encodeWAV(buf){
  const data=buf.getChannelData(0), sr=buf.sampleRate, n=data.length;
  const ab=new ArrayBuffer(44+n*2), dv=new DataView(ab); let p=0;
  const ws=s=>{for(let i=0;i<s.length;i++)dv.setUint8(p++,s.charCodeAt(i));};
  const u32=v=>{dv.setUint32(p,v,true);p+=4;}, u16=v=>{dv.setUint16(p,v,true);p+=2;};
  ws('RIFF');u32(36+n*2);ws('WAVE');ws('fmt ');u32(16);u16(1);u16(1);u32(sr);u32(sr*2);u16(2);u16(16);ws('data');u32(n*2);
  for(let i=0;i<n;i++){let s=Math.max(-1,Math.min(1,data[i]));dv.setInt16(p,s<0?s*0x8000:s*0x7fff,true);p+=2;}
  return new Blob([ab],{type:'audio/wav'});
}
async function decodeToWav(blob){
  const ab=await blob.arrayBuffer();
  const ctx=new (window.AudioContext||window.webkitAudioContext)();
  try{ const buf=await ctx.decodeAudioData(ab); return encodeWAV(buf); } finally{ ctx.close(); }
}

// ---- Voice Studio ----------------------------------------------------------
let SCRIPTS = [];                 // easy sentences from the server
let REC = {mr:null, slot:null};   // active recorder
const SLOTS = {                   // recording targets: btn id, preview id, status id
  vm:    {btn:'vmRec',   prev:'vmPrev', stat:'vmRunStat'},
  train: {btn:'vcRecBtn',prev:'vcPrev', stat:'vcProgress'},
  stt:   {btn:'sttRec',  prev:'sttPrev',stat:'sttStatus'},
};
let VM_AUDIO = null;              // captured "audio in" wav for Use-mode voice input
let TRAIN_WAV = null;            // last captured training clip (data URL)
let vcSentences = [], vcIdx = 0, vcSamples = [];

function vsModeChange(){const m=$('vsMode').value; $('vsUse').classList.toggle('hide',m!=='use'); $('vsCreate').classList.toggle('hide',m!=='create');
  const u=$('vsModeUse'),c=$('vsModeCreate'); if(u)u.classList.toggle('active',m==='use'); if(c)c.classList.toggle('active',m==='create');}
function vmInputChange(){const v=$('vmInput').value; $('vmTextWrap').classList.toggle('hide',v!=='text'); $('vmVoiceWrap').classList.toggle('hide',v!=='voice');}

async function loadVoices(){
  try{
    const j=await get('/api/voices'); const voices=j.voices||[];
    for(const selId of ['vsVoice','abVoiceModel']){
      const sel=$(selId); if(!sel) continue; const cur=sel.value; sel.innerHTML='';
      voices.forEach(v=>{const o=document.createElement('option'); o.value=v.name; o.textContent=v.name+(v.samples?` (${v.samples} clips)`:''); sel.appendChild(o);});
      if(!voices.length){const o=document.createElement('option'); o.value=''; o.textContent='— no voices yet, create one —'; sel.appendChild(o);}
      if(cur) sel.value=cur;
    }
    reeval();
  }catch(e){}
}

// ---- Audiobook -------------------------------------------------------------
let AB_SOURCE='story', abTimer=null;
// Kokoro's 54 built-in voices (id prefix = language: a=US b=British e=Spanish f=French
// h=Hindi i=Italian j=Japanese p=BR-Portuguese z=Mandarin; 2nd letter f=female m=male).
const KOKORO_VOICES={
  'US English — female':['af_heart','af_alloy','af_aoede','af_bella','af_jessica','af_kore','af_nicole','af_nova','af_river','af_sarah','af_sky'],
  'US English — male':['am_adam','am_echo','am_eric','am_fenrir','am_liam','am_michael','am_onyx','am_puck','am_santa'],
  'British — female':['bf_alice','bf_emma','bf_isabella','bf_lily'],
  'British — male':['bm_daniel','bm_fable','bm_george','bm_lewis'],
  'Spanish':['ef_dora','em_alex','em_santa'],'French':['ff_siwis'],
  'Hindi':['hf_alpha','hf_beta','hm_omega','hm_psi'],'Italian':['if_sara','im_nicola'],
  'Japanese':['jf_alpha','jf_gongitsune','jf_nezumi','jf_tebukuro','jm_kumo'],
  'Brazilian Portuguese':['pf_dora','pm_alex','pm_santa'],
  'Mandarin':['zf_xiaobei','zf_xiaoni','zf_xiaoxiao','zf_xiaoyi','zm_yunjian','zm_yunxi','zm_yunxia','zm_yunyang'],
};
(function(){ const fill=sel=>{ if(!sel) return;
    for(const [grp,ids] of Object.entries(KOKORO_VOICES)){ const og=document.createElement('optgroup'); og.label=grp;
      ids.forEach(id=>{const o=document.createElement('option'); o.value=id; o.textContent=id; og.appendChild(o);}); sel.appendChild(og); }
    sel.value='af_heart'; };
  fill($('abVoice')); fill($('ttsVoice'));
})();
function abEngineChange(){
  const e=$('abEngine').value;
  $('abVoiceWrap').classList.toggle('hide', e!=='kokoro');
  $('abVoiceModelWrap').classList.toggle('hide', e!=='voice');
  $('abZonosCtl').classList.toggle('hide', e!=='zonos');
  $('abChatterCtl').classList.toggle('hide', e!=='chatterbox');
  $('abRefWrap').classList.toggle('hide', !(e==='zonos'||e==='chatterbox'));  // cloning engines
  reeval();
}
async function abLoadRefs(){
  const sel=$('abRef'); if(!sel) return; const cur=sel.value; sel.innerHTML='';
  const def=document.createElement('option'); def.value=''; def.textContent='Default narrator'; sel.appendChild(def);
  try{ const j=await get('/api/refs'); if((j.refs||[]).length){ const og=document.createElement('optgroup'); og.label='Reference clips';
    j.refs.forEach(r=>{const o=document.createElement('option'); o.value='lib:'+r; o.textContent=r.replace(/\.(wav|mp3|m4a)$/i,''); og.appendChild(o);}); sel.appendChild(og);} }catch(e){}
  try{ const j=await get('/api/voices'); if((j.voices||[]).length){ const og=document.createElement('optgroup'); og.label='Clone a saved voice';
    j.voices.forEach(v=>{const o=document.createElement('option'); o.value='voice:'+v.name; o.textContent=v.name; og.appendChild(o);}); sel.appendChild(og);} }catch(e){}
  const up=document.createElement('option'); up.value='upload'; up.textContent='Upload a clip…'; sel.appendChild(up);
  if(cur) sel.value=cur;
}
function abRefChange(){ $('abRefUpWrap').classList.toggle('hide', $('abRef').value!=='upload'); }
async function abRefBody(){
  const e=$('abEngine').value; if(e!=='zonos'&&e!=='chatterbox') return {};
  const v=$('abRef').value;
  if(v.indexOf('lib:')===0) return {ref_lib:v.slice(4)};
  if(v.indexOf('voice:')===0) return {ref_voice:v.slice(6)};
  if(v==='upload'){ const f=$('abRefFile').files[0]; if(!f) return {}; const b64=await fileB64($('abRefFile')); return b64?{ref:b64}:{}; }
  return {};
}
function abControls(){
  const e=$('abEngine').value, c={speed:parseFloat($('abSpeed').value)};
  if(e==='zonos'){ c.pitch_std=parseFloat($('abPitch').value); const em=$('abEmotion').value; if(em) c.emotion=em; }
  if(e==='chatterbox'){ c.exaggeration=parseFloat($('abExag').value); }
  return c;
}
function abSetSource(src){
  AB_SOURCE=src;
  document.querySelectorAll('#abSourceSeg button').forEach(b=>b.classList.toggle('active',b.dataset.src===src));
  $('abStoryWrap').classList.toggle('hide', src!=='story');
  $('abTextWrap').classList.toggle('hide', src!=='text');
}
async function abRefreshStories(){
  try{ const j=await get('/api/story_list'); const sel=$('abStorySel'); const cur=sel.value; sel.innerHTML='';
    (j.stories||[]).forEach(s=>{const o=document.createElement('option'); o.value=s.id; o.textContent=s.title+' · '+s.beats+' beats'; sel.appendChild(o);});
    if(!j.stories||!j.stories.length){const o=document.createElement('option'); o.value=''; o.textContent='— no stories —'; sel.appendChild(o);}
    if(cur) sel.value=cur;
  }catch(e){}
}
/* ---------------- Lullaby Studio ---------------- */
let lbTimer=null, LB_MODE='remix';
function lbSetMode(m){
  LB_MODE=m;
  document.querySelectorAll('#lbModeSeg button').forEach(b=>b.classList.toggle('active',b.dataset.mode===m));
  $('lbRemixCtl').classList.toggle('hide',m!=='remix');
  $('lbPianoCtl').classList.toggle('hide',m!=='piano');
  $('lbMelodyCtl').classList.toggle('hide',m!=='melody-match');
  document.querySelectorAll('#lbStems .mmRoute').forEach(el=>el.classList.toggle('hide',m!=='melody-match'));
}
// Melody Match only: split the ticked Tracks-panel stems into two groups by
// each row's Route selector — 'melody' stems get mixed and traced onto the
// chosen instrument, 'arranged' stems instead go through the Piano engine's
// own transcribe+quantize+rebuild pipeline, then both are mixed together.
function lbMelodyRouting(){
  const solo={}, arranged={};
  document.querySelectorAll('#lbStems [data-stem]').forEach(row=>{
    if(!row.querySelector('input[type=checkbox]').checked) return;
    const level=+row.querySelector('input[type=range]').value;
    const route=row.querySelector('.mmRouteSel').value;
    (route==='arranged'?arranged:solo)[row.dataset.stem]=level;
  });
  return {solo, arranged};
}
let LB_NAME=null;
function lbFileChange(){
  const f=$('lbFile').files[0];
  document.querySelector('#lbFile + .filedrop-name').textContent = f ? f.name : '';
  LB_NAME=null; $('lbWorkbench').classList.add('hide'); $('lbRenderBtn').classList.add('hide');
  $('lbAnalyzeBtn').classList.remove('hide'); setStatus('lbStatus','','');
}
async function lbAnalyze(){
  const f=$('lbFile').files[0];
  if(!f){ setStatus('lbStatus','choose a song file first','err'); return; }
  if(f.size > 80*1024*1024){ setStatus('lbStatus','file too big (80MB max) — use an mp3 instead','err'); return; }
  const audio=await fileB64($('lbFile'));
  $('lbAnalyzeBtn').disabled=true; $('lbBarWrap').classList.remove('hide');
  $('lbFiles').innerHTML=''; $('lbEmpty').classList.add('hide'); $('lbBar').style.width='0%';
  setStatus('lbStatus','uploading & analyzing…','run');
  try{ await post('/api/lullaby_analyze',{audio, filename:f.name}); lbPoll(); }
  catch(e){ setStatus('lbStatus',e.message,'err'); $('lbAnalyzeBtn').disabled=false; }
}
async function lbRender(){
  if(!LB_NAME){ setStatus('lbStatus','analyze a song first','err'); return; }
  $('lbRenderBtn').disabled=true; $('lbBarWrap').classList.remove('hide');
  $('lbFiles').innerHTML=''; $('lbBar').style.width='0%';
  setStatus('lbStatus','starting render…','run');
  const tempo = LB_MODE==='melody-match' ? +$('lbMelodyTempo').value : +$('lbTempo').value;
  const routing = LB_MODE==='melody-match' ? lbMelodyRouting() : {solo:lbStemWeights(), arranged:null};
  const polish = LB_MODE==='melody-match' ? $('lbMelodyPolish').checked : $('lbPolish').checked;
  try{ await post('/api/lullaby_render',{name:LB_NAME, mode:LB_MODE,
        stems:routing.solo, arranged_stems:routing.arranged, focus:$('lbFocus').value,
        denoise:+$('lbSoft').value, slowdown:+$('lbSlow').value, tempo,
        instrument:$('lbInstrument').value, polish}); lbPoll(); }
  catch(e){ setStatus('lbStatus',e.message,'err'); $('lbRenderBtn').disabled=false; }
}
async function lbPoll(){
  try{
    const s=await get('/api/lullaby_status');
    const pct=s.total?Math.round((s.step/s.total)*100):0; $('lbBar').style.width=pct+'%';
    if(s.state==='running'){ setStatus('lbStatus',`Stage ${s.step}/${s.total} — ${s.current||'starting'}`,'run'); }
    else if(s.state==='error'){ setStatus('lbStatus','Error: '+s.message,'err'); }
    else if(s.state==='done' && s.phase==='analyze'){
      setStatus('lbStatus','✓ Analyzed — pick your tracks and an engine, then render','ok');
      $('lbBar').style.width='100%'; LB_NAME=s.name; await lbLoadInfo();
    }
    else if(s.state==='done'){ setStatus('lbStatus','✓ Lullaby complete','ok'); $('lbBar').style.width='100%'; }
    lbRenderFiles(s.files||[]);
    if(s.state==='running'){ lbTimer=setTimeout(lbPoll,2000); return; }
    $('lbAnalyzeBtn').disabled=false; $('lbRenderBtn').disabled=false;
  }catch(e){ lbTimer=setTimeout(lbPoll,3000); }
}
const LB_STEM_DEFAULTS={vocals:1, guitar:1, piano:1, other:1, bass:0, drums:0};
function lbStemWeights(){
  const w={};
  document.querySelectorAll('#lbStems [data-stem]').forEach(row=>{
    const on=row.querySelector('input[type=checkbox]').checked;
    w[row.dataset.stem]=on ? +row.querySelector('input[type=range]').value : 0;
  });
  return w;
}
async function lbLoadInfo(){
  const info=await get('/api/lullaby_info?name='+encodeURIComponent(LB_NAME));
  $('lbSummary').textContent=`${info.key} · ${info.tempo} bpm · ${info.bars.length} bars`;
  // unhide FIRST so clientWidth-based canvas sizing below measures real layout,
  // not a display:none parent (that 0-width readout was forcing a hardcoded
  // 600px fallback, which flex items can't shrink below — the row then
  // overflowed the sidebar and visually spilled under the output panel)
  $('lbWorkbench').classList.remove('hide');
  lbDrawWave($('lbWave'), info.waveform, info.bars);
  const sbox=$('lbStems'); sbox.innerHTML='';
  for(const [name,st] of Object.entries(info.stems||{})){
    const def=LB_STEM_DEFAULTS[name]!==undefined?LB_STEM_DEFAULTS[name]:1;
    const checkboxHtml=`<input type="checkbox" ${def>0?'checked':''} style="width:auto;flex:0 0 auto"
      onchange="this.closest('[data-stem]').style.opacity=this.checked?1:.45">`;
    const row=lbBuildScrubRow(name, st.preview, '#lbStems audio', checkboxHtml);
    row.insertAdjacentHTML('beforeend', `
      <div style="display:flex;align-items:center;gap:8px">
        <span class="note" style="flex:0 0 auto">Level</span>
        <input type="range" min="0" max="1" step="0.05" value="${def}" style="flex:1"
               title="level carried into the lullaby"
               oninput="this.nextElementSibling.textContent=Math.round(this.value*100)+'%'">
        <span class="note" style="flex:0 0 auto;width:34px;text-align:right">${Math.round(def*100)}%</span>
      </div>
      <div class="mmRoute hide" style="display:flex;align-items:center;gap:8px">
        <span class="note" style="flex:0 0 auto">Route</span>
        <select class="mmRouteSel" style="flex:1">
          <option value="melody" ${name==='vocals'?'selected':''}>→ Melody Match (traced onto the instrument)</option>
          <option value="arranged" ${name!=='vocals'?'selected':''}>→ Piano arrangement (rebuilt, quantized)</option>
        </select>
      </div>`);
    if(def<=0) row.style.opacity=.45;
    sbox.appendChild(row);
    // workbench is already unhidden above, so clientWidth is a real measurement
    // here — no more falling back to a hardcoded width that overflowed the column
    lbFinishScrubRow(row, st.peaks||[], '#lbStems audio');
  }
  $('lbRenderBtn').classList.remove('hide');
  $('lbAnalyzeBtn').classList.add('hide');
}
// Shared scrubbable stem-player builder — used by the Lullaby Tracks panel
// and the Track Splitter tab. headerExtraHtml slots in a control between the
// name and play button (e.g. the Lullaby include-checkbox); pass '' for none.
// Caller must append `row` to the DOM, add whatever belongs below the
// waveform (slider / download link), THEN call lbFinishScrubRow — the canvas
// needs real layout before it can size itself.
function lbBuildScrubRow(name, previewUrl, pauseSelector, headerExtraHtml){
  const row=document.createElement('div');
  row.className='card'; row.dataset.stem=name;
  row.style.cssText='padding:10px;gap:7px';
  row.innerHTML=`
    <div style="display:flex;align-items:center;gap:8px;min-width:0">
      ${headerExtraHtml||''}
      <b style="flex:0 0 auto;text-transform:capitalize;font-size:13px">${name}</b>
      <button type="button" class="rec scrubplay" style="flex:0 0 auto;margin-left:auto">▶</button>
    </div>
    <canvas height="72" style="width:100%;border-radius:6px;background:rgba(255,255,255,.04);cursor:pointer;touch-action:none"></canvas>
    <audio preload="metadata" src="${previewUrl}"></audio>`;
  const audio=row.querySelector('audio');
  row.querySelector('.scrubplay').onclick=()=>{
    if(audio.paused){
      document.querySelectorAll(pauseSelector).forEach(x=>{ if(x!==audio) x.pause(); });
      audio.play().catch(()=>{});
    } else audio.pause();
  };
  return row;
}
function lbFinishScrubRow(row, peaks, pauseSelector){
  const cv=row.querySelector('canvas'), audio=row.querySelector('audio');
  cv.width=Math.max(1,cv.clientWidth)*2;
  lbWireStemScrub(cv, audio, peaks, pauseSelector);
}
function lbWireStemScrub(cv, audio, peaks, pauseSelector){
  pauseSelector = pauseSelector || '#lbStems audio';
  const g=cv.getContext('2d');
  function draw(){
    const W=cv.width, H=cv.height;
    g.clearRect(0,0,W,H);
    const n=peaks.length, mx=Math.max(...peaks,0.01)||0.01;
    const barW=Math.max(1,W/n-1);
    // played portion in a brighter shade so scrubbed position reads at a glance
    const playedX=(audio.duration && isFinite(audio.duration))
      ? (audio.currentTime/audio.duration)*W : 0;
    for(let i=0;i<n;i++){
      const x=i/n*W;
      const h=Math.max(2,(peaks[i]/mx)*(H-8));
      g.fillStyle = x<playedX ? 'rgba(200,210,255,.85)' : 'rgba(160,175,255,.5)';
      g.fillRect(x,(H-h)/2,barW,h);
    }
    if(playedX>0){
      g.fillStyle='#fff';
      g.fillRect(Math.max(0,playedX-1.5),0,3,H);
    }
  }
  let raf=null;
  function loop(){ draw(); if(!audio.paused) raf=requestAnimationFrame(loop); }
  audio.addEventListener('play',()=>{ if(raf) cancelAnimationFrame(raf); loop(); });
  audio.addEventListener('pause',draw);
  audio.addEventListener('seeked',draw);
  audio.addEventListener('loadedmetadata',draw);
  function seekTo(evt){
    const rect=cv.getBoundingClientRect();
    const frac=Math.min(1,Math.max(0,(evt.clientX-rect.left)/rect.width));
    if(audio.duration && isFinite(audio.duration)) audio.currentTime=frac*audio.duration;
    draw();
  }
  cv.style.cursor='pointer';
  // pointer capture: once the drag starts on the canvas, this element keeps
  // receiving move/up events even if the cursor leaves it — no window-level
  // listeners needed, and it behaves correctly on touch devices too
  cv.addEventListener('pointerdown',e=>{
    cv.setPointerCapture(e.pointerId);
    document.querySelectorAll(pauseSelector).forEach(x=>{ if(x!==audio) x.pause(); });
    seekTo(e);
    audio.play().catch(()=>{});
  });
  cv.addEventListener('pointermove',e=>{ if(e.buttons & 1) seekTo(e); });
  draw();
}
// Play the checked tracks in a scrub-row container together, roughly in sync
// (shared by the Lullaby Tracks panel and Track Splitter). Re-derives "is
// anything playing" fresh on every click, so it self-corrects even if a
// row's own solo ▶ button paused the rest of the synced set in the meantime.
let SYNCPLAY_TIMER=null;
function playAllSelected(containerSel, btn){
  const rows=[...document.querySelectorAll(containerSel+' [data-stem]')];
  const all=rows.map(row=>({audio:row.querySelector('audio'), cb:row.querySelector('input[type=checkbox]')}))
                .filter(r=>r.audio);
  if(SYNCPLAY_TIMER){ clearInterval(SYNCPLAY_TIMER); SYNCPLAY_TIMER=null; }
  if(all.some(r=>!r.audio.paused)){
    all.forEach(r=>r.audio.pause());
    btn.textContent='▶ Play all selected';
    return;
  }
  const selected=all.filter(r=>!r.cb||r.cb.checked).map(r=>r.audio);
  if(!selected.length) return;
  const t=selected[0].currentTime||0;
  selected.forEach(a=>{ a.currentTime=t; });
  Promise.all(selected.map(a=>a.play().catch(()=>{})));
  btn.textContent='⏸ Stop';
  SYNCPLAY_TIMER=setInterval(()=>{
    if(selected.every(a=>a.paused||a.ended)){
      clearInterval(SYNCPLAY_TIMER); SYNCPLAY_TIMER=null; btn.textContent='▶ Play all selected'; return;
    }
    const ref=selected[0].currentTime;
    selected.forEach(a=>{ if(a!==selected[0]&&!a.paused&&Math.abs(a.currentTime-ref)>0.15) a.currentTime=ref; });
  },1000);
}

/* ---------------- Track Splitter ---------------- */
let SP_FORMAT='mp3', splTimer=null;
function spFileChange(){
  const f=$('spFile').files[0];
  document.querySelector('#spFile + .filedrop-name').textContent = f ? f.name : '';
}
function spSetFormat(fmt){
  SP_FORMAT=fmt;
  document.querySelectorAll('#spFormatSeg button').forEach(b=>b.classList.toggle('active',b.dataset.fmt===fmt));
}
async function spSplit(){
  const f=$('spFile').files[0];
  if(!f){ setStatus('splStatus','choose a song file first','err'); return; }
  if(f.size > 80*1024*1024){ setStatus('splStatus','file too big (80MB max) — use an mp3 instead','err'); return; }
  const audio=await fileB64($('spFile'));
  $('spGoBtn').disabled=true; $('splBarWrap').classList.remove('hide');
  $('spTracks').innerHTML=''; $('splEmpty').classList.add('hide'); $('splBar').style.width='0%';
  $('spZipBtn').classList.add('hide'); $('spPlayAllBtn').classList.add('hide');
  setStatus('splStatus','uploading & splitting…','run');
  try{ await post('/api/split_start',{audio, filename:f.name, format:SP_FORMAT}); splPoll(); }
  catch(e){ setStatus('splStatus',e.message,'err'); $('spGoBtn').disabled=false; }
}
async function splPoll(){
  try{
    const s=await get('/api/split_status');
    const pct=s.total?Math.round((s.step/s.total)*100):0; $('splBar').style.width=pct+'%';
    if(s.state==='running'){ setStatus('splStatus',`Stage ${s.step}/${s.total} — ${s.current||'starting'}`,'run'); }
    else if(s.state==='done'){
      setStatus('splStatus','✓ Tracks saved','ok'); $('splBar').style.width='100%';
      await spLoadResults(s.name, s.files||[]);
      spRefreshLibrary();
    }
    else if(s.state==='error'){ setStatus('splStatus','Error: '+s.message,'err'); }
    if(s.state==='running'){ splTimer=setTimeout(splPoll,2000); return; }
    $('spGoBtn').disabled=false;
  }catch(e){ splTimer=setTimeout(splPoll,3000); }
}
async function spLoadResults(name, files){
  let info={};
  try{ info=await get('/api/lullaby_info?name='+encodeURIComponent(name)); }catch(e){}
  const stems=info.stems||{};
  const order=['vocals','guitar','piano','other','bass','drums'];
  const sorted=[...files].sort((a,b)=>order.indexOf(a.split('.')[0])-order.indexOf(b.split('.')[0]));
  const box=$('spTracks'); box.innerHTML='';
  for(const fname of sorted){
    const stem=fname.split('.')[0];
    const st=stems[stem];
    // prefer the pre-computed preview+peaks from the shared analysis cache
    // (faster to load, has a real waveform); fall back to the saved file
    // itself if that cache is missing — still scrubbable via Range support
    const preview = st ? st.preview : `/splits/${encodeURIComponent(name)}/${fname}`;
    const checkboxHtml=`<input type="checkbox" checked title="include in Play all selected" style="width:auto;flex:0 0 auto">`;
    const row=lbBuildScrubRow(stem, preview, '#spTracks audio', checkboxHtml);
    const dlUrl=`/splits/${encodeURIComponent(name)}/${fname}`;
    row.insertAdjacentHTML('beforeend',
      `<a class="rec" style="text-decoration:none;text-align:center" href="${dlUrl}" download>⬇ ${fname}</a>`);
    box.appendChild(row);
    lbFinishScrubRow(row, (st&&st.peaks)||[], '#spTracks audio');
  }
  $('spZipBtn').href='/api/split_zip?name='+encodeURIComponent(name);
  $('spZipBtn').classList.remove('hide');
  $('spPlayAllBtn').classList.remove('hide');
}
async function spRefreshLibrary(){
  try{
    const j=await get('/api/split_list');
    const sel=$('spLibrary'); const cur=sel.value;
    sel.innerHTML='<option value="">— pick a previous split —</option>'+
      (j.splits||[]).map(s=>`<option value="${s.name}">${s.name}</option>`).join('');
    if(cur) sel.value=cur;
  }catch(e){}
}
async function spOpenLibrary(){
  const name=$('spLibrary').value;
  if(!name) return;
  try{
    const j=await get('/api/split_list');
    const entry=(j.splits||[]).find(s=>s.name===name);
    if(!entry) return;
    setStatus('splStatus','',''); $('splEmpty').classList.add('hide');
    await spLoadResults(name, entry.files||[]);
  }catch(e){ setStatus('splStatus',e.message,'err'); }
}

function lbDrawWave(cv, wf, bars){
  const W=cv.width=cv.clientWidth*2, H=cv.height, g=cv.getContext('2d');
  g.clearRect(0,0,W,H);
  if(!wf||!wf.peaks) return;
  const dur=wf.duration, peaks=wf.peaks, n=peaks.length, mx=Math.max(...peaks,0.01);
  const waveH=H-26;
  g.fillStyle='rgba(160,175,255,.55)';
  for(let i=0;i<n;i++){
    const h=Math.max(2,(peaks[i]/mx)*waveH);
    g.fillRect(i/n*W, (waveH-h)/2+2, Math.max(1,W/n-1), h);
  }
  // chord strip: alternating blocks labeled with the detected chord per bar
  (bars||[]).forEach((b,i)=>{
    const x0=b.start/dur*W;
    const x1=(bars[i+1]?bars[i+1].start:dur)/dur*W;
    g.fillStyle=i%2?'rgba(150,170,255,.16)':'rgba(150,170,255,.08)';
    g.fillRect(x0,waveH+4,x1-x0,20);
    if(x1-x0>26){ g.fillStyle='rgba(230,235,255,.75)'; g.font='11px system-ui';
      g.fillText(b.chord,x0+4,waveH+18); }
  });
}
function lbRenderFiles(files){
  const box=$('lbFiles'); if(!files.length){ box.innerHTML=''; return; }
  box.innerHTML=files.map(f=>{
    const url='/lullabies/'+f.split('/').map(encodeURIComponent).join('/');
    const name=f.split('/').pop();
    const isMp3=/\.mp3$/i.test(name);
    return `<div class="card" style="padding:12px;gap:8px">
      <div style="display:flex;align-items:center;gap:8px"><b style="font-size:13px">${isMp3?'🎹 ':'💾 '}${name}</b>
        <a class="rec" style="margin-left:auto;flex:0 0 auto;text-decoration:none" href="${url}" download>⬇</a></div>
      ${isMp3?`<audio controls style="width:100%" src="${url}"></audio>`:''}
    </div>`;
  }).join('');
}

function abLoadFile(){
  const f=$('abFile').files[0]; if(!f) return;
  const r=new FileReader(); r.onload=()=>{ $('abText').value=r.result; if(!$('abTitle').value) $('abTitle').value=f.name.replace(/\.(txt|md)$/i,''); }; r.readAsText(f);
}
async function abPreview(){
  if(!LOADED || !(LOADED.indexOf('tts:')===0 || LOADED.indexOf('voicemodel:')===0)){
    setStatus('abStatus','load a voice first, then preview','err'); return;
  }
  const sample="Here is a short preview of this narration voice, reading a line from your story.";
  setStatus('abStatus','synthesizing preview…','run'); $('abPreviewBtn').disabled=true;
  try{
    const j = LOADED.indexOf('voicemodel:')===0
      ? await post('/api/voicemodel',{text:sample, ...abControls()})
      : await post('/api/tts',{text:sample, voice:$('abVoice').value.trim(), ...abControls(), ...(await abRefBody())});
    const a=$('abPreviewAudio'); a.src=j.audio; a.classList.remove('hide'); a.play().catch(()=>{});
    setStatus('abStatus','preview ready — happy with this voice? then render','ok');
  }catch(e){ setStatus('abStatus',e.message,'err'); }
  finally{ $('abPreviewBtn').disabled=false; }
}
async function abGenerate(){
  if(!LOADED || !(LOADED.indexOf('tts:')===0 || LOADED.indexOf('voicemodel:')===0)){
    setStatus('abStatus','load a narration voice first','err'); return;
  }
  const body={source:AB_SOURCE, voice:$('abVoice').value.trim(), ...abControls(), ...(await abRefBody())};
  if(AB_SOURCE==='story'){ body.id=$('abStorySel').value; if(!body.id){setStatus('abStatus','pick a story','err');return;} }
  else { body.text=$('abText').value.trim(); body.title=$('abTitle').value.trim()||'Audiobook'; if(!body.text){setStatus('abStatus','paste or upload some text','err');return;} }
  $('abGenBtn').disabled=true; $('abBarWrap').classList.remove('hide'); $('abFiles').innerHTML='';
  setStatus('abStatus','starting…','run');
  try{ await post('/api/audiobook_start',body); abPoll(); }
  catch(e){ setStatus('abStatus',e.message,'err'); $('abGenBtn').disabled=false; }
}
async function abPoll(){
  try{
    const s=await get('/api/audiobook_status');
    const pct=s.total?Math.round((s.step/s.total)*100):0; $('abBar').style.width=pct+'%';
    if(s.state==='running'){ setStatus('abStatus',`Rendering ${s.step}/${s.total} — ${s.current||''}`,'run'); }
    else if(s.state==='done'){ setStatus('abStatus','✓ Audiobook complete','ok'); $('abBar').style.width='100%'; }
    else if(s.state==='error'){ setStatus('abStatus','Error: '+s.message,'err'); }
    abRenderFiles(s.files||[]);
    if(s.state==='running'){ abTimer=setTimeout(abPoll,2000); return; }
    $('abGenBtn').disabled=false;
  }catch(e){ abTimer=setTimeout(abPoll,3000); }
}
function abRenderFiles(files){
  const box=$('abFiles'); if(!files.length){ box.innerHTML=''; return; }
  box.innerHTML=files.map(f=>{
    const url='/audiobooks/'+f.split('/').map(encodeURIComponent).join('/');
    const name=f.split('/').pop();
    const full=/\.mp3$/.test(name) && !/^\d/.test(name);
    return `<div class="card" style="padding:12px;gap:8px">
      <div style="display:flex;align-items:center;gap:8px"><b style="font-size:13px">${full?'📚 ':'🎧 '}${name}</b>
      <a class="dlbtn" style="margin-left:auto;padding:5px 11px" href="${url}" download="${name}">⬇ Download</a></div>
      <audio controls preload="none" src="${url}" style="width:100%"></audio></div>`;
  }).join('');
}

// recording shared by Use-mode audio-in and Create-mode training clips
async function vsRec(slot){
  const cfg=SLOTS[slot], btn=$(cfg.btn);
  if(REC.mr && REC.slot===slot){ REC.mr.stop(); return; }
  if(REC.mr){ return; }
  try{
    const stream=await navigator.mediaDevices.getUserMedia({audio:{echoCancellation:false,noiseSuppression:false,autoGainControl:false}});
    const chunks=[], mr=new MediaRecorder(stream);
    mr.ondataavailable=e=>{if(e.data.size)chunks.push(e.data);};
    mr.onstop=async()=>{
      stream.getTracks().forEach(t=>t.stop());
      btn.textContent='🎙 Record'; btn.classList.remove('on'); REC={mr:null,slot:null};
      try{
        const wav=await decodeToWav(new Blob(chunks));
        $(cfg.prev).src=URL.createObjectURL(wav); $(cfg.prev).classList.remove('hide');
        const data=await blobToDataURL(wav);
        if(slot==='vm'){ VM_AUDIO=data; setStatus(cfg.stat,'captured','ok'); }
        else if(slot==='stt'){ STT_AUDIO=data; try{$('sttFile').value='';}catch(_){} setStatus(cfg.stat,'captured — click Transcribe','ok'); }
        else { TRAIN_WAV=data; $('vcNextBtn').disabled=false; $('vcRedoBtn').disabled=false; setStatus('vcProgress',`recorded — review, then Save & next  (${vcSamples.length}/${vcSentences.length} saved)`,'ok'); }
      }catch(e){ setStatus(cfg.stat,'could not process recording: '+e.message,'err'); }
    };
    REC={mr:mr,slot:slot}; mr.start(); btn.textContent='⏹ Stop'; btn.classList.add('on');
    setStatus(cfg.stat,'recording… click Stop when done','run');
  }catch(e){ setStatus(cfg.stat,'mic unavailable (needs HTTPS/localhost): '+e.message,'err'); }
}
async function vsPick(slot){
  const cfg=SLOTS[slot], f=$('vmFile').files[0]; if(!f)return;
  setStatus(cfg.stat,'processing…','run');
  try{ const wav=await decodeToWav(f); $(cfg.prev).src=URL.createObjectURL(wav); $(cfg.prev).classList.remove('hide'); VM_AUDIO=await blobToDataURL(wav); setStatus(cfg.stat,'ready','ok'); }
  catch(e){ setStatus(cfg.stat,'could not read file: '+e.message,'err'); }
}

// Use a voice -> speak
async function runVoiceModel(){
  const mode=$('vmInput').value;
  const body={};
  if(mode==='text'){ const t=$('vmText').value.trim(); if(!t){setStatus('vmRunStat','type some text','err');return;} body.text=t; }
  else { if(!VM_AUDIO){setStatus('vmRunStat','record or upload some audio first','err');return;} body.audio=VM_AUDIO; }
  setStatus('vmRunStat','speaking…','run'); $('vmOut').classList.add('hide'); $('vmDl').classList.add('hide'); $('vmHeard').classList.add('hide');
  try{
    const j=await post('/api/voicemodel',body);
    $('vmEmpty').classList.add('hide');
    if(mode==='voice' && j.text){ $('vmHeard').textContent='heard: “'+j.text+'”'; $('vmHeard').classList.remove('hide'); }
    $('vmOut').src=j.audio; $('vmOut').classList.remove('hide'); $('vmDl').href=j.audio; $('vmDl').classList.remove('hide');
    setStatus('vmRunStat','done','ok');
  }catch(e){ setStatus('vmRunStat',e.message,'err'); }
}

// Create a voice
function vcAmountChange(){ vcSentences = SCRIPTS.slice(0, +$('vcAmount').value); vcIdx=0; vcSamples=[]; vcShow(); updateTrainBtn(); }
function vcShow(){
  $('vcPrev').classList.add('hide'); TRAIN_WAV=null; $('vcNextBtn').disabled=true; $('vcRedoBtn').disabled=true;
  const pct=vcSentences.length?Math.round(vcSamples.length/vcSentences.length*100):0;
  if($('vcRing')){ $('vcRing').style.setProperty('--p',pct); $('vcRingText').textContent=pct+'%'; }
  if(vcIdx>=vcSentences.length){ $('vcSentence').textContent='✓ All sentences recorded. Click “Train voice”.'; setStatus('vcProgress',`${vcSamples.length}/${vcSentences.length} saved`,'ok'); return; }
  $('vcSentence').textContent = vcSentences[vcIdx];
  setStatus('vcProgress',`${vcSamples.length}/${vcSentences.length} saved · on sentence ${vcIdx+1}`,'');
}
function vcAccept(){
  if(!TRAIN_WAV) return;
  vcSamples.push({text:vcSentences[vcIdx], audio:TRAIN_WAV});
  vcIdx++; vcShow(); updateTrainBtn();
}
function updateTrainBtn(){ $('vcTrainBtn').disabled = vcSamples.length < 4 || !$('vcName').value.trim(); }

async function vcTrain(){
  const name=$('vcName').value.trim();
  if(!name){setStatus('vcTrainStat','give the voice a name','err');return;}
  if(vcSamples.length<4){setStatus('vcTrainStat','record at least 4 sentences','err');return;}
  const epochs = vcSamples.length>=35 ? 10 : (vcSamples.length>=25 ? 12 : 16);
  $('vcTrainBtn').disabled=true; $('vcBarWrap').classList.remove('hide');
  setStatus('vcTrainStat','starting… (frees other models; this runs a few minutes)','run');
  try{
    await post('/api/voice_train',{name,language:'en',epochs,samples:vcSamples});
    pollTrain(name);
  }catch(e){ setStatus('vcTrainStat',e.message,'err'); $('vcTrainBtn').disabled=false; }
}
async function pollTrain(name){
  try{
    const s=await get('/api/voice_train_status');
    const pct = s.total ? Math.round((s.epoch/s.total)*100) : 0;
    $('vcBar').style.width=pct+'%';
    if(s.state==='preparing'){ setStatus('vcTrainStat','preparing dataset…','run'); }
    else if(s.state==='training'){ setStatus('vcTrainStat',`training… epoch ${s.epoch}/${s.total}`,'run'); }
    else if(s.state==='done'){ setStatus('vcTrainStat','✓ done — “'+name+'” saved','ok'); $('vcBar').style.width='100%'; toast('Voice “'+name+'” trained','ok'); await loadVoices(); $('vsVoice').value=name; $('vsMode').value='use'; vsModeChange(); return; }
    else if(s.state==='error'){ setStatus('vcTrainStat','training failed: '+s.message,'err'); $('vcTrainBtn').disabled=false; return; }
    setTimeout(()=>pollTrain(name), 4000);
  }catch(e){ setTimeout(()=>pollTrain(name), 4000); }
}

// init
get('/api/scripts').then(s=>{SCRIPTS=s||[]; vcAmountChange();}).catch(()=>{});
$('vcName').addEventListener('input', updateTrainBtn);
loadVoices();
wireDrops();
abEngineChange(); abRefreshStories(); abLoadRefs();
spRefreshLibrary();
document.documentElement.style.setProperty('--viewtone','var(--hue-home)');
document.querySelectorAll('#imgAspects button').forEach(b=>b.addEventListener('click',()=>{
  $('imgSize').value=b.dataset.ar;
  document.querySelectorAll('#imgAspects button').forEach(x=>x.classList.toggle('active',x===b));
}));

// restore state on load + light poll so the global pill stays accurate
get('/api/status').then(s=>{LOADED=s.key; paint();}).catch(()=>paint());
setInterval(()=>{ if(BUSY) return; get('/api/status').then(s=>{ if(s.key!==LOADED){ LOADED=s.key; paint(); } }).catch(()=>{}); }, 5000);

// live GPU bar in the topbar (server proxies nvidia-smi at /api/gpu)
async function pollGpu(){
  try{
    const g=await get('/api/gpu'); if(!g.total) return;
    const pct=Math.round(g.used/g.total*100);
    $('gpuFill').style.width=pct+'%';
    $('gpuFill').className = pct>88?'danger':(pct>70?'warn':'');
    $('gpuText').textContent=(g.used/1024).toFixed(1)+' GB';
    $('gpuPill').title=`GPU VRAM ${(g.used/1024).toFixed(1)} / ${(g.total/1024).toFixed(0)} GB (${pct}%) · ${g.util}% utilisation`;
  }catch(e){}
}
pollGpu(); setInterval(pollGpu,5000);

// ============================ STORY MAKER ============================
const ST_COLORS = ['#e2683c','#3fb950','#58a6ff','#bc8cff','#f0883e','#ec6cb9','#39c5cf','#d29922','#7ee787','#ff7b72'];
const POVS = ['first-person','third-person limited','third-person omniscient','second-person'];
const TENSES = ['past','present'];
const LENGTHS = ['flash fiction','short story','novelette','novella','novel'];
let STORY=null, SEL=null, DRAG=null, stDirtyT=null, stGenTimer=null;
const esc = s => (s||'').replace(/[&<>"]/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c]));
const uid = p => (p||'id')+'_'+Math.random().toString(36).slice(2,9);

function stBlank(){ return {
  id:'story_'+Date.now().toString(36), title:'Untitled Story', genre:'', pov:'third-person limited',
  tense:'past', styleNotes:'', targetLength:'short story',
  characters:[], locations:[], timelines:[{id:'main',label:'Main timeline',kind:'main'}],
  beats:[], flashbacks:[], generation:{scenes:{},rollingSynopsis:'',fullStory:'',order:[]} };
}
function stNormalize(s){
  s.characters ||= []; s.locations ||= []; s.beats ||= []; s.flashbacks ||= [];
  s.timelines ||= [{id:'main',label:'Main timeline',kind:'main'}];
  if(!s.timelines.some(t=>t.kind!=='parallel')) s.timelines.unshift({id:'main',label:'Main timeline',kind:'main'});
  s.generation ||= {scenes:{},rollingSynopsis:'',fullStory:'',order:[]};
  s.characters.forEach(c=>{c.traits ||= [];});
  s.beats.forEach(b=>{b.characterIds ||= [];});
}
function stColor(n){ return ST_COLORS[n % ST_COLORS.length]; }

function stMarkDirty(){ if(!STORY) return; clearTimeout(stDirtyT); stDirtyT=setTimeout(stSave, 700); }
async function stSave(){ if(!STORY) return; try{ await post('/api/story_save',{story:STORY}); stSetStatus('saved','ok'); }catch(e){ stSetStatus('save failed','err'); } }
function stSetStatus(m,c){ const e=$('stStatus'); if(e){ e.textContent=m; e.className='status '+(c||''); } if(c==='err') toast(m,'err'); }

// ---- option builders -------------------------------------------------------
const stOpt = (list,sel)=>list.map(v=>`<option ${v===sel?'selected':''}>${esc(v)}</option>`).join('');
function stBeatOpts(selId, withStart){
  let o = withStart ? `<option value="" ${!selId?'selected':''}>From the start</option>` : '';
  o += STORY.beats.map(b=>`<option value="${b.id}" ${b.id===selId?'selected':''}>${esc(b.title||'(untitled beat)')}</option>`).join('');
  return o;
}

// ---- render: palette -------------------------------------------------------
function stRenderPalette(){
  const p=$('stPalette'); if(!p) return;
  let h='';
  h+=`<div class="palsec"><h3>Story</h3>
    <div class="fld"><label>Genre</label><input value="${esc(STORY.genre)}" oninput="stSet('genre',this.value)" placeholder="e.g. dark fantasy"></div>
    <div class="fld"><label>Point of view</label><select onchange="stSet('pov',this.value)">${stOpt(POVS,STORY.pov)}</select></div>
    <div class="fld"><label>Tense</label><select onchange="stSet('tense',this.value)">${stOpt(TENSES,STORY.tense)}</select></div>
    <div class="fld"><label>Target length</label><select onchange="stSet('targetLength',this.value)">${stOpt(LENGTHS,STORY.targetLength)}</select></div>
    <div class="fld"><label>Style notes</label><textarea oninput="stSet('styleNotes',this.value)" placeholder="tone, influences, things to avoid…">${esc(STORY.styleNotes)}</textarea></div>
  </div>`;
  // characters
  h+=`<div class="palsec"><h3>Characters <button class="miniadd" onclick="stAddChar()">＋</button></h3>`;
  if(!STORY.characters.length) h+=`<div class="palhint">Add a character, then drag them onto a plot point.</div>`;
  STORY.characters.forEach(c=>{ h+=`<div class="chip ${SEL&&SEL.type==='char'&&SEL.id===c.id?'sel':''}" draggable="true"
      ondragstart="stDragStart('char','${c.id}',event)" ondragend="stDragEnd()" onclick="stSelect('char','${c.id}')">
      <span class="cdot" style="background:${c.color}"></span>${esc(c.name||'Unnamed')}</div>`; });
  h+=`</div>`;
  // locations
  h+=`<div class="palsec"><h3>Locations <button class="miniadd" onclick="stAddLoc()">＋</button></h3>`;
  if(!STORY.locations.length) h+=`<div class="palhint">Drag a location onto a plot point to set its setting.</div>`;
  STORY.locations.forEach(l=>{ h+=`<div class="chip ${SEL&&SEL.type==='loc'&&SEL.id===l.id?'sel':''}" draggable="true"
      ondragstart="stDragStart('loc','${l.id}',event)" ondragend="stDragEnd()" onclick="stSelect('loc','${l.id}')">
      <span class="cdot" style="background:${l.color}"></span>📍 ${esc(l.name||'Unnamed')}</div>`; });
  h+=`</div>`;
  // tools
  h+=`<div class="palsec"><h3>Drag onto the timeline</h3>
    <div class="chip tool" draggable="true" ondragstart="stDragStart('tool-beat','',event)" ondragend="stDragEnd()">➕ Plot point — drop on a lane</div>
    <div class="chip tool" draggable="true" ondragstart="stDragStart('tool-flashback','',event)" ondragend="stDragEnd()">⤺ Flashback — drop on a beat</div>
    <button class="miniadd" style="margin-top:4px" onclick="stAddParallel()">＋ Parallel timeline</button>
  </div>`;
  p.innerHTML=h;
}

// ---- render: canvas --------------------------------------------------------
function stMainBeats(){ const ids=new Set(STORY.timelines.filter(t=>t.kind!=='parallel').map(t=>t.id));
  return STORY.beats.filter(b=>ids.has(b.timelineId)||['main','',null,undefined].includes(b.timelineId)).sort((a,b)=>(a.order||0)-(b.order||0)); }
function stLaneBeats(tlId){ return STORY.beats.filter(b=>b.timelineId===tlId).sort((a,b)=>(a.order||0)-(b.order||0)); }

function stBeatCard(b){
  const loc=STORY.locations.find(l=>l.id===b.locationId);
  const dots=(b.characterIds||[]).map(id=>{const c=STORY.characters.find(x=>x.id===id); return c?`<span class="cdot" style="background:${c.color}" title="${esc(c.name)}"></span>`:''; }).join('');
  let pips='';
  STORY.characters.forEach(c=>(c.traits||[]).forEach(t=>{ if(t.acquiredAtBeat===b.id) pips+=`<span class="pip">✦ ${esc(c.name)}: ${esc(t.label||'trait')}</span>`; }));
  return `<div class="beat ${SEL&&SEL.type==='beat'&&SEL.id===b.id?'sel':''}" data-beat="${b.id}" draggable="true"
      ondragstart="stDragStart('beat-reorder','${b.id}',event)" ondragend="stDragEnd()"
      ondragover="stOver(event)" ondragleave="stOut(event)" ondrop="stBeatDrop('${b.id}',event)"
      onclick="stSelect('beat','${b.id}')">
      <div class="bt">${esc(b.title||'Untitled beat')}</div>
      ${b.summary?`<div class="bs">${esc(b.summary)}</div>`:''}
      <div class="brow">${dots}${loc?`<span class="bloc">📍 ${esc(loc.name)}</span>`:''}</div>
      ${pips?`<div class="brow">${pips}</div>`:''}
    </div>`;
}
function stLaneHtml(t){
  const beats=stLaneBeats(t.id);
  const isPar=t.kind==='parallel';
  const anchor=isPar?STORY.beats.find(b=>b.id===t.anchorBeatId):null;
  let h=`<div class="lane"><div class="lanehead ${isPar?'parallel':''}">
     <span class="ltag">${isPar?'⥂ ':''}${esc(t.label)}</span>`;
  if(isPar) h+=`<span>simultaneous with “${esc(anchor?anchor.title:'—')}”</span>
     <button class="iconbtn" style="margin-left:auto;padding:3px 8px" onclick="stDelTimeline('${t.id}')">✕</button>`;
  h+=`</div><div class="track" ondragover="stOver(event)" ondragleave="stOut(event)" ondrop="stLaneDrop('${t.id}',event)">`;
  h+=beats.map(stBeatCard).join('');
  h+=`<div class="dropadd" ondragover="stOver(event)" ondragleave="stOut(event)" ondrop="stLaneDrop('${t.id}',event)" onclick="stAddBeat('${t.id}')">＋ plot point</div>`;
  h+=`</div></div>`;
  return h;
}
function stRenderCanvas(){
  const c=$('stCanvas'); if(!c) return;
  const mains=STORY.timelines.filter(t=>t.kind!=='parallel');
  const pars=STORY.timelines.filter(t=>t.kind==='parallel');
  let h='';
  mains.forEach(t=>{ h+=stLaneHtml(t); });
  h+=`<div class="stfblane" id="stFbLane"></div>`;
  pars.forEach(t=>{ h+=stLaneHtml(t); });
  if(!STORY.beats.length) h+=`<div class="emptyhint">Drag “Plot point” here (or click ＋ plot point) to start your timeline.</div>`;
  c.innerHTML=h;
  requestAnimationFrame(stDrawFlashbacks);
}
function stDrawFlashbacks(){
  const lane=$('stFbLane'); if(!lane) return; lane.innerHTML='';
  const lr=lane.getBoundingClientRect();
  STORY.flashbacks.forEach(fb=>{
    const s=document.querySelector(`.beat[data-beat="${fb.startAnchorBeatId}"]`);
    const e=document.querySelector(`.beat[data-beat="${fb.endAnchorBeatId}"]`)||s;
    if(!s) return;
    const sr=s.getBoundingClientRect(), er=e.getBoundingClientRect();
    const left=Math.min(sr.left,er.left)-lr.left, right=Math.max(sr.right,er.right)-lr.left;
    const bar=document.createElement('div'); bar.className='stfb'; bar.style.left=left+'px'; bar.style.width=Math.max(26,right-left)+'px';
    bar.textContent='⤺ '+(fb.label||'Flashback');
    bar.onclick=()=>stSelect('flashback',fb.id);
    const h1=document.createElement('div'); h1.className='stfbh'; h1.style.left='-6px'; h1.title='drag: flashback start'; h1.onmousedown=ev=>stFbDrag(ev,fb,'startAnchorBeatId');
    const h2=document.createElement('div'); h2.className='stfbh'; h2.style.right='-6px'; h2.title='drag: flashback end'; h2.onmousedown=ev=>stFbDrag(ev,fb,'endAnchorBeatId');
    bar.appendChild(h1); bar.appendChild(h2); lane.appendChild(bar);
  });
}
function stFbDrag(ev,fb,which){
  ev.preventDefault(); ev.stopPropagation();
  const beats=[...document.querySelectorAll('.beat')];
  const move=e=>{ let best=null,bd=1e9; beats.forEach(el=>{const r=el.getBoundingClientRect();const d=Math.abs(e.clientX-(r.left+r.width/2));if(d<bd){bd=d;best=el;}}); if(best){ fb[which]=best.dataset.beat; stDrawFlashbacks(); } };
  const up=()=>{ document.removeEventListener('mousemove',move); document.removeEventListener('mouseup',up); stMarkDirty(); };
  document.addEventListener('mousemove',move); document.addEventListener('mouseup',up);
}

// ---- drag & drop -----------------------------------------------------------
function stDragStart(kind,id,ev){ DRAG={kind,id}; try{ev.dataTransfer.setData('text/plain',kind);ev.dataTransfer.effectAllowed='all';}catch(e){} }
function stDragEnd(){ DRAG=null; document.querySelectorAll('.over').forEach(e=>e.classList.remove('over')); }
function stOver(ev){ if(!DRAG) return; ev.preventDefault(); ev.currentTarget.classList.add('over'); }
function stOut(ev){ ev.currentTarget.classList.remove('over'); }
function stBeatDrop(beatId,ev){
  ev.preventDefault(); ev.stopPropagation(); ev.currentTarget.classList.remove('over');
  const b=STORY.beats.find(x=>x.id===beatId); if(!b||!DRAG) return;
  if(DRAG.kind==='char'){ b.characterIds=b.characterIds||[]; if(!b.characterIds.includes(DRAG.id)) b.characterIds.push(DRAG.id); }
  else if(DRAG.kind==='loc'){ b.locationId=DRAG.id; }
  else if(DRAG.kind==='tool-flashback'){ stAddFlashback(beatId); }
  else if(DRAG.kind==='beat-reorder'){ stReorder(DRAG.id, beatId); }
  DRAG=null; stMarkDirty(); stRenderCanvas(); stRenderInspector();
}
function stLaneDrop(tlId,ev){
  ev.preventDefault(); ev.currentTarget.classList.remove('over');
  if(DRAG && DRAG.kind==='tool-beat'){ stAddBeat(tlId); }
  DRAG=null;
}
function stReorder(dragId,targetId){
  const d=STORY.beats.find(b=>b.id===dragId), t=STORY.beats.find(b=>b.id===targetId);
  if(!d||!t||d===t) return;
  d.timelineId=t.timelineId;
  const lane=stLaneBeats(t.timelineId).filter(b=>b.id!==dragId);
  const ti=lane.findIndex(b=>b.id===targetId);
  lane.splice(ti,0,d);
  lane.forEach((b,i)=>b.order=i);
}

// ---- create / mutate -------------------------------------------------------
function stSet(k,v){ STORY[k]=v; stMarkDirty(); }
function stAddChar(){ const c={id:uid('ch'),name:'New character',role:'',color:stColor(STORY.characters.length),description:'',traits:[]}; STORY.characters.push(c); stSelect('char',c.id); stMarkDirty(); stRenderPalette(); }
function stAddLoc(){ const l={id:uid('loc'),name:'New location',color:stColor(STORY.locations.length+3),description:''}; STORY.locations.push(l); stSelect('loc',l.id); stMarkDirty(); stRenderPalette(); }
function stAddBeat(tlId){ const order=stLaneBeats(tlId).length; const b={id:uid('beat'),timelineId:tlId,order,type:'plot',title:'New plot point',summary:'',notes:'',locationId:null,characterIds:[]}; STORY.beats.push(b); stSelect('beat',b.id); stMarkDirty(); stRenderCanvas(); }
function stAddParallel(){ const anchor = (SEL&&SEL.type==='beat')?SEL.id : (stMainBeats().slice(-1)[0]||{}).id;
  if(!anchor){ stSetStatus('add a main plot point first','err'); return; }
  const n=STORY.timelines.filter(t=>t.kind==='parallel').length+1;
  STORY.timelines.push({id:uid('tl'),label:'Parallel '+n,kind:'parallel',anchorBeatId:anchor}); stMarkDirty(); stRenderCanvas(); }
function stDelTimeline(id){ STORY.beats=STORY.beats.filter(b=>b.timelineId!==id); STORY.timelines=STORY.timelines.filter(t=>t.id!==id); stMarkDirty(); stRenderCanvas(); }
function stAddFlashback(beatId){ const fb={id:uid('fb'),label:'Flashback',summary:'',startAnchorBeatId:beatId,endAnchorBeatId:beatId}; STORY.flashbacks.push(fb); stSelect('flashback',fb.id); stMarkDirty(); stRenderCanvas(); }

// ---- selection + inspector -------------------------------------------------
function stSelect(type,id){ SEL={type,id}; stRenderPalette(); stRenderCanvas(); stRenderInspector(); }
function stRenderInspector(){
  const box=$('stInspector'); if(!box) return;
  if(!SEL){ box.innerHTML=`<div class="emptyhint">Select a character, location, plot point, or flashback to edit it here.</div>`; return; }
  if(SEL.type==='char'){ const c=STORY.characters.find(x=>x.id===SEL.id); if(!c){SEL=null;return stRenderInspector();}
    let traits=(c.traits||[]).map((t,i)=>`<div class="traitrow">
      <input value="${esc(t.label)}" oninput="stTrait('${c.id}',${i},'label',this.value)" placeholder="ability / trait">
      <select onchange="stTrait('${c.id}',${i},'kind',this.value)">${stOpt(['trait','ability'],t.kind||'trait')}</select>
      <select onchange="stTrait('${c.id}',${i},'acquiredAtBeat',this.value)">${stBeatOpts(t.acquiredAtBeat,true)}</select>
      <button class="iconbtn" onclick="stTraitDel('${c.id}',${i})">✕</button></div>`).join('');
    box.innerHTML=`<h3>Character</h3>
      <div class="fld"><label>Name</label><input value="${esc(c.name)}" oninput="stEdit('char','${c.id}','name',this.value)"></div>
      <div class="fld"><label>Role</label><input value="${esc(c.role)}" oninput="stEdit('char','${c.id}','role',this.value)" placeholder="protagonist, mentor…"></div>
      <div class="fld"><label>Colour</label><div class="swatches">${ST_COLORS.map(col=>`<span class="sw ${c.color===col?'on':''}" style="background:${col}" onclick="stEdit('char','${c.id}','color','${col}');stRenderInspector();"></span>`).join('')}</div></div>
      <div class="fld"><label>Description</label><textarea oninput="stEdit('char','${c.id}','description',this.value)" placeholder="appearance, voice, motivation…">${esc(c.description)}</textarea></div>
      <div class="fld"><label>Traits &amp; abilities <span class="note">(set when each is acquired)</span></label>${traits||'<div class="palhint">none yet</div>'}
        <button class="iconbtn" style="margin-top:4px" onclick="stTraitAdd('${c.id}')">＋ trait / ability</button></div>
      <button class="del" onclick="stRemove('char','${c.id}')">Delete character</button>`;
  } else if(SEL.type==='loc'){ const l=STORY.locations.find(x=>x.id===SEL.id); if(!l){SEL=null;return stRenderInspector();}
    box.innerHTML=`<h3>Location</h3>
      <div class="fld"><label>Name</label><input value="${esc(l.name)}" oninput="stEdit('loc','${l.id}','name',this.value)"></div>
      <div class="fld"><label>Colour</label><div class="swatches">${ST_COLORS.map(col=>`<span class="sw ${l.color===col?'on':''}" style="background:${col}" onclick="stEdit('loc','${l.id}','color','${col}');stRenderInspector();"></span>`).join('')}</div></div>
      <div class="fld"><label>Description</label><textarea oninput="stEdit('loc','${l.id}','description',this.value)" placeholder="atmosphere, sensory detail…">${esc(l.description)}</textarea></div>
      <button class="del" onclick="stRemove('loc','${l.id}')">Delete location</button>`;
  } else if(SEL.type==='beat'){ const b=STORY.beats.find(x=>x.id===SEL.id); if(!b){SEL=null;return stRenderInspector();}
    const chars=STORY.characters.map(c=>`<label class="cbrow"><input type="checkbox" ${(b.characterIds||[]).includes(c.id)?'checked':''} onchange="stBeatChar('${b.id}','${c.id}',this.checked)"><span class="cdot" style="background:${c.color}"></span>${esc(c.name)}</label>`).join('')||'<div class="palhint">no characters yet</div>';
    const locs=`<option value="" ${!b.locationId?'selected':''}>— none —</option>`+STORY.locations.map(l=>`<option value="${l.id}" ${l.id===b.locationId?'selected':''}>${esc(l.name)}</option>`).join('');
    const sc=(STORY.generation&&STORY.generation.scenes)?STORY.generation.scenes[b.id]:null;
    box.innerHTML=`<h3>Plot point</h3>
      <div class="fld"><label>Title</label><input value="${esc(b.title)}" oninput="stEdit('beat','${b.id}','title',this.value)"></div>
      <div class="fld"><label>What happens</label><textarea oninput="stEdit('beat','${b.id}','summary',this.value)" placeholder="the beat to dramatize…">${esc(b.summary)}</textarea></div>
      <div class="fld"><label>Setting</label><select onchange="stEdit('beat','${b.id}','locationId',this.value||null)">${locs}</select></div>
      <div class="fld"><label>Characters present</label>${chars}</div>
      <div class="fld"><label>Notes for the writer</label><textarea oninput="stEdit('beat','${b.id}','notes',this.value)" placeholder="optional guidance…">${esc(b.notes)}</textarea></div>
      ${sc&&sc.prose?`<div class="fld"><label>Generated scene</label><div class="out" style="max-height:160px;overflow:auto">${esc(sc.prose).slice(0,1200)}</div><button class="iconbtn" style="margin-top:5px" onclick="stRegen('${b.id}')">↻ Regenerate this scene</button></div>`:''}
      <button class="del" onclick="stRemove('beat','${b.id}')">Delete plot point</button>`;
  } else if(SEL.type==='flashback'){ const fb=STORY.flashbacks.find(x=>x.id===SEL.id); if(!fb){SEL=null;return stRenderInspector();}
    box.innerHTML=`<h3>Flashback</h3>
      <div class="fld"><label>Label</label><input value="${esc(fb.label)}" oninput="stEdit('flashback','${fb.id}','label',this.value)"></div>
      <div class="fld"><label>What the flashback shows</label><textarea oninput="stEdit('flashback','${fb.id}','summary',this.value)" placeholder="the remembered scene…">${esc(fb.summary)}</textarea></div>
      <div class="fld"><label>Starts at</label><select onchange="stEdit('flashback','${fb.id}','startAnchorBeatId',this.value)">${stBeatOpts(fb.startAnchorBeatId,false)}</select></div>
      <div class="fld"><label>Returns at</label><select onchange="stEdit('flashback','${fb.id}','endAnchorBeatId',this.value)">${stBeatOpts(fb.endAnchorBeatId,false)}</select></div>
      <div class="note">Tip: drag the handles on the purple bar to move the anchors.</div>
      <button class="del" onclick="stRemove('flashback','${fb.id}')">Delete flashback</button>`;
  }
}
function stEdit(type,id,k,v){ const map={char:'characters',loc:'locations',beat:'beats',flashback:'flashbacks'}; const o=STORY[map[type]].find(x=>x.id===id); if(!o) return; o[k]=v; stMarkDirty(); stRenderPalette(); stRenderCanvas(); }
function stBeatChar(beatId,charId,on){ const b=STORY.beats.find(x=>x.id===beatId); b.characterIds=b.characterIds||[]; if(on){ if(!b.characterIds.includes(charId)) b.characterIds.push(charId);} else b.characterIds=b.characterIds.filter(x=>x!==charId); stMarkDirty(); stRenderCanvas(); }
function stTraitAdd(cid){ const c=STORY.characters.find(x=>x.id===cid); c.traits=c.traits||[]; c.traits.push({id:uid('tr'),label:'',kind:'ability',acquiredAtBeat:''}); stMarkDirty(); stRenderInspector(); }
function stTrait(cid,i,k,v){ const c=STORY.characters.find(x=>x.id===cid); if(!c||!c.traits[i])return; c.traits[i][k]=v; stMarkDirty(); stRenderCanvas(); }
function stTraitDel(cid,i){ const c=STORY.characters.find(x=>x.id===cid); c.traits.splice(i,1); stMarkDirty(); stRenderInspector(); stRenderCanvas(); }
function stRemove(type,id){ const map={char:'characters',loc:'locations',beat:'beats',flashback:'flashbacks'}; STORY[map[type]]=STORY[map[type]].filter(x=>x.id!==id);
  if(type==='beat'){ STORY.flashbacks=STORY.flashbacks.filter(f=>f.startAnchorBeatId!==id&&f.endAnchorBeatId!==id); STORY.timelines=STORY.timelines.filter(t=>t.anchorBeatId!==id||t.kind!=='parallel'); STORY.characters.forEach(c=>(c.traits||[]).forEach(t=>{if(t.acquiredAtBeat===id)t.acquiredAtBeat='';})); }
  SEL=null; stMarkDirty(); stRenderPalette(); stRenderCanvas(); stRenderInspector(); }

// ---- modes + generation ----------------------------------------------------
function stMode(m){ $('stPlan').classList.toggle('hide',m!=='plan'); $('stRead').classList.toggle('hide',m!=='read');
  $('stModePlan').classList.toggle('active',m==='plan'); $('stModeRead').classList.toggle('active',m==='read');
  if(m==='read') stRenderReader(); }
function stRenderReader(){
  const r=$('stReader'); if(!r||!STORY) return;
  const gen=STORY.generation||{}; const order=gen.order&&gen.order.length?gen.order:Object.keys(gen.scenes||{});
  if(!order.length){ r.innerHTML=`<div class="emptyhint">No story yet. Build your timeline, load the Language model (tick <b>🔓 Unlocked</b> on the Language tab first for uncensored prose), then click <b>✨ Generate story</b>.</div>`; return; }
  let h=`<h1>${esc(STORY.title)}</h1>`;
  order.forEach(key=>{ const sc=(gen.scenes||{})[key]; if(!sc||!sc.prose) return;
    const cls=sc.kind&&sc.kind.startsWith('flashback')?'fb':(sc.parallel?'par':'');
    const tag=sc.kind&&sc.kind.startsWith('flashback')?'⤺ flashback':(sc.parallel?'⥂ meanwhile':'scene');
    const paras=sc.prose.split(/\n{2,}/).map(x=>`<p>${esc(x.trim())}</p>`).join('');
    h+=`<div class="scene ${cls}"><div class="scenehd">${tag} · ${esc(sc.label||'')}<button class="regen" onclick="stRegen('${key}')">↻ regenerate</button></div>${paras}</div>`; });
  r.innerHTML=h;
}
async function stGenerate(){
  if(!STORY) return;
  if(!STORY.beats.length){ stSetStatus('add a plot point first','err'); return; }
  await stSave(); stMode('read');
  $('stGenBtn').disabled=true; $('stGenProg').textContent='Starting…'; $('stGenProg').className='status run';
  try{ await post('/api/story_generate',{id:STORY.id}); stPollGen(); }
  catch(e){ $('stGenProg').textContent=e.message; $('stGenProg').className='status err'; $('stGenBtn').disabled=false; }
}
async function stPollGen(){
  try{
    const s=await get('/api/story_status');
    if(s.sid===STORY.id){
      if(s.state==='running'){ $('stGenProg').textContent=`Writing scene ${s.step}/${s.total} — ${s.current||''}`; $('stGenProg').className='status run'; }
      else if(s.state==='done'){ $('stGenProg').textContent='✓ Story complete'; $('stGenProg').className='status ok'; }
      else if(s.state==='error'){ $('stGenProg').textContent='Error: '+s.message; $('stGenProg').className='status err'; }
      try{ const j=await get('/api/story_load?id='+encodeURIComponent(STORY.id)); STORY=j.story; stRenderReader(); }catch(e){}
      if(s.state==='running'){ stGenTimer=setTimeout(stPollGen,2500); return; }
    }
    $('stGenBtn').disabled=false; stRefreshList();
  }catch(e){ stGenTimer=setTimeout(stPollGen,3000); }
}
async function stRegen(key){
  if(!STORY) return;
  try{ const j=await post('/api/story_regen_scene',{id:STORY.id,key}); const r=await get('/api/story_load?id='+encodeURIComponent(STORY.id)); STORY=r.story; stRenderReader(); stRenderInspector(); }
  catch(e){ stSetStatus(e.message,'err'); }
}
async function stExport(){
  if(!STORY) return;
  try{ const j=await post('/api/story_export',{id:STORY.id}); const blob=new Blob([j.markdown||''],{type:'text/markdown'});
    const a=document.createElement('a'); a.href=URL.createObjectURL(blob); a.download=((j.title||'story').replace(/[^\w-]+/g,'_')||'story')+'.md'; a.click(); }
  catch(e){ stSetStatus(e.message,'err'); }
}

// ---- library + lifecycle ---------------------------------------------------
function stTitleChange(){ if(!STORY) return; STORY.title=$('stTitle').value; stMarkDirty(); }
async function stRefreshList(){ try{ const j=await get('/api/story_list'); const sel=$('stLoadSel'); const cur=STORY?STORY.id:'';
    sel.innerHTML=''; (j.stories||[]).forEach(s=>{ const o=document.createElement('option'); o.value=s.id; o.textContent=s.title+' · '+s.beats+' beats'; sel.appendChild(o); });
    if(!j.stories||!j.stories.length){ const o=document.createElement('option'); o.value=''; o.textContent='— no saved stories —'; sel.appendChild(o); }
    if(cur) sel.value=cur; }catch(e){} }
async function stOpen(){ const id=$('stLoadSel').value; if(!id) return; try{ const j=await get('/api/story_load?id='+encodeURIComponent(id)); STORY=j.story; stNormalize(STORY); SEL=null; stMode('plan'); stRenderAll(); }catch(e){ stSetStatus(e.message,'err'); } }
function stNew(){ STORY=stBlank(); SEL=null; stMode('plan'); stRenderAll(); stSave().then(stRefreshList); }
async function stDelete(){ if(!STORY) return; if(!confirm('Delete “'+STORY.title+'”? This cannot be undone.')) return; const id=STORY.id; try{ await post('/api/story_delete',{id}); }catch(e){} STORY=stBlank(); SEL=null; stRenderAll(); stRefreshList(); }
function stRenderAll(){ if(!STORY) return; $('stTitle').value=STORY.title||''; stRenderPalette(); stRenderCanvas(); stRenderInspector(); }

STORY=stBlank(); stRenderAll(); stRefreshList();
</script>
</body>
</html>
```

## File 7 of 27 — `%USERPROFILE%\local-ai-studio\studio_gui.pyw`

```python
#!/usr/bin/env python3
"""Local AI Studio - Control Panel (visual GUI).

A small tkinter desktop app to start / stop / restart and live-monitor the whole
Local AI Studio stack, so you can confirm at a glance that everything it needs is
actually running:

    Ollama          :11434   Language / Story Maker LLM backend
    ComfyUI headless:8188    Image Generate / Edit backend
    Studio server   :8800    the web UI  (http://127.0.0.1:8800)
    Tailscale Serve          optional remote HTTPS access

Pure stdlib + tkinter - no pip installs. Services are launched detached, so they
keep running if you close this panel. The per-tab STT / TTS / Voice-Studio workers
are spawned on demand by the Studio itself (one resident model at a time); this
panel shows which one is currently loaded.
"""
from __future__ import annotations

import json
import os
import queue
import subprocess
import sys
import threading
import time
import urllib.request
import webbrowser
import tkinter as tk
from tkinter import ttk

# --------------------------------------------------------------------------
# Configuration (paths verified on this machine)
# --------------------------------------------------------------------------
HERE        = os.path.dirname(os.path.abspath(__file__))
LOGDIR      = os.path.join(HERE, "logs")
os.makedirs(LOGDIR, exist_ok=True)

LOCALAPPDATA = os.environ.get("LOCALAPPDATA", "")
USERPROFILE  = os.environ.get("USERPROFILE", os.path.expanduser("~"))
PROGRAMFILES = os.environ.get("ProgramFiles", r"C:\Program Files")

OLLAMA_EXE  = os.path.join(LOCALAPPDATA, "Programs", "Ollama", "ollama.exe")
COMFY_DIR   = os.path.join(USERPROFILE, "comfyui-src")
COMFY_PY    = os.path.join(USERPROFILE, "comfyui-src", ".venv", "Scripts", "python.exe")
COMFY_BASE  = os.path.join(USERPROFILE, "Documents", "ComfyUI")
STUDIO_MAIN = os.path.join(HERE, "server.py")
STUDIO_PY   = sys.executable
TAILSCALE   = os.path.join(PROGRAMFILES, "Tailscale", "tailscale.exe")
NVIDIA_SMI  = os.path.join(os.environ.get("WINDIR", r"C:\Windows"), "System32", "nvidia-smi.exe")

OLLAMA_HEALTH = "http://127.0.0.1:11434/api/tags"
OLLAMA_PS     = "http://127.0.0.1:11434/api/ps"
COMFY_HEALTH  = "http://127.0.0.1:8188/system_stats"
STUDIO_HEALTH = "http://127.0.0.1:8800/health"
STUDIO_STATUS = "http://127.0.0.1:8800/api/status"
STUDIO_STOP   = "http://127.0.0.1:8800/api/stop"
STUDIO_URL    = "http://127.0.0.1:8800"

# Windows process-creation flags
CREATE_NO_WINDOW         = 0x08000000
DETACHED_PROCESS         = 0x00000008
CREATE_NEW_PROCESS_GROUP = 0x00000200

# Colours
BG      = "#1e1f24"
PANEL   = "#26282f"
FG      = "#e6e6e6"
MUTED   = "#9aa0a8"
GREEN   = "#3ec27e"
RED     = "#e2554e"
YELLOW  = "#e9b94a"
GREY    = "#5a5f68"
ACCENT  = "#5b8def"


# --------------------------------------------------------------------------
# Backend helpers (no GUI)
# --------------------------------------------------------------------------
def _noconsole():
    return {"creationflags": CREATE_NO_WINDOW} if os.name == "nt" else {}


def http_ok(url, timeout=2):
    try:
        with urllib.request.urlopen(url, timeout=timeout) as r:
            return 200 <= r.getcode() < 500
    except urllib.error.HTTPError:
        return True   # server answered (even if 4xx) => it's alive
    except Exception:
        return False


def http_json(url, timeout=2):
    try:
        with urllib.request.urlopen(url, timeout=timeout) as r:
            return json.loads(r.read().decode("utf-8"))
    except Exception:
        return None


def http_post(url, body=b"{}", timeout=10):
    try:
        req = urllib.request.Request(url, data=body, method="POST",
                                     headers={"Content-Type": "application/json"})
        urllib.request.urlopen(req, timeout=timeout).read()
        return True
    except Exception:
        return False


def listening_pid(port):
    """PID owning the LISTENING socket on `port`, via netstat (no extra deps)."""
    try:
        out = subprocess.run(["netstat", "-ano", "-p", "TCP"],
                             capture_output=True, text=True, **_noconsole()).stdout
    except Exception:
        return None
    for line in out.splitlines():
        p = line.split()
        if len(p) >= 5 and p[0] == "TCP" and p[3] == "LISTENING" and p[1].endswith(f":{port}"):
            try:
                return int(p[4])
            except ValueError:
                pass
    return None


def kill_tree(pid):
    """Force-kill a process and its children (taskkill /T -> handles the Studio's
    on-demand STT/TTS worker subprocesses)."""
    try:
        subprocess.run(["taskkill", "/PID", str(pid), "/F", "/T"],
                       capture_output=True, **_noconsole())
        return True
    except Exception:
        return False


def spawn(cmd, cwd=None, env=None, logname="proc"):
    """Launch a windowless background process; stdout/stderr -> logs/<logname>.log.
    CREATE_NO_WINDOW, not DETACHED_PROCESS: venv pythons here are uv launcher shims
    that spawn the real interpreter as a *child* — under a detached (console-less)
    parent that child allocates a fresh console and Windows Terminal pops a window.
    CREATE_NO_WINDOW gives the shim an invisible console the child inherits."""
    out = open(os.path.join(LOGDIR, f"{logname}.out.log"), "ab", buffering=0)
    err = open(os.path.join(LOGDIR, f"{logname}.err.log"), "ab", buffering=0)
    flags = CREATE_NO_WINDOW | CREATE_NEW_PROCESS_GROUP if os.name == "nt" else 0
    return subprocess.Popen(cmd, cwd=cwd, env=env, stdout=out, stderr=err,
                            stdin=subprocess.DEVNULL, creationflags=flags, close_fds=True)


def gpu_stats():
    if not os.path.exists(NVIDIA_SMI):
        return None
    try:
        line = subprocess.run(
            [NVIDIA_SMI, "--query-gpu=memory.used,memory.total,utilization.gpu,temperature.gpu",
             "--format=csv,noheader,nounits"],
            capture_output=True, text=True, **_noconsole()).stdout.strip().splitlines()
        if not line:
            return None
        used, total, util, temp = [x.strip() for x in line[0].split(",")]
        return {"used": int(used), "total": int(total), "util": util, "temp": temp}
    except Exception:
        return None


# --------------------------------------------------------------------------
# Service descriptors
# --------------------------------------------------------------------------
class Service:
    def __init__(self, key, name, port, health):
        self.key, self.name, self.port, self.health = key, name, port, health


SERVICES = [
    Service("ollama", "Ollama",   11434, OLLAMA_HEALTH),
    Service("comfy",  "ComfyUI",  8188,  COMFY_HEALTH),
    Service("studio", "Studio",   8800,  STUDIO_HEALTH),
]


# --------------------------------------------------------------------------
# Start / stop logic (run on worker threads)
# --------------------------------------------------------------------------
def wait_up(url, timeout, log):
    t0 = time.time()
    while time.time() - t0 < timeout:
        if http_ok(url, 2):
            return True
        time.sleep(0.8)
    return False


def start_ollama(log):
    if http_ok(OLLAMA_HEALTH):
        log("Ollama already running.");  return True
    if not os.path.exists(OLLAMA_EXE):
        log(f"ERROR: ollama.exe not found: {OLLAMA_EXE}");  return False
    log("Starting Ollama ...")
    spawn([OLLAMA_EXE, "serve"], logname="ollama")
    ok = wait_up(OLLAMA_HEALTH, 30, log)
    log("Ollama up." if ok else "Ollama did NOT come up in 30s (see logs/ollama.err.log).")
    return ok


def start_comfy(log):
    if http_ok(COMFY_HEALTH):
        log("ComfyUI already running.");  return True
    if not (os.path.exists(COMFY_PY) and os.path.exists(COMFY_DIR)):
        log("ERROR: ComfyUI venv/dir not found.");  return False
    log("Starting ComfyUI headless (model load ~30s) ...")
    env = os.environ.copy(); env["PYTHONUTF8"] = "1"
    spawn([COMFY_PY, "main.py", "--listen", "127.0.0.1", "--port", "8188",
           "--base-directory", COMFY_BASE], cwd=COMFY_DIR, env=env, logname="comfyui")
    ok = wait_up(COMFY_HEALTH, 90, log)
    log("ComfyUI up." if ok else "ComfyUI did NOT come up in 90s (see logs/comfyui.err.log).")
    return ok


def start_studio(log, lan=False):
    if http_ok(STUDIO_HEALTH):
        log("Studio already running.");  return True
    if not os.path.exists(STUDIO_MAIN):
        log(f"ERROR: server.py not found: {STUDIO_MAIN}");  return False
    log("Starting Studio server ...")
    env = os.environ.copy()
    env["STUDIO_HOST"] = "0.0.0.0" if lan else "127.0.0.1"
    spawn([STUDIO_PY, STUDIO_MAIN], cwd=HERE, env=env, logname="studio")
    ok = wait_up(STUDIO_HEALTH, 20, log)
    log(f"Studio up -> {STUDIO_URL}" if ok else "Studio did NOT come up in 20s (see logs/studio.err.log).")
    return ok


def stop_service(svc, log, graceful=False):
    if graceful and svc.key == "studio" and http_ok(STUDIO_HEALTH):
        http_post(STUDIO_STOP)         # unload resident worker, free VRAM
    pid = listening_pid(svc.port)
    if not pid:
        log(f"{svc.name} not running.");  return True
    kill_tree(pid)
    log(f"{svc.name} stopped (pid {pid}).")
    return True


def tailscale_serve(log):
    if not os.path.exists(TAILSCALE):
        log("Tailscale not found.");  return False
    try:
        subprocess.run([TAILSCALE, "serve", "--bg", "8800"], capture_output=True, **_noconsole())
        log("Tailscale Serve publishing 8800 (HTTPS, tailnet-only).")
        return True
    except Exception as e:
        log(f"Tailscale serve failed: {e}");  return False


def tailscale_publishing():
    if not os.path.exists(TAILSCALE):
        return None
    try:
        out = subprocess.run([TAILSCALE, "serve", "status"],
                             capture_output=True, text=True, **_noconsole()).stdout
        return "8800" in out
    except Exception:
        return None


# --------------------------------------------------------------------------
# GUI
# --------------------------------------------------------------------------
class App:
    def __init__(self, root):
        self.root = root
        self.busy = False
        self.polling = False
        self.q = queue.Queue()              # worker threads -> main thread (tk is not thread-safe)
        self.remote = tk.BooleanVar(value=False)
        self.lan = tk.BooleanVar(value=False)
        self.auto = tk.BooleanVar(value=True)
        self.rows = {}
        self._build()
        self._pump()                        # start the main-thread UI queue drain
        self.refresh()                      # immediate
        self._schedule()                    # periodic

    # ---- UI construction -------------------------------------------------
    def _build(self):
        r = self.root
        r.title("Local AI Studio - Control Panel")
        r.configure(bg=BG)
        r.geometry("680x560")
        r.minsize(620, 500)

        head = tk.Frame(r, bg=BG)
        head.pack(fill="x", padx=16, pady=(14, 6))
        tk.Label(head, text="Local AI Studio", bg=BG, fg=FG,
                 font=("Segoe UI Semibold", 16)).pack(side="left")
        self.clock = tk.Label(head, text="", bg=BG, fg=MUTED, font=("Segoe UI", 9))
        self.clock.pack(side="right")

        # control bar
        bar = tk.Frame(r, bg=BG)
        bar.pack(fill="x", padx=16, pady=(0, 8))
        self.btn_start = self._btn(bar, "Start All", self.on_start_all, ACCENT)
        self.btn_start.pack(side="left")
        self.btn_stop = self._btn(bar, "Stop All", self.on_stop_all, "#3a3d45")
        self.btn_stop.pack(side="left", padx=(8, 0))
        self.btn_restart = self._btn(bar, "Restart", self.on_restart, "#3a3d45")
        self.btn_restart.pack(side="left", padx=(8, 0))
        self._btn(bar, "Open Studio", lambda: webbrowser.open(STUDIO_URL), "#3a3d45").pack(side="left", padx=(8, 0))

        opt = tk.Frame(r, bg=BG)
        opt.pack(fill="x", padx=16)
        for var, txt in ((self.lan, "LAN (0.0.0.0)"), (self.remote, "Tailscale remote"),
                         (self.auto, "Auto-refresh")):
            tk.Checkbutton(opt, text=txt, variable=var, bg=BG, fg=MUTED, selectcolor=PANEL,
                           activebackground=BG, activeforeground=FG, bd=0,
                           highlightthickness=0, font=("Segoe UI", 9)).pack(side="left", padx=(0, 14))

        # service rows
        card = tk.Frame(r, bg=PANEL)
        card.pack(fill="x", padx=16, pady=12)
        for svc in SERVICES:
            self.rows[svc.key] = self._service_row(card, svc)
        self.rows["tailscale"] = self._service_row(card, Service("tailscale", "Tailscale", 0, ""))

        # GPU
        gpu = tk.Frame(r, bg=BG)
        gpu.pack(fill="x", padx=16)
        self.gpu_label = tk.Label(gpu, text="GPU: -", bg=BG, fg=MUTED,
                                  font=("Consolas", 9), anchor="w")
        self.gpu_label.pack(side="left")
        self.gpu_bar = ttk.Progressbar(gpu, length=180, maximum=100)
        self.gpu_bar.pack(side="right", pady=4)

        # log
        tk.Label(r, text="Activity log", bg=BG, fg=MUTED, anchor="w",
                 font=("Segoe UI", 9)).pack(fill="x", padx=16, pady=(8, 0))
        logwrap = tk.Frame(r, bg=BG)
        logwrap.pack(fill="both", expand=True, padx=16, pady=(2, 14))
        self.log_text = tk.Text(logwrap, bg="#15161a", fg="#cdd2da", bd=0,
                                font=("Consolas", 9), height=8, wrap="word",
                                insertbackground=FG, state="disabled")
        sb = ttk.Scrollbar(logwrap, command=self.log_text.yview)
        self.log_text.configure(yscrollcommand=sb.set)
        sb.pack(side="right", fill="y")
        self.log_text.pack(side="left", fill="both", expand=True)

    def _btn(self, parent, text, cmd, bg):
        b = tk.Button(parent, text=text, command=cmd, bg=bg, fg="white",
                      activebackground=bg, activeforeground="white", bd=0,
                      relief="flat", padx=14, pady=6, font=("Segoe UI Semibold", 9),
                      cursor="hand2")
        return b

    def _service_row(self, parent, svc):
        row = tk.Frame(parent, bg=PANEL)
        row.pack(fill="x", padx=14, pady=8)
        dot = tk.Canvas(row, width=14, height=14, bg=PANEL, highlightthickness=0)
        oval = dot.create_oval(2, 2, 12, 12, fill=GREY, outline="")
        dot.pack(side="left")
        name = tk.Label(row, text=svc.name, bg=PANEL, fg=FG, width=10, anchor="w",
                        font=("Segoe UI Semibold", 10))
        name.pack(side="left", padx=(8, 0))
        detail = tk.Label(row, text="checking...", bg=PANEL, fg=MUTED, anchor="w",
                          font=("Segoe UI", 9))
        detail.pack(side="left", fill="x", expand=True, padx=(8, 8))
        rec = {"dot": dot, "oval": oval, "detail": detail, "svc": svc}
        if svc.key == "tailscale":
            b = tk.Button(row, text="Serve", command=lambda: self._run(self._do_serve),
                          bg="#3a3d45", fg="white", bd=0, relief="flat", padx=10, pady=3,
                          font=("Segoe UI", 8), cursor="hand2")
            b.pack(side="right")
        else:
            tk.Button(row, text="Stop", command=lambda s=svc: self._run(lambda log: self._do_stop_one(s, log)),
                      bg="#3a3d45", fg="white", bd=0, relief="flat", padx=10, pady=3,
                      font=("Segoe UI", 8), cursor="hand2").pack(side="right", padx=(6, 0))
            tk.Button(row, text="Start", command=lambda s=svc: self._run(lambda log: self._do_start_one(s, log)),
                      bg="#3a3d45", fg="white", bd=0, relief="flat", padx=10, pady=3,
                      font=("Segoe UI", 8), cursor="hand2").pack(side="right")
        return rec

    # ---- main-thread UI queue (tk must only be touched from main thread) -
    def _pump(self):
        try:
            while True:
                fn = self.q.get_nowait()
                try:
                    fn()
                except Exception:
                    pass
        except queue.Empty:
            pass
        self.root.after(50, self._pump)

    def ui(self, fn):
        """Schedule a tk-touching callable to run on the main thread."""
        self.q.put(fn)

    # ---- logging (thread-safe via queue) --------------------------------
    def log(self, msg):
        def _append():
            self.log_text.configure(state="normal")
            self.log_text.insert("end", f"[{time.strftime('%H:%M:%S')}] {msg}\n")
            self.log_text.see("end")
            self.log_text.configure(state="disabled")
        self.ui(_append)

    def set_dot(self, key, color):
        rec = self.rows.get(key)
        if rec:
            self.ui(lambda: rec["dot"].itemconfig(rec["oval"], fill=color))

    def set_detail(self, key, text, color=MUTED):
        rec = self.rows.get(key)
        if rec:
            self.ui(lambda: rec["detail"].configure(text=text, fg=color))

    # ---- action runner ---------------------------------------------------
    def _run(self, fn):
        """Run a backend action on a worker thread; disable top buttons while busy."""
        if self.busy:
            self.log("Busy - please wait for the current action to finish.")
            return
        self.busy = True
        for b in (self.btn_start, self.btn_stop, self.btn_restart):
            b.configure(state="disabled")

        def worker():
            try:
                fn(self.log)
            except Exception as e:
                self.log(f"ERROR: {e}")
            finally:
                self.busy = False
                self.ui(lambda: [b.configure(state="normal")
                                 for b in (self.btn_start, self.btn_stop, self.btn_restart)])
                self.ui(self.refresh)
        threading.Thread(target=worker, daemon=True).start()

    # ---- button handlers -------------------------------------------------
    def on_start_all(self):
        self._run(self._do_start_all)

    def on_stop_all(self):
        self._run(self._do_stop_all)

    def on_restart(self):
        def seq(log):
            self._do_stop_all(log)
            time.sleep(2)
            self._do_start_all(log)
        self._run(seq)

    def _do_start_all(self, log):
        log("=== Starting stack ===")
        start_ollama(log)
        start_comfy(log)
        start_studio(log, lan=self.lan.get())
        if self.remote.get():
            tailscale_serve(log)
        log("Start sequence done.")

    def _do_stop_all(self, log):
        log("=== Stopping stack ===")
        stop_service(SERVICES[2], log, graceful=True)   # studio (frees workers first)
        stop_service(SERVICES[1], log)                  # comfy
        log("Ollama left running (use the per-row Stop button to stop it).")

    def _do_start_one(self, svc, log):
        {"ollama": lambda: start_ollama(log),
         "comfy":  lambda: start_comfy(log),
         "studio": lambda: start_studio(log, lan=self.lan.get())}[svc.key]()

    def _do_stop_one(self, svc, log):
        stop_service(svc, log, graceful=(svc.key == "studio"))

    def _do_serve(self, log):
        tailscale_serve(log)

    # ---- status polling --------------------------------------------------
    def _schedule(self):
        if self.auto.get():
            self.refresh()
        self.root.after(4000, self._schedule)

    def refresh(self):
        if self.polling:
            return
        self.polling = True
        threading.Thread(target=self._poll, daemon=True).start()

    def _poll(self):
        try:
            # Ollama
            if http_ok(OLLAMA_HEALTH):
                ps = http_json(OLLAMA_PS) or {}
                resident = [m.get("name") for m in ps.get("models", [])]
                tags = http_json(OLLAMA_HEALTH) or {}
                n = len(tags.get("models", []))
                detail = ("resident: " + ", ".join(resident)) if resident else f"{n} model(s) installed, none in VRAM"
                self.set_dot("ollama", GREEN); self.set_detail("ollama", detail, FG)
            else:
                self.set_dot("ollama", RED); self.set_detail("ollama", ":11434 not answering")

            # ComfyUI
            if http_ok(COMFY_HEALTH):
                st = http_json(COMFY_HEALTH) or {}
                dev = (st.get("devices") or [{}])[0]
                if dev.get("vram_total"):
                    free = dev.get("vram_free", 0) / 1e9
                    tot = dev.get("vram_total", 0) / 1e9
                    detail = f"VRAM free {free:.1f}/{tot:.1f} GB"
                else:
                    detail = "running"
                self.set_dot("comfy", GREEN); self.set_detail("comfy", detail, FG)
            else:
                self.set_dot("comfy", RED); self.set_detail("comfy", ":8188 not answering")

            # Studio
            if http_ok(STUDIO_HEALTH):
                stat = http_json(STUDIO_STATUS) or {}
                worker = stat.get("key")
                detail = f"{STUDIO_URL}  |  worker: {worker}" if worker else f"{STUDIO_URL}  |  idle"
                self.set_dot("studio", GREEN); self.set_detail("studio", detail, FG)
            else:
                self.set_dot("studio", RED); self.set_detail("studio", ":8800 not answering")

            # Tailscale
            pub = tailscale_publishing()
            if pub is None:
                self.set_dot("tailscale", GREY); self.set_detail("tailscale", "not installed / unavailable")
            elif pub:
                self.set_dot("tailscale", GREEN); self.set_detail("tailscale", "Serve publishing 8800 (HTTPS)", FG)
            else:
                self.set_dot("tailscale", GREY); self.set_detail("tailscale", "Serve off (tick 'Tailscale remote' + Start)")

            # GPU
            g = gpu_stats()
            if g:
                free = g["total"] - g["used"]
                pct = int(g["used"] / g["total"] * 100) if g["total"] else 0
                txt = f"GPU: {g['used']}/{g['total']} MB used ({free} MB free)  util {g['util']}%  {g['temp']}C"
                self.ui(lambda: (self.gpu_label.configure(text=txt),
                                 self.gpu_bar.configure(value=pct)))
            self.ui(lambda: self.clock.configure(text=time.strftime("%H:%M:%S")))
        finally:
            self.polling = False


def main():
    root = tk.Tk()
    try:
        ttk.Style().theme_use("clam")
    except Exception:
        pass
    App(root)
    root.mainloop()


if __name__ == "__main__":
    main()
```

## File 8 of 27 — `%USERPROFILE%\local-ai-studio\studioctl.ps1`

```powershell
<#
.SYNOPSIS
    Control + monitor the Local AI Studio stack (Ollama + ComfyUI + Studio server).

.DESCRIPTION
    One tool to start, stop, restart and monitor everything the Local AI Studio
    web app needs in order to actually work:

        1. Ollama          :11434  (Language / Story Maker LLM backend)
        2. ComfyUI headless:8188   (Image Generate / Edit backend)
        3. Studio server   :8800   (the web UI -> http://127.0.0.1:8800)
        4. Tailscale Serve         (optional remote HTTPS access, -Remote)

    The per-tab STT / TTS / Voice-Studio workers are NOT long-running services:
    the Studio's own Manager spawns them on demand (one resident model at a time)
    and frees them on Stop, so this tool does not pre-start them. 'status' will
    show which worker (if any) the Studio currently has loaded.

.PARAMETER Command
    start | stop | restart | status | monitor

.PARAMETER Remote
    (start) Also publish the Studio over Tailscale Serve (HTTPS, tailnet-only).

.PARAMETER IncludeOllama
    (stop) Also stop the Ollama server. By default Ollama is left running
    because it is a shared background service used by other tools.

.PARAMETER Lan
    (start) Bind the Studio to 0.0.0.0 (LAN-visible) instead of 127.0.0.1.

.PARAMETER Interval
    (monitor) Seconds between refreshes. Default 5.

.EXAMPLE
    .\studioctl.ps1 start
    .\studioctl.ps1 start -Remote
    .\studioctl.ps1 status
    .\studioctl.ps1 monitor -Interval 3
    .\studioctl.ps1 stop
    .\studioctl.ps1 stop -IncludeOllama
#>
[CmdletBinding()]
param(
    [Parameter(Position = 0)]
    [ValidateSet('start', 'stop', 'restart', 'status', 'monitor')]
    [string]$Command = 'status',

    [switch]$Remote,
    [switch]$IncludeOllama,
    [switch]$Lan,
    [int]$Interval = 5
)

$ErrorActionPreference = 'Stop'
$script:Failures = 0   # services that failed to reach the desired state this run

# --------------------------------------------------------------------------
# Configuration (paths verified against this machine)
# --------------------------------------------------------------------------
$HERE      = Split-Path -Parent $MyInvocation.MyCommand.Definition
$LOGDIR    = Join-Path $HERE 'logs'
if (-not (Test-Path $LOGDIR)) { New-Item -ItemType Directory -Path $LOGDIR -Force | Out-Null }

$OLLAMA_EXE   = Join-Path $env:LOCALAPPDATA 'Programs\Ollama\ollama.exe'
$COMFY_DIR    = Join-Path $env:USERPROFILE 'comfyui-src'
$COMFY_PY     = Join-Path $env:USERPROFILE 'comfyui-src\.venv\Scripts\python.exe'
$COMFY_BASE   = Join-Path $env:USERPROFILE 'Documents\ComfyUI'
$STUDIO_PY    = (Get-Command python -ErrorAction SilentlyContinue).Source
$STUDIO_MAIN  = Join-Path $HERE 'server.py'
$TAILSCALE    = Join-Path $env:ProgramFiles 'Tailscale\tailscale.exe'

# service definitions ------------------------------------------------------
$SERVICES = [ordered]@{
    Ollama = @{ Name = 'Ollama';   Port = 11434; Health = 'http://127.0.0.1:11434/api/tags' }
    Comfy  = @{ Name = 'ComfyUI';  Port = 8188;  Health = 'http://127.0.0.1:8188/system_stats' }
    Studio = @{ Name = 'Studio';   Port = 8800;  Health = 'http://127.0.0.1:8800/health' }
}

# --------------------------------------------------------------------------
# Helpers
# --------------------------------------------------------------------------
function Write-Status($label, $state, $detail) {
    $color = switch ($state) {
        'UP'      { 'Green' }
        'DOWN'    { 'Red' }
        'STARTING'{ 'Yellow' }
        'OK'      { 'Green' }
        default   { 'Gray' }
    }
    $pad = $label.PadRight(14)
    Write-Host ("  {0} " -f $pad) -NoNewline
    Write-Host ("[{0}]" -f $state).PadRight(11) -ForegroundColor $color -NoNewline
    if ($detail) { Write-Host " $detail" -ForegroundColor Gray } else { Write-Host '' }
}

function Test-Http($url, $timeoutSec = 3) {
    try {
        $r = Invoke-WebRequest -Uri $url -UseBasicParsing -TimeoutSec $timeoutSec -ErrorAction Stop
        return ($r.StatusCode -ge 200 -and $r.StatusCode -lt 500)
    } catch {
        # A 4xx still means the server is alive and answering.
        if ($_.Exception.Response) { return $true }
        return $false
    }
}

function Get-Json($url, $timeoutSec = 3) {
    try { return Invoke-RestMethod -Uri $url -UseBasicParsing -TimeoutSec $timeoutSec -ErrorAction Stop }
    catch { return $null }
}

function Get-ListeningPid($port) {
    try {
        $c = Get-NetTCPConnection -LocalPort $port -State Listen -ErrorAction Stop | Select-Object -First 1
        return $c.OwningProcess
    } catch { return $null }
}

function Wait-Up($name, $url, $timeoutSec) {
    $sw = [Diagnostics.Stopwatch]::StartNew()
    while ($sw.Elapsed.TotalSeconds -lt $timeoutSec) {
        if (Test-Http $url 2) { return $true }
        Start-Sleep -Milliseconds 800
        Write-Host '.' -NoNewline -ForegroundColor DarkGray
    }
    return $false
}

# --------------------------------------------------------------------------
# Start
# --------------------------------------------------------------------------
function Start-HiddenSvc($exe, $argList, $cwd, $outLog, $errLog) {
    # Start-Process -WindowStyle Hidden is IGNORED when Windows Terminal is the
    # default terminal (it pops a window anyway). CreateNoWindow is honored, and
    # the invisible console is inherited by console children (uv venv pythons are
    # launcher shims that spawn the real interpreter as a child). cmd /c does the
    # log redirection so no stream pumping is needed.
    $quoted = ($argList | ForEach-Object { if ("$_" -match '\s') { '"' + $_ + '"' } else { "$_" } }) -join ' '
    $psi = New-Object System.Diagnostics.ProcessStartInfo
    $psi.FileName = "$env:SystemRoot\System32\cmd.exe"
    $psi.Arguments = '/d /c ""' + $exe + '" ' + $quoted + ' > "' + $outLog + '" 2> "' + $errLog + '""'
    if ($cwd) { $psi.WorkingDirectory = $cwd }
    $psi.UseShellExecute = $false
    $psi.CreateNoWindow  = $true
    [System.Diagnostics.Process]::Start($psi) | Out-Null
}

function Start-OllamaSvc {
    if (Test-Http $SERVICES.Ollama.Health 2) { Write-Status 'Ollama' 'UP' 'already running'; return }
    if (-not (Test-Path $OLLAMA_EXE)) { Write-Status 'Ollama' 'DOWN' "exe not found: $OLLAMA_EXE"; return }
    Write-Host '  Starting Ollama' -NoNewline -ForegroundColor Yellow
    Start-HiddenSvc $OLLAMA_EXE @('serve') $null `
        (Join-Path $LOGDIR 'ollama.out.log') (Join-Path $LOGDIR 'ollama.err.log')
    if (Wait-Up 'Ollama' $SERVICES.Ollama.Health 30) { Write-Host ''; Write-Status 'Ollama' 'UP' '' }
    else { Write-Host ''; Write-Status 'Ollama' 'DOWN' 'did not come up in 30s (see logs\ollama.err.log)'; $script:Failures++ }
}

function Start-ComfySvc {
    if (Test-Http $SERVICES.Comfy.Health 2) { Write-Status 'ComfyUI' 'UP' 'already running'; return }
    if (-not (Test-Path $COMFY_PY))  { Write-Status 'ComfyUI' 'DOWN' "venv python not found: $COMFY_PY"; return }
    if (-not (Test-Path $COMFY_DIR)) { Write-Status 'ComfyUI' 'DOWN' "ComfyUI not found: $COMFY_DIR"; return }
    Write-Host '  Starting ComfyUI (model load can take ~30s)' -NoNewline -ForegroundColor Yellow
    $env:PYTHONUTF8 = '1'   # emoji in startup logs crash cp1252 otherwise
    $args = @('main.py', '--listen', '127.0.0.1', '--port', '8188', '--base-directory', $COMFY_BASE)
    Start-HiddenSvc $COMFY_PY $args $COMFY_DIR `
        (Join-Path $LOGDIR 'comfyui.out.log') (Join-Path $LOGDIR 'comfyui.err.log')
    if (Wait-Up 'ComfyUI' $SERVICES.Comfy.Health 90) { Write-Host ''; Write-Status 'ComfyUI' 'UP' '' }
    else { Write-Host ''; Write-Status 'ComfyUI' 'DOWN' 'did not come up in 90s (see logs\comfyui.err.log)'; $script:Failures++ }
}

function Start-StudioSvc {
    if (Test-Http $SERVICES.Studio.Health 2) { Write-Status 'Studio' 'UP' 'already running'; return }
    if (-not $STUDIO_PY)              { Write-Status 'Studio' 'DOWN' 'python not on PATH'; return }
    if (-not (Test-Path $STUDIO_MAIN)){ Write-Status 'Studio' 'DOWN' "server.py not found: $STUDIO_MAIN"; return }
    Write-Host '  Starting Studio server' -NoNewline -ForegroundColor Yellow
    if ($Lan) { $env:STUDIO_HOST = '0.0.0.0' } else { $env:STUDIO_HOST = '127.0.0.1' }
    Start-HiddenSvc $STUDIO_PY @($STUDIO_MAIN) $HERE `
        (Join-Path $LOGDIR 'studio.out.log') (Join-Path $LOGDIR 'studio.err.log')
    if (Wait-Up 'Studio' $SERVICES.Studio.Health 20) {
        Write-Host ''
        $bind = if ($Lan) { '0.0.0.0 (LAN-visible)' } else { '127.0.0.1' }
        Write-Status 'Studio' 'UP' "http://127.0.0.1:8800  (bound $bind)"
    } else { Write-Host ''; Write-Status 'Studio' 'DOWN' 'did not come up in 20s (see logs\studio.err.log)'; $script:Failures++ }
}

function Start-RemoteServe {
    if (-not (Test-Path $TAILSCALE)) { Write-Status 'Tailscale' 'DOWN' 'tailscale.exe not found'; return }
    try {
        & $TAILSCALE serve --bg 8800 2>&1 | Out-Null
        Write-Status 'Tailscale' 'OK' 'Serve published 8800 -> https://<machine>.<tailnet>.ts.net'
    } catch {
        Write-Status 'Tailscale' 'DOWN' "serve failed: $($_.Exception.Message)"
    }
}

function Invoke-Start {
    Write-Host "`n=== Starting Local AI Studio stack ===" -ForegroundColor Cyan
    Start-OllamaSvc
    Start-ComfySvc
    Start-StudioSvc
    if ($Remote) { Start-RemoteServe }
    Write-Host ''
    Show-Status
}

# --------------------------------------------------------------------------
# Stop
# --------------------------------------------------------------------------
function Stop-ByPort($label, $port) {
    $procId = Get-ListeningPid $port
    if (-not $procId) { Write-Status $label 'DOWN' 'not running'; return }
    try {
        # also kill child processes (e.g. Studio's STT/TTS worker subprocesses)
        $children = Get-CimInstance Win32_Process -Filter "ParentProcessId=$procId" -ErrorAction SilentlyContinue
        Stop-Process -Id $procId -Force -ErrorAction Stop
        foreach ($ch in $children) {
            try { Stop-Process -Id $ch.ProcessId -Force -ErrorAction SilentlyContinue } catch {}
        }
        Write-Status $label 'OK' "stopped (pid $procId)"
    } catch {
        Write-Status $label 'DOWN' "could not stop pid ${procId}: $($_.Exception.Message)"
    }
}

function Invoke-Stop {
    Write-Host "`n=== Stopping Local AI Studio stack ===" -ForegroundColor Cyan
    # Ask the Studio to unload any resident worker first (frees VRAM, kills serve workers cleanly)
    if (Test-Http $SERVICES.Studio.Health 2) {
        try { Invoke-RestMethod -Uri 'http://127.0.0.1:8800/api/stop' -Method Post -Body '{}' `
                -ContentType 'application/json' -TimeoutSec 10 -UseBasicParsing | Out-Null } catch {}
    }
    Stop-ByPort 'Studio'  8800
    Stop-ByPort 'koboldcpp' 5001   # Story Maker fiction backend (frees ~13GB VRAM)
    Stop-ByPort 'ComfyUI' 8188
    if ($IncludeOllama) {
        Stop-ByPort 'Ollama' 11434
        # the tray app re-spawns serve; stop it too if present
        Get-Process 'ollama app' -ErrorAction SilentlyContinue | Stop-Process -Force -ErrorAction SilentlyContinue
    } else {
        Write-Status 'Ollama' 'OK' 'left running (use -IncludeOllama to stop)'
    }
}

# --------------------------------------------------------------------------
# Status / Monitor
# --------------------------------------------------------------------------
function Get-Gpu {
    $smi = Join-Path $env:WINDIR 'System32\nvidia-smi.exe'
    if (-not (Test-Path $smi)) { return $null }
    try {
        $line = & $smi --query-gpu=memory.used,memory.total,utilization.gpu,temperature.gpu `
                       --format=csv,noheader,nounits 2>$null | Select-Object -First 1
        if (-not $line) { return $null }
        $p = $line -split ',' | ForEach-Object { $_.Trim() }
        return [pscustomobject]@{ UsedMB = [int]$p[0]; TotalMB = [int]$p[1]; UtilPct = $p[2]; TempC = $p[3] }
    } catch { return $null }
}

function Show-Status {
    Write-Host "Local AI Studio - status  ($(Get-Date -Format 'HH:mm:ss'))" -ForegroundColor Cyan
    Write-Host ('-' * 64) -ForegroundColor DarkGray

    # Ollama
    if (Test-Http $SERVICES.Ollama.Health 2) {
        $tags = Get-Json $SERVICES.Ollama.Health 3
        $models = if ($tags -and $tags.models) {
            $ps = Get-Json 'http://127.0.0.1:11434/api/ps' 3
            if ($ps -and $ps.models) { "loaded in VRAM: " + (($ps.models | ForEach-Object { $_.name }) -join ', ') }
            else { "$($tags.models.Count) model(s) installed, none resident" }
        } else { 'running' }
        Write-Status 'Ollama' 'UP' $models
    } else { Write-Status 'Ollama' 'DOWN' ':11434 not answering' }

    # ComfyUI
    if (Test-Http $SERVICES.Comfy.Health 2) {
        $st = Get-Json $SERVICES.Comfy.Health 3
        $detail = 'running'
        if ($st -and $st.devices) {
            $d = $st.devices[0]
            $freeGB = [math]::Round($d.vram_free / 1GB, 1)
            $totGB  = [math]::Round($d.vram_total / 1GB, 1)
            $detail = "$($d.name)  VRAM free ${freeGB}/${totGB} GB"
        }
        Write-Status 'ComfyUI' 'UP' $detail
    } else { Write-Status 'ComfyUI' 'DOWN' ':8188 not answering' }

    # Studio
    if (Test-Http $SERVICES.Studio.Health 2) {
        $stat = Get-Json 'http://127.0.0.1:8800/api/status' 3
        $worker = if ($stat -and $stat.key) { "active worker: $($stat.key)" } else { 'no worker loaded (idle)' }
        Write-Status 'Studio' 'UP' "http://127.0.0.1:8800  |  $worker"
    } else { Write-Status 'Studio' 'DOWN' ':8800 not answering' }

    # koboldcpp (Story Maker fiction backend) - on-demand; Studio spawns/frees it.
    if (Test-Http 'http://127.0.0.1:5001/api/v1/model' 2) {
        Write-Status 'koboldcpp' 'UP' ':5001  Story model resident (Cydonia-24B)'
    } else { Write-Status 'koboldcpp' '-' ':5001 idle (loads on demand for Story Maker)' }

    # Tailscale Serve (informational)
    if (Test-Path $TAILSCALE) {
        try {
            $serve = (& $TAILSCALE serve status 2>$null | Out-String)
            if ($serve -match '8800') { Write-Status 'Tailscale' 'OK' 'Serve published (8800)' }
            else { Write-Status 'Tailscale' '-' 'Serve not publishing 8800 (start with -Remote)' }
        } catch { Write-Status 'Tailscale' '-' 'status unavailable' }
    }

    # GPU
    $gpu = Get-Gpu
    if ($gpu) {
        $freeMB = $gpu.TotalMB - $gpu.UsedMB
        Write-Host ('-' * 64) -ForegroundColor DarkGray
        Write-Host ("  GPU            {0} MB used / {1} MB total  ({2} MB free) | util {3}% | {4}C" -f `
            $gpu.UsedMB, $gpu.TotalMB, $freeMB, $gpu.UtilPct, $gpu.TempC) -ForegroundColor Gray
    }
    Write-Host ('-' * 64) -ForegroundColor DarkGray
}

function Invoke-Monitor {
    Write-Host 'Monitoring Local AI Studio (Ctrl+C to stop)...' -ForegroundColor Cyan
    while ($true) {
        Clear-Host
        Show-Status
        Write-Host ("  refreshing every ${Interval}s - Ctrl+C to exit") -ForegroundColor DarkGray
        Start-Sleep -Seconds $Interval
    }
}

# --------------------------------------------------------------------------
# Dispatch
# --------------------------------------------------------------------------
switch ($Command) {
    'start'   { Invoke-Start }
    'stop'    { Invoke-Stop;  Write-Host '' }
    'restart' { Invoke-Stop;  Start-Sleep -Seconds 2; Invoke-Start }
    'status'  { Show-Status }
    'monitor' { Invoke-Monitor }
}

# Deterministic exit code: 0 = all good, otherwise the number of services that
# failed to reach their target state (prevents stray native exit codes leaking).
exit $script:Failures
```

## File 9 of 27 — `%USERPROFILE%\local-ai-studio\studio.cmd`

```bat
@echo off
REM Local AI Studio control launcher.
REM Usage:  studio start | stop | restart | status | monitor   [extra -Flags]
REM   studio start            start the whole stack (Ollama + ComfyUI + Studio)
REM   studio start -Remote    also publish over Tailscale Serve (HTTPS)
REM   studio start -Lan       bind Studio to 0.0.0.0 (LAN-visible)
REM   studio stop             stop Studio + ComfyUI (Ollama left running)
REM   studio stop -IncludeOllama   also stop Ollama
REM   studio status           one-shot health report
REM   studio monitor          live refreshing report
powershell -NoProfile -ExecutionPolicy Bypass -File "%~dp0studioctl.ps1" %*
```

## File 10 of 27 — `%USERPROFILE%\local-ai-studio\Studio Control Panel.cmd`

```bat
@echo off
REM Launch the Local AI Studio visual control panel (studio_gui.pyw).
REM The default Python on PATH has no tkinter, so we pick the first Python that
REM does (ComfyUI venv / conda envs) and launch the app windowless via pythonw.
setlocal
set "GUI=%~dp0studio_gui.pyw"

for %%P in (
  "%USERPROFILE%\Documents\ComfyUI\.venv\Scripts\pythonw.exe"
  "%USERPROFILE%\.conda\envs\xtts\pythonw.exe"
  "%USERPROFILE%\.conda\envs\chatterbox\pythonw.exe"
  "%USERPROFILE%\.conda\envs\nemo-asr\pythonw.exe"
  "%USERPROFILE%\.conda\envs\kokoro\pythonw.exe"
) do (
  if exist %%P (
    %%P -c "import tkinter" >nul 2>nul
    if not errorlevel 1 (
      start "" %%P "%GUI%"
      goto :done
    )
  )
)

echo Could not find a Python with tkinter installed.
echo Tried the ComfyUI venv and the conda envs.
pause
:done
```

## File 11 of 27 — `%USERPROFILE%\local-ai-studio\Studio Control Panel.vbs`

```vbs
' Launch the Local AI Studio visual control panel (studio_gui.pyw) with NO console
' window at all — wscript runs this file windowless, and the tkinter probe below is
' executed hidden (window style 0). Double-click this instead of the .cmd launcher
' (which necessarily flashes a batch console). Same probe order as the .cmd.
Option Explicit
Dim sh, fso, home, gui, cands, p
Set sh  = CreateObject("WScript.Shell")
Set fso = CreateObject("Scripting.FileSystemObject")
home = sh.ExpandEnvironmentStrings("%USERPROFILE%")
gui  = fso.GetParentFolderName(WScript.ScriptFullName) & "\studio_gui.pyw"

cands = Array( _
  home & "\Documents\ComfyUI\.venv\Scripts\pythonw.exe", _
  home & "\comfyui-src\.venv\Scripts\pythonw.exe", _
  home & "\.conda\envs\xtts\pythonw.exe", _
  home & "\.conda\envs\chatterbox\pythonw.exe", _
  home & "\.conda\envs\nemo-asr\pythonw.exe", _
  home & "\.conda\envs\kokoro\pythonw.exe")

For Each p In cands
  If fso.FileExists(p) Then
    ' hidden tkinter probe; sh.Run returns the exit code when told to wait
    If sh.Run("""" & p & """ -c ""import tkinter""", 0, True) = 0 Then
      sh.Run """" & p & """ """ & gui & """", 0, False
      WScript.Quit 0
    End If
  End If
Next

MsgBox "Could not find a Python with tkinter installed." & vbCrLf & _
       "Tried the ComfyUI venvs and the conda envs.", 48, "Local AI Studio"
```

## File 12 of 27 — `%USERPROFILE%\Documents\ComfyUI\custom_nodes\ram_websocket_save.py`

```python
"""RAM-only image output for the Local AI Studio.

This build of ComfyUI does not ship the core ``SaveImageWebsocket`` node, so the
studio's image generate/edit pipeline (gen.py) had no way to receive a result
without ComfyUI first writing a PNG to its output dir. This node restores that
capability: it streams the finished PNG straight over ComfyUI's websocket and
writes nothing to disk. Drop-in compatible with the standard SaveImageWebsocket
protocol (8-byte preview header + PNG bytes), which gen.py decodes.
"""
import time

import numpy as np
from PIL import Image

from server import PromptServer, BinaryEventTypes


class SaveImageWebsocket:
    @classmethod
    def INPUT_TYPES(cls):
        return {"required": {"images": ("IMAGE",)}}

    RETURN_TYPES = ()
    FUNCTION = "save_images"
    OUTPUT_NODE = True
    CATEGORY = "api/image"
    DESCRIPTION = "Stream the image over the websocket (RAM-only; nothing is written to disk)."

    def save_images(self, images):
        server = PromptServer.instance
        for image in images:
            arr = 255.0 * image.cpu().numpy()
            img = Image.fromarray(np.clip(arr, 0, 255).astype(np.uint8))
            # max_size=None -> full-resolution PNG; routed as [evt=1][fmt=2(PNG)][bytes].
            server.send_sync(BinaryEventTypes.UNENCODED_PREVIEW_IMAGE,
                             ["PNG", img, None], server.client_id)
        return {"ui": {"images": []}}

    @classmethod
    def IS_CHANGED(cls, images):
        return time.time()


NODE_CLASS_MAPPINGS = {"SaveImageWebsocket": SaveImageWebsocket}
NODE_DISPLAY_NAME_MAPPINGS = {"SaveImageWebsocket": "Save Image (Websocket · RAM-only)"}
```

## File 13 of 27 — `%USERPROFILE%\Documents\ComfyUI\custom_nodes\ace15_studio_encode.py`

```python
"""AceStep15StudioEncode: TextEncodeAceStepAudio1.5 with optional musical metas.

The stock node forces a concrete bpm / keyscale / timesignature (strict combo
validation), but ACE-Step 1.5's tokenizer happily omits metas it isn't given —
the planning LM then infers them from the tags, which is the better default for
the studio. bpm=0 / empty keyscale / empty timesignature = let the model decide.
Used by local-music/scripts/musicgen.py over the HTTP API.
"""


class AceStep15StudioEncode:
    @classmethod
    def INPUT_TYPES(cls):
        return {"required": {
            "clip": ("CLIP",),
            "tags": ("STRING", {"multiline": True, "default": ""}),
            "lyrics": ("STRING", {"multiline": True, "default": ""}),
            "seed": ("INT", {"default": 0, "min": 0, "max": 0xffffffffffffffff}),
            "duration": ("FLOAT", {"default": 120.0, "min": 1.0, "max": 2000.0, "step": 0.1}),
            "language": ("STRING", {"default": "en"}),
            "generate_audio_codes": ("BOOLEAN", {"default": True}),
            "cfg_scale": ("FLOAT", {"default": 2.0, "min": 0.0, "max": 100.0, "step": 0.1}),
            "temperature": ("FLOAT", {"default": 0.85, "min": 0.0, "max": 2.0, "step": 0.01}),
            "top_p": ("FLOAT", {"default": 0.9, "min": 0.0, "max": 2000.0, "step": 0.01}),
            "top_k": ("INT", {"default": 0, "min": 0, "max": 100}),
            "min_p": ("FLOAT", {"default": 0.0, "min": 0.0, "max": 1.0, "step": 0.001}),
            "bpm": ("INT", {"default": 0, "min": 0, "max": 300}),
            "keyscale": ("STRING", {"default": ""}),
            "timesignature": ("STRING", {"default": ""}),
        }}

    RETURN_TYPES = ("CONDITIONING",)
    FUNCTION = "encode"
    CATEGORY = "conditioning"

    def encode(self, clip, tags, lyrics, seed, duration, language, generate_audio_codes,
               cfg_scale, temperature, top_p, top_k, min_p, bpm, keyscale, timesignature):
        kw = dict(lyrics=lyrics, duration=duration, language=(language.strip() or "en"),
                  seed=seed, generate_audio_codes=generate_audio_codes, cfg_scale=cfg_scale,
                  temperature=temperature, top_p=top_p, top_k=top_k, min_p=min_p)
        if bpm > 0:
            kw["bpm"] = int(bpm)
        if keyscale.strip():
            kw["keyscale"] = keyscale.strip()
        if timesignature.strip():
            kw["timesignature"] = int(timesignature.strip())
        tokens = clip.tokenize(tags, **kw)
        return (clip.encode_from_tokens_scheduled(tokens),)


NODE_CLASS_MAPPINGS = {"AceStep15StudioEncode": AceStep15StudioEncode}
NODE_DISPLAY_NAME_MAPPINGS = {"AceStep15StudioEncode": "TextEncode ACE-Step 1.5 (studio, auto metas)"}
```

## File 14 of 27 — `%USERPROFILE%\.claude\skills\local-llm\SKILL.md`

````markdown
---
name: local-llm
description: Hand off a text/code/research/image-understanding subtask to a LOCAL model on this PC (via Ollama) instead of answering it yourself — use to save API tokens on bulk, cheap, or specialized work (summarize many files, draft boilerplate, classify, OCR/describe an image). Models run on the user's RTX 4080 SUPER; nothing is billed to any API.
---

# local-llm — delegate to a local model (Ollama)

Use this to **hand work to a local model** running on this machine instead of doing it yourself. The
orchestrator (you) stays in charge of planning, verifying, and assembling; the local model does the
bulk/cheap/specialized turn. Everything runs offline on the user's GPU — zero API cost.

## When to delegate here
- **code** — generate boilerplate, a function, a regex, a quick refactor, a unit test.
- **research** — summarize/synthesize text, extract facts, draft a write-up from provided material.
- **vision** — describe / OCR / answer questions about an image (or video frames).
Reserve your own (Claude) turns for planning, hard reasoning, multi-file judgment, and final review.

## Backends (auto-selected by `--task`)
| `--task` | Ollama model | For |
|---|---|---|
| `code` (default) | `gpt-oss:20b` | code generation/editing |
| `research` | `gpt-oss:20b` | summarize / synthesize / reason over text |
| `vision` | `qwen3-vl:8b` | image / video-frame understanding, OCR |

Requires Ollama serving on `127.0.0.1:11434` (the desktop/tray app or `ollama serve`).

## Usage
```bash
PY=python   # or the system python
# text / code (prompt as arg or via stdin)
$PY ~/.claude/skills/local-llm/scripts/ask.py --task code "Write a Python function that ..."
cat big_file.txt | $PY ~/.claude/skills/local-llm/scripts/ask.py --task research "Summarize the key risks"

# image understanding (repeat --image for video frames)
$PY ~/.claude/skills/local-llm/scripts/ask.py --task vision --image shot.png "What enemy is shown? One line."

# options: --max-tokens N (default 1024), --model NAME (override), --system "..."
```
The script prints only the model's text reply to stdout (errors to stderr, non-zero exit on failure).

## Notes
- 16GB VRAM = one heavy model at a time; Ollama swaps automatically when `--task` changes the model.
- For *video*, sample frames (e.g. with ffmpeg) and pass several `--image` flags.
- `gpt-oss:20b` covers both code and research; add a `qwen3-coder` tag and point `--task code` at it
  if you want max coding quality later.
````

## File 15 of 27 — `%USERPROFILE%\.claude\skills\local-llm\scripts\ask.py`

```python
#!/usr/bin/env python3
"""local-llm: hand a subtask to a local model via Ollama's OpenAI-compatible API.

Prints only the model's text reply to stdout. Errors -> stderr, non-zero exit.
"""
from __future__ import annotations

import argparse
import base64
import json
import mimetypes
import os
import sys
import urllib.error
import urllib.request

# Force UTF-8 stdio so model output (em-dashes, non-breaking hyphens, etc.) prints
# on Windows' default cp1252 console without a UnicodeEncodeError.
for _s in (sys.stdout, sys.stderr):
    try:
        _s.reconfigure(encoding="utf-8", errors="replace")
    except Exception:
        pass

OLLAMA = os.environ.get("OLLAMA_URL", "http://127.0.0.1:11434").rstrip("/")

TASK_MODEL = {
    "code": "gpt-oss:20b",
    "research": "gpt-oss:20b",
    "vision": "qwen3-vl:8b",
}


def _image_block(path: str) -> dict:
    with open(path, "rb") as f:
        data = base64.b64encode(f.read()).decode("ascii")
    mime = mimetypes.guess_type(path)[0] or "image/png"
    return {"type": "image_url", "image_url": {"url": f"data:{mime};base64,{data}"}}


def main() -> int:
    ap = argparse.ArgumentParser(description="Delegate a subtask to a local Ollama model.")
    ap.add_argument("prompt", nargs="*", help="Prompt text (or pipe via stdin).")
    ap.add_argument("--task", choices=list(TASK_MODEL), default="code")
    ap.add_argument("--model", help="Override the model tag.")
    ap.add_argument("--system", help="Optional system prompt.")
    ap.add_argument("--image", action="append", default=[], help="Image path (repeat for video frames).")
    ap.add_argument("--max-tokens", type=int, default=1024)
    ap.add_argument("--temperature", type=float, default=0.2)
    ap.add_argument("--top-p", type=float, default=None, help="Nucleus sampling top_p.")
    ap.add_argument("--frequency-penalty", type=float, default=None,
                    help="Penalize tokens by frequency (curbs repetition loops).")
    ap.add_argument("--presence-penalty", type=float, default=None,
                    help="Penalize tokens already present (encourages new content).")
    ap.add_argument("--reasoning-effort", choices=["low", "medium", "high"], default="low",
                    help="Reasoning depth for gpt-oss reasoning models. 'low' keeps the hidden "
                         "reasoning short so it doesn't consume the whole token budget and leave an "
                         "empty answer.")
    args = ap.parse_args()

    prompt = " ".join(args.prompt).strip()
    if not sys.stdin.isatty():
        piped = sys.stdin.read().strip()
        prompt = (prompt + "\n\n" + piped).strip() if prompt else piped
    if not prompt and not args.image:
        print("error: no prompt (pass as args or via stdin)", file=sys.stderr)
        return 2

    model = args.model or TASK_MODEL[args.task]

    if args.image:
        content: list = [{"type": "text", "text": prompt or "Describe this."}]
        for p in args.image:
            if not os.path.isfile(p):
                print(f"error: image not found: {p}", file=sys.stderr)
                return 2
            content.append(_image_block(p))
        user_msg = {"role": "user", "content": content}
    else:
        user_msg = {"role": "user", "content": prompt}

    messages = []
    if args.system:
        messages.append({"role": "system", "content": args.system})
    messages.append(user_msg)

    payload = {
        "model": model,
        "messages": messages,
        "max_tokens": args.max_tokens,
        "temperature": args.temperature,
    }
    if args.top_p is not None:
        payload["top_p"] = args.top_p
    if args.frequency_penalty is not None:
        payload["frequency_penalty"] = args.frequency_penalty
    if args.presence_penalty is not None:
        payload["presence_penalty"] = args.presence_penalty
    # gpt-oss (incl. abliterated) are reasoning models: without a low reasoning effort
    # they can spend the entire token budget "thinking" and return an empty answer.
    if "gpt-oss" in model.lower():
        payload["reasoning_effort"] = args.reasoning_effort
    req = urllib.request.Request(
        f"{OLLAMA}/v1/chat/completions",
        data=json.dumps(payload).encode("utf-8"),
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    try:
        with urllib.request.urlopen(req, timeout=600) as resp:
            data = json.loads(resp.read().decode("utf-8"))
    except urllib.error.HTTPError as e:
        body = e.read().decode("utf-8", "replace")
        print(f"error: HTTP {e.code} from Ollama: {body[:500]}", file=sys.stderr)
        return 1
    except OSError as e:
        print(f"error: cannot reach Ollama at {OLLAMA} ({e}). Is it running?", file=sys.stderr)
        return 1

    try:
        msg = data["choices"][0]["message"]
    except (KeyError, IndexError):
        print(f"error: unexpected response: {json.dumps(data)[:500]}", file=sys.stderr)
        return 1
    text = (msg.get("content") or "").strip()
    if not text:
        # reasoning models sometimes return only the hidden reasoning channel; use it
        # rather than nothing so the caller never sees an empty reply.
        text = (msg.get("reasoning") or msg.get("reasoning_content") or "").strip()
    if not text:
        print(f"error: empty reply (finish={data['choices'][0].get('finish_reason')}); "
              f"try a higher --max-tokens", file=sys.stderr)
        return 1
    print(text)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

## File 16 of 27 — `%USERPROFILE%\.claude\skills\local-image\SKILL.md`

````markdown
---
name: local-image
description: Generate images / game sprites / concept art LOCALLY via FLUX.2 Klein 4B on the user's ComfyUI (Apache-2.0, commercial-safe, free/unlimited). Use whenever the user wants an image, sprite, icon, texture, or concept art made on this machine instead of a paid/cloud API. Drives the local ComfyUI HTTP API; can launch ComfyUI headless if it isn't running.
---

# local-image — local image gen (FLUX.2 Klein 4B via ComfyUI)

Generates images on the user's RTX 4080 via a local ComfyUI server. Free, unlimited, commercial-safe.

## Launching ComfyUI headless (if `:8188` isn't up)
The studio runs a **git checkout** of ComfyUI (`~/comfyui-src`, own `.venv`) — newer than the
Desktop app, needed for ACE-Step 1.5; the Desktop app's models/data dir is reused via
`--base-directory`. Start the server without the GUI:
```bash
# free VRAM first (16GB = one heavy model at a time)
"$LOCALAPPDATA/Programs/Ollama/ollama.exe" stop gpt-oss:20b 2>/dev/null
cd "$USERPROFILE/comfyui-src"
PYTHONUTF8=1 PYTHONIOENCODING=utf-8 \
  "$USERPROFILE/comfyui-src/.venv/Scripts/python.exe" main.py \
  --listen 127.0.0.1 --port 8188 --base-directory "$USERPROFILE/Documents/ComfyUI"   # run in background
```
`PYTHONUTF8=1` is required (its startup logs contain emoji that crash the cp1252 console otherwise).
Check up: `curl -s :8188/system_stats`.

## Generate
```bash
python ~/.claude/skills/local-image/scripts/gen.py --prompt "PROMPT" --out PATH.png \
  [--model base4b|miraclein9b] [--size 1:1|3:4|4:3|9:16|16:9] [--width N --height N] \
  [--steps 20] [--cfg 5] [--seed N]
```
Prints the saved path. **Always Read the output PNG to inspect it**, regenerate weak ones with a new `--seed`.

## Models (`--model`)
- **base4b** (DEFAULT) — FLUX.2 Klein 4B base, **Apache-2.0, commercial-safe** → use for game assets / anything shipped.
- **miraclein9b** — community **uncensored** Klein 9B finetune (artistic freedom, e.g. classical/Renaissance
  nude studies). **Personal/artistic use only — NOT commercial-safe.** Heavier (9B class); fits 16GB but slower.

Each Klein size needs its **own** qwen3 text encoder, auto-selected per `--model`
(4B→`qwen_3_4b.safetensors`, 9B→`qwen_3_8b_fp8mixed.safetensors`). A mismatch = hard tensor-shape error
(`mat1 and mat2 shapes cannot be multiplied`), not a soft fail. New model files added while ComfyUI is
already running are picked up on the next `/prompt` (folder list refreshes).

## Edit (remove / reframe / outpaint / recompose)
```bash
python ~/.claude/skills/local-image/scripts/gen.py --out PATH.png \
  --image SRC.png [--image SRC2.png ...]  --prompt "remove the person on the left; fill background" \
  [--model …] [--size 16:9]  # omit --size = keep input size (in-place); pass one = reframe/outpaint
```
Any `--image` switches to **edit mode** (FLUX.2 `ReferenceLatent` graph; up to ~4 reference images).
⚠️ **Edit needs LOW guidance** — defaults to `--cfg 1.5`. High cfg (e.g. 5) **blows out / regenerates from
scratch** instead of editing. Raise toward 2.5–4 only for big changes; keep ~1.5 for faithful removal/reframe.
Both models can edit; **miraclein9b** is the edit-tuned one (Miraclein = "Generation & Edit").

## Sprites with transparent background
Reuse the bundled `image-gen` skill's `cutout.py` (rembg) for alpha cutout:
`python ~/.claude/skills/image-gen/cutout.py PATH.png` (do this for character/object sprites).

## Notes
- Models live in `Documents\ComfyUI\models\`: `diffusion_models/{flux-2-klein-base-4b-fp8,miraclein-9b-fp8}.safetensors`,
  `text_encoders/{qwen_3_4b,qwen_3_8b_fp8mixed}.safetensors`, `vae/flux2-vae.safetensors`. Override via `FLUX2_UNET/CLIP/VAE` env.
- Prompt for game art: "full body … sprite of X, … , plain flat background, centered, generous margin."
- First generation loads ~11GB of weights; subsequent ones are fast. Free Ollama VRAM before generating.
````

## File 17 of 27 — `%USERPROFILE%\.claude\skills\local-image\scripts\gen.py`

```python
#!/usr/bin/env python3
"""local-image: generate an image with FLUX.2 Klein 4B via the local ComfyUI API.

Builds the FLUX.2 Klein text-to-image graph, submits it to ComfyUI (:8188), and
receives the output PNG in memory over ComfyUI's websocket (SaveImageWebsocket) so
nothing is written to ComfyUI's output dir. With --out PATH it writes that file;
with --out - it prints a data:image/png;base64 URL (RAM-only). Plain stdlib.
"""
from __future__ import annotations

import argparse
import base64
import json
import os
import socket
import struct
import sys
import time
import urllib.request
import uuid

for _s in (sys.stdout, sys.stderr):
    try:
        _s.reconfigure(encoding="utf-8", errors="replace")
    except Exception:
        pass

COMFY = os.environ.get("COMFYUI_URL", "http://127.0.0.1:8188").rstrip("/")
VAE = os.environ.get("FLUX2_VAE", "flux2-vae.safetensors")
# ComfyUI's input dir — LoadImage reads from here. We copy edit sources into it.
INPUT_DIR = os.environ.get("COMFYUI_INPUT") or os.path.join(
    os.path.expanduser("~"), "Documents", "ComfyUI", "input")

# Selectable diffusion models. Each Klein size needs its OWN qwen3 text encoder
# (4B->qwen3-4b, 9B->qwen3-8b); a mismatch is a tensor-shape error, not a soft fail.
# All share the flux2 VAE.
#   base4b      = official FLUX.2 Klein 4B base, Apache-2.0, commercial-safe (DEFAULT).
#   miraclein9b = community uncensored Klein 9B finetune; personal/artistic use only,
#                 NOT commercial-safe. Heavier (9B class) — fits 16GB but slower.
MODELS = {
    "base4b":      {"unet": "flux-2-klein-base-4b-fp8.safetensors", "clip": "qwen_3_4b.safetensors"},
    "miraclein9b": {"unet": "miraclein-9b-fp8.safetensors",          "clip": "qwen_3_8b_fp8mixed.safetensors"},
}
# FLUX2_UNET / FLUX2_CLIP env still win if set, for ad-hoc overrides.

SIZES = {  # FLUX.2-friendly resolutions
    "1:1": (1024, 1024), "3:4": (896, 1152), "4:3": (1152, 896),
    "9:16": (768, 1344), "16:9": (1344, 768),
}


def _post(path: str, payload: dict) -> dict:
    req = urllib.request.Request(f"{COMFY}{path}", data=json.dumps(payload).encode(),
                                 headers={"Content-Type": "application/json"}, method="POST")
    with urllib.request.urlopen(req, timeout=30) as r:
        return json.loads(r.read().decode())


def _get(path: str) -> bytes:
    with urllib.request.urlopen(f"{COMFY}{path}", timeout=30) as r:
        return r.read()


def build_graph(prompt: str, w: int, h: int, steps: int, cfg: float, seed: int, unet: str, clip: str) -> dict:
    return {
        "10": {"class_type": "UNETLoader", "inputs": {"unet_name": unet, "weight_dtype": "default"}},
        "11": {"class_type": "CLIPLoader", "inputs": {"clip_name": clip, "type": "flux2"}},
        "12": {"class_type": "VAELoader", "inputs": {"vae_name": VAE}},
        "13": {"class_type": "CLIPTextEncode", "inputs": {"text": prompt, "clip": ["11", 0]}},
        "14": {"class_type": "CLIPTextEncode", "inputs": {"text": "", "clip": ["11", 0]}},
        "15": {"class_type": "EmptyFlux2LatentImage", "inputs": {"width": w, "height": h, "batch_size": 1}},
        "16": {"class_type": "Flux2Scheduler", "inputs": {"steps": steps, "width": w, "height": h}},
        "17": {"class_type": "KSamplerSelect", "inputs": {"sampler_name": "euler"}},
        "18": {"class_type": "CFGGuider", "inputs": {"model": ["10", 0], "positive": ["13", 0],
                                                     "negative": ["14", 0], "cfg": cfg}},
        "19": {"class_type": "RandomNoise", "inputs": {"noise_seed": seed}},
        "20": {"class_type": "SamplerCustomAdvanced", "inputs": {"noise": ["19", 0], "guider": ["18", 0],
                "sampler": ["17", 0], "sigmas": ["16", 0], "latent_image": ["15", 0]}},
        "21": {"class_type": "VAEDecode", "inputs": {"samples": ["20", 0], "vae": ["12", 0]}},
        # SaveImageWebsocket streams the PNG back over the ws — nothing is written to ComfyUI's output dir.
        "22": {"class_type": "SaveImageWebsocket", "inputs": {"images": ["21", 0]}},
    }


def build_edit_graph(prompt: str, image_names: list[str], w, h, steps: int,
                     cfg: float, seed: int, unet: str, clip: str) -> dict:
    """FLUX.2 reference-editing graph: edit/remove/reframe using 1+ input images.

    Each input image is loaded, scaled to ~1MP, VAE-encoded, and chained through
    ReferenceLatent so the model conditions on it while following the edit prompt.
    If w/h are None the output matches the first image's size (in-place edit);
    pass an explicit size to reframe / outpaint to a new aspect.
    """
    g = {
        "10": {"class_type": "UNETLoader", "inputs": {"unet_name": unet, "weight_dtype": "default"}},
        "11": {"class_type": "CLIPLoader", "inputs": {"clip_name": clip, "type": "flux2"}},
        "12": {"class_type": "VAELoader", "inputs": {"vae_name": VAE}},
        "13": {"class_type": "CLIPTextEncode", "inputs": {"text": prompt, "clip": ["11", 0]}},
        "14": {"class_type": "ConditioningZeroOut", "inputs": {"conditioning": ["13", 0]}},
        "17": {"class_type": "KSamplerSelect", "inputs": {"sampler_name": "euler"}},
        "19": {"class_type": "RandomNoise", "inputs": {"noise_seed": seed}},
    }
    prev = ["13", 0]  # positive conditioning, extended once per reference image
    for i, name in enumerate(image_names):
        load, scale, enc, ref = f"{100+i}", f"{120+i}", f"{140+i}", f"{160+i}"
        g[load] = {"class_type": "LoadImage", "inputs": {"image": name}}
        g[scale] = {"class_type": "ImageScaleToTotalPixels",
                    "inputs": {"image": [load, 0], "upscale_method": "lanczos",
                               "megapixels": 1.0, "resolution_steps": 64}}
        g[enc] = {"class_type": "VAEEncode", "inputs": {"pixels": [scale, 0], "vae": ["12", 0]}}
        g[ref] = {"class_type": "ReferenceLatent", "inputs": {"conditioning": prev, "latent": [enc, 0]}}
        prev = [ref, 0]

    if w and h:  # explicit reframe / outpaint target
        ow, oh = w, h
    else:        # match the first (scaled) image -> in-place edit
        g["90"] = {"class_type": "GetImageSize", "inputs": {"image": ["120", 0]}}
        ow, oh = ["90", 0], ["90", 1]
    g["15"] = {"class_type": "EmptyFlux2LatentImage", "inputs": {"width": ow, "height": oh, "batch_size": 1}}
    g["16"] = {"class_type": "Flux2Scheduler", "inputs": {"steps": steps, "width": ow, "height": oh}}
    g["18"] = {"class_type": "CFGGuider", "inputs": {"model": ["10", 0], "positive": prev,
                                                     "negative": ["14", 0], "cfg": cfg}}
    g["20"] = {"class_type": "SamplerCustomAdvanced", "inputs": {"noise": ["19", 0], "guider": ["18", 0],
               "sampler": ["17", 0], "sigmas": ["16", 0], "latent_image": ["15", 0]}}
    g["21"] = {"class_type": "VAEDecode", "inputs": {"samples": ["20", 0], "vae": ["12", 0]}}
    g["22"] = {"class_type": "SaveImageWebsocket", "inputs": {"images": ["21", 0]}}
    return g


def stage_inputs(paths: list[str]) -> list[str]:
    """Copy edit source images into ComfyUI's input dir; return their filenames."""
    import shutil
    os.makedirs(INPUT_DIR, exist_ok=True)
    names = []
    for p in paths:
        ext = os.path.splitext(p)[1] or ".png"
        name = f"edit_{uuid.uuid4().hex}{ext}"
        shutil.copyfile(p, os.path.join(INPUT_DIR, name))
        names.append(name)
    return names


# ---- minimal stdlib WebSocket client (no websocket-client dependency) -------
# Used to receive the generated PNG from the SaveImageWebsocket node directly in
# memory, so the image is never written to ComfyUI's output dir or to disk here.
class _WS:
    def __init__(self, sock, rest=b""):
        self.s = sock
        self.buf = rest

    def _need(self, n):
        while len(self.buf) < n:
            ch = self.s.recv(65536)
            if not ch:
                raise RuntimeError("websocket closed")
            self.buf += ch

    def _frame(self):
        self._need(2)
        b0, b1 = self.buf[0], self.buf[1]
        fin = bool(b0 & 0x80); op = b0 & 0x0F
        masked = bool(b1 & 0x80); n = b1 & 0x7F; i = 2
        if n == 126:
            self._need(4); n = struct.unpack(">H", self.buf[2:4])[0]; i = 4
        elif n == 127:
            self._need(10); n = struct.unpack(">Q", self.buf[2:10])[0]; i = 10
        mask = b""
        if masked:
            self._need(i + 4); mask = self.buf[i:i + 4]; i += 4
        self._need(i + n)
        data = self.buf[i:i + n]; self.buf = self.buf[i + n:]
        if masked:
            data = bytes(c ^ mask[k % 4] for k, c in enumerate(data))
        return fin, op, data

    def send(self, op, data=b""):
        m = os.urandom(4); n = len(data)
        hdr = bytes([0x80 | op])
        if n < 126:
            hdr += bytes([0x80 | n])
        elif n < 65536:
            hdr += bytes([0x80 | 126]) + struct.pack(">H", n)
        else:
            hdr += bytes([0x80 | 127]) + struct.pack(">Q", n)
        hdr += m
        self.s.sendall(hdr + bytes(c ^ m[k % 4] for k, c in enumerate(data)))

    def message(self):
        op0 = None; data = b""
        while True:
            fin, op, payload = self._frame()
            if op == 0x8:          # close
                return ("close", b"")
            if op == 0x9:          # ping -> pong
                self.send(0xA, payload); continue
            if op == 0xA:          # pong
                continue
            if op != 0x0:
                op0 = op; data = payload
            else:
                data += payload
            if fin:
                return ("text" if op0 == 0x1 else "binary", data)


def _ws_open(client_id: str):
    from urllib.parse import urlparse
    u = urlparse(COMFY)
    host = u.hostname or "127.0.0.1"
    port = u.port or (443 if u.scheme == "https" else 80)
    s = socket.create_connection((host, port), timeout=60)
    key = base64.b64encode(os.urandom(16)).decode()
    req = ("GET /ws?clientId=%s HTTP/1.1\r\nHost: %s:%d\r\nUpgrade: websocket\r\n"
           "Connection: Upgrade\r\nSec-WebSocket-Key: %s\r\nSec-WebSocket-Version: 13\r\n\r\n"
           ) % (client_id, host, port, key)
    s.sendall(req.encode())
    buf = b""
    while b"\r\n\r\n" not in buf:
        ch = s.recv(4096)
        if not ch:
            raise RuntimeError("websocket handshake closed")
        buf += ch
    head, _, rest = buf.partition(b"\r\n\r\n")
    if b" 101 " not in head.split(b"\r\n", 1)[0]:
        raise RuntimeError("websocket upgrade failed: " + head.split(b"\r\n", 1)[0].decode("latin1"))
    return s, rest


def capture_ws(graph: dict, client_id: str, target: str = "22", timeout: int = 600) -> bytes:
    """Submit the graph and receive the output PNG bytes over the ws (in memory)."""
    sock, rest = _ws_open(client_id)
    ws = _WS(sock, rest)
    try:
        sock.settimeout(timeout)
        resp = _post("/prompt", {"prompt": graph, "client_id": client_id})
        pid = resp.get("prompt_id")
        if not pid:
            raise RuntimeError("no prompt_id in response: %s" % resp)
        img = None; cur = None
        deadline = time.time() + timeout
        while time.time() < deadline:
            kind, data = ws.message()
            if kind == "close":
                break
            if kind == "text":
                try:
                    msg = json.loads(data.decode("utf-8", "replace"))
                except Exception:
                    continue
                mtype = msg.get("type")
                if mtype == "executing":
                    d = msg.get("data", {})
                    if d.get("node") is None and d.get("prompt_id") == pid:
                        break  # this prompt finished
                    cur = d.get("node")
                elif mtype == "execution_error":
                    raise RuntimeError("ComfyUI execution error: %s" % json.dumps(msg.get("data", {}))[:400])
            else:  # binary frame: SaveImageWebsocket payload = 8-byte header + PNG bytes
                if cur == target:
                    img = data[8:]
        return img
    finally:
        try:
            sock.close()
        except Exception:
            pass


def main() -> int:
    ap = argparse.ArgumentParser(description="Generate or edit an image with FLUX.2 Klein via ComfyUI.")
    ap.add_argument("--prompt", required=True)
    ap.add_argument("--out", required=True,
                    help="Output PNG path, or '-' to print a data:image/png;base64 URL to stdout "
                         "(RAM-only: nothing is written to disk).")
    ap.add_argument("--model", choices=list(MODELS), default="base4b",
                    help="base4b=Apache-2.0 commercial-safe (default); miraclein9b=uncensored, personal art only.")
    ap.add_argument("--size", choices=list(SIZES), default=None,
                    help="Output aspect. Generate: defaults to 1:1. Edit: omit to keep the input size, "
                         "or pass one to reframe/outpaint to a new aspect.")
    ap.add_argument("--width", type=int)
    ap.add_argument("--height", type=int)
    ap.add_argument("--image", action="append", default=[],
                    help="Source image to EDIT (remove/reframe/recompose). Repeat for multi-reference "
                         "(up to ~4). Presence of --image switches to edit mode.")
    ap.add_argument("--steps", type=int, default=20)
    ap.add_argument("--cfg", type=float, default=None,
                    help="Guidance. Generate default 5.0. Edit default 1.5 (low = faithful to source; "
                         "raise toward 3-4 for bigger changes — too high blows out / regenerates).")
    ap.add_argument("--seed", type=int, default=-1)
    args = ap.parse_args()

    edit_mode = bool(args.image)
    cfg = args.cfg if args.cfg is not None else (1.5 if edit_mode else 5.0)
    # Resolve output dimensions. In edit mode, no size -> match the input (w=h=None).
    if args.width and args.height:
        w, h = args.width, args.height
    elif args.size:
        w, h = SIZES[args.size]
    elif edit_mode:
        w, h = None, None
    else:
        w, h = SIZES["1:1"]
    seed = args.seed if args.seed >= 0 else uuid.uuid4().int % (2**63)
    unet = os.environ.get("FLUX2_UNET") or MODELS[args.model]["unet"]
    clip = os.environ.get("FLUX2_CLIP") or MODELS[args.model]["clip"]

    # ComfyUI must be reachable
    try:
        _get("/system_stats")
    except Exception as e:
        print(f"error: ComfyUI not reachable at {COMFY} ({e}). Is it running?", file=sys.stderr)
        return 1

    if edit_mode:
        missing = [p for p in args.image if not os.path.isfile(p)]
        if missing:
            print(f"error: input image(s) not found: {missing}", file=sys.stderr)
            return 1
        names = stage_inputs(args.image)
        graph = build_edit_graph(args.prompt, names, w, h, args.steps, cfg, seed, unet, clip)
    else:
        graph = build_graph(args.prompt, w, h, args.steps, cfg, seed, unet, clip)
    client_id = uuid.uuid4().hex
    try:
        data = capture_ws(graph, client_id)   # PNG bytes received in memory over the ws
    except urllib.error.HTTPError as e:
        print(f"error: /prompt rejected: {e.read().decode()[:600]}", file=sys.stderr)
        return 1
    except Exception as e:
        print(f"error: {e}", file=sys.stderr)
        return 1
    if not data:
        print("error: no image returned over websocket", file=sys.stderr)
        return 1

    if args.out == "-":
        # RAM-only: hand the image straight to the caller as a data URL, no disk write.
        sys.stdout.write("data:image/png;base64," + base64.b64encode(data).decode())
        sys.stdout.flush()
    else:
        os.makedirs(os.path.dirname(os.path.abspath(args.out)), exist_ok=True)
        with open(args.out, "wb") as f:
            f.write(data)
        print(args.out)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

## File 18 of 27 — `%USERPROFILE%\.claude\skills\local-music\scripts\musicgen.py`

```python
#!/usr/bin/env python3
"""local-music: generate music with ACE-Step 1.5 XL (SFT) via the local ComfyUI API.

Builds the ACE-Step 1.5 text-to-audio graph (tags + lyrics + optional musical
metas), submits it to ComfyUI (:8188), waits for completion, fetches the audio
via /view, then deletes the file from ComfyUI's output dir (nothing lingers on
disk). With --out PATH it writes the file(s); with --out - it prints one JSON line:
  {"audios": ["data:audio/...;base64,..."], "seed": N}

ACE-Step 1.5 is a hybrid LM + DiT model: a planning LM first generates audio
codes from the tags/lyrics (this is most of the quality win — leave it on),
then the DiT synthesizes audio conditioned on them. bpm / keyscale /
timesignature default to "let the model decide" via the AceStep15StudioEncode
custom node (Documents/ComfyUI/custom_nodes/ace15_studio_encode.py).
Optional audio2audio remix: --audio SRC (+ --denoise < 1) conditions on an
input track instead of an empty latent (audio codes are disabled then, per the
model's guidance for audio references). Plain stdlib.
"""
from __future__ import annotations

import argparse
import base64
import json
import os
import random
import re
import shutil
import subprocess
import sys
import tempfile
import time
import urllib.parse
import urllib.request
import uuid

for _s in (sys.stdout, sys.stderr):
    try:
        _s.reconfigure(encoding="utf-8", errors="replace")
    except Exception:
        pass

COMFY = os.environ.get("COMFYUI_URL", "http://127.0.0.1:8188").rstrip("/")
HOME = os.path.expanduser("~")
# ComfyUI runs with --base-directory Documents/ComfyUI -> its input/output dirs.
INPUT_DIR = os.environ.get("COMFYUI_INPUT") or os.path.join(HOME, "Documents", "ComfyUI", "input")
OUTPUT_DIR = os.environ.get("COMFYUI_OUTPUT") or os.path.join(HOME, "Documents", "ComfyUI", "output")
# Two DiT variants ship. turbo (default): distilled, 8 steps, NO CFG -> smoother,
# artifact-free vocals, ~6x faster. sft: 50 steps + CFG 7 -> strongest prompt
# adherence but CFG can add vocal harshness. Overridable via env for custom files.
DITS = {"turbo": os.environ.get("ACESTEP_DIT_TURBO", "acestep_v1.5_xl_turbo_bf16.safetensors"),
        "sft": os.environ.get("ACESTEP_DIT_SFT", "acestep_v1.5_xl_sft_bf16.safetensors")}
MODEL_DEFAULTS = {"turbo": (8, 1.0), "sft": (50, 7.0)}      # (steps, cfg)
CLIP1 = os.environ.get("ACESTEP_CLIP1", "qwen_0.6b_ace15.safetensors")
CLIP2 = os.environ.get("ACESTEP_CLIP2", "qwen_4b_ace15.safetensors")
VAE = os.environ.get("ACESTEP_VAE", "ace_1.5_vae.safetensors")

MIME = {"mp3": "audio/mpeg", "flac": "audio/flac", "opus": "audio/ogg"}
SAVE_NODE = "40"  # id of the save node in the graph (looked up in /history outputs)
# No console windows for ffmpeg children (this script runs windowless under the studio).
NOWIN = {"creationflags": 0x08000000} if os.name == "nt" else {}


def _post(path: str, obj: dict) -> dict:
    req = urllib.request.Request(COMFY + path, data=json.dumps(obj).encode(),
                                 headers={"Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=120) as r:
        return json.loads(r.read().decode("utf-8"))


def _get_json(path: str) -> dict:
    with urllib.request.urlopen(COMFY + path, timeout=60) as r:
        return json.loads(r.read().decode("utf-8"))


def _get_bytes(path: str) -> bytes:
    with urllib.request.urlopen(COMFY + path, timeout=300) as r:
        return r.read()


def stage_audio(path: str) -> str:
    """Copy the remix source into ComfyUI's input dir; return its filename.
    Prefixed music_ so the studio can sweep it when the music worker stops."""
    import shutil
    os.makedirs(INPUT_DIR, exist_ok=True)
    ext = os.path.splitext(path)[1] or ".wav"
    name = f"music_{uuid.uuid4().hex}{ext}"
    shutil.copyfile(path, os.path.join(INPUT_DIR, name))
    return name


def build_graph(a, seed: int, lyrics: str) -> dict:
    """ACE-Step 1.5 graph: UNET/CLIP/VAE loaders -> AuraFlow shift -> studio
    encode (tags+lyrics+metas, LM audio codes) -> KSampler over an empty 1.5
    audio latent (or a VAE-encoded remix source) -> decode -> save."""
    g = {
        "1": {"class_type": "UNETLoader",
              "inputs": {"unet_name": DITS[a.model], "weight_dtype": "default"}},
        "2": {"class_type": "DualCLIPLoader",
              "inputs": {"clip_name1": CLIP1, "clip_name2": CLIP2, "type": "ace"}},
        "3": {"class_type": "VAELoader", "inputs": {"vae_name": VAE}},
        "4": {"class_type": "ModelSamplingAuraFlow",
              "inputs": {"model": ["1", 0], "shift": a.shift}},
        "5": {"class_type": "AceStep15StudioEncode",
              "inputs": {"clip": ["2", 0], "tags": a.tags, "lyrics": lyrics,
                         "seed": seed, "duration": a.seconds,
                         "language": a.language,
                         "generate_audio_codes": (not a.audio) and (not a.no_audio_codes),
                         "cfg_scale": a.lm_cfg, "temperature": a.lm_temperature,
                         "top_p": a.lm_top_p, "top_k": 0, "min_p": 0.0,
                         "bpm": a.bpm, "keyscale": a.keyscale,
                         "timesignature": a.timesignature}},
        "6": {"class_type": "ConditioningZeroOut", "inputs": {"conditioning": ["5", 0]}},
    }
    if a.audio:  # audio2audio: condition on an existing track
        g["8"] = {"class_type": "LoadAudio", "inputs": {"audio": stage_audio(a.audio)}}
        g["9"] = {"class_type": "VAEEncodeAudio", "inputs": {"audio": ["8", 0], "vae": ["3", 0]}}
        latent = ["9", 0]
    else:
        g["7"] = {"class_type": "EmptyAceStep1.5LatentAudio",
                  "inputs": {"seconds": a.seconds, "batch_size": a.batch}}
        latent = ["7", 0]
    g["10"] = {"class_type": "KSampler",
               "inputs": {"model": ["4", 0], "seed": seed, "steps": a.steps, "cfg": a.cfg,
                          "sampler_name": a.sampler, "scheduler": a.scheduler,
                          "positive": ["5", 0], "negative": ["6", 0],
                          "latent_image": latent, "denoise": a.denoise}}
    g["11"] = {"class_type": "VAEDecodeAudio", "inputs": {"samples": ["10", 0], "vae": ["3", 0]}}
    save = {"mp3": ("SaveAudioMP3", {"quality": a.quality or "V0"}),
            "opus": ("SaveAudioOpus", {"quality": a.quality or "128k"}),
            "flac": ("SaveAudio", {})}[a.format]
    g[SAVE_NODE] = {"class_type": save[0],
                    "inputs": {"audio": ["11", 0], "filename_prefix": "studio_music/m", **save[1]}}
    return g


def run(graph: dict, timeout: int) -> list[bytes]:
    """Submit the graph, poll /history until done, fetch + delete the audio files."""
    pid = _post("/prompt", {"prompt": graph, "client_id": uuid.uuid4().hex}).get("prompt_id")
    if not pid:
        raise RuntimeError("ComfyUI accepted no prompt (are the ACE-Step 1.5 models installed?)")
    deadline = time.time() + timeout
    entry = None
    while time.time() < deadline:
        hist = _get_json(f"/history/{pid}")
        if pid in hist:
            entry = hist[pid]
            break
        time.sleep(1.0)
    if entry is None:
        raise RuntimeError(f"timed out after {timeout}s waiting for ComfyUI")
    status = entry.get("status") or {}
    if status.get("status_str") == "error":
        msgs = [m[1].get("exception_message", "") for m in status.get("messages", [])
                if m and m[0] == "execution_error"]
        raise RuntimeError("ComfyUI execution error: " + ("; ".join(msgs) or "unknown")[:400])
    files = ((entry.get("outputs") or {}).get(SAVE_NODE) or {}).get("audio") or []
    if not files:
        raise RuntimeError("no audio in ComfyUI outputs")
    blobs = []
    for f in files:
        q = urllib.parse.urlencode({"filename": f["filename"], "subfolder": f.get("subfolder", ""),
                                    "type": f.get("type", "output")})
        blobs.append(_get_bytes("/view?" + q))
        path = os.path.join(OUTPUT_DIR, f.get("subfolder", ""), f["filename"])
        for _ in range(5):  # nothing lingers in ComfyUI's output dir; retry —
            try:            # a fresh file can be briefly locked (AV scan etc.)
                os.remove(path)
                break
            except FileNotFoundError:
                break
            except Exception:
                time.sleep(0.6)
    return blobs


def _ffmpeg_exe():
    p = shutil.which("ffmpeg")
    if p:
        return p
    import glob
    hits = glob.glob(os.path.join(os.environ.get("LOCALAPPDATA", ""), "Microsoft", "WinGet",
                                  "Packages", "Gyan.FFmpeg*", "ffmpeg-*", "bin", "ffmpeg.exe"))
    return hits[0] if hits else None


def analyze_volume(blob: bytes, fmt: str):
    """(mean_dB, peak_dB) of an audio blob via ffmpeg volumedetect; (None, None) if unmeasurable."""
    exe = _ffmpeg_exe()
    if not exe:
        return None, None
    with tempfile.NamedTemporaryFile(suffix="." + fmt, delete=False) as f:
        f.write(blob)
        path = f.name
    try:
        r = subprocess.run([exe, "-i", path, "-af", "volumedetect", "-f", "null", "-"],
                           capture_output=True, text=True, timeout=120, **NOWIN)
        mean = re.search(r"mean_volume:\s*(-?[\d.]+) dB", r.stderr or "")
        peak = re.search(r"max_volume:\s*(-?[\d.]+) dB", r.stderr or "")
        return (float(mean.group(1)) if mean else None,
                float(peak.group(1)) if peak else None)
    except Exception:
        return None, None
    finally:
        try:
            os.unlink(path)
        except Exception:
            pass


def limit_peaks(blob: bytes, fmt: str, quality) -> bytes:
    """Re-encode with a -1 dBFS peak limiter (the official Playground normalizes to
    -1 dB too). ACE's raw decode can crest at ~0 dBFS, which reads as harsh/crackly
    on loud vocals. Only called when a peak actually exceeds -1 dB."""
    exe = _ffmpeg_exe()
    if not exe:
        return blob
    enc = {"mp3": ["-c:a", "libmp3lame"] + (["-b:a", quality] if quality and quality.endswith("k")
                                            else ["-q:a", "0"]),
           "opus": ["-c:a", "libopus", "-b:a", (quality or "128k")],
           "flac": ["-c:a", "flac"]}[fmt]
    src = dst = None
    try:
        with tempfile.NamedTemporaryFile(suffix="." + fmt, delete=False) as f:
            f.write(blob)
            src = f.name
        dst = src + ".lim." + fmt
        r = subprocess.run([exe, "-y", "-i", src, "-af", "alimiter=limit=0.891:level=false"]
                           + enc + [dst], capture_output=True, text=True, timeout=300, **NOWIN)
        if r.returncode == 0 and os.path.getsize(dst) > 0:
            return open(dst, "rb").read()
        return blob
    except Exception:
        return blob
    finally:
        for p in (src, dst):
            try:
                if p:
                    os.unlink(p)
            except Exception:
                pass


def main() -> int:
    ap = argparse.ArgumentParser(description="Generate music with ACE-Step 1.5 via ComfyUI.")
    ap.add_argument("--tags", required=True,
                    help="style description: genre, instruments, mood, tempo, vocal style")
    ap.add_argument("--lyrics", default="", help="lyrics with [verse]/[chorus] markers; empty = instrumental")
    ap.add_argument("--lyrics-file", default=None, help="read lyrics from a file (beats cmdline limits)")
    ap.add_argument("--model", choices=["turbo", "sft"], default="turbo",
                    help="turbo: distilled, 8 steps, no CFG — cleaner vocals, ~6x faster (default). "
                         "sft: 50 steps + CFG — strongest prompt adherence")
    ap.add_argument("--seconds", type=float, default=120.0)
    ap.add_argument("--steps", type=int, default=None, help="default: 8 (turbo) / 50 (sft)")
    ap.add_argument("--cfg", type=float, default=None, help="default: 1 (turbo) / 7 (sft)")
    ap.add_argument("--seed", type=int, default=-1)
    ap.add_argument("--shift", type=float, default=3.0, help="ModelSamplingAuraFlow shift")
    ap.add_argument("--sampler", default="euler")
    ap.add_argument("--scheduler", default="simple")
    ap.add_argument("--bpm", type=int, default=0, help="beats per minute; 0 = let the model decide")
    ap.add_argument("--keyscale", default="", help='e.g. "E minor"; empty = let the model decide')
    ap.add_argument("--timesignature", default="", help="2/3/4/6; empty = let the model decide")
    ap.add_argument("--language", default="en", help="lyrics language code")
    ap.add_argument("--no-audio-codes", action="store_true",
                    help="skip the planning-LM audio codes (faster, lower quality)")
    ap.add_argument("--lm-cfg", type=float, default=2.0, help="planning-LM cfg scale")
    ap.add_argument("--lm-temperature", type=float, default=0.85)
    ap.add_argument("--lm-top-p", type=float, default=0.9)
    ap.add_argument("--audio", default=None, help="source track for audio2audio remix")
    ap.add_argument("--denoise", type=float, default=1.0, help="lower with --audio to keep its structure")
    ap.add_argument("--batch", type=int, default=1, help="variations per run (1-4 sensible)")
    ap.add_argument("--format", default="mp3", choices=["mp3", "flac", "opus"])
    ap.add_argument("--quality", default=None, help="mp3: V0|128k|320k · opus: 64k..320k")
    ap.add_argument("--timeout", type=int, default=1800)
    ap.add_argument("--out", required=True, help="output path, or - for a JSON line with data URLs")
    a = ap.parse_args()

    lyrics = a.lyrics
    if a.lyrics_file:
        with open(a.lyrics_file, encoding="utf-8") as f:
            lyrics = f.read()
    dsteps, dcfg = MODEL_DEFAULTS[a.model]
    if a.steps is None:
        a.steps = dsteps
    if a.cfg is None:
        a.cfg = dcfg
    seed = a.seed if a.seed >= 0 else random.randrange(2 ** 48)
    blobs = run(build_graph(a, seed, lyrics), a.timeout)
    # ACE-Step 1.5's planning LM collapses on rare seeds -> near-silent garbage
    # (deterministic per seed). Detect it and reroll, unless the seed was pinned.
    retried = 0
    while a.seed < 0 and retried < 2:
        v, _ = analyze_volume(blobs[0], a.format)
        if v is None or v > -35.0:
            break
        retried += 1
        seed = random.randrange(2 ** 48)
        print(f"degenerate output ({v:.0f} dB mean) — retrying with seed {seed}", file=sys.stderr)
        blobs = run(build_graph(a, seed, lyrics), a.timeout)
    # Peak protection: raw decodes can crest at ~0 dBFS (clipping harshness on loud
    # vocals). Limit to -1 dBFS like the official Playground, only when needed.
    for i, b in enumerate(blobs):
        _, peak = analyze_volume(b, a.format)
        if peak is not None and peak > -1.0:
            blobs[i] = limit_peaks(b, a.format, a.quality)

    if a.out == "-":
        urls = ["data:%s;base64,%s" % (MIME[a.format], base64.b64encode(b).decode()) for b in blobs]
        print(json.dumps({"audios": urls, "seed": seed, "retried": retried}))
    else:
        base, ext = os.path.splitext(a.out)
        ext = ext or "." + a.format
        for i, b in enumerate(blobs):
            path = a.out if len(blobs) == 1 else f"{base}_{i + 1}{ext}"
            with open(path, "wb") as f:
                f.write(b)
            print(path)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## File 19 of 27 — `%USERPROFILE%\.claude\skills\local-stt\SKILL.md`

````markdown
---
name: local-stt
description: Transcribe an audio file to text using a LOCAL speech-to-text model on this PC (NVIDIA Parakeet-TDT via NeMo). Use when the user gives an audio/voice file (wav/mp3/m4a) and wants the words out, or to caption a clip — runs on the user's GPU, no API cost.
---

# local-stt — local speech-to-text (Parakeet-TDT / NeMo)

Hand an audio file to the local STT worker and get the transcript back. Runs offline on the GPU.

## Usage
```bash
conda run -n nemo-asr python ~/.claude/skills/local-stt/scripts/transcribe.py path/to/audio.wav
# options: --model nvidia/parakeet-tdt-0.6b-v2 (default) | --timestamps
```
Prints the transcript to stdout. Accepts wav/mp3/m4a/flac; non-16kHz-mono inputs are auto-converted
with ffmpeg if available (else convert first: `ffmpeg -i in.mp3 -ar 16000 -ac 1 out.wav`).

## When to use
- The user provides a recording / voice memo / clip and wants the text, a summary, or captions.
- After transcribing, you can hand the text to `local-llm --task research` to summarize.

## Notes
- ⚠️ **Free VRAM first.** On a 16GB GPU, if Ollama has an LLM loaded (gpt-oss ~14GB), Parakeet can't fit
  and returns an **empty transcript with no error**. Before transcribing, run
  `"$LOCALAPPDATA\Programs\Ollama\ollama.exe" stop gpt-oss:20b` (+ any other loaded tag). An empty
  result almost always means VRAM contention, not bad audio.
- First call downloads the Parakeet weights (~0.6–1GB) and loads the model (a few seconds).
- Requires the `nemo-asr` conda env (NVIDIA NeMo + CUDA torch). The transcript is the LAST stdout line.
````

## File 20 of 27 — `%USERPROFILE%\.claude\skills\local-stt\scripts\transcribe.py`

```python
#!/usr/bin/env python3
"""local-stt: transcribe an audio file with NVIDIA Parakeet-TDT (NeMo).

Prints the transcript to stdout. Run inside the `nemo-asr` conda env.
"""
from __future__ import annotations

import argparse
import os
import shutil
import subprocess
import sys
import tempfile

for _s in (sys.stdout, sys.stderr):
    try:
        _s.reconfigure(encoding="utf-8", errors="replace")
    except Exception:
        pass


def _to_wav16k_mono(src: str) -> tuple[str, bool]:
    """Return (path, is_temp). Convert to 16kHz mono wav via ffmpeg if needed/available."""
    if src.lower().endswith(".wav"):
        return src, False
    ffmpeg = shutil.which("ffmpeg")
    if not ffmpeg:
        print("warning: non-wav input and ffmpeg not found; trying NeMo directly", file=sys.stderr)
        return src, False
    fd, out = tempfile.mkstemp(suffix=".wav")
    os.close(fd)
    subprocess.run([ffmpeg, "-y", "-i", src, "-ar", "16000", "-ac", "1", out],
                   check=True, capture_output=True)
    return out, True


def serve(model_name: str) -> int:
    """Persistent mode: load the model once, then handle newline-delimited JSON
    requests on stdin ({"in": path}) and reply on stdout ({"ok":true,"text":...}).
    Library log noise is redirected to stderr so stdout carries only the protocol."""
    import json
    real = os.dup(1)            # keep the real stdout (the protocol channel)
    os.dup2(2, 1)              # send any library prints/logs to stderr instead
    proto = os.fdopen(real, "w", buffering=1, encoding="utf-8", errors="replace")

    def reply(obj):
        proto.write(json.dumps(obj) + "\n"); proto.flush()

    try:
        import nemo.collections.asr as nemo_asr
        model = nemo_asr.models.ASRModel.from_pretrained(model_name=model_name)
    except Exception as e:
        reply({"error": f"load failed: {e}"}); return 1
    reply({"ready": True})

    for line in sys.stdin:
        line = line.strip()
        if not line:
            continue
        try:
            req = json.loads(line)
            wav, is_temp = _to_wav16k_mono(req["in"])
            try:
                out = model.transcribe([wav])
            finally:
                if is_temp:
                    try: os.remove(wav)
                    except OSError: pass
            item = out[0] if out else ""
            reply({"ok": True, "text": getattr(item, "text", item)})
        except Exception as e:
            reply({"error": str(e)})
    return 0


def main() -> int:
    ap = argparse.ArgumentParser(description="Local speech-to-text via Parakeet-TDT (NeMo).")
    ap.add_argument("audio", nargs="?", help="Audio file (wav/mp3/m4a/flac).")
    ap.add_argument("--model", default="nvidia/parakeet-tdt-0.6b-v2")
    ap.add_argument("--timestamps", action="store_true", help="Also print word timestamps.")
    ap.add_argument("--serve", action="store_true", help="Persistent stdin/stdout worker mode.")
    args = ap.parse_args()

    if args.serve:
        return serve(args.model)
    if not args.audio:
        print("error: audio file required (or use --serve)", file=sys.stderr)
        return 2

    if not os.path.isfile(args.audio):
        print(f"error: file not found: {args.audio}", file=sys.stderr)
        return 2

    try:
        import nemo.collections.asr as nemo_asr
    except Exception as e:
        print(f"error: NeMo not importable ({e}). Is the nemo-asr env set up?", file=sys.stderr)
        return 1

    wav, is_temp = _to_wav16k_mono(args.audio)
    try:
        model = nemo_asr.models.ASRModel.from_pretrained(model_name=args.model)
        out = model.transcribe([wav], timestamps=args.timestamps) if args.timestamps \
            else model.transcribe([wav])
    except Exception as e:
        print(f"error: transcription failed: {e}", file=sys.stderr)
        return 1
    finally:
        if is_temp:
            try:
                os.remove(wav)
            except OSError:
                pass

    # NeMo returns either a list[str] or a list[Hypothesis] depending on version.
    item = out[0] if out else ""
    text = getattr(item, "text", item)
    print(text)
    if args.timestamps and hasattr(item, "timestamp"):
        for w in (item.timestamp or {}).get("word", []):
            print(f"  [{w.get('start'):.2f}-{w.get('end'):.2f}] {w.get('word')}", file=sys.stderr)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

## File 21 of 27 — `%USERPROFILE%\.claude\skills\local-tts\SKILL.md`

````markdown
---
name: local-tts
description: Turn text into spoken audio (wav) using a LOCAL text-to-speech model on this PC. Kokoro for fast clean narration, Chatterbox for highest-quality / voice-cloning. Use when the user wants a line, script, or narration voiced, or a character voice for a game — runs on the user's GPU, no API cost.
---

# local-tts — local text-to-speech (Kokoro + Chatterbox)

Synthesize speech from text locally. Two engines:
- **kokoro** (default) — fast, clean, 54 voices; great for narration/UI/quick lines.
- **chatterbox** — most natural; supports **voice cloning** from a short reference clip.

## Usage
```bash
# Kokoro (fast default) — conda env `kokoro`
conda run -n kokoro python ~/.claude/skills/local-tts/scripts/speak.py "Welcome, survivor." --out line.wav
conda run -n kokoro python ~/.claude/skills/local-tts/scripts/speak.py "..." --voice af_heart --out vo.wav

# Chatterbox (quality / cloning) — conda env `chatterbox`
conda run -n chatterbox python ~/.claude/skills/local-tts/scripts/speak.py "..." --engine chatterbox --out vo.wav
conda run -n chatterbox python ~/.claude/skills/local-tts/scripts/speak.py "..." --engine chatterbox --ref voice_sample.wav --out cloned.wav
```
Writes a wav to `--out` (default `tts_out.wav`) and prints its path to stdout.

## When to use
- Voice a generated line of dialogue, narration, or a character barks for the game.
- Pair with `local-llm` (write the line) then `local-tts` (speak it).

## Notes
- **Kokoro is the reliable default** (verified clean). Use it for general narration/lines.
- **Chatterbox is a voice-CLONING model** — always pass `--ref <clean 5–15s wav>`. Its reference-free
  default voice is unstable and can produce garbled output, so only use Chatterbox WITH a `--ref` clip.
- Kokoro output is 24kHz; Chatterbox uses its own sample rate (saved correctly in the wav).
- Engines live in separate conda envs (`kokoro`, `chatterbox`); the skill picks the env per `--engine`.
- chatterbox env quirk fixed during setup: pinned `setuptools==75.8.0` (newer setuptools drops
  `pkg_resources`, which the Perth watermarker needs) and installed the cu124 torch build for GPU.
````

## File 22 of 27 — `%USERPROFILE%\.claude\skills\local-tts\scripts\speak.py`

```python
#!/usr/bin/env python3
"""local-tts: synthesize speech to a wav file with Kokoro (default) or Chatterbox.

Run inside the matching conda env (`kokoro` or `chatterbox`). Prints the output path.
"""
from __future__ import annotations

import argparse
import sys

for _s in (sys.stdout, sys.stderr):
    try:
        _s.reconfigure(encoding="utf-8", errors="replace")
    except Exception:
        pass


def _kokoro(text: str, out: str, voice: str) -> str:
    import soundfile as sf
    import numpy as np
    from kokoro import KPipeline

    pipeline = KPipeline(lang_code="a")  # 'a' = American English
    chunks = [audio for _gs, _ps, audio in pipeline(text, voice=voice or "af_heart")]
    if not chunks:
        raise RuntimeError("Kokoro produced no audio")
    audio = np.concatenate(chunks) if len(chunks) > 1 else chunks[0]
    sf.write(out, audio, 24000)
    return out


def _chatterbox(text: str, out: str, ref: str | None) -> str:
    import torch
    import torchaudio as ta
    from chatterbox.tts import ChatterboxTTS

    device = "cuda" if torch.cuda.is_available() else "cpu"
    model = ChatterboxTTS.from_pretrained(device=device)
    wav = model.generate(text, audio_prompt_path=ref) if ref else model.generate(text)
    ta.save(out, wav, model.sr)
    return out


def serve(engine: str) -> int:
    """Persistent mode: load the chosen engine once, then handle newline-delimited
    JSON requests on stdin ({"text","out","voice"|"ref"}) and reply on stdout
    ({"ok":true,"out":path}). Library log noise is redirected to stderr."""
    import os
    import json
    real = os.dup(1)
    os.dup2(2, 1)
    proto = os.fdopen(real, "w", buffering=1, encoding="utf-8", errors="replace")

    def reply(obj):
        proto.write(json.dumps(obj) + "\n"); proto.flush()

    try:
        import numpy as _np

        def _shape(audio, sr, pad_ms):
            """Audiobook naturalness: 5 ms edge fades (butt-joined chunks click
            otherwise) + the requested trailing silence (sentence/paragraph pause)."""
            audio = _np.asarray(audio, dtype=_np.float32)
            f = min(int(sr * 0.005), len(audio) // 2)
            if f > 0:
                audio[:f] *= _np.linspace(0.0, 1.0, f, dtype=_np.float32)
                audio[-f:] *= _np.linspace(1.0, 0.0, f, dtype=_np.float32)
            pad = int(sr * max(0, int(pad_ms or 0)) / 1000)
            if pad:
                audio = _np.concatenate([audio, _np.zeros(pad, dtype=_np.float32)])
            return audio

        if engine == "kokoro":
            import soundfile as sf
            import numpy as np
            from kokoro import KPipeline
            # One pipeline per language, chosen from the voice-id prefix (a=US, b=British,
            # e=Spanish, f=French, h=Hindi, i=Italian, j=Japanese, p=BR-Portuguese, z=Mandarin),
            # so all 54 voices work — not just US English.
            pipes = {}

            def _pipe(voice):
                lc = (voice or "af_heart")[0]
                if lc not in pipes:
                    pipes[lc] = KPipeline(lang_code=lc)
                return pipes[lc]

            def synth(req):
                text = (req.get("text") or "").strip()
                voice = req.get("voice") or "af_heart"
                speed = float(req.get("speed") or 1.0)          # 0.5–2.0
                chunks = [a for _g, _p, a in _pipe(voice)(text, voice=voice, speed=speed)]
                if not chunks:
                    raise RuntimeError("Kokoro produced no audio")
                gap = np.zeros(int(24000 * 0.25), dtype=np.float32)  # breath between segments
                joined = chunks[0]
                for c in chunks[1:]:
                    joined = np.concatenate([joined, gap, c])
                sf.write(req["out"], _shape(joined, 24000, req.get("pad_ms")), 24000)
                return req["out"]
        else:
            import torch
            import torchaudio as ta
            from chatterbox.tts import ChatterboxTTS
            model = ChatterboxTTS.from_pretrained(device="cuda" if torch.cuda.is_available() else "cpu")

            def synth(req):
                text = (req.get("text") or "").strip()
                kw = {}
                if req.get("exaggeration") is not None:
                    kw["exaggeration"] = float(req["exaggeration"])   # 0–1, emotion intensity
                if req.get("cfg") is not None:
                    kw["cfg_weight"] = float(req["cfg"])              # lower = slower/steadier
                ref = req.get("ref")
                wav = model.generate(text, audio_prompt_path=ref, **kw) if ref else model.generate(text, **kw)
                shaped = _shape(wav.squeeze(0).cpu().numpy(), model.sr, req.get("pad_ms"))
                ta.save(req["out"], torch.from_numpy(shaped).unsqueeze(0), model.sr)
                return req["out"]
    except Exception as e:
        reply({"error": f"load failed: {e}"}); return 1
    reply({"ready": True})

    for line in sys.stdin:
        line = line.strip()
        if not line:
            continue
        try:
            req = json.loads(line)
            if not (req.get("text") or "").strip():
                raise RuntimeError("no text")
            path = synth(req)
            reply({"ok": True, "out": path})
        except Exception as e:
            reply({"error": str(e)})
    return 0


def main() -> int:
    ap = argparse.ArgumentParser(description="Local text-to-speech (Kokoro / Chatterbox).")
    ap.add_argument("text", nargs="*", help="Text to speak (or pipe via stdin).")
    ap.add_argument("--engine", choices=["kokoro", "chatterbox"], default="kokoro")
    ap.add_argument("--voice", default="af_heart", help="Kokoro voice id.")
    ap.add_argument("--ref", help="Chatterbox: reference wav to clone (5-15s).")
    ap.add_argument("--out", default="tts_out.wav")
    ap.add_argument("--serve", action="store_true", help="Persistent stdin/stdout worker mode.")
    args = ap.parse_args()

    if args.serve:
        return serve(args.engine)

    text = " ".join(args.text).strip()
    if not sys.stdin.isatty():
        piped = sys.stdin.read().strip()
        text = (text + " " + piped).strip() if text else piped
    if not text:
        print("error: no text (pass as args or via stdin)", file=sys.stderr)
        return 2

    try:
        if args.engine == "kokoro":
            path = _kokoro(text, args.out, args.voice)
        else:
            path = _chatterbox(text, args.out, args.ref)
    except Exception as e:
        print(f"error: {args.engine} synthesis failed: {e}", file=sys.stderr)
        return 1

    print(path)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

## File 23 of 27 — `%USERPROFILE%\.claude\skills\local-tts\scripts\voicechange.py`

```python
#!/usr/bin/env python3
"""local voice changer/creator — Chatterbox TTS (text->voice clone) + VC (voice->voice).

Run inside the `chatterbox` conda env. Persistent worker only (--serve): loads both
ChatterboxTTS and ChatterboxVC once, then handles newline-delimited JSON requests on
stdin and replies on stdout. Library log noise is redirected to stderr.

Request shapes:
  {"mode":"text",  "text":..., "target": WAV, "out": WAV}   -> speak `text` in the target voice
  {"mode":"voice", "source": WAV, "target": WAV, "out": WAV} -> convert `source` to the target voice
Reply: {"ok":true, "out": path}  |  {"error": "..."}
"""
from __future__ import annotations

import argparse
import json
import os
import sys


def serve() -> int:
    real = os.dup(1)
    os.dup2(2, 1)  # send library prints/logs to stderr; stdout = protocol only
    proto = os.fdopen(real, "w", buffering=1, encoding="utf-8", errors="replace")

    def reply(obj):
        proto.write(json.dumps(obj) + "\n"); proto.flush()

    try:
        import torch
        import torchaudio as ta
        from chatterbox.tts import ChatterboxTTS
        from chatterbox.vc import ChatterboxVC
        device = "cuda" if torch.cuda.is_available() else "cpu"
        tts = ChatterboxTTS.from_pretrained(device=device)
        vc = ChatterboxVC.from_pretrained(device=device)
    except Exception as e:
        reply({"error": f"load failed: {e}"}); return 1
    reply({"ready": True})

    for line in sys.stdin:
        line = line.strip()
        if not line:
            continue
        try:
            req = json.loads(line)
            mode = req.get("mode", "text")
            out = req["out"]
            target = req.get("target")
            if mode == "voice":
                src = req.get("source")
                if not src or not os.path.isfile(src):
                    raise RuntimeError("no source audio")
                if not target or not os.path.isfile(target):
                    raise RuntimeError("no target voice")
                wav = vc.generate(src, target_voice_path=target)
                ta.save(out, wav, vc.sr)
            else:  # text -> voice
                text = (req.get("text") or "").strip()
                if not text:
                    raise RuntimeError("no text")
                method = req.get("method", "tts")
                have_target = bool(target and os.path.isfile(target))
                if method == "tts_vc" and have_target:
                    # Stronger likeness: synthesize neutral speech, then voice-convert to the
                    # target. VC matches speaker identity far better than zero-shot TTS cloning.
                    neutral = out + ".neutral.wav"
                    ta.save(neutral, tts.generate(text), tts.sr)
                    wav = vc.generate(neutral, target_voice_path=target)
                    ta.save(out, wav, vc.sr)
                    try:
                        os.remove(neutral)
                    except OSError:
                        pass
                else:
                    wav = tts.generate(text, audio_prompt_path=target) if have_target else tts.generate(text)
                    ta.save(out, wav, tts.sr)
            reply({"ok": True, "out": out})
        except Exception as e:
            reply({"error": str(e)})
    return 0


def main() -> int:
    ap = argparse.ArgumentParser(description="Local voice changer (Chatterbox TTS + VC).")
    ap.add_argument("--serve", action="store_true", help="Persistent stdin/stdout worker mode.")
    ap.parse_args()
    return serve()


if __name__ == "__main__":
    raise SystemExit(main())
```

## File 24 of 27 — `%USERPROFILE%\.claude\skills\local-voice\scripts\xtts_train.py`

```python
#!/usr/bin/env python3
"""Fine-tune XTTS-v2 on a personal voice and save a reusable model.

Runs in the `xtts` conda env. Input is a dataset dir built from the user's
recordings of KNOWN scripts (so transcripts are exact, no ASR needed):

    dataset/
      wavs/clip_000.wav ...
      metadata_train.csv   # "audio_file|text|speaker_name" (pipe, with header)
      metadata_eval.csv

Trains the XTTS GPT on top of the local base checkpoint, then saves a compact,
ready-to-use model to --out-dir: model.pth, config.json, vocab.json,
reference/*.wav (speaker conditioning), meta.json.

Progress markers are printed to stdout (TRAIN_START / per-step lines from the
trainer / TRAIN_DONE <dir>) so a caller can follow along.
"""
from __future__ import annotations

import argparse
import glob
import json
import os
import shutil
import sys


def base_dir() -> str:
    return os.path.join(os.environ.get("LOCALAPPDATA", ""), "tts",
                        "tts_models--multilingual--multi-dataset--xtts_v2")


def main() -> int:
    ap = argparse.ArgumentParser(description="Fine-tune XTTS-v2 on a personal voice.")
    ap.add_argument("--dataset-dir", required=True)
    ap.add_argument("--out-dir", required=True, help="Final saved-model directory to create.")
    ap.add_argument("--language", default="en")
    ap.add_argument("--epochs", type=int, default=12)
    ap.add_argument("--batch-size", type=int, default=3)
    ap.add_argument("--grad-accum", type=int, default=4)
    ap.add_argument("--max-audio-length", type=int, default=255995)  # ~11.6s @22.05k
    args = ap.parse_args()

    base = base_dir()
    XTTS_CHECKPOINT = os.path.join(base, "model.pth")
    XTTS_CONFIG = os.path.join(base, "config.json")
    TOKENIZER = os.path.join(base, "vocab.json")
    DVAE = os.path.join(base, "dvae.pth")
    MEL = os.path.join(base, "mel_stats.pth")
    for p in (XTTS_CHECKPOINT, XTTS_CONFIG, TOKENIZER, DVAE, MEL):
        if not os.path.isfile(p):
            print(f"error: missing base file {p}", file=sys.stderr)
            return 1

    from TTS.tts.layers.xtts.trainer.gpt_trainer import GPTArgs, GPTTrainer, GPTTrainerConfig
    from TTS.tts.models.xtts import XttsAudioConfig
    from TTS.config.shared_configs import BaseDatasetConfig
    from TTS.tts.datasets import load_tts_samples
    from trainer import Trainer, TrainerArgs

    run_dir = os.path.join(args.out_dir, "_run")
    os.makedirs(run_dir, exist_ok=True)

    dataset = BaseDatasetConfig(
        formatter="coqui", dataset_name="ft", path=args.dataset_dir,
        meta_file_train="metadata_train.csv", meta_file_val="metadata_eval.csv",
        language=args.language,
    )

    model_args = GPTArgs(
        max_conditioning_length=132300, min_conditioning_length=66150,
        debug_loading_failures=False, max_wav_length=args.max_audio_length, max_text_length=200,
        mel_norm_file=MEL, dvae_checkpoint=DVAE, xtts_checkpoint=XTTS_CHECKPOINT, tokenizer_file=TOKENIZER,
        gpt_num_audio_tokens=1026, gpt_start_audio_token=1024, gpt_stop_audio_token=1025,
        gpt_use_masking_gt_prompt_approach=True, gpt_use_perceiver_resampler=True,
    )
    audio = XttsAudioConfig(sample_rate=22050, dvae_sample_rate=22050, output_sample_rate=24000)
    config = GPTTrainerConfig(
        epochs=args.epochs, output_path=run_dir, model_args=model_args,
        run_name="ft", project_name="xtts_ft", audio=audio,
        batch_size=args.batch_size, batch_group_size=48, eval_batch_size=args.batch_size,
        num_loader_workers=0, num_eval_loader_workers=0,  # 0 = safe on Windows
        print_step=10, plot_step=100, log_model_step=1000, save_step=50,
        save_n_checkpoints=1, save_checkpoints=True, save_best_after=0, print_eval=False,
        optimizer="AdamW", optimizer_wd_only_on_weights=True,
        optimizer_params={"betas": [0.9, 0.96], "eps": 1e-8, "weight_decay": 1e-2},
        lr=5e-06, lr_scheduler="MultiStepLR",
        lr_scheduler_params={"milestones": [900000, 2700000, 5400000], "gamma": 0.5, "last_epoch": -1},
        test_sentences=[],
    )

    model = GPTTrainer.init_from_config(config)
    train_samples, eval_samples = load_tts_samples(
        [dataset], eval_split=True, eval_split_max_size=256, eval_split_size=0.15)
    print(f"TRAIN_SAMPLES {len(train_samples)} EVAL_SAMPLES {len(eval_samples)}", flush=True)

    trainer = Trainer(
        TrainerArgs(restore_path=None, skip_train_epoch=False, start_with_eval=False,
                    grad_accum_steps=args.grad_accum),
        config, output_path=run_dir, model=model,
        train_samples=train_samples, eval_samples=eval_samples,
    )
    print("TRAIN_START", flush=True)
    trainer.fit()
    print("TRAIN_FIT_DONE", flush=True)

    # locate the fine-tuned checkpoint produced under run_dir
    ckpts = (glob.glob(os.path.join(run_dir, "**", "best_model.pth"), recursive=True)
             or glob.glob(os.path.join(run_dir, "**", "checkpoint_*.pth"), recursive=True)
             or glob.glob(os.path.join(run_dir, "**", "*.pth"), recursive=True))
    if not ckpts:
        print("error: no checkpoint produced", file=sys.stderr)
        return 1
    ckpt = max(ckpts, key=os.path.getmtime)

    os.makedirs(args.out_dir, exist_ok=True)
    shutil.copyfile(ckpt, os.path.join(args.out_dir, "model.pth"))
    shutil.copyfile(XTTS_CONFIG, os.path.join(args.out_dir, "config.json"))
    shutil.copyfile(TOKENIZER, os.path.join(args.out_dir, "vocab.json"))

    # speaker conditioning = a few of the longest training clips
    ref_dir = os.path.join(args.out_dir, "reference")
    os.makedirs(ref_dir, exist_ok=True)
    wavs = sorted(glob.glob(os.path.join(args.dataset_dir, "wavs", "*.wav")),
                  key=os.path.getsize, reverse=True)[:6]
    for i, w in enumerate(wavs):
        shutil.copyfile(w, os.path.join(ref_dir, f"ref_{i}.wav"))

    json.dump({"language": args.language, "samples": len(train_samples) + len(eval_samples)},
              open(os.path.join(args.out_dir, "meta.json"), "w"))
    import logging
    logging.shutdown()  # release the trainer's file log handle so cleanup can remove it
    shutil.rmtree(run_dir, ignore_errors=True)
    print(f"TRAIN_DONE {args.out_dir}", flush=True)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

## File 25 of 27 — `%USERPROFILE%\.claude\skills\local-voice\scripts\xtts_infer.py`

```python
#!/usr/bin/env python3
"""Synthesize speech from a fine-tuned XTTS voice model (persistent worker).

Runs in the `xtts` conda env. Loads one saved voice model (produced by
xtts_train.py) and serves synthesis requests over stdin/stdout JSON:

    {"text": "...", "out": "path.wav"[, "language": "en"]}  ->  {"ok": true, "out": path}

The saved model dir must contain: config.json, model.pth, vocab.json, reference/*.wav.
Library log noise is redirected to stderr so stdout carries only the protocol.
"""
from __future__ import annotations

import argparse
import glob
import json
import os
import sys


def serve(model_dir: str) -> int:
    real = os.dup(1)
    os.dup2(2, 1)
    proto = os.fdopen(real, "w", buffering=1, encoding="utf-8", errors="replace")

    def reply(obj):
        proto.write(json.dumps(obj) + "\n"); proto.flush()

    try:
        os.environ["COQUI_TOS_AGREED"] = "1"
        import torch
        import torchaudio
        from TTS.tts.configs.xtts_config import XttsConfig
        from TTS.tts.models.xtts import Xtts

        config = XttsConfig()
        config.load_json(os.path.join(model_dir, "config.json"))
        model = Xtts.init_from_config(config)
        model.load_checkpoint(config, checkpoint_dir=model_dir, use_deepspeed=False)
        if torch.cuda.is_available():
            model.cuda()
        refs = sorted(glob.glob(os.path.join(model_dir, "reference", "*.wav")))
        if not refs:
            raise RuntimeError("no reference wavs in model dir")
        gpt_cond_latent, speaker_embedding = model.get_conditioning_latents(audio_path=refs)
        meta = {}
        mp = os.path.join(model_dir, "meta.json")
        if os.path.isfile(mp):
            meta = json.load(open(mp))
        default_lang = meta.get("language", "en")
    except Exception as e:
        reply({"error": f"load failed: {e}"}); return 1
    reply({"ready": True})

    for line in sys.stdin:
        line = line.strip()
        if not line:
            continue
        try:
            req = json.loads(line)
            text = (req.get("text") or "").strip()
            if not text:
                raise RuntimeError("no text")
            out = req["out"]
            lang = req.get("language", default_lang)
            # Fixed seed per chunk: XTTS's voice timbre drifts across a long book
            # otherwise (sampling state carries over) — pin it for one narrator.
            torch.manual_seed(int(req.get("seed", 42)))
            result = model.inference(
                text, lang, gpt_cond_latent, speaker_embedding,
                temperature=float(req.get("temperature", 0.7)),
            )
            wav = torch.tensor(result["wav"], dtype=torch.float32)
            # audiobook naturalness: edge fades (concat clicks) + requested pause
            n = wav.shape[-1]
            f = min(int(24000 * 0.005), n // 2)
            if f > 0:
                wav[:f] *= torch.linspace(0.0, 1.0, f)
                wav[-f:] *= torch.linspace(1.0, 0.0, f)
            pad = int(24000 * max(0, int(req.get("pad_ms") or 0)) / 1000)
            if pad:
                wav = torch.cat([wav, torch.zeros(pad)])
            torchaudio.save(out, wav.unsqueeze(0), 24000)
            reply({"ok": True, "out": out})
        except Exception as e:
            reply({"error": str(e)})
    return 0


def main() -> int:
    ap = argparse.ArgumentParser(description="Synthesize from a fine-tuned XTTS voice.")
    ap.add_argument("--model-dir", required=True)
    ap.add_argument("--serve", action="store_true")
    args = ap.parse_args()
    return serve(args.model_dir)


if __name__ == "__main__":
    raise SystemExit(main())
```

## File 26 of 27 — `%USERPROFILE%\.claude\skills\local-voice\scripts\scripts.json`

```json
[
  "The dog ran across the yard.",
  "I had eggs and toast for breakfast.",
  "She closed the door and went outside.",
  "We watched a movie last night.",
  "The sky turned dark before the rain.",
  "He put the keys on the table.",
  "Can you pass me the salt, please?",
  "The kids played in the park all day.",
  "I need to buy milk and bread.",
  "The train arrived right on time.",
  "My phone is almost out of battery.",
  "Let's meet at the coffee shop later.",
  "The cat slept on the warm windowsill.",
  "I forgot my umbrella at home.",
  "They painted the fence a bright blue.",
  "Dinner will be ready in ten minutes.",
  "The book was better than the film.",
  "Please turn off the lights when you leave.",
  "We drove to the beach on Sunday.",
  "The garden is full of red flowers.",
  "I can hear the birds singing outside.",
  "He fixed the bike in the garage.",
  "She wrote a letter to her friend.",
  "The soup needs a little more salt.",
  "We took a long walk after lunch.",
  "The store closes at nine tonight.",
  "I left my jacket on the chair.",
  "The baby fell asleep in the car.",
  "They built a sandcastle near the water.",
  "Thanks for helping me move the boxes.",
  "It started raining just as we left.",
  "The coffee is still too hot to drink.",
  "I parked the car behind the house.",
  "She smiled and waved from the window.",
  "We need more chairs for the party.",
  "The road was quiet early in the morning.",
  "He read the news while eating lunch.",
  "Put your shoes by the front door.",
  "The lake looked calm and clear today.",
  "I will call you back in an hour."
]
```

## File 27 of 27 — `%USERPROFILE%\.claude\skills\local-tts\scripts\zonos_tts.py`

```python
#!/usr/bin/env python3
"""local-tts (Zonos): high-quality expressive TTS via Zyphra Zonos-v0.1-transformer.

Runs in the `zonos` conda env. Same persistent worker protocol as speak.py:

    --serve  then newline JSON on stdin:
        {"text": "...", "out": "path.wav" [, "ref": "speaker.wav"] [, "language": "en-us"]}
      -> {"ok": true, "out": path}   (or {"error": "..."})

Zonos needs a speaker embedding for a consistent voice. Priority:
  1. per-request "ref" wav (cloned; cached by path),
  2. a default reference next to this script (zonos_default_speaker.wav),
  3. unconditioned with a fixed seed (least consistent — provide a ref for real books).
"""
from __future__ import annotations

import argparse
import os
import sys

for _s in (sys.stdout, sys.stderr):
    try:
        _s.reconfigure(encoding="utf-8", errors="replace")
    except Exception:
        pass

_HERE = os.path.dirname(os.path.abspath(__file__))
_DEFAULT_REF = os.path.join(_HERE, "zonos_default_speaker.wav")

# Zonos emotion vector order: [happiness, sadness, disgust, fear, surprise, anger, other, neutral]
_EMOTIONS = {
    "neutral": [0.2, 0.05, 0.05, 0.05, 0.05, 0.05, 0.1, 0.8],
    "warm":    [0.7, 0.02, 0.02, 0.02, 0.10, 0.02, 0.05, 0.30],
    "somber":  [0.02, 0.75, 0.03, 0.05, 0.02, 0.02, 0.05, 0.30],
    "tense":   [0.02, 0.10, 0.05, 0.60, 0.15, 0.10, 0.05, 0.20],
    "intense": [0.05, 0.05, 0.05, 0.10, 0.15, 0.65, 0.05, 0.20],
}


def _harden_torch():
    """Windows fixes: Zonos calls torch.compile (needs MSVC cl.exe, absent here) — force
    eager. Also let dynamo fall back instead of hard-failing."""
    import torch

    def _no_compile(model=None, *a, **k):
        return model if model is not None else (lambda f: f)
    torch.compile = _no_compile
    try:
        torch._dynamo.config.suppress_errors = True
        torch._dynamo.disable()
    except Exception:
        pass
    # Zonos builds its speaker-embedding mel filterbank under `with torch.device('cuda')`,
    # and torchaudio's melscale_fbanks then mixes a CPU constant with CUDA tensors →
    # "found at least two devices". Force every MelSpectrogram to build on CPU, then move
    # to the (default) target device — so speaker cloning works and the narrator stays
    # consistent across chunks. (Avoids patching the Zonos library source.)
    try:
        import torchaudio
        _Mel = torchaudio.transforms.MelSpectrogram
        if not getattr(_Mel, "_cpu_patched", False):
            class _CpuMel(_Mel):
                _cpu_patched = True

                def __init__(self, *a, **k):
                    tgt = torch.empty(0).device
                    with torch.device("cpu"):
                        super().__init__(*a, **k)
                    if str(tgt) != "cpu":
                        self.to(tgt)
            torchaudio.transforms.MelSpectrogram = _CpuMel
    except Exception:
        pass
    return torch


def serve() -> int:
    import json
    real = os.dup(1)
    os.dup2(2, 1)                      # keep stdout(1) clean for the JSON protocol
    proto = os.fdopen(real, "w", buffering=1, encoding="utf-8", errors="replace")

    def reply(obj):
        proto.write(json.dumps(obj) + "\n"); proto.flush()

    try:
        torch = _harden_torch()
        import torchaudio
        from zonos.model import Zonos
        from zonos.conditioning import make_cond_dict

        device = "cuda" if torch.cuda.is_available() else "cpu"
        model = Zonos.from_pretrained("Zyphra/Zonos-v0.1-transformer", device=device)
        sr_out = model.autoencoder.sampling_rate
        spk_cache = {}

        def speaker_for(ref):
            key = ref or "__default__"
            if key in spk_cache:
                return spk_cache[key]
            path = ref or (_DEFAULT_REF if os.path.isfile(_DEFAULT_REF) else None)
            emb = None
            if path and os.path.isfile(path):
                try:
                    wav, sr = torchaudio.load(path)
                    emb = model.make_speaker_embedding(wav, sr)
                except Exception as e:
                    sys.stderr.write(f"[zonos] speaker embedding failed, using seeded voice: {e}\n")
                    emb = None
            spk_cache[key] = emb
            return emb

        def synth(req):
            torch.manual_seed(1234)   # consistent narrator across chunks
            text = (req.get("text") or "").strip()
            speed = float(req.get("speed") or 1.0)                 # 0.5–2.0
            rate = max(5.0, min(35.0, 15.0 * speed))               # Zonos speaking_rate
            pitch = float(req.get("pitch_std") if req.get("pitch_std") is not None else 20.0)
            cond_kw = dict(text=text, speaker=speaker_for(req.get("ref")),
                           language=req.get("language") or "en-us",
                           speaking_rate=rate, pitch_std=pitch)
            emo = _EMOTIONS.get((req.get("emotion") or "").lower())
            if emo is not None:
                cond_kw["emotion"] = emo
            codes = model.generate(model.prepare_conditioning(make_cond_dict(**cond_kw)))
            wav = model.autoencoder.decode(codes).cpu()[0]
            # audiobook naturalness: edge fades (concat clicks) + requested pause
            n = wav.shape[-1]
            f = min(int(sr_out * 0.005), n // 2)
            if f > 0:
                wav[..., :f] *= torch.linspace(0.0, 1.0, f)
                wav[..., -f:] *= torch.linspace(1.0, 0.0, f)
            pad = int(sr_out * max(0, int(req.get("pad_ms") or 0)) / 1000)
            if pad:
                wav = torch.cat([wav, torch.zeros(wav.shape[0], pad)], dim=-1)
            torchaudio.save(req["out"], wav, sr_out)
            return req["out"]
    except Exception as e:
        reply({"error": f"load failed: {e}"}); return 1

    reply({"ready": True})
    for line in sys.stdin:
        line = line.strip()
        if not line:
            continue
        try:
            req = json.loads(line)
            if not (req.get("text") or "").strip():
                raise RuntimeError("no text")
            path = synth(req)
            reply({"ok": True, "out": path})
        except Exception as e:
            reply({"error": str(e)})
    return 0


def main() -> int:
    ap = argparse.ArgumentParser(description="Zonos TTS worker.")
    ap.add_argument("text", nargs="*")
    ap.add_argument("--out", default="zonos_out.wav")
    ap.add_argument("--ref")
    ap.add_argument("--language", default="en-us")
    ap.add_argument("--serve", action="store_true")
    args = ap.parse_args()
    if args.serve:
        return serve()
    # one-shot
    torch = _harden_torch()
    import torchaudio
    from zonos.model import Zonos
    from zonos.conditioning import make_cond_dict
    text = " ".join(args.text).strip() or "Hello."
    device = "cuda" if torch.cuda.is_available() else "cpu"
    model = Zonos.from_pretrained("Zyphra/Zonos-v0.1-transformer", device=device)
    torch.manual_seed(1234)
    spk = None
    ref = args.ref or (_DEFAULT_REF if os.path.isfile(_DEFAULT_REF) else None)
    if ref and os.path.isfile(ref):
        try:
            wav, sr = torchaudio.load(ref)
            spk = model.make_speaker_embedding(wav, sr)
        except Exception as e:
            sys.stderr.write(f"[zonos] speaker embedding failed, using seeded voice: {e}\n")
    cond = make_cond_dict(text=text, speaker=spk, language=args.language)
    codes = model.generate(model.prepare_conditioning(cond))
    out_wav = model.autoencoder.decode(codes).cpu()
    torchaudio.save(args.out, out_wav[0], model.autoencoder.sampling_rate)
    print(args.out)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```
