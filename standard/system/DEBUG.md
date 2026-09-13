# Debugging Standard

This document defines the required operational process for debugging production,
integration, workflow, and regression failures. The goal is to move from symptom
to proven cause to verified fix without speculative patching.

## Objective

Debugging must produce four things:

- a reproducible or bounded failure condition;
- evidence that identifies the cause/effect mechanism;
- a fix that addresses the mechanism at the correct boundary;
- a regression guard that prevents the same failure from returning.

Do not start implementation from a guessed fix. Start from the observed failure,
state the mechanism, then change the smallest correct owner that can remove the
mechanism.

## Required Debug Flow

Follow these steps in order unless an incident commander explicitly prioritizes
service restoration before full root-cause analysis.

1. Capture the failure.
2. State the problem specification.
3. Establish the expected contract.
4. Reproduce or bound the condition.
5. Isolate the cause/effect mechanism.
6. Choose the owning boundary for the fix.
7. Implement the smallest correct fix.
8. Verify the fix and the non-regression cases.
9. Record the resolution and prevention guard.

If a step cannot be completed, record why and what residual risk remains.

## Problem Specification

Every debugging task must start with this shape:

```text
Conclusion: Consequence/Risk
- Issue: Up to 10 words.
- Why: Up to 20 words.
- How: Cause/Effect mechanism, up to 30 words.
- When: When it occurs and when it does not, up to 15 words.
- Where: Where it occurs and where it does not, up to 15 words.
- Fix: Proposed mechanism-level fix, up to 20 words.
```

Example:

```text
Conclusion: Outbound job events may fail after async refactor
- Issue: messaging_factory sync contract is unenforced.
- Why: Runner accepts coroutine results without validation.
- How: Coroutine becomes JobOutSender.service; later send_event() calls missing service methods and fails.
- When: Future async factory; not current sync factory.
- Where: Runner._build_sender; not Director construction.
- Fix: Reject awaitable factory results before sender construction.
```

The specification is not complete until `How` describes a concrete mechanism,
not just a symptom.

## Evidence Requirements

Gather evidence before changing behavior.

Required evidence:

- failing command, request, job id, trace id, or reproduction path;
- observed result and expected result;
- relevant logs, events, stack traces, or persisted records;
- input data or a minimized fixture when safe to store;
- changed code range or deployment version when known;
- scope statement: affected and not affected paths.

Evidence must be bounded and safe. Do not paste secrets, raw tokens,
authorization headers, broad provider payloads, or large user data into reports.
Use stable identifiers, sanitized error categories, hashes, and trimmed excerpts.

## Reproduction Rules

Prefer a deterministic reproduction. If the failure is intermittent, bound it.

Allowed reproduction levels:

- unit test that fails before the fix;
- integration test with mocked external dependencies;
- local command with fixed inputs;
- replay of a stored job/request fixture;
- bounded live reproduction with explicit environment and rollback plan.

For intermittent failures, record:

- trigger condition;
- observed frequency or sample count;
- timing, concurrency, queue, cache, or provider conditions;
- what similar paths do not fail.

Do not use live production experiments as the first debugging tool unless the
failure only exists in production and the experiment is explicitly bounded.

## Isolation Rules

The cause must be isolated at the boundary where the contract breaks.

Ask:

- What contract was expected at this boundary?
- What value, state, timing, or dependency violated it?
- Which component owns detecting that violation?
- Which component owns preventing it?
- Which downstream behavior only happened because the violation escaped?

Use temporary instrumentation only when existing evidence cannot answer those
questions. Temporary instrumentation must be safe, bounded, and removed or made
intentional before completion.

## Fix Selection

Fix the mechanism, not only the symptom.

Required fix criteria:

- preserves the architecture boundary in `doc/standards/system/DESIGN.md`;
- validates or normalizes data at the owning boundary;
- fails closed on contract-breaking state unless an explicit recovery contract
  exists;
- avoids broad compatibility shims unless a removal condition is recorded;
- does not move provider, transport, storage, or framework details into a higher
  layer;
- does not add speculative abstraction unrelated to the observed mechanism.

Patch work is not acceptable when it only suppresses the visible symptom while
leaving the cause/effect mechanism active.

## Verification

Every fix must include verification commands or a documented reason they could
not run.

Minimum verification:

```text
syntax/type/lint check relevant to the changed files
focused regression test that fails before the fix
focused success-path test when the fix changes accepted behavior
boundary/import/config guard test when the bug was a boundary leak
manual replay or live smoke test when the failure is integration-only
```

Verification must prove:

- the original failing condition no longer fails;
- the expected non-failing condition still works;
- invalid or unexpected input fails at the owning boundary;
- no unrelated workflow status, error, or retry path is polluted by the fix.

---

# Analysis - Cause / Effect

## Conclusion First

### Format


---
# Closure - Report


## Resolution Report

Close every debugging task with this report:

