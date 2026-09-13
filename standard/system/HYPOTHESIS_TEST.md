# Hypothesis Test Standard

## Principle

An experiment is useful only if it can change a decision for a known reason.
Before testing an answer, test the question:

1. **Establish the problem.** Show that the observed effect is real and present
   in the available data. Name the decision that depends on it. If the data do
   not contain the required signal, the question is untestable—not evidence for
   or against anything.
2. **Expose the causal claim.** State why the proposed cause could produce the
   effect, what competing causes remain, and what result would distinguish
   them. A measurable proxy is not the problem merely because it is convenient.
3. **Change one thing.** Isolate control from treatment so that inputs,
   knowledge, outputs, expected answers, evaluator decisions, and prior results
   cannot cross between them. Otherwise, attribution is impossible.
4. **Prove the instrument.** Before trusting a result, show that the harness,
   evaluator, and data can detect both success and failure. An experiment that
   cannot reject its hypothesis produces ceremony, not knowledge.
5. **Buy evidence in increments.** Begin with the smallest representative slice.
   Examine every failure, repair the experiment rather than its outputs, and
   scale only after the current slice is valid and reproducible. Never risk the
   full budget on an unproved setup.

```text
problem -> cause -> discriminating prediction -> isolated test -> decision
```

If two setups differ in more than the one tested variable, the test is broken.
You may still learn that "something changed," but you cannot know what caused
it.

## Method

1. **Observe**: state what happened without guessing why.
2. **Explain**: propose one possible cause.
3. **Predict**: state what should happen if that cause is correct.
4. **Test**: change only one thing and keep everything else the same.
5. **Check**: compare the result with the prediction and check that working
   behaviour did not break.
6. **Learn**: accept, reject, or revise the explanation from the evidence.

Use the smallest test that can prove the explanation wrong. Record the setup
and result before interpreting them. Do not change the test after seeing its
result.

If the result is unclear, improve the test and try again. Test a new explanation
only in a new experiment.

## Scale In Verified Slices

Do not put the full experiment budget into one unverified batch. Start with a
small representative slice that can expose prompt, schema, tool, context,
artifact, and scoring defects cheaply.

After each slice:

1. verify every mechanical and contamination gate;
2. fix the experiment harness rather than repairing generated outputs;
3. freeze a new experiment version when an input, instruction, tool, model
   setting, or scoring rule changes after generation; and
4. scale to the next larger slice only when the current slice is reproducible
   and valid.

Predeclare slice sizes and stop rules before generation. A failed small slice
must stop the larger batch. Do not mix outputs from failed and corrected slices
to recover the planned sample size. This limits wasted model calls and prevents
one setup defect from invalidating an expensive full run.

## Pre-Generation Validation Gate

Do not launch the experiment immediately after writing its plan. First perform
and record an independent step-back review of the complete frozen setup.

The gate must verify:

- **Problem formation:** factual observations establish the problem, the
  proposed hypothesis addresses a plausible cause rather than a convenient
  proxy, competing explanations are named, and the possible results can inform
  a real decision.
- **Test conditions:** the hypothesis is falsifiable, the prediction is
  measurable, the control and treatment differ by exactly one variable, and
  every outcome has a predeclared classification.
- **Failure modes:** list model, prompt, context, tool, schema, parsing,
  validation, codegen, scoring, ordering, concurrency, timeout, contamination,
  and artifact failures that could imitate or hide the predicted effect.
- **Experiment setup:** verify exact inputs, hashes, prompt rendering, model and
  settings, fresh-context policy, call limits, fan-out, slice order, retry rule,
  output schema, evaluator, mutation schedule, stop rules, and authority
  boundaries are complete and mutually consistent.
- **Evidence sufficiency:** inspect the available corpus and logs and prove they
  contain the event, variation, control condition, and measurable outcome the
  hypothesis requires. Distinguish an observed zero from missing, filtered,
  capped, unavailable, or never-recorded evidence.
- **Threats to validity:** actively look for ambiguous instructions, multiple
  changed variables, target leakage, historical-answer leakage, unbalanced
  strata, post-output discretion, evaluator dependence on the treatment,
  capped or hidden errors, irreproducible tools, and a full-batch launch before
  the harness is proven.

For every identified failure mode, record one of:

```text
prevented by <gate>
detected by <check>
accepted limitation with <bounded impact>
experiment blocker
```

