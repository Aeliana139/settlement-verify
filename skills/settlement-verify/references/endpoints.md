# endpoints.md — full request/response contracts

Base URL: `https://socseal.xyz` (public HTTPS, agent-discovery via `/.well-known/agent.json`).
All responses JSON. No API key, no account, no KYC.

## GET /health

```
curl https://socseal.xyz/health
```
```json
{"ok": true, "armed": true}
```
Use as a liveness + "is the verifier live and accepting work" check.

## GET /.well-known/agent.json

```
curl https://socseal.xyz/.well-known/agent.json
```
Agent discovery card (A2A-style). `id: settle-prover`, `supportedInterfaces` bind
`jsonrpc/http` to `https://socseal.xyz`; an `x402` extension advertises the
verification as pay-per-use.

## POST /invoice

Create a settlement-verification invoice. No body required.

```
curl -X POST https://socseal.xyz/invoice -H "Content-Type: application/json" -d '{}'
```
```json
{
  "invoice_id": "sv_xxxxxxxxxx",
  "service":    "soc_verify",
  "soc":        0.9,
  "soc_atoms":  900000000,
  "memo":       "soc_verify:sv_xxxxxxxxxx",
  "worker":     "soc113404bebe...",          // fundee address for the escrow OPEN
  "status":     "awaiting_open",
  "open_txid":  null,
  "deliverable": null,
  "created":    "2026-09-08T..."
}
```
Volume discounts: 0.9 SOC each base; price multiplier drops at 9 and 81 verifications.

## POST /payment/submit_open

Submit a signed `WorkPayOpen` escrow transaction that pays the invoice's `worker`
address exactly `soc_atoms`. This is the SEPTA release gate — the deliverable is
granted only after the OPEN is **block-mined** (never mempool-accepted).

```
curl -X POST https://socseal.xyz/payment/submit_open \
  -H "Content-Type: application/json" \
  -d '{"invoice_id":"sv_xxxxxxxxxx","transaction":"<hex raw WorkPayOpen>"}'
```

## POST /verify/settlement

Submit a txid to get a proof-of-mined verification artifact.

```
curl -X POST https://socseal.xyz/verify/settlement \
  -H "Content-Type: application/json" \
  -d '{"txid":"<64-hex>"}'
```

**Mined (positive):**
```json
{
  "verified": true,
  "evidence": {
    "service": "soc_verify",
    "txid": "<64-hex>",
    "mined": true,
    "block_height": "<block>",
    "checked_at": "<iso>",
    "verifier": "<hex-sealed worker fingerprint>"
  },
  "proof_hash": "<sha256 of canonical evidence>"
}
```

**Not mined / unknown (honest negative — fail-closed):**
```json
{"verified": false, "error": "not-mined: <reason>"}
```
`still-in-mempool` = the tx is broadcast but not yet in a confirmed block — NOT settled.
`node:HTTP 404` = the node does not know this txid on its confirmed chain — treat as
not proven, never as a confirmed settlement.

## GET /payment/{invoice_id}

Invoice / escrow lifecycle:
```
GET /payment/sv_xxxxxxxxxx
```
`status` transitions: `awaiting_open` → `submitted` → `MINED@<block>` → `granted`.
`granted` means the verification deliverable was released (payment OPEN block-confirmed).