```text
Conclusion: Fixed consequence/risk
- Issue: What was wrong.
- Why: Why the old behavior allowed it.
- How: Mechanism that now prevents it.
- When: Conditions covered and excluded.
- Where: Files/functions changed.
- Fix: Short description of implemented fix.
- Verification: Commands/tests run, or why not run.
- Regression guard: Test, monitor, static guard, or runbook check added.
```

Example:

```text
Conclusion: Async factory refactor no longer causes delayed sender failure
- Issue: Coroutine factory results were accepted.
- Why: Runner trusted sync type hints only.
- How: Awaitable service results are rejected before JobOutSender construction.
- When: Future async factory; current sync factory unaffected.
- Where: src/app/scribe/agent/job.py:1157
- Fix: Added awaitable guard before sender construction.
- Verification: Focused job sender tests pass.
- Regression guard: Test rejects awaitable messaging_factory result.
```

## Regression Blocking

Every confirmed bug needs a prevention guard.

Accepted guards:

- unit or integration regression test;
- static guard for forbidden imports, flags, fields, or dependencies;
- schema/config validation case;
- replay fixture in a defect corpus;
- monitor or alert when the failure can only be observed operationally;
- runbook check for manual operational risks.

The guard must encode the cause/effect mechanism, not merely assert the final
surface symptom. Run relevant prevention tests on every related change.

## KT Differential Diagnosis

Use this when the cause is unclear.

| Question | IS | IS NOT |
| --- | --- | --- |
| WHAT | What object has the deviation? | What similar object does not? |
| WHERE | Where is the deviation observed? | Where is it absent? |
| WHEN | When did it first occur? | When does it not occur? |
| EXTENT | How many or how large? | What is the normal magnitude? |

Distinctions and recent changes are candidate causes. They are not conclusions
until evidence proves the mechanism.

## Operational Checklist

Before marking a debug task complete:

- Problem specification is filled in.
- Expected contract is named.
- Failure evidence is captured or bounded.
- Cause/effect mechanism is stated.
- Fix owner and boundary are identified.
- Regression guard exists.
- Verification commands are recorded.
- Residual risk is stated when verification is incomplete.


---
# Examples

Verify if this will help accelerate issue understanding and resolution.

## Python concurrency hotspots

in src/app/scribe/agent and src/app/scribe/frontend.

```text
1. Conclusion: Terminal events can be lost before client delivery.

- Issue: SQS delete before stream consumption.
- Why: Queue insertion is treated as delivery, but client may disconnect before consuming.
- How: Dispatcher enqueues event, then deletes SQS message; abandoned in-memory queue drops terminal state.
- When: Disconnect/shutdown after enqueue; not after stream consumes.
- Where: frontend/dispatcher.py, frontend/application.py.
- Fix: Delete after consumer processing, or persist terminal handoff.

2. Conclusion: Jobs can target a dead frontend queue.

- Issue: Lease loss races register/submit.
- Why: Health checks are separate from queue registration and job submission.
- How: Lease renewal can fail after registration or submission, stopping dispatcher while job still references that queue.
- When: Lease loss window; not stable leases.
- Where: frontend/director.py, frontend/main.py.
- Fix: Serialize lease state, register, submit with one lock/token.

3. Conclusion: Shutdown can accept orphaned in-flight jobs.

- Issue: Stop does not mark unhealthy first.
- Why: Requests can register while shutdown is stopping dispatcher.
- How: stop() releases resources after dispatcher stop, but register() may still see healthy state.
- When: Shutdown overlap; not normal steady state.
- Where: frontend/director.py.
- Fix: Set unhealthy before stopping; lock lifecycle operations.

4. Conclusion: Liveness abort can be delayed by queue backpressure.

- Issue: Abort signal uses blocking queue put.
- Why: Full per-job queues can prevent timely abort delivery.
- How: _liveness_loop awaits per_job_queue.put; progress backlog can block the control event.
- When: Full queue; not draining streams.
- Where: frontend/application.py.
- Fix: Use priority put, eviction, or nonblocking terminal control path.

5. Conclusion: Timed-out sync work can consume worker threads.

- Issue: to_thread cancellation does not stop work.
- Why: wait_for cancels awaiter, not the underlying synchronous call.
- How: Slow Lambda/CourseLoop calls continue in executor after timeout or frontend-dead cancellation.
- When: Slow sync calls; not fast timeout-respecting clients.
- Where: agent/tool/scribe.py, agent/tool/courseloop.py.
- Fix: Use async clients, hard request timeouts, and bounded semaphores.

6. Conclusion: Future parallel jobs may contend on shared dependencies.

- Issue: Pipeline dependencies are shared per process.
- Why: Current director is sequential, but parallel execution would reuse providers concurrently.
- How: Shared LLM, knowledge, Scribe, and pipeline deps would receive overlapping calls with mixed safety guarantees.
- When: Future parallel jobs; not current sequential loop.
- Where: agent/director.py, agent/scriber.py, agent/workflow/scribe_ast/nodes.py.
- Fix: Preserve sequential invariant or create per-job dependency instances.
```
