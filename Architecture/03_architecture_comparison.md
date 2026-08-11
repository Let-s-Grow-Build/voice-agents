# 3. Architecture-Level Comparison: Traditional vs. Modern STS

## 3.1 The two architectures, side by side

```
TRADITIONAL (cascaded)                       MODERN STS (native)
─────────────────────                        ────────────────────
   Speech                                        Speech
     │                                             │
     ▼                                             ▼
   ┌───┐  ASR vendor A                    ┌─────────────────┐
   │STT│  (separately trained/served)      │  Native Audio /  │  single model, jointly
   └───┘                                   │  Speech Model    │  trained on interleaved
     │  text (lossy)                       │  (encoder +      │  audio + text tokens
     ▼                                     │   backbone +     │
   ┌───┐  LLM vendor B                     │   decoder)       │
   │LLM│                                   └─────────────────┘
   └───┘                                             │
     │  text                                          ▼
     ▼                                             Speech
   ┌───┐  TTS vendor C
   │TTS│
   └───┘
     │
     ▼
   Speech
```

## 3.2 Comparison table — every requested dimension

| Dimension | Traditional STT → LLM → TTS | Modern Native STS | Why the architecture causes this difference |
|---|---|---|---|
| **Components** | 3+ independently trained/served models (ASR, LLM, TTS), plus VAD and an orchestrator | 1 model, or 2 tightly coupled sub-models (encoder/backbone/decoder or Thinker/Talker) trained together | Native STS collapses the model boundary; the cascade keeps it because each stage was built by a different research team for a different objective (WER, next-token accuracy, MOS naturalness) |
| **Data representation** | Text string is the *only* thing that crosses stage boundaries | Discrete audio tokens (from an RVQ/neural codec) — often carrying both semantic and acoustic codebooks — co-modeled with text tokens inside the same context | The token vocabulary of a neural codec is learned to preserve acoustic reconstruction quality, not just linguistic content, so it doesn't force a lossy reduction the way a plain transcript does |
| **Information loss** | High: prosody, emphasis, overlap, non-speech sounds are discarded at the ASR step | Low-to-moderate: acoustic codebooks retain timbre/prosody; the model can also still lose fine detail through codec compression (e.g., Mimi's 1.1kbps bitrate is lossy too) | The cascade forces a *categorical* representation change (waveform → discrete words); native STS uses a *learned, information-preserving* discretization instead |
| **Latency** | Additive across stages: STT-final + LLM-first-token + TTS-first-audio + network hops (~0.5–3s+ in practice) | Roughly one model's streaming step (Moshi: 160–200ms theoretical/practical; commercial APIs target sub-500ms perceived) | No serialization boundary between independently-served models; audio in, audio out, in one loop |
| **Streaming** | Streams within a stage, but must wait for "finality" before handing off to the next stage | Streams continuously through the whole loop — input tokens and output tokens are produced frame-by-frame in the same pass | Because there's one model and one token stream, there's no need to wait for a discrete artifact ("final transcript," "complete sentence") before proceeding |
| **Turn-taking** | External VAD + silence/endpoint heuristic, invisible to the reasoning model | Either modeled inside the network as genuine full-duplex processing (Moshi's two-channel design) or handled by a tightly integrated, faster internal detector (commercial APIs, less publicly documented) | The cascade's LLM never receives raw audio, so it structurally cannot reason about pauses/pace; STS models can, if trained on full-duplex data |
| **Interruption handling (barge-in)** | Reconstructed after the fact: orchestrator detects new user speech, kills TTS playback, discards in-flight LLM output | Can be a native property of a full-duplex model (the "conversation" was always two simultaneous streams) or a fast internal cancel-and-resume | Barge-in is a first-class case in the training data/objective for genuinely full-duplex models, vs. an engineering patch on top of a turn-based cascade |
| **Context (conversational)** | Text-only memory; anything acoustic is gone forever after ASR | Can retain acoustic context across turns (e.g., "the user sounded stressed two turns ago") if the model's context window includes audio tokens, not just text | Native STS keeps the richer representation in-context, not just its lossy textual summary |
| **Prosody** | TTS synthesizes from text + optional static style tag; no grounding in how the user actually spoke | Output prosody can be conditioned on the model's internal state, which includes the input's acoustic content | The generation stage and the understanding stage share a representation space, or at least share a training objective, instead of being fully decoupled |
| **Emotion** | Requires a bolted-on emotion classifier to detect, and a bolted-on style-tag system to express | Can be modeled implicitly through the acoustic token stream (e.g., Gemini's "Affective Dialog," Nova Sonic's prosody-adaptive response) | Same reasoning as prosody — emotion is a paralinguistic signal, and native STS doesn't discard paralinguistic signals by design |
| **Multilingual conversation** | Each stage (ASR, LLM, TTS) must separately support a language, and code-switching mid-sentence is fragile across 3 independently-tuned models | Models like Nova 2 Sonic and Qwen3-Omni are trained end-to-end on multilingual audio+text, enabling smoother mid-conversation language switching (Qwen3-Omni: 19 speech input / 10 speech output languages; Nova 2 Sonic: 7 languages with "polyglot voices") | A jointly trained model can learn cross-lingual acoustic-to-semantic mappings directly, rather than relying on 3 separately-tuned language-specific components staying in sync |
| **Tool calling** | Mature, well-understood — it's just standard LLM function calling on the text stage | Increasingly supported natively (OpenAI Realtime added MCP server support; Nova Sonic supports "asynchronous tool use"; Gemini Live supports mid-dialogue function calling) but historically less mature/flexible than pure text-LLM tool calling | Tool calling was designed for text LLMs first; STS vendors have had to retrofit equivalent capability into an audio-native loop |
| **Observability / debugging** | Excellent: transcript at every hop, each stage individually loggable and testable | Weaker by default: no forced transcript checkpoint unless the model explicitly emits one (many now do, as a courtesy/requirement, e.g., OpenAI's audio transcript event) | Observability was a designed-in property of separately-served text-based stages; it has to be deliberately re-added to a unified audio-native model |
| **Determinism** | More deterministic at the text/tool-calling layer (can pin transcripts, replay exact prompts) | Less deterministic — subtle input audio variation can change output in ways a fixed transcript replay cannot reproduce | Once reasoning happens over raw acoustic detail, tiny changes in the actual audio (background noise, mic gain) can influence output in ways invisible to a transcript-based test harness |
| **Customization** | Mix-and-match best-in-class STT/LLM/TTS vendors per stage | Mostly locked to whatever the single model vendor provides for that model; less granular control | Modularity is precisely what a unified model gives up in exchange for lower latency and richer signal |
| **Cost** | Sum of 3 separate vendor bills, but each is individually optimizable/cheap at scale | Single vendor bill, often priced per audio token (e.g., OpenAI's `gpt-realtime` at $32/1M audio input tokens, $64/1M audio output tokens; Gemini Live Flash around $3/1M input, $12/1M output) — can be cheaper or pricier depending on call volume and provider | Pricing reflects whichever component structure the vendor chose to build; not a fixed architectural law, but native STS providers increasingly compete aggressively on audio-token price |
| **Production complexity** | High operational surface area (3+ vendors, multiple SLAs, an orchestration framework) but each piece is individually well-understood | Lower integration complexity (fewer moving parts) but a new, less mature set of production concerns (session length limits, reconnection logic, fewer fallback options) | Fewer components mechanically means fewer integration points, but "fewer components" also means less redundancy if that one component has an outage |
| **Vendor lock-in** | Low — any STT/LLM/TTS combination can be swapped independently | High — the entire conversational behavior lives inside one proprietary model | A cascade's modularity *is* portability; a unified model's tight coupling *is* the source of both its latency win and its lock-in cost |

## 3.3 The single sentence that explains almost every row above

**In the cascade, the interface between "understanding" and "generating" is a lossy, discrete, human-readable string that has to be produced in full (or in stable partial form) before the next stage can act on it. In native STS, the interface is a learned, information-dense token stream that the same model (or tightly coupled pair of models) both consumes and produces in one continuous pass.** Nearly every difference in the table above — latency, prosody preservation, turn-taking, streaming behavior, and even the tradeoffs in observability and lock-in — is a direct consequence of that one structural choice.
