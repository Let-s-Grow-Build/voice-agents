# 1. The Traditional Voice Architecture: STT → LLM → TTS

## 1.1 The pipeline, in simple terms

Think of the traditional pipeline as three separate people passing a note down a line, where the note can only contain *words* — never tone of voice:

```
 ┌──────────┐   ┌──────────────┐   ┌──────────┐   ┌──────────────┐   ┌──────────┐
 │  Speech  │──▶│  STT / ASR   │──▶│   Text   │──▶│   LLM Agent  │──▶│   Text   │──▶ TTS ──▶ Speech
 │ (audio)  │   │ (transcribe) │   │ (string) │   │  (reasoning) │   │ (string) │
 └──────────┘   └──────────────┘   └──────────┘   └──────────────┘   └──────────┘
```

1. **STT (Automatic Speech Recognition)** converts an audio waveform into a text transcript. Nothing else survives this step — pitch, emphasis, pauses, laughter, sighs, and overlapping speech are all discarded unless a separate model is bolted on to detect them.
2. **LLM** reasons over the transcript (plus conversation history, tool results, RAG context) and produces a text response.
3. **TTS (Text-to-Speech)** synthesizes that text response back into audio, using a voice model that has no idea what the user actually sounded like.

Each arrow in that diagram is a **network hop and a full-model inference pass**, typically served by three different vendors, three different scaling systems, and three different failure domains.

## 1.2 Why this pipeline was — and still is — a reasonable choice

- **Modularity**: each component can be swapped, fine-tuned, or A/B tested independently. Need better transcription accuracy? Swap the ASR vendor without retraining anything else.
- **Observability**: the transcript is a first-class, human-readable artifact. It's trivial to log, audit, redact, and feed to compliance tooling.
- **Maturity**: STT and TTS are decades-old problems with highly optimized, specialized models (Whisper-derived ASR, Deepgram Nova, Cartesia Sonic, ElevenLabs).
- **Reuse of text-only LLM infrastructure**: the "reasoning" stage is a plain text LLM, so all existing agent tooling (function calling, RAG, guardrails, prompt engineering) works unmodified.

This is why, per 2026 production guidance from voice-infra vendors, cascaded architecture remains the *default* recommendation for most teams — the limitations below are real, but they are increasingly engineered around rather than eliminated.

## 1.3 The limitations, one at a time

### Latency
Cascaded latency is **additive**: time-to-final-transcript (STT) + time-to-first-token (LLM) + time-to-first-audio (TTS), plus network round trips between each hop. In well-tuned mid-2026 production stacks this looks roughly like:

| Stage | Typical figure (2026, streaming, well-tuned) |
|---|---|
| STT first partial | ~150–250ms (Deepgram Nova-3 ~150ms, AssemblyAI Universal-Streaming ~200ms, gpt-4o-transcribe ~250ms) |
| End-of-turn / endpoint detection | ~200ms with VAD alone; +20ms with smarter semantic turn-detectors |
| LLM first token | ~200–300ms (GPT-4.1-mini ~250ms, Claude Sonnet ~300ms, Gemini 2.5 Flash ~200ms) |
| TTS first audio chunk | ~40–200ms (Cartesia Sonic 4 ~40ms, ElevenLabs Flash v2.5 ~75ms, Deepgram Aura-2 tunable to ~90ms) |

A naive sum puts a *reasonably tuned* cascade at roughly 600ms–1s of turn-taking latency before the caller hears anything — and a poorly tuned one can run 3–5s, which is the range that breaks the feel of conversation and pushes callers back toward touch-tone menus. Every additional network hop between separately-hosted services adds further, less predictable tail latency.

### Streaming
Cascaded systems only stream *within* a stage, not *across* stages cleanly. The ASR must decide when a chunk of audio is "final" before the LLM stage can safely start reasoning over it (otherwise the LLM might reason over a transcript that later gets corrected). This forces a **serialization point**: streaming ASR → (wait for finalization) → streaming LLM tokens → (wait for a speakable chunk, e.g., a sentence or clause) → streaming TTS. Engineering around this (e.g., feeding partial transcripts to the LLM speculatively, or streaming TTS on partial LLM output at the sentence level) is possible but adds complexity and risk of the LLM reasoning over a since-corrected transcript.

