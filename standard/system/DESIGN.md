# System Design Standard

This document defines system-level design rules for preserving clear
architecture, layering, and ownership boundaries. Code-level standards describe
how individual files and functions should be written; this standard describes
how system concepts must be placed so implementation details do not leak across
layers.

## Core Principle: Boundaries Are Architecture

A boundary is the line where one concern ends and another begins.

Examples:

- user workflow vs infrastructure transport
- UI orchestration vs frontend HTTP protocol
- application API client vs provider network workaround
- environment selection vs resource discovery
- domain behavior vs provider-specific implementation

Architecture stays clear only when each side of a boundary has a small,
explicit contract. A module may depend only on contracts permitted by the
chosen architecture. Higher-level policy must not depend on lower-level
implementation details, and lower-level details must not appear in higher-level
APIs.

## Execution Namespace Is A Boundary

An application operates inside one or more provisioned namespaces: bounded
naming contexts in which a name is visible, discoverable, and uniquely
identifiable relative to that context. Examples include a filesystem root, DNS
zone or search domain, JNDI context, database schema, cloud account or resource
scope, and service registry. A workspace, container mount, deployment root, or
repository is therefore one concrete filesystem namespace, not the definition
of namespace itself.

The namespace context is a boundary contract:

- discover or receive the context once per execution scope at its application
  boundary
- express an application-referenced resource by a name scoped to that context
- resolve names only through the boundary adapter that owns the context
- reject a name that is ambiguous or resolves outside the selected context;
  whether absence is valid belongs to the operation contract (for example,
  lookup rejects absence while create may require it)
- inject the context or resolved resource into consumers; domain and workflow
  modules must not repeat discovery or depend on ambient lookup state

A namespace may be hierarchical, but visibility of one context does not imply
authority over its parent or siblings. Host-global paths, implicit DNS search
state, default JNDI contexts, default cloud tenants, and similar ambient
coordinates must not silently become application contracts. An adapter may
materialise a provider-native identifier after resolution, but that
representation must not widen the selected namespace.

## Contracts Make Boundaries Enforceable

Architecture is the act of cutting boundaries and enforcing them with
contracts.

A contract is the formal promise that lets components communicate and interact
correctly across a boundary. Without a contract, a boundary is only a location
in the codebase; with a contract, the boundary becomes testable, reviewable,
and replaceable.

One contract must uphold one responsibility. If a contract promises multiple
unrelated responsibilities, the boundary is unclear and the contract must be
split. This is how separation of concern is enforced: each interaction contract
states one reason to change, one owner, and one stable meaning for callers.

Contracts include:

- schemas
- protocols
- interfaces
- event shapes
- API request and response models
- state snapshots
- error models
- ordering, timing, idempotency, and retry rules

Schema is a contract. It is a foundation of correct communication because it
defines what data may cross a boundary, what each field means, who owns that
meaning, and what callers may rely on.

Every boundary must declare:

- what contract crosses the boundary
- what single responsibility that contract upholds
- which component owns the contract
- which component owns each side of the interaction
- what responsibility each side has
- what implementation details must not cross
- how unknown values, invalid values, and errors are represented
- what is stable enough for callers to depend on

Components must communicate through the contract, not through each other's
internal objects, incidental fields, provider payloads, UI props, log formats,
or storage records.

## Defensive Failure: See Evil, Hear Evil, Say Evil

System engineering is predictability. Unknown, unexpected, or contract-breaking
states must be made visible immediately and must not continue into downstream
workflow.

When a component receives data that violates its contract, or when an internal
policy boundary reaches a state the design did not explicitly allow, the
component must:

- detect the condition as close as possible to the boundary where it appears
- log or emit a clear error with the owning component, operation, and failed
  contract field or invariant
- fail closed by raising or returning a terminal error state
- prevent downstream components from receiving partial, guessed, corrupted, or
  policy-bypassing data

Do not gamble on unknown state. Continuing after an unexpected condition turns
one local fault into unpredictable downstream behavior. If the system can
recover, that recovery path must be an explicit, tested contract with a named
state transition. It must not be an accidental catch-all fallback.

Best-effort behavior is allowed only for explicitly non-authoritative side
effects, such as secondary telemetry, optional cleanup, or duplicate
notifications. Best-effort paths must be documented as non-authoritative and
must not hide failure of business state, workflow routing, schema validation,
security checks, persistence correctness, or memory/policy ownership.

## Observable Execution: Tell The Request Story

Every externally invoked workflow must be traceable as one execution thread.
Operators and developers must be able to answer, from logs or events alone:

