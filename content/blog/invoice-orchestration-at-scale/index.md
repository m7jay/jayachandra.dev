---
title: "Scaling the Invoice Pipeline 20x"
date: 2026-07-12
summary: "What changes in the invoice pull-and-present design at 100K and 1M invoices a day — concurrency, partitions, async workers, partner rate limits, and the notification bill."
tags: ["case-study", "distributed-systems", "payments", "reliability", "scaling"]
showTableOfContents: true
layout: "simple"
---

> **TL;DR:**
> The as-built design clears 50K–60K invoices a day in a four-hour window. At 100K, partitions and thread-per-invoice workers stop being enough. At 1M the partner rate limit and the SMS bill bind before capacity does.

This is the design-notes companion to [Invoice Presentation Orchestration](/blog/invoice-orchestration/). That post describes what runs today; this one asks what would have to change at 100K and 1M — and what would stay exactly as it is.

The current shape was sized for 50-60K and works because three things were true: 20 partitions were enough consumer parallelism, one worker thread could hold one in-flight invoice, and every partner would accept our request rate inside a common 4-hour window. At 100K the first two bend and the third still holds. At 1M all three break, and capacity stops being the interesting constraint.

## Sizing: same model, two rungs up

Running the same model forward:

| | 50K (today) | 100K | 1M |
| --- | --- | --- | --- |
| Worker-seconds required | 262,500 | 525,000 | 5,250,000 |
| Concurrency required in 4h | 18.2 → 19 | 36.5 → 37 | 364.6 → 365 |
| Sustained message rate | 3.5/s | 6.9/s | 69.4/s |
| Partitions at one in-flight per partition | 20 (current) | 37 | 365 |

## Ingestion: shard the sweep, stream the rows

At 100K, one job is still fine, but it should stream the due-list through a server-side cursor and bulk-insert in 5-10K chunks instead of materialising the list in the application. Publishing should be a relay over the outbox rows, not a second write from the same code path. *Why:* the 2pm sweep is one large write burst, and a single 100K-row transaction holds locks and WAL for its whole duration while a crash at 90% throws away the run.

At 1M, shard the sweep by partner cohort — one ingestion job per shard, each with its own `request_id` of `date + shard`. *Why:* a million rows in one transaction is minutes of WAL, and retrying a failed run means re-deriving a million invoices. Per-shard `request_id`s make progress partial and retries cheap, and the outbox key `(request_id, sequence_number)` keeps working unchanged — `sequence_number` just restarts per shard.

## Queue: keep it keyless, add partitions carefully

Keep Kafka, keep no partition key, raise the count: roughly 40-60 partitions at 100K, 120-200 at 1M.

*Why keyless is still right at 20x:* you do not need ordering — dedupe is on the row key and present is idempotent by contract — and a partner key would be actively harmful. With 400 partners and 60x volume skew, a partner carrying 25% of a 1M day puts 250K messages on a single partition and serialises the whole window behind itself. Default partitioning spreads load evenly; partner isolation belongs in the consumer as a per-partner semaphore, not in partition assignment.

*Why not 365 partitions:* partition count is only a parallelism ceiling if you assume one in-flight message per partition. Once consumers multiplex, 120 partitions each feeding 3-5 concurrent requests covers 365 in-flight without paying for rebalance time, file handles and leader elections.

## Workers: stop paying for a thread per in-flight request

This is the change that matters, because the arithmetic says the work is not compute.

At 100K: an async worker (asyncio plus aiohttp or equivalent) holding 20-25 concurrent partner calls per task, so two or three tasks cover the window. At 1M: fifteen tasks at 25 concurrent, or eight at 50.

*Why async:* a 5s invoice spends almost all of that waiting on someone else's HTTP endpoint. At ~30ms of CPU per invoice, a million invoices is roughly 2 vCPU sustained across the entire four-hour window. Buying 365 request slots by paying for 365 containers is paying twenty times over for the same idle time.

*Why the fleet should be elastic rather than 24/7:* the same fleet provisioned continuously costs about $12,900/month on-demand at 1M (92 `m6i.xlarge` hosts at four worker containers each); scheduled to exist only for the window, on Spot, about $800; async on Fargate Spot, about $60-150 with real headroom. The 4-hour window is a gift for cost — a fixed daily peak is the easiest shape to schedule against.

