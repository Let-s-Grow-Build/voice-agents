# Speech-to-Speech (STS) Architecture Deep Dive

A technical research repo on how modern **Speech-to-Speech / native audio models** differ architecturally from the traditional voice-agent pipeline (`Speech → STT → LLM → TTS → Speech`), and how to actually use STS to improve a production voice system — without giving up determinism, auditability, and control.

This isn't a generic "what is speech-to-speech" explainer. It's an architecture-first breakdown: what's inside these models (audio tokens, codecs, Thinker/Talker splits), how current models (OpenAI Realtime, Gemini Live, Amazon Nova Sonic, Kyutai Moshi, Sesame CSM, Qwen-Omni) actually differ, and how to design a production system that uses STS without handing it your business logic.

## Why this repo exists

If you're running (or building) a voice agent on the classic `STT → LLM → TTS` stack, the honest questions are:
- What does a native STS model actually do differently at the architecture level — not just the marketing page?
- Does it really "avoid text," or does text just move somewhere else inside the model?
- Where does STS genuinely help (latency, prosody, turn-taking) — and where does it quietly cost you (observability, determinism, auditability, lock-in)?
- If you already have a production system, what's the least risky way to adopt STS — full replacement, or something more surgical?

This repo tries to answer all of that with sources, not vibes.

## Repo structure

```
.
├── README.md                      ← you are here
├── architecture/                  ← the full research, as separate markdown files
│   ├── 00_executive_summary.md
│   ├── 01_traditional_architecture_and_limitations.md
│   ├── 02_modern_sts_deep_dive.md
│   ├── 03_architecture_comparison.md
│   ├── 04_current_sts_models_2026.md
│   ├── 05_latency_analysis.md
│   ├── 06_case_studies.md
│   ├── 07_upgrading_existing_system.md
│   ├── 08_production_architecture.md
│   ├── 09_tradeoffs_and_evaluation_framework.md
│   └── 10_final_recommendation_and_future.md
└── fine Tune Moshi/                ← hands-on: fine-tuning Kyutai's Moshi
    └── moshi_finetune_official_annotated.ipynb
```

## What's in `architecture/`

| # | File | What it covers |
|---|---|---|
| 00 | [Executive Summary](Architecture/00_executive_summary.md) | The whole research in one page — key findings, the core architectural shift, and the bottom-line recommendation |
| 01 | [Traditional Architecture & Limitations](architecture/01_traditional_architecture_and_limitations.md) | Deep dive on `STT → LLM → TTS`: latency, streaming, turn-taking, barge-in, context loss, prosody loss, system complexity |
| 02 | [Modern STS Deep Dive](architecture/02_modern_sts_deep_dive.md) | What's actually inside an STS model — audio encoders/codecs, audio tokens, where reasoning happens, whether text really disappears |
| 03 | [Architecture Comparison](architecture/03_architecture_comparison.md) | Side-by-side tables: components, data representation, latency, turn-taking, cost, observability, lock-in — and *why* each difference exists |
| 04 | [Current STS Models (2026)](architecture/04_current_sts_models_2026.md) | Model-by-model: OpenAI Realtime, Gemini Live, Amazon Nova Sonic, Kyutai Moshi, Sesame CSM, Qwen3/3.5-Omni — the actual architectural innovation each one introduced |
| 05 | [Latency Deep Dive](architecture/05_latency_analysis.md) | Where latency lives in each architecture, with published numbers (Moshi: 160–200ms, Qwen3-Omni: ~234ms, cascades: ~600ms–1s well-tuned) |
| 06 | [Case Studies](architecture/06_case_studies.md) | Customer support, real-time translation, voice assistants, enterprise agents — old vs. new architecture, what improves, what gets worse |
| 07 | [Upgrading an Existing System](architecture/07_upgrading_existing_system.md) | Four concrete upgrade paths, from "optimize your cascade" to "hybrid STS + deterministic agent" |
| 08 | [Recommended Production Architecture](architecture/08_production_architecture.md) | Full 2026 production blueprint — what goes inside the STS model vs. what stays external (tools, memory, auth, safety, fallback) |
| 09 | [Tradeoffs & Evaluation Framework](architecture/09_tradeoffs_and_evaluation_framework.md) | What you gain vs. lose (debuggability, determinism, cost, compliance), plus a decision matrix |
| 10 | [Final Recommendation & Future](architecture/10_final_recommendation_and_future.md) | The direct answer: hybrid, not full replacement, for most production systems — and where the field is heading |

**Suggested reading order:** start with `00`, then read straight through `01 → 10`. If you only have 10 minutes, read `00` and `10`.

## What's in `finetune-moshi/`

A working, annotated notebook for **LoRA fine-tuning Kyutai's Moshi 7B** on your own spoken dialogue data — based on Kyutai's official `moshi-finetune` repo and tutorial notebook, with explanation cells added throughout so you understand *why* each step exists, not just what to run. Covers: cloning the training code, preparing stereo-audio + transcript data, the config knobs that actually matter for memory/speed (`lora.rank`, `duration_sec`, `batch_size`, `gradient_checkpointing`), running training, and talking to your fine-tuned model live via Gradio.

## Epistemic notes

Claims in `architecture/` are sourced from published papers, technical reports, and vendor documentation where available (Kyutai's Moshi/Mimi paper, Alibaba's Qwen-Omni technical reports, Sesame's CSM model card, AWS's Nova Sonic documentation). Where a closed vendor (OpenAI, Google, Amazon) hasn't published full internal architecture details, this is explicitly flagged as **inference** rather than presented as fact. See each file's closing notes for specifics.

## License / usage

This research is provided for learning and reference. Model weights, code, and trademarks referenced throughout (Moshi, Mimi, Gemini, GPT, Nova Sonic, Qwen, CSM) belong to their respective creators — check their individual licenses before using them in your own projects.

---

⭐ If this helped you understand the shift from cascaded voice pipelines to native speech-to-speech, consider starring the repo and sharing feedback via an issue.
