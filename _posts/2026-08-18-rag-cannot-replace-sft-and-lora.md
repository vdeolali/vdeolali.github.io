---
title: "RAG cannot replace SFT and LoRA"
date: 2026-08-18
---

# RAG cannot replace SFT and LoRA

*The redundancy tax: RAG gets worse with redundancy, and it adds to the cost of context too. Below are the numbers, the mechanism, and the one job retrieval gets to keep.*

---

## TL;DR

A claim making the rounds says retrieval makes fine-tuning optional: skip the training, just stuff the context. I tested it on my own error-recovery task - the same 2,159 training pairs delivered two ways (baked into weights vs pasted into the context), 200 held-out cases, one exam. Three findings:

- **Retrieval rescues a model that knows nothing.** The base model picks my tools at chance (1%); three pasted fixes take it to 61.5%. Sixty points of knowledge, zero training. RAG works.
- **Training beats retrieval on identical knowledge.** 77.5% vs 61.5% - sixteen points, with cleaner output to boot (91% vs 77% valid JSON). RAG cannot replace SFT and LoRA.
- **Stacking is worse than either alone.** LoRA + RAG drops to 69%: the trained model loses 8.5 points of tool accuracy the moment retrieval is added. Worse answers, on more context. The redundancy tax is real.

## Reigning Wisdom: Stack Them

The standard advice for small specialist models: **fine-tune for behavior, add RAG for knowledge, stack them.** Fine-tuning teaches the model *how* to act; retrieval keeps it *current*. Every reference architecture draws them as two layers of the same cake. The stronger version goes further: with a good retriever, why fine-tune at all?

So I tried it. RAG on top of a LoRA-trained base model - and it performed **worse**. Not the same. Worse. Stacking actually makes it worse. Here is the scoreboard first, then what each row means:

| Condition | valid JSON | right tool | exact args |
| --- | --- | --- | --- |
| A. base model | 93% | **1%** | 0% |
| B. base + RAG | 77% | **61.5%** | 2% |
| C. base + LoRA | 91% | **77.5%** | 2.5% |
| D. LoRA + RAG | 85.5% | **69.0%** | 1.5% |

**The rows.** The task is my own error-recovery idiom, the corpus and harness from [The $50 Specialist]({% post_url 2026-08-05-the-50-specialist %}): given a situation (task, recent activity, the error), emit one tool call as JSON. 200 held-out cases, never trained on, never retrieved from. **A** is the raw 1.5B base model with no help - the null control. **B** is the same base model with the three most similar past fixes (from the 2,159-pair corpus) pasted into its context. **C** is the base model plus the LoRA adapter (rank 16, 74 MB) trained on those same 2,159 pairs. **D** is the stack: the trained model *and* the retrieved examples. B and C see the identical corpus on purpose - same knowledge, two delivery channels, no excuses about who had better data. The columns run strictest left to right: valid JSON (speaks the contract), right tool (knows the fix), exact arguments (a deliberately brutal bar - many phrasings are equally correct). Greedy decoding, so every number is reproducible digit-for-digit; per-condition noise is about +/-3.5 points at n=200.

## Misleading intuition

The intuition that fails here is that extra information is helpful or neutral because the model can always ignore it - but an LLM cannot ignore its context; there is no skip mechanism. **The rule: retrieval pays when it tells the model something new, and it taxes when it repeats something known.**