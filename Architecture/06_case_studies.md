# 6. Case Studies: Where STS Provides a Meaningful Improvement

Each case study shows the existing cascaded architecture, the STS (or hybrid) alternative, and an honest accounting of what improves versus what potentially gets worse.

---

## 6.1 Case Study: Customer Support Voice Agent

**Existing architecture (cascaded)**
```
Caller audio → Streaming STT → Intent/LLM agent (with tools: CRM lookup,
order status, ticket creation) → Streaming TTS → Caller
```
Turn-taking via VAD/endpointing; barge-in handled by the orchestrator killing TTS playback on detected user speech; transcript logged at every stage for QA and compliance.

**STS / hybrid alternative**
```
Caller audio → Native STS model (conversational surface: listening, backchanneling,
tone-matching, barge-in) ⇄ External deterministic agent layer (CRM lookup, order
status, ticket creation, policy enforcement) → Caller
```
The STS model handles the *feel* of the call — natural pacing, sounding calm with an angry caller (the exact behavior Amazon highlights for Nova Sonic), fast and graceful interruption handling — while tool calls for account lookups, refunds, or ticket creation are routed through a deterministic external system rather than trusted entirely to the model's judgment.

**What improves**
- Emotional attunement: the model can genuinely detect a frustrated tone and adjust its delivery, rather than relying on sentiment analysis bolted onto a transcript.
- Interruption handling feels natural instead of jarring — callers can jump in mid-sentence ("no, I said the OTHER order") without a visible stall.
- Lower perceived latency reduces the "am I talking to a robot" cue that comes from awkward pauses.

**What potentially gets worse**
- Auditability: a support call is a regulated interaction in many industries (financial services, healthcare); losing a clean, forced transcript checkpoint at the tool-calling boundary makes after-the-fact review and dispute resolution harder unless the STS model's transcript stream is explicitly captured and retained.
- Deterministic policy enforcement (e.g., "never quote a refund amount without manager approval") is safer left in an external, testable system rather than fully delegated to a generative model's judgment — this is precisely why the hybrid split (STS for conversation, deterministic layer for actions) is recommended rather than full replacement.

---

## 6.2 Case Study: Real-Time Speech Translation

**Existing architecture (cascaded)**
```
Speaker A audio (language 1) → STT → Text (lang 1) → MT (machine translation)
→ Text (lang 2) → TTS (lang 2) → Speaker B
```
Each stage independently optimized; prosody and speaker identity are entirely lost — the translated voice sounds nothing like the original speaker and carries none of their emphasis or emotion.

**STS / native alternative**
Google's Live Speech Translation (rolling out in Google Translate) is the clearest documented example: continuous listening and two-way real-time translation with **style transfer that preserves the original speaker's intonation**, covering 70+ languages and roughly 2,000 language pairs.

**What improves**
- Prosody and emotional tone carry across the language barrier — sarcasm, urgency, and warmth are far more likely to survive translation when the model isn't collapsing everything to a flat text intermediate.
- Latency: translation via a native pipeline can be closer to true simultaneous interpretation rather than the visibly sequential "wait, translate, then speak" cascade pattern.
- Speaker style transfer means the translated voice can plausibly resemble the original speaker's delivery, which matters enormously for perceived naturalness in live conversation.

**What potentially gets worse**
- Translation accuracy auditability: text-based MT pipelines are easy to spot-check ("here's exactly what was translated, word for word"); a native audio-to-audio translation path makes it harder to produce a clean, reviewable intermediate transcript for quality assurance or dispute resolution.
- Domain-specific terminology control (e.g., precise legal or medical terms) is typically easier to enforce with a text-based MT stage where you can inject glossaries/constraints deterministically; a fully native audio path may be harder to steer with hard terminology constraints.

---

## 6.3 Case Study: Voice Assistant (device-embedded, ambient)

**Existing architecture (cascaded)**
```
Wake-word detector → STT → LLM → TTS → Speaker
```
Every utterance after the wake word is assumed to be directed at the assistant; conversations are strictly turn-based; interruptions require the user to wait for the assistant to finish or explicitly re-trigger the wake word.

