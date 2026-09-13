# Plan: Chaos Monkey Simulation

Project names, paths, and identifiers in the examples are anonymised.

## Purpose

Document what happens when each external dependency is unavailable or
misbehaving during a Rules job. For each scenario: the exception raised, the
log messages emitted, the audit events written, and the user-facing outcome.

Use this as a testing checklist and a debugging reference when diagnosing
unexpected job states.

## System Map

```
Client HTTP
    |
    v
Frontend (application.py / Application)
    |-- DynamoDB: rules_job, rules_job_log
    |-- SQS job-in: submits job payload to Y
    |-- SQS job-out: receives events from Y
    |
    v
Worker (job.py / Runner)
    |-- DynamoDB: rules_job, rules_job_log
    |-- SQS job-out: sends JOB_ACCEPTED / JOB_WAITING / JOB_PROGRESS / JOB_COMPLETED
    |
    v
Workflow (nodes.py)
    |-- LLM provider
    |-- Knowledge service
    |-- Rules parse endpoint
    |-- Requirements service
```

## Dependency Scenarios

---

### 1. Rules Parse Endpoint Down

**What is it:** The HTTP endpoint that validates Rules DSL syntax.
Called from `parse_rules_block` node via `execute_rules_parse(...)`.

**What happens:**

`_await_with_liveness(...)` wraps the call. If the endpoint is down, the HTTP
client raises a connection or timeout exception. Because `parse_rules_block`
does not catch exceptions from the parse call itself, the exception propagates
to the workflow executor.

**Exception raised:**

Connection refused, timeout, or HTTP error from the Rules client. The exact
type depends on the HTTP client implementation (typically `aiohttp.ClientError`
or a wrapped provider exception).

**Audit event written:**

`PARSE_CALL` is only written on a successful or failed-response call. If the
endpoint is unreachable before a response is returned, no `PARSE_CALL` audit
row is written for that attempt.

If the error escapes `parse_rules_block`, the workflow aborts. `Runner._execute`
catches the exception:

```
log.error("job setup or pipeline raised unhandled exception", ...)
extra: {"action": "pipeline_error", "job_id": ...}
```

Best-effort ABORTED write follows.

**Log messages:**

| Source | Level | Message / action key |
|---|---|---|
| `nodes.py` | WARNING | `send_waiting failed; continuing workflow phase` / `send_waiting_failed` |
| `job.py` | ERROR | `job setup or pipeline raised unhandled exception` / `pipeline_error` |
| `job.py` | WARNING | `best-effort ABORTED write failed` / `abort_failed` (if DDB also fails) |

**User-facing SSE event:**

`JOB_ABORTED` with `status: "ABORTED"`. The abort payload includes the job
record status from DDB and any log events fetched by `_abort_payload(...)`.

**Job final state in DDB:** `ABORTED`

**Normal parse failure (endpoint returns ok=False) is different:**

If the parse endpoint responds with `ok: false`, that is not a crash. The
workflow routes to `repair_rules_block`. After `_MAX_REPAIR_ATTEMPTS` (3)
failed repairs, `semantic_review` sets `requires_human_review: true`. The job
completes with `JOB_COMPLETED` but the result is flagged for human review.

Audit events written on each attempt:

```
PARSE_CALL: ok, status_code, message, raw_response, rules_text, parser_rules_text, labeltag_rewrite
```

---

### 2. LLM Provider Down

**What is it:** The external LLM API (e.g., Anthropic/Bedrock).
Called from `_call_llm_tool_loop(...)` via `llm.complete(...)` in all
generation phases: `extract_rules_ast`, `plan_rules_block`, `draft_rules_block`,
`repair_rules_block`, `semantic_review`.

**What happens:**

The LLM provider raises. The call is wrapped in `_await_with_liveness(...)` for
cancellation safety. An exception from `llm.complete(...)` is caught in
`_call_llm_tool_loop(...)`:

