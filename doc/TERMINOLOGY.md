# Terminology & Naming Conventions

## Script Naming by Purpose

| Term | Convention | Purpose |
|------|-----------|---------|
| `verify_*` | Deployment validation | "Was this resource deployed correctly?" — one-shot, post-deploy checks to confirm resources match expected state |
| `monitor_*` | Observability | "Is this system behaving to contract right now?" — ongoing operational checks to verify the system complies with its contract |

### Distinctions

- **`verify_*`** scripts run after deployment to validate infrastructure and configuration are correct
- **`monitor_*`** scripts run continuously to observe system behavior and contract compliance at runtime
