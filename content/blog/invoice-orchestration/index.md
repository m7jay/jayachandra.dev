---
title: "Invoice Presentation Orchestration"
date: 2026-07-12
summary: "How an asynchronous, stateful workflow supported 50K–60K invoices per day across roughly 400 partner systems."
tags: ["case-study", "distributed-systems", "payments", "reliability"]
showTableOfContents: true
layout: "simple"
---

> **TL;DR:**
> How an asynchronous, stateful workflow supported 50K–60K invoice processing per day across roughly 400 partner systems.

## The problem

A consumer sets up a recurring debit mandate (NACH or UPI) against a fee due amount shown at the time of setup.
That amount is not guaranteed to stay correct until the actual debit date, because the source of truth, the partner's own system (ERP/Inhouse management system) can change it:

- a cash payment done reduces the balance amount to be collected
- a previously applied discount gets reverted, increasing the amount due
- a new discount gets applied, decreases the due amount
- the due date itself gets moved to a future date

Debiting a stale amount has real financial consequences. A wrong debit can result in a bounce and carry penalty charges for the end customer. So the amount presented for debit has to reflect the partner's current state, not the state at mandate-setup time.

**Why integrations, not just manual updates?**

The alternative, partners bulk-upload dues once a day, creates a single point of failure and forces the partner's system into a freeze window until the upload is processed.
That doesn't scale across partners with different operational rhythms, and it puts an availability constraint on someone else's system, which we don't control.

Instead, we fetch the latest details directly from each partner system, reconcile the delta against what's currently recorded, and adjust before presenting the invoice for debit.

**Why the 4 hour window specifically?**

- All due invoices need to be processed in a window of approx 4 hours.
- Many partners run their own updates (cash collection entries, discount changes) in the first half of the day. Fetching earlier risks pulling pre-update, stale data.
- Once an invoice is presented, we owe the customer a debit intimation notice. We don't want that notice going out late in the evening.

---

## Scale and shape of the problem

- **Volume is spiky, not steady.** Daily invoice counts range from ~1,000 to ~60,000, depending on the day of month and partner systems.
- **Bulk counts land on different days per partner.** Most partners settle their large volume on their own day of the month, so the 1,000-60,000 swing is largely a function of whose bulk day it is. The rare day when several partners' bulk days coincide is when we provision extra capacity by hand.
- **Partner heterogeneity.** Each partner runs its own tech stack, with its own release cycles, a partner-side fix or change can take weeks to months to ship. Partner team competence varies widely - some are well-staffed engineering teams, others have outsourced, slow-to-respond vendors.
- **Latency variance per partner is extreme** — anywhere from a few milliseconds to 15 seconds per request, and this is _unpredictable_ in advance.
- **Shared infrastructure contention.** This pipeline doesn't run in isolation. At the same time, the same core services are handling inbound webhooks from partners, outbound webhook dispatch to partners, high-volume push API traffic, and internal bulk uploads/report downloads. A spike in any of these can degrade the pull-and-present flow's throughput, and vice versa.
- **No safe prefetch/cache.** We can't fetch once and cache for the day - that reintroduces the staleness problem we're solving for. Every invoice needs a fresh fetch within the window.
- **Dual reporting requirement.** After processing, two different reports are generated: an internal report for ops/follow-up, and an external report for subscribed clients.

The core tension: a hard 4-hour window, partner latency we don't control, volume that swings 60x day-to-day, and shared infra that's busy with other traffic - all while correctness failures have direct financial consequences for end users.

---

## High-level architecture

![Pull and Present - Invoice Processing Architecture](pull_and_present_invoice_processing_architecture.png)

### Stage 1 — Enqueue due invoices

{{< mermaid >}}
flowchart TD
subgraph Enqueue["Enqueue Due Invoices"]
A[Cron @ 2pm] --> B[Lambda trigger]
B --> C[Core Auto-Debit Service]
C --> D[Async task: fetch all due invoices]
D --> E[(Outbox table)]
D --> F[Enqueue to Kafka in batches]
F --> G[Slack alert: ingestion count]
end
{{< /mermaid >}}

A 2pm cron triggers a Lambda, which hits the core auto-debit service. The core service kicks off an async task (Celery) that fetches all invoices due, writes them to a table using the **transactional outbox pattern**, and enqueues them to Kafka in batches sized to the day's volume. A Slack alert confirms the trigger fired and reports the invoice count for the day - this gives an early signal if the current day's volume looks abnormal before any processing starts. We can also have an alert to get the due invoice counts for next few days, so that we can plan for any infra changes needed.

### Stage 2 — Process enqueued invoices

{{< mermaid >}}
flowchart TD
subgraph Process["Process enqueued invoices"]
H[Integration Service Worker] --> I{Invoice still in valid state?}
I -- no --> Skip[Skip / mark invalid]
I -- yes --> J[Fetch latest details from partner API]
J --> K[Diff: compute required update]
K --> L[Update core service with new amount/date]
L --> M{Still in valid state?}
M -- no --> Skip
M -- yes --> N[Present invoice for debit]
N --> O[Enqueue customer notification]
end
{{< /mermaid >}}

Integration service workers consume from Kafka and process invoices one at a time:

1. Check the invoice is still in a valid state (guards against races with other flows touching the same invoice).
2. Fetch the latest amount/due-date details from the partner system over HTTP.
3. Diff against current state to determine what update is needed.
4. Update the core services over internal HTTP calls.
5. Re-check valid state (the fetch + diff + update round-trip takes time, state may have changed).
6. Present the invoice for debit.
7. Enqueue a customer notification about the upcoming debit.

