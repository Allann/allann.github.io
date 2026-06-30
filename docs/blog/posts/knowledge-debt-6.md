---
title: "Paying Down Knowledge Debt: The Shared Database, the Hardest Constraint"
date: 2026-06-08
authors: [allann]
categories:
  - ai
  - programming
tags: [ai, software-engineering, architecture, database]
---

# Paying Down Knowledge Debt: The Shared Database, the Hardest Constraint

Every step in this series has been building toward this one. The seam is found, the tests are written, the routing is in place. Then you look at the data model and see the problem: the table the old code writes to is the same table everything else reads from. The database is the heart, and you cannot carve a clean boundary around the heart first.

<!-- more -->

This is where most strangler fig migrations stall. Not because the approach is wrong, but because shared state does not respect the boundaries you draw around the code.

The constraint forces a choice. You can own the schema in the new service, which means migrating the data and keeping it synchronised during the overlap period. You can leave the schema with the original, which means the new service has a dependency you said you were cutting. Or you can run a transitional model, where both read from a shared store under a defined contract, and migration of the schema happens as a later step.

There is no clean answer. What there is, is a pattern: own the synchronisation explicitly rather than letting it happen by accident. If both old and new write to the same table for a period, make that a documented decision with a removal date, not an oversight.

Event-driven approaches help here. If the new service emits events and the old code, or a translation layer, consumes them, neither owns the other's data model directly. The coupling becomes temporal rather than structural. This is not always possible, but when it is, it gives you the cleanest cut.

AI has a limited role in this phase. It can map which parts of the schema each service actually touches, faster than you can trace it by hand. It cannot tell you what the right ownership model is. That is a conversation between technical lead, product owner, and sometimes legal or compliance. The mapping informs the conversation. It does not replace it.

The shared database is the last constraint to fall. That is by design. You earn the right to touch it by successfully migrating everything around it first.

*Next in this series: making knowledge capture a habit, not a project.*

---

**Knowledge Debt Series:** [1](./knowledge-debt-1.md) · [2](./knowledge-debt-2.md) · [3](./knowledge-debt-3.md) · [4](./knowledge-debt-4.md) · [5](./knowledge-debt-5.md) · Part 6 · [7](./knowledge-debt-7.md) · [8](./knowledge-debt-8.md)
