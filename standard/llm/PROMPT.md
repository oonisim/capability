# LLM Prompt Contract Standard

Project names, paths, and identifiers in the examples are anonymised.

This standard defines what a production LLM prompt contract must do. A prompt is
not a bag of hints. It is an interface contract between source evidence, the
model, validators, and downstream code.

## Core Rule

A good prompt makes the correct output the simplest output.

Design prompts so the shortest reasoning path leads to the correct answer.

It does that by stating:

- the exact task for this call;
- the authority order for conflicting information;
- the output contract and forbidden outputs;
- the structural invariant behind local rules;
- the decision rule for when output is allowed;
- how to preserve facts that cannot be represented safely;
- the verification boundary after the model returns.

Do not patch prompts with one-off warnings when one deeper invariant explains the
class of errors.

Prompt changes must be small and controlled. LLM behaviour is sensitive to even
minor wording changes, so change one thing at a time, keep the diff minimal, and
measure before changing the next thing. If multiple prompt variables change at
once, the result cannot identify what caused the behaviour change.

## Explicitness Directs Attention

A prompt does not only describe a task. It allocates the model's attention.

Attention is a finite budget spread unevenly across the context, and the model
holds no persistent note of what matters (see MECHANISM.md). Ambiguity forces
the model to guess where to look, and every guess spends attention on the wrong
token.

Reduce ambiguity by making things explicit. Explicit structure lets the model
answer one question directly:

> What should I pay attention to?

Name the parts that carry the answer:

```text
Authority        who or what governs this decision
Current Task     what to do in this call
Evidence         the facts to reason over
Output Contract  the exact shape the result must take
```

Labeled sections are navigable. An undifferentiated block of mixed instructions,
history, examples, tool output, logs, and notes is not. Naming the parts
pre-allocates attention on the model's behalf, so the shortest reasoning path
stays aligned with the correct output.

This is the root behind the rest of this standard. Authority order, scoped
rules, explicit invariants, and output contracts are all ways of telling the
model where to look. The mechanism, including position effects and working
context, is in MECHANISM.md.

## Responsibility Boundary

Each prompt has one primary responsibility.

Supporting responsibilities are allowed only when they directly serve that
primary responsibility. For example, an extraction prompt may include task,
schema, unsupported-fact policy, and authority order, because those all support
one job: turning source evidence into the target representation. It should not
also become the compiler, workflow controller, incident log, and validator
manual.

Examples:

- extraction prompt: source evidence -> typed JSON/AST;
- planning prompt: source evidence -> grounded plan;
- repair prompt: parser or validator error -> bounded correction;
- compiler prompt: validated representation -> target syntax.

Do not make one prompt simultaneously act as schema manual, compiler design,
workflow controller, repair policy, and incident report unless the runtime truly
uses it for all of those roles. If a document must be self-contained, make the
sections explicit so the model can distinguish task instructions from reference
material.

State what the model must not emit. For example, an AST extraction prompt should
say it must not emit compiler result objects, rendered DSL text, prose,
Markdown fences, or validation summaries.

## Precedence, Scope, And Formal Prompt Structure

Models do not inherently prove a unique ordering among applicable instructions.
When precedence is implicit, multiple interpretations may be reasonable.

Treat prompts as specifications rather than conversational prose. Every implicit
assumption is a potential source of nondeterminism.

### Authority And Precedence

Every prompt that uses evidence must define authority order. Authority determines
where an instruction originates. Precedence determines which applicable
instruction wins when instructions conflict.

The authority order should distinguish:

- current user or source facts, which own semantic intent;
- schemas, grammars, APIs, parsers, and validators, which own representation;
- local conventions, which own project-specific policy;
- retrieved context, which owns evidence only inside its source boundary;
- examples, which are style evidence only unless independently supported.

Examples must never override current source facts, parser feedback, schemas, or
grammar. Mark examples as low-authority when they are included for style or
shape.

Do not rely on ordinary-language implication for conflicts. State the ordering
explicitly:

