---
title: "Webhooks are a product, not a callback"
date: 2026-05-20
summary: "The delivery guarantees and operating model behind a trustworthy event platform."
tags: ["event-driven", "webhooks", "platform-engineering"]
---

A webhook endpoint is simple; a webhook platform is an agreement with another engineering team. Consumers expect timely delivery, useful recovery behaviour, and a way to answer what happened.

## Preserve the event, then deliver it

Store an immutable event record before delivery. Delivery attempts should reference that record, carry an idempotency identifier, and be independently inspectable. This separates event creation from delivery success.

## Use bounded, intentional retries

Retries need exponential backoff, a maximum attempt policy, and a terminal state that creates an actionable operational queue. Blind retries amplify downstream incidents; classified retries create recovery options.

## Design for unequal consumers

Prioritisation and queue isolation prevent a noisy consumer from delaying time-sensitive events. Track delivery age per consumer and expose enough audit history for support and partner engineers to diagnose failures together.

## Operate the contract

Document payload versions, signing, retry semantics, and status codes. Instrument delivery latency, endpoint response classes, and dead-letter volume. A reliable webhook platform turns an integration boundary into a capability teams can depend on.