- which request or job this execution belongs to
- who or what invoked it
- which component and operation ran
- what boundary was crossed
- whether the operation started, completed, was rejected, or failed
- how long it took
- which contract, policy, or provider condition caused a rejection or failure

The correlation mechanism is a boundary contract. It must provide a stable
request, job, or operation identifier that is unique for the execution thread
and propagated through downstream components. Components must not invent
unrelated identifiers when a caller-supplied correlation identifier already
exists; they may derive child identifiers only when the parent identifier remains
logged or emitted with the child.

Log and event records at boundary crossings must carry enough structured fields
to filter the execution path without reading unrelated logs. At minimum, use
the available correlation identifier, invoking principal or service identity
when known, owning component, operation name, outcome, duration when applicable,
and sanitized error category for failures.

Logging levels must match operational meaning:

- `DEBUG`: detailed progress useful for debugging a single request path.
- `INFO`: successful boundary-crossing completion or important lifecycle state.
- `WARNING`: denied, rejected, degraded, retry-exhausted, or recoverable failure
  that changes the requested outcome.
- `ERROR`: unrecoverable failure, contract breach, data corruption risk, or
  unexpected exception that prevents the workflow from completing correctly.

Do not log secrets, raw security tokens, authorization headers, broad provider
payloads, or large user data. Log stable identifiers, operation names, contract
fields, sanitized categories, and safe provider error codes instead. The goal is
not more logs; the goal is a small, structured story that makes the correct
request path immediately filterable and understandable.

## Simplicity

- one objective
- one responsibility
- one mechanism

ask:

- What is the one objective?
- What is the one responsibility?
- What is the generic mechanism addressing?
- Are we fixing the mechanism, or addressing symptoms?

Unix Pipe:

system is linear transformation chain. pipe is the one mechanism to chain.
Think until reduce to the one mechanism that achieve the objective.

## Proportional Simplicity

Design effort must be proportional to the problem. A small requirement should
not become a broad subsystem, a new layer, or a multi-step mechanism unless the
extra structure pays for itself by reducing real risk, repeated complexity, or
future change cost.

Prefer the simplest design that keeps the correct boundary intact. If a fix can
be expressed as a small local rule without leaking concepts across layers, keep
it local. Add abstractions when they protect ownership or remove meaningful
duplication, not merely because a more general shape is possible.

## Split Early At Real Boundaries

Split modules when a real ownership boundary is visible. Do not defer the split
until the file is large.

If a module is likely to grow across multiple responsibilities, concerns, or
ownership boundaries, stop and split it before adding the new behavior. The
split must preserve the correct scope: isolate the concern that changes for one
reason, and keep it from knowing concepts owned by adjacent modules.

Refactoring later is expensive. A delayed split usually forces changes across
implementation, tests, design documents, prompts, fixtures, and public
contracts. It also encourages boundary leaks: concepts from one concern spill
into another concern because both are temporarily stored in the same file.

The safeguard is a clear module docstring. Every module that owns a system
boundary must state:

- the rationale for the module
- the responsibility and concern it owns
- the boundary it protects
- the dependencies it may import
- the dependencies it must not import
- the call or data flow through the module

A module docstring that only says where code came from is not sufficient. The
docstring must explain why the boundary exists and why the module is the right
place for that responsibility.

### Isolated Policy Decisions

When several policy rules decide the same outcome, keep one public policy
authority. Isolate each independently changing rule in its own module when that
separation provides a real plug-in/out boundary.

Use the smallest composition mechanism:

```text
public policy authority
    ├── asks ordered isolated rules: decide or decline
    └── owns the fallback outcome
```

Callers must not know which rule decided, reproduce rule ordering, or implement
fallback behavior. A rule must be addable, removable, or reorderable at the
authority without changing consumers. Keep ordering explicit and static unless
runtime discovery is an actual requirement; do not introduce registries,
decorators, or a plugin framework merely to obtain isolation.

### Boundary Comments For Non-Obvious Mechanisms

Boundary-owning code must explain non-obvious mechanisms at the point where a
future maintainer would otherwise cross the wrong boundary.

Use comments for:

- boundary rationale: why this component owns this concern
- external-standard mapping: which protocol, grammar, provider API, persisted
  schema, or policy the code mirrors
- mechanism: what the block does, why it exists, and how it works
- risk containment: why a deliberately narrow implementation is correct now
- temporary compatibility: why the path exists and when it should disappear

Do not use comments to narrate obvious control flow. Use them to prevent
knowledge spillover.

