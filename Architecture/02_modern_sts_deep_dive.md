# 2. What Is Modern Speech-to-Speech? A Deep Internal Architecture Dive

## 2.1 Simple version first

Instead of translating speech into words and back, a modern STS model treats speech itself as a kind of language — it breaks audio into a sequence of discrete "audio tokens," similar in spirit to how text is broken into word/subword tokens, and then runs a single (or tightly-coupled) neural network that reads those audio tokens and writes new audio tokens back out, the same way a text LLM reads text tokens and writes text tokens. A decoder then turns those output audio tokens back into a soundwave. Nothing forces the model to fully collapse what it heard into a plain sentence before it can "think" — it can keep some sense of tone and rhythm the whole way through.

## 2.2 The general shape

```
 Speech (waveform)
       │
       ▼
 ┌───────────────────┐
 │  Audio Encoder /   │   turns continuous waveform into a short sequence
 │  Neural Codec      │   of discrete "audio tokens" (~12.5–25 tokens/sec)
 └───────────────────┘
       │  audio tokens (+ often a parallel/aligned text-token stream)
       ▼
 ┌───────────────────┐
 │  Model / Reasoning │   a transformer (or MoE transformer) that has been
 │  ("Thinker" /      │   trained to jointly model audio tokens, text tokens,
 │  backbone)         │   tool calls, and dialogue state
 └───────────────────┘
       │  output tokens: audio tokens (+ often text)
       ▼
 ┌───────────────────┐
 │  Audio Decoder /   │   turns discrete audio tokens back into a waveform,
 │  Codec Decoder /   │   streaming frame-by-frame so audio can start playing
 │  "Talker"          │   before the whole response is generated
 └───────────────────┘
       │
       ▼
 Speech (waveform)
```

This is architecturally very different from `Audio → STT → Text → LLM → Text → TTS → Audio` in one crucial way: **there is no point in the pipeline where the representation is forced down to a single, lossy string and handed across a network boundary to a separately-trained model.** The audio tokens either *are* the model's native vocabulary, or they are co-modeled alongside text tokens inside one training run.

## 2.3 Component by component

### Audio encoder / neural audio codec
This is the piece that turns a continuous waveform into a short, discrete sequence a transformer can consume. The best-documented example is Kyutai's **Mimi codec**, used by both Moshi and Sesame's CSM: it compresses 24kHz audio down to a **1.1 kbps stream running at 12.5 Hz** (i.e., 12.5 discrete "frames" per second of audio, each frame holding several codebook indices), while remaining fully streaming with roughly 80ms of algorithmic latency. Mimi is trained with **Residual Vector Quantization (RVQ)**: it produces multiple stacked "codebooks" per frame — the first codebook captures mostly *semantic* content (what was said), while additional codebooks capture *acoustic* detail (voice timbre, prosody, background texture). This semantic/acoustic split is what lets a single token stream carry both linguistic content and paralinguistic detail simultaneously — something a plain text transcript structurally cannot do.

Other systems use different codec designs: Qwen3-Omni's audio pathway (**AuT**) is a continuous, lightweight audio encoder at 12.5Hz feeding semantic features into the reasoning stage, paired with a separate multi-codebook discrete codec for the *output* side; Kimi-Audio uses a 12.5Hz tokenizer combining discrete semantic tokens with continuous acoustic features, decoded by a flow-matching-based streaming vocoder.

### Audio tokens — what they actually are
An "audio token" is a small integer index into a learned codebook, the audio equivalent of a subword-token ID in text. A short frame of audio (tens of milliseconds) is mapped to a handful of these indices (multiple codebooks stacked per frame, e.g., Mimi uses 8 codebooks per 80ms frame). This is fundamentally different from a phoneme or a text token: it's a *learned* discretization optimized so that the codec's decoder can reconstruct natural-sounding audio, not so that it maps cleanly onto linguistic units. That's precisely why paralinguistic information can ride along in the same stream — the tokens were never forced to be "just the words."

### Where reasoning happens
This is the most architecturally varied part across current systems, and it's worth being precise about it because implementations differ:

