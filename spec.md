# Execution Boundary Specification

**Draft 0.1** | 2026-04-06

**Authors:** AEOESS (Tymofii Pidlisnyi), QueBallSharken

**Status:** Draft for review

**License:** Apache-2.0

---

## Abstract

This specification defines normative requirements for execution boundaries in agent orchestration systems. An execution boundary is the trust perimeter within which an irreversible action is evaluated, authorized, and committed. The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

Four invariants are defined. Violations of any invariant create exploitable gaps between policy intent and mutation outcome.

---

## 1. Mutation Authority Separation

### 1.1 Normative Requirement

The component that executes the irreversible primitive MUST be the same component that evaluates admissibility. Separation of the evaluator from the executor creates a time-of-check-to-time-of-use (TOCTOU) window in which authorization state can change between the policy decision and the mutation.

### 1.2 Rationale

When evaluation and execution are co-located in a single atomic boundary, the authorization decision and the mutation are indivisible. No external event can alter delegation state, revocation status, or constraint parameters between "policy says yes" and "action fires." Separating these components introduces a temporal gap that attackers can exploit through delegation revocation racing, constraint parameter manipulation, or scope escalation.

### 1.3 Reference Implementation

In the Agent Passport System (APS), `ProxyGateway.processToolCall()` is the atomic boundary. The gateway:

1. Receives a signed tool call request from the agent
2. Evaluates all 15 constraint facets (identity, scope, revocation, time, spend, merchant, autonomy, risk, geographic, purpose, consent, data-lifecycle, fidelity, freshness, replay)
3. Executes the tool
4. Produces a 3-signature proof chain (request, decision, receipt)

All four steps occur within a single synchronous execution context. The receipt is signed by the gateway, not the requesting agent.

**Test reference:** `tests/gateway.test.ts` - "ProxyGateway - Core Flow", "Property 1: Gateway is Executor", "Property 4: Gateway Signs Receipt"

### 1.4 Test Vectors