For any non-obvious block, apply the WHAT / WHY / HOW test:

```text
WHAT: What does this block protect, compute, route, or enforce?
WHY:  Why is this block needed at this boundary?
HOW:  What mechanism, rule, or algorithm does it use?
```

A comment is sufficient when a new maintainer can answer those questions
without reading unrelated modules or importing concepts from another boundary.

### Boundary-Leak Impact Test

Run this test during design review:

```text
If this module changes, how many other files may need to change?
```

Expected result:

- A well-bounded module change affects the module itself, focused tests, and
  directly owned documentation.
- A mixed-concern module change affects many unrelated files, tests, prompts,
  fixtures, and docs.

The second result is the tell-tale sign of boundary breakage. It means the file
contains too many responsibilities, and knowledge has spilled across module
boundaries.

### Module Boundary Validation

Every new module and every module split must include an explicit boundary
validation in the design review. The validation must be specific enough that a
reviewer can tell whether the module owns exactly one concern and whether it is
allowed to know about the modules it imports.

Use this checklist:

- State the design standard section the module aligns with.
- State the single concern the module owns.
- State the dependency direction: what it may import, and what must not import
  it.
- State the forbidden responsibilities that must not leak into the module.
- State whether a boundary leak was introduced.

Example:

```text
Boundary-leak check:

- Aligns with doc/standards/system/DESIGN.md:39.
- The new module owns one concern: schema-valid-but-not-renderable coverage.
- It imports schema models, but schema modules do not import it.
- It does not render Rules, mutate AST, call parser, or own schema validity.
- No new boundary leak introduced.
```

If any line cannot be answered concretely, the boundary is not ready. Do not
merge the split or new module until the ownership and dependency direction are
clear.

## Required Dependency Direction

Dependencies must follow a clear, consistent direction. Use this as the default
layer model for workflow-to-infrastructure designs:

```text
user / operator entry points
    -> UI / workflow facade
        -> application or frontend API facade
            -> transport / provider implementation
                -> infrastructure SDKs and CLIs
```

Higher layers express user intent. Lower layers implement how that intent is
fulfilled.

For ports-and-adapters designs, infrastructure adapters may depend inward on
application ports or domain contracts. Higher-level policy must still not
depend on lower-level provider implementations. See "Ports, Adapters, And
Infrastructure" below for the dependency-inversion form.

Correct:

```text
user asks to run a business workflow
    -> workflow facade resolves the required capability
        -> adapter chooses the current implementation
            -> provider helper performs infrastructure-specific work
```

Incorrect:

```text
user-facing workflow accepts provider resource IDs, internal hostnames,
transport modes, local ports, and provider regions
```

The incorrect shape exposes infrastructure implementation as if it were user
workflow.

### Layer Vocabulary

Different systems may use different names for these layers. The dependency rule
is the same: user intent enters through workflow/application code, and provider
implementation details stay behind adapters.

Canonical mapping:

```text
user / operator entry points
    same role as: interfaces, CLI, API handlers, event handlers

UI / workflow facade
    same role as: application facade, use-case boundary, workflow API

application or frontend API facade
    same role as: port-facing client, SDK facade, application service

transport / provider implementation
    same role as: adapter, gateway, repository implementation

infrastructure SDKs and CLIs
    same role as: cloud provider, database, filesystem, vendor, shell command
```

Example implementation mapping:

```text
CLI, API handler, scheduled job, or UI event
    -> workflow facade
        -> application service or port-facing client
            -> adapter or gateway
                -> provider SDK, database driver, shell command, or remote API
```

## Leakage

Leakage is when a lower-layer concept appears in a higher-layer API, data
model, configuration file, command-line flag, test fixture, or docstring.

Leakage is not only a naming issue. It is architectural damage: it moves
knowledge to the wrong place, makes the caller responsible for details it cannot
own, and spreads temporary implementation constraints across the system.

### Common Leakage Symptoms

Treat these as design smells that require refactoring:

- User-facing commands require provider resource IDs, ports, internal
  hostnames, queue names, storage locations, or provider regions.
  This pollutes user intent with deployment topology.
- A higher layer has flags such as `--tunnel`, `--direct`, `--skip-tls`, or
  `--local-port` for normal operation.
  This makes temporary transport mechanics part of the user contract.
- A caller passes two names for the same logical target, such as public TLS host
  and internal service hostname.
  This makes the caller know split-horizon or network implementation details.
- A UI or workflow module imports provider SDKs directly.
  This couples workflow semantics to a provider implementation.
