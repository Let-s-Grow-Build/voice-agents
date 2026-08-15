# 8. Recommended 2026 Production Architecture for an Enterprise Voice Agent

## 8.1 Full architecture diagram

```
┌──────────┐     WebRTC/SIP      ┌──────────────────────────────────────────────┐
│  Client  │────────────────────▶│         Real-time Media / Transport Layer      │
│ (phone,  │◀────────────────────│  (LiveKit / Pipecat / Daily / raw WebRTC SFU;   │
│  browser,│    audio stream     │   SIP trunk for telephony numbers)              │
│  app)    │                      └───────────────────┬────────────────────────────┘
└──────────┘                                            │ audio frames
                                                          ▼
                                          ┌───────────────────────────┐
                                          │   Client-side / edge VAD   │  cheap first-pass
                                          │  (barge-in trigger, silence│  gate before the
                                          │   suppression)             │  expensive model
                                          └─────────────┬─────────────┘
                                                          │
                                                          ▼
                              ┌───────────────────────────────────────────────┐
                              │        NATIVE STS MODEL (conversation core)     │
                              │  - streaming audio in / audio+text out          │
                              │  - turn-taking, barge-in, prosody, tone-match   │
                              │  - emits: spoken audio + text transcript event  │
                              │    + structured tool-call events                │
                              └───────────────┬─────────────────┬───────────────┘
                                               │ tool-call events │ transcript stream
                                               ▼                  ▼
                     ┌─────────────────────────────────┐   ┌──────────────────────┐
                     │   Agent / Orchestration Layer     │   │  Observability /     │
                     │  (deterministic, stateless w.r.t.  │   │  Logging /           │
                     │   the STS model choice)            │   │  Transcription store │
                     │                                     │   │  (compliance-grade,  │
                     │  ┌───────────┐  ┌────────────────┐ │   │   immutable, per-call│
                     │  │  Tool      │  │  Memory /       │ │   │   audit trail)       │
                     │  │  Execution │  │  Session State  │ │   └──────────┬────────────┘
                     │  │  (CRM,     │  │  (short + long  │ │              │
                     │  │  billing,  │  │   term)         │ │              ▼
                     │  │  ticketing)│  └────────────────┘ │   ┌──────────────────────┐
                     │  └───────────┘                       │   │  Monitoring / Alerting│
                     │  ┌───────────┐  ┌────────────────┐  │   │  (latency, error rate,│
                     │  │  Safety /  │  │  Authentication │  │   │   escalation, QA      │
                     │  │  Guardrails│  │  / Authorization│  │   │   sampling)           │
                     │  └───────────┘  └────────────────┘  │   └──────────────────────┘
                     └───────────┬─────────────────────────┘
                                 │ result / structured response
                                 ▼
                    (fed back into the STS model's context to speak the outcome)


                     ┌───────────────────────────────────────────┐
                     │  FALLBACK: Cascaded STT → LLM → TTS system  │  activated on STS
                     │  (separate vendor / separate infra,          │  outage, degraded
                     │   kept warm, health-checked)                 │  audio quality, or
                     └───────────────────────────────────────────┘  session-length limits
```

## 8.2 What belongs inside the STS model vs. outside it

