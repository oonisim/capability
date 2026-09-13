# What SQS Is

## Core Model

SQS is for **distributed random (competing) dispatch**, not destination-aware routing.

Because of the "Distributed" nature, SQS is designed under CAP theorem constraints as a **PA system** — Partition-tolerant and Available — **sacrificing Consistency**.

Therefore, a message **can be delivered more than once** (at-least-once delivery model).

It is a **"give it to whoever is listening" model**. You never know which consumer will receive the message. SQS randomly selects who to deliver to.

---

## What SQS Actually Guarantees

Amazon SQS guarantees:

- A message is delivered to **one consumer at a time** (not all)
- Delivery is **not consumer-aware**
- **Any worker** can receive any message in the queue

So the actual model is:

```
Queue → any consumer polls → receives message → processes or ignores
```

---

## Why SQS Is Designed This Way

SQS is intentionally:

- A **decoupled work queue**
- **Not** a secure per-consumer delivery channel

It assumes: *"any worker consuming from this queue is allowed to process any message."*

Responsibility split:
- ✔ Producer decides message content
- ✔ Consumer decides whether to act

---

## What Actually Happens With Competing Consumers

When multiple consumers poll the same queue:

```
SQS → delivers message to ONE consumer
```

Not:

```
SQS → delivers message to ALL → each filters
```

### Concrete example

```
Frontend X polls
Frontend Y polls

Message intended for X
   ↓
SQS randomly gives it to Y
   ↓
Y ignores it (does not delete)
   ↓
Visibility timeout expires
   ↓
Message returns to queue
   ↓
Eventually delivered to X (or Y again)
```

### Timeline

| Time | Event |
|------|-------|
| t1 | Message for X delivered to Y → ignored |
| t2 | Visibility timeout expires |
| t3 | Delivered again — maybe X, maybe Y again |

---

## Why "Reject and Retry" Breaks Your System

The message is **not permanently lost**, but the pattern creates serious problems.

### Problem 1 — Unbounded latency

Worst case with N consumers where only 1 is correct:

```
Y → ignore → timeout
Z → ignore → timeout
A → ignore → timeout
...
Eventually X
```

Latency becomes:

```
visibility_timeout × number_of_wrong_consumers
```

That is not deterministic or acceptable for progress streaming.

### Problem 2 — Head-of-line blocking (worse with FIFO)

In a FIFO queue, a message stuck in wrong-consumer retries **blocks subsequent messages in the same group**, delaying the entire stream.

### Problem 3 — Hot-loop amplification under load

Many consumers polling → many wrong picks → repeated visibility timeouts → extra SQS traffic, cost, and CPU waste.

### Problem 4 — No fairness guarantee

SQS does **not** guarantee the correct consumer gets the message quickly. It may keep delivering to the same wrong consumer repeatedly.

### Problem 5 — Misuse of visibility timeout

Visibility timeout is designed for: *"processing is in progress."*

Using it as a *routing retry mechanism* is an abuse of the feature.

---

## The Key Misunderstanding

**What you might think:**
> "Consumers can reject messages and SQS will retry intelligently."

**Reality:**
> SQS is **blind retry**, not smart routing. It does not learn who should receive a message or who previously rejected it.

You have effectively built: **random dispatch + eventual correct handling**

Instead of: **deterministic routing**

---

## What Makes This Work Correctly

You need one of these approaches:

### Option A — Single consumer (simplest)

```
SQS → one FastAPI process → routes to clients
```
✔ Works. Matches a centralized routing design.

### Option B — Broadcast layer after SQS

```
SQS → one consumer → pub/sub → all instances
```

### Option C — Separate queues (true isolation)

```
Queue_X → only X consumes
Queue_Y → only Y consumes
```

---

## Correct Architecture: SQS + FastAPI + WebSocket

```
Backend Worker
   ↓
Amazon SQS  (progress event buffer)
   ↓
FastAPI on EC2  (consumer + router + WebSocket server)
   ↓
WebSocket
   ↓
Browser Frontend
```

### Role breakdown

| Component | Responsibility |
|-----------|---------------|
| Backend Worker | Heavy processing; publishes progress events to SQS |
| Amazon SQS | Async event queue; accumulates `task_id`-tagged events |
| FastAPI (EC2) | Polls SQS, manages WebSocket connections, routes by `task_id` |
| Browser | WebSocket client; displays progress |

The essence:
```
SQS        = event buffer
FastAPI    = event router
WebSocket  = delivery channel
Browser    = viewer
```

---

## FastAPI Core Design

### Connection registry

```python
task_connections: dict[str, set[WebSocket]] = {}
```

### WebSocket endpoint

```python
@app.websocket("/ws/{task_id}")
async def ws(websocket: WebSocket, task_id: str):
    await websocket.accept()
    task_connections.setdefault(task_id, set()).add(websocket)
    try:
        while True:
            await websocket.receive_text()
    finally:
        task_connections[task_id].discard(websocket)
```

### SQS polling loop

```python
async def poll_sqs():
    while True:
        response = sqs.receive_message(
            QueueUrl=QUEUE_URL,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20,
        )
        for msg in response.get("Messages", []):
            data = json.loads(msg["Body"])
            await dispatch(data)
            sqs.delete_message(
                QueueUrl=QUEUE_URL,
                ReceiptHandle=msg["ReceiptHandle"],
            )
```

### Routing

```python
async def dispatch(msg: dict):
    task_id = msg["task_id"]
    for ws in task_connections.get(task_id, []):
        await ws.send_json(msg)
```

---

## Single Queue Is Enough

### Why you do not need one queue per user

| Issue with per-user queues | Result |
|---------------------------|--------|
| Queue explosion | Cost and management hell |
| SQS polling duplicated | Resource waste |
| State consistency broken | Anti-pattern |

A **single shared queue** is correct:

- Messages are independent
- User isolation is handled by message fields
- Scale is controlled by the number of consumers, not queues

### Message design

```json
{
  "task_id": "t1",
  "user_id": "u1",
  "progress": 42
}
```

The separation keys are `task_id` (processing unit) and `user_id` (authorization unit).

### FastAPI routing (multi-user)

```python
connections: dict[str, WebSocket] = {}  # user_id → websocket

async def dispatch(msg: dict):
    user_id = msg["user_id"]
    ws = connections.get(user_id)
    if ws:
        await ws.send_json(msg)
```

### 1 queue vs. per-user queues

| Dimension | Single queue | Per-user queues |
|-----------|-------------|-----------------|
| Scalability | ✔ | ✗ |
| Management | ✔ | ✗ |
| Cost | ✔ | ✗ |
| Safety | ✔ (by app design) | ✔ |
| Simplicity | ✔ | ○ |

**Per-user queues are only justified when:**
- Finance or regulated industries require physical data isolation
- Multi-tenant SaaS with contractual isolation requirements

For progress notifications (UI display), a single queue is the correct choice.

---

## Scale Considerations (Multi-EC2)

When running multiple EC2 instances, `task_connections` is split across processes.

| Option | Approach |
|--------|---------|
| A — Simple | Sticky sessions at the load balancer |
| B — Scalable | One EC2 as the WebSocket hub; others forward events to it |
| C — Full scale | SQS → one consumer → Redis pub/sub → all FastAPI instances |

---

## One-Line Summary

> SQS is a shared, partition-tolerant work queue. Safety and routing are the application's responsibility — not the queue's. One queue with `task_id`/`user_id` message fields, consumed by a single FastAPI process that holds WebSocket connections, is the correct and scalable architecture for per-user progress streaming.
