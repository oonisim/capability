# Principle

Write statement by a clever mind with structure, intent, hard effort to communicate simple/easy.
Be simple, succinct, but lose no information. Compress (think harder here) the idea.

1. Think deep to the root.
2. Think deep the communication structure to convey the root idea/concept/mechanism.
3. Organize the communication structure into the writing format.

## Intention

Reader must be able to recognize the Intention of the author that drives the objective,
conclusion, structure, navigation, and style. 

The **actionable** conclusion must come first.
Then the root cause, cause/effect mechanism, and evidences that supports the logic.
Writing that lets the reader ask 'so what?' has no value.

## Simplicity

# Structure

## MECE

## Logic Tree

Write design policies in this order:

1. **Objective** — problem the rule prevents
2. **Policy** — rule developers must follow
3. **Constraint** — system property requiring the rule
4. **Convention** — naming or structural rule
5. **Example** — minimal illustration

## Template

Objective  
Prevent `<problem>`.

Policy  
`<rule developers must follow>`

Constraint  
`<system property>`

Failure Mode  
`<what breaks without the rule>`

Convention
```
<pattern>
```

Example
```
<example>
```

## Example: Design Policy Structure

Use the following structure when writing design policies.

```
Shell Scripting
Namespace

Objective
Avoid function name conflicts when smst-tool is sourced into other repositories.

Policy
Sourceable shell modules must use the namespace

<category>_<service>_<verb>[_<object>]

in both filenames and function names.

Constraint
Shell functions share a single global namespace when scripts are sourced.

Failure Mode
Without deterministic naming, functions from different modules can override each other when multiple files are sourced in the same shell process.

This commonly occurs because smst-tool is reused across repositories, often as a git submodule.

Scope
smst-tool contains two types of shell scripts:

scripts executed directly
scripts sourced as reusable modules

Only sourced modules follow the namespace convention.

Convention

File
_<category>_<service>.sh

Function
_<category>_<service>_<verb>[_<object>]

Example

_aws_s3.sh

_aws_s3_upload
_aws_s3_download
_aws_s3_sync_bucket

Result
smst-tool can be safely sourced by any repository without introducing function name conflicts.
```

Structure Explanation

- **Objective** — states the problem being solved
- **Policy** — the rule developers must follow
- **Constraint** — system property that requires the rule
- **Failure Mode** — what breaks if the rule is ignored
- **Scope** — where the rule applies
- **Convention** — naming or structural pattern
- **Example** — concrete illustration of the rule
- **Result** — outcome when the rule is followed

---
# Terminology

## Test

1. Validate - Fit for Purpose. Assure what we do has real world value added.
2. Verify - Conformance and Compliance. Assure AS IS complies with TO BE. (Requirement testing)

### Validation
Confirm the system solves the real problem.

Comparison:
User need ↔ System behavior

Typical methods:

- user acceptance
- real-world usage
- product evaluation
- field testing

Question it answers:
“Does this actually solve the user’s problem?”

Example
Confirm that the namespace convention actually prevents conflicts across repositories.

### Verification
- tests
- formal checks
- reviews
- static analysis

Question it answers:
“Did we build it according to the spec?”

Example
Spec: function must follow <category>_<service>_<verb>
Verify: check all functions follow the rule.

---
