# LLM Lessons

## Diagnostics Must Record Rejected Choices

When an experiment depends on understanding LLM behaviour, do not ask only for
what the model did. Also ask for what it deliberately did not do.

Decision traces must capture:

- selected candidates and reasons;
- rejected candidates and reasons;
- tool searches used and reasons;
- tool searches skipped and reasons;
- source atoms split or bound together and reasons;
- unsupported handoffs and reasons.

This matters because failures often happen before the visible action. In the
A6004 AST experiment, the model considered `GroupRule` but rejected it before
symbol lookup because it had split the mutual-exclusion note away from the
counted population. A trace of only executed searches would hide the real
failure.

For diagnostic prompts, require concise checkable records such as:

```json
{"source_atom_id":"","decision_kind":"candidate_rejected","candidate":"","decision":"","reason":"","evidence":"","authority_action":"lookup_skipped","request_id":null}
```

Do not request private chain-of-thought. Ask for observable decisions, evidence,
candidate names, lookup status, and short reasons.

## B6039: Treat LLM Input Shape as Behaviour

An LLM request schema is part of runtime behaviour, even when a changed field
adds no academic meaning. B6039 exposed this boundary: adding `pool_id` to pool
objects repeated identity already carried elsewhere and changed the request
shape. In a bounded rebuilt sample, the field-absent shape produced valid
initial AST JSON in three runs; the field-present shape produced malformed JSON
in two. This is correlation, not proof of deterministic model causation, but
the field had no required value and was correctly removed.

For every LLM-facing data change:

1. Capture and compare the exact Distillation output before and after.
2. Compare the complete downstream LLM input schema and shape, including
   repeated fields and nesting—not only token count or intended semantics.
3. Account for every added, removed, renamed, or repeated field and prove the
   receiving phase needs it.
4. If the fix should not change the input contract, revert incidental shape
   changes. Retain one only when measured value justifies its behavioural and
   maintenance cost.
5. Rebuild and use a bounded A/B control for prompt-sensitive paths. Report
   stochastic outcomes as evidence, never as deterministic causation.

When inspecting a request, parse its real payload boundary. Prompt examples can
contain placeholder facts such as `selection_pool`; a raw text search can
mistake those examples for duplicate record obligations.

## B2036: Comma-Formatted Unit Lists Mean Choice, Not Require-All

When Scribe unit-list semantics matter, LLM-facing compiler projections must
use explicit `and` or `or`. Do not use a comma as a neutral list separator for
unit lists.

B2036 exposed the failure mode. The source requirement was a mandatory listed
set:

```text
You must complete the following four units (24 credit points):

ACF1001 Accounting fundamentals
ECF1100 Microeconomics
ETF1100 Business statistics
MGF1010 Introduction to management
```

The generated Scribe used a comma `CourseList`:

```text
4 Classes in ACF 1001, ECF 1100, ETF 1100, MGF 1010
  Label LABELTAG "Part A. Core studies"
```

That is a choice/OR carrier in Scribe, not a require-all carrier. The LLM did
not infer that a four-line mandatory source list meant all four units. It copied
the comma surface form it received.

Local log review confirmed the behavior:

- 853 local `*_AST_EXTRACTED.log` files were scanned.
- 808 contained parseable AST/IR payloads.
- CourseList operators found locally: `single=6666`, `comma=1230`, `or=191`,
  `and=58`.
- Every `operator="and"` CourseList had exactly two items.
- `operator="and"` CourseLists with three or more items: `0`.
- `operator="comma"` CourseLists with three or more items: `917`.

The broader investigation over 851 AST_EXTRACTED logs reached the same
conclusion: multi-item comma lists were not recovered as AND. Two-code AND
lists appeared only when the source itself carried explicit `and` or `both`
wording.

The B2036 request path had two OR-shaped inputs:

1. The obligation ledger exposed only flat `source_unit` facts and no grouped
   `specified_units` / require-all authority.
2. Distillation flattened the vertical unit list into comma text:
   `ACF1001, ECF1100, ETF1100, MGF1010`.

The prompt correctly taught that comma lists have OR-style Scribe semantics.
Given comma text and no typed all-of authority, the model generated
`operator="comma"`.

Rule:

- Require-all unit lists must be projected as explicit `and` text, for example
  `ACF1001 and ECF1100 and ETF1100 and MGF1010`.
- Choice / one-of unit lists must be projected as explicit `or` text, for
  example `ACF1001 or ECF1100 or ETF1100 or MGF1010`.
- Nested mixed lists must preserve grouping, for example
  `(ACF1001 or ACW1001) and ECF1100`.
- Comma may still appear in quoted original source, JSON syntax, prose that is
  not a semantic unit list, or explicit Scribe/BNF examples where comma is the
  intended OR/choice carrier.

The fix is deterministic, not prompt-side guessing: facts emit typed grouped
`specified_units` source authority, distillation exposes
`source_semantics=require_all`, `course_list_operator=AND`, and
`required_units_text` joined with `and`, and downstream reconciliation/codegen
must reject comma/OR carriers for source-proven require-all lists.