```text
Authority Order:
System > Developer > User > Retrieved Context > Examples

Priority 1: Never reveal confidential information.
Priority 2: Answer the user's question as completely as possible.

If Priority 1 conflicts with Priority 2, follow Priority 1.
```

This is different from merely listing rules. A flat list can leave the model to
infer whether safety, correctness, completeness, source fidelity, or style should
dominate. Production prompts should define the conflict resolution policy
directly.

### Scope And Lifetime

Temporary or local instructions should explicitly define where they apply and
when they expire.

```text
The following instructions apply only to Section 3.

At the end of Section 3, these instructions expire.

Resume applying only the global rules.
```

Humans naturally infer contextual boundaries. LLMs are generally more reliable
when scope, lifetime, and reset points are stated explicitly.

### Policy And Procedure

Separate what must remain true from how the task should be performed.

```text
Policy:
- Never fabricate facts.
- Preserve source fidelity.
- Emit schema-valid JSON.

Procedure:
1. Read the provided sources.
2. Resolve references.
3. Generate the answer.
4. Validate against the schema.
```

Policies define invariants. Procedures define workflows. Changing one should not
unintentionally change the other.

### Invariants And Ordinary Rules

Some rules should never be overridden.

```text
Invariants:
- Never invent citations.
- Never modify quoted text.
- Never emit invalid JSON.

Global Rules:
1. Preserve source fidelity.
2. Be concise.

Local Override:
Scope: Analysis section only

Override:
Replace "Be concise" with "Be exhaustive."

The invariants remain in effect.
```

Explicit invariants reduce the risk that a local instruction accidentally weakens
a fundamental constraint.

### Structured Rules

Avoid relying on nearby prose to provide meaning. Each rule should stand on its
own.

Instead of:

```text
Generate SQL.

It should be PostgreSQL.
```

prefer:

```text
Task:
Generate SQL.

Constraint:
Dialect = PostgreSQL.
```

Prefer explicit conditions over nested exceptions.

Instead of:

```text
Do not summarize.
Except when...
Unless...
```

prefer:

```text
Condition:
If X, summarize.
Otherwise, preserve the original content.
```

Positive applicability rules are usually easier to reason about than chains of
negated exceptions.

### Output Contracts, Assumptions, And Verification

Rather than requesting an output style, specify an output contract:

```text
Output Contract:
- Valid JSON only.
- No surrounding prose.
- UTF-8.
- Schema version 2.
- Every required field present.
```

State important assumptions explicitly:

```text
Assumptions:
- Retrieved documents are authoritative only for their quoted source content.
- Missing information must not be inferred.
- Unknown values should be reported as unknown.
```

Design the prompt so correctness is easy to verify:

```text
Verification:
- Every factual claim cites a source.
- Every citation corresponds to an input document.
- Every JSON document validates against the schema.
```

Contracts and verification rules make outputs easier to validate
automatically.

### Recommended Policy Structure

For complex prompts, prefer a small policy specification over conversational
prose.

```text
Authority Order:
System > Developer > User > Retrieved Context > Examples

Conflict Resolution:
1. Higher priority wins.
2. If priorities are equal, the narrower scope wins.
3. If scope is equal, the most recent applicable rule wins.
4. Otherwise, use the fallback behavior.

Invariants:
- Never fabricate facts.
- Never invent citations.
- Never emit invalid JSON.

Global Rules:
1. Preserve source fidelity.
2. Emit only schema-valid output.
3. Be concise.

Local Override:
Scope: Analysis section only
Override: Replace Rule 3 with "Be exhaustive."

End Override:
Resume applying only the global rules.

Output Contract:
- Valid JSON.
- No surrounding prose.

Fallback:
If no rule safely applies, explain why rather than guessing.
```

Define:

- authority: where an instruction originates;
- precedence: which rule wins in a conflict;
- applicability: the conditions under which the rule is active;
- scope: where the rule applies;
- lifetime: when the rule starts and stops;
- inheritance: which global rules still apply inside a local scope;
- invariants: rules that cannot be overridden;
- output contract: structural guarantees for the result;
- fallback: what to do when no rule safely applies.

