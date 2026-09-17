# endpoints.md — full request/response contracts (live rail, USDC x402)

Base URL: `https://socseal.xyz` (public HTTPS, agent-discovery via `/.well-known/agent.json`).
All responses JSON. No API key, no account, no KYC. **Payment is USDC on Polygon** (x402).

## GET /health

```
curl https://socseal.xyz/health
```
```json
{"ok": true, "service": "settle-prover"}
```
Liveness + "is the verifier accepting work" check.

## GET /.well-known/agent.json

```
curl https://socseal.xyz/.well-known/agent.json
```
Agent discovery card. `id: settle-prover`; an `x402` extension advertises the
verification as pay-per-use in USDC on Polygon (`eip155:137`, native USDC
`0x3c499c...`); an `oracle` extension advertises the PQ-signed earned-rate snapshot.

## GET /pubkey

```
curl https://socseal.xyz/pubkey
```
```json
{"alg": "ML-DSA-87", "pubkey_hex": "d63db7e5...", "pubkey_sha256": "..."}
```
Retrieve the public key to independently re-verify any paid artifact offline
(no trust in us required).

## GET /oracle

```
curl https://socseal.xyz/oracle
```
PQ-signed (ML-DSA-87) snapshot of the self-anchored earned rate + machine
lineage: `{earned_usdc_per_soc, step, adopted_count, signature}`. SOC's value is
self-anchored (effort/scarcity/sovereignty), never indexed to BTC/USDC price.

## POST /verify

Submit a txid to get a payment invoice (USDC on Polygon).

```
curl -X POST https://socseal.xyz/verify \
  -H "Content-Type: application/json" \
  -d '{"txid":"<64-hex>"}'
```
NOTE: txid is **bare 64-hex, no `0x` prefix** (SOC/Vanity chain txids). A bodiless
request returns HTTP 402 (x402 payment challenge) — that is the discovery probe
response, not an error.

**Invoice created (HTTP 200):**
```json
{
  "status": "invoice_created",
  "invoice_id": "sv_xxxxxxxxxx",
  "pay": {
    "currency": "USDC",
    "network": "Polygon",
    "pay_to": "0xBA3f8D621d226dC795BCE02BF7C106f1bA278819",
    "amount_atoms": 250000,
    "price_usdc": 0.25
  },
  "next": "POST /confirm_payment with {invoice_id, payment_txid}"
}
```

**Not mined / unknown (honest negative — fail-closed, HTTP 400):**
```json
{"error": "not-mined: not-found", "verified": false}
```
`still-in-mempool` = broadcast but not in a confirmed block — NOT settled.
`node-unreachable` = verification node could not be reached — retry later; never
treat as confirmed.

## POST /confirm_payment

Submit the block-confirmed USDC payment txid to release the signed artifact.

```
curl -X POST https://socseal.xyz/confirm_payment \
  -H "Content-Type: application/json" \
  -d '{"invoice_id":"sv_xxxxxxxxxx","payment_txid":"<0x Polygon USDC txid>","payer":"<your id>"}'
```
- `payment_txid` must be a `0x` Polygon USDC txid that our server block-confirms
  on-chain (>= price_atoms to `pay_to`).
- Within the free cap (2/address) it returns an **UNSIGNED** free-trial verdict +
  `proof_hash` (prove the rail, no cash).
- When paid and block-confirmed: returns the **ML-DSA-87-signed** artifact
  `{mode:"paid", deliverable:{service, txid, mined:true, block_height,
  checked_at, verifier, signature, ...}}`.

## GET /openapi.json

```
curl https://socseal.xyz/openapi.json
```
OpenAPI 3.1 machine contract (title `socseal settle-prover`, x402 payment info,
input/output schemas) — the canonical discovery contract for x402scan/Circle.