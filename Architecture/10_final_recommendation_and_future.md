# 10. Final Recommendation and the Future of Voice-Agent Architecture

## 10.1 Direct answer to the central question

**Should a production voice-agent system completely replace STT → LLM → TTS with STS, or should STS become another layer/component within the existing architecture?**

Based on everything in this research: **STS should become a layer within a hybrid architecture, not a full replacement of the system's reasoning and control plane — for the majority of production enterprise use cases.** The strongest, most consistently supported finding across every model architecture, case study, and piece of 2026 production guidance reviewed here is the same one, stated in three different ways by three different sources: OpenAI/Google/Amazon's own marketing frames STS as replacing the *conversational pipeline*, not the *agent*; independent 2026 voice-infrastructure engineering guidance explicitly recommends defaulting to cascade and reaching for native STS specifically where naturalness is the product; and Amazon's own case studies (mortgage origination, AI receptionists) describe native audio models being layered into systems that clearly still depend on external business logic and workflow state.

The one exception this research surfaces clearly: **for use cases where the entire value proposition IS the conversational surface itself** — companion apps, casual voice assistants, real-time translation, language tutoring — full native STS is not just acceptable but architecturally correct, because there is no meaningful "external business logic" layer to protect in the first place.

## 10.2 Recommended architecture, restated simply

- **STS owns**: the sound of the conversation — timing, tone, interruption, emotional responsiveness, multilingual fluency.
- **External deterministic systems own**: the substance of the conversation's consequences — facts, permissions, actions, and the durable record of what happened.
- **A fallback cascade exists** for resilience, sharing the same deterministic core, so the conversational front-end can degrade gracefully without touching business logic.

This is Approach 4 from file 07, architected in detail in file 08, and justified against the tradeoffs in file 09.

## 10.3 Use-case fit summary

| Use case | Best-fit architecture | Why |
|---|---|---|
| Casual/companion voice assistant | **Fully native STS** | Naturalness is the entire product; low consequence of error; no deterministic workflow to protect |
| Real-time speech translation | **Fully native STS** | Prosody/style preservation is a first-order requirement a cascade structurally cannot meet |
| Ambient device assistant | **Native STS with proactive-listening** | Requires in-model judgment about relevance that a cascade cannot provide, but weigh privacy tradeoffs explicitly |
| General customer support | **Hybrid STS** | Needs both naturalness (customer experience) and determinism (tool calls, policy) |
| Enterprise workflow agent (finance, healthcare, insurance) | **Hybrid STS, heavily constrained**, or **optimized cascade** if compliance posture demands maximum auditability | Consequences are high-stakes; determinism and audit trail matter more than marginal latency/naturalness gains |
| High-volume, low-complexity flows (reminders, simple FAQ, IVR replacement) | **Optimized traditional cascade** | Naturalness gains are wasted on trivial interactions; cost and operational maturity favor the cascade |

## 10.4 Future of voice-agent architecture (near-term extrapolation, explicitly flagged as informed speculation beyond this point)

The following are reasonable extrapolations from the trends documented in this research, not verified facts:

1. **The Thinker–Talker / two-stage pattern looks likely to become the dominant design**, because it's the one architecture in this survey that explicitly preserves strong, swappable-in-spirit reasoning quality (closer to a pure text LLM) while still gaining native-audio latency and prosody benefits — it's a more conservative, incrementally-adoptable bet than a fully unified single-loop model like Moshi, and multiple independent labs (Alibaba, and implicitly Sesame's backbone/decoder split) have converged on structurally similar designs.

2. **Tool-calling maturity in native STS models will likely keep closing the gap with text-LLM tool calling** — this is one of the fastest-moving areas documented in this research (MCP support added to OpenAI's Realtime API within the past year, Nova 2 Sonic's asynchronous tool use, Gemini Live's ComplexFuncBench improvements), suggesting the "tool integration maturity" disadvantage in file 09 may be a temporary, not permanent, tradeoff.

3. **Observability tooling for native STS will likely mature to partially close the debuggability/auditability gap** — every major vendor now exposes some transcript stream, and it's a reasonable bet that dedicated tracing/observability tooling (analogous to what exists for text-LLM agents today) will emerge specifically for audio-native conversational models, reducing (but probably not eliminating) the structural observability disadvantage described in file 09.

4. **Full-duplex, no-explicit-turn-model architectures (Moshi's approach) may see more commercial adoption** if the open-research proof of concept (fully learned turn-taking and overlap handling) translates into products, but as of this research, no major commercial vendor has publicly confirmed shipping a Moshi-style architecture — commercial vendors have instead favored managed turn-taking with strong internal barge-in handling, likely because it's easier to reason about, constrain, and productize safely.

5. **Hybrid discrete+continuous audio representations** (as seen in Kimi-Audio's combination of discrete semantic tokens with continuous acoustic features) may become more common, since they could offer a way to retain fine acoustic detail (continuous) while keeping the efficiency and modeling convenience of discrete tokens for the semantic/reasoning path — this is a genuinely open research question, not a settled trend.

6. **Regulatory and compliance frameworks will likely need to catch up specifically to native-audio conversational agents** — existing compliance patterns are built around transcript-based auditing of text-based systems; as native STS adoption grows in regulated industries, expect increased pressure (and eventually explicit guidance) around mandatory transcript capture, audio retention policies, and disclosure requirements (the EU AI Act's Article 50 AI-interaction-disclosure requirement, effective August 2026, is an early, concrete example of regulation already catching up to this category).

## 10.5 Closing synthesis

The emergence of native speech-to-speech models does not obsolete the STT→LLM→TTS architecture — it **specializes** it. The traditional cascade's core strength (deterministic, auditable, modular, swappable) remains exactly as valuable as it always was for the parts of a voice system that carry real-world consequences. What's genuinely new is a class of models that can own the *conversational surface* with a fidelity the cascade could never achieve, because they were never forced to compress speech down to a bare string in the first place. The architecturally sound response to that development is not to hand STS the keys to the whole system, but to **redraw the boundary**: let STS be the ears and the voice, and keep a deterministic, auditable system as the brain and the hands.
