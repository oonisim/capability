# Git Repository Hygiene

This file defines git repository hygiene rules for Git commits and branches.

## Spellcheck

All Bash scripts and BATS tests must pass spell‑check.

Scope:

```
shared-tool/**/*.sh
shared-tool/tests/**/*.bats
```

Recommended tool:

```
codespell shared-tool
```

## Formatting

- Use consistent indentation (2–4 spaces).
- Quote variables.
- Prefer `set -euo pipefail` for scripts.

## Test Location

Tests must mirror the implementation structure.

Example:

```
aws/authentication/_aws_session_utility.sh
→ tests/aws/authentication/test_aws_session_utility.bats
```

Create directories if they do not exist.

