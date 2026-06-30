---
title: "Paying Down Knowledge Debt: Finding the First Seam"
date: 2026-06-05
authors: [allann]
categories:
  - ai
  - programming
tags: [ai, software-engineering, architecture]
---

# Paying Down Knowledge Debt: Finding the First Seam

In my [last post](./knowledge-debt-2.md) I argued that the way to modernise a large, tangled system is not a full rewrite but the strangler fig, carving off one piece at a time. Which leaves the real question: which piece first? A seam is simply a place you can change one side of a line without disturbing the other. In a tightly coupled system those lines are faint, but rarely absent. Here is the order I look in.

<!-- more -->

**Start at the edges, not the heart.** The instinct is to attack the most central, most painful module first. Resist it, that is where the most rules are hidden and the most things break. Begin where you can win quietly: the parts that talk to the outside world, like a report, an export, a notification, an integration. These have a naturally narrow boundary. Data goes in, a result comes out.

**Follow the data, not the code.** Code structure misleads; data flow rarely does. Trace what reads and writes a given set of records. A cluster of behaviour that touches its own data and little else is a candidate. The shared table everything writes to is the heart, that comes later, once you have practice.

**Prefer the stable over the busy.** Carving a boundary around code three people are editing this week is a fight you do not need. Find a corner that has been quiet for a while.

**Then let the easy win and the pain point coincide.** The ideal first seam is something the business already complains about, slow, fragile, frequently broken, that also happens to sit behind a narrow boundary. Visibly worth doing, low technical risk. That is what earns you the second seam.

This is where AI first earns its place. Tracing every reader and writer of a record across a sprawling codebase is exactly the tedious, mechanical work it is good at, surfacing dependencies you would have missed and drawing the rough map far faster than you could by hand. It will get things wrong, so you verify rather than trust. But as a way to see the tangle, it is a genuine accelerant.

Once you have a candidate, the first real act is not to change it. It is to capture what it does today in tests, exactly as it behaves now, quirks included, turning undocumented rules into documented ones. This is where AI helps again: pointing it at the existing code to draft those characterisation tests is far quicker than writing them cold. You still read every one and confirm it pins the real behaviour, but the blank page is gone.

That is the pattern this whole series is circling. AI is not rebuilding your legacy system, and it should not. But it can map the tangle and help capture the behaviour, the two slowest, most thankless parts of carving a safe boundary. You supply the judgement about where the seam belongs and whether the tests are honest. You get to include your tribal knowledge. The system gets easier to change for everyone, human and machine, one seam at a time.

*Next in this series: writing the tests that capture behaviour nobody documented.*

---

**Knowledge Debt Series:** [1](./knowledge-debt-1.md) · [2](./knowledge-debt-2.md) · Part 3 · [4](./knowledge-debt-4.md) · [5](./knowledge-debt-5.md) · [6](./knowledge-debt-6.md) · [7](./knowledge-debt-7.md) · [8](./knowledge-debt-8.md)
