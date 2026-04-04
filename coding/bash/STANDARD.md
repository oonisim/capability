# Bash Function Implementation Knowledge

This document defines the **standard rules for implementing reusable Bash
functions** in the SMST tooling repositories. The goal is to ensure that all
utility functions are predictable, composable, testable, and safe to reuse
across scripts.

## Core Principles

1. **Pure Functions**
    - Functions should avoid modifying global state.
    - All configuration should be passed via arguments.
    - Only well‑known system defaults may be assumed (e.g. `~/.aws`).

2. **Explicit Inputs**
    - Every configurable parameter must be passed as an argument.
    - Avoid reading environment variables unless explicitly documented.

3. **Deterministic Output**
    - Return values should be communicated via **exit codes**.
    - Output to stdout should be minimal or explicitly documented.

4. **No Side Effects**
    - Functions should not modify global variables.
    - Functions should not change directories.
    - Functions should not export environment variables.

5. **Fast Failure**
    - Validate inputs early.
    - Return non‑zero on invalid arguments.

6. **Reusable by Design**
    - Avoid repository‑specific paths.
    - Prefer dependency injection via arguments.

7. **Meaningful Code Block no more 25 lines without Comment.
    - If a block of code spans more than 25 lines (without comment), split into functions.
    - Wrap a long line at the meaningful places to wrap around.

8. **Informative Comments**
    - Have block/function comments what they do clearly but succinct.
    - Developers must not spend time to reverse-engineer what codes do.

9. **ATOMIC Guard**
    - An atomic sequence of work must be guarded from TERM, INT signals.
      (For instance, mktemp and do work in the mktemp and delete the temp should not be aborted by INT or TERM)

10. **Secure**
- Variable to hold sensitive information must be cleared after use, e.g. unset.
- Consider injection attacks and verify the input is limited to specific formats or length and not executable.

11. **Safe**
- Confirm the user upon any destructive actions to get the yes/no input.

12. **Shellcheck**
- 100% pass Shellcheck

13. **Watch-Outs**
- || true hiding true errors and continue as if no error (meant to coexist with set -e but side effect)

---

# Required Function Documentation Format

Every function **must include a documentation header** immediately before the
function definition.

Example format:

```
#----------------------------------------------------------------------
# Verify VPC endpoints: count, restrictive policies, availability.
#
# Checks that at least 9 endpoints exist, none have unrestricted
# Principal:* without conditions, and all are in 'available' state.
#
# Args:
#   $1  log_file — path to the debug log file
#   $2  vpc_id   — VPC ID
#   $3  region   — AWS region
#----------------------------------------------------------------------
verify_vpc_endpoints() {
    ...
}
```

Rules:

- Header delimiter must use `#----------------------------------------------------------------------`
- A short one‑line description is required.
- A longer explanation is recommended when behaviour is non‑obvious.
- All arguments must be documented.
- Arguments must follow `$1 $2 $3` ordering.
- Use an em dash (`—`) to describe parameters.

---

# Bash Function Design Rules

## Argument Handling

Use local variables immediately:

```
my_function() {
    local arg1="$1"
    local arg2="$2"
}
```

Never access positional parameters deep inside the function body.

## Return Codes

Use standard semantics:

| Code | Meaning |
|-----|--------|
| 0 | Success |
| 1 | General failure |
| 2 | Invalid arguments |

Example:

```
[[ -d "$cache_dir" ]] || return 2
```

## Safety Practices

- Always quote variables and enclose with braces as "${VAR}".
- Use `local` for function variables
- Avoid unbounded globbing
- Redirect expected errors to `/dev/null`
- Use local if a variable no needs to be mutated. Immutable is king.

Example:

```
local cache_file
cache_file="$(ls -t "${cache_dir}"/*.json 2>/dev/null | head -n1)"
readonly cache_file
```

---

# Example: AWS Credential Cache Check

```
#----------------------------------------------------------------------
# Check AWS CLI cached credential validity.
#
# Reads the newest AWS CLI cache JSON file and verifies whether the
# temporary credentials are still valid with an optional safety margin.
#
# Args:
#   $1  cache_dir         — directory containing AWS CLI cache files
#                          (default: ~/.aws/cli/cache)
#   $2  safety_margin_sec — seconds before expiration to treat token
#                          as expired
#----------------------------------------------------------------------
aws_is_cli_cached_token_valid() {
    local cache_dir="${1:-$HOME/.aws/cli/cache}"
    local margin_sec="${2:-0}"
}
```
