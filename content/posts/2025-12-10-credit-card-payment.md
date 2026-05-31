---
title: "How Credit Card Payments Work: An Introduction to the Card Payment Ecosystem"
description: "Visa and Mastercard don't move your money—they coordinate the institutions that do. Authorization, clearing, settlement, interchange and the MDR, ISO 8583, and the three- vs four-party model, for engineers in financial services."
date: 2025-12-10
type: posts
tags: ["Payments", "Financial Services", "Distributed Systems", "Event-Driven Architecture"]
draft: false
---

Most engineers who join the payments industry arrive with a simple mental model:

> Visa and Mastercard move money from my account to the merchant.

That model is wrong in an important way.

A more accurate way to think about it is this:

> Card networks do not move money. They route messages and coordinate the institutions that move money.

The actual movement of funds happens between issuing banks and acquiring banks, through settlement banks and clearing houses, often hours or days after you have walked out of the store.

Card payments are among the most widely used payment instruments in the modern economy, yet many software engineers entering financial services have only a partial understanding of how a transaction is authorized, cleared, and settled. The ecosystem is a complex web of networks, issuing banks, acquiring banks, processors, gateways, merchants, and settlement institutions—and each participant performs a distinct role in the lifecycle of a single swipe.

This post provides a structured overview of that ecosystem, explains the three major stages of a transaction, and examines the economic relationships among the participants.

---

## Overview of the Card Payment Ecosystem

The modern card ecosystem is commonly referred to as the **four-party model**, which is used by Visa and Mastercard.

The four principal participants are:

1. Cardholder
2. Merchant
3. Acquiring Bank (Acquirer)
4. Issuing Bank (Issuer)

The card network acts as the intermediary that connects issuers and acquirers. Notice that the network itself is not one of the four parties—it is the rail they communicate over.

Additional participants frequently include:

* Payment Gateway
* Payment Processor
* Settlement Bank
* Point-of-Sale (POS) Providers
* Payment Facilitators (PayFacs)

### Ecosystem Diagram

![Card Payment Ecosystem](/images/card-payment.png)

---

## Transaction Lifecycle Overview

A card transaction proceeds through three major phases:

1. Authorization
2. Clearing
3. Settlement

These phases are logically distinct and occur at different points in time. Conflating them is the source of most confusion for engineers new to payments.

| Phase         | Primary Objective                                  | Timing                 |
| ------------- | -------------------------------------------------- | ---------------------- |
| Authorization | Determine whether a transaction should be approved | Real-time              |
| Clearing      | Exchange finalized transaction records             | Minutes to hours later |
| Settlement    | Transfer funds between financial institutions      | Hours to days later    |

A useful way to think about these phases is:

| Phase         | Question Being Answered             |
| ------------- | ----------------------------------- |
| Authorization | Can this transaction occur?         |
| Clearing      | What transaction actually occurred? |
| Settlement    | How should funds be transferred?    |

The key takeaway: the moment the terminal says "Approved," no money has moved. That comes much later.

---

## Authorization

### Purpose

Authorization is the process by which the issuer determines whether a transaction should be approved.

Authorization is fundamentally a risk decision rather than a movement of money.

The issuer evaluates:

* Card validity
* Account status
* Available credit
* Fraud indicators
* Velocity limits
* Regulatory controls

If the transaction satisfies the issuer's risk criteria, an approval response is returned.

### Characteristics

| Attribute        | Description           |
| ---------------- | --------------------- |
| Timing           | Real-time             |
| Duration         | Typically 1–3 seconds |
| Money Movement   | No                    |
| Customer Waiting | Yes                   |

### Typical Authorization Sequence

1. Customer presents card.
2. Merchant captures transaction details.
3. Gateway/Processor formats authorization request.
4. Acquirer forwards request to network.
5. Network routes request to issuer.
6. Issuer performs risk evaluation.
7. Issuer returns approval or decline.
8. Response propagates back to merchant.

An approved transaction generally results in an authorization hold against the cardholder's available credit.