- A canary imports a lower-level frontend or transport client directly instead
  of the UI facade.
  This bypasses the layer that owns workflow semantics.
- A temporary workaround appears in many modules.
  This turns temporary infrastructure state into permanent architecture.
- Tests assert provider-specific call details from user-facing modules.
  This freezes the wrong boundary and makes future architecture changes
  expensive.
- Documentation tells users how to choose transport mechanism rather than target
  environment or use case.
  This exposes internal topology and trains users to depend on it.

### Generic Examples

The rule applies across domains, not only infrastructure access:

```text
payment workflow
    -> payment port
        -> Stripe, PayPal, bank-transfer, or internal-ledger adapter
```

The order workflow should ask for "take payment", not know card network,
merchant account, API key, retry gateway, or settlement provider details.

```text
document ingestion
    -> ingestion service
        -> object-store, local-filesystem, vendor-document-API adapter
```

The consuming logic should ask for documents, not know bucket names, filesystem
paths, presigned URLs, or vendor SDK clients.

```text
notification use case
    -> notification port
        -> email, SMS, push, SES, SendGrid, or internal-message adapter
```

The use case should ask to notify a recipient, not know SMTP hosts, provider
regions, template storage, or vendor-specific delivery status fields.

### Example: Infrastructure Access Leakage

Problem shape:

```text
run-workflow --env dev
    -> passes provider instance ID
    -> passes internal service hostname
    -> passes TLS hostname
    -> passes local tunnel port
    -> passes provider region
```

This leaks network topology into the user-facing workflow. The user does not
want "route through this temporary transport workaround"; the user wants "run
the dev workflow".

Correct shape:

```text
run-workflow --env dev
    -> passes environment dev
    -> workflow facade resolves current access method
```

The temporary connection method is isolated behind the workflow boundary:

```text
access.py
    public user-facing workflow API

connection.py
    private connection policy and infrastructure metadata

provider_tunnel.py
    provider-specific tunnel implementation
```

When network access changes, only the connection policy should change. The user
command, canary API, and workflow code should not.

## Ownership Rules

### User and Operator Entry Points

Entry points express user intent only.

Allowed:

```text
--env dev
--workflow health-check
--data course.json
```

Not allowed in user-facing workflows:

```text
--provider-resource-id
--internal-hostname
--transport-target
--local-port
--skip-provider-security
--provider-region-for-transport
```

The only exception is an explicit debug or break-glass path. Such flags must be
documented as debug/deprecated, must not be required for normal operation, and
must not be used by the default workflow.

### UI / Workflow Facade

The UI or workflow facade owns user-facing workflow concepts and hides transport
implementation.

It may expose:

```python
WorkflowRequest(environment="dev")
WorkflowClient
open_workflow(request)
```

It must not expose:

```python
ProviderClient
provider instance ID
internal transport target
public/internal hostname split
local tunnel port
provider transport-security bypasses
low-level provider security mechanics
```

If the UI layer needs a lower-level client, it wraps it in a UI-owned type. This
prevents user code from skipping the UI layer and coupling directly to the
frontend or transport layer.

### Connection Policy Module

Temporary infrastructure decisions belong in one isolated place.

Recommended pattern:

```text
access.py
    public API: WorkflowRequest, WorkflowClient, open_workflow

connection.py
    private policy: resolve environment to current connection method

provider_tunnel.py
    provider implementation: starts and stops the temporary tunnel
```

The connection module may know provider-specific details because that is its
purpose. Higher layers must not.

### Lower-Level Client Modules

Lower-level clients own protocol and transport details. They are not user
workflow APIs.

Example:

```text
provider_sdk.py owns ProviderClient.
workflow_access.py owns WorkflowClient.
```

User-side code should depend on `WorkflowClient`, not `ProviderClient`, even if
`WorkflowClient` delegates internally to `ProviderClient`.

## Dependency Visibility

Knowledge creates dependency. A module that knows how to find, configure, or
construct another resource is dependent on that lookup mechanism, even when the
dependency is hidden behind a string, environment variable, DNS name, SDK call,
or service locator.

Good design makes dependencies visible in the structure of the code:

```python
class CourseProcessor:
    def __init__(self, repository: CourseRepository) -> None:
        self._repository = repository
```

Poor design hides dependencies inside the implementation:

```python
class CourseProcessor:
    def process(self) -> None:
        repository = lookup("course-repository")
```

The first shape tells reviewers, tests, static analysis, and callers what the
module needs. The second shape delays failure until runtime and makes the
dependency graph difficult to trace.