- **Single unified stack (Moshi):** one 7B "Helium" language-model backbone processes interleaved user-audio tokens, system-audio tokens, and a parallel system-text stream ("Inner Monologue") simultaneously, at every timestep, via a **hierarchical RQ-Transformer** — a "Temporal Transformer" that models the coarse per-timestep representation across 17 parallel streams (1 text + 8 system-audio codebooks + 8 user-audio codebooks), followed by a smaller "Depth Transformer" that expands that representation into the multiple RVQ codebook tokens for the current audio frame. Reasoning and audio generation happen in the *same* autoregressive loop, not in separate passes.
- **Two-stage "Thinker–Talker" (Qwen2.5/3/3.5-Omni):** an explicit division of labor. The **Thinker** is a standard multimodal transformer (with an audio encoder and vision encoder feeding in) that does the actual reasoning and produces text output. The **Talker** is a second autoregressive transformer that conditions on the Thinker's outputs (in Qwen2.5-Omni, on the Thinker's hidden representations directly; in Qwen3-Omni, the Talker was changed to condition only on audio/visual features rather than the Thinker's text hidden states, on the reasoning that text tokens and their embeddings are close to informationally equivalent) and streams out multi-codebook speech tokens. A lightweight causal vocoder (Code2Wav) then renders those tokens to waveform frame-by-frame.
- **Two-transformer split by function (Sesame CSM):** a Llama-architecture **backbone** transformer processes interleaved text and audio context and predicts the first (semantic) RVQ codebook token per frame; a smaller **audio decoder** transformer then autoregressively fills in the remaining acoustic codebook tokens conditioned on that first token. This is a generation-only design (not a full duplex conversational reasoning loop by itself — CSM is a contextual TTS/dialogue-generation model, not a full agent).
- **Unified single model, vendor-undisclosed internals (OpenAI's `gpt-realtime`, Google's Gemini Live native-audio models, Amazon Nova Sonic):** these are described by their vendors as single, natively multimodal models that process audio (and text/image) as first-class token types within one architecture, but the vendors have not published architecture papers with the level of detail Kyutai, Alibaba, and Sesame have. **This is inference, not verified fact**: it is reasonable to assume something structurally similar to the Thinker–Talker or unified-token approaches above, since that is the dominant published design pattern in the field, but the exact internals (codec design, codebook count, whether there's an explicit two-stage split) are not public.

### How linguistic and non-linguistic information are understood together
Because the input token stream carries semantic *and* acoustic codebooks together (or a continuous audio embedding alongside discrete text), the same attention layers that figure out *what was said* also have access to *how it was said* at every step — pitch contour, loudness, pace, and pauses are latent in the acoustic codebooks or continuous audio features, not stripped out beforehand. This is the direct architectural answer to "how does the model understand both linguistic and non-linguistic information": **it doesn't separate the two into different modules; both are present in the same token stream the reasoning layers attend over.**

### How tone, emotion, pitch, pauses, and speaking style are preserved
Preservation happens on both sides:
- **On the input side**, the acoustic-detail codebooks (or continuous audio embeddings) retain enough information for the model to condition its response on the user's affect — this is what Amazon calls "adaptive speech response that dynamically adjusts delivery based on the prosody... of input speech" in Nova Sonic, and what Google calls "Affective Dialog" in Gemini's native-audio models (understanding and responding appropriately to a user's emotional expression).
- **On the output side**, because the model generates *audio tokens directly* rather than text-then-resynthesize, it can vary pitch, pace, and emphasis as a continuous function of the conversational context, rather than being limited to whatever a downstream TTS system can infer from a static string plus an optional style tag.

### How native STS improves real-time interaction and latency
Because there's one autoregressive loop generating output tokens (rather than: wait for ASR finalization → wait for LLM generation → wait for TTS synthesis), the "critical path" collapses to a single streaming forward pass. Moshi's paper reports a **theoretical latency of 160ms and ~200ms in practice** for its full duplex loop — essentially just the codec's frame size plus one model step, because there is no cross-service serialization at all.

### How it handles interruptions and turn-taking
This is where the biggest structural divergence between models shows up:
- **True full duplex, no explicit turn model (Moshi):** the model processes a continuous *two-channel* stream — one channel of tokens for "what the user is saying," one channel for "what Moshi is saying" — at every single timestep, simultaneously, with no explicit speaker-turn state machine. Interruptions and overlapping speech are just... what the token streams look like; the model was trained on data with real interruptions and back-channels (e.g., "mm-hmm") and learned to handle them as a natural consequence of always listening while speaking.
- **Managed turn-taking with model-level signals (Gemini Live, Nova Sonic, gpt-realtime):** these systems expose explicit turn/interruption handling as part of the API contract — e.g., Gemini's native-audio models ship "improved barge-in" as a named capability, and Nova Sonic advertises "graceful handling of user interruptions without dropping conversational context" — but the vendors have not disclosed whether this is implemented via genuine full-duplex internal modeling (like Moshi) or via a tightly integrated internal VAD/turn-detector plus fast cancellation. **This distinction is inference where not explicitly documented.**

### What happens during streaming inference
The encoder turns each small chunk of incoming audio (tens of milliseconds) into tokens as it arrives; the reasoning/generation stage consumes and produces tokens frame-by-frame rather than waiting for an entire utterance; and the decoder renders each output frame's tokens to waveform incrementally, so audio can start playing before the model has "finished" generating the whole response — this is what all the case studies above mean by "frame-by-frame streaming generation" (explicitly described for Qwen3.5-Omni's Code2Wav renderer, and structurally implied by Mimi's 80ms streaming codec latency).

### What's eliminated, merged, or fundamentally changed vs. the traditional pipeline

| Traditional component | What happens to it in native STS |
|---|---|
| Standalone ASR service producing a final transcript | **Merged into the encoder + backbone.** A text-like signal may still exist internally (Moshi's Inner Monologue, Thinker's text output) but it's produced *by* the same model, time-aligned with audio, not by a separate, independently-served model whose output is the only thing downstream stages see. |
| Standalone LLM/agent service | **Becomes the backbone/"Thinker."** Tool calling and reasoning still happen here, but the model natively receives audio-derived context (and often keeps generating text alongside it) rather than only ever seeing a transcript. |
| Standalone TTS service | **Merged into the "Talker"/audio decoder.** Voice generation is conditioned on the model's own internal state (which includes acoustic context from the input) instead of a bare string. |
| External VAD + turn-taking heuristic | **Either absorbed into the model** (Moshi's full-duplex design) **or kept external but tightly coupled** (most commercial APIs still expose/require some VAD signal at the transport layer, but the model itself is more turn-aware than a plain text LLM would be). |
| Network hops between 3 vendors | **Collapsed to (usually) one vendor, one model, one streaming connection** — fewer serialization points, but also fewer places to intervene. |

## 2.4 Does STS "avoid text," or does text still exist inside the model?

Based on the published architectures reviewed here, the honest answer is: **text-like representations persist inside almost every serious STS model — they just stop being the sole, forced interchange format between independently-trained components.**

- Moshi explicitly generates a parallel **text token stream** ("Inner Monologue") alongside audio tokens, because the authors found this significantly improves the linguistic quality of the generated speech and incidentally gives you streaming ASR/TTS behavior "for free" from the same model.
- Qwen-Omni's Thinker literally **is** a text-generating model; the Talker conditions on its output. Text generation is not eliminated — it's the reasoning currency, but it's produced *inside* the same forward pass/session rather than round-tripped through a separate service.
- OpenAI's Realtime API surfaces an audio transcript alongside audio output as a first-class part of its event stream, implying some internal text (or text-aligned) representation is available even if not user-facing by default.
- Amazon Nova Sonic's documentation describes "real-time text transcription without requiring a separate model," again implying text is generated as a byproduct of the same unified model, not eliminated.

So the precise architectural claim is: **the elimination is of the hard, lossy hand-off boundary between separately trained ASR/LLM/TTS models — not of text as a concept.** Text becomes one co-modeled stream among several (alongside semantic and acoustic audio tokens) rather than the only surviving representation.

## 2.5 Where this section's claims sit on the verified-fact ↔ speculation spectrum

- **Verified / directly sourced from papers and technical reports:** Moshi/Mimi architecture and latency figures (Kyutai's Moshi paper and GitHub); Qwen2.5/3/3.5-Omni Thinker–Talker design (Alibaba's technical reports); Sesame CSM's two-transformer design (Sesame's Hugging Face model card and GitHub); Amazon Nova Sonic's unified-model claims and named capabilities (AWS documentation and technical report).
- **Reasonable inference:** that OpenAI's and Google's closed models likely follow a broadly similar unified-token or Thinker–Talker-style pattern, since that is the dominant architecture in the published literature — but their exact internal codec/token design is not public.
- **Speculation flagged explicitly:** any claim about exact parameter counts, codebook counts, or training data for closed commercial STS models beyond what vendors have disclosed.
