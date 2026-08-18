---
title: "RAG cannot replace SFT and LoRA"
date: 2026-08-18
---

# RAG cannot replace SFT and LoRA

*The redundancy tax: RAG gets worse with redundancy, and it adds to the cost of context too. Below are the numbers, the mechanism, and the one job retrieval gets to keep.*

---

## 0. TL;DR

A claim making the rounds says retrieval makes fine-tuning optional: skip the training, just stuff the prompt. On my own error-recovery task, it does not hold. Same 2,159 training pairs delivered two ways - baked into weights (LoRA) or pasted into the prompt (RAG) - on a 200-case held-out exam: base model 1%, RAG 61.5%, LoRA 77.5%, and LoRA + RAG **69%**. Retrieval rescues a model that knows nothing, loses the head-to-head against training on identical knowledge, and *taxes* a trained model instead of helping it. RAG cannot replace SFT and LoRA - and stacking it on top of them made things worse, not better. Worse answers, on more context.

## 1. The folklore: stack them

The standard advice for small specialist models: **fine-tune for behavior, add RAG for knowledge, stack them.** Fine-tuning teaches the model *how* to act; retrieval keeps it *current*. Every reference architecture draws them as two layers of the same cake. The stronger version goes further: with a good retriever, why fine-tune at all? I believed the stacking version too, which is why I ran the four-condition experiment instead of just shipping the stack.

## 2. The setup: a fair fight

The task is my own error-recovery idiom - the same corpus, harness, and specialist model described in [The $50 Specialist]({% post_url 2026-08-05-the-50-specialist %}). The LoRA adapter (rank 16, 74 MB) and the RAG index were built from the **identical** 2,159 pairs. That is deliberate: same knowledge, two delivery channels, no excuses about who had better data.

The exam: 200 held-out cases, never trained on, never retrieved from. Each shows the model a situation (task, recent activity, the error) and demands one tool call as JSON. Three scores, strictest last: **valid JSON** (does it speak the contract), **right tool** (does it know the fix), **exact arguments** (a deliberately brutal bar - many phrasings are equally correct). Greedy decoding, so every number here is reproducible digit-for-digit. Per-condition noise is about +/-3.5 points at n=200.

## 3. The scoreboard

| Condition | valid JSON | right tool | exact args |
| --- | --- | --- | --- |
| A. base model | 93% | **1%** | 0% |
| B. base + RAG | 77% | **61.5%** | 2% |
| C. base + LoRA | 91% | **77.5%** | 2.5% |
| D. LoRA + RAG | 85.5% | **69.0%** | 1.5% |

## 4. Three findings

**Retrieval rescues a model that knows nothing.** The base model writes perfectly parseable JSON and picks my tools at chance. Show it three similar past fixes in the prompt and tool accuracy jumps to 61.5% - sixty points of knowledge, zero training. RAG works.

**Training beats retrieval on identical knowledge.** Same corpus in weights instead of context: 77.5% vs 61.5%, a sixteen-point win, with cleaner output to boot (91% valid vs 77%). If RAG could replace fine-tuning, this is where it would have happened - same data, same exam, no GPU session required. It did not.

**Stacking is worse than either alone.** I expected D to roughly equal C: the model already knows this material, so it should ignore the pasted examples and lose nothing. Instead it lost **8.5 points** of tool accuracy and six points of JSON validity. The redundant examples did not go unread. They did damage.

## 5. The redundancy tax

The intuition that fails here: extra information is helpful or neutral, because the model can always ignore it. An LLM cannot ignore its context - there is no skip mechanism. Every token in the prompt bends the output. The only question is whether a token buys more than it costs.

Retrieval charges on every call. There is the literal bill first - three pasted examples riding on every prompt: more tokens, more latency, more money, every single call. And then there are three quality taxes:

- **Attention on near-misses.** The retriever returns the *most similar* past cases, and similar-looking situations often call for different tools - that is why they are separate pairs in the corpus. The examples whisper "situations like this got tool Y" at exactly the cases that need tool X. The fine distinctions live in the weights; the retrieved neighbors smear them.
- **Blending instead of arbitrating.** A frontier model can weigh the prompt against its training and referee the conflict. A 1.5B cannot arbitrate - it blends. Blending three near-misses with the right answer pulls the answer off target, hardest on the knife-edge cases where trained discrimination earns its keep.
- **Format leakage.** Long example blocks have their own surface shape, and a small model imitates whatever fills its context. Hence the validity drops: 93 to 77 on the base model, 91 to 85.5 on the trained one. The shots teach content and tax form.

Same tax every call. When the information is new (condition B), the benefit dwarfs the cost. When it is already in the weights (condition D), the benefit is zero - and the tax collects anyway.

The case file: pairing the 200 exams head-to-head, retrieval knocked out a correct trained answer **12 times** and rescued a wrong one **6 times**. One flip is almost too clean an exhibit: the situation called for running a command (`exec_command`); the trained model said so; the RAG-augmented model emitted a full `apply_patch` payload instead - a faithful imitation of one of its retrieved near-misses. The shots did not inform its decision. They outbid it.

## 6. What this does not kill

Honest boundaries, because the headline is easy to over-read:

- **RAG as the update channel is untested, not refuted.** Index and adapter came from the *same* pairs - redundancy by construction. The open case is retrieval carrying knowledge the weights lack: fixes mined *after* the last retrain. Train on N, retrieve on N+delta. That is the next experiment, and it is where RAG should earn its seat: the chart supplement between revisions, not a second pilot reading the chart aloud on final.
- **Scale may soften the tax.** Arbitration is a capability and it grows with model size; a much larger model might absorb redundant shots gracefully. That bounds this finding to the small-model regime - but the small-model regime is the point of this lab.
- **Retrieval-aware training exists** - train on retrieval-shaped prompts and the tax presumably shrinks. But that spends training budget to accommodate a sensor, which inverts the economics that made retrieval attractive.
- **n=200, one task family, one retriever, K=3.** The effect clears the error bars and the mechanism is visible in the raw outputs, but generality is earned one replication at a time.

## 7. The rule

**Retrieval pays when it tells the model something new. It taxes when it repeats something known.**

So: RAG cannot replace SFT and LoRA - measured, same data, same exam. And the reverse holds only with an asterisk: weights cannot absorb knowledge that did not exist at train time, which is the one seat retrieval keeps. Everything else collapses into a division of labor: facts that change faster than you retrain go to retrieval; form goes to weights; and the improvement channel for a trained specialist is not more context and not more capacity - it is more data. The corpus is the moat. Everything else is plumbing.