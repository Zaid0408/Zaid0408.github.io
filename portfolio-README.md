# Zaid Hasan

**Backend Engineer · Bengaluru, India**

I build backend services — distributed systems, event-driven architectures, and the concurrency problems that come with them. Currently an SDE 1 at **Suki AI**, where I work on production Go/gRPC microservices for an ambient AI platform used by clinicians.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-zaidhasan0804-0A66C2?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/zaidhasan0804/)
[![GitHub](https://img.shields.io/badge/GitHub-Zaid0408-181717?style=flat&logo=github&logoColor=white)](https://github.com/Zaid0408)
[![LeetCode](https://img.shields.io/badge/LeetCode-zaidhasan07-FFA116?style=flat&logo=leetcode&logoColor=white)](https://leetcode.com/u/zaidhasan07/)
[![Email](https://img.shields.io/badge/Email-zaid.has07@gmail.com-EA4335?style=flat&logo=gmail&logoColor=white)](mailto:zaid.has07@gmail.com)

---

## About

I started out validating distributed systems and now I build them, which turned out to be an unusually useful order to do it in. Before I wrote my first production service I'd already spent a year and a half load-testing one across three geographic regions and writing integration suites against its Redis and MySQL layers — so I came into building with a fairly concrete picture of how these things fail.

Most of my work sits at the boundary where correctness gets hard: concurrent writes to the same row, at-least-once message delivery, state machines that must never enter an invalid transition. Outside of work I contribute to open source and build backend projects to go deeper than the day job allows.

---

## Experience

### Suki AI — Software Development Engineer 1
**Jul 2025 – Present · Bengaluru, India**
*Ambient AI for clinicians — regulated healthcare data, distributed microservice platform.*

**Production services**

- **`ms-feedback`** — *owned end to end.* The single platform service storing clinician feedback on AI-generated notes, summaries, QnA responses, and medical forms. Two gRPC endpoints over MySQL, multi-tenant with organization and user identity taken from platform metadata rather than the request body, and a submit-once rule that keeps the analytics dataset immutable. Shipped with unit tests and service-level gRPC integration tests.
- **`ms-appointments`** — *co-developed.* System of record for clinical appointments: visit CRUD, schedule queries by provider and date range, note and composition attachments to encounters, and Pub/Sub notifications on every appointment change.

**Platform reliability**

- Designed and ran **distributed load simulations across 3 GCP VM clusters** — K6 at **600 concurrent users per VM**, plus 200 CCU coordinated in parallel across all three regions. The **p90/p95 latency analysis** I delivered to engineering leadership was accepted and the configuration **shipped as Suki's production US-West cluster**.
- Migrated Suki Developer Platform API performance benchmarking to **K6**, establishing the high-concurrency baselines that define SLOs for public-facing endpoints.

**Platform delivery**

- Delivered full regression coverage for the **Interoperability initiative** (platform-wide alignment of SDP APIs, Web SDK, and iOS SDK) and **Form Filling** (an ambient AI feature integrated with partner APIs), unblocking both launches.
- Built integration suites for internal RPC services against **Redis with production-identical key patterns**, validating data consistency and state transitions across microservice boundaries.

---

## Open Source

**5 merged pull requests across 3 external repositories.**

### Pluggable GCS Storage Backend
**[`halcyon-past/quarantine` #64](https://github.com/halcyon-past/quarantine/pull/64)** · `MERGED` · Python

[`quarantine`](https://github.com/halcyon-past/quarantine) is a library that stops one bad record from killing a loop of 10,000 — failures are set aside with their input and traceback so the job finishes and you can retry just the failures later. I added a **Google Cloud Storage backend** so a fleet of workers can share one quarantine instead of each writing to its own local disk.

The interesting problem was **atomic ID claiming**. Two workers must never grab the same record number. S3 does this with the `If-None-Match: *` header; GCS has no equivalent. The analog is a **generation precondition** — upload a zero-byte `.claim` object with `if_generation_match=0`, meaning *"write this only if no live version exists."* The loser of the race gets a `PreconditionFailed` (412) and takes the next number. It's compare-and-swap at the object-storage layer.

Also handled **non-idempotent deletion** (GCS raises `NotFound` where S3 shrugs, so routine cleanup tolerates benign races), wrote the **commit protocol** so readers never observe a half-written record, set up CI against the **fake-gcs-server** emulator, and kept a **90% coverage gate across macOS, Windows, and Linux runners**.

### Redis Sliding-Window Rate Limiter
**[`GDGBabcockUniversity/gdg-babcock-hacktoberfest-2026` #55](https://github.com/GDGBabcockUniversity/gdg-babcock-hacktoberfest-2026/pull/55)** · `MERGED` · Python · FastAPI · Redis · Lua

A distributed **sliding-window rate limiter at 100 req/s** protecting the authentication and LLM-backed grading endpoints of an exam-grading platform.

Built on **Redis sorted sets** — one entry per request, timestamp as the score — so evicting everything older than the window gives a window that genuinely slides, avoiding the 2× burst a fixed-window counter allows at its boundary. The whole count-check-increment sequence runs inside a **Lua script executed atomically server-side**, because running it as separate round-trips lets two concurrent requests both read 99 and both increment, pushing 101 through a limit of 100.

Timestamps come from **Redis server time** rather than the application clock, so multiple pods share one authoritative clock, and `PEXPIRE` makes idle keys clean themselves up. Auth limits key on client IP, grading limits key on authenticated user ID, both returning **HTTP 429**. Shipped with **8 integration tests** run locally, inside the backend container, and in CI with a `redis:7-alpine` service container.

### Also merged

| PR | Repo | What |
|---|---|---|
| [#80](https://github.com/ashrafee-dev/scamshield-api/pull/80) | `ashrafee-dev/scamshield-api` | Fixed a Redis rate limiter creating keys with **no expiry** — a slow memory leak and a correctness bug, since a key without a TTL never resets its window |
| [#79](https://github.com/ashrafee-dev/scamshield-api/pull/79) | `ashrafee-dev/scamshield-api` | Added a `GET /health` endpoint to the FastAPI entry point |
| [#41](https://github.com/GDGBabcockUniversity/gdg-babcock-hacktoberfest-2026/pull/41) | `GDGBabcockUniversity/...` | GitHub Actions CI with lint checks — migrated the frontend to ESLint v9 flat config and fixed the Python test dependency |

---

## Projects

### Inventory Reservation System
**[github.com/Zaid0408/Inventory-reservation-Analytics](https://github.com/Zaid0408/Inventory-reservation-Analytics)**

`Java` `Spring Boot` `RabbitMQ` `Spring Data JPA` `PostgreSQL` `Docker Compose`

A two-service event-driven reservation system. The hard problem is **preventing oversell when multiple orders hit the same product at once**; the second problem is **releasing stock automatically when an order is never confirmed**.

- **Concurrency control** — pessimistic row-level locking (`SELECT … FOR UPDATE`) on the product row during reservation. Chosen over optimistic locking deliberately: contention on a popular product is high enough that version-check retries would thrash, and the critical section is short enough that holding a row lock is cheap.
- **State machine** — `RESERVED → CONFIRMED | CANCELLED | EXPIRED`, with separate *available* and *reserved* stock counters. Every transition is guarded, so confirm-after-expire or double-cancel fails loudly instead of corrupting the counters.
- **Expiry without Redis or cron** — reservations expire through a **RabbitMQ TTL delay queue with no consumer**. When the TTL fires, the message dead-letters with a different routing key into a queue that *does* have a consumer, which releases the stock. The dead-letter mechanism used as a scheduled-delivery primitive.
- **6 durable queues across 3 exchanges**, with a dead-letter exchange catching anything that still fails after 3 listener retries with exponential backoff.
- **Idempotency** — a unique constraint on `order_id` means a redelivered message can't double-reserve, which matters because RabbitMQ is at-least-once.

### Order Tracking & Analytics System
**[github.com/Zaid0408/OrderTrackingAnalytics](https://github.com/Zaid0408/OrderTrackingAnalytics)**

`Java` `Spring Boot` `RabbitMQ` `WebSocket (STOMP)` `Spring Data JPA`

An event-driven notification system built around a **fanout exchange** — the complement to the direct-exchange routing in the inventory project.

- One order event is copied to **three independent queues** (email, SMS, analytics) with three independent consumers, so a failure in email delivery leaves SMS and analytics completely unaffected. Without the broker in between, one failing downstream call fails the whole request.
- **Real-time dashboard over WebSocket** — the analytics consumer pushes updated metrics to a STOMP topic, so clients see live counts with no polling.
- Thread-safe counters via `AtomicLong`, since three consumers increment concurrently, plus a persisted audit trail of every notification.

### Task Management System — Kanban Board
**[github.com/Zaid0408/task-manager-app](https://github.com/Zaid0408/task-manager-app)**

`Python` `FastAPI` `PostgreSQL` `SQLAlchemy` `Pydantic` `React` `Docker` `JWT`

A full-stack Kanban application. JWT authentication with password hashing protecting every endpoint, a relational schema for users, projects, tasks, and comments with CRUD through SQLAlchemy models and Pydantic schemas, and the whole stack containerized into isolated backend, frontend, and database services on a custom bridge network.

---

## Skills

| | |
|---|---|
| **Languages** | Java · Go · Python · SQL |
| **Frameworks & APIs** | Spring Boot · FastAPI · REST · gRPC · WebSocket |
| **Databases** | PostgreSQL · MySQL · BigQuery |
| **Caching & Messaging** | Redis · RabbitMQ · Google Pub/Sub |
| **Infrastructure** | GCP · Docker · Docker Compose · CI/CD (GitHub Actions, Buildkite) |
| **Performance** | K6 · Load testing · Latency profiling (p90/p95) · SLO definition |

---

## Also on my GitHub

- **[LLD-Notes](https://github.com/Zaid0408/LLD-Notes)** — working through low-level design: SOLID, and the creational, structural, and behavioural patterns, each with a worked example in Java.
- **[Java-DSA-Practice](https://github.com/Zaid0408/Java-DSA-Practice)** — data structures and algorithms in Java, organized by topic, with written notes on dynamic programming, sliding window, and string patterns.
- **[LeetCode](https://leetcode.com/u/zaidhasan07/)** — 280+ problems solved. Strongest in arrays, dynamic programming, trees, and graph traversal.

---

## Education

**Bachelor of Engineering, Information Science** · 2021 – 2025
BMS College of Engineering, Bengaluru · **CGPA 8.3 / 10**

---

## Get in touch

I'm open to backend engineering roles.

- **Email** — [zaid.has07@gmail.com](mailto:zaid.has07@gmail.com)
- **LinkedIn** — [linkedin.com/in/zaidhasan0804](https://www.linkedin.com/in/zaidhasan0804/)
- **GitHub** — [github.com/Zaid0408](https://github.com/Zaid0408)
- **LeetCode** — [leetcode.com/u/zaidhasan07](https://leetcode.com/u/zaidhasan07/)