| Component | Location | Why |
|---|---|---|
| Turn-taking / barge-in behavior | **Inside** the STS model | This is precisely the capability native STS is architecturally best at; re-externalizing it gives up the main benefit |
| Prosody / tone matching / emotional responsiveness | **Inside** the STS model | Requires the acoustic-token representation the model natively has; a downstream text-only system structurally cannot do this |
| Multilingual fluency, mid-sentence code-switching | **Inside** the STS model | Same reasoning — this is a strength of jointly trained audio+text models |
| "What should I say right now, conversationally" | **Inside** the STS model, but constrained by system instructions from the orchestration layer | The model should decide *phrasing*, not *policy* |
| Tool/function definitions and execution | **Outside**, in the agent/orchestration layer | Execution needs to be deterministic, testable, authenticated, and independently auditable |
| Business logic (pricing rules, eligibility checks, workflow state machines) | **Outside** | Must be reviewable by compliance/legal, versioned, and testable independent of any model behavior change |
| Long-term memory / customer history | **Outside**, in a proper session/data store | STS model context windows are finite and not designed as a system of record; also needed for cross-channel consistency (the same customer record should be visible whether they call, chat, or email) |
| Authentication / authorization | **Outside**, in a hardened identity system | Never delegate "is this really the account holder" to a generative model's judgment |
| Safety guardrails / policy enforcement | **Outside** (with a defense-in-depth layer also inside via system prompting) | Belt-and-suspenders: prompt the model toward safe behavior, but enforce hard constraints in code that the model cannot talk its way around |
| Observability / transcript logging | **Outside**, capturing the STS model's transcript event stream into an immutable audit log | Even though native STS collapses the pipeline internally, the transcript stream most vendors expose (OpenAI's audio-transcript events, Nova Sonic's parallel text output) should still be captured externally as a first-class compliance artifact |
| Fallback cascaded pipeline | **Outside**, entirely separate infrastructure | Needed for resilience — see 8.4 |

## 8.3 Component notes

- **Transport layer**: WebRTC for browser/app clients (lower latency, better jitter handling than plain WebSocket audio); SIP trunking for telephony numbers feeding into the same media layer. Frameworks like LiveKit Agents and Pipecat exist specifically to abstract this layer and provide production primitives for turn-taking, interruption, function calling, and telephony as first-class features.
- **Edge/client-side VAD**: even with a native STS model handling sophisticated turn-taking internally, a cheap client- or edge-side VAD gate is still valuable as a first-pass filter — it avoids streaming pure silence or background noise into the (more expensive) STS model continuously, and it can provide a fast, low-latency signal for immediate playback interruption on barge-in before the model's own turn-taking logic has fully processed the interruption.
- **Agent/orchestration layer statelessness w.r.t. the STS model**: this is the single most important design discipline from file 07's Approach 4 — the business logic, memory, and safety layer should be written against a stable internal API contract (structured tool-call requests/results), not against any particular STS vendor's quirks, so that the conversational model can be swapped, upgraded, or fallback-routed without touching the system of record.
- **Observability**: because native STS collapses the pipeline internally, you lose the automatic, forced checkpoint a cascade gives you for free at each stage. This has to be deliberately re-added: capture the model's transcript event stream (nearly every serious commercial vendor now exposes one), log every tool-call request/response pair, and consider periodically sampling raw audio for QA review, since a text transcript alone won't capture tone-related quality issues (e.g., the agent sounding inappropriately cheerful during a bad-news call).
- **Fallback system**: keep a warm, health-checked cascaded STT→LLM→TTS path on entirely separate infrastructure/vendor, specifically because native STS is a newer, less battle-tested category with fewer years of production hardening, session-length limits that require reconnection logic, and — critically — represents a single point of failure if that one vendor has an outage. A fallback path also matters for graceful degradation on poor network conditions, where a lighter-weight cascade might tolerate packet loss/jitter better than a model expecting a clean continuous audio stream.

## 8.4 Why the fallback path matters architecturally, not just operationally

Because Approach 4 (file 07) keeps all business logic external and stateless with respect to the conversational model, **the fallback cascade can share the same agent/orchestration layer, memory, and tool-execution systems as the primary native-STS path.** Only the conversational "front end" needs to differ. This is a direct, practical payoff of the hybrid architectural discipline recommended throughout this research: the deterministic core doesn't care whether the words it's executing on behalf of arrived via a native STS model's tool-call event or via a traditional LLM's function call from a cascaded pipeline.

## 8.5 A note on cost architecture

Native STS audio-token pricing (e.g., OpenAI's `gpt-realtime` at roughly $32–64 per 1M audio tokens, versus $5–20 per 1M text tokens) means the *conversational* leg of a call is now meaningfully more expensive per minute than the old cascade's LLM leg was — while STT and TTS legs disappear as separate costs. Whether this nets out cheaper or more expensive than an optimized cascade depends heavily on call volume, call length, and how much of the conversation is "small talk" vs. substantive tool-calling turns; this should be modeled explicitly per use case rather than assumed in either direction, and is a legitimate input into the Approach 1 vs. 2 vs. 3 decision in file 07.
