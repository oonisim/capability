# AWS Lambda — Design Considerations

## Concurrency Model and Thread Safety

### Lambda execution model

Each Lambda container processes **one invocation at a time**.  AWS achieves
concurrency by provisioning additional container instances, not by
parallelising requests inside a single container.

Implications for Python module-level state:

- **Warm reuse is serial**: a warm container is always idle when the next
  invocation arrives.  There is no race between two invocations sharing the
  same container.
- **Cross-invocation contention**: impossible — invocations are sequential
  per container.
- **Intra-invocation contention**: possible if the handler spawns threads
  (e.g. `concurrent.futures.ThreadPoolExecutor`) and those threads share
  mutable state.

### Thread safety for the Bearer token cache

Python `threading.Lock` is not needed for cross-invocation safety (Lambda
is serial per container) but IS needed for intra-invocation thread safety when
the handler fans out parallel API calls.

| Scenario                                                 | Lock needed?                        |
|----------------------------------------------------------|-------------------------------------|
| Cross-invocation (sequential warm reuse)                 | No — Lambda is serial per container |
| Intra-invocation, main thread only                       | No                                  |
| Intra-invocation, threaded fan-out can call concurrently | Yes — need `threading.Lock`         |

A contentious property (via getter/setter) uses the same lock for consistency.
The setter is called once per invocation in the main thread before any
threads are spawned, so contention on the setter itself is theoretical;
keeping the lock is correct practice and costs nothing.