```python
# nodes.py
except Exception as exc:
    await context.log_step(
        "LLM_RESPONSE",
        duration_ms=elapsed_ms(t0),
        detail={"state": state_label, "model": ..., "error": str(exc)},
    )
    raise
```

The exception is re-raised and propagates to the workflow executor, which
aborts the job.

**Retry behavior:**

`LLMServiceConfig` controls `max_retries` and `retry_base_delay`. If retries
are configured, the LLM service layer retries before raising. The
`JOB_WAITING` event sent before the call includes an `expected_seconds`
estimate that accounts for retry backoff. If all retries are exhausted, the
exception escapes.

**Audit events written:**

`LLM_RESPONSE` with `error: str(exc)`. The `duration_ms` reflects elapsed time
through all retries.

If the failure is in `_call_llm_tool_loop` before a secondary fallback model
is tried, the fallback is attempted first. If both fail, `LLM_RESPONSE` is written
for the last attempt.

**Log messages:**

| Source | Level | Message / action key |
|---|---|---|
| `nodes.py` | WARNING | `send_waiting failed; continuing workflow phase` / `send_waiting_failed` (if JOB_WAITING send also failed) |
| `job.py` | ERROR | `job setup or pipeline raised unhandled exception` / `pipeline_error` |

**User-facing SSE event:**

`JOB_ABORTED` with `status: "ABORTED"`.

**Job final state in DDB:** `ABORTED`

---

### 3. Knowledge Service Down

**What is it:** The knowledge retrieval service backing `KNOWLEDGE_CALL` events.
Called from `retrieve_context` node via `_retrieve_knowledge_by_hint(...)`,
`_retrieve_ast_targeted_knowledge(...)`, and `_retrieve_indexed_example_matches(...)`.

**What happens:**

Knowledge failures are caught and **do not abort the job**. The workflow
continues in degraded mode.

```python
# nodes.py
except Exception as exc:
    await context.log_step(
        "KNOWLEDGE_CALL",
        duration_ms=elapsed_ms(t0),
        detail=_knowledge_exchange_log_detail(
            source_hint=source_hint, ..., error=str(exc)
        ),
    )
    return source_hint, [], [f"{source_hint} knowledge search failed: {exc}"]
```

The retrieval returns an empty evidence list. The error string is collected
into `retrieval_errors`. Workflow state is updated with
`retrieval_degraded: True`.

**Audit events written:**

`KNOWLEDGE_CALL` with `error: str(exc)` and empty `matches_count`.

If all retrieval attempts fail, a `KNOWLEDGE_DEGRADED` audit event is written
summarizing the degradation.

**Oversize query retry:**

If `KnowledgeQueryOversizeError` is raised, the query is reduced to 80% and
retried up to `_KNOWLEDGE_OVERSIZE_RETRY_LIMIT` (2) times. Each retry writes
a `KNOWLEDGE_QUERY_OVERSIZE_RETRY` audit event. If retries are exhausted, the
error is treated as a normal knowledge failure (degraded mode).

**Log messages:**

| Source | Level | Message / action key |
|---|---|---|
| `nodes.py` | WARNING | `rules block example retrieval failed; continuing` (for indexed example failures) |
| `nodes.py` | WARNING | `indexed_example_retrieval_degraded` / `indexed_example_retrieval_degraded` |
| `nodes.py` | WARNING | `knowledge_search_skipped_empty_query` / `knowledge_search_skipped_empty_query` (if query is blank) |

**User-facing SSE event:**

`JOB_COMPLETED` with `requires_human_review: true` if `semantic_review` flags
the degraded result. The job completes but its output should be reviewed.

If the evidence gap caused critical workflow failures (e.g., AST extraction
fails), the workflow aborts and sends `JOB_ABORTED`.

**Job final state in DDB:** `PROCESSED` (flagged for human review) or `ABORTED`

---

### 4. Requirements Service Down

**What is it:** The service that resolves course requirements by `course_sys_id`.
Called from `collect_input` node via `_fetch_requirements_service(...)`.

