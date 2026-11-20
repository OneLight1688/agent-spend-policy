# Agent Spend Policy — ASPL v0.1 (draft)

Status: **draft**, implemented by the Gate reference engine, exercised in
production dogfood since 2026-11. Breaking changes are versioned, never silent.

## 1. Design principles

1. **Never touches funds.** ASPL evaluates intent; settlement stays on rails the
   enterprise already controls. Keys belong to the enterprise.
2. **Agent identity is the primary key.** Every policy binds to a registered agent
   identity (signing pubkey), not to a human username.
3. **Verification is per transaction.** The gateway is the checkout counter: whatever
   the agent claims, only transactions that pass evaluation get co-signed.
4. **Rail-agnostic.** Budgets, allowlists and approvals work on any rail; only the
   payment adapter knows protocol details.
5. **Honest risk data.** Merchant risk fields are tiered and labelled by data source —
   never presented as uniform coverage.

## 2. Policy file

A policy is a single YAML document, versioned and signed. The reference engine uses
an equivalent JSON encoding. Evaluation order: identity → budget → allowlist →
threshold → approval.

```yaml
apiVersion: aspl/v0.1
kind: SpendPolicy
metadata:
  name: research-team-q1
  org: acme-labs
  validFrom: "2026-12-01T00:00:00Z"
  validUntil: "2027-03-01T00:00:00Z"

subjects:                      # who is governed — registered agent identities
  - agent: agent:acme:scraper-01
    pubkey: ed25519:9F86D081...
    owner: team/research
  - agent: agent:acme:browse-02
    pubkey: ed25519:2C26B36B...
    owner: team/research

budgets:
  - scope: team/research       # per-team monthly budget
    currency: USD
    monthly: 250.00
    alertAt: 0.8               # notify finance at 80% burn
  - scope: agent:acme:scraper-01
    currency: USD
    monthly: 80.00

allowlists:
  - scope: agent:acme:scraper-01
    merchants:
      trustRank: {min: 0.6}    # import merchants above Trust Rank tier
      pinned:
        - domain: onesource.io
        - resource: https://api.example.com/x402/weather
    rails: [x402, mcp]         # which rails this agent may use

thresholds:
  autoApproveBelow: 0.50       # USD per transaction
  requireApprovalAbove: 2.00
  approvers:
    - role: team-lead
    - role: finance-oncall

keys:                          # co-signing key sources — BYO, never ours
  policyChange: {source: webauthn, requireStepUp: true}
  paymentSign:   {source: kms, keyRef: aws:kms:alias/acme-agent-gate}
  # alternatives: hsm (PKCS#11), raw-key (dev only)

audit:
  chain: hashlinked            # each event links the previous hash
  export:
    - format: jsonl
      destination: s3://acme-audit/agent-spend/
  retention: 7y
```

## 3. Governance tiers

| Tier | Meaning | Verification |
|---|---|---|
| NATIVE | Speaks ASPL natively (SDK / spec) | Identity registered; full policy surface |
| CONFORMANT | Speaks open protocols: AP2 mandates, MCP, x402 | Gateway reads standard messages; per-transaction evaluation. No vendor certification. |
| UNGOVERNABLE | Closed agents outside enterprise controls | Not interceptable. Reconciliation reports surface unmapped spend as a gap. |

Vendors may self-declare conformance via the checklist in §6. Implementations of this
spec do not certify, rank or resell third-party agents.

## 4. Merchant risk tiers

Risk fields carry an explicit data-source label. Engines must not present heuristic
scores as settlement-derived.

```yaml
risk:
  - source: settlement-behaviour   # real volume, buyer diversity, ticket
    rails: [x402, tempo, solana]   # sanity, identity (e.g. Trust Rank, live)
  - source: static-heuristics      # domain age, TLS age, public pricing page
    rails: [card, bank]            # computed at evaluation time
  - source: customer-history       # your own disputes and returns, in your
    rails: [all]                   # gateway instance — never shared
```

When a `trustRank` floor is configured and no settlement data exists for a merchant,
the evaluation **must reject** rather than guess. Silence is not consent.

## 5. Audit event schema

Every evaluated payment emits one event. Events are hash-linked; a broken chain is
itself an alarm.

```json
{"v":"aspl/v0.1","type":"payment.evaluated","ts":"2026-11-20T04:24:54Z",
 "agent":"agent:acme:scraper-01","intent":{"resource":"https://api.../weather",
 "amount":"1000","currency":"USDC","rail":"x402"},
 "decision":{"allow":true,"status":"auto-approved",
 "reason":"allowlist:trust-rank>=0.6; budget ok",
 "coSignature":{"source":"kms","keyRef":"aws:kms:alias/acme-agent-gate"}},
 "prevHash":"0x9af3...","hash":"0x1c8e..."}
```

Event types (v0.1): `policy.loaded` · `payment.evaluated` · `payment.approved` ·
`payment.rejected` · `payment.settled` · `budget.threshold` · `chain.verified`

Decision statuses: `auto-approved` · `approved` · `pending` · `rejected`

## 6. Governance-readiness checklist

Third-party agent vendors may self-declare conformance. Implementations of this spec
neither certify nor rate agents — enterprises verify claims themselves.

- [ ] Registers a signing identity for each agent instance (policies can bind)
- [ ] Emits a payment intent **before** settlement, on a standard protocol
      (ASPL hook / AP2 mandate / MCP)
- [ ] Honours policy denials — no charge without a co-signature
- [ ] Retries and failures are reported, not hidden

## 7. Key sources

Policy changes and payment co-signing reference keys the enterprise controls:

| source | meaning |
|---|---|
| `webauthn` | platform authenticator / FIDO2 (passkeys, security keys) |
| `kms` | cloud KMS key reference (AWS KMS, GCP KMS, …) |
| `hsm` | PKCS#11 HSM slot |
| `raw-key` | local key file — development only |

No hardware is sold or required; bring the keys you already have.

## 8. Multi-user & roles (planned v0.2)

An enterprise gateway is operated by several people with different duties. Implementations
SHOULD support the following minimal role model:

| role | permissions |
|---|---|
| `admin` | write policies, register agent identities, manage users |
| `approver` | approve / reject pending intents above the auto-approve threshold |
| `operator` | register agents, view dashboards and reports |
| `auditor` | read-only: audit chain, reports, policy history |

Requirements:

1. **Approvals are authenticated.** Every approval or rejection records an authenticated
   principal; anonymous approvals are invalid and MUST fail chain verification semantics.
2. **Policy changes are step-up protected.** Writing a policy requires `admin` plus the
   configured key source (§7: webauthn step-up, KMS signature, …).
3. **Two identity namespaces.** Human users (approvers, admins) and agent identities
   (policy subjects) are separate. A policy binds agent pubkeys; an approval binds an
   authenticated user principal. The audit chain records both.
4. **Do not invent a user system.** Self-hosted deployments map roles onto the
   enterprise's existing IdP (OIDC / SAML). When an IdP is configured, the gateway
   MUST NOT store passwords. Local users are acceptable only for single-operator
   deployments.

## 9. Feedback

Open an issue or propose fields via PR. The reference engine
(`engine/gate_policy.py`) is the executable form of this document.
