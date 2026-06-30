---
title: "Paying Down Knowledge Debt: The Tests That Tell the Truth"
date: 2026-06-06
authors: [allann]
categories:
  - ai
  - programming
tags: [ai, software-engineering, testing]
---

# Paying Down Knowledge Debt: The Tests That Tell the Truth

Last time I said the first act when you've found a seam is not to change it. It is to write tests that capture what it does today, exactly as it behaves now, quirks included. That sentence is easy to write and surprisingly hard to sit with.

<!-- more -->

These are not unit tests in the usual sense. You are not specifying what the code should do. You are documenting what it actually does. Michael Feathers called them characterisation tests. I think of them as a confession the code makes about itself.

The process is blunt. You run the code with representative inputs and record the outputs. Whatever comes back is correct, by definition, for now. That result becomes the test assertion. You are pinning behaviour, not validating it.

This matters because the rules you are capturing are often invisible. No ticket was raised. No architecture decision record was written. A senior developer fixed something four years ago and the fix became load-bearing. The business has been relying on it ever since. Characterisation tests make that rule executable. It now lives in your version control, not in someone's memory about to walk out the door.

AI earns its keep here. Point it at the existing code and ask it to draft characterisation tests. It will trace the paths, identify the branches, and produce a scaffold far faster than you can write it cold. You will still read every test and confirm it pins real behaviour. Some will be wrong. A few will reveal things the code does that genuinely surprise you. Both outcomes are useful.

The tests you do not write are the interesting ones. Where the code is hard to test is where the coupling lives, where behaviour leaks across boundaries, where the rules are most hidden. Those gaps are not a failure of your testing effort. They are a map of the problem.

What you end up with is something valuable beyond the immediate task: a living specification. When you rebuild that piece, these tests are your acceptance criteria. When the rebuilt version passes them all, you have done something genuinely hard: changed the implementation without changing the behaviour the business depends on.

The knowledge that was locked in the code is now locked in the tests instead. That is a better place for it.

*Next in this series: routing traffic through your new implementation.*

---

**Knowledge Debt Series:** [1](./knowledge-debt-1.md) · [2](./knowledge-debt-2.md) · [3](./knowledge-debt-3.md) · Part 4 · [5](./knowledge-debt-5.md) · [6](./knowledge-debt-6.md) · [7](./knowledge-debt-7.md) · [8](./knowledge-debt-8.md)
