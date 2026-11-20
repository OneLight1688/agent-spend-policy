# Agent Spend Policy (ASPL) — open spec + reference engine

**ASPL is an open format describing what an AI agent is allowed to pay for**: budgets,
merchant allowlists, approval thresholds, key sources and audit events. One policy
file governs any agent on any rail.

This repo contains the **spec** and a **production-tested reference engine** — the same
evaluator that governs [PayCensus's own purchasing agent](https://gate.paycensus.com/dashboard.html)
in public dogfood (every decision, including rejections, is published).

> **Gate never touches funds.** Keys belong to the enterprise; the gateway only evaluates
> policy and co-signs with keys you control. No custodial risk, no money transmission.

## Why open

Policy engines are a commodity — the value is in **risk data** (who actually receives
agent payments, and how reliably) and in a **standard that everyone can adopt for free**.
So: the format and the engine are MIT. Commercial layers (managed cloud, merchant risk
API, SSO/SIEM) live separately at [gate.paycensus.com](https://gate.paycensus.com/).

## Status

- **v0.1 draft** — implemented, exercised in production dogfood since 2026-11
- Breaking changes are versioned, never silent

## Repo layout

```
spec/ASPL-v0.1.md      the specification
engine/gate_policy.py  reference evaluator (hash-linked audit, CLI)
examples/              a real policy file (our dogfood agent)
```

## Quickstart

```bash
python3 engine/gate_policy.py eval agent:acme:scraper-01 \
    https://api.example.com/x402/weather 0.001 x402
# -> {"allow": true, "status": "auto-approved", "reason": "...", "eventHash": "..."}

python3 engine/gate_policy.py verify   # verify the hash chain
```

Point it at your policy and audit files with env vars:

```bash
GATE_POLICY=my-policy.json GATE_AUDIT=audit.jsonl python3 engine/gate_policy.py eval ...
```

Wire it before any money moves:

```python
import gate_policy
gate = gate_policy.evaluate("agent:acme:scraper-01", resource_url,
                            amount_usd, rail="x402", pay_to=merchant_address)
if not gate["allow"]:
    raise PermissionError(gate["reason"])
# ... proceed to sign & pay with YOUR key
```

## Governance tiers

| Tier | Meaning |
|---|---|
| NATIVE | Speaks ASPL natively — full policy surface |
| CONFORMANT | Speaks open protocols (AP2 mandates, MCP, x402) — passes the gateway, verified per transaction |
| UNGOVERNABLE | Closed agents outside enterprise controls — surfaced as reconciliation gaps, not faked |

Vendors may self-declare conformance (checklist in spec §6). This project does not
certify, rate or resell agents.

## Honest risk data

Merchant risk fields carry an explicit source label:

- `settlement-behaviour` — real volume / buyer diversity / ticket sanity / identity
  on stablecoin rails (this is what [Trust Rank](https://agent.paycensus.com/) provides)
- `static-heuristics` — domain age, TLS age, public pricing pages (computed locally)
- `customer-history` — your own disputes, in your gateway instance, never shared

No engine may present heuristic scores as settlement-derived.

## License

MIT — see [LICENSE](LICENSE).

## Links

- Spec site: https://gate.paycensus.com/spec/
- Live audit trail: https://gate.paycensus.com/dashboard.html
- Trust Rank (merchant risk data): https://agent.paycensus.com/
- PayCensus census: https://paycensus.com/