### Turn-taking
Turn-taking in the cascade is handled by a component *external* to the reasoning model: Voice Activity Detection (VAD) plus an end-of-turn heuristic (silence duration, or a learned "smart turn" classifier). This is fundamentally a guess about intent based on acoustic silence — it doesn't know if the user paused to think, finished a sentence, or is about to say "aaand one more thing." The LLM never "hears" the pause; it only sees the transcript after the external system has decided the turn is over.

### Interruptions / barge-in
Because TTS audio is being played out separately from anything that's "listening," barge-in has to be implemented as a race condition handled by the orchestration layer: detect user speech via VAD while TTS is playing, stop TTS playback, discard whatever the LLM was mid-generating, and re-enter the STT/LLM loop. This works, but it's bolted on rather than a native property of the model — the LLM itself has no innate sense that it was interrupted; the orchestrator has to reconstruct "what was actually said" versus "what was intended to be said" from separate signals.

### Context loss
The single most consequential limitation: **the text transcript is a lossy compression of speech.** Everything a text string cannot represent is gone the moment ASR finishes: emphasis ("I did NOT say that" vs. "I did not say THAT"), sarcasm, hesitation, sighing, crying, laughing, speaking rate as a signal of urgency, and who is speaking during overlapping speech. The LLM reasons entirely over what's left.

### Prosody, emotion, tone, non-verbal information
This is a direct consequence of context loss above. The reasoning model never receives paralinguistic information, so it cannot condition its response on "the customer sounds furious" versus "the customer sounds confused" — unless a separate emotion-classification model is bolted onto the pipeline as a fourth component (adding yet more latency and complexity). Downstream, the TTS model has no idea what to convey emotionally beyond what's explicit in the text or a manually specified style tag; it cannot naturally mirror the user's affect.

### Speech understanding
ASR is optimized for word-error-rate on the *content* of speech, not for capturing intent-bearing acoustic features. Accents, code-switching mid-sentence, cross-talk, and background noise all degrade the one representation (text) that the rest of the pipeline depends on — errors compound downstream with no way to recover once the transcript is wrong.

### Response generation
The LLM's response generation is, in principle, unconstrained by pipeline architecture — but in practice it is text-only, so any "response" is really "in words," and the shaping of *how* to say those words is entirely deferred to the TTS stage, which sees no context beyond the text and (optionally) a static style instruction.

### Voice generation
TTS models generate speech from text with no grounding in the actual conversational moment. Even highly expressive modern TTS (Cartesia, ElevenLabs, Deepgram Aura) is synthesizing from a string plus, at best, a coarse style/emotion tag — it never "heard" how the user said what they said, so mirroring or reactive prosody is approximate at best.

### System complexity
A production cascade is effectively a distributed system: a VAD/streaming-audio front end, an ASR service, an LLM/agent service (with its own tool-calling, memory, and guardrail layers), and a TTS service, usually from three-plus vendors, glued together by an orchestration framework (LiveKit Agents, Pipecat, or a custom loop). Each hop is a new failure domain, a new place for version skew, and a new thing to load-test and monitor.

### Scalability and production concerns
Because each stage scales independently, capacity planning has to account for the fact that a burst in call volume simultaneously stresses three separate scaling domains (STT concurrency limits, LLM token throughput, TTS synthesis throughput), each with its own vendor-specific rate limits, cold-start behavior, and regional availability. Multi-vendor cascades also multiply the number of SLAs, and a partial outage of any single stage can degrade the entire conversational experience even if the other two stages are healthy.

## 1.4 Summary table

| Dimension | Root cause of the limitation |
|---|---|
| Latency | Additive hops across independently-served models |
| Streaming | Hard serialization boundary at "final transcript" and "speakable text chunk" |
| Turn-taking | Externalized to acoustic-only VAD/heuristics, invisible to the reasoning model |
| Interruptions | Barge-in reconstructed after the fact by the orchestrator, not modeled natively |
| Context loss | Text is the only surviving representation between stages |
| Prosody/emotion | No acoustic information reaches the LLM; no conversational grounding reaches the TTS |
| Speech understanding | Errors in one lossy transcript propagate uncorrected downstream |
| System complexity | Distributed system across 3+ vendors with independent scaling and failure domains |

The rest of this research explores how native STS architectures attack these limitations directly — and where they introduce new problems of their own.
