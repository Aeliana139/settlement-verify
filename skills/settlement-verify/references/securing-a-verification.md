# securing-a-verification.md — step-by-step paid flow (live rail, USDC x402)

The full agent flow from "I need to prove this settlement mined" to holding a
granted verification artifact. All steps are HTTP against `https://socseal.xyz`.

## 1. Confirm the verifier is live

```
GET /health  ->  {"ok":true,"service":"settle-prover"}
```

## 2. Request a verification invoice

```
POST /verify {"txid":"<64-hex>"}  ->  {invoice_id:"sv_...",
   pay:{currency:"USDC", network:"Polygon", pay_to:"0xBA3f...8819",
        amount_atoms:250000, price_usdc:0.25}}
```
Keep `invoice_id` and `pay.pay_to` / `pay.amount_atoms` — you need all three for payment.
Note: txid is bare 64-hex (no `0x` prefix).

## 3. Pay in USDC on Polygon

Send `amount_atoms` (250000 = $0.25) of **native USDC** (`0x3c499c...`) on
Polygon (chain 137) to the returned `pay_to` (`0xBA3f...8819`) — the only receive
address. Record the payment's `0x` Polygon txid.

## 4. Confirm payment — wait for BLOCK confirmation (SEPTA — the whole point)

```
POST /confirm_payment {"invoice_id":"sv_...","payment_txid":"<0x Polygon USDC txid>","payer":"<your id>"}
```
The service block-confirms the USDC transfer on-chain (>= amount_atoms to pay_to).
Accept ONLY a `granted` / `paid` response. A payment that is broadcast but not yet
mined returns `awaiting_payment` — keep waiting, never treat mempool as settled.

## 5. Receive the signed artifact

Once block-confirmed, the response carries the **ML-DSA-87-signed** artifact:
```
{invoice_id, status:"granted", mode:"paid",
 deliverable:{service, txid, mined:true, block_height, checked_at,
              verifier, signature, proof_hash}}
```

## 6. Keep the artifact — and re-verify offline with no trust in us

- `mined:true` + `block_height` prove it landed in a confirmed block.
- `verifier` + `signature` let any third party re-verify the artifact **offline**
  against the public key from `GET /pubkey` (ML-DSA-87, FIPS-204).
- `proof_hash` binds the evidence canonically.

## Free trial (honest)

A capped free trial (2 per address) returns an **UNSIGNED** verdict + `proof_hash`
so you can prove the rail works before paying. Signed artifacts require the paid
USDC path above.

## Billing note (honest)

Base is **$0.25 USDC per verification** (Polygon, block-confirmed). No KYC, no
account, no credit card. There is no unlimited free tier — the USDC payment IS
the rail. SOC is the underlying sovereign store being proven; we charge USDC as
the liquid operating currency and never sell SOC to raise operating cash.