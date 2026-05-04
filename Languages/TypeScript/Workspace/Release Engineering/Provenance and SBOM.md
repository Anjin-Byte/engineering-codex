---
title: Provenance and SBOM
tags: [release-engineering, typescript, npm, supply-chain, provenance, sbom]
summary: Critical-path packages publish with npm provenance attestations and emit an SBOM. CycloneDX is operational/security-first; SPDX is compliance/licensing-first with ISO standing — many teams emit both.
keywords: [npm provenance, sigstore, attestation, cyclonedx, spdx, sbom, openssf scorecard, supply chain]
---

*Provenance answers "is this artifact what the build said it was?" SBOM answers "what's in it?" Both are written down, not assumed.*

# Provenance and SBOM

> **Rule:** every published critical-path package emits an [npm provenance attestation](https://docs.npmjs.com/generating-provenance-statements) signed by Sigstore from a public CI workflow, and produces a Software Bill of Materials in CycloneDX, SPDX, or both. Provenance proves the artifact came from the workflow it claims to; SBOM enumerates what's inside it. Neither replaces the other.

The supply-chain failure modes these address:

1. **Compromised publisher.** A maintainer's npm token is stolen; an attacker publishes a malicious version. Provenance breaks because the package is not built from the public workflow.
2. **Compromised build environment.** The build server is compromised; the artifact differs from what the source code would produce. Provenance cryptographically links the artifact to the workflow run; tampering breaks the link.
3. **Vulnerability response gap.** A CVE is published for a transitive dependency. Without an SBOM, the operator can't quickly answer "do we use that?"
4. **Compliance audit.** A customer or auditor asks "what's in this?" SBOM is the documented answer.

Each of these is a risk the principle covers; none is a risk a single mechanism alone covers.

## Provenance attestations

npm's provenance feature (and analogous mechanisms in other registries) attaches a Sigstore-signed attestation to the publish record that links the artifact to:

- The CI workflow file path and run ID that produced it.
- The git commit SHA the build used.
- The repository URL.
- The build environment (GitHub Actions, GitLab CI, etc.).

Consumers verify the attestation to confirm the artifact actually came from the public workflow it claims to. The keyless Sigstore signature uses the workflow's OIDC identity, so there is no maintainer-held signing key to lose or compromise.

### When to enable provenance

For any package that:

- Is published to a public registry (npm, GitHub Packages).
- Has consumers who care where it came from (most do, even if they haven't said so).
- Builds from a public CI workflow.

The mechanism is a one-line change in the publish workflow:

```yaml
- run: npm publish --provenance --access public
```

with `id-token: write` permissions on the workflow. The cost is negligible; the benefit is that consumers (or auditors) can verify the artifact's origin.

### When provenance is not the answer

For internal packages distributed within a single organization, provenance is overkill if the registry itself is internal and authenticated. The supply-chain risk is different — internal compromise has different vectors than public-registry compromise.

For binaries (services deployed as containers, not npm packages), provenance is a different mechanism — typically [SLSA](https://slsa.dev/) attestations on the container image — but the principle is the same.

## SBOM formats: CycloneDX and SPDX

Two open SBOM standards dominate. They have different design priorities, and a project's choice (or both) reflects what it cares about most.

### CycloneDX

[CycloneDX](https://cyclonedx.org/) is OWASP-stewarded. Its data model is optimized for cyber-risk and operational use:

- Vulnerability disclosure (VEX — Vulnerability Exploitability eXchange) is first-class.
- Tooling integrates with SCA workflows.
- Format versions track the security ecosystem.

**Pick CycloneDX when:** the SBOM's primary consumer is the security organization, and the use case is "what vulnerabilities apply to what's in the artifact, and what's our exposure?"

### SPDX

[SPDX](https://spdx.dev/) is Linux-Foundation-stewarded. It is published as ISO/IEC 5962:2021. Its data model is optimized for compliance and licensing:

- License-metadata is first-class (every component's license expression).
- Long-form provenance metadata (creator, supplier, originator, downloader) is structured.
- ISO ratification gives it the strongest standing for cross-organization compliance exchange.

**Pick SPDX when:** the SBOM's primary consumer is the compliance / legal organization, and the use case is "what licenses apply to what's in the artifact, and is our distribution compliant?"

### Why many teams emit both

The two formats are complementary, not competing. CycloneDX answers operational security questions; SPDX answers compliance questions. The marginal cost of generating both is small (the same dependency tree feeds both formats), and downstream consumers may have a strong preference. Emitting both removes the consumer-format guess.

## Generating SBOMs in the pipeline

A typical release-gate addition:

```
Build artifact
  │
  ├─ Generate CycloneDX SBOM (cyclonedx-npm or syft)
  ├─ Generate SPDX SBOM (syft)
  ├─ Sign / attach SBOMs to the artifact (or upload separately)
  │
  ▼
Publish artifact + attestation + SBOMs
```

Tools to consider:

- **`syft`** (Anchore) — produces both CycloneDX and SPDX from a wide range of inputs (npm trees, container images, source dirs).
- **`cyclonedx-npm`** — npm-specific CycloneDX generator.
- **GitHub-native** dependency-graph SBOMs — auto-generated for repos; useful but less rich than tool-generated SBOMs.

The choice is operational; the principle is "SBOMs exist and are kept current with each release."

## OpenSSF Scorecard for inbound dependencies

The mirror image of provenance/SBOM (which describe the artifact you ship) is the question of what artifacts you *consume*. [OpenSSF Scorecard](https://scorecard.dev/) scores a repository against ~18 security-hygiene heuristics:

- Branch protection
- Signed releases
- Pinned dependencies
- Fuzzing
- SAST
- Code review enforcement
- License presence
- Maintenance signals (recent commits, recent releases)

For each non-trivial new dependency a project pulls in, looking up the Scorecard signal is a cheap risk-assessment step. A 2/10 Scorecard for a critical-path dependency is worth pausing on; a 9/10 isn't a guarantee but is a defensible signal.

This is upstream of [[Languages/TypeScript/Workspace/Release Engineering/npm Lockfile and Install Discipline]]: dependency-review at PR time can include "Scorecard signal for new packages."

## Composing with other practices

- [[Languages/TypeScript/Workspace/Release Engineering/Release Gate Pipeline]] — SBOM generation and provenance attestation are pipeline stages.
- [[Languages/TypeScript/Workspace/Release Engineering/npm Lockfile and Install Discipline]] — the lockfile is the SBOM's source of truth.
- [[Engineering Philosophy/Principles/Configuration as Code Review]] — the SBOM is a release artifact that travels with the code; the discipline applies.
- [[Engineering Philosophy/Principles/Layered Risk Categories]] — supply-chain layer; this is most of what mitigates it.

## Related

- [[Languages/TypeScript/Workspace/Release Engineering/Release Gate Pipeline]]
- [[Languages/TypeScript/Workspace/Release Engineering/npm Lockfile and Install Discipline]]
- [[Languages/TypeScript/Workspace/Release Engineering/Permission Model Adoption]]
- [[Engineering Philosophy/Principles/Layered Risk Categories]]