Complete determinism is not available from prompt text alone. Models interpret
prompt text rather than execute it as a formal program. Deployed systems may
introduce additional policies, and semantically similar instructions can still
admit multiple reasonable interpretations.

The purpose of formal prompt structure is not to guarantee determinism. It is to
reduce ambiguity, improve consistency, simplify reasoning, and enable
deterministic validation of the model's outputs.

## Structural Invariants

Prefer one root invariant over many local prohibitions.

### Semantic Contract

Preserve meaning, not representation.

Transformation is allowed only when the output remains semantically equivalent to
the source. Do not:

- strengthen a requirement;
- weaken a requirement;
- broaden applicability;
- narrow applicability;
- change temporal meaning;
- change logical meaning;
- change quantified populations;
- introduce new assumptions.

Schema-valid output is not enough. A representation can be structurally valid and
still wrong if it changes the source meaning.

### Decision Before Generation

Decision precedes generation.

First determine whether the output is justified. Only then generate the output.
Generation must not influence the decision.

Use this rule for any prompt that maps evidence into output:

```text
Decide whether the source authorizes the output before writing the output.
If authorization is missing, emit the handoff or ask for clarification.
Do not start generating and then make the fields fit.
```

If the output is a typed projection of a grammar, schema, protocol, API, or file
format, say that directly:

```text
Each output node is the structured form of one source production.

When a node maps to a production, preserve that production's required children
and sequence in the node contract. Omit only punctuation and keywords that are
mechanically implied. If the output has no field for a required child, the
construct is unsupported.
```

This is stronger than scattered rules like "do not put X inside Y" because it
gives the model a reusable test. Local node sections should then state the
production shape, not prose exceptions.

Good prompts state the invariant and the emission gate. The invariant tells the
model what a correct output is. The gate tells the model when it is allowed to
emit that output.

For compiler-style prompts, use a pattern like:

```text
Executable output must be supported by the target representation and by the
current compiler/runtime. If the source fact is real but exact supported output
is unavailable, emit the diagnostic handoff. Do not approximate by dropping
operands, flattening logic, defaulting required identifiers, or relying on
evidence from the wrong node.
```

For non-grammar domains, use the equivalent invariant:

- API prompts: preserve required request fields, types, and call order;
- database prompts: preserve keys, cardinality, constraints, and ownership;
- document extraction prompts: preserve required source facts and citation
  links;
- transformation prompts: preserve meaning, identity, quantities, and ordering.

### Operational Prompt Invariants

Representation prompts and execution prompts need different invariants.

| Prompt kind | Input -> output | Primary invariants |
| --- | --- | --- |
| Representation | source -> AST/JSON/IR/text | semantic fidelity, completeness, schema validity, no silent information loss |
| Execution | state -> tool/action/side effect | preconditions, authority, ownership, dispatch, idempotence |

Prompts that control tools, workflows, or multi-step actions need three root
invariants:

| Invariant | Question it answers | Failure it prevents |
| --- | --- | --- |
| Precondition | When is acting allowed? | Speculative execution, gap-filling, inference |
| Ownership / dispatch | Which action owns this input? | Wrong tool, wrong workflow |
| Idempotence | What happens if this runs again? | Duplicate side effects, repeated writes |

The precondition invariant must state when the model may act at all. Use a
direct rule:

```text
Run a tool or workflow only when its required inputs are explicit in the user
request, source evidence, prior tool result, or opened file. If a required input
is missing, ask one short clarification question. Do not search, infer, or
substitute another workflow to fill the gap.
```

The ownership/dispatch invariant must state which action owns each input shape.
For tool prompts, prefer a small dispatch table with explicit match criteria and
priority order:

