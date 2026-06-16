---
title: "Paying Down Knowledge Debt: Routing Traffic Through Your New Implementation"
date: 2026-07-01
authors: [allann]
categories:
  - ai
  - programming
tags: [ai, software-engineering, architecture]
---

# Paying Down Knowledge Debt: Routing Traffic Through Your New Implementation

The characterisation tests pass. The rebuilt piece behaves exactly as the original did. Now you have to make the real system use it.

<!-- more -->

This is the strangler fig in action. The idea is simple: put a routing layer in front of the old code. Traffic goes in, the router decides whether to send it to the original implementation or the new one. You start with nothing going to the new one. You end with everything going there. In between, the old code waits.

The routing layer does not need to be clever. An environment variable, a feature flag, a percentage rollout. What matters is that it can be changed without a deployment. If switching requires a release cycle, you have made the cut-over riskier than it needs to be.

Before you commit, run both implementations in parallel. Not as an A/B test, where different users get different behaviour. Both handle the same request, and you compare their outputs. Any divergence is a finding. Sometimes it is a bug in your new code. Sometimes it is a gap in your characterisation tests. Occasionally it is behaviour in the original that nobody knew was wrong. All three outcomes are worth discovering before your users do.

This is where AI assists again. Comparing outputs at scale, flagging divergences, categorising them by type and frequency, is exactly the kind of mechanical analysis it handles well. You still investigate each class of divergence and decide what it means. The model surfaces the signal; you supply the judgement.

The characterisation tests are your acceptance gate. The new implementation does not leave shadow mode until it passes them completely, in the environment it will actually run in.

When you cut over, keep the old code in place for a period. Do not delete it on the day. Give yourself a circuit breaker: if something surfaces in production that your tests did not capture, you want to route back without ceremony. Schedule the removal. Put it in a ticket. Set a date. Then do it.

The strangler fig earns its name here. The old implementation does not fall when you first start growing around it. It is still standing when the new one is carrying the weight. It disappears later, quietly, once the risk is gone.

*Next in this series: the shared database, the hardest constraint.*

---

**Knowledge Debt Series:** [1](./knowledge-debt-1.md) · [2](./knowledge-debt-2.md) · [3](./knowledge-debt-3.md) · [4](./knowledge-debt-4.md) · Part 5 · [6](./knowledge-debt-6.md) · [7](./knowledge-debt-7.md) · [8](./knowledge-debt-8.md)
