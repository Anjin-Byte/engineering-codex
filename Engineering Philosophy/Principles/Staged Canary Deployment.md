---
title: Staged Canary Deployment
tags: [principle, delivery, sre, philosophy]
summary: Default the deployment of any high-consequence change to a partial, time-limited rollout against a control population, with auto-rollback on SLO regression — because the alternative leaves no observation window.
keywords: [canary, progressive rollout, blue green, control population, slo gating, rollback, release pipeline]
---

*Roll high-consequence changes out partially, for a time, against a control population. Promote on signal, roll back on regression — automatically.*

# Staged Canary Deployment

> **Rule:** any change whose failure would matter goes to production in stages — a small fraction of traffic, for a bounded time, against a population whose behavior is observable against a control. Promotion is automatic on health; rollback is automatic on regression.

A staged canary deployment is not just "deploy to a few servers first." It is a controlled experiment where the new version receives a fraction of the relevant traffic, the old version receives the rest as a control, both are observed against the same set of metrics over the same window, and the promotion or rollback decision is gated on a *quantitative* health comparison, not a vibe.

## Why the alternative is structurally worse

The two alternatives:

- **All-at-once deployment.** The new version becomes 100% of traffic instantly. If it has a problem the test suite missed, the blast radius is total. There is no observation window where the impact is bounded; by the time the problem is detected, everyone has hit it.
- **All-at-once with manual rollback.** Same blast radius, but humans can revert. The detection-to-rollback gap is whatever it takes humans to notice, decide, and execute. In practice this is many minutes of full-impact incident, plus whatever cleanup the rollback can't undo.

A canary collapses both gaps. Detection is automated against the control. Rollback is automated against the gate. The blast radius is bounded by the canary's traffic share until the gate decides.

## What a real canary requires

A canary that promotes on a stale metric or on "nothing exploded for an hour" is not earning the protection. The minimum:

- **A control population.** Old version receives traffic in parallel. Without a control, you can't distinguish "the canary is bad" from "the world changed."
- **Defined health metrics.** The same metrics that define [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]] gate the canary: error rate, latency percentile, success rate, runtime-health signals (event-loop pressure, GC pause, etc.). Defined *before* the canary starts.
- **A statistical comparison, not a threshold.** Compare canary metrics to control metrics over the same window. "Canary error rate is within X% of control" is a test; "canary error rate is below Y" is a hope.
- **A bounded observation window.** Long enough to detect problems that don't manifest immediately (memory leaks, traffic-pattern-dependent bugs); short enough that the canary doesn't become "the new state of the world by inertia."
- **Automatic rollback on regression.** Manual rollback defeats the purpose. The canary's gate is the policy; the policy enforces itself.
- **Promotion in stages.** A canary at 1% → 10% → 50% → 100% gives multiple decision points. A binary canary (1% or 100%) is barely better than no canary.

## When canaries are insufficient

The canary pattern assumes the change can be observed at the traffic-fraction level and rolled back without losing data. Some changes break that assumption:

- **Schema migrations.** Once the schema is altered, rolling back requires reverse-migrating data. Canaries help with the *code* that uses the schema; they don't help with the schema change itself. Use expand/contract: deploy the new schema as a superset (canary the writers and readers separately), then later drop the old.
- **Irreversible side effects.** A canary that sends emails or charges credit cards has already done the irreversible thing on its 1%. Rollback is rolling forward with apology. Treat these paths as needing a stronger pre-deploy gate (more testing, more review) and a smaller-than-usual canary fraction.
- **Stateful long-running connections.** WebSockets, long-polling, persistent jobs. The canary may not see the failure mode until connections cycle. Adjust the observation window or use a shadow-traffic pattern instead.
- **Changes whose impact is invisible at the request level.** A change to retry logic whose impact only appears under saturation. A canary at 1% won't reveal a problem that needs 80% load to manifest. Combine with load testing.

In each case the principle (bounded blast radius, observable impact, automatic gating) still applies; the mechanism shifts.

## Connection to other principles

- **Error budgets.** [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]] gates the canary. Outside SLO → canary fails → rollback → no feature releases until budget recovers.
- **Configuration as deployment.** [[Engineering Philosophy/Principles/Configuration as Code Review]] applies the canary discipline to config flips, not just code releases.
- **Capability slices.** Feature-flag-gated rollouts are a kind of canary. [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]] decides what's a real flag vs an internal switch.
- **Risk layering.** Canary deployment is the deployment-layer control in [[Engineering Philosophy/Principles/Layered Risk Categories]]. It does not replace controls at other layers.

## When the policy should *not* canary

A useful counter-rule: changes whose failure mode is detectable instantly and whose blast radius is genuinely tiny do not need a canary. A typo fix in a log message; a tooling change with no production effect; a doc string. A rule that demanded canaries for these would be ceremony.

The canary line is roughly: *if this change fails badly, would I want it on 1% before it's on 100%?* For most non-trivial changes, yes.

## Related

- [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]]
- [[Engineering Philosophy/Principles/Configuration as Code Review]]
- [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]]
- [[Engineering Philosophy/Principles/Layered Risk Categories]]
- [[Engineering Philosophy/Principles/Regression Discipline]]