**What happens:**

`_fetch_requirements_service(...)` does not catch exceptions. If the RequirementsService client
raises, the exception propagates out of `collect_input`. The workflow aborts at
the first node.

**Partial failure (service responds with error field):**

If RequirementsService returns a response with `raw_course["error"]` set, the call
succeeds at the HTTP level. The audit event records `degraded: true` and the
`error` string. The workflow continues with whatever requirements RequirementsService
returned.

**Audit events written:**

`REQUIREMENTS_SERVICE_CALL` with `degraded: true` and `error: str(...)` for partial
failures. Nothing written if the exception is raised before the call returns.

**Log messages:**

| Source | Level | Message / action key |
|---|---|---|
| `job.py` | ERROR | `job setup or pipeline raised unhandled exception` / `pipeline_error` (if RequirementsService raises) |

**User-facing SSE event:**

- If RequirementsService raises: `JOB_ABORTED` with `status: "ABORTED"`.
- If RequirementsService returns partial data: job continues normally; `JOB_COMPLETED`
  is the likely outcome unless downstream failures occur.

**Job final state in DDB:** `ABORTED` (unavailable) or `PROCESSED` (partial data)

---

### 5. DynamoDB Down

DynamoDB is used by both the frontend and the worker for different operations.
Failures have different consequences depending on when they occur.

#### 5a. DynamoDB Down at Job Submission (Frontend)

**What happens:**

`Application.submit_job(...)` calls `job_service.create_job(...)`. If DDB
raises `JobServiceError` or `BotoCoreError`:

```python
except (JobServiceError, botocore.exceptions.BotoCoreError) as exc:
    raise JobQueueUnavailableError("Job database is temporarily unavailable...") from exc
```

**Log messages:** None at this layer (exception propagated to HTTP handler).

**User-facing HTTP response:** HTTP 503. The client should retry.

**Job final state in DDB:** Not created.

#### 5b. DynamoDB Down at Job Claim (Worker)

**What happens:**

`Runner._claim_job(...)` calls `job_service.claim_job(...)`.

If `JobClaimConflictError` or `JobNotFoundError` is raised (expected races):

```
log.info("job not claimable; deleting SQS message", ...)
extra: {"action": "claim_skip", "job_id": ..., "reason": ...}
```

SQS message is deleted. Worker moves to next message.

If a transient DDB error is raised (other than conflict/not-found): exception
propagates. SQS message is **not** deleted and will redeliver.

**User-facing outcome:** Job remains in `SUBMITTED` state in DDB and will be
retried when the SQS message redelivers.

#### 5c. DynamoDB Down Mid-Execution (Worker)

**Audit log writes fail:**

```
log.warning("log_step failed; continuing", ...)
extra: {"action": "log_step_failed", "job_id": ...}
```

Pipeline continues. Audit trail is incomplete but the job is not aborted.

**Liveness monitor poll fails:**

```
log.warning("liveness poll failed; will retry next interval", ...)
extra: {"action": "liveness_poll_failed", "job_id": ...}
```

Retries on next interval. If the record is missing for
`liveness_max_missing_records` consecutive polls, the frontend is declared
dead and the job aborts (see section 7).

**PROCESSED transition fails (state conflict):**

```
log.warning("PROCESSED write rejected; frontend may have already aborted", ...)
extra: {"action": "complete_conflict", "job_id": ...}
```

Silently returns. The frontend's `ABORTED` state wins.

**JOB_COMPLETED send succeeds but PROCESSED write fails:**

Best-effort abort attempted. The job may be left in `PROCESSING` state if
both the PROCESSED write and the abort write fail.

#### 5d. DynamoDB Down at Frontend Stream

**Heartbeat writes fail:**

```
log.warning("heartbeat failed", ...)  # from _best_effort()
```

Swallowed. `updated_at` not refreshed in DDB. If the backend is polling
`updated_at` for frontend liveness, it may eventually declare the frontend dead.

**Audit log writes fail:**

