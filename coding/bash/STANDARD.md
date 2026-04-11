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

# Naming Conventions

## Sourced Files (prefixed with `_`)

Files that are sourced (not executed directly) **must** be prefixed with
`_<service>_` or `_<abbreviation>_` for namespace safety. This prevents
function name collisions when multiple libraries are sourced into the
same shell session.

| Service          | Prefix           | File example               | Function example                      |
|------------------|------------------|----------------------------|---------------------------------------|
| EC2              | `_aws_ec2_`      | `_aws_ec2.sh`              | `_aws_ec2_get_lifecycle_state`        |
| SSM              | `_aws_ssm_`      | `_aws_ssm.sh`              | `_aws_ssm_command_run`                |
| Secrets Manager  | `_aws_sm_`       | `_aws_sm_rotate_secret.sh` | `_aws_sm_read_secret`, `_sm_rotate`   |
| S3               | `_aws_s3_`       | `_aws_s3_common.sh`        | `_aws_s3_bucket_exists`               |
| Bedrock          | `_aws_bedrock_`  | `_aws_bedrock_api.sh`      | `_aws_bedrock_get_apikey`             |
| Session mgmt     | `_aws_session_`  | `_aws_session_utility.sh`  | `_aws_session_is_role_matched`        |
| Profile mgmt     | `_aws_profile_`  | (same file)                | `_aws_profile_is_token_valid`         |

Rules:

1. **Filename**: `_<vedor/product>_<service>_<purpose>.sh` — the service prefix makes the
   file's domain immediately obvious in `source` statements.
2. **Functions**: All functions in the file must share the same
   `_<service>_` prefix (or `_aws_<service>_` for AWS-specific utilities).
   This prevents collisions when sourced alongside other libraries.
3. **Internal helpers**: Nested or file-private functions should also use
   the service prefix: `_bedrock_load_key_file`, not `_load_key_file`.
4. **No bare names**: Never define a sourced function without a namespace
   prefix (e.g. `check_bucket_exists` is wrong; `_s3_bucket_exists` is
   correct).

## Executed Scripts (no `_` prefix)

Scripts that are executed directly (not sourced) use descriptive names
without the `_` prefix: `aws_login_monash.sh`, `empty_bucket.sh`,
`bedrock_monitor_models.sh`.

Functions inside executed scripts may use any naming scheme since they
are not exposed to other scripts' namespaces.

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
_aws_session_is_cli_cache_valid() {
    local cache_dir="${1:-$HOME/.aws/cli/cache}"
    local margin_sec="${2:-0}"
}
```

---

# When to Create a Utility Function

Create a reusable function when:

- Logic appears in **2+ scripts**
- Logic interacts with **external systems** (AWS, Azure, Git, etc.)
- Logic requires **structured error handling**
- Logic is **non‑trivial** (>10 lines)

---

# Repository Usage Pattern

Shared utilities should live in:

```
aws/authentication/_aws_session_utility.sh
```

Scripts should load utilities using:

```
source "$(dirname "$0")/_aws_session_utility.sh"
```

---

# Summary

All Bash utilities must:

- include standardized documentation
- behave like pure functions
- accept configuration via arguments
- avoid global side effects
- use clear exit codes

Following these rules keeps the SMST tooling predictable, maintainable,
and safe to reuse across projects.
