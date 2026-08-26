# Model stack review — August 2026

*Audit of every model in Local AI Studio against what has shipped as of 26 August 2026.
Nothing in this document has been applied. It is a decision aid, not a changelog.*

Hardware constraint throughout: **RTX 4080 SUPER, 16 GB VRAM**, one heavy model resident at
a time. That single fact rules out most of 2026's headline releases, so "better" here always
means "better *and* it fits".

---

## Verdict at a glance

| Area | Current | Verdict |
|---|---|---|
| Image | FLUX.2 Klein 4B | **Keep** — still best-in-class at this size |
| Music | ACE-Step 1.5 XL | **Keep** — is the latest |
| Fiction LLM | Cydonia-24B-v4.3 | **Keep** — newest at this size |
| Preset TTS | Kokoro-82M | **Keep** |
| Orchestral | SymphonyGen | **Keep** — no competitor |
| Melody | basic-pitch + FCPE | **Keep** |
| Video | MiniMax H3 | **Keep** — see §1.2 |
| **STT** | **Parakeet-TDT 0.6b-v2** | **Superseded → v3** |
| **Cutout** | **rembg u2net** | **Superseded → BiRefNet** |
| **General LLM** | **gpt-oss:20b** | **Keep the model, fix the config** — §4 |
| Separation | htdemucs_ft | Partial — see §3.1 |
| Cloning TTS | Chatterbox / Zonos | Optional upgrades — §3.2 |
| Voice fine-tune | XTTS-v2 | ⚠️ Licence liability — §3.4 |

---

## 1. Already current — keep

### 1.1 The clear holds

