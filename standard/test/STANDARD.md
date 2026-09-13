# Testing Knowledge — BATS BDD Convention

This document defines **mandatory conventions for writing BATS tests** in this
repository so that both humans and automated agents produce consistent tests.

The primary rule is that every BATS test **must include a BDD-style comment
block immediately above the test**.

---

## Required BDD Comment Format

Each test must contain the following BDD lines directly above the `@test`.
Multiple **When/Then** pairs may be used if the behavior has several
observable outcomes.

```
# Scenario: <human description of the scenario>
# Given:    <initial conditions>
# When:     <action performed>
# Then:     <expected outcome>
# When:     <additional action>            (optional, repeat as needed)
# Then:     <additional expected outcome>  (optional, repeat as needed)
```

Spacing should match the format above so the comments align cleanly.

Example:

```
# Scenario: User asks for help.
# Given:    Only --help is passed.
# When:     The parser encounters --help.
# Then:     Exit 0 with usage text.
# When:     The help text is printed.
# Then:     Output contains the word "Usage".
@test "--help prints usage" {
  run my_script.sh --help
  [ "$status" -eq 0 ]
  [[ "$output" == *"Usage:"* ]]
}
```

---

## Why This Is Required

The BDD comment block ensures:

- Tests are **self‑documenting**.
- Developers can quickly understand the intent of the test.
- AI agents generating tests follow a **consistent reasoning structure**.
- Reviewers can validate test correctness without reading the whole body.

---

## Rules for Agents and Contributors

When creating or modifying BATS tests:

1. Always include the **BDD comment block** above each test.
2. Ensure the `Scenario` and `Given` lines are present.
3. Include **at least one `When` and one `Then`**.
4. Additional `When`/`Then` pairs may be added for clarity.
5. Keep wording concise and descriptive.
6. The `@test` name should summarize the scenario in short form.

Correct structure:

```
# Scenario
# Given
# When
# Then
@test "short description" {
    ...
}
```

Incorrect examples:

Missing BDD block:

```
@test "works" {
  run cmd
}
```

Incomplete BDD block:

```
# Scenario: something
@test "works" {
}
```

---

## Recommended Additional Practices

Although not mandatory, tests should also:

- Mock external services (AWS, Azure, curl) in **unit tests**.
- Use **integration tests** only under `tests/*/integration/`.
- Avoid real network calls unless explicitly marked as live tests.

---

## Mandatory Rule: Every Function Must Have Tests

Whenever a new Bash function is created, a corresponding **BATS test must be
added under the repository test directory**.

Rules:

1. Every reusable function must have **at least one unit test**.
2. Tests must live under:

```
smst-tool/tests/
```

3. The test should be placed in the **directory matching the module location**.

Example mapping:

| Function Location | Test Location |
|------------------|--------------|
| `aws/authentication/_aws_session_utility.sh` | `tests/aws/authentication/` |
| `aws/s3/_s3_common.sh` | `tests/aws/` |
| `azure/authentication/*.sh` | `tests/azure/authentication/` |
| `bash/_utility.sh` | `tests/bash/` |

4. If the corresponding directory does not exist, it **must be created**.

Example:

```
mkdir -p smst-tool/tests/aws/authentication
```

5. Tests must verify both:

- **success paths**
- **failure or edge cases**

6. All tests must follow the **BDD comment convention** described above.

---

## Enforcement

The repository contains a guard test:

```
tests/bash/enforce_bats_bdd_convention.bats
```

This test ensures:

- All BATS tests include Scenario/Given/When/Then blocks.
- Test documentation remains consistent across the repository.

The test runner (`tests/run_all.sh`) executes this check **before all other
tests**, causing CI to fail if documentation rules are violated.

---

## Mandatory Rule: Spellcheck Bash Tests and Scripts

All Bash scripts and BATS tests must pass a basic **spell‑check** to prevent
typos in comments, help text, and user‑facing messages.

Scope:

```
smst-tool/**/*.sh
smst-tool/tests/**/*.bats
```

Rules:

1. New or modified Bash scripts must be spell‑checked.
2. BATS tests must also be spell‑checked because they contain documentation
   and BDD comments.
3. Typos in user‑facing messages, help text, or comments must be corrected
   before merging.

Recommended tools:

```
codespell
aspell
```

Example usage:

```
codespell smst-tool
```

Agents and contributors should run spell‑check when:

- creating new Bash utilities
- modifying help text or error messages
- adding BATS tests

This ensures the repository remains professional and documentation remains
clear and readable.

---

## Example from This Repository

```
# Scenario: Valid AWS session already exists.
# Given:    AWS_PROFILE matches the role and cache is valid.
# When:     _aws_session_is_role_matched is executed.
# Then:     Function returns success (0).
@test "valid session returns success" {
  export AWS_PROFILE="roleA"
  run _aws_session_is_role_matched "roleA"
  [ "$status" -eq 0 ]
}
```

---

This convention applies to all directories under:

```
smst-tool/tests/
```

and should be followed by both **developers and automated coding agents**.