*Why scale on projected finish time and not CPU:* CPU sits near idle by design, so a CPU alarm never fires before the deadline. The signal that matters is the queue lag expressed in invoice-seconds and the projected finish time; if ETA exceeds the deadline minus a safety margin, add capacity, and if it lands early, scale in.

*Why Spot is safe here:* interruptions redeliver messages, and republishing is harmless because present returns "already presented". At 100K, Lambda is a reasonable alternative — the per-invoice work fits well inside a function invocation — but at 1M the sustained 70/s rate, VPC/NAT egress and reserved concurrency make long-lived Spot tasks cheaper and simpler.

## Partners: the window has to become per-partner

At 100K the global 2pm-6pm window still works. Enforce it with a per-partner token bucket, a per-partner semaphore (bulkhead) so one slow integration cannot consume the shared pool, and a breaker that stops retrying a partner whose error rate has collapsed — three retries against a 15s timeout turns one 5.5s invoice into ~50 worker-seconds, and the window closes regardless.

At 1M the constraint moves to the partner side:

| Partner share of a 1M day | Invoices | Required rate to finish in 4h | Duration at a 2 req/s cap |
| --- | --- | --- | --- |
| 5% | 50,000 | 3.5/s | 6.9h |
| 10% | 100,000 | 6.9/s | 13.9h |
| 25% | 250,000 | 17.4/s | 34.7h |

No amount of our capacity fixes a partner that will not accept the rate. So at 1M the single global window has to be replaced with per-partner scheduling: fetch after that partner's own cutoff, present before its intimation deadline, and stagger cohorts across the day. *Why this is the correct decision rather than adding workers:* the equation is bounded by the smallest of our concurrency and the partner's rate limit, and at 1M the partner is usually smaller. Staggering also flattens our peak, which is what makes the scheduled-Spot numbers above achievable — a smoother curve needs fewer purchased slots.

The second half of that decision is deadline-aware admission: if a cohort's projected finish exceeds its deadline, stop admitting it and park the remainder with a reason, rather than half-processing everything and finishing nothing on time. A parked invoice that a POC resolves tomorrow is a better outcome than a debit presented after the intimation window.

## Presentation and storage: batch the calls, partition the rows

At 100K the presentation path is roughly 24 internal calls/second — 3-4 per invoice — which a handful of replicas absorb. At 1M it is ~240/s plus a million row updates, and that is the point where per-invoice RPC becomes the bill of materials: a bulk present endpoint that takes 50 per call and returns per-item results cuts both the call count and the transaction count by an order of magnitude. *Why correctness survives batching:* idempotency is defined per item, so a partially failed batch retries item by item and duplicates still return success.

Storage stays in Postgres with monthly range partitions. At 1M/day that is 365M rows a year, roughly 165GB — size is not the problem; write rate and vacuum are. Partitioning keeps the live index small, and closing a month becomes a detach plus a Parquet export to S3 rather than a delete. *Why not DynamoDB:* the access pattern is date-range scans for the reports plus point updates by row key, and Postgres serves both while keeping the Redash queries ops relies on.

## Notifications: the cost centre that dwarfs the infrastructure

At 50K invoices a day, an SMS intimation at roughly ₹0.20 is about $120/day. At 1M it is about **$2,400/day — around $72,000 a month**, against a compute bill measured in tens or hundreds of dollars. Notifications are the only line item here that grows linearly with volume and has no Spot market.

*So the design decision at 1M is channel selection, not throughput:* push or email wherever the mandate type permits, digest multiple upcoming debits where the regulation allows one message, and reserve SMS for the statutory intimation. That is a product and compliance decision shaped by the same volume curve as the workers, and it deserves to be made deliberately rather than inherited.

## What stays exactly as it is

The outbox with `(request_id, sequence_number)`, idempotent present returning success for a duplicate, per-partner configuration via the template method, and the decision to hand an ambiguous case to a POC rather than guess at an amount. What changes with scale is the plumbing around those decisions, not the decisions themselves — usually a sign they were right.

*Cost figures above are modelled, not billed: us-east-1 on-demand `m6i.xlarge` at $0.192/h, Spot at roughly 30% of on-demand, Fargate Spot at $0.0127/vCPU-hour, 30 days, four to five hours of fleet life per day. Treat them as one significant figure.*

---

*Back to [Invoice Presentation Orchestration](/blog/invoice-orchestration/) for the as-built system and the capacity model these figures extend.*