### Dependency Injection Over Lookup

Application and domain code declare what resources they need. They must not
decide how those resources are acquired. This applies to services, clients,
repositories, clocks, configuration, secrets, and any other resource required to
run the module.

Allowed:

```text
module declares: needs Repository, Clock, Config, WorkflowClient
composition root provides: concrete repository, clock, config, workflow client
```

Not allowed:

```text
module reads environment variables directly
module reads parameter store or secret manager values directly
module opens cloud SDK clients directly
module performs DNS, service-discovery, or provider lookup directly
module constructs infrastructure clients inside business logic
```

Lookup is not inherently wrong. DNS, service discovery, parameter stores, secret
managers, and directories are necessary at system boundaries. The rule is about
placement:

```text
boundary / composition root
    -> lookup, validate, construct
    -> inject typed resources
application / domain
    -> use injected resources only
```

This keeps "finding" separate from "using". It also keeps temporary
infrastructure constraints from becoming permanent application contracts.
For credentials and temporary sessions, `SECURITY.md` applies this rule to
acquisition authority, expiry, and renewal.

### Configuration Injection

Configuration, including policy, must be isolated from the code that applies
it. Configuration declares values and selections; mechanism consumes those
inputs without embedding copies of them.

For declarative infrastructure, keep the same boundary:

```text
variables / tfvars
    -> configuration values and selection patterns
locals
    -> derive and wire values
resources
    -> apply the derived values
```

Do not hide configuration values or concrete selection lists in `locals` or
resource definitions. Conversely, do not expose wiring expressions as caller
configuration. A policy change should edit its configuration owner; the
mechanism should remain unchanged.

### Configuration As Input

Configuration is a dependency, so the dependency injection rule applies to it.
Treat configuration as input to the system, not as a global API.

Correct shape:

```text
environment / files / parameter store / secret manager
    -> config loader
        -> typed, validated config object
            -> injected into modules
```

Incorrect shape:

```text
application code
    -> reads environment variables wherever needed
    -> parses strings locally
    -> applies local defaults without central ownership
```

Rules:

- Read environment variables, parameter store values, and secrets at the edge.
- Convert raw values into typed, validated configuration before use.
- Fail fast when required configuration is missing or malformed.
- Keep a single owner for each configuration value.
- Do not log secrets or broad config objects that may contain secrets.

The goal is not to eliminate environment variables. The goal is to prevent
uncontrolled global string dependencies.

### Composition Root

Every runtime should have a clear composition root: the place where external
resources are discovered, configuration is loaded, concrete implementations are
selected, and dependencies are injected.

Examples:

- `main.py`
- a CLI entry point
- a Lambda handler bootstrap
- a FastAPI application factory
- a canary bootstrap
- a test fixture that wires fake dependencies

Only the composition root should decide which implementation to use for a port
or interface. Business modules should not know whether a resource comes from a
cloud provider, local disk, a mock, a database, or an HTTP service.

Composition roots may import concrete lower-level implementations to wire
dependencies. They must not expose those implementation types through
user-facing contracts.

Allowed in a composition root:

```text
parse user intent
load raw configuration
perform service discovery or lookup
validate and type configuration
construct concrete implementations
inject dependencies into application code
```

Forbidden outside a composition root or boundary adapter:

```text
read raw environment variables
perform provider lookup
choose concrete infrastructure implementations
construct provider SDK clients
switch behavior based on deployment topology
```

### Ports, Adapters, And Infrastructure

Use ports and adapters when a module needs an external capability.

```text
domain
    pure business rules

application
    use cases and ports that describe required capabilities

interfaces
    CLI, API, event, and user-facing adapters

infrastructure
    cloud providers, databases, filesystems, vendor SDKs, and network clients
```

Dependency direction:

The arrows below describe source-code dependency direction, not runtime data
flow. Infrastructure adapters implement application ports or domain contracts.

```text
interfaces -> application -> domain
infrastructure -> application ports / domain contracts
```

Forbidden:

```text
domain -> infrastructure
application -> provider SDK
application -> interface framework
infrastructure -> user workflow facade
```

This preserves replaceability. A cloud provider, framework, database, model
vendor, or transport mechanism can change without rewriting business rules.

### Framework And Tool Boundaries

Frameworks and tools are implementation choices. They must not force
application or domain code to know framework internals unless the framework is
itself the product boundary.

Design review must ask whether a tool introduces concepts that leak across
layers. If it does, isolate those concepts behind an adapter or record the
decision in an ADR before making them part of the public contract.

## Implementation And Testability

