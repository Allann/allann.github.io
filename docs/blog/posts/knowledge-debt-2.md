---
title: "Paying Down Knowledge Debt: The Rewrite Temptation"
date: 2026-06-10
authors: [allann]
categories:
  - ai
  - programming
tags: [ai, software-engineering, architecture]
---

# Paying Down Knowledge Debt: The Rewrite Temptation

In my [last post](./knowledge-debt-1.md) I wrote about Knowledge Debt, what happens when we stop understanding the systems we ship. The obvious follow-up: what do we do about it, especially in the large, long-lived codebases where it has compounded the longest?

AI coding demonstrations rarely resemble those systems. They start from an empty folder, which has no history to misunderstand. We work in the opposite conditions: decades of accumulated behaviour, outdated documentation, and critical rules living only in the memory of people who may have long since left.

<!-- more -->

AI assistance has a real limitation here. Even the largest models can only see a fraction of a substantial codebase at once. In a well-structured system that fraction can be enough; in a tightly coupled one it rarely is. A one-line change can have consequences in a module far away, enforced by a rule nobody wrote down. The model does not know that rule exists. Increasingly, neither does the team. That is Knowledge Debt compounding quietly until something forces the issue.

When it is forced, the tempting answer is a full rewrite. Start clean, rebuild fast with modern tooling.

In my experience, full rewrites of business-critical systems rarely succeed. The old system is the specification. Decades of edge cases, regulatory requirements and hard-won fixes live in that code and nowhere else. Rebuilding from memory means rediscovering each lesson in production, while the business keeps running on the system being retired. The rewrites that succeed tend to involve small, stable, well-understood systems, precisely not the ones causing the problem.

There is a quieter approach with a far better record: the strangler fig, named for the tree that grows around its host. Carve off one piece at a time. Pin down its current behaviour with tests, so undocumented rules become documented ones. Put a clean boundary around it. Rebuild just that piece, route traffic through it, verify nothing changed. Repeat.

Each piece is small enough to genuinely understand, for the team, the next developer, and yes, an AI assistant. The boundaries that make a system safe for people are the same ones that make it workable for machines.

For engineering leaders, this changes the modernisation conversation. A rewrite is a large, risky bet with value arriving only at the end. A strangler approach is incremental investment: each step delivers a bounded, tested, documented component, value lands continuously, and the work can pause without stranding the business between two systems. The productivity gains from AI tooling, which depend on exactly these boundaries, give that investment a return leadership can see on a budget line, not just in an architecture diagram.

Paying down Knowledge Debt and preparing for the AI era are the same work. We finally have a commercial reason to do what we always knew was right.

---

**Knowledge Debt Series:** [1](./knowledge-debt-1.md) · Part 2 · [3](./knowledge-debt-3.md) · [4](./knowledge-debt-4.md) · [5](./knowledge-debt-5.md) · [6](./knowledge-debt-6.md) · [7](./knowledge-debt-7.md) · [8](./knowledge-debt-8.md)
