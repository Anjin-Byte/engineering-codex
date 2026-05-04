---
title: Error Budgets and Reliability Policy
tags: [principle, reliability, sre, philosophy]
summary: A service's reliability is a budget, not an aspiration — when the budget is exhausted, feature releases pause and reliability work takes priority by policy, not by negotiation.
keywords: [slo, sli, error budget, reliability policy, sre, release gating, postmortem]
---

*Reliability is a budget. When it's spent, feature releases stop and reliability work takes priority — not because someone insisted, but because the policy said so.*

# Error Budgets and Reliability Policy

> **Rule:** define the reliability target before the system ships, derive a budget from it, and write down — in advance — what happens when the budget is exhausted. Then enforce that policy automatically, not by argument.

A service-level objective (SLO) is the target reliability: 99.9% of requests complete within 200 ms, 99.95% of jobs complete without manual intervention, and so on. The **error budget** is what's left when you subtract the SLO from 100%: 0.1% of requests, 0.05% of jobs. That budget is not aspirational headroom — it is a finite resource the team is allowed to spend on changes that put reliability at risk. Once spent, the spending stops.

## Why budgets, not aspirations

"Don't break things" is an aspiration. It's not a control. It loses every argument with deadlines and feature pressure, because there's nothing concrete to point at. A budget changes the conversation: instead of "should we ship this risky change?" the question becomes "do we have budget for it?" The answer comes from a metric, not a negotiation.

Three things follow once a budget exists:

1. **Reliability becomes measurable in the same units as features.** Both spend the same finite resource (engineering time × risk of breakage). You can compare them directly.
2. **The trade-off is no longer cultural.** A team that has burned its budget can't ship more risky changes this period — not because someone got upset, but because policy says so.
3. **Postmortems become arithmetic.** A serious incident consumed N% of the quarterly budget; the team has 100% − (already-burned + N%) left. The budget tells you whether the next risky change is affordable.

## What the policy must specify in advance

A reliability policy that's invented during the incident is not a policy. The following must exist *before* the budget is at risk:

- **The SLO.** Target percentile, target value, measurement window. "99.9% of requests under 200 ms p99 over a rolling 28 days" is a concrete SLO. "Pretty fast" is not.
- **The SLI** (the indicator). The exact metric and query that produces the SLO percentage. If the team can't reproduce the number from the metric pipeline, the SLO is ceremony.
- **Burn-rate thresholds.** What rate of budget consumption triggers what response. Slow burn (small fraction of budget per day) → investigate; fast burn (entire budget in hours) → page humans.
- **The release-gate consequence.** What happens when the budget is exhausted. The default policy: feature releases pause, only reliability-improving changes ship, until the next measurement window or until the cause is resolved.
- **The exception process.** Who can override the gate, on what authority, with what audit trail. (No exception process is honest. Pretending exceptions never happen is dishonest and the gate becomes a fiction.)

## The budget is a control, not a complaint mechanism

A common failure mode: a team adopts SLOs and budgets, watches the budget burn, and never gates anything on it. The burn becomes a complaint to wave at leadership. The budget did no work.

The control is the *automatic* link from "budget exhausted" to "feature releases paused." The policy must be enforced by the release pipeline, not by humans deciding case-by-case at the moment of pressure. Humans deciding case-by-case is exactly what budgets are designed to replace.

This connects directly to [[Engineering Philosophy/Principles/Staged Canary Deployment]]: a canary that promotes despite SLO regression has bypassed the budget. The canary's promotion criterion *is* the budget enforcement.

## When the budget is a poor fit

Error budgets work well for services with high request volume, well-defined success criteria, and recoverable failures. They work poorly for:

- **Low-volume systems.** A monthly batch job has almost no statistical power; "99.9% reliable" needs years to validate.
- **Systems with non-recoverable failures.** A hospital pacemaker firmware does not have an error budget. The budget is zero.
- **Systems where every failure is a regulatory event.** Once each failure is its own outage, the SLO is structural, not statistical.

For these, the budget framing is the wrong tool. The principle behind it — make the reliability commitment explicit and the enforcement automatic — still applies, but the mechanism is different: design-time analysis, formal models, redundancy, certification. See [[Engineering Philosophy/Principles/Formal Modeling for Distributed State]] for one such mechanism.

## How this composes

- **With deployment.** [[Engineering Philosophy/Principles/Staged Canary Deployment]] is the deployment-layer expression: canary promotion is gated on SLO health.
- **With observability.** The SLO depends on an SLI that exists. If the metric isn't there, the budget is fiction.
- **With configuration.** [[Engineering Philosophy/Principles/Configuration as Code Review]] applies to the SLO definition itself: editing a target reliability number is a serious change and should go through review, not a config-console click.

## Related

- [[Engineering Philosophy/Principles/Staged Canary Deployment]]
- [[Engineering Philosophy/Principles/Configuration as Code Review]]
- [[Engineering Philosophy/Principles/Layered Risk Categories]]
- [[Engineering Philosophy/Principles/Regression Discipline]]
