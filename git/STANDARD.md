# Bash Repository Hygiene

This file defines repository‑wide hygiene rules for Bash code and tests.

## Spellcheck

All Bash scripts and BATS tests must pass spell‑check.

Scope:

```
smst-tool/**/*.sh
smst-tool/tests/**/*.bats
```

Recommended tool:

```
codespell smst-tool
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