See [Appendix B, Vectors 1.1-1.3](#appendix-b-test-vectors).

---

## 2. Boundary Integrity at Commit

### 2.1 Normative Requirement

The authorization envelope MUST be re-derived from live state at the moment of execution. Cached authorization is NOT admissible at the mutation boundary. An approval token or cached policy decision that was valid at time T MUST be re-validated against current state at execution time T+n.

### 2.2 Rationale

Agent orchestration systems frequently implement a two-phase pattern: approve first, execute later. If the execution phase trusts the cached approval without re-checking live state, any state change between phases (delegation revocation, scope narrowing, budget exhaustion) is invisible to the executor. The compound digest of all authorization inputs MUST be re-derived, not recalled.

### 2.3 Reference Implementation

In APS, the gateway implements `recheckRevocationOnExecute: true` by default. The two-phase flow (`gateway.approve()` then `gateway.executeApproval()`) re-validates delegation state at execution time. If the delegation is revoked between approval and execution, the execution is denied.

The `AuthorizationWitness` captures the full constraint evaluation at execution time, cryptographically linked to the receipt via `AuthorizationRef`. This provides forensic evidence that live state was checked at the mutation boundary.

**Test reference:** `tests/gateway.test.ts` - "Property 3: Revocation Recheck", `tests/gateway-constraints.test.ts` - "Authorization Witness", "Forensic Linkage"

### 2.4 Test Vectors

See [Appendix B, Vectors 2.1-2.3](#appendix-b-test-vectors).

---

## 3. Continuity / Anti-Interleaving

### 3.1 Normative Requirement

The signed receipt MUST be produced inside the atomic execution boundary. No temporal gap between "policy permits" and "action fires" is permissible. The execution boundary MUST prevent interleaving of concurrent requests that would allow one action to consume authorization meant for another.

### 3.2 Rationale

If receipts are generated outside the execution boundary, they can be forged, replayed, or associated with actions that did not actually execute under the recorded constraints. Anti-interleaving prevents a class of attacks where an adversary submits rapid concurrent requests to exploit race conditions in authorization consumption.

### 3.3 Reference Implementation

In APS, replay protection is enforced inside the execution boundary:

1. Each request carries a unique `requestId`
2. The gateway records consumed request IDs atomically during execution
3. Duplicate request IDs are rejected with a structured `ConstraintFailure` (facet: replay)
4. Two-phase approvals are single-use: executing an already-consumed approval is denied

The `ExecutionEnvelope` bundles intent, decision, receipt, and delegation reference into a single signed artifact, proving all components were produced atomically.

**Test reference:** `tests/gateway.test.ts` - "Property 5: Replay Protection", `tests/execution-envelope.test.ts` - "Execution Envelope"

### 3.4 Test Vectors

See [Appendix B, Vectors 3.1-3.3](#appendix-b-test-vectors).

---

## 4. Enforceability Classification

> **Placeholder:** This section will be authored by QueBallSharken.

Three tiers:
- **Strong:** Mutation authority co-located with evaluator (atomic path)
- **Bounded:** Version-anchored envelope re-verified at commit
- **Detectable-only:** Post-hoc behavioral anomaly detection

**Normative:** Implementations MUST declare their enforceability tier.

---

## Appendix A: Reference Implementation

The Agent Passport System (APS) SDK provides a reference implementation of all four invariants.

- **ProxyGateway:** `src/core/gateway.ts` - atomic execution boundary with 15 constraint facets
- **Constraint Architecture:** `src/types/gateway.ts` - `ConstraintFacet`, `ConstraintVector`, `AuthorizationWitness`
- **Execution Envelope:** `src/core/execution-envelope.ts` - signed atomic bundle
- **Test Suite:** `tests/gateway.test.ts`, `tests/gateway-constraints.test.ts`, `tests/transactional-integrity.test.ts`, `tests/execution-envelope.test.ts`

Published as `agent-passport-system` on npm.

---

## Appendix B: Test Vectors

### Vector 1.1: Atomic Evaluation-Execution (PASS)

The evaluator and executor are the same component. A valid tool call produces a complete proof chain.

```json
{
  "id": "TV-1.1",
  "title": "Atomic evaluation-execution produces proof chain",
  "input": {
    "request": {
      "requestId": "req-001",
      "agentId": "agent-alice",
      "tool": "api:fetch",
      "params": { "url": "https://example.com/data" },
      "scope": "data:read",
      "requestSignature": "<ed25519-signature>"
    },
    "delegation": {
      "delegatorId": "principal-bob",
      "delegatedTo": "agent-alice",
      "scope": ["data:read"],
      "status": "active"
    }
  },
  "expected": {
    "executed": true,
    "proof": {
      "requestSignature": "<present>",
      "decisionSignature": "<present>",
      "receiptSignature": "<present>",
      "policyReceipt": { "policyReceiptId": "<present>" }
    },
    "receipt": {
      "agentId": "<gateway-id, NOT agent-alice>"
    }
  },
  "rationale": "Receipt is signed by the gateway (executor), not the requesting agent. All three signatures are produced within the same execution context."
}
```

### Vector 1.2: Separated Evaluator-Executor (FAIL)

When evaluation and execution are separated, a TOCTOU window exists.

```json
{
  "id": "TV-1.2",
  "title": "Separated evaluation creates TOCTOU vulnerability",
  "input": {
    "phase1_evaluate": {
      "timestamp": "2026-04-06T12:00:00Z",
      "delegation_status": "active",
      "decision": "permit"
    },
    "state_change": {
      "timestamp": "2026-04-06T12:00:01Z",
      "action": "delegation_revoked"
    },
    "phase2_execute": {
      "timestamp": "2026-04-06T12:00:02Z",
      "uses_cached_decision": true
    }
  },
  "expected": {
    "vulnerability": "TOCTOU",
    "description": "Execution proceeds under revoked delegation because the executor trusts the cached evaluation from T+0, unaware of revocation at T+1"
  },
  "rationale": "Demonstrates why evaluation and execution MUST NOT be separated. The 1-second gap between check and use is exploitable."
}
```

### Vector 1.3: Gateway-Owned Execution on Tool Error (PASS)

The gateway maintains authority even when the tool itself errors.

```json
{
  "id": "TV-1.3",
  "title": "Gateway maintains receipt integrity on tool error",
  "input": {
    "request": {
      "requestId": "req-002",
      "agentId": "agent-alice",
      "tool": "api:fetch",
      "params": { "url": "https://down.example.com" }
    },
    "tool_behavior": "throws Error('Connection refused')"
  },
  "expected": {
    "executed": true,
    "result": { "error": "Connection refused" },
    "receipt": "<present, signed by gateway>",
    "proof": "<complete chain>"
  },
  "rationale": "The gateway, not the agent, handles tool errors. The receipt records what happened, maintaining the audit trail even on failure."
}
```

### Vector 2.1: Live State Re-derivation at Commit (PASS)

Delegation revoked after registration but before execution is correctly denied.

```json
{
  "id": "TV-2.1",
  "title": "Revoked delegation denied at execution time",
  "input": {
    "sequence": [
      { "step": 1, "action": "register_agent", "delegation_status": "active" },
      { "step": 2, "action": "revoke_delegation", "reason": "Compromised" },
      { "step": 3, "action": "process_tool_call", "tool": "api:fetch" }
    ]
  },
  "expected": {
    "executed": false,
    "denialReason": "delegation revoked"
  },
  "rationale": "Registration cached the delegation as active, but execution re-checks live state and finds it revoked."
}
```

### Vector 2.2: Two-Phase Stale Approval Rejected (PASS)

Approval succeeds, then delegation is revoked, then execution is denied.

```json
{
  "id": "TV-2.2",
  "title": "Two-phase: approval valid, delegation revoked, execution denied",
  "input": {
    "sequence": [
      { "step": 1, "action": "approve", "result": "approved=true" },
      { "step": 2, "action": "revoke_delegation" },
      { "step": 3, "action": "execute_approval" }
    ]
  },
  "expected": {
    "executed": false,
    "denialReason": "removed since approval|invalidated"
  },
  "rationale": "The approval token was valid when issued. The execution phase re-derives authorization from live state and finds the delegation invalidated."
}
```

### Vector 2.3: Authorization Witness at Execution (PASS)

Successful execution produces an AuthorizationWitness capturing live constraint evaluation.

```json
{
  "id": "TV-2.3",
  "title": "AuthorizationWitness captures live state at execution",
  "input": {
    "request": {
      "requestId": "req-003",
      "agentId": "agent-alice",
      "tool": "api:fetch",
      "scope": "data:read"
    }
  },
  "expected": {
    "executed": true,
    "receipt": {
      "authorizationRef": {
        "witnessHash": "<sha256>",
        "constraintVector": {
          "facets_evaluated": ["identity", "scope", "revocation", "time", "spend", "replay"]
        }
      }
    }
  },
  "rationale": "The AuthorizationWitness is forensic evidence that all constraint facets were evaluated from live state at execution time, not from a cache."
}
```

### Vector 3.1: Replay Rejection (PASS)

Duplicate request IDs are blocked atomically.

```json
{
  "id": "TV-3.1",
  "title": "Duplicate requestId blocked by replay protection",
  "input": {
    "sequence": [
      { "step": 1, "action": "process_tool_call", "requestId": "fixed-id-001" },
      { "step": 2, "action": "process_tool_call", "requestId": "fixed-id-001" }
    ]
  },
  "expected": {
    "step1": { "executed": true },
    "step2": { "executed": false, "denialReason": "replay" }
  },
  "rationale": "The request ID is consumed atomically inside the execution boundary. A second submission of the same ID is an interleaving attempt."
}
```

### Vector 3.2: Approval Single-Use Consumption (PASS)

A two-phase approval can only be executed once.

```json
{
  "id": "TV-3.2",
  "title": "Approval consumed on first execution, rejected on second",
  "input": {
    "sequence": [
      { "step": 1, "action": "approve", "result": "approved=true, approvalId=appr-001" },
      { "step": 2, "action": "execute_approval", "approvalId": "appr-001" },
      { "step": 3, "action": "execute_approval", "approvalId": "appr-001" }
    ]
  },
  "expected": {
    "step2": { "executed": true },
    "step3": { "executed": false, "denialReason": "consumed|replay" }
  },
  "rationale": "The approval nonce is consumed atomically during execution. Re-execution would interleave a second mutation under a single authorization."
}
```

### Vector 3.3: Signed Envelope Tamper Detection (PASS)

Tampering with any field in the execution envelope breaks the signature.

```json
{
  "id": "TV-3.3",
  "title": "Tampered execution envelope fails verification",
  "input": {
    "envelope": {
      "run_id": "run-001",
      "intent": { "tool": "api:fetch", "scope": "data:read" },
      "decision": { "verdict": "permit" },
      "receipt": { "result": "success" },
      "signature": "<gateway-ed25519-signature>"
    },
    "tampering": { "field": "run_id", "new_value": "run-002" }
  },
  "expected": {
    "verification": "FAILED",
    "reason": "signature mismatch"
  },
  "rationale": "The envelope bundles intent, decision, and receipt into a single signed artifact. Modification of any field proves the envelope was not produced inside the atomic boundary."
}
```

---

## Appendix C: Drift Protocol Case Study

> **Placeholder:** Will map the 4 invariants to the Drift/Solana $285M failure, showing exactly where each invariant was violated. To be completed with QueBallSharken's enforceability classification.

### Preliminary Mapping

1. **Mutation Authority Separation:** Drift's liquidation engine evaluated eligibility in a separate component from the one that executed the liquidation. State drift between oracle price feed evaluation and liquidation execution created a TOCTOU window.

2. **Boundary Integrity at Commit:** The oracle price used for liquidation was cached from a prior block. By the time the liquidation executed, the true price had shifted, but the cached authorization was not re-derived.

3. **Continuity / Anti-Interleaving:** Multiple liquidation transactions were interleaved within the same block, each consuming the same stale oracle price as authorization. No anti-replay mechanism prevented duplicate liquidations against the same position.

4. **Enforceability Classification:** The system was operating at "Detectable-only" tier (post-hoc anomaly detection) rather than "Strong" (atomic evaluation-execution). The detection mechanisms existed but could not prevent the mutations.

---

## References

- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) - Key words for use in RFCs
- [Agent Passport System](https://www.npmjs.com/package/agent-passport-system) - Reference implementation (npm)
- [MITRE ATLAS](https://atlas.mitre.org/) - Adversarial Threat Landscape for AI Systems
- Pidlisnyi, T. "The Agent Social Contract" (2026). Zenodo DOI: 10.5281/zenodo.18749779
- Pidlisnyi, T. "Faceted Authority Attenuation" (2026). Zenodo DOI: 10.5281/zenodo.19260073
