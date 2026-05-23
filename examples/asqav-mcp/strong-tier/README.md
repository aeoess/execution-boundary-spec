# Strong tier worked example: asqav-mcp `enforced_tool_call`

This directory holds a paired payload example for the bilateral-receipt
requirement defined in [spec §4.1 Strong Tier](../../../spec.md#41-strong-tier).
Both receipts are produced by [asqav-mcp](https://github.com/jagmarques/asqav-mcp)
inside the atomic execution boundary of `enforced_tool_call`. They share an
`action_ref` (the `gate_id`) so verifiers can match permit to outcome.

- [`permit-receipt.json`](./permit-receipt.json) - pre-execution evaluator signature; produced by `gate_action` / `enforced_tool_call`. Carries the §4.1 MUST fields: `action_ref`, `verdict`, `scope_evaluated`, `delegation_chain_hash`, `evaluator_signature`.
- [`outcome-receipt.json`](./outcome-receipt.json) - post-execution executor signature with `output_hash`; produced by `complete_action`. Carries the §4.1 MUST fields: `action_ref` (matching the permit), `execution_result`, `executor_signature`, `receipt_hash`.

The signing algorithm is ML-DSA-65 (NIST FIPS 204). An asqav-mcp implementation
that satisfies §4.1 may also use Ed25519; the field shape and linkage pattern
shown here are algorithm-agnostic.

## Status: illustrative / non-normative

These example receipts are structurally conformant placeholders. The signature
and hash values show the expected fields and linkage pattern; they are not live
verification artifacts. The `verification_url` entries point at the asqav cloud
endpoint shape but the specific signature ids will return `404` because no
reproducible generator is wired up yet. A future revision under
`examples/asqav-mcp/strong-tier/` will swap these placeholders for receipts
emitted by a runnable generator with real signed bytes; until then, treat the
files as illustrative of `§4.1` field names, ordering, and bilateral linkage.
