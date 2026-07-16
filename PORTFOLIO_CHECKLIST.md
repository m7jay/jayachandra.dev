# Portfolio upgrade checklist

This file tracks the staff-engineer portfolio work. Claims must be based on work that can be shared publicly; incident narratives, dashboards, and vendor evaluations require Jayachandra's confirmation before publication.

## P0 — Structural fixes

- [x] Add a separate Case Studies navigation item.
- [x] Rewrite the homepage positioning around verified scale and choose a headline.
  - Considered: “Staff engineer who shipped payment infrastructure processing ₹100Cr+/day across ~400 partner systems.”
  - Considered: “I build payment and integration systems that stay correct under real-world latency.”
  - Considered: “Backend and platform engineer for high-throughput financial workflows.”
  - Chosen: the first option, because it leads with concrete scope and scale.
- [x] Move “Setup Macbook Pro” out of Posts into Notes.
- [x] Add scope-of-influence signals: team leadership, architecture reviews, hiring, mentoring, standards, and incidents.

## P1 — Case studies

- [-] Invoice orchestration platform: published the architecture, known scale, latency range, and synchronous-path tradeoff. **Still needs: a shareable correctness near-miss.**
- [ ] Go auth proxy: prior tools evaluated, fit gaps, build-vs-buy tradeoff, and maintenance cost. **Needs: the actual tools considered and decision record.**
- [ ] Webhook/event platform: queue isolation, priority, retry/backoff, delivery guarantees, and anonymised outage postmortem. **Needs: shareable incident details and any publishable metrics.**

## P2 — Deepen technical posts

- [ ] Add a transaction-lifecycle state diagram and a resolved timeout ambiguity to “Designing partner integrations.” **Needs: a factual production example.**
- [ ] Add real retry/backlog thresholds and redacted dashboard evidence to “Webhooks are a product.” **Needs: approved metrics/screenshots.**
- [ ] Add “What we got wrong first” to “Faster feedback loops in a monorepo.” **Needs: the early approaches that failed or underperformed.**
- [ ] Expand relevant notes to 800–1,500 words and add a TL;DR/pull quote.

## P3 — Leadership and operating philosophy

- [ ] Write “Leading a 3-engineer integrations team.” **Needs: on-call, review, and incident-process details you want public.**
- [ ] Write a backend hiring/interviewing post. **Needs: your hiring rubric and examples that are safe to share.**
- [x] Write “What production-ready means to me” / “How I work.”
- [ ] Write an anonymised postmortem. **Needs: an incident to publish.**

## P4 — Polish and mechanics

- [ ] Add author-written TL;DRs/pull quotes to all long-form posts.
- [ ] Add Talks / Writing elsewhere when links are available.
- [x] Fix the `loading · loading` metadata issue by removing unresolved optional view/like counters from article metadata.
- [ ] Add approved peer/manager testimonials.
- [ ] Cross-link relevant résumé bullets to case studies.
