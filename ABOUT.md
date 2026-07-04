# Forge — Reliable Job Queue with Dead-Letter Recovery

> A durable, crash-safe background job engine built from scratch. No Celery, no BullMQ, no Sidekiq — the primitives, implemented and understood.

**Status:** Planned. Not started.
**Target start:** [fill in date]
**Target demo-ready:** [fill in date, ~2-3 weeks of focused work]

---

## Why this project exists

Every backend job posting lists "experience with background jobs / queues" as a throwaway line. Almost nobody applying has actually built one — they've configured Celery and called it a day. This project exists to be able to say, truthfully, in an interview: *I know why your queue drops jobs on crash, why your retries double-charge customers, and how to fix both.*

This is not a CRUD app with a queue bolted on. The queue **is** the product.

---

## Core Guarantees (non-negotiable, must all work)

1. **Durability** — jobs live in persistent storage (Postgres/SQLite/Redis-with-AOF, TBD), not memory. A worker crash or process kill must not lose a job.
2. **Priority + delayed execution** — jobs can be scheduled for future execution and processed in priority order.
3. **Exponential backoff retries** — configurable max attempts, jitter to avoid thundering herd.
4. **Dead Letter Queue** — jobs exceeding max retries move to DLQ, inspectable and replayable.
5. **Exactly-once processing (visibility timeout)** — no two workers ever process the same job concurrently. Lease/lock-based, with timeout-based reclaim if a worker dies mid-job.
6. **Idempotency** — retrying a job that partially completed must not duplicate side effects. Idempotency keys enforced at the job-handler level.
7. **Observability** — CLI or API to inspect queue depth, in-flight jobs, DLQ contents, per-job history.

If any of these seven are half-baked, the project isn't done. This list is the definition of done — not "it runs."

---

## Stretch Goals (only after core is airtight)

- Job dependencies (Job B blocks on Job A's completion — DAG-lite)
- Web dashboard for live queue monitoring (depth, throughput, failure rate over time)

---

## Design Decisions to Make Before Coding

*(fill these in when I sit down to build — forces me to think, not just start typing)*

- [ ] Storage backend: Postgres (SKIP LOCKED for concurrency-safe dequeue) vs Redis vs SQLite for local-first simplicity
- [ ] Language/runtime: Node/TS (matches my stack) — confirm
- [ ] Locking mechanism: row-level lock + visibility timeout vs Redis SETNX-style lease
- [ ] Backoff formula: base * 2^attempt + jitter — define caps
- [ ] Idempotency key scope: per-job UUID vs caller-supplied key
- [ ] How jobs are defined: typed handler registry vs generic payload + type string

---

## What Makes This "Reliable" Not "Toy"

The difference between a resume-filler project and a real one is what happens when things go wrong. This project is explicitly designed around failure modes, not the happy path:

- Kill a worker mid-job → job must reappear for another worker after visibility timeout, not vanish.
- Kill the whole process during a burst of enqueues → no job loss on restart.
- Simulate a flaky external API in a job handler → backoff + DLQ must behave correctly, and idempotency must prevent duplicate side effects on retry.
- Load test with concurrent workers → provable no double-processing (test this explicitly, don't just assume the lock works).

**A demo video/gif showing a worker crash and recovery, live, is the actual deliverable — not just clean code.**

---

## Testing Plan

- Unit tests per component (retry math, priority ordering, lock acquisition)
- Integration test: spin up N workers, enqueue M jobs, kill a worker mid-run, assert no job lost and no job double-processed
- Idempotency test: handler that fails after partial side-effect, assert retry doesn't duplicate it
- Load test: throughput under concurrent workers, note numbers for README (real metrics > vague claims)

---

## How This Gets Positioned on Resume / Interviews

- One line: *"Built a crash-safe distributed job queue in [stack] — durable persistence, exponential backoff retries, DLQ, and distributed locking to guarantee exactly-once processing under concurrent workers."*
- Interview story ready: visibility timeout mechanics, why idempotency matters, what happens when a worker dies mid-job.
- Pairs with backend/infra-leaning roles specifically — lead with this project for those, not VoltSense.

---

## Repo Structure (draft)

```
forge/
├── src/
│   ├── queue/          # enqueue, dequeue, priority, scheduling
│   ├── worker/          # job execution, locking, visibility timeout
│   ├── retry/            # backoff logic, DLQ transition
│   ├── storage/         # persistence layer (swappable backend)
│   └── cli/                # inspect queue depth, in-flight, DLQ
├── tests/
│   ├── unit/
│   └── integration/        # crash simulation, concurrency proof
├── docs/
│   └── ARCHITECTURE.md  # design decisions + diagrams, write as you go
└── README.md
```

---

## Notes to Future Self

- Don't start coding until the design decisions checklist above is filled in. Half of what makes this "reliable" is deciding the hard parts on paper first.
- Resist the urge to add the dashboard before the core seven guarantees are bulletproof and tested.
- Write ARCHITECTURE.md as you build, not after — capture *why*, not just *what*.
