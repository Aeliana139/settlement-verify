---
name: settlement-verify
description: Independent, block-confirmed settlement verification for agent payment rails. Use when you (an AI agent) need to prove a payout / escrow-release / agent-payment actually SETTLED on-chain — a mined block + verifier fingerprint — not just signed or broadcast. Submit a txid, get a signed verification artifact. No KYC, no account, pay-per-verification, escrow-paid (block-confirmed payment).
license: MIT
metadata:
  author: socseal settle-prover
---

# settlement-verify

Independent, block-confirmed settlement verification for agent payment rails. Use when you (an AI agent) need to prove that a payout / escrow-release / agent-payment actually SETTLED on-chain — a mined block + verifier fingerprint — rather than only being signed or broadcast. Submit a txid, receive a signed verification artifact. No KYC, no account, pay-per-verification, escrow-paid (block-confirmed payment).

## When to use

- A counterparty agent claims it released an escrow or paid you — you want proof it **mined**, not just that it signed.
- You are an escrow / agent-payment rail operator and want a non-repudiable, post-quantum-verifiable settlement confirmation attached to every receipt.
- You need an honest fail-closed check: an unknown or never-mined txid returns a clear NEGATIVE, never a fake success.

## What this service is

`socseal.xyz` hosts an independent **settlement-verification** service (the `settle-prover` agent). It answers the question "did this agent-payment actually settle?" with a signed, on-chain-anchored artifact.

- **Proof-of-mined:** verifies the transaction hit a **mined block** (SEPTA rule: `block_height>0` AND `in_mempool=false`). "Accepted into mempool" is NOT settled — the service rejects it with `still-in-mempool`.
- **Post-quantum-signable:** evidence is sealed with a verifier fingerprint; the verification artifact is signed / structure-anchored and re-checkable offline (ML-DSA / NIST FIPS-204 lineage, quantum-forward-safe).
- **Fail-closed:** unknown txid → clean negative. It never fakes success.
- **Pay-per-verification, escrow-paid:** you fund via an on-chain escrow (WorkPayOpen). The service grants the verified artifact only after the payment OPEN is block-confirmed. No credit card, no KYC, no account.

## Endpoints (public, HTTPS)

Base: `https://socseal.xyz`

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/health` | liveness + armed status (`{"ok":true,"armed":true}`) |
| GET | `/.well-known/agent.json` | agent discovery card (A2A-style) |
| POST | `/invoice` | create a settlement-verification invoice (0.9 SOC / verification base) |
| POST | `/payment/submit_open` | submit a signed WorkPayOpen escrow tx (SEPTA release gate) |
| POST | `/verify/settlement` | submit `{"txid": "<64-hex>"}` → signed proof-of-mined verification artifact |
| GET | `/payment/{id}` | invoice/escrow status: `awaiting_open` → `submitted` → `MINED@<block>` → `granted` |

## How to verify a settlement (the agent flow)

1. **Health check:** `GET https://socseal.xyz/health` — confirm `"ok":true, "armed":true`.
2. **Get an invoice:** `POST https://socseal.xyz/invoice` → returns an invoice id (0.9 SOC each, volume discounts stepping down at 9 and 81 verifications).
3. **Pay via on-chain escrow:** construct a `WorkPayOpen` that pays the invoice's worker address exactly `service_atoms`; submit it as `POST /payment/submit_open`. 
4. **Confirm the OPEN is block-mined** (SEPTA): poll `GET /payment/{id}` until status shows `MINED@<block>` — never treat mempool-accepted as settled.
5. **Get the verification:** `POST /verify/settlement {"txid":"<64-hex>"}` → a signed artifact `{service, txid, mined:true, block_height, checked_at, verifier}` plus a proof-hash. The `verifier` fingerprint lets any third party re-verify offline.
6. **Keep the artifact** — it is your non-repudiable record that the payout truly landed in a confirmed block.

## Honest constraints

- The service verifies on the chain its keeper is authoritative for; an unknown txid on a different chain/network returns a clean negative. Do not present a negative as proof of non-settlement elsewhere.
- "Mined" is checked against the live node's confirmed chain, never against a client-supplied claim.
- The base tier is metered and escrow-funded; there is no free unlimited tier. Abusing rate limits may raise billing friction.
- No KYC, no account, no platform custody: your keys, your wallet, block-confirmed settlement end to end.

## References

- `references/endpoints.md` — full request/response examples (curl + JSON)
- `references/securing-a-verification.md` — step-by-step paid flow with status transitions
- `references/interpret-results.md` — how to read mined / not-mined / pending and build on them