```
log.warning("log_event failed", ...)
extra: {"job_id": ..., "event": ...}
```

Swallowed. Frontend audit trail incomplete.

**Job record lookup fails during stream:**

```
log.warning("get_job failed in _handle_abort", ...)
```

Frontend synthesizes a best-effort `JOB_ABORTED` using whatever state it has.

**User-facing SSE event:** `JOB_ABORTED` with `status: "ABORTED"` if abort
path is reached, or stream stalls until the SSE abort deadline elapses.

---

### 6. SQS Job-In Queue Down (Submission)

**What is it:** The SQS queue the frontend writes to when submitting a job to
the worker.

**What happens:**

After `create_job(...)` writes to DDB, the frontend sends the SQS message:

```python
except SqsQueueNotFoundError as exc:
    await self._abort_job(job_id)
    raise JobQueueConfigError(...) from exc  # HTTP 500

except botocore.exceptions.BotoCoreError as exc:
    await self._abort_job(job_id)
    raise JobQueueUnavailableError(...) from exc  # HTTP 503
```

The job record in DDB is aborted before the HTTP error is returned.

**Log messages:**

None at this layer (exception propagated to HTTP handler). Abort write failure
would log:

```
log.warning("abort write failed", ...)
```

**User-facing HTTP response:**
- Queue not found (permanent misconfiguration): HTTP 500.
- Transient SQS error: HTTP 503. Client should retry.

**Job final state in DDB:** `ABORTED` (best-effort; may remain `SUBMITTED` if
abort write also fails).

---

### 7. SQS Job-Out Queue Down (Worker Events)

**What is it:** The SQS queue the worker sends events on (JOB_ACCEPTED,
JOB_WAITING, JOB_PROGRESS, JOB_COMPLETED). The frontend polls this queue.

All sends are best-effort. A failure does not abort the job.

#### 7a. JOB_ACCEPTED Send Fails

```
log.warning("JOB_ACCEPTED send failed; continuing", ...)
extra: {"action": "send_accepted_failed", "job_id": ...}
```

Pipeline continues. Frontend does not learn the backend accepted the job
until a subsequent event arrives (JOB_WAITING or JOB_PROGRESS).

#### 7b. JOB_WAITING Send Fails

```
log.warning("send_waiting failed; continuing workflow phase", ...)
extra: {"action": "send_waiting_failed", "state": ...}
```

Audit event `WAITING_SEND_FAILED` is also written to `rules_job_log`.

Frontend `abort_at` deadline is not extended. If the LLM or parse call takes
longer than the existing `abort_at`, the frontend may time out and abort the
job while the backend is still working.

#### 7c. JOB_PROGRESS Send Fails

```
log.warning("send_progress failed; continuing", ...)
extra: {"action": "send_progress_failed", "job_id": ...}
```

Pipeline continues. Frontend misses the progress update but will receive the
next event.

#### 7d. Trace JOB_PROGRESS Send Times Out

```
log.warning("trace progress send timed out; dropping", ...)
extra: {"action": "trace_progress_timeout", "job_id": ...}
```

Or on non-timeout exception:

```
log.warning("trace progress send failed; dropping", ...)
extra: {"action": "trace_progress_failed", "job_id": ...}
```

The trace notification is dropped. The durable audit row in `rules_job_log`
is unaffected. The frontend will not see this trace event in SSE.

#### 7e. JOB_COMPLETED Send Fails

```
log.warning("JOB_COMPLETED send failed; frontend will synthesize from DDB", ...)
extra: {"action": "send_completed_failed", "job_id": ...}
```

The job is already in `PROCESSED` state in DDB before the send is attempted.
The frontend synthesizes `JOB_COMPLETED` from the DDB record on its next poll
or when its abort deadline is reached.

**User-facing SSE event:** `JOB_COMPLETED` (synthesized from DDB). The client
sees the correct result, just with a delay.

---

### 8. Frontend Dead / SSE Client Disconnects