```text
Evaluate rows top to bottom. Use the first row whose criteria are satisfied.
If no row matches, ask one short clarification question.

| Priority | Criteria that must be explicit | Action |
| --- | --- | --- |
| 1 | User explicitly requests search or discovery | search |
| 2 | Existing file path is supplied and action is read/inspect | read file |
| 3 | Existing file path is supplied and action is generate from that file | file-generation tool |
| 4 | Requirements text is supplied directly | generate |
| 5 | Raw target syntax is supplied directly and no update target is supplied | parse |
| 6 | Raw target syntax plus explicit target ID/version/update intent are supplied | update dry-run |
```

The criteria column is the contract. Do not dispatch from weak signals such as
filename patterns, guessed identifiers, likely user intent, or a tool name that
the model prefers. If a row requires a target ID, version, source path, or update
intent, that value must be explicit in the user request, source evidence, prior
tool result, or opened file. Otherwise the precondition invariant applies: ask
instead of acting.

Dispatch rows should be mutually exclusive where possible. When overlap is
unavoidable, priority order is part of the prompt contract. Put the most
specific and safest row before broad fallback rows. For example, "generate from
this ReferenceDataService JSON file" should outrank "read a workspace file", while "parse
raw syntax" should not become "update a block" merely because a filename looks
like a block ID.

Do not replace these invariants with scattered prohibitions like "do not use X
for Y" or "do not search for Z". Such prohibitions are symptoms that the prompt
is missing either a precondition or a dispatch rule. Add the root rule, then keep
only local prohibitions that express a genuinely separate safety boundary.

Execution prompts with side effects must define idempotence:

```text
Running the same prompt twice with the same inputs must produce equivalent
outputs and must not duplicate side effects unless the workflow explicitly
permits repetition.
```

Every source fact or workflow input should have one transforming owner. Other
phases may reference it, validate it, or report on it, but they must not
reinterpret it or transform it into a different meaning.

### Source Obligation Over Pattern Habit

LLMs learn strong patterns from ordinary writing. Headings, indentation,
bullets, ordering, and visual grouping often look like structure, but they are
not executable meaning unless the source language says so.

For extraction and compiler prompts, state the invariant directly:

```text
Derive executable rules only from explicit source obligations. Do not derive
rules from source layout.
```

This is a general discipline, not a case-specific warning. The prompt should
force a conscious decision from facts and logic instead of letting the model
follow a familiar document pattern. Visual arrangement may provide labels,
evidence locations, or presentation hints. It must not create quantities,
cardinality, choices, distributions, conditions, filters, or executable
grouping.

Pair the prompt invariant with a deterministic validation boundary wherever the
wrong output has product impact. Prompt text steers the model; validators enforce
the contract after the model returns.

For complex composite inputs, prompt guidance is not enough. Add a deterministic
atom ledger before generation when the model could choose the wrong boundary for
a handoff. Use an obligation ledger separately when the model needs
source-backed evidence for rule selection.

```text
source text -> deterministic atom ledger -> model output -> validator
```

The ledger should name the smallest source-backed units that matter to the
product: required actions, quantities, identities, populations, filters,
conditions, and unsupported residues. The model may decide how to represent each
atom, but it must not decide that a whole composite section is one unsupported
atom when supported child atoms exist.

Use this rule:

```text
Unsupported parent structure does not make supported child atoms unsupported.
```

The validator should then check every ledger atom explicitly. Supported atoms
must appear in executable output. Unsupported residue must appear in the handoff
shape. A diagnostic for a whole parent section must not satisfy coverage for
mandatory child atoms unless the ledger marks those child atoms themselves as
unsupported.

Practical review test:

1. If the model used a tool to fill a missing fact, the prompt is missing or
   weakening the precondition invariant.
2. If the model used the wrong tool for a known input, the prompt is missing or
   weakening the ownership/dispatch invariant.
3. If fixing a prompt means adding one prohibition per incident, stop and replace
   the patch set with the smallest invariant that explains the incident class.
4. If the model converted visual source arrangement into executable meaning, the
   prompt is missing or weakening the source-obligation invariant.
5. If the model moved a whole complex section to handoff while losing supported
   child facts, the system is missing or weakening deterministic atomization.

### Obligation Wording Blast Radius

An obligation ledger is evidence. It must not become an unnecessary authority
gate unless the downstream validator enforces the same boundary
deterministically.

