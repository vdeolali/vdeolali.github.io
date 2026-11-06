---
title: "Grok Bot is the cheapest way I run an agent"
date: 2026-11-06
---

# Grok Bot is the cheapest way I run an agent

*The agent work I was sending to the Codex API now runs on a Grok plan. The plan is about $300 a year. The API was $100+ a month.*

---

## TL;DR

- My Grok plan is about $300 a year. [Grok Bot](https://docs.x.ai/grok-bot/overview) is included on that plan. It is not a second subscription.
- The same kind of work, sent to the Codex API, costs me $100+ a month. That is $1,200+ a year. At least four times the plan.
- The bot is cheaper because the loop sits inside the subscription: a computer that stays on, the apps, the memory, and the schedule. On the API I pay for every turn of that loop.
- It stops being cheap if I blow the weekly usage pool, or if a script already does the job.

## The question: why was the API the expensive one?

An agent is not one answer. It is a loop. Look at the task. Use a tool. Read what came back. Try the next step. Remember it. Do it again tomorrow.

On an API, every step of that loop is a bill. The model sees the growing context again. The tool call is another charge. A short chat is cheap. A bot that keeps working after the laptop is closed is not, if you pay per turn. That is how a coding-agent API lands at $100+ a month without looking expensive on a per-token card.

## What the $300 a year actually buys

The public list price for Premium, paid annually, is $300 a year. That is the cheapest current plan that includes Grok Bot. Super and Ultra include it too. The $8 Plus plan does not. Bot usage draws from the same weekly pool as Grok and Cursor on that seat. It is not a separate meter. Past the pool, you buy top-up credits. Prices move. The current table is on [the pricing page](https://docs.x.ai/x-subscription/pricing).

| | Codex API, as I use it | Grok Bot on this plan |
| --- | --- | --- |
| What I pay | $100+ a month | about $300 a year |
| A year of that | $1,200+ | $300 |
| The computer | mine to keep running | included, and it stays on |
| Sign-ins and apps | I wire them | connected apps, or the browser |
| Doing it again tomorrow | I write the schedule | a routine |
| The meter | every turn | a weekly pool, then top-ups |

**How to read it.** The left column is my Codex API bill for agent work. The right column is the plan I already pay, with the bot included. The year row is the comparison that matters: $1,200+ against $300.

The bot has not added much on top of that plan. The subscription did not go up. Usage past the weekly pool is the only extra, and for this work that extra has stayed small.

## What the bot does that the API was doing

I hand it a job. It works on a cloud computer with a browser, a command line, and files. The computer stays up when my laptop is closed. Sign-ins persist. It uses a connected app when that is the reliable path, and the browser when there is no app. It asks before it sends, pays, or deletes, if I set that rule. Show it a job once, save the steps, and run them on a schedule. The walkthrough is in [How Grok Bot works](https://docs.x.ai/grok-bot/how-grok-bot-works).

For the work I was sending to the Codex API — read the task, use the tools, come back with a result — the bot covers it. It can also hand coding work to a Cursor cloud agent and attach the result. I do not keep a second harness, and I do not pay per turn.

I wrote earlier about the multi-agent knobs in [Grok Build]({% post_url 2026-07-11-using-xai-for-multiagent-ai %}). This post is the bill, not the knobs.

## Where this stops being the cheap path

Three boundaries, so the headline does not overclaim.

1. **The cheap plan is Premium, not Plus.** $300 a year includes the bot. The $8 plan does not. A plan with no bot is not a cheap agent.
2. **The pool is weekly, and it is shared.** Grok chat, Cursor, and the bot draw from one allowance. A heavy week spills into top-ups. Top-ups are an API bill again. Watch the usage page. If the top-ups start looking like $100 a month, this argument is over.
3. **A script you already trust is still cheaper.** If the job is the same every time and a cron job does it, do not put a bot on it. The bot earns its place on work that crosses apps, needs a browser, or changes often enough that the script rots.

One more limit, and it is not a cost limit. All of your bots share one computer. A sign-in for one is a sign-in for the others. Do not leave a credential on that computer if another bot should not have it.

## What I actually concluded

The expensive part of agentic AI was never the clever reply. It was paying for the loop, and paying again to keep the computer, the sign-ins, and the schedule alive.

A Grok Bot folds that loop into a plan I was already close to paying. About $300 a year, against $100+ a month on the Codex API. Same class of job. A quarter of the bill, or less. That is the cheapest agent I run.