**What is it:** The frontend process dies, the client closes the SSE connection,
or the job's `abort_at` deadline elapses without a terminal event.

#### 8a. Client Disconnects (CancelledError)

```
log.info("stream_job: cancelled (SIGTERM)", ...)
extra: {"job_id": ...}
```

Best-effort `ABORTED` write to DDB. Best-effort `JOB_ABORTED` SSE yield.
`CancelledError` is re-raised.

The backend worker continues running unless its own liveness monitor detects
the stale `updated_at`.

#### 8b. Abort Deadline Elapsed (stream timeout)

```
log.info("stream_job: abort deadline elapsed; job aborted", ...)
extra: {"job_id": ...}
```

Frontend reads DDB to find the current job status. Returns the appropriate
terminal SSE event based on what it finds:

- `PROCESSED` in DDB: synthesizes and yields `JOB_COMPLETED`.
- `ABORTED` in DDB: synthesizes and yields `JOB_ABORTED`.
- `CANCELLED` in DDB: synthesizes and yields `JOB_CANCELLED`.
- No terminal state: writes `ABORTED` to DDB and yields `JOB_ABORTED`.

#### 8c. Worker Declares Frontend Dead

The worker's liveness monitor polls `updated_at` on the job record every
`liveness_check_interval_seconds`.

If `updated_at` is stale beyond `worker_stale_threshold_seconds`:

```
log.warning("liveness stale", ...)
extra: {"action": "liveness_stale", "age_ms": ..., "threshold_ms": ...}
```

`_frontend_dead` is set to `True`. Any in-progress `_await_with_liveness(...)`
awaitable is cancelled immediately via `_frontend_dead_event`.

```
log.warning("frontend declared dead", ...)
extra: {"action": "frontend_dead", "job_id": ...}
```

`_handle_frontend_dead()` is called:

- Best-effort `ABORTED` write to DDB.
- Best-effort `FRONTEND_DEAD` audit row to `rules_job_log`.
- If `ABORTED` write is rejected (frontend already wrote ABORTED):
  ```
  log.info("ABORTED write rejected; frontend already aborted", ...)
  extra: {"action": "frontend_dead_race", "job_id": ...}
  ```

If record is missing for multiple consecutive polls:

```
log.warning("liveness: record missing", ...)
extra: {"action": "liveness_no_record", "missing_count": ..., "missing_limit": ...}
```

If missing count exceeds `liveness_max_missing_records`, same `_frontend_dead`
path is triggered.

**Job final state in DDB:** `ABORTED`

---

### 9. Worker Unexpected Pipeline Exception

**What is it:** Any unhandled exception from the workflow graph or node
functions that is not `FrontendDeadError` or `asyncio.CancelledError`.

**What happens:**

`Runner._execute(...)` catches all exceptions:

```python
except asyncio.CancelledError:
    # best-effort abort, re-raise
    log.warning("best-effort abort raised on cancel", ...)
    extra: {"action": "abort_raised_on_cancel", "job_id": ...}

except Exception:
    # first: log if inside pipeline vs setup
    log.error("job setup or pipeline raised unhandled exception", ...)
    extra: {"action": "pipeline_error", "job_id": ...}
    # OR
    log.error("pipeline or completion raised before or after job was set up", ...)
    extra: {"action": "completion_error", "job_id": ...}
    # then: best-effort abort, re-raise
```

**Audit events written:**

No specific audit event is written by this path. Whatever the failing node
wrote before the exception is in `rules_job_log`.

**User-facing SSE event:**

The worker sends `JOB_ABORTED` only if the liveness-monitor path or frontend
synthesizes it from the `ABORTED` DDB state. The worker itself does not send
`JOB_ABORTED` over SQS directly after a pipeline exception.

**Job final state in DDB:** `ABORTED` (best-effort; may remain `PROCESSING` if
the ABORTED write also fails).

---

### 10. SQS Delete Fails (After Processing)

**What is it:** After the worker finishes a job (success or abort), it deletes
the SQS message to prevent redelivery.