### Authorization Versus Capture

Approval is not the same as collecting the money. Engineers building payment systems need to distinguish two separate actions:

* **Authorization** places a *hold* on the cardholder's available credit. No funds move; the issuer simply earmarks the amount.
* **Capture** is the merchant's later instruction to actually collect the held funds. Capture is what feeds into clearing.

This split is the source of several real-world behaviors:

* **Delayed capture** — a hotel or rental car authorizes at check-in but captures the final amount at check-out.
* **Partial capture** — the captured amount is less than the authorized amount (e.g., an out-of-stock item).
* **Authorization reversal** — the merchant releases a hold it will not capture.
* **Hold expiry** — an uncaptured authorization eventually falls off, which is why a pending charge can silently disappear.

### A Note on Message Formats

Authorization requests and responses have historically been exchanged using the **ISO 8583** message standard, which defines the bitmap-based format that powers most card networks. The industry is gradually migrating toward **ISO 20022**, the same modern, XML/JSON-friendly standard reshaping bank-to-bank messaging. If you have worked with SWIFT messaging, this evolution will feel familiar.

### Authorization Diagram

![Card Payment Authorization](/images/card-authorization-flow.png)

---

## Clearing

### Purpose

Clearing is the process by which finalized transaction records are exchanged among participants after authorization has occurred.

Where authorization answers *"Can the transaction occur?"*, clearing answers *"What transaction actually occurred?"*

Clearing records frequently contain more information than the original authorization request.

Examples include:

* Final transaction amount
* Taxes
* Tips
* Currency information
* Merchant data
* Settlement references

### Characteristics

| Attribute        | Description                          |
| ---------------- | ------------------------------------ |
| Timing           | Minutes to hours after authorization |
| Processing Style | Typically batch-oriented             |
| Money Movement   | No                                   |
| Customer Waiting | No                                   |

### Multi-Network and Multi-Issuer Environment

Merchants generally do not create separate clearing files for each network or issuing bank.

Instead, processors aggregate captured transactions and:

1. Identify the network using BIN/IIN ranges (Bank Identification Number / Issuer Identification Number—the leading digits of the card that map to the issuing institution).
2. Group transactions by network.
3. Generate network-specific clearing files.
4. Submit clearing files to the appropriate card networks.

Networks subsequently route clearing records to the relevant issuing institutions.

### Clearing Diagram

![Card Payment Clearing](/images/card-clearing-flow.png)

---

## Settlement

### Purpose

Settlement is the process by which funds are transferred between issuing and acquiring institutions for cleared transactions.

Settlement is the first stage in the transaction lifecycle during which money actually moves.

### Settlement Participants

Settlement generally involves:

* Issuing Banks
* Acquiring Banks
* Card Networks
* Settlement Banks
* Clearing Houses

### Net Settlement versus Gross Settlement

Card networks predominantly employ **net settlement** rather than **gross settlement**.

Under **gross settlement**, every obligation is settled individually—each issuer-to-acquirer obligation results in a separate transfer. This is operationally inefficient and liquidity-hungry at scale.

Under **net settlement**, obligations are netted across participants, and only each participant's net debit or credit position is settled.

The power of netting becomes clear when you remember that a single institution is often *both* an issuer and an acquirer. Its outgoing and incoming obligations offset each other:

| Participant | Owes (as issuer) | Receives (as acquirer) | Net position   |
| ----------- | ---------------- | ---------------------- | -------------- |
| Bank A      | $5M              | $4M                    | Owes $1M       |
| Bank B      | $3M              | $4M                    | Receives $1M   |
| Bank C      | $2M              | $2M                    | $0 (no move)   |

Gross settlement would move $10M out and $10M in across these banks. With netting, the system reaches the same end state by moving just **$1M**—from Bank A to Bank B. Bank C, perfectly offset, transfers nothing at all.

This dramatically reduces:

* Liquidity requirements
* Settlement risk
* Operational complexity

### Settlement Workflow

