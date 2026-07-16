---
title: "Hi there!"

date: 2026-07-10

showDate: false
showAuthor: false
showDateOnlyInArticle: false
showDateUpdated: false
showHeadingAnchors: false
showPagination: false
showReadingTime: false
showTableOfContents: false
showTaxonomies: false
showWordCount: false
showSummary: false
sharingLinks: false
showEdit: false
showViews: false
showLikes: false
layoutBackgroundHeaderSpace: false
layout: "simple"
---

<div class="resume-download"><a href="/resume/Jayachandra_M.pdf" download>Download Resume <span aria-hidden="true">↓</span></a></div>

## About me

Hi, I’m Jayachandra, an experienced backend-focused Staff Software Engineer.

Most recently at Jodo, I led the integrations team and worked on business-critical payment and invoice-processing workflows. I architected an invoice orchestration platform that processed **~50K–60K invoices per day in a ~4 hour window** and supported payment volumes exceeding **~₹100Cr daily**. It coordinated hundreds of partner integrations, accommodated high-latency external dependencies, and maintained strict correctness guarantees—because a missed or incorrect invoice could have direct financial consequences.

My work also includes event-driven webhook platforms, Kafka-based asynchronous processing, and secure partner-integration frameworks for real-time notifications and data exchange. Alongside hands-on technical work, I led architecture reviews, mentored engineers, supported backend hiring, and raised engineering standards across the team.

Earlier at Tally Solutions, I contributed to the TallyPrime 1.0 release and to developer-productivity work, including reducing build times for a large monorepo from about 45 minutes to 15 minutes.

After my most recent role, I took a short planned break before my next opportunity. I’m now looking for backend and platform-engineering roles where I can work on distributed systems, scalability, reliability, and high-impact platforms.

## Experience

### Jodo — Staff Software Engineer

**Oct 2024 – Jan 2026 · Bengaluru, India**<br>
_Python, Django, MySQL, Redis, Kafka, AWS, Docker, Terraform_

- Architected an [invoice orchestration platform](/case-studies/invoice-orchestration/) that retrieves real-time fee details from ~400 partner systems and presents invoices for automated debit workflows.
- Designed for **50K–60K invoices/day** and payment volumes above **₹100Cr/day**, while handling upstream response times from sub-second to roughly 60 seconds.
- Built asynchronous Kafka and Redis pipelines with persistent state, recovery paths, and correctness controls across fetch, synchronization, and presentation stages.
- Designed an event-driven webhook platform with queue isolation, prioritisation, retries, exponential backoff, and durable audit trails.
- Improved platform security and operability through RBAC, a central Go authentication proxy, Terraform automation, and Grafana, Prometheus, Sentry, and OpenTelemetry instrumentation.
- Led a three-engineer integrations team, including two direct reports; owned architecture reviews, hiring, standards, incidents, and production operations.

### Jodo — Senior Software Engineer

**Jul 2021 – Sep 2024 · Bengaluru, India**<br>
_Python, Django, MySQL, Redis, Celery_

- Owned REST APIs and partner integrations for payment and fee-collection workflows; push APIs consistently achieved approximately **500ms p99** latency.
- Reduced a critical metadata service latency by about **75%**, from roughly two seconds to 500ms.
- Delivered bidirectional integrations with major EdTech partners for onboarding, fee synchronisation, and automated payment workflows.
- Built reusable push/pull integration frameworks and contributed to the first Lambda-based webhook delivery system.

### Tally Solutions — Software Development Engineer II

**Jun 2020 – Jul 2021 · Bengaluru, India**<br>
_C/C++_

- Built installer and application-management systems for TallyPrime Release 1.0, supporting deployment and version management across enterprise products.
- Developed native Windows features using low-level system APIs, custom UI rendering, and environment/configuration management.

### Tally Solutions — Software Development Engineer I

**Jan 2019 – Jun 2020 · Bengaluru, India**<br>
_C/C++, Python, Shell_

- Improved Jenkins build and deployment pipelines for a large monorepo, reducing build time from about **45 minutes to 15 minutes**.

### Tally Solutions — Software Engineer Trainee

**Jul 2018 – Dec 2018 · Bengaluru, India**<br>
_Python, TDL_

- Developed SME financial reporting solutions and contributed to feature validation, defect discovery, and unit testing.

## Technology

Python · Django · MySQL · Redis · Kafka · AWS · Terraform · Docker · Celery · Prometheus · Grafana · OpenTelemetry · Sentry · C/C++ · Go

## Education

**Bachelor of Engineering, Computer Science & Engineering**<br>
[R. V. College of Engineering](https://www.rvce.edu.in/), Bangalore · 2014 – 2018