**What happens:**

If the delete fails:

```
log.warning("SQS delete failed; message may redeliver", ...)
extra: {"action": "sqs_delete_failed", "job_id": ...}
```

The SQS message redelivers after its visibility timeout. The next worker
that picks it up will attempt `_claim_job(...)`. Because the job is now in
`PROCESSED` or `ABORTED`, `claim_job(...)` raises `JobClaimConflictError`:

```
log.info("job not claimable; deleting SQS message", ...)
extra: {"action": "claim_skip", "job_id": ..., "reason": "JobClaimConflictError"}
```

The second worker deletes the message and moves on. No duplicate execution.

---

## Outcome Reference

| Scenario | Job terminal state | SSE event to client | Key log action |
|---|---|---|---|
| Parse endpoint unreachable | ABORTED | JOB_ABORTED | `pipeline_error` |
| Parse returns ok=false, repairs exhausted | PROCESSED | JOB_COMPLETED (human review) | `PARSE_CALL` + `requires_human_review` |
| LLM provider unreachable | ABORTED | JOB_ABORTED | `pipeline_error` |
| Knowledge service down | PROCESSED (degraded) | JOB_COMPLETED (human review) or JOB_ABORTED | `knowledge_search_failed` + `KNOWLEDGE_DEGRADED` |
| RequirementsService unreachable | ABORTED | JOB_ABORTED | `pipeline_error` |
| DDB down at submission | Not created | HTTP 503 | none (propagated) |
| DDB down mid-execution (audit writes) | PROCESSING (incomplete audit) | none immediately | `log_step_failed` |
| DDB down mid-execution (liveness) | ABORTED (eventual) | JOB_ABORTED (synthesized) | `liveness_poll_failed` then `frontend_dead` |
| SQS job-in queue not found | ABORTED | HTTP 500 | none (propagated) |
| SQS job-out JOB_COMPLETED lost | PROCESSED | JOB_COMPLETED (synthesized from DDB) | `send_completed_failed` |
| SQS job-out JOB_WAITING lost | PROCESSING (may timeout) | JOB_ABORTED (if deadline elapses) | `send_waiting_failed` |
| Client disconnects | ABORTED | JOB_ABORTED (best-effort) | `stream_job: cancelled` |
| Abort deadline elapsed, backend still running | ABORTED or PROCESSED | JOB_ABORTED or JOB_COMPLETED (from DDB) | `abort deadline elapsed` |
| Worker declares frontend dead | ABORTED | JOB_ABORTED (backend writes DDB; frontend synthesizes if reconnecting) | `frontend_dead` |
| SQS delete fails after completion | PROCESSED or ABORTED (unchanged) | none | `sqs_delete_failed` then `claim_skip` on redeliver |

## Simulation Notes

To exercise each scenario manually in a local canary run:

- **Parse endpoint down:** Set `RULES_PARSE_ENDPOINT` to an unreachable URL.
  Expect `JOB_ABORTED` and `pipeline_error` in logs.

- **LLM provider down:** Block outbound traffic to the LLM host or set an
  invalid `RULES_LLM_MODEL`. Expect `pipeline_error` after retries are
  exhausted.

- **Knowledge service down:** Set knowledge service endpoint to an unreachable
  URL. Expect job to complete with `requires_human_review: true` rather than
  abort.

- **DDB down:** Stop the LocalStack DynamoDB process (local testing) or use AWS
  Service Control Policies to deny access. Expect HTTP 503 on submission, or
  `log_step_failed` logs mid-execution.

- **SQS JOB_COMPLETED lost:** Inject a fault after `PROCESSED` write but before
  the SQS send. Verify the frontend synthesizes `JOB_COMPLETED` from DDB on
  the next poll.

- **Frontend dead (liveness):** Submit a job, then stop heartbeating (pause the
  frontend process). The worker's liveness monitor will log `liveness_stale`
  and abort after `worker_stale_threshold_seconds`.