System design decisions must be enforceable in code and tests. A design is not
complete when it only describes boundaries; it must also define how those
boundaries are implemented, verified, and protected from regression.

Design documents and refactor plans must include an implementation section that
states which modules, scripts, adapters, and tests enforce the boundary.

### Implementation Requirements

Required:

- Public modules, types, and functions have names that match their ownership.
- Modules are split early when new behavior crosses a responsibility, concern,
  or ownership boundary.
- Boundary-owning modules have module docstrings that explain rationale,
  responsibility, protected boundary, dependency rules, and call/data flow.
- Non-obvious boundary mechanisms have comments that explain what they do, why
  they are placed there, and how they work.
- Entrypoints parse input and wire dependencies only.
- Business logic receives dependencies through constructors, arguments, request
  objects, or ports.
- Environment, provider SDK, filesystem, network, and config lookup stay in the
  module that owns that boundary.
- Modules stay small enough that ownership and control flow are reviewable.
- Provider-specific client construction stays behind an adapter or composition
  root.

Forbidden:

```text
domain or workflow code reading os.environ directly
domain or workflow code constructing provider SDK clients directly
user-facing code importing lower-level implementation clients directly
large mixed-concern modules that cross entrypoint, config, transport, and logic
module docstrings that describe history but not ownership, boundary, and flow
```

### Test Requirements

Every system design change must include tests that prove the boundary and
prevent regression.

Required:

- Tests verify public contracts at the layer that owns the contract.
- Boundary tests cover rejected leaked fields, invalid targets, missing config,
  and direct imports that must remain forbidden.
- Unit tests mock external providers, remote APIs, databases, and shell calls.
- Integration or live tests are separated and explicitly marked.
- Static guard tests prevent user-facing modules from importing lower-level
  implementation clients or provider-specific SDKs.
- Boundary values are tested for numeric and enum inputs.

Composition roots may be excluded from forbidden-import guard rules when the
imports are used only to wire dependencies and the concrete types do not escape
through public contracts.

Test ownership follows the same layer boundaries as implementation:

```text
domain/application unit tests
    assert business behavior without provider SDKs or live infrastructure

UI/workflow facade tests
    assert public request/client contracts and hidden transport details

connection/adapter tests
    assert direct-vs-provider policy using mocked provider helpers

infrastructure tests
    assert provider integration behind adapters, separated from unit tests

entrypoint/acceptance tests
    assert user-visible commands, flags, exit codes, and output

static guard tests
    assert forbidden imports, leaked fields, and forbidden user-facing flags
```

Tests must not skip a facade just to make setup easier. If user code is only
allowed to call a workflow-owned client, tests for user-side workflows must
also go through that workflow-owned type.

Forbidden:

```text
tests that assert lower-level provider calls from user-facing modules
tests that require live infrastructure for unit coverage
large volumes of tests before the test fixture/import harness works
```

### Required Verification

Design plans must list the verification commands that prove compliance. The
exact commands depend on the changed files, but the plan must cover:

```text
language syntax and lint checks
focused unit tests
entrypoint or wrapper checks
static guard tests for forbidden imports, flags, and leaked fields
```

If a verification step cannot run, the final change report must state why and
what residual risk remains.

## Design Review Checklist

Use this checklist while reviewing a proposed design or refactor plan. It is
diagnostic: any "no" answer identifies work to do before implementation or
merge approval.

1. Can the user-facing API be described using user concepts only?
2. Does any user-facing function, command, or config mention infrastructure
   IDs, ports, internal names, provider SDKs, or temporary workarounds?
3. Is a temporary workaround isolated in one module that can be deleted or
   changed later without touching user-facing callers?
4. Does each layer depend only on contracts allowed by the chosen architecture,
   without higher-level policy depending on lower-level implementation details?
5. Is any caller skipping a facade and importing a lower-level client directly?
6. Is the public type owned by the layer that exposes it?
7. Would a future infrastructure change require edits in user-facing scripts,
   docs, or workflow entry points? If yes, the boundary is leaking.
8. Do tests assert behavior at the correct layer, or are they freezing
   lower-level implementation details into high-level modules?
9. Are dependencies visible in constructors, function arguments, request
   objects, or ports rather than hidden behind lookup strings?
10. Is raw configuration read only at the edge and converted into typed,
    validated objects before use?
11. Is there one composition root that resolves resources and injects them into
    the application?
12. Are framework, platform, or tool concepts isolated behind adapters unless
    they are intentionally part of the public contract?
