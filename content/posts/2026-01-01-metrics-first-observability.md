---
title: "Metrics-First Observability: A Conceptual Model"
description: "Why metrics should lead observability—and what to measure across business, workflow, platform, and reliability dimensions—in modern distributed systems."
date: 2026-01-01
type: posts
tags: ["Observability", "Distributed Systems", "SRE", "Event-Driven Architecture"]
draft: false
---

This is the first part of a two-part series on metrics-first observability. This part covers *why* metrics should lead and *what* to measure. Part two, [Metrics-First Observability in Practice](/posts/2026-01-15-metrics-first-observability-in-practice/), covers the operating model and engineering discipline needed to run it.

## Why Logs Should Support, Not Lead

Observability rests on three pillars: **metrics**, **traces**, and **logs**. Each plays a distinct role. **Metric-first** observability means dashboards and alerts are driven by metrics; traces and logs are the drill-down layers you use once a signal tells you something is wrong.

In many engineering organizations, observability starts with logs. When something breaks, engineers open log search tools, filter by timestamps, scan error messages, and try to reconstruct what happened.

Logs are useful. But they should not be the primary way to understand whether a system is healthy.

In modern distributed systems, especially event-driven systems, observability should start with metrics. Metrics should be treated as first-class citizens. Traces show the path of a request or event across services. Logs provide supplementary context to debug incidents, explain failures, and support root-cause analysis.

In simple terms:

> Metrics should tell us that something is wrong. Traces and logs should help us understand why.

## The Problem with Log-First Operations

A log-first operating model does not scale well.

Logs are often noisy, expensive, and difficult to interpret. They are useful when an engineer already knows where to look, but they are not always effective at showing the overall health of a system. Many observability vendors charge based on ingestion and retention, so teams often pay to index large volumes of data that are rarely searched during incidents. Logs can also introduce privacy and compliance risks when PII or sensitive business data is written accidentally—and that data may be indexed, replicated, and retained longer than intended.

In high-throughput, event-driven systems, excessive logging can introduce synchronous disk I/O, network serialization, and payload-processing overhead that competes with core business processing under peak load.

Consider an algorithmic trading system. The system may process thousands or millions of market events, orders, and executions in a very short time window. If every event produces heavy log payloads, the logging path itself can become a bottleneck. In extreme cases, the system may spend more time formatting, serializing, shipping, and storing logs than processing the actual business event.

This does not mean logs should be removed. It means logs should be intentional, lightweight, and supplementary. High-volume systems should rely on well-designed metrics to understand health, throughput, latency, and failure rates, while using logs selectively for investigation.

> A log-first system often tells the story only after the damage is done.

Metric-first observability changes that model. Instead of waiting for someone to search through logs across services, queues, and databases during an incident, the system continuously exposes its health through meaningful signals.

## What Metric-First Observability Means

Metric-first observability means every critical system, service, and workflow is designed to emit meaningful metrics from the beginning.

These metrics should cover multiple dimensions:

- **Business metrics** — payments processed, payments failed, payments delayed, payments pending approval
- **Workflow metrics** — stuck states, retry counts, SLA breaches, lifecycle transition delays
- **Platform metrics** — Kafka lag, consumer errors, API latency, database latency. For services, the **RED** method (Rate, Errors, Duration) is a useful starting point; for infrastructure resources, **USE** (Utilization, Saturation, Errors) applies well.
- **Reliability metrics** — availability, error rate, saturation, recovery time

The goal is not just to know whether servers are running. The goal is to know whether the business process is working as expected.

> The best observability systems do not only tell us whether the platform is running. They tell us whether the business process is meeting its promise.

## Why This Matters More in Event-Driven Systems

Metric-first observability becomes even more important in event-driven distributed systems.

In a synchronous system, failure is often visible immediately. A request fails, an API returns an error, and the caller knows something went wrong.

But in an event-driven system, everything operates asynchronously. A producer may successfully publish an event, but that does not mean the business workflow has completed. A consumer may face delays, or a message may retry automatically; meanwhile, a downstream system rejects the event, leaving a payment quietly stuck in an intermediate state.

In asynchronous systems, success at one step does not guarantee success of the workflow.

That is why metrics are critical. Metrics connect the dots across producers, topics, consumers, databases, and downstream systems.

For example, an event-driven platform should expose metrics such as:

- Events published per minute
- Events consumed per minute
- Consumer lag by topic and consumer group
- Event processing latency
- Retry count
- Dead-letter queue count
- Failed event count by reason
- Workflow completion time
- Business SLA breach count

In an event-driven architecture, the absence of an error does not mean the presence of success. Only metrics can prove that the workflow is moving forward.

## Metrics, Traces, and Logs

Metric-first does not mean traces or logs are unimportant. It means each pillar should play the right role.

A useful mental model is:

> Metrics are the dashboard. Traces are the map. Logs are the evidence.

