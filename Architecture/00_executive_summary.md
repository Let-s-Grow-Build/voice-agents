# Executive Summary — Speech-to-Speech Architecture Deep Dive (2026)

## The one-paragraph version

The traditional voice-agent stack — **Speech → STT → LLM → TTS → Speech** — is a chain of three independently-trained models glued together by text. It works, but text is a narrow pipe: it throws away tone, emotion, pauses, overlap, and timing, and every hop across the chain adds latency and a synchronization point that has to be engineered (turn-taking, barge-in, buffering). Modern **Speech-to-Speech (STS) / native-audio models** (OpenAI's `gpt-realtime`, Google's Gemini Live native-audio models, Amazon Nova Sonic, Kyutai's Moshi, Alibaba's Qwen3.5-Omni, Sesame's CSM, and others) replace some or all of that chain with a single model that consumes audio (or audio+text) tokens and emits audio (and often text) tokens directly, in one continuous stream. The fundamental architectural shift is **from "text as the universal interface between speech understanding and speech generation" to "audio tokens (and often still text tokens, but co-modeled) as the native interface."** Text usually doesn't disappear — it moves *inside* the model as a parallel or interleaved stream rather than existing as a hard boundary between separate services.

## What actually changes, architecturally

| Traditional pipeline | Modern STS |
|---|---|
| 3 separately trained models (ASR, LLM, TTS) stitched by an orchestrator | 1 model (or 2 tightly coupled sub-models: a "thinker"/backbone + a "talker"/decoder) trained/tuned jointly on interleaved audio+text |
| Interface between stages = **text string** | Interface between/inside stages = **discrete audio tokens** (from a neural codec) often time-aligned with text tokens |
| Turn-taking = external VAD + endpointing heuristic | Turn-taking is frequently modeled **inside** the network (full-duplex multi-stream processing, or a learned turn-detector) |
| Latency = sum of ASR-final + LLM-first-token + TTS-first-audio + network hops | Latency = mostly a single model's streaming forward pass, often sub-300ms theoretical |
| Prosody/emotion/tone are **lost** at the STT step (text has no pitch, no emphasis) | Prosody/emotion/tone are **preserved** because the model never fully collapses to text-only |
| Debuggable: every stage produces an inspectable transcript | Harder to debug: no forced transcript checkpoint unless the model is explicitly trained to also emit one |
| Easy to swap vendors per stage | Tighter vendor lock-in to a single foundation-model provider |

## Key finding: text does not disappear, it gets demoted

Across nearly every serious current STS system researched here — OpenAI's Realtime models, Google's Gemini Live native-audio models, Amazon Nova Sonic, Kyutai's Moshi, Sesame's CSM, and Alibaba's Qwen-Omni family — **text tokens are still generated internally**, either as:
- a parallel "monologue" stream time-aligned with audio (Moshi's "Inner Monologue"),
- a "Thinker" stage whose hidden representations condition a separate "Talker" stage (Qwen-Omni's Thinker–Talker design), or
- an internal reasoning/transcript channel exposed via the API for observability (OpenAI Realtime's `response.audio_transcript`, Nova Sonic's parallel text output).

This is an important nuance for the "does STS avoid text?" question: **the hard, brittle serialization boundary between ASR output and LLM input is what's eliminated — not the concept of text itself.** Text becomes a co-trained, time-aligned signal rather than a lossy chokepoint.

## Key finding on latency

Cascaded pipelines in mid-2026 production deployments can realistically hit ~500ms–1.5s response latency when every component (streaming STT, LLM first token, streaming TTS first audio) is well-tuned; native STS models publish theoretical latencies in the 160–380ms range (Moshi: 160ms theoretical/200ms practical; Sesame CSM: ~380ms average; OpenAI and Google's realtime models target sub-500ms perceived latency). The gap is real but has narrowed considerably as cascaded-pipeline components (Deepgram, Cartesia, ElevenLabs Flash) have also gotten much faster — meaning the *latency argument alone* is no longer a slam-dunk reason to fully replace a well-tuned cascade.

## Key finding on production reality

Independent 2026 engineering guidance (LiveKit, Pipecat, and voice-infra vendors) converges on a pragmatic default: **default to cascaded STT→LLM→TTS, and reach for native speech-to-speech only where naturalness/emotional expressiveness is the actual product differentiator** — because cascades keep the transcript-based observability, determinism, tool-calling maturity, and swappability that enterprises need, while modern STT/TTS components have closed much of the latency gap.

## Bottom-line recommendation (detailed in file 10)

For a production enterprise voice agent, the research supports a **hybrid architecture**: native STS (or a near-native model) handles the *conversational surface* — listening, turn-taking, barge-in, prosody, emotional responsiveness — while a **deterministic external orchestration layer** continues to own tools, business logic, memory, authentication, safety policy, and auditable logging. Full replacement of STT→LLM→TTS is justified only for narrow, low-stakes, highly conversational use cases (companion apps, casual assistants, language tutors) where determinism and auditability matter less than naturalness.

## How to use these files

| File | Contents |
|---|---|
| `01_traditional_architecture_and_limitations.md` | Deep dive on STT→LLM→TTS and its failure modes |
| `02_modern_sts_deep_dive.md` | What's actually inside an STS model: tokens, codecs, reasoning location |
| `03_architecture_comparison.md` | Side-by-side comparison tables across every dimension requested |
| `04_current_sts_models_2026.md` | Model-by-model breakdown with architectural innovations and sources |
| `05_latency_analysis.md` | Latency budget breakdown, cascaded vs. native |
| `06_case_studies.md` | Customer support, translation, voice assistant, enterprise agent |
| `07_upgrading_existing_system.md` | The four upgrade approaches for an existing production system |
| `08_production_architecture.md` | Recommended 2026 production architecture diagram + component table |
| `09_tradeoffs_and_evaluation_framework.md` | What you gain vs. lose, and how to decide |
| `10_final_recommendation_and_future.md` | Final recommendation + where the field is heading |

**Epistemic note:** claims about publicly documented APIs, published papers (Moshi/Mimi, Qwen-Omni, Sesame CSM), and vendor announcements are marked as verified facts with their source named inline. Claims about internal, non-disclosed implementation details of closed models (e.g., exact GPT-Realtime internals) are explicitly flagged as **inference** or **speculation**, since OpenAI, Google, and Amazon have not published full architecture papers for their production STS models the way Kyutai, Sesame, and Alibaba have for theirs.