13. Does the implementation plan identify the modules, scripts, adapters, and
    tests that enforce the boundary?
14. Are verification commands listed for linting, syntax checks, focused tests,
    and static guard tests?
15. If a changed module must change, how many other files are likely to change
    with it? If the answer crosses unrelated tests, prompts, fixtures, docs, or
    modules, the boundary is leaking.
16. Does each boundary-owning module docstring state its rationale,
    responsibility, protected boundary, allowed dependencies, forbidden
    dependencies, and call/data flow?

## Refactoring Rule

When leakage is found, do not simply rename flags or variables. Move ownership
to the correct layer.

Refactoring pattern:

Apply these steps in order unless a smaller compatibility step is explicitly
documented.

1. Define the user intent as a small request object.
2. Define a layer-owned client/facade returned to callers.
3. Move provider or topology decisions into a private connection or adapter
   module.
4. Keep lower-level client types private to the adapting layer.
5. Add guard tests so higher layers cannot re-import leaked lower-layer types.
6. Replace internal lookup with constructor or argument injection.
7. Move environment/config reads to the composition root or config loader.
8. Update implementation and test code to comply with applicable code standards.
9. Add verification commands to the plan and run the focused checks.

## Acceptance Criteria For Clear Layering

Use these criteria for merge or sign-off after implementation and verification.
A design preserves boundaries when:

- User-facing APIs talk in user concepts.
- Infrastructure details are private to infrastructure or connection modules.
- Temporary workarounds have one owner.
- Public client/request types are owned by the layer exposing them.
- Higher layers do not import lower-layer implementation clients directly.
- Changing deployment topology does not require changing user workflows.
- Dependencies are visible in type signatures or explicit request objects.
- Lookup is confined to system boundaries and composition roots.
- Configuration is typed, validated, and injected.
- Applicable implementation and test standards are followed.
- Tests prove the boundary with measurable assertions.
- Verification commands have been run or explicitly reported as not run.

## Simplification Guardrails

Boundary discipline is not permission to add layers for their own sake. The
goal is to reduce accidental complexity while keeping ownership clear.

Simplification means:

- Condense: remove accidental complexity.
- Name: define the right minimal concepts.
- Compose: make concepts combine cleanly.

The bar is whether the design reduces complexity, makes sound tradeoffs, and
leaves the system easier to evolve.

Prefer the smallest design that preserves the boundary:

- Add a facade only when callers would otherwise learn lower-level details.
- Add a type only when it names a real concept or removes invalid states.
- Add an adapter only when the provider can change or the provider details leak.
- Add a config object only when raw config would otherwise spread.
- Add guard tests only for boundaries that are easy to regress.

Avoid speculative structure:

- Do not create public wrapper types before a caller needs to name them.
- Do not add optional fields plus complex validation when two simpler private
  types remove invalid states.
- Do not introduce a reusable helper before there are multiple real users.
- Do not keep compatibility paths without a removal condition.
- Do not make debug or break-glass paths part of the normal API.

Review question:

```text
Does this change make the system easier to reason about, test, and evolve, or
does it only add structure?
```

If the answer is unclear, choose the smaller change and document the boundary
that must not be crossed.


---

# Notes

## Boundary Boundary Boundary

Leakage cause architecture collapse. Leaks are:

- Knowledge 
- Concern to address
- Responsibility

Boundary is the mechanism to isolate, contain, seal the leakages.

### Finding Boundary

When the user-facing concern is: Run the use-case workflow against environment dev or prd.

Then the concerns to focus by the implementation is:

* use case
* environment

How to connect and technical details should not be the concerns. Handling them are eroding
the boundaries.


## Optimal 

O(n) or at most O(n * log(n)) is enforced. Finding violation later will cost a lot.




Boundary,Containment, Facade (to prevent spill over, single point of contact).
How important to be able to contain manageable system.

Note: the state-machine service should be the transition authority. Use-case
code may define states, events, terminal states, and transition rules, but the
state-machine service owns transition evaluation and returns the accepted or
rejected transition result. Graph libraries such as LangGraph can be service
implementation providers behind that contract; they are not limited to agentic
AI use cases.

---

# Notes

## Thought Process

Interface <- Contract <- Boundary <- Responsibility

Clear-cut Boundary drives correct steps -> Decomposition, Device and Concur module

## Monitoring - Liveness vs. Readiness vs. Startup