Metrics show the health of the system. Traces show the path of a request or event. Logs explain the details.

When an alert fires, the engineer should first understand the impact through metrics and dashboards. Then traces and logs can help explain the root cause.

## Business SLO-Driven Metrics

Metric-first observability should also support business SLOs.

For example, in a payment platform, it is not enough to know that Kafka is healthy or that APIs are responding. The real question is:

> Are payments moving through the business workflow within the expected time?

For an STP payment flow, we may define a business SLO like this:

> 99.5% of eligible STP payments should be released within one minute.

That SLO can be translated into alertable metrics:

- Total STP payments processed
- STP payments released within one minute
- STP payments taking more than one minute
- Percentage of delayed STP payments
- Delayed payments by client, currency, region, rail, or payment type

An alert rule could be:

> If more than 0.5% of STP payments take longer than one minute to release within a defined rolling window, alert the rota.

This is powerful because the alert is based on business degradation, not just technical failure.

## Metric-First Observability for Change Data Capture

Change Data Capture is another strong example.

In a CDC pipeline, data moves asynchronously from a source system to a target system. The source write may succeed, the CDC connector may capture the change, the event may be published, and the target database may eventually apply it.

Each step can fail or lag independently.

That is why CDC observability should not depend only on logs. Logs can explain why a specific record failed, but metrics should tell us whether the pipeline is healthy.

Important CDC metrics include:

- Source table row count
- Target table row count
- Source-to-target row-count mismatch
- CDC lag
- Events captured per minute
- Events applied per minute
- Failed apply count
- Retry count
- Dead-letter count
- Last successful sync timestamp

A simple but powerful control is comparing the source and target row counts. If the source table has 1,000,000 rows and the target table has 999,500 rows, the system should not wait for an engineer to discover that manually.

A metric should capture the mismatch, and an alert should notify the rota. For example:

> If `abs(source_row_count - target_row_count)` exceeds a defined threshold for more than 15 minutes on a critical table, classify as **URGENT** and page the rota.

> In CDC pipelines, correctness is not just about whether events are moving. It is about whether the target system faithfully represents the source system.

## Conclusion

Metric-first observability changes how teams understand distributed systems. Instead of reconstructing events from logs after something breaks, teams define meaningful metrics upfront that continuously expose the health of business workflows, event pipelines, CDC processes, and platform components. Traces and logs still matter—they explain incidents and support debugging—but they should not be the starting point. In modern event-driven systems, observability should begin with metrics.

> The log should complete the story, not start it.

## Appendix

### High-Resolution vs Low-Resolution

|Area                 |High-Resolution Metrics                             |Low-Resolution Metrics                 |
|:--------------------|:---------------------------------------------------|:--------------------------------------|
| Meaning             | Metrics captured at very small intervals           | Metrics aggregated over larger intervals |
| Typical interval    | Every 5, 10, 15, or 30 seconds                     | Every 5 minutes, hourly, daily, or monthly |
| Main purpose        | Recent incident debugging and real-time operations | Trend analysis, capacity planning, and reporting |
| Example             | API latency every 10 seconds                       | Hourly average API latency |
| Example             | Kafka consumer lag every 30 seconds                | Daily maximum Kafka consumer lag |
| Example             | JVM memory usage every 15 seconds                  | Daily average JVM memory usage |
| Example             | Error rate per minute                              | Monthly error-rate trend |
| Retention           | Usually 15–30 days                                 | Usually 3 months to multiple years |
| Cost                | Higher storage and query cost                      | Lower storage and query cost |
| Best for            | “What happened during the incident?”               | “How is the system trending over time?” |

### Blackbox vs Whitebox

|Area              |Blackbox Metrics           |Whitebox Metrics                |
|------------------------------------|------------------------------------------------|--------------------------------------------------|
| Meaning | Metrics observed from outside the system | Metrics emitted from inside the application or service |
| Main question answered | “Is the system working for users?” | “Why is the system behaving this way?” |
| Perspective | User/client/external perspective | Internal application/platform perspective |
| Examples | API availability, response time, error rate | JVM memory, thread count, DB pool usage, Kafka lag |
| Payment example | Can a payment be submitted successfully? | How long did validation, approval, and release steps take? |
| CDC example | Is the target table updated within SLA? | CDC lag, retry count, failed apply count, DLQ count |
| Best for | Detecting user-visible or business-visible impact | Diagnosing internal causes and bottlenecks |
| Alert usage | Good for urgent/page alerts | Good for investigation, warning, and root-cause analysis |
| Limitation | May not explain the root cause | May miss real user/business impact if used alone |
| Simple summary | Measures outcomes | Explains internals |

---

**Continue to part two:** [Metrics-First Observability in Practice](/posts/2026-01-15-metrics-first-observability-in-practice/) — the operating model, naming, cardinality, retention, alerting, and escalation discipline behind a metrics-first program.
