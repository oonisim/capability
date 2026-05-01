# Python Implementation Standards

# Style

## 1. Standard
PEP 8 and [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html) apply.

Use `ruff` and `pylint` as the canonical linters. All code must be 100% clean.
Suppress only known false-positives; always add a comment explaining the suppression.

```python
result = some_call()  # pylint: disable=no-member  # dynamic attr set by metaclass
```

Line length: **100 characters maximum**.

## 2. Documentation

### Module Docstring

Every module begins with a docstring that includes:
- Purpose in one sentence.
- A **dependency table** listing runtime and optional packages.
- A **call/data-flow diagram** showing how the public entities relate.

```python
"""Post-retrieval reranking utilities for RAG and LLM pipelines.

Dependencies
------------
| Package              | Required | Purpose                          |
|----------------------|----------|----------------------------------|
| litellm              | yes      | LLM completion gateway           |
| numpy                | yes      | nDCG score array operations      |
| pydantic             | yes      | Schema validation for judge JSON |
| scikit-learn         | yes      | ndcg_score reference impl        |
| sentence-transformers| optional | Cross-encoder model loading      |

Call Flow
---------
rerank_search_results()
    ├── [cross_encoder] → rerank_results_cross_encoder()
    │       └── build_result_document()
    └── [llm_judge]    → rerank_results_llm_judge()
            ├── requeset_result_with_llm()
            │       └── build_result_document()
            └── select_top_reranked_results()
"""
```

### Function Docstring

Every function docstring includes:
- A `summary` on what it does, informative, but succinct.
- `Args:` block — one line per argument; include type, constraint, and when to use.
- `Returns:` block.
- `Raises:` block (if any).
- A `Usage::` section showing a concrete call example.

```python
def rerank_search_results(query: str, results: list[dict], ...) -> list[dict]:
    """Dispatch reranking to a named strategy and return the filtered result list.

    Args:
        query:    Original user query that defines relevance.
        results:  Candidate retrieval-result dicts (must contain 'url', 'title',
                  'content', 'published_date').
        strategy: Backend to use — "cross_encoder" or "llm_judge".

    Returns:
        Results sorted by descending relevance score, filtered to >= min_score,
        capped at `limit` when given.

    Raises:
        ValueError: When `strategy` is not a recognised backend.

    Usage::

        top = rerank_search_results(
            query="python async patterns",
            results=search_hits,
            strategy="llm_judge",
            min_score=0.70,
            limit=5,
        )
    """
```

### Inline Comments

Add a comment only when the logic is not self-evident from the code.
Prefer a **why** comment over a **what** comment.

```python
# Normalise to float so downstream comparisons never fail on NumPy int64 labels.
return float(sum(top_k) / max(1, len(top_k)))
```

Use tables and ASCII diagrams in docstrings wherever a visual layout conveys
structure more clearly than prose.

## 3. Naming Conventinos

Word navigates thought. A name must tell what it is and what it does, so the
code self-explains — recreating the author's thought process in the reader's mind.

### Principles

| Kind        | Rule                               | Example                                              |
|-------------|------------------------------------|------------------------------------------------------|
| Module      | Noun describing the domain         | `reranking`, `embeddings`                            |
| Class       | Noun, singular                     | `RelevanceJudge`, `RankingMetrics`                   |
| Function    | `<verb>_<object>`                  | `build_result_document`, `evaluate_ranking_results`  |
| Variable    | Descriptive noun (no `tmp`, `val`) | `relevance_labels`, `score_key`                      |
| Boolean var | Adjective or `is_`/`has_` prefix   | `is_relevant`, `answer_useful`                       |
| Constant    | `UPPER_SNAKE_CASE`                 | `DEFAULT_MIN_SCORE`                                  |

### Function Naming Rationale

A function is an action. Pick the verb that matches the action category:

| Action category        | Verb       | Example                        |
|------------------------|------------|--------------------------------|
| Create / assemble      | `build`    | `build_result_document`        |
| Compute / derive       | `compute`  | `compute_ndcg_score`           |
| Score and reorder      | `rerank`   | `rerank_results_cross_encoder` |
| Filter / keep subset   | `select`   | `select_top_reranked_results`  |
| Evaluate / measure     | `evaluate` | `evaluate_ranking_results`     |
| Ask an external system | `request`  | `requeset_result_with_llm`     |

Use the **same verb** for the same action category across the codebase.
Consistency lets readers predict names before they search.

Each function can be tested, replaced, or reused independently.

## 4. Function and Block Length

| Unit               | Limit                                   |
|--------------------|-----------------------------------------|
| Function body      | 25 lines (excl. comments and docstring) |
| Nested block depth | 3 levels max                            |

If a function exceeds 25 lines, extract the excess into a named helper.

## 5. Line Length

**100 characters maximum.**