You should leverage the distinct purposes of health probes to manage the complexity:
* Startup: Use this for the initial stagger. It prevents the container from being killed while workers are sequentially spinning up and performing initial LLM/VDB connectivity checks.
* Liveness: Check process presence. Use a shell-based aggregator like the healthcheck.sh example (checking pgrep or a supervisor's status command). 
* Readiness: Check functional health. It verifies not just that the processes are running, but that they are responsive and able to fulfill the contract.

The "Best Practice" extension is to bubble those internal status signals up to the supervisor.
```
┌─────────────────┬──────────────────────────┬────────────────────────────────────────────────────────────────────────┐
│ Feature         │ Single Process           │ Multi-Process                                                          │
├─────────────────┼──────────────────────────┼────────────────────────────────────────────────────────────────────────┤
│ Liveness Source │ Internal AsyncIO loop    │ Aggregated Heartbeats from Children                                    │
│ Health Port     │ Direct to App            │ Bound to Parent                                                        │
│ Failure Scope   │ Container Restart        │ Supervisor can restart individual children                             │
└─────────────────┴──────────────────────────┴────────────────────────────────────────────────────────────────────────┘
```

## Failure Modes

Every design should have failure modes considered.
What can go wrong, where, when, how, probability, severity
-> Risk Assessment & Priority.

Estimate of footprints -> Capacity Planning -> Non-Functional Requirement coverage.


## Reduce Review Fix Cycles

Ask: list all the decision choices and ambiguities with 
* pros and cons, 
* effort required

Prefer to 
* simplicity
* maintainability
* future-proof

### What can cause endless Review/Fix cycles

Ambiguity, Choices, Opinions, and No decisions.
How to solve:

Ask the questions to narrow down, 
reveal the evil cycle, 
overengineering 
- is it fit for purpose? 
- without it, what is the consequence? 
- Cause Technical debts
- Damage future-proof?
- ROI? (effort required and benefit achieved)
- Pros and Cons

* Scrum principle of identify "NOT TO DO" first, which narrows down the search/combination space.
* Decide, Try, Fail fast, learn fast, and move on <- Just do it instead of for-ever contemplating
* Reduce, Remove, Resize

However, handle the issues at the right timing. 
Technical debts and future complexity will just grow if not addressed now, then handle it now. 

* Shift Left
* Front Loading

Tackle the complex issue, future-proof, robust, scalability, observability issues now. NOT later.

### 5/95 ROI Rule

5/95 rule may exist. There can be a simpler option A than B which delivers
95% of the same value with 5% of the effort of B. Take the option A.

Concrete example: the goal was to scale out agent jobs.

Option A:
Run more containers with one worker in each. Existing container management already handled
worker isolation, health, and restart. This delivered 95% of the scaling value with about 5%
of the effort of the option B.

Option B:
Run several workers inside each container. This required custom process supervision,
per-worker health checks, and coordinated shutdown and restart.
The small 5% extra value did not justify the much 95% larger investment unless container count
or cost became a proven constraint.

# Responsibility Driven Architecture

Responsibility/Concern -> Boundary -> Contract -> Interface



A useful principle from Brooks, Lampson, Kay, and others is:

Good systems reveal their conceptual model.
Bad systems reveal their implementation.


---
# Lessons Learned


## No Patch Work but Hit the Root Solution

If we start patching issues to workaround case by case, it will result in messy Frankenstein patch works.
Think like Ken Thompson. Think deeper to condense the idea and come up with concise hit-the-root-cause right solution.

Look for opportunities to simplify and reduce redundancy. The goal is not
reduction for its own sake; the goal is condensation without losing function or
information.

80/20 rule. Find less than 20% can cover more than 80% solution.

Ken Thompson approach.
Reduce and condense the ideas to the essences, identify a simple but genric core mechanism to address the issues.

Verify the solution is what Ken would do. If not, think harder and deeper.


## Regression Blocking

Keep track of the fix of an issue and its cause/effect mechanism, the way to catch it and prevent from being reintroduced.
Run the prevention tests every change.


## Lessons Learned: M2 Boundary Failure

M2 failed because ownership was replaced by convenience: when a module needed a
concept, it imported an outer, downstream, provider, or compatibility object
instead of defining the contract owned by its own boundary. That turns an
implementation detail into an API, spreads knowledge sideways, and guarantees
spaghetti dependencies. The rule is simple: a boundary module consumes only the
explicit upstream contracts named by the design, owns the primitives it exposes,
and never imports provider internals, compatibility DTOs, old topology/layout
readers, scanners, rule builders, or downstream modules to fill a local gap.
When such leakage appears, do not wrap or rename the leaked object; move the
concept to the boundary that owns it and add a static import guard so it cannot
return.
