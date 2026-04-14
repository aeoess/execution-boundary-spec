# Contributing to execution-boundary-spec

Thanks for showing up here. This repo is the Execution Boundary Specification — a normative RFC 2119 document defining mutation-authority separation in agent orchestration. It's a spec, not a library. The deliverable is prose, fixture vectors, and conformance criteria.

## Quick start

**For editorial or clarity fixes** to spec prose, submit a PR directly. Small changes that improve readability without changing normative behavior are welcome.

**For normative changes** (anything changing a MUST / SHOULD / MAY), open an issue first. Normative changes affect every conformant implementation; direction needs alignment before prose.

**For fixture or test vector additions**, submit a PR with the vector and its reproducible generator.

**Submission mechanics:** fork the repo, create a feature branch from `main`, open a PR against `main`.

---

## What makes a PR mergeable

1. **Normative language is precise.** MUST / SHOULD / MAY / MUST NOT / SHOULD NOT per RFC 2119, applied consistently. Changes to normative clauses are tracked in the change log section with rationale.
2. **Fixtures are reproducible.** Any fixture included must be regenerable from the documented generator. No hand-crafted bytes.
3. **Backward compatibility explicit.** Changes that would break existing conformant implementations require a major version bump and a documented migration path.
4. **Conformance criteria update together.** If the normative clause changes, the conformance test vectors and the conformance checklist update in the same PR.

## Stability expectations

This is a versioned spec. The published version is the reference contract for implementations; changes to published versions are additive or documented as explicit breaking. Draft versions (`0.x` range) can churn without major version discipline.

## Out of scope

- **Implementation-specific feature proposals** that don't generalize across multiple implementations. This spec is normative for what implementations must do; optional per-vendor features go in each implementation's own docs.
- **Changes to referenced specs** (APS, SINT, MCP, A2A). Upstream changes go through upstream processes.
- **Claims of consensus without demonstrated multi-implementation convergence.** Single-implementation features stay marked as non-normative until a second independent implementation adopts.

---

## How review works

Every PR is evaluated against five questions, applied to every contributor equally:

1. **Identity.** Is the contributor identifiable, with a real GitHub presence?
2. **Format.** Does the change match existing patterns (RFC 2119 usage, section structure)?
3. **Substance.** Are normative claims accurate and verifiable?
4. **Scope.** Does the PR stay within spec scope, or reach into implementation-specific territory?
5. **Reversibility.** Can the change be reverted cleanly?

Substantive declines include the reason.

---

## Practical details

- **Maintainer:** [@aeoess](https://github.com/aeoess) (Tymofii Pidlisnyi)
- **Review timing:** maintainer-bandwidth dependent. If a PR has had no response after 5 business days, ping it.
- **CLA / DCO:** no CLA is required. Contributions accepted on the understanding that the submitter has the right to contribute under the Apache 2.0 license.
- **Security issues:** open a private security advisory via GitHub rather than a public issue.
- **Code of Conduct:** Contributor Covenant 2.1 — see [`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md).

---

## Licensing

Apache License 2.0 (see [`LICENSE`](./LICENSE)). By contributing, you agree that your contributions will be licensed under the same license.
