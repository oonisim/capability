
## 5. Separation of Concerns

One function, one responsibility.  If a function does two things, split it.

```
build_result_document()       — format one result into text
judge_result_with_llm()       — call LLM for one result
rerank_results_llm_judge()    — apply judge across all results
select_top_reranked_results() — filter by score threshold
```

Each can be tested, replaced, or reused independently.

---

## 4. Dependency Injection

Modules must not import or construct their own tools, config, or clients at
module scope.  Every external dependency (model name, client, threshold) must
be **passed in** as a parameter.

```python
# Bad — hidden coupling to a specific LLM provider
response = litellm.completion(model="openai/gpt-4o-mini", ...)

# Good — caller controls the model; module is provider-agnostic
def judge_result_with_llm(query: str, result: dict, model: str) -> RelevanceJudge:
    response = litellm.completion(model=model, ...)
```

---