**FLUX.2 Klein 4B** (Jan 2026, Apache-2.0) is still the model to have on a 16 GB card.
Community comparisons through 2026 put it **first on prompt adherence**, with Z-Image Turbo
second and Qwen-Image 2512 third, and ahead on portraits (4.2/5 vs Z-Image's 4.0). The only
reason to add Qwen-Image would be rendering readable text *inside* an image, which is not
what Sprite Studio or Image Edit are for. No change.

**ACE-Step 1.5 XL** — v1.5 released 28 January 2026. This *is* the current release; there is
no v2. DiffRhythm 2 (Xiaomi/ASLP, Apache-2.0) is the most architecturally novel thing in the
2026 slate with its Block Flow Matching, but novelty is not a quality win and it would mean
rewriting the four places ACE-Step is wired (generation plus three audio-to-audio polish
passes). No change.

**Cydonia-24B-v4.3** — TheDrummer's most recent 24B. Everything newer from that author is
either much smaller (Rocinante-X-12B, Anubis-Mini-8B) or far too large (Behemoth-128B). No
change.

**Kokoro-82M**, **SymphonyGen**, **basic-pitch + FCPE** — all still the right calls, none
with a credible local successor.

### 1.2 Video: LTX-2.5 considered and declined

LTX-2.5 (Lightricks, August 2026) is the obvious challenger — open weights, native
synchronised audio+video in one pass like H3, native ComfyUI support, and a much faster
architecture. Three reasons it loses here:

1. **The licence argument is gone.** It was the strongest case for switching, but this
   machine now holds individual authorisation from MiniMax for local deployment.
2. **VRAM.** LTX's own documentation lists **32 GB+** for the standard local path. 16 GB is
   the "minimum to run at all" figure, achievable only with FP8 at the low end of the
   resolution table and with the upscaler off on longer clips.
3. **Sunk tuning.** `h3gen.py` carries hard-won, measured settings for this exact card — the
   int8-beats-GGUF finding, the 768p floor, the latent-grid arithmetic, and an ETA cache
   built from real timings. All of that would have to be rediscovered.

Revisit if native 4K becomes a requirement, or if a 16 GB-native LTX quantisation lands.

### 1.3 Ruled out on VRAM

| Model | Size | Why not |
|---|---|---|
| Qwen3.6:27b | 18 GB | Best local coder (77.2% SWE-bench Verified) but spills to system RAM |
| Qwen3.8:27b | 18 GB | Same |
| GLM-4.7-flash | 19 GB | Same |
| Gemma4:26b (MoE) | 19 GB | Same |

**Gemma4:12b (7.6 GB, April 2026, Apache-2.0)** is the one 2026 model that fits with room to
spare, and it handles text, image *and* audio with a 256K context — it could in principle
replace both `gpt-oss:20b` and `qwen3-vl:8b` with one tag at half the VRAM. Recorded here
because it is the only genuine "newer and it fits" option; not recommended for action since
gpt-oss stays.

---

## 2. Superseded — both are drop-in

These two swap points are already parameterised. Testing either costs nothing.

### 2.1 Parakeet-TDT `0.6b-v2` → `0.6b-v3`

**Where:** `~/.claude/skills/local-stt/scripts/transcribe.py:79` — the `--model` default.

Same 600M-parameter architecture, same NeMo API, same `from_pretrained` call. v3 was
initialised from a CTC multilingual checkpoint pretrained on the Granary dataset, then
fine-tuned on ~7,500 hours of human-transcribed audio.

**What it buys:** 25 European languages with **automatic language detection** (no prompting
needed), *and* better accuracy than v2 — NVIDIA describe it as "smaller yet more accurate".
It has the highest throughput of any multilingual model on the Hugging Face leaderboard.

**Affects:** the Speech → Text tab, and Voice Studio's speak-instead-of-typing path
(`server.py:634-641`).

**Check before adopting:** confirm `--timestamps` still returns word-level hypotheses under
the installed NeMo 2.7.3, since `video-understand` depends on them.

### 2.2 rembg `u2net` → `birefnet-general`

**Where:** `spritekit.py:43` — already reads `os.environ.get("REMBG_MODEL", "u2net")`. A swap
needs no code change at all, only the env var.

u2net was trained for general salient-object detection. BiRefNet uses a bilateral reference
network built to preserve fine-grained edges, and handles **hair, thin geometry and complex
fabric** dramatically better — which is precisely where sprite cutouts fail today. Bria's
RMBG-2.0 is built on the same BiRefNet architecture. rembg ≥ 2.0.75 ships
`birefnet-general` and a lighter `birefnet-general-lite`.

**Cost:** BiRefNet is a heavier model than u2net, so the post-process pass per frame will
slow down. For a 9-action sprite set that is worth measuring before committing.

---

## 3. Worth considering — real gain, real work

### 3.1 Mel-Band RoFormer, for the vocal stem only

Not a wholesale Demucs replacement. The consensus position in 2026 is that **htdemucs_ft
remains the right general default**, and that RoFormer variants win in specific places —
BS-RoFormer on bass, Mel-Band RoFormer on **vocals**, where the mel-band projection buys
0.43–0.58 dB over band-split and produces smoother output with more consistent loudness.

That matters here because Lullaby's entire pipeline hangs off a clean *vocal* stem: the
Melody Match engine traces f0 from it, and basic-pitch transcribes it. A better vocal stem
propagates through every downstream stage.

**Shape of the work:** add `melband-roformer-infer` or `audio-separator` to `lullabykit/.venv`,
run it for vocals only, keep htdemucs_ft for the remaining stems.

**Caveat worth knowing up front:** the community model registry has significant link-rot —
of 89 registry models audited, only 37 had both checkpoint and config still live. Pin a
specific checkpoint and vendor it rather than resolving at runtime.

### 3.2 TTS: two optional upgrades

- **Chatterbox → Chatterbox Multilingual / Turbo.** Turbo is 350M, MIT, sub-200 ms with
  tuning, and still wins blind tests against ElevenLabs (63.75% preference) — but it is
  English-only. Multilingual is the broader upgrade. Modest either way; current Chatterbox
  already does the cloning job.
- **Higgs Audio V2 (~5.8B, Apache-2.0)** is the notable one: best overall naturalness in the
  2026 field. Zonos v0.1 has been quiet since release, and Higgs would be a stronger
  max-quality Audiobook narrator. It would slot in the same way Zonos does — own conda env,
  own worker.

### 3.3 What NOT to bother with

Fish Speech, CosyVoice2, IndexTTS-2, NeuTTS Air, OmniVoice, VibeVoice, Supertonic 3 all
appear in 2026 roundups. None of them beats what is already installed across four TTS
engines. This slot is saturated.

### 3.4 ⚠️ XTTS-v2 is a licence liability

Voice Studio's base model is Coqui XTTS-v2, released under the **Coqui Public Model Licence —
non-commercial, personal/artistic use only**. This is the same class of constraint that the
H3 notice exists for, and it currently gets one line in the README's credits.

The commercial-safe equivalent is **Fish Speech (Apache-2.0, MOS 4.1, eight languages)** with
a reference-audio workflow deliberately familiar to Coqui users. But Voice Studio is a
*fine-tuning* pipeline (`xtts_train.py` / `xtts_infer.py`), not just inference, and the saved
`voices/Me/model.pth` is a 5.6 GB XTTS checkpoint that would not carry over. This is flagged,
not recommended — the work is disproportionate unless commercial use of cloned voices is
actually on the table.

---

## 4. The LLM question: is the hardware the limit?

**No.** An RTX 4080 SUPER runs gpt-oss:20b at roughly **111–189 tok/s** — near the top of what
consumer hardware achieves with this model, and an order of magnitude above what a
CPU-offloaded larger model would give. The 20B is an MoE with only ~3.6B parameters active
per token, which is why it feels fast for its size. The box is not the constraint.

**Do not switch harness.** vLLM's advantage is concurrent batching, which a single-user
studio never generates. llama.cpp offers finer control and is already on this machine
(koboldcpp is a llama.cpp fork), but every lever below is available inside Ollama.

The gap is configuration, and there are four findings.

### 4.1 Context is silently capped at 4096 tokens ⚠️

This is the most consequential finding in the review.

`ask.py:104` posts to Ollama's **`/v1/chat/completions`**. Ollama's OpenAI-compatible
endpoint **does not accept `num_ctx`** — it is documented as not settable there — and
Ollama's default context window is **4096 tokens**. gpt-oss:20b supports 128K. Nothing in
`server.py`, `studioctl.ps1` or the launchers sets `OLLAMA_CONTEXT_LENGTH`.

Every long input to the Language tab is therefore being truncated, with no warning.

The sharpest case is Composer's "Arrange" planner at `server.py:3376-3379`, which requests
`max_tokens=4096` from a model whose entire window is 4096. Once the system prompt and brief
are counted, the requested output length is unreachable — a plausible mechanism behind
flaky/truncated Composer JSON.

**Fix:** either set `OLLAMA_CONTEXT_LENGTH=32768` on the Ollama service, or `ollama create` a
Modelfile variant with `PARAMETER num_ctx 32768` and point the studio at that tag. Note the
precedence rule: a Modelfile `num_ctx` **overrides** the environment variable, so a baked-in
4096 would silently win.

**VRAM:** a larger window costs KV cache. That is what §4.4 pays for.

### 4.2 `reasoning_effort` is pinned to "low"

`ask.py:55` defaults `--reasoning-effort` to `low`, and **no `server.py` caller ever passes
it**. So the Language tab, Composer's planner and the Story fallback all run gpt-oss at its
shallowest reasoning setting. This is the single largest quality knob the model has.

The code comment explains the original reason honestly — without a low effort the model
"can spend the entire token budget thinking and return an empty answer". But that root cause
was separately fixed: `server.py:801-802` already raises `max_tokens` specifically because
"the hidden reasoning is also billed against max_tokens". **The workaround has outlived its
cause.**

Two things to note:
- Ollama's `/v1` endpoint does support `reasoning_effort`, so the plumbing already works.
- Ollama now accepts **`"max"`** (above `"high"`) and `"none"`. `ask.py`'s
  `choices=["low","medium","high"]` rejects both.

### 4.3 Temperature defaults to 0.2

`ask.py:49`. OpenAI's guidance for gpt-oss is **temperature 1.0 / top_p 1.0**. Reasoning
models degrade when over-constrained — the chain-of-thought collapses toward repetition
rather than exploring. 0.2 is a sensible default for a *non*-reasoning code model, which is
likely where it came from.

### 4.4 Flash attention and KV cache quantisation are off

Neither `OLLAMA_FLASH_ATTENTION` nor `OLLAMA_KV_CACHE_TYPE` is set anywhere.

`OLLAMA_FLASH_ATTENTION=1` plus `OLLAMA_KV_CACHE_TYPE=q8_0` roughly **halves KV cache
memory** for a perplexity increase benchmarked at 0.002–0.05 — imperceptible in practice.
KV quantisation requires flash attention to be active. This is what makes §4.1's larger
context affordable inside 16 GB.

**Caveat:** on architectures not in Ollama's allowlist the server **silently falls back to
f16**. Setting the variable is not proof it took effect — verify against `nvidia-smi`.

### 4.5 How to adopt these

All four are configuration, all four are reversible, and all four change model *output*.
Per this project's standing rule, none should be accepted on a benchmark or a wall-clock
number: A/B them at a fixed seed on a real prompt from the tab in question — a long
document for §4.1, a Composer brief for §4.2 — and judge the artifact.

Suggested order: **§4.1 first** (it is a correctness issue, not a tuning preference), then
§4.4 to pay for it, then §4.2, then §4.3.

---

## Sources

- [Best Ollama Models 2026 (Morph)](https://www.morphllm.com/best-ollama-models) ·
  [Ollama library](https://ollama.com/library) ·
  [Ollama OpenAI compatibility](https://docs.ollama.com/openai) ·
  [Ollama FAQ](https://docs.ollama.com/faq)
- [Ollama env var reference](https://modelpiper.com/blog/ollama-environment-variables) ·
  [KV cache quantisation](https://modelpiper.com/blog/ollama-kv-cache-quantization) ·
  [Bringing K/V quantisation to Ollama](https://smcleod.net/2024/12/bringing-k/v-context-quantisation-to-ollama/)
- [gpt-oss run guide (Unsloth)](https://unsloth.ai/docs/models/gpt-oss-how-to-run-and-fine-tune) ·
  [GPT-OSS 20B local hardware guide](https://runaihome.com/blog/gpt-oss-20b-local-ai-hardware-guide-2026/)
- [parakeet-tdt-0.6b-v3](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3) ·
  [Canary-1B-v2 & Parakeet-TDT-0.6B-v3 paper](https://arxiv.org/pdf/2509.14128)
- [BiRefNet vs rembg vs U2Net](https://dev.to/om_prakash_3311f8a4576605/birefnet-vs-rembg-vs-u2net-which-background-removal-model-actually-works-in-production-4830) ·
  [RMBG v2.0](https://bria.ai/introducing-the-rmbg-v2-0-model-the-next-generation-in-background-removal-from-images)
- [htdemucs vs BS-RoFormer benchmark 2026](https://dev.to/codesugar_lin_037a57b06a4/htdemucs-vs-bs-roformer-vs-spleeter-a-2026-audio-source-separation-benchmark-2ll8) ·
  [Mel-Band RoFormer paper](https://arxiv.org/pdf/2310.01809)
- [Z-Image vs FLUX.2 Klein](https://zimage.run/blog/zi-121-z-image-vs-flux2-klein-en-20260703) ·
  [Best open-source image models 2026](https://www.thundercompute.com/blog/best-open-source-image-generation-models)
- [LTX-2.5 open-weight release](https://www.opensourceforu.com/2026/08/ltx-2-5-brings-open-weight-video-generation/) ·
  [LTX-2.5 local VRAM guide](https://runaihome.com/blog/ltx-2-5-local-ai-video-hardware-guide-2026/)
- [ACE-Step](https://github.com/ace-step/ACE-Step) ·
  [Best open-source TTS 2026](https://www.bentoml.com/blog/exploring-the-world-of-open-source-text-to-speech-models) ·
  [TheDrummer on Hugging Face](https://huggingface.co/TheDrummer)