Avoid negative retry wording when the real repair is local. Negative wording
tells the model what to avoid, but often fails to keep the model inside the
smallest valid replacement. It can make the model search for a different broad
strategy instead of fixing the exact validator error.

Bad broad negative pattern:

```text
Do not repair by converting layout facts into rules.
If not sure, hand it off.
```

Also avoid strong enforcement or "binding" language when the ledger is only
evidence for the current repair:

```text
Source obligation ledger is binding.
```

Those instructions look safe, but together they can change the model's repair
target.
In the CASE-B Rules regression, this style of wording was added for a CASE-D
layout-heading problem. It did not reliably fix CASE-D, and it shifted CASE-B
retry behavior: the model stopped reproducing the old `SubsetRule` under
`GroupRule.items` failure, but began placing file-level `change_log` and
`diagnostics` fields inside `block`. The prompt had moved the model from a
narrow schema repair into a broader "preserve unsupported residue as handoff"
rewrite while it was editing the block subtree.

The trigger is the combined instruction shape, not one isolated word. A model
can reasonably read that pattern as:

```text
The previous structural interpretation was too aggressive.
Be conservative.
Prefer handoff over executable structure when layout or heading evidence is
involved.
```

That changes the perceived task. Instead of staying inside the validator error:

```text
Fix invalid SubsetRule under GroupRule.items.
```

the model may broaden the work into:

```text
Re-evaluate which parts of this AST are executable versus handoff.
Move unsupported or uncertain material into diagnostics.
```

That broad rewrite mode is the problem. It gives the model permission to edit
nearby or root-level review structures even when the validator asked for a
local schema repair.

In the CASE-B case, the highest-risk phrases were:

| Wording | Why it is dangerous |
| --- | --- |
| `Source obligation ledger is binding` | Elevates the ledger from evidence to authority, so the model polices the whole AST against it instead of fixing one error. |
| `emit one rule for that population or hand it off` | Creates a binary rewrite decision that can suppress legitimate nested structures and push the model toward diagnostics. |
| `Do not repair by converting layout facts into rules` | Correct source-layout discipline becomes risky when used as a retry repair gate instead of a local replacement rule. |

The likely internal task drift is:

```text
Original task: repair schema.
New perceived task: audit all grouping against the obligation ledger and move
questionable structures to handoff.
```

Use this rule:

```text
Add evidence to the prompt; keep prohibitions narrow and local to the failing
schema or semantic boundary.
```

If a negative instruction is needed, it must name one exact invalid output and
one exact replacement decision. Do not combine unrelated prohibitions,
source-layout policy, unsupported-handoff policy, and counted-population policy
in the same paragraph. Each added prohibition must have a regression test that
checks both the target fix and adjacent schema placement.

For obligation-ledger prompts:

- include the ledger as source evidence when the model needs it;
- do not tell the model the ledger is globally "binding" unless every binding
  consequence is enforced by validation;
- do not use obligation wording to make unsupported parent handoff decisions
  when supported child atoms may still be executable;
- keep retry prompts focused on the actual validator error;
- verify unrelated fields after a prompt change, especially root-vs-child
  placement such as `RuleFile.diagnostics` versus `Block`.

Prefer local retry wording:

```text
For this validation error only, do not replace a
GroupRule.items[].rule SubsetRule with a nested GroupRule unless that same
node's source_evidence explicitly says all child rules are required.
Otherwise place one UnsupportedRuleConstruct in RuleFile.diagnostics.
Do not move change_log, diagnostics, metadata, or block root fields.
```

This keeps the model inside the failing boundary.

## Fail Closed

The prompt must define how to handle real facts that cannot be represented.

Use an explicit unsupported, diagnostic, TODO, or handoff shape instead of
letting the model guess, omit, or approximate. The handoff must preserve:

- the source fact;
- why it is unsupported;
- where it belongs in the target structure when known;
- enough evidence for a human or deterministic repair step.

Information must never disappear silently. Every material source fact must be
represented, preserved, handed off, or diagnosed.

