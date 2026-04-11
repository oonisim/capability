# Bash Skill Architecture

This document explains how Bash standards are organized in this skill.

The directory acts as a **mini knowledge base** used by agents.

Structure:

```
skills/bash-standards/
    SKILL.md
    skill_architecture.md
    skill_implementation_knowledge.md
    skill_test_knowledge.md
    skill_repository_hygiene.md
```

Each file covers a specific category:

| File | Category |
|-----|-----------|
| skill_implementation_knowledge.md | Bash implementation standards |
| skill_test_knowledge.md | BATS testing standards |
| skill_repository_hygiene.md | Repository hygiene rules |

Agents should load the relevant document depending on the task they are
performing.

