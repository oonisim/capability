# LLM Lessons

Project names, paths, and identifiers in the examples are anonymised.

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
CASE-A AST experiment, the model considered `GroupRule` but rejected it before
symbol lookup because it had split the mutual-exclusion note away from the
counted population. A trace of only executed searches would hide the real
failure.

For diagnostic prompts, require concise checkable records such as:

```json
{"source_atom_id":"","decision_kind":"candidate_rejected","candidate":"","decision":"","reason":"","evidence":"","authority_action":"lookup_skipped","request_id":null}
```

Do not request private chain-of-thought. Ask for observable decisions, evidence,
candidate names, lookup status, and short reasons.

## CASE-B: Treat LLM Input Shape as Behaviour

An LLM request schema is part of runtime behaviour, even when a changed field
adds no domain meaning. CASE-B exposed this boundary: adding `pool_id` to pool
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

## CASE-C: Comma-Formatted Item Lists Mean Choice, Not Require-All

When target rule-language list semantics matter, LLM-facing compiler projections must
use explicit `and` or `or`. Do not use a comma as a neutral list separator for
item lists.

CASE-C exposed the failure mode. The source requirement was a mandatory listed
set:

```text
The record requires the following four items (24 credits):

ITEM1001 Example item one
ITEM1002 Example item two
ITEM1003 Example item three
ITEM1004 Example item four
```

The generated target representation used a comma `ItemList`:

```text
4 Items in ITEM 1001, ITEM 1002, ITEM 1003, ITEM 1004
  Label LABELTAG "Part A. Core requirements"
```

That is a choice/OR carrier in target rule language, not a require-all carrier. The LLM did
not infer that a four-line mandatory source list meant all four items. It copied
the comma surface form it received.

Local log review confirmed the behavior:

- 853 local `*_AST_EXTRACTED.log` files were scanned.
- 808 contained parseable AST/IR payloads.
- ItemList operators found locally: `single=6666`, `comma=1230`, `or=191`,
  `and=58`.
- Every `operator="and"` ItemList had exactly two items.
- `operator="and"` ItemLists with three or more items: `0`.
- `operator="comma"` ItemLists with three or more items: `917`.

The broader investigation over 851 AST_EXTRACTED logs reached the same
conclusion: multi-item comma lists were not recovered as AND. Two-item AND
lists appeared only when the source itself carried explicit `and` or `both`
wording.

The CASE-C request path had two OR-shaped inputs:

1. The obligation ledger exposed only flat `source_item` facts and no grouped
   `specified_items` / require-all authority.
2. Distillation flattened the vertical item list into comma text:
   `ITEM1001, ITEM1002, ITEM1003, ITEM1004`.

The prompt correctly taught that comma lists have OR-style target rule-language semantics.
Given comma text and no typed all-of authority, the model generated
`operator="comma"`.

Rule:

- Require-all item lists must be projected as explicit `and` text, for example
  `ITEM1001 and ITEM1002 and ITEM1003 and ITEM1004`.
- Choice / one-of item lists must be projected as explicit `or` text, for
  example `ITEM1001 or ITEM1002 or ITEM1003 or ITEM1004`.
- Nested mixed lists must preserve grouping, for example
  `(ITEM1001 or ITEM1005) and ITEM1002`.
- Comma may still appear in quoted original source, JSON syntax, prose that is
  not a semantic item list, or explicit target-language/BNF examples where comma is the
  intended OR/choice carrier.

The fix is deterministic, not prompt-side guessing: facts emit typed grouped
`specified_items` source authority, distillation exposes
`source_semantics=require_all`, `item_list_operator=AND`, and
`required_items_text` joined with `and`, and downstream reconciliation/codegen
must reject comma/OR carriers for source-proven require-all lists.
