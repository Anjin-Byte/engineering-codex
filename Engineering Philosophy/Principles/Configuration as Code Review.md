---
title: Configuration as Code Review
tags: [principle, delivery, configuration, philosophy]
summary: Configuration changes deserve the same review rigor, rollback discipline, and audit trail as code changes — a runtime config flip with no review is a deployment with no review.
keywords: [feature flags, gitops, config drift, audit trail, change control, release governance]
---

*A config change that bypasses review is a deployment that bypassed review. The runtime doesn't know the difference, and neither should the release pipeline.*

# Configuration as Code Review

> **Rule:** any change that alters production behavior — whether it edits a function or flips a flag — goes through the same review and audit pipeline. There is no console where production behavior changes without a reviewer's approval and a rollback path.

Production behavior is determined by the union of code and configuration. A feature flag that switches `routing.mode` from `legacy` to `experimental` changes behavior just as decisively as a code commit. A connection-pool tuning value that drops max-connections from 100 to 10 can take down a service. A schema-registry entry that promotes a new contract is a release.

Yet teams routinely treat code as a reviewed, version-controlled artifact and configuration as a console-clickable convenience. The asymmetry is a structural bug.

## The parity rule

The standard for any production-affecting change:

- It is **stored in version control** (or a system with equivalent audit and rollback).
- It is **reviewed** by someone other than the author, with the same scrutiny code receives. The review checks intent, blast radius, rollback strategy, and observability of the result.
- It is **deployed by the release pipeline**, not by ad-hoc tooling. The pipeline applies the same gates (canary, SLO watch, automatic rollback) that code deployment uses. See [[Engineering Philosophy/Principles/Staged Canary Deployment]].
- It has **a rollback procedure tested in advance**, not invented mid-incident.
- It produces an **audit record** linking the change to a reviewer, a timestamp, and a prior state.

Any "change" that lacks one of these is, by this rule, not allowed in production at the level of the impacted assurance tier.

## What "configuration" includes

The category is broader than runtime config files. The parity rule applies to:

- **Feature flags** — release flags, kill switches, permanent control flags. Each carries an owner, an expiry (when applicable), a default, and a rollback. See [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]] for the design rule on what should be a flag in the first place.
- **Infrastructure config** — load balancer rules, autoscaler bounds, DNS, network ACLs, IAM policies. These look like infra; they are deployment.
- **Runtime tuning** — connection pool sizes, timeouts, retry counts, GC heap limits. The wrong number here is an outage.
- **Deployed schemas and contracts** — a registered schema, a public API contract version, a rate-limit policy. Promoting one is a release.
- **Secrets rotation policy** — when secrets are rotated, who has them, who is alerted. Operationally critical and easy to leak.
- **Deployed feature flags' default values** — the default *is* code that runs when the flag service is unreachable.

Each of these has been the root cause of high-profile outages because it was treated as "just config" and bypassed normal change control.

## The "click-ops" failure mode

A common pattern: the code path is exemplary — version-controlled, reviewed, gated, observable, rolled back automatically. But there's a console somewhere (a feature-flag UI, a database admin panel, a load-balancer page) where a single human can change production behavior in a click, without review, without audit, without rollback.

This isn't an exception to the rule; it's the failure mode the rule exists to prevent. The fix is structural:

- Disable direct console writes for any change that affects production. The console becomes read-only.
- Route changes through a pull-request-shaped flow: propose → review → apply via pipeline.
- Where the underlying system requires API access (rather than a config file), the API call lives in version control as a configuration artifact, not in someone's command history.

## Rollback as a first-class operation

The rollback path matters more than the rollout path. Specifically:

- **Tested.** The rollback is exercised in non-production at least once before the change is in production. An untested rollback is a hope, not a procedure.
- **Documented at the time of review.** The rollback procedure goes in the change description, not in a separate document that may not exist.
- **Time-bounded.** "We will need 4 hours to roll back" is a critical fact for the on-call. If the rollback takes longer than the incident-response budget, the change is too risky.
- **Observable.** The rollback's success is visible in the same metrics the rollout's success was. If rollout health is gauged on metric X, rollback health is too.

## When the rule is honored only on code

If your team's deployment pipeline is excellent for code and absent for configuration, you have one of these two outcomes:

1. **Configuration is rarely changed**, in which case the inconsistency is mostly hidden — until the day it's needed.
2. **Configuration is the easy way to change behavior**, in which case the team will route around code review by preferring config changes. The code-review pipeline becomes ceremony; the config console is the real release pipeline.

Both are bad. The fix is the parity rule.

## Composing with other principles

- [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]] decides *what* belongs as a flag; this note decides *how* flags are governed.
- [[Engineering Philosophy/Principles/Staged Canary Deployment]] is the deployment-pipeline expression for both code and config.
- [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]] gates flips on error-budget health, the same way it gates feature releases.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] places this rule at the deployment layer.

## Related

- [[Engineering Philosophy/Principles/Capability Slices vs Implementation Switches]]
- [[Engineering Philosophy/Principles/Staged Canary Deployment]]
- [[Engineering Philosophy/Principles/Error Budgets and Reliability Policy]]
- [[Engineering Philosophy/Principles/Layered Risk Categories]]
