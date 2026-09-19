# Platform Observability Implementation

## Observability

Observability is the ability to reconstruct the story of an application
execution from evidence.

Every application and component must emit enough telemetry to make each
execution visible and traceable from start to finish. The primary evidence is
structured logs, supported by platform telemetry such as infrastructure audit
events, network flow records, health signals, and metrics.

For each execution, operators must be able to answer:

1. What execution is this? Its correlation ID, request ID, job ID, trace ID, or
   other unique execution identity.
2. Who or what initiated it? The user, service, scheduler, deployment, or
   external system that started the execution.
3. When did it happen? Its start time, end time, duration, and ordered state
   changes.
4. Where did it run? The application, component, host, container, function,
   region, or environment involved.
5. What happened? The meaningful lifecycle events, decisions, failures, retries,
   side effects, and final outcome.

## Responsibility and boundary

Observability documentation is layered so each document stays at the right level
of abstraction. This document defines the top-level system observability
direction. It establishes the principles that guide observability design from
system architecture through platform and IaC, application observability, and
code implementation.

| Layer | Responsibility |
| --- | --- |
| System observability direction | Defines what must be observable and why. Establishes the evidence contract. |
| Platform and IaC observability | Defines how the platform collects, stores, protects, searches, and exposes telemetry. |
| Application observability | Defines what the application emits, how its lifecycle is monitored, and how operators read its signals. |
| Code implementation | Defines concrete logging, metrics, tracing, health-check, and error-handling implementation. |

Lower layers should be checked against this document: does the implementation
preserve the evidence contract, converge into a single operational view, and
keep lifecycle concerns separate?

## Principles

### Evidence trail

Every execution must leave enough facts to reconstruct what happened. Evidence
is the raw material of observability.

### Navigable evidence

Evidence must be structured so operators can find, filter, correlate, and
understand it. Logs and telemetry should be separated by application, component,
environment, and execution identity.

### Single observability view

Structured evidence must converge into one operational view across the stack.
Operators must be able to reconstruct routine activity, failures, state
transitions, and execution outcomes from centralised logs and related telemetry.
They should not need to SSH into a host or inspect a container as the normal way
to understand what happened.

Applications should emit structured logs to stdout and stderr. The platform is
responsible for collecting, retaining, protecting, and exposing those records
through the stack's observability backend.

### Lifecycle separation

Lifecycle separation structures evidence by system state over time: startup,
liveness, readiness, execution, failure, and shutdown. This prevents one signal
from being overloaded and helps operators understand which phase a component is
in.

### Responsibility layering

Responsibility layering places each observability rule at the correct abstraction
level: system, platform and IaC, application, or code. It lets teams identify
where a requirement belongs and who owns it.

## Platform evidence destinations

Log groups are a platform implementation of the single-view and navigable-
evidence principles. IaC, such as Terraform, must create application log groups
before the application stack starts. Runtime configuration should set
`awslogs-create-group: "false"`, or the equivalent, so a missing log group fails
visibly instead of being created with ad hoc retention or encryption settings.

Platform implementation documents must define:

- deterministic telemetry-destination naming by application and component;
- retention and encryption policy;
- stream or partition structure;
- failure behaviour when a required destination is missing; and
- where each application documents its concrete destinations.

## Lifecycle

A deployed component has a lifecycle. Probe meaning depends on the component's
current lifecycle phase, so startup, liveness, and readiness must remain
separate concerns.

| Lifecycle phase | What it means | Why it exists |
| --- | --- | --- |
| Startup | The runtime container has started and is initialising the system. It may load configuration, start workers, bind health surfaces, and perform initial dependency handshakes. | Gives the component time to become observable before liveness enforcement can terminate it. It protects intentionally slow starts, such as worker staggering and initial dependency checks. |
| Liveness | The component is alive inside the runtime container. Its process tree exists and its internal execution loop or supervisor is making basic progress. | Answers whether the component should be restarted. It does not prove that the component can serve real work. |
| Readiness | The component can fulfil its application contract. Required dependencies, constraints, and prerequisites are satisfied. | Controls whether work or traffic should be sent to the component. A component can be live but not ready. |

Readiness and liveness must not be collapsed unless the application document
explicitly states why the same signal is sufficient for both concerns. Each
application should document its concrete probe endpoints, commands, thresholds,
and failure actions in its own observability document.

```text
/startup   process has booted enough to expose health HTTP
/live      process or event loop is alive; do not check slow dependencies
/ready     full application contract is ready; dependencies and workers are warm
```

## Application logging

### Architecture

Application containers should use the standard application logging framework for
their runtime, emit structured single-line JSON to stdout and stderr, and rely
on the container runtime to ship those records to CloudWatch Logs.

```text
application log call
        |
        v
application logger
        |
        v
JSON formatter
        |
        v
single-line JSON on stdout or stderr
        |
        v
container log driver
        |
        v
CloudWatch Logs
```

The application should have no direct CloudWatch dependency for ordinary
container logs. The runtime captures stdout and stderr and ships them.

### Log levels

| Level | When to use | Example |
| --- | --- | --- |
| `ERROR` | An unexpected failure from which the system cannot recover automatically. | Dependency unavailable after retries; message parse failure. |
| `WARNING` | A recoverable condition or unexpected state worth noting. | Retry attempt before success; deprecated configuration value detected. |
| `INFO` | Normal operational milestones. | Request received; job completed; record stored. |
| `DEBUG` | Internal state useful for focused troubleshooting. | Request payload details; provider parameters. |

The default log level should be `INFO`. `DEBUG` should not be enabled in
production by default because it can produce high log volume and may include
request payload content.

### Baseline structured fields

Every JSON application log record should contain these baseline fields:

| Field | Always present | Description |
| --- | --- | --- |
| `timestamp` | Yes | ISO 8601 timestamp with timezone offset. |
| `level` | Yes | `DEBUG`, `INFO`, `WARNING`, `ERROR`, or `CRITICAL`. |
| `logger` | Yes | Logger name, usually the full module or component path. |
| `message` | Yes | Human-readable event description. |
| `process` | Yes | Operating-system process ID, when available. |
| `processName` | Yes | Runtime process name, when available. |
| `exc_info` | On exceptions | Full traceback string or equivalent exception detail. |

Application-specific context fields, correlation IDs, and query patterns belong
in the application observability document.

## Dashboards

Shared observability modules may create CloudWatch dashboards for EC2 health,
log activity, and network reject counts. The platform owns dashboard plumbing
and module behaviour; applications own the interpretation of application-specific
widgets and alerts.

## Log-level configuration

The platform may pass log-level configuration through deployment configuration
and container environment variables, but each application owns:

- valid component-level log-level keys;
- default values;
- whether debug logging may include request payloads; and
- the redeploy or restart process required for changes to take effect.
