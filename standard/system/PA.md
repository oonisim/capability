# Problem Analysis Standard

This document defines how a suspected defect is analysed, evidenced, and
written up so that reports are consistent, falsifiable, and safe to act on.

`DEBUG.md` covers the debugging of a **known failure**: capture, isolate, fix,
verify. This standard covers the step before and around it: deciding whether a
suspected problem is real, how strong the evidence actually is, and how to write
it down so the next reader can act without repeating the work.

## Core Principle: A Finding Is A Claim, And Claims Must Be Falsifiable

Anything written in a defect report is a claim about the system. Every claim
must state what would prove it wrong, and must separate what was **proved** from
what was **assumed**.

The failure mode of analysis is not missing a bug. It is reporting a bug that is
not there, or reporting a real bug with a wrong cause, because both send the
next engineer somewhere useless while carrying the authority of a written
report.

## The Three Phases

Every problem analysis runs three phases in order. A report that skips a phase
must say so.

| Phase | Question | Output |
| --- | --- | --- |
| 1. Find | Is there a defect here at all? | A mechanism stated as cause and effect |
| 2. Concretise | What exactly goes wrong, with real data? | A worked failure mode example |
| 3. Re-verify | Is every statement still true? | An evidence status, and corrections |

Phase 3 is not optional and is not a formality. In practice it is where most
errors are found, including errors in phases 1 and 2 of the same report.

## Phase 1: Find The Mechanism

State the defect as a **cause and effect chain**, not as a smell.

- Not: "this function duplicates another one".
- But: "a reviewer tightens validation in A, the identical copy in B keeps the
  old rule, the two paths silently disagree, and no import links them so no
  tooling flags it".

Rules:

1. **Name the owning boundary.** Which module owns the concern that is wrong.
   A defect with no owner is an observation, not a defect.
2. **Trace to a consumer.** A wrong value that nothing reads is dormant, not
   active. Grep for every reader before claiming impact. If there is no reader,
   say so and classify the defect as dormant.
3. **Prefer one general mechanism over many special cases.** If several findings
   share a cause, report the cause once. See `ken_thompson.md`.
4. **Distinguish design consequence from bug.** If the code does exactly what
   its design implies, the defect is in the design decision, and the report must
   say that rather than implying a local patch will fix it.

## Phase 2: Concretise With A Worked Failure Mode Example

A mechanism without an example is not actionable. Every finding carries an
example that shows the failing action end to end.

An example must be:

- **Concrete.** Real identifiers, real source text, real line numbers.
- **Traced.** Show the value or state at each hop, not just the start and end.
- **Attributed.** Say whether it is taken from a real run, or constructed from
  the code's logic. Never let a constructed example read as an observed one.
- **Counterfactual.** State what would have happened had the defect not been
  present. "Had step 2 fired, step 3 would never have run" is the sentence that
  proves the mechanism is the cause.

Where the audience is mixed, add a short plain-language section alongside the
technical one. Both describe the same mechanism; neither replaces the other.

### Corpus check: is the defect active or dormant?

Before assigning severity, ask whether data exists that triggers the defect.

- Scan the real input corpus for the triggering shape and report the count.
- If no triggering data exists, the defect is **dormant**. Say so, and lower the
  severity. A real logic flaw with no data to exercise it is not a high-priority
  defect.
- Report corpus counts as **upper bounds** when they come from text or shape
  matching rather than from the system's own classification. Say which.

## Phase 3: Re-verify Before And After Writing

Re-verification is a separate pass with a hostile mindset: try to falsify your
own report.

### Re-read from disk

Always re-read the file immediately before asserting anything about it. Files
change during an investigation. A stale read produces confident false findings.

### Verify every anchor

For each cited `path:line`, read that exact line and assert the expected token
is present. Do not trust a line number carried over from an earlier step.

Cite the line that contains the claim. A range that starts on the previous
statement's closing bracket is an error, even though it "contains" the code.

### Re-run every executable claim

If the report contains a reproduction, run it again at write-up time. If it
contains a count, recompute it. If it contains arithmetic, redo it, including
unit conversions. Bytes are not kilobytes.

### A failing check is not automatically a failing claim

When a verification command contradicts the report, suspect the command first.
Confirm by hand before changing the report. Verification scripts have bugs at
the same rate as any other code.

### Check internal consistency

A report must not disagree with itself. Totals must match the rows they
summarise. Severity in an index must match severity in the body. When a finding
is downgraded during re-verification, update every place it appears.

## Evidence Status Is Mandatory

