---
title: "The $50 Specialist: training a tiny model to run my error-recovery loop"
date: 2026-08-05
---

# The $50 Specialist: training a tiny model to run my error-recovery loop

*What if I wanted to add tensors to a small model and see if it learns the very specific tasks I work on every day? I do not need a model that writes poetry. I need one that knows how to run the loop: CLI -> error -> recovery step -> resolution. This post describes my experiment to build exactly that — a custom model that correctly fixes my tasks, and has no other opinion.*

---

## 0. TL;DR

I took three months of my own AI-agent session logs (352 sessions, 2.8 GB), curated them into 2,580 (error -> recovery) training pairs, fine-tuned a small open-weights model with a LoRA adapter (74 MB, 0.1% of parameters), and ended up with a local specialist that recognizes my environment''s failure modes and proposes my fixes — for under $50 of GPU time. It works. It also fails in instructive ways. Both are documented below with numbers.

## 1. The data: raw traces are not training data

**Source.** 352 session logs (rollout JSONL) from my daily agent-driven work: cloud API operations, database work, spreadsheets, git. Every record is a timestamped JSON blob: user messages, assistant reasoning, tool calls, tool outputs.

**Curation.** Raw logs are ore, not metal. The curation pipeline (~100 lines of Python):

1. **Find error moments** — regex over tool outputs (permission denied, not found, timeout, non-zero exit codes): 4,135 candidates across 104 sessions (30% of sessions had errors).
2. **Verify recovery** — keep a moment only if the *next* tool call produced clean output. No verified recovery, no training pair. This is the step that keeps flailing out of the dataset.
3. **Compress context** — task (last user message), recent activity (few calls), the error text; each field truncated.
4. **Scrub** — instance ids, keys, tokens, IPs masked (91k lines matched secret patterns).
5. **Dedupe** — hash on (error, action); 500 identical recoveries count once.

Result: **2,580 pairs**, plus **3,186 more** from shell-history logs (adjacent-command typo/flag fixes, with a destructive-escalation guard: never teach `ls` -> `rm -rf`).

**Per-class distribution** (this matters later):

| Error class | Pairs | Share |
| --- | --- | --- |
| fs/not-found | 187 | 7.2% |
| network/timeout | 163 | 6.3% |
| ssh/auth | 159 | 6.2% |
| cloud auth/region | 144 | 5.6% |
| json/encoding | 116 | 4.5% |
| package/env | 115 | 4.5% |
| **git** | **52** | **2.0%** |
| **shell-parse** | **2** | **0.1%** |
| other | 1,642 | 63.6% |

## 2. The method: a small base + a small tensor

- **Base:** ~1.5B-parameter open-weights instruct model. Small on purpose — the hypothesis was that this task needs reflexes, not breadth.
- **LoRA:** rank 16, alpha 32, on all attention + FFN projections. Trainable params: 18.5M of 1,562M (**1.18%**). The base stays frozen; only the fresh A/B tensors learn.
- **Merge after training:** `W_new = W + alpha * B * A` per layer — one matrix multiply and the adapter becomes ordinary weights.
- **Stack:** torch / transformers / peft / trl (SFTTrainer, chat-format `messages`, loss on assistant tokens only), single A10 GPU.

**Training:** 3 epochs over 2,580 pairs, batch 4, 56 minutes wall time. Loss 2.33 -> 0.35; mean token accuracy ~90%.

**Cost:** well under $50 of GPU time. The model weights are free. Total cost of the custom model: **$50**.

## 3. Packaging gotchas (the part blogs skip)

- **TRL 1.9.x defaults **`**loss_type="chunked_nll"**`, which monkeypatches the model forward and crashes against PEFT adapters. Fix: `loss_type="nll"`.
- **Triton needs Python headers** on a fresh node: `apt-get install python3.10-dev`, or kernel compiles fail cryptically.
- **Ollama''s internal safetensors->GGUF converter hung** on my fp32 merge (every request spun forever — even "hi"). The official `llama.cpp` `convert_hf_to_gguf.py` produced a working GGUF on the first try. Lesson: convert yourself.
- **Bake the system prompt into the Modelfile** (`SYSTEM """..."""`). Without it, the trained format discipline never triggers in a plain `ollama run` session — the model reverts to generic chat.

## 4. Evaluation: what it learned, measured by interrogation

**Identity and format.** Asked "who are you", it answered "I am an agent-loop controller" — the persona from the training system prompt, learned from data alone. Given error states in the trained format, it emits the controller JSON: `{"tool": "exec_command", "args": ...}`.

**The good.** A novel error ("deployment not found", never in training) produced a correct recovery reflex — check existence first — in my own shell idiom, down to `2>/dev/null || true`. That is not in any textbook; it is from my logs.

**The bad.** A git non-fast-forward error produced, without the controller context, a confident recommendation of `git push -f origin master` — the classic destructive mistake. Two lessons: (1) base-model "knowledge" leaks through when the trained context is absent; (2) the baked system prompt now carries an explicit never-recommend-destructive-actions rule. Guardrails live in prompts and harnesses, not hopes.

**The map.** Performance tracks the class distribution almost exactly: strong on ssh/auth and cloud/region errors (150+ pairs each), weak on git (52 pairs), absent on shell-parse (2 pairs). The apprenticeship is literal: no examples, no skill.

## 5. What I actually concluded

1. **The loop is learnable, cheaply.** CLI -> error -> recovery -> resolution, for a specific environment, fits in 74 MB. The $50/dinner comparison is real.
2. **Curation > volume.** 2,580 verified-recovery pairs beat 2.8 GB of raw logs. The extractor rules are where the domain knowledge lives.
3. **The model has no opinion — and that is the point.** It proposes my fixes, in my format, and otherwise stays out of the way. The poetry belongs to other models.
4. **The failure modes are data problems, not model problems.** Git and shell-parse are weak because my logs are thin there. The fix is a simulator for those classes, not a bigger model.

## 6. Reproduce it

The full pipeline (extractors, trainer, merge, GGUF, Modelfile, node runbook) is a folder of small scripts; the whole thing reruns end-to-end in about an hour of GPU time plus curation. If you have three months of agent logs, you already own the hard part.

---

*Next in the series: the per-class evaluation harness — measuring which error classes the specialist actually owns, and the round-two data build (paraphrase robustness, guardrails, and the error-class simulator).*
