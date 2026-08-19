---
title: "RAG helps where you're weak, taxes where you're strong"
date: 2026-08-19
---

# RAG helps where you're weak, taxes where you're strong

*Yesterday's post showed that adding RAG to a trained small model makes it worse overall. Today: a closer look at where exactly it hurts - and where it quietly helps.*

---

## TL;DR

[Yesterday's experiment]({% post_url 2026-08-18-rag-cannot-replace-sft-and-lora %}) ended with a clean result: adding retrieval to my trained 1.5B model cost 8.5 points of accuracy. New breakdown, same data: the tax is not uniform. It depends on the error category.

- **Where the model is strong, RAG taxes it.** In cloud-auth and JSON categories (89% trained accuracy), adding retrieval dropped accuracy to 67%.
- **Where the model is weak, RAG helps.** In package/environment errors (43% trained), retrieval lifted accuracy to 71%.
- **A router beats both.** Send weak categories to retrieval, keep strong ones trained-only: about 79.6% versus 77.5% trained-only. Free points, no retraining.

## The question

Yesterday's exam: 200 held-out error-recovery cases. The trained model scored 77.5%. The trained model plus RAG scored 69%. Stacking hurts, on average.

But an average hides a lot. The 200 cases are not one kind of problem - they span ssh failures, git errors, missing files, network timeouts, and more. So the next question: is the tax the same everywhere?

## The map

Same exam, broken down by error category. **C** is the trained model alone. **D** is the trained model plus RAG. Positive C-D means retrieval hurt; negative means it helped.

| Error category | cases | C (trained) | D (trained + RAG) | C-D |
| --- | --- | --- | --- | --- |
| git | 4 | 50% | 25% | +25 |
| cloud auth/region | 9 | 89% | 67% | +22 |
| JSON/encoding | 9 | 89% | 67% | +22 |
| file not found | 7 | 86% | 71% | +14 |
| other | 134 | 80% | 69% | +11 |
| ssh/auth | 11 | 73% | 73% | 0 |
| network/timeout | 18 | 67% | 78% | -11 |
| package/environment | 7 | 43% | 71% | -29 |

**How to read it.** C is accuracy with the trained adapter alone. D is the same model with three retrieved examples added to its context. The last column is the tax: positive means retrieval cost accuracy, negative means it added some. One category with a single case is omitted.

## What the map says

The more the model already knows a category, the more retrieval hurts it:

- **Strong categories get taxed.** Cloud-auth and JSON: 89% trained, 67% stacked. The model knew these cold, and the examples dragged it off.
- **Weak categories get rescued.** Package/environment: 43% trained, 71% stacked. Network/timeout: 67% to 78%. Where the training was thin, the examples had something real to offer.
- **One tie.** ssh/auth stayed flat at 73%, knockouts and rescues canceling out.
- **One exception.** git was weak (50%) and still got taxed hardest - but with 4 cases, that is one flipped answer. Direction, not gospel.

So yesterday's rule gets a boundary condition: **retrieval pays where knowledge is missing, and taxes where knowledge is trained in** - not as a global law, but category by category.

## The practical fix: route it

If the tax is per-category, the fix is per-category too. Classify the error first (my corpus already has a cheap regex classifier), then decide: strong category - answer from the trained model alone; weak category - add retrieval.

On yesterday's numbers, that router scores about **79.6%**, versus 77.5% trained-only and 69% stack-everything. Not a leap, but free: no retraining, no new data. And the gains land exactly where the model is weakest, which is where mistakes are most expensive.

## Why does the tax happen? Open question.

Three hypotheses, none proven yet:

1. **Context stuffing.** The model copies an example's answer instead of using its training.
2. **Recency bias.** Autoregressive models over-weight the most recent text - and the last thing before the model's answer is the third example's answer. So it copies that one.
3. **Mixing.** The model blends its trained answer with the example's answer and lands in between, which is wrong.

All three are testable: log which examples were retrieved, and compare them against the wrong answers. Copied answers point to stuffing, copies of the last example point to recency, merges point to mixing. That experiment is queued.