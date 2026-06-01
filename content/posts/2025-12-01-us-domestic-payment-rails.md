---
title: "U.S. Domestic Payment Rails"
description: "Understanding U.S. Domestic Payment Rails: ACH, Fedwire, RTP, and FedNow"
date: 2025-12-01
type: posts
tags: ["Payments", "Financial Services", "Distributed Systems", "Architecture"]
draft: false
---

*How money actually moves between banks in the United States.*

## Introduction

Every day, trillions of dollars move through the U.S. financial system.

Payroll is deposited through ACH. Large-value interbank transfers settle through Fedwire. Consumers and businesses increasingly rely on RTP and FedNow for instant payments.

To most people, these systems appear similar: money leaves one account and arrives in another.

Under the hood, however, these payment rails operate very differently. They make different trade-offs around:

* Settlement model
* Payment finality
* Availability
* Infrastructure ownership
* Messaging standards (see [ISO 20022](#iso-20022-the-future-of-payments-messaging) below)

Understanding these differences is essential for architects, developers, treasury professionals, and anyone working in payments.

The following map provides a simplified view of the major domestic payment rails used in the United States.

> ![U.S. Domestic Payment Rails](/images/us-domestic-payment-rails.png)

---

## Reading the Map

The map classifies payment rails using several important dimensions.

### Settlement Model

The first distinction is how payments settle.

#### Gross Settlement

Each payment settles individually.

When Bank A sends $1 million to Bank B, that payment settles immediately.

**Benefits:**

* Immediate finality
* Lower settlement risk
* Reduced counterparty exposure

**Examples:**

* Fedwire
* RTP
* FedNow

---

#### Net Settlement

Payments accumulate over time and settle periodically.

Instead of settling every payment individually:

```text
Bank A owes Bank B = $100M
Bank B owes Bank A = $95M

Net Settlement = $5M
```

**Benefits:**

* Liquidity efficient
* Lower operational costs
* Suitable for high-volume payments

**Example:**

* ACH

---

### Payment Finality

A payment is only valuable if all parties agree that it is complete.

#### Immediate Finality

Once settled, the payment is considered final and generally cannot be reversed, with limited exceptions (for example, Fedwire recall scenarios).

Examples:

* Fedwire
* RTP
* FedNow

---

#### Delayed Finality

Settlement occurs later after clearing and netting activities. On ACH, returns and reversals can still occur for days or weeks after a payment appears to have posted—for example, NSF returns or unauthorized debit claims under NACHA rules.

Example:

* ACH

---

### Infrastructure Ownership

The United States operates both public and private payment infrastructure.

#### Public Infrastructure

Operated by the Federal Reserve.

Examples:

* Fedwire
* FedNow
* FedACH

Public infrastructure promotes broad access and financial stability.

---

#### Private Infrastructure

Operated by industry-owned organizations.

Examples:

* RTP
* ACH clearing through The Clearing House's EPN

RTP is operated by The Clearing House, which is owned by a consortium of major U.S. banks. ACH is governed by NACHA operating rules, but clearing and settlement run through both FedACH (public) and private operators such as EPN.

---

### Availability

The green background in the diagram identifies payment rails that are always available.

#### Always-On Rails

Available:

* 24 hours a day
* 7 days a week
* 365 days a year

Examples:

* RTP
* FedNow

These rails support modern use cases such as:

* Instant person-to-person payments
* Instant business payments
* Immediate account-to-account transfers

---

#### Business-Day Rails

Examples:

* ACH
* Fedwire

Although Fedwire provides immediate settlement, it only operates during defined operating windows.

This distinction is frequently misunderstood.

Many people assume Fedwire and FedNow are similar because both provide real-time settlement.

The key difference is:

| Rail    | Settlement | Availability   |
| ------- | ---------- | -------------- |
| Fedwire | Real-Time  | Business Hours |
| FedNow  | Real-Time  | 24×7×365       |

---

## ACH

### Classification

| Property       | Value                          |
| -------------- | ------------------------------ |
| Settlement     | Net (Batch)                    |
| Finality       | Delayed                        |
| Infrastructure | Mixed (FedACH / EPN, NACHA)    |
| Availability   | Business Days                  |

Standard ACH typically settles on a T+1 or T+2 schedule. Same Day ACH, introduced in 2016, offers faster settlement within defined submission windows and cutoffs.

### Typical Use Cases

* Payroll
* Direct Deposit
* Bill Payments
* Tax Payments
* Vendor Payments
* Recurring Transfers

### Why ACH Exists

ACH optimizes for scale and cost efficiency.

Every day, enormous volumes of low-value payments move through ACH. Most ACH entries are debit-pull or credit-push batch transactions—unlike RTP and FedNow, which are credit-push and settle in real time.

The trade-off is that settlement is not immediate.

ACH remains the workhorse of the U.S. banking system.

---

## Fedwire

### Classification

| Property       | Value          |
| -------------- | -------------- |
| Settlement     | Gross (RTGS)   |
| Finality       | Immediate      |
| Infrastructure | Public         |
| Availability   | Business Hours |

### Typical Use Cases

* Treasury Settlements
* Large Corporate Payments
* Interbank Funding
* High-Value Transfers

### Why Fedwire Exists

Fedwire optimizes for certainty.

When a payment settles through Fedwire, it is final.

This makes it the preferred rail for high-value and time-critical payments. There is no practical upper limit comparable to instant-payment rails; institutions use Fedwire when amount and finality matter more than 24×7 availability.

For many institutions, Fedwire serves as the backbone for liquidity management and interbank funding.

---

## RTP

### Classification

| Property       | Value             |
| -------------- | ----------------- |
| Settlement     | Gross (Real-Time) |
| Finality       | Immediate         |
| Infrastructure | Private           |
| Availability   | 24×7×365          |

### Typical Use Cases

* Instant P2P Payments
* Instant B2B Payments
* Account-to-Account Transfers
* Bill Payments

### Why RTP Exists

The RTP network was designed to bring instant payments to the U.S. banking system.

Unlike ACH, settlement occurs in seconds.

Unlike card networks, RTP is a credit-push model—the sender initiates payment, reducing certain fraud vectors associated with debit-pull models.

RTP also supports rich ISO 20022 messaging and Request for Payment (RfP) workflows.

---

## FedNow

### Classification

| Property       | Value             |
| -------------- | ----------------- |
| Settlement     | Gross (Real-Time) |
| Finality       | Immediate         |
| Infrastructure | Public            |
| Availability   | 24×7×365          |

### Typical Use Cases

* Consumer Instant Payments
* Business Instant Payments
* Disbursements
* Account-to-Account Transfers

### Why FedNow Exists

FedNow extends the Federal Reserve's payment infrastructure into the always-on era.

Conceptually, many people think of FedNow as:

> Fedwire redesigned for a 24×7 world.

While technically different systems, that mental model helps explain the purpose behind FedNow.

RTP and FedNow are not interoperable today—participating institutions may connect to one or both networks. FedNow offers broad Federal Reserve reach; RTP benefits from early private-network adoption by major banks. Both use ISO 20022 and credit-push settlement, but coverage and integration paths differ by institution.

---

## ISO 20022: The Future of Payments Messaging

One of the most important trends in modern payments is the adoption of ISO 20022.

ISO 20022 enables:

* Rich payment data
* Structured remittance information
* Improved reconciliation
* Better interoperability between institutions

The major domestic rails currently support ISO 20022 as follows:

| Rail    | ISO 20022 Support |
| ------- | ----------------- |
| RTP     | Yes               |
| FedNow  | Yes               |
| Fedwire | Yes               |
| ACH     | Partial           |

For architects and developers, understanding ISO 20022 is becoming just as important as understanding the payment rails themselves.

---

## A Simple Evolution of U.S. Payments

The history of U.S. domestic payments can be viewed as a progression toward faster settlement and broader availability.

```text
ACH
  ↓

Same Day ACH
  ↓

Fedwire
  ↓

RTP / FedNow
```

Or stated another way:

```text
Batch Settlement
      ↓

Faster Batch (Same Day ACH)
      ↓

Real-Time Settlement
      ↓

Real-Time Settlement
+
Always-On Availability
```

---

## Conclusion

Each payment rail solves a different problem.

* ACH optimizes for scale and efficiency.
* Fedwire optimizes for certainty and finality.
* RTP optimizes for speed and modern banking experiences.
* FedNow brings always-on instant payments to the Federal Reserve ecosystem.

Together, these systems form the foundation of modern U.S. domestic payments.

In future articles, we will dive deeper into each rail, covering message formats, settlement flows, ISO 20022 messages, failure scenarios, APIs, and reference implementations.