Use implicit line continuation inside brackets rather than backslash continuation.

```python
reranked = [
    {**dict(result), score_key: float(score)}
    for result, score in zip(results, scores)
]
```
---
# Structure 

Code is organised into modules that are decoupled, single-responsibility, and
independent of external dependencies. The foundation of this structure is the
**pure function**.

## 6. Immutable
Immutable data structures are preferred. If a function must mutate, it should do so
on a local copy of the data, never mutating an argument in-place.
This ensures that functions do not have side effects that can lead to unexpected
behaviour or bugs. Immutable data also simplifies reasoning about the code,
as it guarantees that data will not change unexpectedly.

## 7. Pure

A function has no state or memory of its own. It holds nothing between calls —
no cache, no counter, no reference to an external system. Its only inputs are its
parameters; its only output is its return value. Given the same arguments it always
produces the same result.

```
             ┌─────────────────────────────┐
parameters ─►│   no internal state         ├──► return value
             │   no hidden memory          │
             │   no external dependency    │
             └─────────────────────────────┘
```

This property is called **referential transparency**: the function call can be
replaced by its return value without changing the meaning of the program.

Consequences:
- The function is fully described by its signature — no hidden knowledge required.
- The function is unconditionally testable — no environment to set up or tear down.
- The function is composable — its output can be passed directly to any function
  that accepts its type.

When a function must interact with an external system (an LLM, a database, a clock),
that system is passed in as a parameter — never constructed or imported inside.
This is Dependency Injection (§8) and Separation of Concerns (§9) as direct
consequences of the pure-function requirement.

## 8. Dependency Injection

Modules must not import or construct their own tools, config, or clients at
module scope. Every external dependency (configurations, external systems) must
be **passed in** as a parameter.

Note: Knowledge is a dependency. If a function relies on knowledge of a specific 
LLM provider, that knowledge is a hidden dependency that violates the principle. 
The caller must control the choice of LLM model and the module be provider-agnostic.

```python
# Bad — hidden coupling to the knowledge of a specific LLM provider
response = litellm.completion(model="openai/gpt-4o-mini", ...)

# Good — caller controls the model; module is provider-agnostic
def request_result_with_llm(query: str, result: dict, model: str) -> RelevanceJudge:
    response = litellm.completion(model=model, ...)
```

## 9. Separation of Concerns

One function, one responsibility. If a function does two things, split it.

```
build_result_document()       — format one result into text
request_result_with_llm()     — call LLM for one result
rerank_results_llm_judge()    — apply judge across all results
select_top_reranked_results() — filter by score threshold
```

# Semantics

## 10. Type

Every function signature and every module-level variable must declare its type. 

```python
def precision_at_k(labels: list[int], k: int) -> float:
    ...
```

Prefer built-in generics (`list[int]`, `dict[str, Any]`) over `typing.List`
and `typing.Dict` (Python ≥ 3.9). See PEP 585.

Types validate **compositional correctness** — the kind and shape of data flowing
through the program. They are the static semantic layer of verification; tests
cover the behavioural layer.

# Code as Document

With the above principles applied, the code itself becomes a readable narrative
of the program's logic and structure. A reader should understand what is 
happening from a single glance — not from a comment.

---
# Error Handling

Repeated `try/except` blocks that perform the same translation — catch library
exception, log, re-raise as domain exception — violate DRY and scatter the
error boundary across the codebase. Pack the common handling into a
**decorator**: the error contract is stated once, each function body stays clean.

## 11. Error-Handling Decorator

### Problem — repeated try/except

```python
# Bad — same error translation duplicated in every AWS function
def fetch_secret(name: str, client: SecretsManagerClient) -> dict:
    try:
        return client.get_secret_value(SecretId=name)
    except ClientError as exc:
        code = exc.response["Error"]["Code"]
        logger.error("AWS error fetching '%s': %s", name, code)
        raise RuntimeError(f"AWS error {code}") from exc

def delete_secret(name: str, client: SecretsManagerClient) -> None:
    try:
        client.delete_secret(SecretId=name, ForceDeleteWithoutRecovery=True)
    except ClientError as exc:
        code = exc.response["Error"]["Code"]
        logger.error("AWS error deleting '%s': %s", name, code)
        raise RuntimeError(f"AWS error {code}") from exc
```

### Solution — decorator

