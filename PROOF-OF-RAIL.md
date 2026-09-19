# PROOF-OF-RAIL — socseal settle-prover (public, verifiable)

> **Claim:** the ML-DSA-87 oracle signature is authentic, AND the public
> `/verify` → `/confirm_payment` rail runs end-to-end over the public HTTPS
> endpoint `https://socseal.xyz`.
> **Honest boundary:** this PROVES the machinery works publicly. It does **NOT**
> prove any external human paid — **R=1 (a non-owned counterparty paying real
> value) is NOT claimed here.** The trial below is the documented free-trial path
> (2/address), which legitimately returns an UNSIGNED verdict to prove the rail
> before a stranger risks money.

Generated: 2026-09-19. All raw HTTP below is against the live public endpoint.
No local service was used for the rail exercise. All artifacts byte-verifiable.

---

## 1. Oracle signature authenticity — FIPS-204 (ML-DSA-87) VERIFIED

Fetched independently over public HTTPS:
- `GET https://socseal.xyz/oracle`
- `GET https://socseal.xyz/pubkey`

**Public key (ML-DSA-87, 2592 bytes):** `GET /pubkey` →
`{"alg":"ML-DSA-87","pubkey_hex":"d63db7e53bee...7263dd48"}` (full 5184 hex).
Writer-verified pubkey byte length = 2592 bytes (correct ML-DSA-87 size).

**Canonical message signed:** `{"adopted_count":1,"earned_usdc_per_soc":10.0,"ok":true,"service":"socseal-oracle","step":8}`

**Signature (value hex):**
`85df743211a201b7...dc5a175` (from `/oracle` → `signature.value`).

**Verify result (real FIPS-204 run):**
```
$ /home/sophia-ai/pqsign/target/release/pqsign verify pub.hex <msg_hex> sig.hex
VALID   (rc=0)
```
The local `pqsign` is the exact wrapped ML-DSA/FIPS-204 binary. Signature
ALGORITHMICALLY verifies against the published public key.

**Stranger repeat steps** (no trust in us required — FIPS-204 from NIST cert):
1. `curl https://socseal.xyz/pubkey -o pubkey.json`
2. `curl https://socseal.xyz/oracle -o oracle.json`
3. Extract `pubkey_hex` and `signature.value`; take `signature.canonical` as the
   message. `msg_hex=$(python3 -c "print('$CANONICAL'.encode().hex())")`.
4. Save pubkey → `pub.hex`, signature.value → `sig.hex`.
5. `pqsign verify pub.hex $msg_hex sig.hex`  → expect `VALID`.
(Any ML-DSA-87 / FIPS-204 tool — BoringSSL, liboqs, `pqsign` — yields Identical:
valid signature over a claimed publish-only public key.)

---

## 2. Free-trial verdict over the PUBLIC endpoint — full rail exercised

Used a **real, mined** SOC-chain txid accepted by the live SOC node
(`e321c202370aa432a429e9f75c617347b30dde14a161b4900e68af6b5f807b90`,
mined @ block 412305 on SOC mainnet), plus a real mined Polygon tx
(`0x6bfa30c425232abc7cf5920f49cd90060d0f787d0cf53faefcb4961290de36e4`).

**POST /verify** → HTTP 200
```json
{"status":"invoice_created","invoice_id":"sv_adeed12da46d",
 "pay":{"currency":"USDC","network":"Polygon",
        "pay_to":"0xBA3f8D621d226dC795BCE02BF7C106f1bA278819",
        "amount_atoms":250000,"price_usdc":0.25},
 "next":"POST /confirm_payment with {invoice_id, payment_txid}"}
```

**POST /confirm_payment** → HTTP 200, **status granted, mode free-trial,
UNSIGNED verdict + proof_hash:**
```json
{
  "invoice_id": "sv_adeed12da46d",
  "status": "granted",
  "mode": "free-trial",
  "deliverable": {
    "mode": "free-trial",
    "free_used": 1,
    "free_left": 1,
    "proof_hash": "757e1e92180a9c1eadabad122a23499d8aca927ef00fbaf887d4cc18b92c6c06",
    "evidence": {
      "service": "settle-prover",
      "txid": "e321c202370aa432a429e9f75c617347b30dde14a161b4900e68af6b5f807b90",
      "mined": true,
      "block_height": "412305",
      "verifier_pubkey": "6b12c77117a6d59d",
      "schema": "socseal-settle-proof/1"
    },
    "checked_at": "2026-09-19T08:31:32.572527+00:00",
    "signature": null,
    "signed": false,
    "attests": ["mined-on-chain","block_height"],
    "does_not_attest": ["financing-terms","single-use","not-financed-elsewhere","settlement-finality"]
  }
}
```

**proof_hash deterministic — independently recomputed and MATCHED:**
```
canonical = {"block_height":"412305","mined":true,"schema":"socseal-settle-proof/1","service":"settle-prover","txid":"e321c202370aa432a429e9f75c617347b30dde14a161b4900e68af6b5f807b90","verifier_pubkey":"6b12c77117a6d59d"}
sha256(canonical) = 757e1e92180a9c1eadabad122a23499d8aca927ef00fbaf887d4cc18b92c6c06  ✓ (matches returned)
```

---

## 3. HONEST — what this proves vs what it does NOT

**Proves (machinery works publicly):**
- The public HTTPS endpoint is live and serves an authenticated ML-DSA-87 /
  FIPS-204 oracle signature that verifies.
- The `/verify` → `/confirm_payment` rail executes end-to-end over the public
  site: a real mined txid returns a mined-on-chain verdict with a deterministic,
  independently-recomputable `proof_hash`.
- Fail-closed honesty: an unknown/unmined txid returns `not-mined: not-found`
  (verified live); free-trial correctly returns `signature:null / signed:false`
  (no forged signed artifact for free).

**Does NOT prove (explicitly not claimed):**
- **R=1 is NOT claimed.** No non-owned counterparty paid real value here; the
  free trial (2/address) proves the rail, not that a stranger paid.
- No signed (ML-DSA-87) paid artifact was collected — signature:null by design
  for the free trial.
- No "market price" of SOC, no financing/single-use attestation, no settlement
  finality. The free-trial verdict itself declares `does_not_attest: [...]`.

This document is an honest proof-of-rail: third parties can independently
re-verify every claim above with the documented FIPS-204 steps and the public
HTTP contract at `https://socseal.xyz/openapi.json`.