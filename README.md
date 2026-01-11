# Temporal-Learning

## 🔹 Week 1 — Temporal Mental Model (MOST IMPORTANT)
Learn

  - Why Temporal exists
  - Workflow history & replay
  - Determinism rules
  - Event sourcing model
  - Workflow code ≠ normal Go code

### Why workflows cannot:

  - Do I/O
  - Use time.Now()
  - Use goroutines freely

### Hands-on Labs
Write a workflow that:
  - Calls an activity
  - Sleeps
  - Retries
  - Kill the worker mid-execution
  - Restart worker → observe replay
  - Break It
  - Add rand.Int() inside workflow
  - Deploy → observe nondeterminism failure

You must be able to explain:

**“Why replay is necessary and how Temporal guarantees durability.”**

## 🔹 Week 2 — Activities, Timeouts & Retries
Learn

Activity lifecycle
Timeout types:
    - ScheduleToStart
    - StartToClose
    - Heartbeat
    - Retry policies
    - Non-retryable errors

Hands-on Labs

Activity that:

  - Sleeps longer than timeout
  - Fails transiently
  - Heartbeats progress
  - Break It
  - Make activity do a side effect twice
  - Observe duplicate execution

Key Insight

Temporal retries assume idempotency — it does NOT guarantee it.

## 🔹 Week 3 — Workflow Evolution & Versioning
Learn

  - Why workflow code is immutable
  - workflow.GetVersion
  - Backward compatibility
  - Patching live workflows

Hands-on Labs

  - Deploy v1 workflow
  - Start long-running workflows
  - Change logic
  - Deploy v2 without versioning → observe failure
  - Fix using GetVersion

You must explain:

**“Why versioning must be added before code changes.”**

## 🔹 Week 4 — Signals, Queries & Updates
Learn

  - Signal delivery semantics
  - Query consistency model
  - Workflow Updates (newer & safer)
  - Human-in-the-loop workflows

Hands-on Labs

  - Pause/resume workflow using signals
  - Approvals via updates
  - Query workflow state mid-flight
  - Break It
  - Send multiple signals quickly
  - Observe ordering guarantees

Key Concept

**Temporal workflows are state machines, not functions.**

## 🔹 Week 5 — Scaling, Workers & Performance
Learn

  - Task queues
  - Pollers
  - Sticky execution
  - Worker concurrency
  - Rate limiting strategies

Hands-on Labs

  - Run multiple workers
  - Throttle activities
  - Introduce 429 errors
  - Tune concurrency

You must explain:

“Why retry + no rate limit = retry storm.”

This is production-grade knowledge.

## 🔹 Week 6 — Failure Modes & Recovery
Learn

  - Worker crash scenarios
  - Activity timeout vs success ambiguity
  - Heartbeat recovery
  - Workflow cancellation
  - Child workflow failures

Hands-on Labs

Kill worker during activity

Force activity timeout after side effect

Cancel workflow mid-flight

Design Exercise

How do you make activities safe when side effects already happened?

🔹 Week 7 — Advanced Patterns (Staff Level)
Learn

Child workflows vs activities

Sagas & compensation

Long-running orchestration

Fan-out / fan-in

Continue-As-New

Hands-on Labs

Build saga workflow with rollback

Implement fan-out parallel steps

Use Continue-As-New to limit history

🔹 Week 8 — Integration & Real-World Design
Build

Temporal + Kubernetes controller

CRD → Workflow mapping

Infra orchestration use case

Write Design Docs

Failure handling

Retry strategy

Versioning plan

Observability

🧪 Mandatory Failure Lab Checklist ✅

If you don’t do these, you’re not “deep” yet:

 Break determinism

 Kill workers repeatedly

 Trigger retry storms

 Cause duplicate side effects

 Patch live workflows

 Cancel workflows mid-execution

🧠 Interview Readiness Check

You’re ready when you can answer:

Why Temporal over queues?

Why workflows must be deterministic?

How do you handle API rate limits?

How do you evolve long-running workflows safely?

How does Temporal compare to Kubernetes controllers?

