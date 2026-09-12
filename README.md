# settlement-verify

An **agent skill** for independent, block-confirmed settlement verification on
agent payment rails. Install it in any agent runtime that reads the `SKILL.md`
format (Claude Code, Cursor, Cline, GitHub Copilot, Windsurf, Gemini CLI, and
other surfaces via `skills.sh`).

When an agent needs to prove that a payout / escrow-release / agent-payment
actually **mined** — a confirmed block + verifier fingerprint — rather than only
being signed or broadcast, this skill points at a live, public, escrow-paid
verification service.

- **Proof-of-mined (SEPTA):** `block_height>0` AND `in_mempool=false`. Mempool-only is NOT settled.
- **Post-quantum-forward:** the verification artifact is sealed with a verifier fingerprint (ML-DSA / FIPS-204 lineage), re-checkable offline.
- **Honest fail-closed:** unknown / never-mined txid returns a clean negative — never a fake success.
- **No KYC, no account, escrow-paid:** fund via on-chain escrow; the artifact is granted only after the payment OPEN is block-confirmed.

## Service

Live, armed, and publicly reachable at **https://socseal.xyz**
(`GET /health` → `{"ok":true,"armed":true}`; discovery card at `/.well-known/agent.json`).

## Install

```bash
# any agent runtime that reads SKILL.md
npx skills add <owner>/settlement-verify
```

## Use (short version)

1. `GET https://socseal.xyz/health` — confirm armed.
2. `POST /invoice` → 0.9 SOC per verification.
3. Fund the escrow OPEN, wait for `MINED@<block>` (never mempool).
4. `POST /verify/settlement {"txid":"<64-hex>"}` → signed proof-of-mined artifact.

Full flow: see `SKILL.md` and `references/`.

## Contents

- `SKILL.md` — when to use + the agent flow
- `references/endpoints.md` — exact request/response contracts
- `references/securing-a-verification.md` — step-by-step paid flow
- `references/interpret-results.md` — mine / not-mined / pending semantics

## License

MIT.