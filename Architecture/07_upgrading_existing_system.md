# 7. How to Upgrade an Existing Production Voice System

**Assumed starting point**: `Audio → Streaming STT → LLM/Agent → Streaming TTS → Audio`, already in production.

Four approaches, from least to most disruptive.

---

## Approach 1 — Keep STT → LLM → TTS, Optimize It

**What stays the same**: the whole architecture. **What changes**: component choice and engineering discipline.

Concrete optimizations, grounded in the latency research in file 05:
- **Swap in faster streaming components**: modern streaming STT (sub-200ms partials), fast small/medium LLMs for the conversational path (reserving larger models for complex reasoning, routed selectively), and ultra-low-latency TTS (sub-100ms time-to-first-audio).
- **Replace plain-VAD endpointing with a semantic turn-detector** (a small model that predicts whether an utterance is *semantically* complete, not just acoustically silent) — this alone can meaningfully cut both false-interruption rate and average endpointing delay.
- **Stream token-by-token into TTS** rather than waiting for full sentences from the LLM, using clause-level chunking to start synthesis earlier.
- **Co-locate services** (STT, LLM, TTS in the same region/data-center-adjacent network) to cut network-hop latency, which is often a bigger, more controllable win than any single model swap.
- **Speculative/partial-transcript processing**: begin LLM inference on likely-stable partial transcripts, with a fast correction/rollback path if the STT's final transcript differs materially.
- **Retain a lightweight prosody/sentiment signal** as a bolted-on side-channel (a small emotion classifier running in parallel with STT) to at least partially recover the paralinguistic signal the cascade structurally discards, feeding it to the LLM as an auxiliary system message rather than trying to encode it in the transcript itself.

**When this is the right call**: when the system's core value is deterministic, auditable, tool-heavy interaction (compliance-critical workflows, regulated industries) where the architecture's transcript-based observability and vendor swappability are worth more than shaving the last few hundred milliseconds or gaining emotional nuance. This remains the *default* recommendation from 2026 voice-infrastructure guidance for the majority of production teams.

**Ceiling**: even a maximally optimized cascade cannot recover paralinguistic information lost at the ASR step, cannot achieve genuine full-duplex conversational overlap, and will always carry at least the latency of one more network hop than a native model.

---

## Approach 2 — Hybrid Architecture (STS for conversation, external components for everything else)

**The core idea**: use a native STS model specifically for the real-time conversational surface (listening, speaking, turn-taking, interruption, tone-matching) while keeping agent orchestration, tools, business logic, memory, safety, authentication, deterministic workflows, and observability **outside** the model, in systems you fully control.

```
                 ┌────────────────────────────┐
   Caller audio ─▶  Native STS Model            │◀── conversational surface:
                 │  (listens, speaks, handles   │    natural turn-taking,
                 │   turn-taking & barge-in,    │    barge-in, prosody,
                 │   tone-adaptive delivery)    │    emotional attunement
                 └──────────────┬─────────────┘
                                │ tool-call events / structured intents
                                ▼
                 ┌────────────────────────────┐
                 │  External Agent /            │◀── deterministic:
                 │  Orchestration Layer         │    business logic, tools,
                 │  (business logic, tools,     │    memory, auth, safety,
                 │   memory, auth, safety,      │    audit logging
                 │   deterministic workflows)   │
                 └────────────────────────────┘
```

**How this works in practice**: nearly every commercial STS API surveyed here (OpenAI Realtime, Gemini Live, Nova Sonic) already ships a function/tool-calling event mechanism — the model emits a structured "call this function" event mid-conversation, which the developer's own backend intercepts, executes deterministically (with whatever validation, authentication, and audit logging the business requires), and returns a result that the model incorporates back into the spoken response. This is precisely the seam where hybrid architecture lives: the STS model never *itself* executes the sensitive action; it only requests it.

**What improves over Approach 1**: genuine gains in naturalness, latency, and emotional responsiveness, without giving up deterministic control over anything consequential.

**What you give up relative to Approach 1**: some vendor flexibility (the conversational surface is now tied to one STS provider), and a bit of the pure text-LLM tool-calling maturity (audio-native tool-calling is newer and, per file 04, still catching up in some vendors' implementations — though it's improved rapidly, e.g., OpenAI's addition of MCP server support and Nova 2 Sonic's asynchronous tool use).

