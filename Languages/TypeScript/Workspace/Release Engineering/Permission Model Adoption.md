---
title: Permission Model Adoption
tags: [release-engineering, typescript, node, permission-model, security]
summary: Node's `--permission` flag restricts filesystem, network, child-process, worker, native-addon, WASI, and inspector access at the process level. Stable as of Node v22.13.0 / v23.5.0; defense in depth, not a sandbox against malicious code.
keywords: [node permission model, --permission, --allow-fs-read, --allow-net, defense in depth, seat belt, runtime hardening]
---

*The Permission Model is a seat belt. It catches accidents that would otherwise be fatal; it is not armor against a deliberate attacker.*

# Permission Model Adoption

> **Rule:** for production Node services that run with `--permission`, the operational model is "default-deny, allowlist what the service actually needs." The Permission Model is defense in depth — Node explicitly says it is not a sandbox against malicious code, and adopters should treat it as one layer among several, not a substitute for the others.

The [Permission Model](https://nodejs.org/api/permissions.html) is stable in Node v22.13.0 / v23.5.0. The mechanism: a process started with `--permission` runs with no access to the filesystem, network, child processes, workers, native addons, WASI subsystems, or the inspector. Each subsystem is then granted via flags:

```sh
node --permission \
     --allow-fs-read=/app/config,/etc/ssl \
     --allow-fs-write=/var/log/app \
     --allow-child-process \
     index.js
```

The service starts under the union of allowed permissions; everything else is refused at the syscall boundary, with a `ERR_ACCESS_DENIED` error.

## Why "defense in depth, not sandbox"

The Permission Model docs are explicit about what it is not:

> The Node.js Permission Model provides a mechanism for restricting access to specific resources during execution. The API exists behind a flag --permission which when enabled, will restrict access to all available permissions. **The Permission Model is not a security sandbox.**

The framing matters. A real sandbox (V8 isolate, a container with seccomp, a hardware-isolated process) is designed to contain *malicious* code. The Permission Model is designed to contain *accidental* code paths — a misconfigured logger that suddenly tries to write `/etc/passwd`, a dependency update that adds an unexpected outbound network call, a debug feature that accidentally lands in production.

Treat it accordingly. It composes with other defenses; it doesn't replace them.

## What it catches

Useful failure modes the Permission Model surfaces:

- **A dependency update that adds outbound network calls** the service didn't previously make. The added calls fail with `ERR_ACCESS_DENIED`; the operator notices on next deploy.
- **A logging library that quietly writes to `/tmp` or `~/.config`.** Restricted to `/var/log/app`, the unintended writes fail and surface in the error log.
- **A feature flag flipped wrongly that enables a debug `child_process.spawn`.** Without `--allow-child-process`, the spawn fails fast.
- **A bug that makes the service exfiltrate data via DNS lookups.** Without `--allow-net`, the lookups fail.

In each case the model converts a silent unexpected behavior into a hard failure. The operator sees the error rather than the wrong behavior.

## What it does *not* catch

Important non-claims:

- **Malicious code already running with privileges.** Code with `--allow-fs-write=/var/log/app` can write whatever it wants to `/var/log/app`; the Permission Model doesn't validate *content*.
- **Bugs in the runtime itself.** A V8 vulnerability that allows arbitrary code execution bypasses the Permission Model.
- **Process-level attacks.** `--permission` doesn't change how a deliberate attacker who already controls the process behaves.
- **Native addons that bypass the model.** The model can refuse to *load* native addons; it can't sandbox what they do once loaded if you've allowed them.

For threats in those categories, the answer is OS-level isolation (containers + seccomp), language isolation (V8 isolates, worker contexts), or hardware isolation (separate processes, separate hosts). The Permission Model is layered alongside those, not in place of them.

## Adoption strategy

Two approaches:

### Greenfield service

Start with `--permission` and an empty allowlist. Run the service in dev / staging; it will fail repeatedly until the allowlist is complete. Each failure is a documentation step — every entry in the allowlist names a real reason the service needs that access.

The allowlist becomes a security artifact: a list of the resources this service touches, in one file, reviewable.

### Existing service

Audit what the service actually accesses (use `strace`, eBPF, or a logging shim) to produce a candidate allowlist. Run with `--permission` plus the allowlist in non-production; observe failures; refine the allowlist; deploy gradually with canary monitoring (see [[Engineering Philosophy/Principles/Staged Canary Deployment]]) since some access patterns may not surface in non-production.

A common pitfall: the allowlist is initially too broad ("just allow everything to ship"), and never narrowed. Treat the broad allowlist as a starting point, with a follow-up commitment to narrow it. The narrow version is the goal; the broad version should not be the long-term state.

## Permissions and other release-engineering controls

Permission Model composes with the rest of the release pipeline:

- **Provenance and SBOM** (see [[Languages/TypeScript/Workspace/Release Engineering/Provenance and SBOM]]) tell consumers what's in the artifact; Permission Model restricts what the running artifact can do.
- **`npm audit`** (see [[Languages/TypeScript/Workspace/Release Engineering/npm Lockfile and Install Discipline]]) catches known vulnerabilities in deps; Permission Model limits the blast radius of *unknown* ones.
- **OpenTelemetry** observability (see [[Languages/TypeScript/Practices/Async and Concurrency/Async Context Propagation]]) makes Permission-Model failures visible in operational dashboards.

Each layer is independent. Skipping any of them shifts more responsibility to the others.

## When *not* to adopt yet

Reasons to defer:

- **The dependency tree includes packages that the team hasn't audited for permission compatibility.** Some popular packages assume unrestricted FS / network access; running them under `--permission` produces failures that take research to resolve.
- **The deployment platform doesn't expose the flag straightforwardly.** Serverless platforms (Lambda, Cloudflare Workers) have their own runtime constraints; check whether they surface `--permission` cleanly.
- **The team's debugging workflow depends on the inspector or `child_process` in ways that conflict with a tight allowlist.**

In each case, the deferral is a known trade-off — adopt other defenses now, plan Permission Model adoption when the blockers clear.

## Composing with other practices

- [[Engineering Philosophy/Principles/Layered Risk Categories]] — runtime layer; Permission Model is one of the few hard runtime-layer controls Node provides.
- [[Languages/TypeScript/Workspace/Release Engineering/Release Gate Pipeline]] — Permission Model adoption appears in the deploy step, not the build step.
- [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]] — services that fit Node also fit the Permission Model; advanced runtimes (Bun, Deno) have their own permission systems.

## Related

- [[Languages/TypeScript/Workspace/Release Engineering/Release Gate Pipeline]]
- [[Languages/TypeScript/Workspace/Release Engineering/npm Lockfile and Install Discipline]]
- [[Languages/TypeScript/Workspace/Release Engineering/Provenance and SBOM]]
- [[Engineering Philosophy/Principles/Layered Risk Categories]]
- [[Languages/TypeScript/Practices/Foundational/Runtime Suitability Boundary]]
