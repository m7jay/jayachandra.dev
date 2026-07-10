---
title: "Designing partner integrations for unreliable upstreams"
date: 2026-06-18
summary: "A practical reliability model for backend teams that depend on hundreds of external systems."
tags: ["distributed-systems", "integrations", "reliability"]
---

Partner integrations are distributed systems at their messiest: latency varies, schemas drift, and a timeout rarely tells you whether an operation happened. The answer is not more retries. It is an explicit operational contract.

## Start with a durable workflow state

Persist each business transition before handing work to the next stage. A worker should be able to restart and answer three questions: what is the current state, what was the last verified result, and what remains safe to retry?

## Separate partner failure from platform failure

Queue isolation keeps one slow or failing integration from consuming all worker capacity. Give every partner a bounded concurrency budget, a timeout policy, and observable error categories. This makes scaling deliberate rather than reactive.

## Treat correctness as a first-class latency requirement

For financial workflows, returning quickly with an unverified amount is not a success. Design idempotency keys, reconciliation paths, and audit records early. They are the tools that let an on-call engineer resolve ambiguity without replaying the whole world.

## Make the system legible

Dashboards should show backlog age, success rate, retry distribution, dependency latency, and business outcomes—not only CPU and error rate. The best integration platform is one an engineer can understand during an incident.
