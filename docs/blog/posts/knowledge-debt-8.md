---
title: "Paying Down Knowledge Debt: The Horizon"
date: 2026-07-22
authors: [allann]
categories:
  - ai
  - programming
tags: [ai, software-engineering]
---

# Paying Down Knowledge Debt: The Horizon

I want to close this series by returning to where it started.

<!-- more -->

In the first post I wrote about Knowledge Debt as a risk distinct from technical debt. Tech debt lives in your code. Knowledge Debt lives in your people, and you cannot schedule a sprint to repay it. I also wrote about why current AI coding tools are not ready to carry the weight we are asking them to carry. They are pattern completers. They do not reason about consequence. World models, approaches like JEDA, the kind of causal reasoning that would make an AI safe to use autonomously on a production system — these do not exist yet in commercial tools.

What this series has been arguing is that the preparation and the payback are the same work.

The bounded, tested, documented codebase that a strangler fig migration produces is not just easier for your current team to work in. It is the kind of codebase a better AI will be able to reason about. Characterisation tests are machine-readable specifications. Architecture Decision Records are machine-readable context. Clean boundaries are the unit of work an autonomous agent needs in order to operate safely.

We are not waiting for better AI before we start. We are doing the work that makes better AI useful when it arrives.

There is something else here too. The engineers who go through this process — who trace the seams, write the tests, map the dependencies, document the decisions — are not being replaced by the tools they used to do it. They are doing the hardest work in software: making the implicit explicit. That is not automatable, because the knowledge is not in the code. It is in the gap between what the code says and what the business needs.

The vibe coders who skip this step will build systems that accumulate a new kind of Knowledge Debt, faster than before. The engineers who do this work will build systems that age well, that a future team can hand to better tools and trust them with.

That is the middle ground I mentioned in part one. This series was the map.

---

**Knowledge Debt Series:** [1](./knowledge-debt-1.md) · [2](./knowledge-debt-2.md) · [3](./knowledge-debt-3.md) · [4](./knowledge-debt-4.md) · [5](./knowledge-debt-5.md) · [6](./knowledge-debt-6.md) · [7](./knowledge-debt-7.md) · Part 8
