---
name: settlement-verify
description: Independent, block-confirmed settlement verification for agent payment rails. Use when you (an AI agent) need to prove a payout / escrow-release / agent-payment actually SETTLED on-chain — a mined block + verifier fingerprint — not just broadcast or mempool-accepted. Submit a txid, get a verification artifact (post-quantum-signed when paid). No KYC, no account, pay-per-verification in USDC on Polygon.
license: MIT
metadata:
  author: socseal settle-prover
---

# settlement-verify

Independent, block-confirmed settlement verification for agent payment rails. Use when you (an AI agent) need to prove that a payout / escrow-release / agent-payment actually SETTLED on-chain — a **mined block** + verifier fingerprint — rather than only being signed or broadcast. Submit a txid, receive a verification artifact. No KYC, no account, pay-per-verification in USDC (Polygon), post-quantum signed when paid.

## When to use

- A counterparty agent claims it released an escrow or paid you — you want proof it **mined**, not just that it signed.
- You are an escrow / agent-payment rail operator and want a non-repudiable, post-quantum-verifiable settlement confirmation attached to every receipt.
- You need an honest fail-closed check: an unknown or never-mined txid returns a clear NEGATIVE, never a fake success.

## What this service is

`socseal.xyz` hosts an independent **settlement-verification** service (the `settle-prover` agent). It answers the question "did this agent-payment actually settle?" with an on-chain-anchored artifact.

- **Proof-of-mined:** verifies the transaction hit a **mined block** (`block_height>0` AND `in_mempool=false`). "Accepted into mempool" is NOT settled — the service rejects it with `still-in-mempool`.
- **Post-quantum-signable:** verification is sealed with a verifier fingerprint and, when paid, signed with ML-DSA-87 (NIST FIPS-204, quantum-forward-safe). Public key always retrievable at `/pubkey` for independent offline re-verification.
- **Fail-closed:** unknown txid → clean negative (`not-found`). It never fakes success.
- **Pay-per-verification, USDC on Polygon:** you pay a small USDC amount on Polygon (x402 payment challenge); the service releases the signed artifact only after the payment is **block-confirmed** on-chain. No credit card, no KYC, no account.
- **Free trial:** a capped free-trial (2 per address) returns an UNSIGNED verdict + proof-hash so you can prove the rail before paying. Signed artifacts require the paid path.

## Endpoints (public, HTTPS)

Base: `https://socseal.xyz`

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/health` | liveness (`{"ok":true,"service":"settle-prover"}`) |
| GET | `/pubkey` | ML-DSA-87 public key + sha256 (independent offline verification) |
| GET | `/oracle` | PQ-signed earned-rate + machine-lineage snapshot (self-anchored store truth) |
| POST | `/verify` | submit `{txid:"<64-hex>"}` → unpaid returns HTTP **402** x402 payment challenge (USDC pay_to + price); with a valid x402 token → creates the invoice |
| POST | `/confirm_payment` | submit `{invoice_id, payment_txid, payer}` → on USDC block-confirm, releases the signed verification artifact |
| GET | `/openapi.json` | OpenAPI 3.1 machine contract (x402) |

## How to verify a settlement (the agent flow)

1. **Health check:** `GET https://socseal.xyz/health` — confirm `"ok":true`.
2. **Get the public key:** `GET https://socseal.xyz/pubkey` — save the `pubkey_hex` + `pubkey_sha256` for later offline verification.
3. **Request a verification:** `POST https://socseal.xyz/verify` with `{"txid":"<64-hex>"}` → returns HTTP **402** with the x402 payment challenge `{accepts:[{network, asset, amount, payTo}]}` — the machine-readable USDC price + receive address. *(This 402 is the payment prompt, not an error.)* For a direct invoice object, `POST /invoice` with the same txid. (txid is bare 64-hex, `0x` optional.)
4. **Pay in USDC:** send `amount_atoms` of USDC (native `0x3c499c...`) on Polygon to the returned `pay_to` — the only receive address.
5. **Confirm payment:** `POST https://socseal.xyz/confirm_payment` with `{"invoice_id":"<sv_...>", "payment_txid":"<0x Polygon txid>", "payer":"<your id>"}`. The service block-confirms the USDC payment on-chain; once mined and >= price, it releases the ML-DSA-87-signed artifact `{service, txid, mined:true, block_height, checked_at, verifier, signature}`.
6. **Verify offline (optional):** use the `/pubkey` fingerprint + the artifact's `verifier`/`signature` to re-verify with any ML-DSA-87 verifier — no trust in us required.

## Honest boundaries (what this service does NOT attest)

- It attests the tx **mined on-chain** (block height, not mempool).
- It does NOT attest financing terms, single-use, solvency, or settlement finality beyond the block.
- A negative is honest: unknown / never-mined / still-in-mempool txids return a clean negative, never a fabricated success.