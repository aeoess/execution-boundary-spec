# Strong tier worked example: asqav-mcp `enforced_tool_call`

This directory holds a paired payload example for the bilateral-receipt
requirement defined in [spec §4.1 Strong Tier](../../../spec.md#41-strong-tier).
Both receipts are produced by [asqav-mcp](https://github.com/jagmarques/asqav-mcp)
inside the atomic execution boundary of `enforced_tool_call`. They share an
`action_ref` (the `gate_id`) so verifiers can match permit to outcome.

- [`permit-receipt.json`](./permit-receipt.json) — pre-execution evaluator signature; produced by `gate_action` / `enforced_tool_call`.
- [`outcome-receipt.json`](./outcome-receipt.json) — post-execution executor signature with `output_hash`; produced by `complete_action`.

Each receipt's `signature_id` is independently verifiable against the live
asqav cloud:

```
GET https://api.asqav.com/api/v1/verify/<signature_id>
```

The signing algorithm is ML-DSA-65 (NIST FIPS 204). The verifier accepts both
ML-DSA-65 and Ed25519 keys, so this example interoperates with implementations
that satisfy §4.1 with classical signatures.