1. Clearing is completed.
2. Network calculates participant obligations.
3. Net positions are determined.
4. Settlement instructions are generated.
5. Settlement bank transfers funds.
6. Acquirers receive settlement funds.
7. Merchants are credited according to payout schedules.

### Settlement Diagram

![Card Payment Settlement](/images/card-settlement-flow.png)

---

## Economics of Card Payments

The card ecosystem involves several distinct revenue streams.

A merchant accepting a card payment generally receives less than the transaction amount. The difference is commonly referred to as the **Merchant Discount Rate (MDR)**.

The MDR is not a single fee—it is the sum of the fees collected by the participants along the way:

> **MDR ≈ Interchange + Network Assessments + Processor Margin**

Of these, the **interchange fee is by far the largest component**, which is why so much regulatory and competitive attention focuses on it.

### Interchange Fee

Paid to the **Issuing Bank**.

Purpose:

* Credit risk compensation
* Fraud risk compensation
* Rewards funding
* Credit program operation

### Network Assessment Fee

Paid to the network (**Visa**, **Mastercard**).

Purpose:

* Network operation
* Routing infrastructure
* Standards development
* Settlement coordination

### Processor Fee

Paid to the **Processor**, **Gateway Provider**, or **Payment Facilitator**.

Purpose:

* Transaction processing
* Merchant services
* Gateway functionality
* Reporting and reconciliation

---

## Why American Express Is Different

Visa and Mastercard primarily operate under a four-party model. American Express historically operates under a **three-party model**.

### Visa and Mastercard

```text
Cardholder
    ↓
Issuer
    ↓
Network
    ↓
Acquirer
    ↓
Merchant
```

The issuer and network are separate organizations.

### American Express

```text
Cardholder
    ↓
American Express
    ↓
Merchant
```

American Express historically performs multiple functions:

* Network Operator
* Card Issuer
* Merchant Acquirer

This vertically integrated structure provides American Express with greater control over the entire transaction lifecycle.

The distinction is not absolute, however. Through its Global Network Services (GNS) program, American Express also lets partner banks issue Amex-branded cards and acquire merchants, which blends elements of the four-party model into what is otherwise a closed loop.

### Key Differences

| Function                   | Visa         | Mastercard   | American Express |
| -------------------------- | ------------ | ------------ | ---------------- |
| Operates Network           | Yes          | Yes          | Yes              |
| Issues Cards               | Typically No | Typically No | Yes              |
| Extends Credit             | No           | No           | Yes              |
| Acquires Merchants         | No           | No           | Yes              |
| Owns Customer Relationship | No           | No           | Yes              |

---

## Discover and the Capital One Acquisition

Discover historically occupied a position between the Visa/Mastercard and American Express models.

Like American Express, Discover:

* Operated its own network
* Issued cards directly
* Managed customer relationships

However, Discover also participated in debit and ATM network operations through the PULSE network.

The acquisition of Discover by Capital One created a unique structure within the United States payments industry.

The combined organization now possesses:

* A major issuing bank
* A card network
* Consumer lending operations
* Merchant payment infrastructure

This combination more closely resembles the vertically integrated model traditionally associated with American Express than the federated model used by Visa and Mastercard.

---

## Conclusion

The card payment ecosystem consists of a network of specialized institutions that cooperate to authorize transactions, exchange financial records, transfer funds, and manage risk.

Understanding the distinction between authorization, clearing, and settlement provides the foundation for understanding virtually every aspect of modern card payments. These concepts are essential for engineers designing payment systems, reconciliation platforms, fraud controls, merchant services, or card-processing infrastructure.

This post deliberately stayed at the lifecycle and ecosystem level. Several topics that build directly on this foundation—chargebacks and disputes, reconciliation, card-present versus card-not-present flows, tokenization, and PCI DSS compliance—are worth a dedicated treatment of their own.

Although the customer experience appears instantaneous, a card transaction is the result of a carefully coordinated sequence of risk decisions, record exchanges, and settlement processes that may continue long after the customer has left the point of sale.
