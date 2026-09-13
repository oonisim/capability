# Security Standard

## Credential and Session Lifecycle

Permanent credentials and temporary session material are different security
objects. They must have separate owners, lifecycles, and interfaces.

### Permanent credentials

Long-lasting passwords, API keys, client secrets, and private keys must:

- reside in a managed secret service protected by encryption, authentication,
  authorization, audit logging, and rotation;
- never be stored in source code, local credential files, shared filesystems,
  command arguments, application configuration, logs, traces, or databases;
- be resolved only inside a downstream authentication provider authorized for
  the exact secret;
- remain contained within that provider while it exchanges the credential for
  temporary session material;
- have provider-owned references released immediately after the exchange, on
  both success and failure;
- never cross the application-user contract or appear in returned dictionaries,
  models, exceptions, or diagnostic representations.

The managed-secret technology is an infrastructure responsibility. Reusable
domain and application libraries must not know its vendor, resource name, ARN,
path, SDK, or retrieval protocol. The downstream composition root constructs
and injects the provider.

### Temporary sessions

After successful authentication, the provider may return a narrowly scoped,
short-lived session containing only what the downstream user needs, such as an
endpoint and bearer token. The session must:

- exclude every permanent credential field;
- redact tokens from `repr`, logs, traces, and exceptions;
- use TLS for every network operation;
- carry the minimum permissions and lifetime supported by the external system;
- be invalidated or discarded when expired, rejected, or no longer needed;
- never be persisted unless the protocol explicitly requires a protected
  session store with an expiry.

The lifecycle is:

```text
managed secret service
        ↓ authorized provider retrieval
contained permanent credential
        ↓ immediate TLS authentication exchange
release permanent credential references
        ↓
temporary session
        ↓ injected application user
discard or invalidate at expiry/end of use
```

### Contract boundary

The reusable contract is a session source, not a credential loader:

```python
class SessionSource(Protocol):
    def acquire(self) -> Session: ...
```

This one boundary allows managed-secret implementations to change without
exposing permanent credentials or secret-management technology to users.

### Session acquisition is privileged composition

Acquiring or renewing a session is a security authority distinct from using an
already issued session. Apply the dependency-injection rule from `DESIGN.md`:

```text
executable composition root
        ↓ construct the one approved SessionSource adapter
authentication provider
        ↓ acquire a narrowly scoped temporary Session
reusable transport/workflow
        ↓ use the injected Session only
```

Rules:

- Only an executable composition root or its authentication adapter may locate
  a provider, call `SessionSource.acquire()`, or invoke a token endpoint.
- Reusable application, domain, transport, and workflow functions receive the
  typed session explicitly through an argument or constructor. They must not
  import a provider factory, call `get_session_token()`-style helpers, or repeat
  credential discovery.
- Inject a `Session`, not a `SessionSource`, unless the consumer explicitly owns
  a reviewed renewal responsibility. A source grants authority to mint or renew
  credentials; most consumers need only the narrower authority of one session.
- Independent executables may each have their own composition root. Repetition
  across those roots is not hidden lookup; acquisition inside reusable modules is.
- Enforce the acquisition allowlist with dependency/AST tests so new call sites
  cannot silently widen credential access.

This pattern does not reduce the permissions carried by a bearer token if that
token itself is compromised. It reduces the number of components authorized to
obtain or renew tokens, makes those privileged paths reviewable, and limits the
code and diagnostics through which credential material can propagate.

Expiration has two permitted mechanisms; choose one explicitly:

1. Fail closed on expiry/rejection, discard the session, and rerun the
   executable root so it acquires a new session.
2. Inject a provider-opaque renewal callback or port with the session. The
   consumer may request renewal but must not know credential storage, token
   endpoints, or provider classes. Bound and test renewal attempts, replace the
   expired token atomically, and surface terminal renewal failure.

Do not let arbitrary consumers reacquire sessions as an incidental retry
helper. That converts a narrow use capability into uncontrolled credential
issuance authority and obscures who can access session material.

Security basis: NIST defines [least privilege](https://csrc.nist.gov/glossary/term/least_privilege)
as restricting users or processes to the minimum resources and authorizations
needed. OWASP recommends centralized, standardized secret handling, fine-grained
access, short-lived/dynamic credentials, auditing of request/use/expiry, and
architectural adapters that decouple applications from secret-management
technology in its [Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html).

### Memory limitation

An application using password-based authentication necessarily creates a
transient plaintext representation for the upstream protocol. In managed
languages, application code cannot guarantee erasure of interpreter, TLS, or
HTTP-library copies. Do not add reversible obfuscation, readable symmetric
keys, DES, XOR, or custom encryption to claim memory protection. Reduce risk by
containing the exchange, minimizing lifetime, preventing propagation and
logging, restricting process inspection/core dumps, and preferring workload
identity or mutual TLS when the external system supports it.
