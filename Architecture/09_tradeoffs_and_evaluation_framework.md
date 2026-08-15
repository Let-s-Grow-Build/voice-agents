# 9. Tradeoffs and Evaluation Framework

STS is not automatically better. This file is deliberately structured to argue *against* uncritical adoption, then provide a framework for deciding.

## 9.1 What STS genuinely improves

| Gain | Evidence from this research |
|---|---|
| Latency | Moshi: 160–200ms; Qwen3-Omni: ~234ms; Sesame CSM: ~380ms — all meaningfully below a naive cascade's 1–5s, though a *well-tuned* cascade closes much of this gap (file 05) |
| Prosody / emotional expressiveness | Native preservation of acoustic detail via semantic+acoustic RVQ codebooks (Mimi); named features like Gemini's Affective Dialog and Amazon Nova Sonic's prosody-adaptive response |
| Turn-taking / interruption naturalness | Genuine full-duplex modeling (Moshi) or tightly-integrated fast barge-in (commercial APIs) vs. externally reconstructed barge-in in a cascade |
| Multilingual fluency / code-switching | Jointly trained models handle mid-sentence language switching more gracefully than 3 separately-tuned per-language components (Nova 2 Sonic, Qwen3-Omni, Gemini Live) |
| System simplicity (fewer moving parts) | 1 model/vendor instead of 3+ independently served components and their orchestration |
| Instruction adherence in some benchmarks | Google reports ~90% instruction adherence for their native-audio model, up from ~84% in a prior version |

## 9.2 What we potentially lose

### Debuggability
A cascade gives you a forced, inspectable checkpoint after every stage — you can look at exactly what the ASR heard, exactly what the LLM reasoned, and exactly what text went to TTS. A unified STS model collapses these into one opaque forward pass; unless the vendor explicitly surfaces an internal transcript (most now do, as a courtesy), there is no equivalent checkpoint, and even when a transcript is surfaced, it doesn't tell you *why* the model chose a particular tone or timing.

### Observability
Related but distinct from debuggability: production observability requires metrics *per stage* to know where a regression is coming from. In a cascade, you can independently monitor STT word-error-rate, LLM latency/quality, and TTS naturalness. In a unified model, a quality regression is a single opaque signal — you often can't tell whether a bad response came from misunderstanding the audio, poor reasoning, or poor speech generation, because those are no longer separable measurement points.

### Determinism
A cascade's text stage can, in principle, be made close to deterministic (fixed prompts, fixed transcripts, replayable test suites). A native STS model reasoning directly over raw acoustic detail means that two calls with the same *words* but slightly different background noise, mic gain, or speaking pace can produce different responses — this is a fundamental consequence of using a richer, less normalized input representation, and it makes building a stable regression test suite for conversational behavior meaningfully harder.

### Transcript-based auditing
Regulated industries (financial services, healthcare, insurance) frequently require exact, reviewable transcripts of what was said and what was promised. This is achievable with STS (most vendors expose a transcript event stream), but it requires deliberate engineering to capture and retain it — it's not a free byproduct of the architecture the way it is in a cascade, where the transcript *is* the interstage interface and therefore automatically exists.

### Safety
Delegating more of the interaction (tone, timing, and potentially more of the substantive content) to a single generative model increases the surface area for the model to produce an unsafe or non-compliant utterance without an intermediate checkpoint to catch it. This is precisely why Approach 4 (file 07) insists on keeping consequential actions in a deterministic external layer — but the *conversational content itself* is still generated end-to-end by the model, and safety review of that content is architecturally harder without a discrete transcript-generation stage to apply guardrails to before speech is produced.

### Cost
As discussed in file 08.5, audio-token pricing for native STS is often substantially higher per unit than the equivalent text-token pricing in a cascade's LLM stage, while eliminating separate STT/TTS costs. Net cost depends heavily on usage pattern and must be modeled, not assumed.

### Vendor lock-in
A cascade's modularity is portability: swap STT, LLM, or TTS vendors independently, at will. A native STS model concentrates the entire conversational experience — latency profile, voice quality, tool-calling behavior, safety posture — inside one vendor's proprietary model. Migrating away later means re-validating the *entire* conversational experience, not swapping one component.