**STS / native alternative**
Gemini's native-audio models' **"Proactive Audio"** capability is the clearest documented example of an architectural shift specific to this use case: the model continuously listens and reasons about whether a given utterance was actually directed at the device at all, responding only to relevant, device-directed queries and filtering out ambient conversation, rather than requiring a hard wake-word gate for every turn.

**What improves**
- More natural, wake-word-light (or wake-word-free) interaction — closer to how a person in the room would selectively respond only to things addressed to them.
- Faster follow-up turns: because there's no need to re-trigger a wake word for a natural follow-up question, multi-turn ambient conversation feels much more fluid.
- Backchannel/barge-in support (interjecting "wait, no" mid-response) is far more natural in a full-duplex or near-full-duplex native model than in a strict cascade.

**What potentially gets worse**
- Privacy and consent: an always-listening, "decides for itself what's relevant" architecture raises materially different privacy considerations than a hard wake-word gate — this is a genuine tradeoff, not just an engineering detail, and needs explicit user-facing disclosure and control.
- False positives (responding to something not actually meant for it) are a new failure mode that a strict wake-word cascade structurally cannot have.
- On-device compute/power budget: a continuously-processing native audio model is a heavier ambient workload than a lightweight wake-word detector gating an otherwise-idle pipeline — most current native-audio proactive-listening deployments run in the cloud rather than fully on-device, which itself raises bandwidth/privacy/offline-availability questions.

---

## 6.4 Case Study: Enterprise Voice Agent (multi-step workflows, e.g., mortgage/loan origination, insurance claims)

**Existing architecture (cascaded)**
```
Caller audio → Streaming STT → Agent orchestration layer (multi-step workflow state
machine, document collection, compliance scripting, authentication) → Streaming TTS
```
Heavy reliance on deterministic conversation flows, exact disclaimer scripts read word-for-word, and strict authentication gates before sensitive actions.

**STS / hybrid alternative**
Real production examples cited by vendors: United Wholesale Mortgage's "Mia" system (built on Gemini's native-audio model via Vertex AI) and Newo.ai's AI receptionists both report using native-audio models specifically for improved instruction-following and natural, emotionally intelligent conversation — while presumably keeping loan-workflow logic, document verification, and compliance scripting in deterministic external systems (not detailed in public materials, but consistent with how these platforms are typically architected).

**What improves**
- Instruction adherence: vendors report meaningfully higher instruction-following rates (e.g., Google cites a jump to ~90% instruction adherence for their native-audio model, up from ~84%) which directly matters for reading exact disclaimer language or following precise compliance scripts.
- Multilingual, code-switching support helps enterprise agents serve diverse customer bases without maintaining separate language-specific STT/TTS pipelines.
- Natural handling of the messy, real parts of enterprise calls — a customer trailing off, correcting themselves, or interrupting to ask a clarifying question mid-explanation.

**What potentially gets worse**
- The entire "most important part" of this research (see file 07) applies most acutely here: enterprise workflows genuinely need deterministic state machines, authentication gates, and auditable logs for compliance and legal reasons — none of which should be delegated to a generative model's judgment, regardless of how good its instruction-following benchmark is. The safe pattern documented across every production voice-infra source reviewed here is **hybrid**: native STS or a well-tuned cascade for the conversational surface, deterministic external systems for everything with legal/financial consequences.
- Session-length limits on some native-audio APIs (e.g., Gemini Live's 10-minute default session cap) require explicit reconnection engineering for the longer calls common in enterprise workflows like mortgage origination.

---

## 6.5 Cross-case-study pattern

In every case study, the improvement from STS clusters around the same axis: **the quality of the conversational surface** (naturalness, emotional attunement, turn-taking, interruption handling, multilingual fluency). The risk from STS clusters around the same axis in every case too: **loss of deterministic control and auditability** at exactly the points (tool calls, compliance scripts, translation of precise terminology, authentication gates) where an enterprise system cannot tolerate probabilistic behavior. This pattern is the direct motivation for the hybrid architecture recommended in files 07–10.