**When this is the right call**: this is the researched recommendation for most production enterprise systems that want the naturalness benefits of STS without inheriting its determinism/auditability weaknesses. See file 08 for the full production architecture built around this pattern.

---

## Approach 3 — Replace STT + TTS with Native STS (keep external LLM/agent as much as possible)

**The idea**: use a native STS model as a *drop-in* replacement specifically for the STT and TTS legs, while trying to preserve as much of the existing text-based agent/orchestration logic as possible — effectively treating the STS model as "STT and TTS, fused, with better prosody" rather than as the full reasoning engine.

**Resulting architecture**:
```
Caller audio → Native STS model (used primarily for its transcription + speech-generation
capability, e.g., OpenAI's parallel audio-transcript event, or Nova Sonic's "real-time text
transcription without requiring a separate model") ⇄ Existing external LLM/agent system
(reasoning, tools, memory) → STS model generates the spoken response
```

**Tradeoffs**:
- You get the latency and prosody benefits of a fused encoder/decoder without having to migrate your entire reasoning stack.
- But you're paying for a full STS model's audio-token pricing (typically far more expensive per unit than dedicated STT/TTS APIs) to get what is, in this configuration, mostly just STT+TTS functionality — you're not using the model's native reasoning strength, so the cost/benefit math needs to be checked carefully against Approach 1's optimized cascade.
- You also lose some of the *emotional* upside relative to Approach 2, because the external LLM never receives the STS model's rich internal acoustic representation — it only receives the transcript the STS model surfaces, which reintroduces some of the same information-loss boundary the cascade had, just with one fewer network hop.

**When this is the right call**: a narrow niche — teams with heavy existing investment in a text-based agent stack who want a fast latency and prosody win on the transcription/synthesis legs specifically, without a full architectural rewrite, and who are willing to pay STS-level pricing for what is functionally an STT+TTS upgrade. In most cases Approach 2 delivers more value for similar migration effort, because it actually uses the STS model's native reasoning-adjacent strengths (tone, interruption handling) rather than discarding them.

---

## Approach 4 — STS + Deterministic Agent Architecture (the recommended production pattern)

This is Approach 2, made explicit and formalized as a design principle rather than just an integration pattern — worth calling out separately because it's the core recommendation of this research (see files 08–10 for the full production blueprint).

**Design principle**: *the STS model owns the conversation; it never owns the consequences.*

- **STS handles**: turn-taking, interruption/barge-in, prosody and emotional responsiveness, multilingual fluency, natural backchanneling, deciding *what to say next* conversationally.
- **External deterministic systems handle**: *what is actually true* (retrieval/RAG over verified data), *what is actually allowed* (authentication, authorization, policy/guardrail enforcement), *what actually happens* (executing a refund, booking an appointment, updating a record), and *what gets remembered/logged* (durable memory, compliance-grade transcripts and audit trails).

**Why this separation matters for production/enterprise systems specifically**:
1. **Auditability and compliance**: regulated industries need a reconstructable, reviewable record of exactly what was said and what actions were taken and why. A purely generative model making both conversational *and* consequential decisions makes that record harder to trust and harder to produce deterministically on demand.
2. **Testability**: deterministic systems can be unit-tested, staged, and rolled back with standard software engineering practices. A model's judgment about whether to issue a refund cannot be tested with the same rigor as an explicit business-rule engine that says "refunds under $50 without manager approval are allowed if X, Y, Z."
3. **Safety and liability**: keeping consequential actions behind an explicit, developer-controlled execution boundary (the model *requests*, the system *decides and executes*) bounds the model's ability to cause real-world harm from a hallucination or an adversarial prompt injection delivered via voice.
4. **Vendor risk management**: if the conversational model is swapped (new vendor, new version, fine-tune) the business logic, data, and compliance posture of the system don't have to be re-validated from scratch — only the conversational layer's behavior needs re-testing.
5. **Graceful degradation**: with business logic external and stateless with respect to the specific STS model in use, a fallback to a cascaded STT→LLM→TTS system (see file 08's fallback design) becomes tractable, because the "brains" of the operation (memory, tools, policy) were never inside the conversational model to begin with.

**This is the architecture recommended for the "Most Important Part" of the original brief**, and it's the direct answer to "how can we improve this existing architecture using modern STS models": **don't ask STS to become the agent. Ask STS to become the mouth and ears, and keep the brain external, deterministic, and testable.**
