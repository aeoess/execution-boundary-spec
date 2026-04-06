# Execution Boundary Specification

Normative requirements for execution boundaries in agent orchestration systems. Defines four invariants that prevent exploitable gaps between policy intent and mutation outcome.

## Status

**Draft 0.1** - Sections 1-3 authored by AEOESS. Section 4 pending co-author QueBallSharken.

## Sections

| Section | Title | Author | Status |
|---------|-------|--------|--------|
| 1 | Mutation Authority Separation | AEOESS | Draft |
| 2 | Boundary Integrity at Commit | AEOESS | Draft |
| 3 | Continuity / Anti-Interleaving | AEOESS | Draft |
| 4 | Enforceability Classification | QueBallSharken | Placeholder |

## Reference Implementation

[Agent Passport System](https://www.npmjs.com/package/agent-passport-system) (npm: `agent-passport-system`)

## Context

This specification was developed in response to [MITRE ATLAS atlas-data#11](https://github.com/mitre-atlas/atlas-data/issues/11).

## License

Apache-2.0