**Scope:** this workflow ends at presentation. Everything after it - whether the debit succeeded, whether it returned, and what happens next - belongs to the core subscription service, not the integration service.

### Stage 3 — Generate reports

{{< mermaid >}}
flowchart TD
subgraph Report["Generate reports (10pm cron)"]
P[Cron @ 10pm] --> Q[Lambda trigger]
Q --> R[Integration Service]
R --> S[Read outbox table]
S --> T[Generate + email internal report]
S --> U[Generate + email external client report]
end
{{< /mermaid >}}

A 10pm cron triggers report generation, reading from the same outbox table populated in Stage 1 - so the report reflects exactly what was ingested and processed, not a live re-query that could double-count or miss records.

Two reports run on this data: an internal one at 6pm, which partner POCs use to see which cases need manual attention, and the partner-facing one at 10pm. Retries update the same outbox row in place, so a case that fails at 6pm and succeeds by 9pm shows as a success in the 10pm report - the 6pm report is a work queue, not a verdict.

---

## Integration Service

- Integration Service acts like an orchestrator, handling fetching data from external partner systems, figuring out if an update is needed, changes to be done and making HTTP requests to internal services to update the new state and finally presenting the invoice for debit.

- Integrations workers are running 24/7, even though we only use them for a few hours every day. We didn't need to do worker auto scaling, when needed for a few days a year, we could monitor and increase the resource on specific days as per the load.

- Each message can take up to ~5s to be processed successfully and multiple calls over HTTP are done (external and internal calls) in processing a single message.

- For each partner, we have configurable timeouts in integration service.

- Partner auth differs by integration: some systems want a username and password, others a secret or API key in a header. Whatever the partner specifies is carried as per-partner config in the integration service.

- Partner systems whitelist our egress IPs, and the responses we fetch are installment and due-amount records without customer-linked identifiers - so PII stays in the partner system and the core service rather than passing through the integration layer.

- For each partner we have configurable number of retries in case of failures while processing the invoice.

- At integration service, we have a default way of processing of invoices, per partner we could have custom implementations using the template method pattern.

- Duplicate invoice processing is avoided by having a composite key in the invoice outbox table before ingestion and invoice present call is idempotent and would raise error if already presented.

- Duplicate customer notification is handled by notification service.

- Deployments need to be managed to avoid disruption during this workflow.

## Capacity planning: how many workers do we need?

{{< katex >}}

The 2pm–6pm window is fixed, so worker count is a direct function of per-invoice processing time and total volume for the day.

**Assumptions:**

- Processing window: 4 hours (14,400s)
- ~5% of due invoices are skipped before any partner call — subscription no longer valid, amount below the minimum, mandate not yet active. These cost milliseconds, so they count as ~0 worker-seconds.
- Of the invoices that do need a partner call, ~5% hit the 15s worst case; the rest average ~5s.

**Total processing time required, for 50,000 due invoices:**

$$
\text{Skipped} = 5\% \times 50{,}000 = 2{,}500 \text{ invoices} \approx 0 \text{ worker-seconds}
$$

$$
\text{Slow invoices} = 5\% \times 50{,}000 = 2{,}500 \text{ invoices} \times 15\text{s} = 37{,}500 \text{ worker-seconds}
$$

$$
\text{Average invoices} = 45{,}000 \times 5\text{s} = 225{,}000 \text{ worker-seconds}
$$

$$
\text{Total work} = 37{,}500 + 225{,}000 = 262{,}500 \text{ worker-seconds}
$$

**Workers required within the 4-hour window:**

$$
\text{Workers required} = \frac{262{,}500}{14{,}400} \approx 18.2 \Rightarrow \textbf{19 workers}
$$

That matches what we ran: about 20 workers on a 50K day, sitting a worker or two above the floor.

| Daily invoice volume | Minimum workers (mean 5.5s, 5% skipped) |
| -------------------- | --------------------------------------- |
| 1,000                | 1                                       |
| 10,000               | 4                                       |
| 25,000               | 10                                      |
| 50,000               | 19                                      |
| 60,000               | 22                                      |

**Sensitivity to the slow-invoice share:**

| Share of invoices at 15s | Workers required |
| ------------------------ | ---------------- |
| 0% (all at 5s)           | 17               |
| 5%                       | 19               |
| 10%                      | 20               |
| 20%                      | 24               |

**Caveat:** this model assumes work is evenly distributed across workers — any worker can pick up any invoice, with no clustering of slow invoices onto specific workers. That assumption carries the most risk here: a single degraded partner pins its 15s calls to one partition and serialises behind them. At the 19-worker floor the window is ~96% utilised, and queueing delay rises non-linearly as utilisation approaches 1, so a partner that goes slow in the last hour has no slack to absorb. On predicted heavy days we provision a few workers above the floor for exactly that reason.

---

## Observability

- Grafana dashboards to monitor service health / resource consumption.
  - For integration service, external partner call latency, MSK health

- Redash dashboards to query the outbox table to check for invoice state and counts.

## What happens when the window closes

Not every invoice finishes cleanly within the 4-hour window.

Two failure modes are handled operationally rather than automatically:

- **Messages stuck in an intermediate state** - an invoice that started processing but didn't reach a terminal state before the window closed. These require manual reprocessing.
- **Final state reached but not presented** - the invoice was validated and updated, but the present-for-debit step didn't complete due to a validation failure. These are also handled operationally rather than auto-retried.

Full automated recovery for every edge case at this volume wasn't worth the complexity, given how rarely these states occur in practice. Operational visibility makes manual intervention fast enough that it wasn't worth automating - so far.
