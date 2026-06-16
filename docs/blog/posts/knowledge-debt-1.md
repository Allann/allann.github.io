---
title: "Knowledge Debt: The Harder Problem"
date: 2026-06-03
authors: [allann]
categories:
  - ai
  - programming
tags: [ai, software-engineering]
---

# Knowledge Debt: The Harder Problem

We talk a lot about tech debt. We're about to learn that Knowledge Debt is the harder problem.

There's a growing debate in our industry, on one side, the "vibe coders" who greenfield entire applications with AI agents without a second thought. On the other, experienced engineers like Zoran Horvat who warn, with good reason, that we're sleepwalking into a profession we no longer understand.

Both sides have a point. But I think we're missing something in the middle.

<!-- more -->

The real risk isn't just 700-line god objects and 110 switch cases in a codebase nobody reviewed. It's what happens inside the heads of the developers who stopped reading. That's Knowledge Debt and unlike technical debt, you can't schedule a sprint to repay it.

Tech debt lives in your repo. Knowledge debt lives in your people. When it compounds, when senior engineers stop understanding the "why" behind their own systems because an LLM generated the "what" under pressure to ship, that knowledge doesn't go into docs or commit messages. It just... stops existing.

But here's where I think the conversation needs to go further:

The tooling isn't ready for what we're asking it to do.

Current AI coding agents are, fundamentally, sophisticated statistical generative models. They are extraordinarily good at pattern completion. They are not good at understanding consequences. They don't model the world your code runs in, the downstream effects, the emergent behaviours at scale, the silent failure modes that only surface months later.

What we actually need, before we hand agentic engines the wheel, are things like world models and approaches like JEDA (Joint Embedding with Discriminative Alignment) that allow AI systems to reason about cause and effect before producing output, not just predict what plausible code looks like.

Until we get there, the middle ground looks something like this:

- Use AI as a force multiplier, not a replacement for comprehension
- Treat code review as a knowledge retention exercise, not a formality
- Recognise that the skill we're protecting isn't typing, it's reasoning

The developers who will thrive aren't the ones who vibe code everything, or the ones who reject AI entirely. They're the ones who use the tools, stay in the loop, and push the industry to demand better ones.

We haven't found that middle ground yet. But we need to and fast.

---

**Knowledge Debt Series:** Part 1 · [2](./knowledge-debt-2.md) · [3](./knowledge-debt-3.md) · [4](./knowledge-debt-4.md) · [5](./knowledge-debt-5.md) · [6](./knowledge-debt-6.md) · [7](./knowledge-debt-7.md) · [8](./knowledge-debt-8.md)