Every report states, explicitly, in its own section:

- **Proved** by code reading, by reproduction, or by production evidence, and by which.
- **Not proved**, and what specifically remains unknown.
- **How to close the gap**, with a method that is known to work.

### Absence of evidence is not evidence of absence

Before treating "zero occurrences" as meaningful, prove the thing would have
been recorded had it happened. If the layer emits no telemetry, a zero count
says nothing about behaviour and must not be reported as if it did.

This is the single most common way a defect report reaches a confident wrong
conclusion.

### Record disproved verification routes

If a proposed way of confirming the defect turns out not to work, record it as
**do not use, and why**. A plausible but broken verification method is worse
than none, because it yields a confident wrong answer. Removing it silently
guarantees someone re-derives it.

## Correcting A Report

Reports are corrected in place, not rewritten to look as though they were always
right.

- **Retain the superseded claim** verbatim, marked as superseded with the date
  and the proof against it. The wrong hypothesis is useful: it stops the next
  reader re-forming it.
- **Correct every dependent statement**, including title, summary, severity, and
  any cross-reference in other documents.
- **Never edit history.** Changelogs and release notes record what was true when
  written. Correct the live report; leave the historical record alone.

## Report Structure

Defect reports follow the mandated JIRA section order in
`doc/application/scribe/agent/issue`. All thirteen sections are required:

```text
Jira Fields, Description, Environment, Impact, Steps To Reproduce,
Expected Result, Actual Result, Evidence, Root Cause Hypothesis,
Proposed Fix, Acceptance Criteria, Non-Goals, Fix Implemented
```

Add, as needed:

- **Failure mode in plain words**, for a mixed audience.
- **Worked failure mode**, the traced example.
- **Hypotheses tested and disproved**, a table with the evidence that killed
  each one, so they are not re-investigated.
- **Evidence status**, the proved and not-proved split.

`Fix Implemented` is one of:

```text
Status: not fixed.
Status: local fix implemented; deployed canary replay still required.
Status: fixed and verified in the deployed dev canary.
```

## Proposed Fix Rules

1. **Offer options only when they are genuinely different**, and say which one
   fixes the defect versus which only removes misleading code. Do not present a
   cosmetic option as equivalent to a behavioural one.
2. **Say what decides between them.** A choice with no decision criterion is an
   unfinished analysis.
3. **Require a regression guard** that pins the specific assumption that failed,
   for example the ordering of two pipeline passes, not merely the output.
4. **Flag behaviour-changing fixes** as needing replay verification, separately
   from diagnostic-only fixes that cannot change a passing run.

## Severity

Severity reflects **demonstrated impact**, not how alarming the code looks.

| Severity | Meaning |
| --- | --- |
| High | Proved to produce wrong output or data loss on real input |
| Medium | Proved mechanism, plausible impact, not yet observed on real input |
| Low | Dormant: no consumer, or no triggering data in the corpus |

A finding whose impact claim is disproved during re-verification is downgraded,
not deleted. The mechanism may still be real.

## Anti-Patterns

| Anti-pattern | Why it fails |
| --- | --- |
| Claiming impact without tracing a consumer | The signal may be unread, making the defect dormant |
| Treating zero log hits as proof of absence | The path may simply not be instrumented |
| Constructed example presented as observed | Inflates confidence and misleads prioritisation |
| Quoting an offending pattern verbatim | Reintroduces the violation into the reporting document |
| Deleting a wrong hypothesis | Guarantees someone re-forms it |
| Rewriting history to match current paths | Destroys the record of what was true when |
| Index and body disagreeing on severity | The reader cannot tell which is current |
| Fixing many symptoms of one cause | Produces patch sprawl instead of a mechanism fix |

## Checklist

Before publishing:

- [ ] Mechanism stated as cause and effect, with an owning boundary.
- [ ] Every consumer traced; dormancy stated if there is none.
- [ ] Worked failure mode example, attributed as observed or constructed.
- [ ] Counterfactual stated.
- [ ] Corpus checked for triggering data; dormant findings marked.
- [ ] Every `path:line` anchor re-read from disk and confirmed.
- [ ] Every reproduction re-run; every count recomputed; arithmetic and units checked.
- [ ] Evidence status section present, splitting proved from not proved.
- [ ] Any disproved verification route recorded as do-not-use.
- [ ] Superseded claims retained and marked, not deleted.
- [ ] Internal consistency: totals, severities, titles, cross-references.
- [ ] All thirteen mandated sections present.
- [ ] No em-dash characters anywhere in the document.