If downstream code cannot safely consume a construct, the prompt must not
encourage the model to emit a lossy partial representation.

Phrase fail-closed behavior as a generation rule, not merely a validator
outcome. "Validation fails" tells the model what happens after a bad answer.
"Emit only when the guard is satisfied; otherwise emit the handoff" tells the
model how to avoid the bad answer.

Approximation must be named when it is dangerous. Common unsafe approximations
include:

- dropping ranges, operands, exclusions, or qualifiers;
- flattening conditional, grouped, or mixed AND/OR logic;
- defaulting required labels, identifiers, or mappings;
- turning enforceable constraints into comments;
- relying on parent evidence for executable child facts.

## Schema And Validation

Schema detail belongs in an extraction prompt when the model must generate that
schema directly. Validator implementation detail belongs only when it changes
what the model should emit.

Useful schema prompt content:

- node purpose;
- required and optional fields;
- valid parent/child placement;
- field meanings;
- source-backed examples;
- unsupported fallback.

Lower-value content:

- full downstream result objects;
- internal validator state;
- implementation module ownership;
- codegen coverage status, unless it changes model output;
- repeated obvious constraints that a schema validator already enforces and that
  do not guide generation.

When validation behavior is included, phrase it as an output contract, not as
internal implementation narrative.

"Lower-value" is contextual. Validator or codegen detail becomes high-value
when it changes what the model may emit. For example, if a schema node exists
but the current renderer cannot safely render it, the prompt should describe
the emission boundary and unsupported fallback. It should not expose irrelevant
runtime knobs or validator state as if the model could set them.

Prefer generation-facing wording:

```text
Emit this node only when the local guard is satisfied.
```

over validator-facing wording:

```text
Validation rejects this node when the local guard is absent.
```

## Extraction Procedure

Complex extraction prompts should include a short operational procedure before
the schema reference.

Use a domain-specific version of:

1. Identify the required quantity or action.
2. Identify the counted or affected population.
3. Identify filters.
4. Identify exclusions.
5. Identify conditions.
6. Identify grouping and ordering.
7. Map to the target node or command.
8. If representation is uncertain or incomplete, emit the unsupported handoff.

This tells the model how to think before it sees a long catalog of node shapes.

## Examples

Examples are powerful and dangerous.

Rules:

- Keep cached/system examples short.
- Use examples to show shape, not to smuggle defaults.
- State when placeholder identifiers are schema-shape markers only.
- Require placeholders to be replaced with source-backed values.
- Prefer retrieved or per-call examples for rare constructs.
- Do not let a long example teach a low-frequency construct as the default.

If examples use fake identifiers, say explicitly that the model must not emit
them.

Examples must never be the only place a required field, cardinality rule, or
guard appears. If the model needs a rule to decide whether to emit a construct,
state the rule in prose near the relevant contract. Examples illustrate
contracts; they do not create them.

## Prompt Split

Use separate stable system contracts when runtime phases have different duties.
This is preferred for large systems, but not mandatory for every production
workflow. A smaller system may use one cached contract when the runtime truly
executes one combined task and the sections are clear.

Typical split:

```text
workflow contract  -> phase discipline and output rules
schema contract    -> typed representation to emit
compiler contract  -> lowering/rendering rules
phase prompt       -> what to do in this call
evidence           -> current source facts and retrieved support
```

Only send the contracts needed by the active phase. An extraction phase should
not receive compiler-only instructions when those instructions distract from the
required output.

Stable contracts should be cache-friendly. Volatile source facts, retrieved
evidence, parser errors, and job data belong in the user message for that call.

When prompts are split, each contract should own one layer:

- workflow contract: phase order, tool purpose, output discipline;
- schema contract: representation vocabulary, fields, placement, guards;
- compiler contract: lowering and render semantics;
- repair contract or phase prompt: bounded correction rules.

Do not duplicate the same rule across layers unless the repetition prevents a
real failure class. If a rule belongs to the schema contract, the workflow
contract should delegate to it rather than restating a weaker copy.

