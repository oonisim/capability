implementation slice by slice within the review workflow.

Always ask. Is it the Right Problem Framing?
Tackling a wrong problem will result in wasted effort and time.
It is better to ask and clarify than to assume and conclude from memory.

1. 本当に解くべき問題は何か
2. 原因はどの境界にあるか
3. 既存設計を変える必要があるか
4. 変更が問題の範囲を越えていないか
5. より小さく一般的な解決方法がないか

Isolate the responsibility in a boundary in a separate .py file. Do not mix.
Create a test file for each .py file.
Be able to plug-in/out the .py file to the main workflow without embedding into the main workflow.
It enforces clear isolation of responsibility and makes it easier to test and maintain.

Keep Simple. No over-engineering. complexity is evil.

## Correct Authority

Identify the source of truth, decision authorities. Do not follow non-authoritative.

# Fact Based, Data Supported

Do not assume or conclude from memory. Do go through the code line by line to verify the truth, verify the logs what exactly occurred so far and the issues match what logs say.
Go through all the data and verify the plan/code/fix/changes will work.

### Concrete Example
Do not go after issues that do not exist.

For a real issue, create a concrete use case or failure mode example
with the real data to detail the cause/effect mechanism and
what is the consequence (issue, error).

No concrete example, no such case. Do not go after issues that do not exist.


# No Patch Work but Hit the Root Solution

If we start patching issues to workaround case by case, it will result in messy Frankenstein patch works.
Think like Ken Thompson. Think deeper to condense the idea and come up with concise hit-the-root-cause right solution.

Look for opportunities to simplify and reduce redundancy. The goal is not
reduction for its own sake; the goal is condensation without losing function or
information.

## 80/20 Rule
80/20 rule. Find less than 20% can cover more than 80% solution.

## Simplicity
Ken Thompson approach.
Reduce and condense the ideas to the essences, identify a simple but genric core mechanism to address the issues.

Keep simple.
No patch work, get to the root/core mechanism. See doc/standards/system/ken_thompson.md.
Verify the solution is what Ken would do. If not, think harder and deeper.

complexity is evil.
do not be too strict, which has been the cause of false validation and damages.


## Isolation and Responsibility
Isolation, Isolation, Isolation.
Isolate the responsibility in a boundary in a separate .py file. Do not mix. Create a test file for each .py file.
Be able to plug-in/out the .py file to the main workflow without embedding into the main workflow. It enforces clear isolation of responsibility and makes it easier to test and
maintain.

Do not touch other code just to make the parser work. Use adapter.py if need adjustment on the parser side.

---

# Prevent False Validation
make sure not to cause false validation that is damaging (cause unnecessary AST LLM retry that can further damage correct AST nodes or false error nodes).
  Unnecessary strict validation had cause it and had serious impacts.
  It is better to avoid false validation. We an catch error in the later verification by comparing the source data and take better fix action.
  False validation wil


---

# Review


ask for review at each slice.

Follow

doc/standards/system/DESIGN.md
doc/standards/system/ken_thompson.md
doc/standards/code/python/STANDARD.md
doc/standards/code/test/STANDARD.md

For compiler only, verify compliance with the project equivalents of these
anonymised example paths:
data/rules_service/landing/md/guide/RuleLanguage.md
data/rules_service/landing/bnf/RuleLanguage_slim.bnf
src/app/rules/agent/doc/DESIGN_COMPILER.md (if there is discrepancy, report and ask me)


Do not just dump review request without factual basis by going through the data/log and prove the plan will work.
data/requirements_service/example for RequirementsService requirements.
data/rules/example/block for Rules.
platform/observability/deployment/rules_service/log/observe/example-run for the execution logs especially AST and LLM logs.