```python
import functools
import logging
from collections.abc import Callable
from typing import TypeVar

from botocore.exceptions import ClientError

_F = TypeVar("_F", bound=Callable)  # pylint: disable=invalid-name  # short generic

logger = logging.getLogger(__name__)


def handle_aws_errors(func: _F) -> _F:
    """Translate botocore ClientError to RuntimeError, with one log entry.

    Apply to any function that makes AWS API calls. Catches ClientError, logs
    the function name, AWS error code and message (never secret values), then
    re-raises as RuntimeError so callers work against a stable exception type.

    Args:
        func: The decorated function. Must be a plain function or method —
              not a coroutine (async functions need an async wrapper).

    Returns:
        Wrapped function with identical signature.

    Raises:
        RuntimeError: When the underlying AWS call raises ClientError.

    Usage::

        @handle_aws_errors
        def fetch_secret(name: str, client: SecretsManagerClient) -> dict:
            return client.get_secret_value(SecretId=name)
    """
    @functools.wraps(func)
    def _wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except ClientError as exc:
            code = exc.response["Error"]["Code"]
            msg = exc.response["Error"]["Message"]
            logger.error("%s failed — AWS %s: %s", func.__qualname__, code, msg)
            raise RuntimeError(
                f"{func.__qualname__} failed — AWS {code}: {msg}"
            ) from exc

    return _wrapper  # type: ignore[return-value]


# Clean — each function body has zero error-handling boilerplate
@handle_aws_errors
def fetch_secret(name: str, client: SecretsManagerClient) -> dict:
    return client.get_secret_value(SecretId=name)

@handle_aws_errors
def delete_secret(name: str, client: SecretsManagerClient) -> None:
    client.delete_secret(SecretId=name, ForceDeleteWithoutRecovery=True)
```

### Rules

| Rule | Rationale |
|------|-----------|
| Always use `functools.wraps` | Preserves `__name__`, `__doc__`, `__qualname__` for introspection |
| Log inside the decorator, not the function | One log site; consistent format across all decorated functions |
| Never log argument values that may be secrets | Log only `func.__qualname__`, error code, and provider message |
| Re-raise as a domain exception | Callers depend on a stable type, not a library-internal type |
| One decorator per error domain | `handle_aws_errors`, `handle_db_errors` — never mix exception types |
| Narrowest possible `except` clause | Catch only what the decorated function can actually raise |

### When NOT to use a decorator

Use an inline `try/except` instead when:

- The handling is unique to that function (no pattern to reuse).
- The `except` clause must inspect the function's arguments to vary behaviour —
  that is a sign the abstraction is wrong.
- The function is a coroutine (`async def`) — use an `async`-aware wrapper.

---
# Verification

Validity of the code requires three dimensions:

| Dimension        | Question                                             | Validated by                            |
|------------------|------------------------------------------------------|-----------------------------------------|
| **Conformance**  | Does the code follow agreed rules and contracts?     | Linter (ruff, pylint), style guide      |
| **Correctness**  | Does the code express the intended semantics?        | Type system (static), tests (behavioural) |
| **Completeness** | Does the code handle all required cases?             | Tests, exhaustiveness checking          |

## 12. Linting

All code must pass `ruff check` and `pylint` with a score of 10/10.
Configure both tools in `pyproject.toml`. Suppress only genuine false-positives
and always add an inline comment explaining the suppression.

```python
# pylint: disable=too-few-public-methods  # Pydantic model; methods generated by metaclass
class RankingMetrics(BaseModel):
    ...
```

## 13. Tests

Every function must have a corresponding test module. Every path through the
function — including all branches, error paths, and strategy options — must be
exercised by at least one test. Use parameterised tests to cover multiple
conditions without duplication.

### Test Condition

A test condition is a joint statement of two constraints:

1. A constraint that the **input** must satisfy.
2. A constraint that the **output** (the map of input through the function) must satisfy.

| Field              | Description                                                  |
|--------------------|--------------------------------------------------------------|
| Requirement ID     | Traceability identifier for the behaviour being verified     |
| Input              | The specific value(s) provided to the function               |
| Input constraint   | The rule the input must satisfy to be a valid argument       |
| Output             | The expected result or side-effect                           |
| Output constraint  | The rule the output must satisfy                             |

### Boundary Condition

The upper or lower limit of an input or output constraint.

- Verify the function **rejects** inputs outside the boundary and raises the
  documented error.
- Verify the function **never produces** output outside the boundary.

### Test Case

A sequence of steps that exercises one or more test conditions and asserts the result.

### Edge Cases

Test cases that probe behaviour at or beyond boundary conditions:

| Kind               | Definition                                                 | Example                                            |
|--------------------|------------------------------------------------------------|----------------------------------------------------|
| **Error case**     | Expected outlier — caller violates a documented constraint | `k=0`, empty list, `score=1.5`                     |
| **Exception case** | Unexpected outlier — the environment fails, not the caller | Network down, memory exhausted, HTTP payload > RAM |

### Test Function Naming

`test_<function_name>__<scenario>` — double underscore separates function from scenario.

## 14. Coverage

Aim for 100% line and branch coverage on all functions. Use `pytest-cov` to
measure and report. Every line of code, every branch, and every error path must
be executed by at least one test case.
