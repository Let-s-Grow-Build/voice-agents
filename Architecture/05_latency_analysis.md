# 5. Latency Deep Dive

## 5.1 What "latency" actually means in a voice conversation

There isn't one latency number — there are several, and they matter differently to a caller:

| Latency type | Definition | Why it matters |
|---|---|---|
| **Time to first audio (TTFA)** | From "user stops talking" to "caller hears the first sound of a response" | The single biggest driver of perceived naturalness |
| **End-to-end latency** | From "user starts talking" to "response fully delivered" | Matters for total interaction time, less for perceived snappiness |
| **Streaming latency** | Gaps *within* a response (e.g., stutter mid-sentence because generation can't keep up with playback) | Breaks immersion even if TTFA was good |
| **Model inference latency** | Pure compute time for one forward pass/token | The floor beneath all the above |
| **Audio encode/decode latency** | Time for the codec to turn waveform → tokens → waveform | Usually small (tens of ms) but non-zero, and multiplies across a cascade |
| **Turn-taking / endpointing latency** | Time to *decide* the user has finished speaking | Directly trades off against interruption risk — decide too fast and you talk over the user; decide too slow and the conversation feels sluggish |
| **Perceived conversational latency** | The subjective "did that feel like talking to a person" number | A blend of TTFA and the smoothness of turn-taking around it |

## 5.2 Cascaded pipeline latency budget (STT → LLM → TTS)

Based on 2026 production benchmarking figures collected from voice-infrastructure vendors and engineering writeups:

```
User stops talking
      │
      ▼
 [VAD / endpoint detection]        ~200ms  (Silero VAD-only fixed threshold)
      │                            or ~220ms with a smarter semantic turn-detector
      ▼                            (which reduces false interruptions ~30% vs. plain VAD)
 [STT final transcript]            ~150–250ms
      │                            (Deepgram Nova-3 ~150ms, AssemblyAI ~200ms,
      │                             gpt-4o-transcribe ~250ms)
      ▼
 [LLM time-to-first-token]         ~200–300ms
      │                            (Gemini 2.5 Flash ~200ms, GPT-4.1-mini ~250ms,
      │                             Claude Sonnet ~300ms)
      ▼
 [TTS time-to-first-audio-chunk]   ~40–200ms
      │                            (Cartesia Sonic 4 ~40ms, ElevenLabs Flash v2.5 ~75ms,
      │                             Deepgram Aura-2 tunable to ~90ms)
      ▼
Caller hears response
```

**Rough sum for a well-tuned modern cascade**: ~600ms–1s of turn-taking latency in the optimistic case (fast VAD, fast STT, fast LLM, fast TTS, all colocated to minimize network hops). This is the figure that shifted the industry's framing over 2025–2026: a "300ms response budget... once aspirational, became the new baseline" for cascaded pipelines too, because every individual component got faster (streaming STT partials, small/fast LLMs, ultra-low-latency TTS).

**Where this budget blows up in practice**:
- Network hops between separately-hosted vendor services (each hop adds tens to low-hundreds of ms of unpredictable tail latency, worse under load)
- LLM tool calls in the middle of a turn (waiting on an external API before the model can finish its response)
- Sentence-level chunking before TTS can start (if the orchestrator waits for a complete sentence rather than streaming token-by-token into TTS)
- Regional/geographic distance between STT, LLM, and TTS providers if they're not co-located
- A poorly tuned cascade can easily land in the 3–5s range that breaks conversational feel and pushes callers toward frustration or hangup

## 5.3 Native STS latency budget

```
User stops talking (or: is still talking, in a full-duplex model)
      │
      ▼
 [Audio encoder tokenizes incoming audio]     ~10–80ms (codec algorithmic latency,
      │                                        e.g., Mimi's causal codec: ~80ms frame)
      ▼
 [Single model forward pass / streaming step]  the bulk of the remaining budget —
      │                                        one model, not three
      ▼
 [Audio decoder renders first output frame]    a few ms to tens of ms, streamed
      │                                        incrementally (Code2Wav-style renderers)
      ▼
Caller hears response
```

**Concrete published figures**:
- **Moshi**: 160ms theoretical, 200ms practical — the cleanest documented number in this survey, because the entire loop genuinely lives inside one model with a fixed codec frame size.
- **Qwen3-Omni-30B-A3B**: ~234ms end-to-end, independently benchmarked (WavBench).
- **Sesame CSM**: reported average ~380ms end-to-end, under a 500ms target, achieved partly through a computational-efficiency trick (1/16 frame sampling) that avoids the generation delays typical of naive multi-codebook RVQ decoding.
- **OpenAI `gpt-realtime`** and **Google Gemini Live native-audio models**: both vendors target/claim **sub-500ms perceived latency**, framed explicitly against the "3–5 seconds" they attribute to legacy cascaded pipelines — though exact internal timing breakdowns are not published.

## 5.4 Why native STS *can* reduce latency — the architectural reason, not just the number

1. **No cross-service network hops.** A cascade's three stages are typically three different vendors on three different networks; native STS keeps everything inside one model's forward pass on one cluster.
2. **No "finality" gate.** A cascade must wait for the ASR stage to commit to a final transcript before the LLM can safely act on it (otherwise the LLM might reason over text that a correction later invalidates). A native model has no such gate — it's continuously updating its own internal state as tokens arrive.
3. **No chunking gate for synthesis.** TTS in a cascade typically waits for at least a clause or sentence of LLM output before starting synthesis (to get natural prosody across the chunk). A native model generates audio tokens frame-by-frame as part of the *same* generation process, so there's no separate "wait, then synthesize" step.
4. **Full-duplex models remove the turn-taking decision entirely.** Moshi doesn't need to *decide* when the user is done talking, because it's always listening and always (potentially) speaking — the "decision" is an emergent property of the trained behavior, not a discrete pipeline stage with its own latency cost.

## 5.5 Why the latency gap has narrowed, not disappeared

The single most important nuance in this analysis: **the cascade's latency floor has been dropping just as fast as, or faster than, native STS's has.** Ultra-fast streaming STT (sub-200ms partials) and ultra-fast streaming TTS (sub-100ms, sometimes sub-50ms time-to-first-audio) mean a well-engineered cascade in mid-2026 is not meaningfully slower in TTFA than a native model's advertised sub-500ms figure — the *architectural* latency advantage of native STS is real and measurable at the model level, but at the *system* level, careful engineering of a cascade can close most of the gap. This is precisely why current production guidance from voice-infrastructure vendors recommends defaulting to cascade and reaching for native STS specifically when the emotional/naturalness dimension is the actual product requirement — not treating latency alone as the deciding factor.

## 5.6 Turn-taking latency specifically

This deserves separate emphasis because it's felt very differently from raw response latency:

- **Pure VAD (silence-threshold based)**: fixed ~200ms delay before deciding a turn is over — regardless of what was actually said. This causes two failure modes: cutting off a user who paused mid-thought, or waiting unnecessarily long after a clearly complete utterance ("What's the weather today?" doesn't need 200ms of silence to be obviously finished).
- **Learned/semantic turn detectors** (Pipecat's Smart Turn Detection, LiveKit's adaptive turn detection): add roughly 20ms of extra compute over plain VAD but use a small model to predict *semantic* completeness, reducing false interruptions (the agent talking over the user) by roughly 30% in reported benchmarks, and reacting faster on genuinely short, complete utterances.
- **Native full-duplex models** (Moshi-style): sidestep the discrete turn-detection step entirely, since the model is continuously processing both channels — turn-taking becomes a learned behavioral pattern rather than an explicit pipeline stage with its own latency line item.

## 5.7 Summary: where latency lives in each architecture

| Architecture | Dominant latency source | Typical mid-2026 figure |
|---|---|---|
| Naive/unoptimized cascade | Sequential sum of 3 independently-served models + network hops | 3–5s+ |
| Well-tuned cascade | Sum of fast streaming components, minimized hops | ~600ms–1s |
| Commercial native STS (OpenAI/Google/Amazon) | Single model forward pass, vendor infra | sub-500ms (vendor-claimed) |
| Open research native STS (Moshi) | Codec frame size + one model step | 160–200ms (measured) |
| Two-stage native STS (Qwen-Omni, Sesame CSM) | Thinker pass + Talker pass, but no cross-vendor hop | 234–380ms (measured) |
