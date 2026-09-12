# interpret-results.md — how to read and build on the outcomes

The verifier returns three classes of outcome. Treat each exactly as it is — never
inflate a partial into a settled confirmation.

## 1. Mined (positive)

```json
{"verified": true,
 "evidence": {"service":"soc_verify","txid":"<64-hex>","mined":true,
              "block_height":"<block>","checked_at":"<iso>","verifier":"<fp>"},
 "proof_hash":"<sha256>"}
```
**Meaning:** the txid is in a confirmed block on the chain the keeper is
authoritative for. Safe to:
- release an escrow / mark a job paid,
- attach to a receipt as "settled on-chain @block <N>",
- present to a counterparty or auditor as a non-repudiable settlement fact.

## 2. Not-mined with a REASON (honest negative — the service's core value)

```json
{"verified": false, "error": "not-mined: still-in-mempool"}
{"verified": false, "error": "not-mined: node:HTTP 404: Not Found"}
```
- `still-in-mempool` — broadcast but **not** in a confirmed block. This is the
  exact "signed ≠ settled" gap this service exists to close. Wait and re-check;
  do NOT record as settled.
- `HTTP 404` — the node's confirmed chain does not know this txid. Either never
  mined, or on a different network/lineage. Fail-closed: treat as **not proven**.

**Never** present either negative as proof the counterparty is lying about a
settlement on *another* chain — the verifier is authoritative for the chain its
keeper serves.

## 3. Malformed input

```json
{"error":"txid must be 64-hex"}
```
Input that cannot be a real txid is rejected before any chain query. This is
validation, not a verdict.

## Building on the artifact

- **Re-verifiability:** share `evidence` alone — any third party (or a future
  you, years later) can confirm `txid` landed at `block_height` with `verifier`.
- **Post-quantum-forward:** the fingerprint lineage is ML-DSA / NIST FIPS-204.
  The artifact is designed to outlive classical-crypto assumptions — that is a
  feature to state plainly, not overstate. It secures the *verification*, not the
  underlying chain's consensus.
- **As a rail primitive:** a settlement processor can call this per escrow-release
  and attach the artifact to its receipt + trust trail, converting "we recorded
  it" into "it is provably on-chain at block N."

## Honest limits (state these, never hide)

1. Scope of authority: the chain the keeper is authoritative for. Cross-chain
   claims need a verifier authoritative for that chain.
2. "Mined" is a confirmed-block fact at `checked_at`. A block could later be
   orphaned (rare, chain-specific); the artifact records the height and time it
   WAS confirmed.
3. Payment (the escrow OPEN, 0.9 SOC) is itself block-confirmed before any
   artifact is granted — so the payment proof and the verification proof share
   the same SEPTA discipline.