---
title: "Policy Pilot: Maker–Checker, Durable Why, and Graph Investigation for Cash SSI"
description: "An FDE-shaped reference pilot for Standard Settlement Instructions — production-grade OPA controls, OIDC identity, versioned single-store writes, security events that preserve why, Neo4j investigation, fail-closed mutation skills, and a DLQ-backed index path."
date: 2026-07-17
type: posts
tags: ["OPA", "Neo4j", "Architecture", "Financial Services", "OIDC", "Event-Driven Architecture", "RAG", "Observability"]
draft: false
---

*The decisive part of an enterprise AI pilot is not the chat interface. It is whether the system encodes real controls — maker–checker, segregation of duties, durable authorization rationale, and a knowledge graph that can answer structural compliance questions — before the language model ever responds.*

Most enterprise AI demonstrations stop at retrieval. [**Policy Pilot**](https://github.com/sanjuthomas/policy-pilot) takes the opposite approach: a **customer-pilot / reference deployment** for the cash leg of Standard Settlement Instructions (SSI). It is AI-native where that helps (intent routing, explanation), and enterprise-shaped where money and policy matter (identity, Rego, transactional aggregates, CDC, graph, operator skills).

I built it as an **FDE-style** showcase — the kind of end-to-end system you stand up with a customer team under real bank constraints, not a conversational layer over unstructured documents.

---

## The Credibility Layer: Production-Grade OPA

The strongest part of the platform is not the UI. It is the **OPA policy pack**.

Two decades in back and middle office leave durable lessons: mutual approvals that should not occur, desks approving outside their book, juniors approving a manager’s work, amount authority that appears sound until it is tested. Those patterns are encoded as Rego — segregation of duties, reporting-line inversion, desk LOB boundaries, amount clubs, lifecycle status, and four-eyes controls. The same engine guards mutations and answers *who can / who did / why*. See the [OPA controls](https://github.com/sanjuthomas/policy-pilot/blob/main/docs/opa-controls.md) documentation for the full control map.

That experience is not something a tutorial policy pack can substitute. It has to be earned in operations.

To be precise about scope: this is **production-grade control design** and battle-tested policy shape — not a claim that the demo stack is running in a live bank. Every Go / No Go confirmation and every investigation answer still rests on policy that mirrors real operating practice, rather than permissive allow-all Rego.

---

## Maker–Checker and Four Eyes on Instructions

The lead control narrative is **maker–checker** and **four eyes** on instructions, framed the way a supervision desk would recognize it (FINRA-aligned language, without stretching the model into an artificial “six eyes” story).

Instruction path:

1. Middle office **creates** and **submits** (same creator class for draft and submit).
2. A profit-center / second pair of eyes **approves**.

Payments support a richer lifecycle — draft → submit → approve → cancel — but the headline control remains four-eyes on the instruction. The payment path is where operator skills and funding desks operate; the instruction is where the control narrative begins. Domain shapes are documented in [domain models](https://github.com/sanjuthomas/policy-pilot/blob/main/docs/domain-models.md).

---

## Identity First: OIDC Subject → OPA

Policy is only as trustworthy as the subject it evaluates.

Policy Pilot uses **ZITADEL** as an OIDC/OAuth2 identity provider. Services validate JWT Bearer tokens via OIDC discovery. The subject carries the attributes OPA requires — roles, LOB, covering LOBs, amount clubs, supervisor — from **identity metadata**, not from a hard-coded application map. Architecture notes are in [architecture decisions](https://github.com/sanjuthomas/policy-pilot/blob/main/docs/architecture-decisions.md).

Runtime directory reads come from ZITADEL; `users.yaml` is seed-only. Chat and mutation skills act as the **logged-in user** (with service on-behalf-of where domain APIs require it). The chain is deliberate:

**OIDC subject → authorization-service → OPA → allow / deny + basis**

Today’s UIs still use a demo login facade on top of ZITADEL sessions. The policy model does not depend on that facade: attributes already live in the IdP, and authorization-code with PKCE is a natural hardening step later — not a rewrite of Rego.

---

## Security Events as System of Record for *Why*

Most banks can answer *who* and *when*. Few retain a durable *why*.

In Policy Pilot, **security events are first-class**. Every authorized and denied mutation persists an authorization block with the domain aggregate — `allow_basis`, violations, and summary — rather than a disposable log line. Domain services write the versioned instruction or payment **and** its security event in one MongoDB transaction. That trail is what chat later presents as Who / When / Why, and what the indexer projects into Neo4j for investigation. Details are in the [authorization audit trail](https://github.com/sanjuthomas/policy-pilot/blob/main/docs/authorization-audit-trail.md).

Authorization is not a side effect. It is part of the record.

---

## Keep Transaction Boundaries Simple (KISS)

Transactional services write to **one store only**: MongoDB. Documents are **versioned**. There is no MongoDB + Kafka distributed transaction and no two-phase commit spanning stores.

Domain services focus on persisting the aggregate correctly. Kafka Connect CDC fans out **downstream**. The indexer, graph, and vectors are eventually consistent projections — not participants in the write path. The [data flow](https://github.com/sanjuthomas/policy-pilot/blob/main/docs/data-flow.md) documentation walks that boundary end to end.

The design is intentionally simple. Correctness lives at the aggregate; integration stays asynchronous.

---

## Why the Knowledge Graph Matters

Vector search finds similar event text. It does **not** ground structural questions.

Without a **Neo4j knowledge graph**, you cannot reliably answer:

> Are there users approving each other’s instructions?

That question requires traversal across users, instructions, and approval edges — creator/approver chains, subordinate approvers, shared routes — not nearest-neighbor prose. Graph and vector are complementary: relationships versus semantic narrative. Compliance investigation needs both; relationship questions need the graph.

Latest instruction and payment versions are marked with a `CURRENT` relationship so inventory and show-by-id answers do not mix historical versions by accident.

---

## Index Path Integrity: CDC and DLQ

Happy-path CDC is not sufficient for a pilot that treats the index as investigation truth.

The indexer uses a **MongoDB DLQ**: quarantine failed records before Kafka commit, pause consumers when the DLQ store is unavailable, replay through a scheduler / Retry Now path, and surface an integrity banner in chat when the index is not trustworthy. Silently advancing offsets after Neo4j or embedding failure is exactly how “who denied what” drifts from MongoDB reality.

Operational maturity belongs in the architecture story, not as an afterthought. Local bootstrap and recovery steps are covered in [local development](https://github.com/sanjuthomas/policy-pilot/blob/main/docs/local-development.md).

---

## Operator Surface: Chat Skills Under the Same Policy Path

Chat is an **operator surface**, not an open-ended agent that invents tools at runtime.

Four payment mutation skills are available — **create**, **submit**, **approve**, and **cancel** — each on the same authorize → confirm → mutate path the domain APIs use:

1. Parse the request.
2. Load current domain state.
3. Dry-run OPA.
4. Present a Go / No Go confirmation card.
5. On **Go**, call payment-service; on deny or **No Go**, stop.

LLM routing selects the path (`RouterDecision`); handlers after that are scripted. Regex remains useful for slot parsing (IDs, amounts), not for replacing semantic intent determination when the model is available. See [intent determination](https://github.com/sanjuthomas/policy-pilot/blob/main/docs/intent-determination.md).

| Skill | Typical persona | Outcome on **Go** | Documentation |
|-------|-----------------|-------------------|---------------|
| Create | Payment creator / middle office | Draft payment on an approved instruction | [create-payment skill](https://github.com/sanjuthomas/policy-pilot/blob/main/docs/create-payment-skill.md) |
| Submit | Owning-LOB desk | DRAFT → SUBMITTED | [submit-payment skill](https://github.com/sanjuthomas/policy-pilot/blob/main/docs/submit-payment-skill.md) |
| Approve | Funding approver | SUBMITTED → APPROVED | [approve-payment skill](https://github.com/sanjuthomas/policy-pilot/blob/main/docs/approve-payment-skill.md) |
| Cancel | Middle-office creator | DRAFT or SUBMITTED → CANCELLED | [cancel-payment skill](https://github.com/sanjuthomas/policy-pilot/blob/main/docs/cancel-payment-skill.md) |

Investigation and action share one conversational loop, but skills remain supporting product shape. The lead story is still controls, durable why, and graph investigation.

---

## External Stress Test: Adversarial Architecture Review

Operational experience encoded as policy is one credibility signal. Independent critique of the architecture is another.

I ran an adversarial architecture review with a frontier model (**Claude Opus**) against production-shaped constraints. The consensus score landed around **8.0 / 10**: clear boundaries (OIDC → OPA → single-store writes → CDC/graph, fail-closed skills), with residual P1 findings kept visible rather than scored away.

That pairing is intentional: **experience encoded as Rego**, plus **external review that resists marketing language**.

---

## Observability and SLOs

A pilot that cannot be measured remains a narrative, not a deployment.

Policy Pilot exports OTLP into a Grafana-centric mesh (Prometheus, Loki, Tempo) with an OpenSLO catalog: chat answer success and latency, non-downvote quality, authorization evaluate latency, skill funnel outcomes, indexer consumer health, and DLQ depth. Clean-slate seeding and chat regression keep the environment repeatable. See the [observability guide](https://github.com/sanjuthomas/policy-pilot/blob/main/docs/observability.md).

More on the operating model in [Metrics-First Observability in Practice](/posts/2026-01-15-metrics-first-observability-in-practice/).

---

## What You Can Ask (and Do)

**Controls / live policy**

- What is the funding approval policy?
- Who may approve payments over $25B for FICC?

**Graph / durable why**

- Are there instances of approving each other’s instructions?
- Who approved instruction X, and why was it allowed?

**Skills**

- Create / submit / approve / cancel a payment — each with OPA preflight and Go / No Go.

The runnable reference is [policy-pilot](https://github.com/sanjuthomas/policy-pilot). A curated question bank is in [sample questions](https://github.com/sanjuthomas/policy-pilot/blob/main/docs/sample-questions.md).

---

## Related Reading

- [The SEC Filing Intelligence Stack](/posts/2026-06-20-sec-filings-chat/) — event-driven RAG for public filings and cited answers.
- [Metrics-First Observability](/posts/2026-01-01-metrics-first-observability/) and [in practice](/posts/2026-01-15-metrics-first-observability-in-practice/) — the operating model behind the SLO mesh.
- [US Domestic Payment Rails](/posts/2025-12-01-us-domestic-payment-rails/) — why SSI and funding approval matter in the cash leg.
)