## Token Economics

Prompt quality is correctness per token, not correctness alone.

Every long section competes with source evidence and retrieved context. Keep
high-value content in stable contracts and move low-frequency or volatile detail
to retrieval, phase prompts, or validator diagnostics when possible.

High-value prompt content:

- root invariants and authority order;
- emission gates and unsupported fallback rules;
- field contracts the model must emit directly;
- extraction procedures that prevent semantic loss;
- examples that prevent common structural errors.

Lower-value prompt content unless it changes output:

- exhaustive validator internals;
- implementation history;
- downstream result schemas not emitted by the model;
- rare edge cases better supplied by retrieval or validator feedback;
- repeated constraints already implied by a stronger invariant.

Do not reduce token use by removing the rule that prevents the dominant failure
mode. Do reduce token use by replacing many local warnings with one reusable
decision rule.

## Repair Budgets

Prompts must not imply unlimited retries.

For production workflows, define:

- maximum extraction retries;
- maximum repair attempts;
- maximum reparse attempts;
- whether a repair may request regeneration;
- what state or attempt counters cannot be reset.

Repair may fix representation errors. It must not change source facts,
cardinality, identity, quantities, or policy unless new authoritative evidence
requires it.

Repair must also be local:

```text
Modify only the smallest source-backed region required to satisfy the contract.
Unrelated structures must remain unchanged.
```

Prompt changes should be monotonic unless the contract explicitly changes. Adding
a new rule should constrain behavior; it should not silently weaken a previously
guaranteed behavior.

## Evidence And Retrieval

Retrieval should answer a specific representation question, not dump loosely
related context.

Prefer:

```text
target node or action -> required grammar/API/schema question -> evidence query
```

over:

```text
entire user request -> broad similarity search
```

Retrieved evidence must be identified in logs and must keep its authority rank.
The prompt should say when evidence may support syntax or style but not invent
missing source facts.

## Observability

Production LLM calls must be debuggable after the fact.

Log enough to answer:

- which system contracts were sent;
- which phase was active;
- what exact user/phase message was sent;
- what evidence was available;
- what the model returned;
- what validator, parser, or downstream system accepted or rejected.

Prompt changes that affect production output should include regression tests or
canary evidence. Tests should assert the contract text that prevents known
failure classes when that text is part of the runtime interface.

## Review Checklist

Before deploying a prompt change, check:

- Does the prompt state one clear task?
- Do any supporting responsibilities directly serve that task?
- Does it define authority order?
- Does it define precedence for conflicts between instructions?
- Are local or temporary rules scoped with clear start, stop, and reset points?
- Does it separate policy/invariants from procedure/workflow?
- Does it state the output object and forbidden outputs?
- Does it state important assumptions instead of relying on implication?
- Does it preserve semantic meaning, not only structure?
- Does it require decision before generation where evidence gates output?
- Does it expose the root structural invariant?
- Does it state the emission gate: when output is allowed and when to hand off?
- For tool or workflow prompts, does it state the precondition invariant: when
  acting is allowed and when to ask instead?
- For tool or workflow prompts, does it state the ownership/dispatch invariant:
  which action owns each input shape?
- For tool or workflow prompts with side effects, does it define idempotence?
- Is the dispatch table ordered by priority, with explicit match criteria and a
  defined ask-clarification fallback when no row matches?
- Does every material source fact have a representation, preservation, handoff,
  or diagnostic path?
- Does it prohibit unsafe approximations rather than only saying validation will fail?
- Are examples short, source-backed, and marked with their authority?
- Are required fields and guards stated outside examples?
- Is validation guidance limited to what changes model output?
- Is the output contract easy to verify automatically?
- Are phase-specific contracts sent only where needed?
- Is duplicated guidance delegated to the owning contract?
- Is the token cost justified by the failure class it prevents?
- Are retries and repair loops bounded?
- Are repairs local, with unrelated structures preserved?
- Are prompt changes monotonic unless the contract explicitly changes?
- Can an incident report reconstruct what the model saw and why it answered?