### Tool integration maturity
Text-LLM function calling is a mature, extensively battle-tested capability. Audio-native tool calling is newer across every vendor surveyed here — rapidly improving (OpenAI's MCP server support, Nova 2 Sonic's asynchronous tool use, Gemini Live's ComplexFuncBench Audio improvements) but with a shorter production track record than pure text-LLM tool calling.

### Customization / model control
A cascade lets you fine-tune or prompt-engineer each stage independently with fine-grained control (e.g., a custom ASR vocabulary for domain jargon, a custom TTS voice clone, a heavily fine-tuned LLM for your specific workflow). A unified STS model offers less granular control — you're generally working within whatever customization surface (system prompts, voice presets, limited fine-tuning) the vendor exposes for the whole model at once.

### Failure handling
A cascade's modularity means a single component's outage (say, one TTS vendor) can often be mitigated by failing over to a backup vendor for just that stage. A unified STS model's outage takes down the entire conversational capability at once, with no partial-degradation path unless you've built a separate fallback cascade (file 08.4) — which is precisely why that fallback is recommended as a standing part of the production architecture.

### Enterprise compliance
Beyond auditing specifically: many enterprise compliance frameworks (SOC 2, HIPAA, financial services regulation) have well-established patterns for reviewing and certifying deterministic, testable systems. A generative model making more of the moment-to-moment interaction decisions is a less mature category for these frameworks to reason about, and compliance review processes may simply take longer or require more novel justification.

## 9.3 Evaluation framework: how to decide

Ask these questions, roughly in priority order:

1. **Is any part of this conversation legally or financially consequential?**
   If yes → those parts must live in a deterministic external system regardless of which conversational architecture you choose (Approach 4, file 07). This question doesn't decide *whether* to use STS — it decides *what STS is and isn't allowed to own*.

2. **Is the actual product value proposition "feels natural to talk to," or is it "gets the task done reliably"?**
   - If naturalness/emotional connection is the product (companion apps, casual assistants, language tutors, ambient device interaction) → weight toward native STS or hybrid.
   - If reliable task completion under audit is the product (claims processing, financial transactions, regulated customer support) → weight toward an optimized cascade or a tightly-constrained hybrid with heavy external guardrails.

3. **What's your current cascade's actual latency, measured, not assumed?**
   If a well-tuned cascade is already hitting sub-1s TTFA, the latency argument for STS is weaker than it first appears (file 05.5) — the decision should hinge on prosody/naturalness/turn-taking quality instead, not raw speed.

4. **What's your tolerance for vendor lock-in and reduced observability?**
   Regulated industries and large enterprises with strict vendor-risk and audit requirements should weight this heavily against full replacement; smaller, faster-moving products may reasonably accept more lock-in for a better user experience.

5. **Can you afford (in engineering time and dollars) to run a warm fallback cascade?**
   If not, full replacement of your cascade with STS leaves you with no degradation path during an outage — a serious operational risk for any production system.

6. **Have you modeled the actual per-minute cost delta**, given your real call-volume and call-length distribution, rather than assuming STS is cheaper (fewer vendors) or more expensive (audio-token pricing) without checking?

## 9.4 Decision matrix

| Situation | Recommended architecture |
|---|---|
| Regulated, high-stakes, audit-heavy (finance, healthcare, legal) | Optimized cascade (Approach 1) or hybrid with STS strictly limited to conversational surface and heavy external guardrails (Approach 4) |
| General enterprise customer support / voice agent | Hybrid: STS for conversation, deterministic agent layer for tools/logic (Approach 2/4) |
| Consumer companion / casual assistant / language learning | Full native STS acceptable — naturalness is the product, and consequences of a misstep are low |
| Real-time translation | Native STS strongly favored — prosody/style preservation is a core value proposition that a cascade structurally cannot deliver |
| Ambient/always-listening device assistant | Native STS with proactive-listening capability (e.g., Gemini's Proactive Audio) — but weigh privacy/consent implications explicitly |
| Cost-sensitive, high call volume, simple flows (e.g., appointment reminders, basic FAQ) | Optimized cascade — usually cheaper and simpler for low-complexity interactions where naturalness matters less |
