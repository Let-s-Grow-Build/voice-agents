# 4. Current Speech-to-Speech / Native Audio Models (2026 Landscape)

This file covers the models with the most publicly documented architectural detail as of August 2026. For each, the focus is: **what architectural innovation did this model actually introduce**, not just "it exists."

---

## 4.1 OpenAI — `gpt-realtime` (Realtime API)

- **Architecture**: Built on the GPT-4o audio-model lineage; described by OpenAI as natively multimodal, understanding and generating audio and text as first-class token types within one model. OpenAI has not published a full architecture paper with codec/token details — this is publicly the least-documented internals among the models covered here.
- **Native vs. cascaded**: Native — OpenAI explicitly frames the Realtime API as collapsing the STT→LLM→TTS stack into a single streaming speech-to-speech process.
- **I/O modalities**: Audio and text input; audio and/or text output. The newer `gpt-realtime-2.1` / `gpt-audio-1.5` generation adds image input.
- **Streaming architecture**: WebSocket/WebRTC event protocol (`wss://api.openai.com/v1/realtime`), with events like `input_audio_buffer.append` (stream audio in) and `response.audio.delta` (stream audio out), plus a transcript event stream for observability.
- **Latency**: OpenAI's own marketing claims sub-500ms response latency for the collapsed pipeline, versus the 3–5s latency they cite for legacy cascaded pipelines.
- **Context handling**: Fine-grained conversation-context controls were added at GA — developers can set intelligent token limits and truncate multiple turns at once to manage cost on long sessions.
- **Multilingual**: Notable strength called out at GA: switching seamlessly between languages mid-sentence.
- **Voice generation**: Built-in preset voices; the model can be instructed on delivery style via natural-language instructions (separately, OpenAI's `gpt-4o-mini-tts` text-to-speech model — used in non-realtime contexts — is explicitly steerable this way too).
- **Emotion/prosody**: Marketed capability — "detect emotion, handle interruptions, and respond" naturally — though not backed by a public technical paper.
- **Tool/function calling**: Mature and actively extended — GA added remote **MCP server support**, image input, and SIP-based phone calling for telephony integration, positioning this squarely as an enterprise-agent-ready feature set.
- **Major architectural innovation**: Being the first major commercial vendor to ship a production, generally-available, single-model speech-to-speech API with mature tool-calling and telephony support — effectively defining the commercial category.
- **Limitations**: Internal architecture is not public (harder to reason about failure modes); audio tokens are priced significantly higher than text tokens (~$32–64 per 1M audio tokens vs. $5–20 per 1M text tokens), which matters at scale; tightly coupled to OpenAI as a single vendor.

---

## 4.2 Google DeepMind — Gemini Live API / Native Audio models (Gemini 2.5 Flash Native Audio, Gemini 3.1 Flash Live)

- **Architecture**: Native-audio variants of the Gemini family that stream audio in and generate audio out directly, without an explicit intermediate transcription step exposed to the pipeline (a transcript is still produced for developer visibility). Full internal token/codec design is not public.
- **Native vs. cascaded**: Native, explicitly marketed as "not an ASR→LLM→TTS pipeline."
- **I/O modalities**: Text, images, audio, and **video** input; text and audio output — the video-input capability is a notable differentiator versus most other STS vendors.
- **Streaming architecture**: Stateful WebSocket "Live API" session; both server-to-server and client-to-server integration patterns are supported.
- **Latency**: Optimized via "thinking level" controls — Gemini 3.1 models default to `minimal` thinking to prioritize the lowest latency, with the option to dial up reasoning depth (`low`/`medium`/`high`) when a task needs more deliberation, trading latency for accuracy.
- **Context handling**: Default 32K context window, upgradable to 128K; default session length caps at 10 minutes, requiring reconnection logic for longer conversations.
- **Multilingual**: Strong: seamless language switching without pre-configuration, live speech translation shipped as a beta feature in Google Translate supporting 70+ languages and roughly 2,000 language pairs with **style transfer** (preserving the original speaker's intonation across the translation).
- **Voice generation**: 30 HD voices across 24 languages.
- **Emotion/prosody**: Two named capabilities: **"Affective Dialog"** (understanding and responding appropriately to the user's emotional expression) and **"Proactive Audio"** (the model decides *whether* a given utterance was even directed at it, filtering out non-device-directed speech rather than responding to everything it hears).
- **Tool/function calling**: Reliable mid-dialogue function calling, benchmarked on "ComplexFuncBench Audio."
- **Major architectural innovation**: **Proactive Audio** — a genuinely distinctive capability: most STS systems assume every utterance in the audio stream is meant for them, but Gemini's native-audio models are trained to decide relevance before responding, which matters enormously for open-mic/ambient device scenarios (unlike a push-to-talk phone-call agent).
- **Limitations**: 10-minute default session cap requires explicit reconnection engineering; full internal architecture undisclosed; per audio-minute pricing detail depends on the specific preview vs. GA model tier.

---

## 4.3 Amazon — Nova Sonic / Nova 2 Sonic

- **Architecture**: Described in Amazon's own technical report as a genuinely **unified architecture that fuses speech and text modalities**, rather than three separate models glued together — Amazon explicitly contrasts this with the "cascaded models" approach they also document (ASR + NLU/LLM + TTS via frameworks like Pipecat) in their own engineering blog series.
- **Native vs. cascaded**: Native, by design — "combines these components into a unified model that processes audio in real time with a single forward pass."
- **I/O modalities**: Bidirectional audio streaming; cross-modal switching between voice and text within a single session (Nova 2 Sonic).
- **Streaming architecture**: Bidirectional streaming API on Amazon Bedrock, event-driven.
- **Latency**: Positioned as low-latency and industry-leading price-performance, but no specific millisecond figure is published in the sources reviewed here.
- **Context handling**: Nova 2 Sonic expanded context window up to **1M tokens** and added asynchronous tool use, letting the model continue conversing while a slow tool call resolves in the background.
- **Multilingual**: Nova 2 Sonic expanded to 7 languages with "polyglot voices" (a single voice that can speak multiple languages).
- **Voice generation**: Expressive masculine- and feminine-sounding voices; "adaptive speech response that dynamically adjusts delivery based on the prosody" of the user's input — i.e., the output speech style is explicitly conditioned on how the user spoke, not just what they said.
- **Emotion/prosody**: Central selling point — Amazon's own example: an angry customer is met with a calm, steady voice, while an excited customer is met with a more upbeat one, driven by the unified model picking up on tone.
- **Tool/function calling**: Native function calling, agentic workflow support, and RAG-based knowledge grounding built into the same session.
- **Major architectural innovation**: A named benchmark result — Nova 2 Sonic reportedly **outperforms other leading conversational AI models on Big Bench Audio**, a reasoning-with-audio-input evaluation, and shows strong BFCL/ComplexFuncBench (function-calling) scores, positioning Amazon's unified model as competitive on *agentic* capability, not just naturalness.
- **Limitations**: Amazon has not published deep architecture/token-design details comparable to Kyutai/Alibaba/Sesame; tightly tied to the Bedrock platform.

---

## 4.4 Kyutai — Moshi (open research model, publicly documented architecture)

- **Architecture (the most fully documented of any model in this survey)**: A 7B-parameter **Helium** text language model backbone combined with the **Mimi** neural audio codec (12.5Hz frame rate, RVQ with 8 codebooks, 1.1kbps, ~80ms streaming latency), fused via a **hierarchical RQ-Transformer**: a "Temporal Transformer" processes 17 parallel streams per timestep (1 text stream + 8 system-audio codebooks + 8 user-audio codebooks) and a smaller "Depth Transformer" autoregressively expands each timestep into the full set of audio codebook tokens.
- **Native vs. cascaded**: Fully native, and uniquely **full duplex** — Moshi processes the user's audio stream and its own audio stream *simultaneously* at every timestep, with no explicit turn-taking state machine.
- **I/O modalities**: Audio in, audio out, plus an internal parallel text stream.
- **Audio representation**: Discrete RVQ tokens from Mimi, split into one semantic codebook (layer 1) and seven acoustic codebooks (layers 2–8).
- **Reasoning mechanism**: The **"Inner Monologue"** method — the model predicts time-aligned text tokens as a prefix to the audio tokens at each step, which both improves the linguistic coherence of generated speech and gives the model streaming ASR/TTS-like behavior as an emergent byproduct of a single training objective, rather than as separate trained capabilities.
- **Streaming architecture**: Fully streaming end-to-end via the codec's causal convolutions and the transformer's autoregressive loop.
- **Latency characteristics**: **160ms theoretical, ~200ms practical** — the lowest, most concretely documented latency figure of any model in this survey, because there genuinely is no cross-service hop.
- **Context handling**: Limited by the 7B model's context window; Moshi is explicitly noted as having "limited abilities for complex tasks and cannot access tools."
- **Multilingual**: Primarily English in the base release; derivative fine-tunes (e.g., a documented Japanese adaptation) exist in the open research community, reusing Mimi's codec largely frozen and retraining only the RQ-Transformer.
- **Voice generation**: Reasonable naturalness at low latency, but explicitly research-grade rather than an enterprise product.
- **Emotion/prosody**: Preserved structurally by the semantic/acoustic codebook split, but not marketed with named "affective dialog"-style features the way commercial vendors do.
- **Tool/function calling**: **None** — Kyutai's own model card states this directly. This is Moshi's single biggest practical limitation for enterprise use.
- **Major architectural innovation**: (1) The **Mimi codec** itself — a widely reused, open building block (also underlying Sesame's CSM) that demonstrated semantic+acoustic RVQ tokens could run fully streaming at very low bitrate without sacrificing quality; (2) **genuine full-duplex, no-turn-model conversation** — trained on real overlapping-speech data (fine-tuned on the Fisher corpus specifically to handle overlap and interruption), Moshi is the clearest existing proof that turn-taking and barge-in can be learned *inside* a model rather than engineered around it.
- **Limitations**: No tool calling, limited complex reasoning versus larger commercial models, safety/watermarking still described as "remaining challenges" by the authors, primarily research/open-source rather than a hardened enterprise product.

---

## 4.5 Sesame — Conversational Speech Model (CSM)

- **Architecture**: Two Llama-architecture autoregressive transformers — a **backbone** that processes interleaved text and audio context and predicts the first (semantic) Mimi codebook token per frame, and a smaller **audio decoder** that autoregressively fills in the remaining acoustic codebook tokens conditioned on that first token. Reuses Kyutai's pretrained Mimi codec rather than training a new one.
- **Native vs. cascaded**: A **contextual TTS / dialogue-generation model**, not a full turn-taking conversational agent by itself — it's best understood as the "Talker" half of a full STS system, generating natural-sounding, context-aware speech from text+audio history, with Sesame's own larger, proprietary internal models (behind their consumer "Maya"/voice-companion product) presumably adding the conversational-agent layer on top (not publicly detailed).
- **I/O modalities**: Text and audio context in; audio out.
- **Audio representation**: Mimi RVQ tokens (12.5Hz), same codec family as Moshi.
- **Reasoning mechanism**: Not a general reasoning engine — CSM's job is expressive, context-aware speech *generation*, conditioned on prior conversational turns (represented as "Segments" of speaker + text + audio).
- **Latency characteristics**: Reported by third-party technical writeups as averaging around 380ms end-to-end, under a 500ms target — achieved partly through a "1/16 frame sampling" computational-efficiency trick to avoid the generation delays typical of naive RVQ decoding.
- **Context handling**: Documented as roughly a 2-minute/2048-token conversational memory window for the open 1B checkpoint.
- **Multilingual**: The public 1B checkpoint is primarily English, with some incidental multilingual leakage from training-data contamination; the open-source community has published guides for fine-tuning it into new languages.
- **Voice generation / emotion**: Marketed around "voice presence" — natural pauses, tone shifts, and contextually appropriate delivery, aiming specifically at conversational naturalness rather than raw TTS fidelity.
- **Tool/function calling**: Not applicable at the CSM-1B level; this is a generation component, not an agent framework.
- **Major architectural innovation**: Demonstrating that a **small (1B), openly released** two-transformer design built on a reused open codec (Mimi) could get expressive, context-aware conversational speech generation into the hands of the open-source community, becoming a widely used baseline for academic comparison in 2025–2026.
- **Limitations**: The publicly released weights are explicitly the "Tiny" version; Sesame's more capable production model (powering their consumer product) remains proprietary; CSM-1B by itself isn't a complete conversational agent.

---

## 4.6 Alibaba — Qwen3-Omni / Qwen3.5-Omni

- **Architecture**: The **"Thinker–Talker"** pattern, now in its third generation (following Qwen2.5-Omni). The **Thinker** is a Mixture-of-Experts transformer that receives audio (via a lightweight continuous encoder called **AuT**, 12.5Hz frame rate) and vision input, reasons, and produces text. The **Talker** is a second autoregressive MoE transformer that generates streaming multi-codebook speech tokens; in Qwen3-Omni the Talker was deliberately changed to condition **only on audio/visual features** (not on the Thinker's text hidden states), reasoning that discrete text tokens and their embeddings are close to informationally equivalent, while direct multimodal conditioning is needed to preserve things like translated prosody/timbre. Qwen3.5-Omni adds a **Hybrid MoE backbone** and an **MTP (multi-token prediction) module** that outputs residual codebook predictions per frame, feeding a "Code2Wav" renderer that streams waveform frame-by-frame.
- **Native vs. cascaded**: Native, single-pass — audio/video/text are interleaved as input with explicit timestamps for temporal alignment, and both text and speech are produced together.
- **I/O modalities**: Text, image, audio, video in; text and speech out. Qwen3.5-Omni supports up to 256K token context, 10 hours of audio, or 400 seconds of 720p video at 1 FPS in a single context.
- **Audio representation**: RVQ-based multi-codebook speech tokens (introduced in Qwen3-Omni, reused in 3.5) chosen specifically to improve inference efficiency.
- **Streaming architecture**: Chunk-wise streaming input processing in the Thinker, paired with a streaming Talker design for genuinely low-latency end-to-end conversation.
- **Latency characteristics**: Independent third-party benchmarking (WavBench) reports the Qwen3-Omni-30B-A3B checkpoint achieving **~234ms end-to-end latency**.
- **Context handling**: Among the largest documented context windows in this survey (256K tokens / 10 hours of audio).
- **Multilingual**: Qwen3-Omni: 119 text languages, 19 speech input languages, 10 speech output languages — explicitly one of the broadest multilingual footprints of any model surveyed here.
- **Voice generation**: Multi-codebook streaming synthesis via the Talker + Code2Wav renderer.
- **Emotion/prosody**: Explicitly cited as a reason for the Qwen3-Omni Talker's redesign — direct multimodal conditioning (rather than text-only conditioning) was needed specifically to preserve prosody/timbre in tasks like audio-video-coordinated speech translation.
- **Tool/function calling**: Supported as part of the Thinker's standard LLM capabilities.
- **Major architectural innovation**: The clearest, most explicit public rationale in this survey for *why* a "Talker" should sometimes skip the "Thinker's" text representation and condition on raw audio/video features directly — a concrete data point on the "does text still exist and how much should downstream generation rely on it" architectural question this research explicitly asked about.
- **Limitations**: MoE architecture means real hardware requirements (Qwen3.5-Omni's 30B-parameter/3B-active model needs a high-memory GPU, e.g., a single 80GB datacenter GPU at FP8 for reasonable throughput); as an open-weight model it requires self-hosting expertise that commercial APIs abstract away.

---

## 4.7 Other notable systems (documented at lower depth)

| Model | Org | Notable architectural point |
|---|---|---|
| **MiniCPM-o 4.5** | OpenBMB | End-to-end omni design connecting modality encoders/decoders directly to the language backbone via hidden-state interactions (not a cascade); combines a Whisper-based audio encoder, CosyVoice2-style tokenization, and a Qwen3-8B backbone, with full-duplex streaming and "timeline modeling" for real-time interaction |
| **Kimi-Audio** | Moonshot AI | 12.5Hz tokenizer combining discrete semantic tokens *and* continuous acoustic features, decoded by a flow-matching-based streaming detokenizer — a hybrid discrete+continuous representation, distinct from Mimi's purely discrete RVQ approach |
| **Baichuan-Omni-1.5** | Baichuan | Custom 8-layer RVQ audio tokenizer trained on a large multi-stage curated dataset |
| **Ming-Lite-Omni-1.5** | Ant Group | Modality-specific encoders feeding an MoE core ("Ling") with modality-aware routers, avoiding task-specific architectural redesign per modality |
| **Freeze-Omni** | — | Streaming omni-modal speech in/out model used as a research baseline for probing single-tower vs. modular architecture tradeoffs under reasoning load |

## 4.8 Cross-model pattern summary

Three dominant architectural "schools" emerge from this survey:

1. **Fully unified, single autoregressive loop over multiple interleaved streams** (Moshi) — maximizes latency and turn-taking naturalness, sacrifices tool-calling maturity and raw reasoning power.
2. **Explicit two-stage Thinker/Talker (or backbone/decoder) split** (Qwen-Omni family, Sesame CSM) — keeps a strong, swappable-in-spirit reasoning stage separate from a speech-generation stage, which tends to preserve tool-calling/reasoning quality closer to a pure text LLM while still gaining native-audio latency and prosody benefits.
3. **Vendor-opaque unified models exposed only via API** (OpenAI Realtime, Gemini Live, Nova Sonic) — commercially mature, strong tool-calling and enterprise features, but the actual internal architecture is undisclosed, so claims about *why* they behave the way they do are necessarily inferential.