The validation gate passes only when no experiment blocker remains and every
test condition is mechanically checkable. Approval of prose or intent alone is
not enough. Validate a dry-run or zero/one-call preflight where practical,
including artifact creation and evaluator execution, before spending the first
real slice.

If the gate fails, revise the setup before generation. If generation has
already started, stop launching new calls, preserve the partial run as invalid
preflight evidence, and create a new experiment version. Never repair the plan
in place and continue the same run.

## Fair Comparison

Before running a control/treatment comparison, write down the single variable
being tested.

Give the experiment set one stable version before it starts. The version must
identify exactly which data, prompts, authority files, tools, model settings,
slice plan, and scoring rules belong to that experiment. Anyone reading an
artifact later must be able to tell unambiguously which experiment set produced
it.

Do not confuse review-cycle numbers with experiment versions. A plan may go
through review v2 or v3 before it is approved, but the experiment remains v1 if
no run has started and the frozen experiment set is still the first one. If the
data, prompts, authority files, tools, model settings, slice plan, or scoring
rules change after any output is generated, start a new experiment version and
do not mix results across versions.

Everything else must match:

- same input data;
- same source extraction;
- same task instruction;
- same output schema;
- same model, model settings, call count, and retry rule;
- same tools, except the authority or feature being tested;
- same validation and scoring rule;
- same context policy: either all fresh or all carried, never mixed.

If any other difference is necessary, name it as a second variable and split the
work into another experiment. Do not explain a result using a variable that was
not isolated.

## Reproducibility

An experiment is not trustworthy unless another person can reproduce the same
result by following the same recorded steps.

Before running, record the exact setup:

- experiment version;
- input data and order;
- prompts and rendered prompt hashes;
- authority files and hashes;
- tools and tool versions;
- model name and settings;
- batching, retry, and stop rules;
- output schema;
- scoring rule.

During the run, preserve the raw evidence:

- raw model output;
- parsed output;
- tool telemetry;
- validation/codegen results;
- decision records, including accepted and rejected choices.

Do not rely on memory, chat history, or hidden local context. If a future runner
cannot repeat the run from the documented files and steps, the experiment is only
an observation, not evidence.

## Bias Control

Keep the answer out of the question.

Do not give one arm hints, examples, labels, expected outputs, source oracles,
prior results, or reports that the other arm does not receive. Do not use an
example that has the same shape, unit codes, or target-specific solution as the
record being tested.

Segregate artifacts:

- old experiments are not inputs;
- sibling outputs are not inputs;
- review comments are not inputs;
- evaluation oracles are frozen before outputs are opened;
- scoring is blind to arm identity until semantic verdicts are recorded;
- contaminated runs are deleted or archived outside the allowed read paths.

Randomize or balance record order when order can affect the result. Record the
seed or fixed schedule. If a launch/order deviation happens, report it as a
limitation instead of hiding it.

## After the Test

Do not stop at "it worked" or "it failed." Ask:

1. Did the predicted cause actually occur?
2. Did the test measure that cause, or something else?
3. Did every test case receive the intended change?
4. Did the change solve the whole problem, or only remove one wrong result?
5. Did an unrelated failure distort the result?
6. Were the control and treatment identical except for the tested variable?
7. Did any prompt, example, artifact, or evaluator knowledge leak the answer?

Before blaming the tested change:

- compare the exact input and output bytes or structure;
- prove whether the defect already existed in the input;
- inspect every validation layer, not only the first error list;
- treat capped, grouped, or hidden errors as unknown, not absent;
- verify summaries and decision traces against the saved artifacts.

If contamination or uncontrolled setup differences are found, do not salvage a
conclusion from the run. Keep only factual observations, fix the design, and
rerun from a clean setup.

When several validators own different checks, combine their evidence. Do not
let one non-empty error list hide another validator's errors.

Separate these conclusions:

- the explanation was wrong;
- the explanation was partly right, but the solution was incomplete;
- the test was flawed, so no conclusion is possible.

Record the lesson before designing the next test. The next test should examine
only the smallest remaining uncertainty.

## Existing Experiment Examples

For a compact source-grounded synthetic experiment, see
`doc/application/scribe/agent/compiler/SYNTETIC_EXPERIMENT_EXAMPLE.md`.

For established control/treatment plans, execution records, and result reports,
see `doc/application/scribe/agent/compiler/experiment/`.
