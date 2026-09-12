# securing-a-verification.md — step-by-step paid flow

The full agent flow from "I need to prove this settlement mined" to holding a
granted verification artifact. All steps are HTTP against `https://socseal.xyz`.

## 1. Confirm the verifier is armed

```
GET /health  ->  {"ok":true,"armed":true}
```
If not armed, hold off — an unarmed verifier cannot release a granted artifact.

## 2. Create an invoice

```
POST /invoice  ->  {invoice_id:"sv_...", soc:0.9, soc_atoms:900000000,
                    worker:"soc1134...", status:"awaiting_open"}
```
Keep `invoice_id`, `worker`, and `soc_atoms` — you need all three for the payment.

## 3. Construct and submit the escrow OPEN

Build a `WorkPayOpen` transaction from your wallet that pays the invoice `worker`
address exactly `soc_atoms` (0.9 SOC at launch). Submit it:

```
POST /payment/submit_open {"invoice_id":"sv_...","transaction":"<raw hex>"}
```

The service will not accept a fake or under-funded OPEN. This is the escrow: your
funds are committed on-chain and held until the verification is granted.

## 4. Wait for BLOCK confirmation (SEPTA — the whole point)

Poll `GET /payment/sv_...`. Accept ONLY:
- `MINED@<block>` — your OPEN landed in a confirmed block.

Reject / keep waiting on:
- `submitted` — broadcast, not confirmed.
- `still-in-mempool` — in the mempool, NOT settled.

Why this matters: an escrow that is mempool-accepted but block-rejected never
settled. The service will not grant a deliverable against a mempool-only OPEN.

## 5. Request the verification

```
POST /verify/settlement {"txid":"<the-settlement-txid-you-care-about>"}
```

If your payment reached `granted`, verification works against the confirmed chain.
A mined txid returns `{verified:true, evidence:{...mined, block_height, verifier}}`.

## 6. Keep the artifact

The `evidence` + `proof_hash` is your non-repudiable record:
- `mined:true` + `block_height` prove it landed in a confirmed block.
- `verifier` is a hex-sealed fingerprint — any third party can re-verify offline.
- `proof_hash` binds the evidence canonically.

## Billing note (honest)

Base is 0.9 SOC per verification, escrow-paid, block-confirmed. There is no free
unlimited tier — the escrow IS the payment rail. Volume discounts kick in at 9 and
81 verifications for high-frequency settlement monitoring.