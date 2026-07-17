---
title: "Policy Pilot: Maker–Checker, Durable Why, and Graph Investigation for Cash SSI"
description: "An FDE-shaped reference pilot for Standard Settlement Instructions — production-grade OPA controls, OIDC identity, versioned single-store writes, security events that preserve why, Neo4j investigation, fail-closed mutation skills, and a DLQ-backed index path."
date: 2026-07-17
type: posts
tags: ["OPA", "Neo4j", "Architecture", "Financial Services", "OIDC", "Event-Driven Architecture", "RAG", "Observability"]
draft: false
---

*The interesting part of an enterprise AI pilot is not the chat box. It is whether the system encodes real controls — maker–checker, segregation of duties, durable authorization why, and a knowledge graph that can answer structural compliance questions — before the LLM ever speaks.*

Most “enterprise AI” demos stop at retrieval. [**Policy Pilot**](https://github.com/sanjuthomas/policy-pilot) is the opposite bet: a **customer-pilot / reference deployment** shape for the cash leg of Standard Settlement Instructions (SSI). It is AI-native where that helps (intent routing, explanation), and enterprise-shaped where money and policy matter (identity, Rego, transactional aggregates, CDC, graph, operator skills).

I built it as an **FDE-style** showcase — the kind of end-to-end system you stand up with a customer team under real bank constraints, not a chatbot bolted onto a PDF.

![Policy Pilot — end-to-end architecture](/images/policy-pilot-architecture.png)

---

## The Credibility Layer: Production-Grade OPA

The part I’m most proud of is not the UI. It’s the **OPA policy pack**.

Twenty years in back and middle office leave scars: mutual approvals that shouldn’t happen, desks approving outside their book, juniors green-lighting their manager’s work, amount authority that looks fine until it isn’t. Those patterns are encoded as Rego — SoD, reporting-line inversion, desk LOB boundaries, amount clubs, lifecycle status, four-eyes. The same engine guards mutations and answers *who can / who did / why*.

You don’t invent that from a tutorial. You earn it on the floor.

Tone matters here: this is **production-grade control shape** and battle-tested policy design — not a claim that the demo stack is live in a bank. The point is that every Go / No Go and every investigation answer sits on policy that looks like operations, not toy allow-all Rego.

---

## Maker–Checker and Four Eyes on Instructions

Lead control story: **maker–checker** and **four eyes** on instructions, framed the way a supervision desk would recognize it (FINRA-aligned language, not “six eyes” theater).

Instruction path:

1. Middle office **creates** and **submits** (same creator class for draft + submit).
2. A profit-center / second pair of eyes **approves**.

Payments are richer — draft → submit → approve → cancel — but the headline is still four-eyes on the instruction. The payment lifecycle is where operator skills and funding desks live; the instruction is where the control narrative starts.

---

## Identity First: OIDC Subject → OPA

Policy is only as honest as the subject it evaluates.

Policy Pilot uses **ZITADEL** as an OIDC/OAuth2 IdP. Services validate JWT Bearer tokens via OIDC discovery. The subject carries the attributes OPA needs — roles, LOB, covering LOBs, amount clubs, supervisor — from **identity metadata**, not from a hard-coded app map.

Runtime directory reads come from ZITADEL; `users.yaml` is seed-only. Chat and mutation skills act as the **logged-in user** (with service on-behalf-of where domain APIs require it). The chain is deliberate:

**OIDC subject → authorization-service → OPA → allow / deny + basis**

Today’s UIs still use a demo login facade on top of ZITADEL sessions. The policy model does not depend on that facade: attributes already live in the IdP, and authorization-code + PKCE is a natural hardening step later — not a rewrite of Rego.

---

## Security Events as System of Record for *Why*

Most banks can answer *who* and *when*. Almost none keep a durable *why*.

In Policy Pilot, **security events are first-class**. Every authorized (and denied) mutation persists an authorization block with the domain aggregate — `allow_basis`, violations, summary — not a disposable log line. Domain services write the versioned instruction or payment **and** its security event in one Mongo transaction. That trail is what chat later formats as Who / When / Why, and what the indexer projects into Neo4j for investigation.

Authorization is not a side effect. It is part of the record.

---

## Keep Transaction Boundaries Stupid (KISS)

Transactional services write to **one store only**: MongoDB. Documents are **versioned**. There is no Mongo + Kafka distributed transaction, no 2PC, no “saga that pretends to be ACID.”

Domain services focus on getting the aggregate saved correctly. Kafka Connect CDC fans out **downstream**. The indexer, graph, and vectors are eventually consistent projections — not participants in the write.

That boundary is boring on purpose. Correctness lives at the aggregate; integration stays asynchronous.

---

## Why the Knowledge Graph Matters

Vector search finds similar event text. It does **not** ground structural questions.

Without a **Neo4j knowledge graph**, you cannot honestly answer:

> Are there users approving each other’s instructions?

That is hops across users, instructions, and approval edges — creator/approver chains, subordinate approvers, shared routes — not “nearest neighbor prose.” Graph and vector are complementary: relationships vs semantic narrative. Compliance investigation needs both; relationship questions need the graph.

Latest instruction and payment versions are marked with a `CURRENT` relationship so inventory and show-by-id answers do not accidentally mix historical versions.

---

## Index Path Honesty: CDC and DLQ

Happy-path CDC is not enough for a pilot that claims investigation truth.

The indexer uses a **Mongo DLQ**: quarantine failed records before Kafka commit, pause consumers when the DLQ store is down, replay through a scheduler / Retry Now path, and surface an integrity banner in chat when the index is not trustworthy. Silent offset advance on Neo4j or embedding failure is exactly how “who denied what” drifts from Mongo reality.

Ops maturity is part of the architecture story, not an appendix.

---

## Operator Surface: Chat Skills Under the Same Policy Path

Chat is an **operator surface**, not a free-form agent that invents tools.

Four payment mutation skills are live — **create**, **submit**, **approve**, and **cancel** — each on the same authorize → confirm → mutate path the domain APIs use:

1. Parse the request.
2. Load current domain state.
3. Dry-run OPA.
4. Show a Go / No Go confirmation card.
5. On **Go**, call payment-service; on deny or **No Go**, stop.

LLM routing picks the path (`RouterDecision`); handlers after that are scripted. Regex is for slots (IDs, amounts), not for pretending to understand intent when the model is available.

| Skill | Typical persona | Outcome on **Go** |
|-------|-----------------|-------------------|
| Create | Payment creator / middle office | Draft payment on an approved instruction |
| Submit | Owning-LOB desk | DRAFT → SUBMITTED |
| Approve | Funding approver | SUBMITTED → APPROVED |
| Cancel | Middle-office creator | DRAFT or SUBMITTED → CANCELLED |

Investigation and action share one loop — but skills are supporting product shape, not the only lead. The lead is still controls, why, and graph.

---

## External Stress Test: Adversarial Architecture Review

Human scars in policy are one credibility signal. Machine critique of the architecture is another.

I ran an adversarial architecture review with a frontier model (**Claude Opus**) against production-shaped constraints — not a vanity pass. The consensus score landed around **8.0 / 10**: teachable boundaries (OIDC → OPA → single-store writes → CDC/graph, fail-closed skills), with residual P1s called out honestly rather than scored away.

That is the pairing I want the pilot to demonstrate: **experience encoded as Rego**, plus **external review that refuses marketing language**.

---

## Observability and SLOs

A pilot you cannot measure is a story, not a deployment.

Policy Pilot fans OTLP into a Grafana-centric mesh (Prometheus, Loki, Tempo) with an OpenSLO catalog: chat answer success and latency, non-downvote quality, authorization evaluate latency, skill funnel outcomes, indexer consumer health and DLQ depth. Clean-slate seeding and chat regression keep the demo re-runnable — operability proof, not a one-off laptop theater.

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

The runnable reference is [policy-pilot](https://github.com/sanjuthomas/policy-pilot). Sample questions and skill write-ups live under `docs/`.

---

## Narrative Order (Why This Shape)

If you only remember one sequencing from this post, use this:

1. **OPA scars** — production-grade policy from real BO/MO experience  
2. **Maker–checker / four eyes** — instruction control story  
3. **OIDC subject** — identity attributes feed OPA  
4. **Why-as-SoR** — security events preserve authorization basis  
5. **KISS writes** — one store, versioned aggregates, no 2PC  
6. **Graph investigation** — structural questions vectors cannot ground  
7. **DLQ** — index honesty when downstream fails  
8. **Chat skills** — AI where it helps; deterministic where money moves  
9. **External review + SLOs** — stress-tested and measurable  

That is the FDE shape of the problem: ship a working system under real constraints — not another RAG demo.

---

## Related Reading

- [The SEC Filing Intelligence Stack](/posts/2026-06-20-sec-filings-chat/) — event-driven RAG for public filings and cited answers.
- [Metrics-First Observability](/posts/2026-01-01-metrics-first-observability/) and [in practice](/posts/2026-01-15-metrics-first-observability-in-practice/) — the operating model behind the SLO mesh.
- [US Domestic Payment Rails](/posts/2025-12-01-us-domestic-payment-rails/) — why SSI and funding approval matter in the cash leg.
)
