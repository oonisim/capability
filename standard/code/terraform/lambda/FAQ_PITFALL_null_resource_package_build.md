# Pitfall: Lambda Package Build with `null_resource` + `archive_file` Fails in CI

## Problem Statement

A Terraform configuration that builds a Lambda deployment package locally
using a `null_resource` provisioner and then zips it with `archive_file`
(either `data` or `resource` form) works correctly on a developer laptop but
fails every time it runs in CI (e.g., Bitbucket Pipelines, GitHub Actions):

```
Error: Archive creation error

  with module.lambda.archive_file.lambda_scribe_endpoint,
  on modules/lambda/build_lambda_scribe_endpoint.tf line 121

error creating archive: error archiving directory: could not archive
missing directory: /opt/atlassian/pipelines/agent/build/.../modules/lambda/_build/lambda-scribe-endpoint
```

The `_build/` directory does not exist at the time Terraform tries to create
the zip.

---

## Cause

The root cause is a **mismatch between Terraform's state model and the
ephemeral nature of CI workspaces**.

Terraform state persists across runs (stored in a remote backend such as S3).
CI workspaces do not — every pipeline run starts from a clean clone with no
local filesystem artefacts.

`null_resource` is **state-driven**, not **filesystem-driven**. It runs its
provisioner only when the resource is **created or replaced** — that is, when
its `triggers` map differs from what is stored in state. If the triggers have
not changed (same source file hashes, same requirements hash) the provisioner
is skipped entirely. Terraform assumes the local side effects from the
previous run still exist.

They do not. The CI container was destroyed.

---

## Mechanism — Step-by-Step Failure Sequence

```
Pipeline Run 1 (initial deploy)
───────────────────────────────
1. null_resource has no state → triggers changed → provisioner runs
2. _build/ directory created, packages installed
3. archive_file reads _build/ → zip created → success
4. State saved:
     null_resource.build: { source_hash = "abc", requirements_hash = "xyz" }
     archive_file.lambda_function: { output_path = "_build/lambda.zip", ... }

Pipeline Run 2 (same code, fresh CI container)
───────────────────────────────────────────────
1. Terraform pulls state from remote backend (S3 / Terraform Cloud)
2. State says: null_resource already exists, triggers = { source_hash = "abc", ... }
3. Code still has same hashes → triggers UNCHANGED → null_resource: "no change, skip"
4. Provisioner NEVER runs → _build/ does not exist (container is fresh)
5. archive_file tries to read _build/ → FAILS
```

The failure manifests differently depending on the form of `archive_file`:

| Form | When it fails | Error type |
|------|--------------|------------|
| `data "archive_file"` | During **plan** (evaluated at plan time) | Plan-phase error |
| `resource "archive_file"` | During **plan refresh** of existing state entries | Refresh-phase error |

Switching from `data` to `resource` does **not** fix the problem. Both forms
of the `archive` provider evaluate or refresh at plan time. The Terraform
`depends_on` relationship between `archive_file` and `null_resource` only
controls *apply ordering*, not *plan-time evaluation*.

### Why `always_run = timestamp()` Also Fails

A common attempted fix is to add a trigger that always changes:

```hcl
triggers = {
  always_run = timestamp()   # forces null_resource replacement every apply
  ...
}
```

This forces the `null_resource` to be **planned for replacement**, but
`timestamp()` is `(known after apply)` during the plan phase. Terraform
therefore also marks dependent `archive_file` resources as needing
replacement — and tries to pre-validate or refresh those resources during
planning, before the `null_resource` provisioner has had a chance to run.

The failure sequence with `always_run`:

```
Plan phase:
  null_resource triggers change → planned for replacement
  archive_file depends on null_resource → planned for replacement
  archive_file provider attempts to validate the source_dir during planning
  source_dir doesn't exist yet → FAILS during plan
```

The `null_resource` provisioner never gets to run because the plan itself
errors out first.

---

## Lesson Learned

1. **`null_resource` is not a build system.** It is a Terraform state
   primitive. It models "did I run this command at least once with these
   inputs" — not "does the artefact this command produced still exist".

2. **Local filesystem side effects are invisible to Terraform state.** State
   records that the resource exists and what its triggers were; it has no
   knowledge of whether the files the provisioner created are still on disk.

3. **`data "archive_file"` and `resource "archive_file"` both evaluate at
   plan time** in the `hashicorp/archive` provider. Switching between them
   does not defer evaluation to apply time. The distinction does not help.

4. **`depends_on` does not defer plan-time evaluation.** It only enforces
   apply ordering. It cannot prevent the archive provider from attempting to
   read the source directory during planning.

5. **The laptop works because the `_build/` directory from a prior run is
   still present.** CI fails because every run is a fresh workspace.

---

## The Right Way: Build Before Terraform

**The deployment package must be built before `terraform apply` runs.**

Terraform is an infrastructure provisioning tool, not a build system.
Mixing build steps into Terraform via `null_resource` creates a fragile
coupling between CI workspace state and Terraform state that is guaranteed
to break in ephemeral environments. The correct separation of concerns is:

```
CI Pipeline
───────────
Step 1: Build          → produce lambda.zip (pip install, copy sources, zip)
Step 2: terraform init → initialise providers and modules
Step 3: terraform apply → deploy infrastructure, reference the pre-built zip
```

Terraform's role is to **reference** the artefact, never to **build** it.

### CI Pipeline Pattern (Bitbucket Pipelines example)

```yaml
pipelines:
  default:
    - step:
        name: Build Lambda package
        script:
          - cd src/app/scribe/endpoint
          - pip3 install -r requirements.txt --target ./package --no-compile
          - cp *.py ./package/
          - cd package && zip -r ../lambda_function.zip . && cd ..
        artifacts:
          - src/app/scribe/endpoint/lambda_function.zip

    - step:
        name: Terraform apply
        script:
          - terraform -chdir=platform/iac/terraform/deployment/degreeworks-scribe apply
```

### Terraform references the pre-built zip

```hcl
module "scribe_endpoint" {
  source  = "terraform-aws-modules/lambda/aws"
  version = "~> 7.0"

  create_package         = false
  local_existing_package = "${path.root}/../../../src/app/scribe/endpoint/lambda_function.zip"

  handler = "lambda_handler.dispatch"
  runtime = "python3.11"
  # ...
}
```

Change detection works correctly because `local_existing_package` causes the
module to compute `filebase64sha256` of the zip. If the zip content has not
changed, the Lambda function is not redeployed.

### For a Lambda layer (same principle)

```yaml
# CI build step for boto3 layer:
- pip3 install boto3==1.40.61 botocore==1.40.61 --target ./layer/python --no-compile
- cd layer && zip -r ../boto3_layer.zip python/ && cd ..
```

```hcl
module "boto3_layer" {
  source  = "terraform-aws-modules/lambda/aws"
  version = "~> 7.0"

  create_layer        = true
  create_function     = false
  compatible_runtimes = ["python3.11"]

  create_package         = false
  local_existing_package = "${path.root}/../../../src/app/scribe/endpoint/boto3_layer.zip"
}
```

---

## Workaround: Module-Native Packaging (`create_package = true`)

If restructuring the CI pipeline is not immediately feasible,
`terraform-aws-modules/lambda/aws` supports `create_package = true` with
`source_path`, which runs the packaging internally during apply. This avoids
the `null_resource` + `archive_file` pattern, but **still mixes build and
deploy** and is therefore a workaround, not the architectural right answer.

```hcl
module "my_function" {
  source  = "terraform-aws-modules/lambda/aws"
  version = "~> 7.0"

  create_package = true
  source_path = [
    {
      path             = var.source_dir
      pip_requirements = true
    }
  ]
  # ...
}
```

With this approach:
- No `null_resource`, no `archive_file`, no `_build/` directory.
- The module handles packaging internally with correct apply-time sequencing.
- Works in CI, but `terraform apply` now performs a `pip install` on every
  run, increasing apply time and coupling deploy to Python tooling being
  present in the Terraform runner image.

### Import path consequence when using `source_path` directly

When `source_path` points to the source directory, all files land at the
**zip root**. Update imports in `lambda_handler.py` accordingly:

```python
# Before (files nested inside app/scribe/endpoint/ in zip):
from app.scribe.endpoint.api import ScribeClient

# After (files at zip root):
from api import ScribeClient
```

---

## Cleaning Up Stale `archive_file` State Entries

If `archive_file` resources were previously applied and exist in state,
Terraform will try to refresh them on every subsequent run — and fail because
`_build/` no longer exists. Use `removed` blocks (Terraform ≥ 1.7) to drop
the state entries without destroying anything:

```hcl
# In the module where the archive_file resources lived:
removed {
  from = archive_file.lambda_layer_boto3
  lifecycle {
    destroy = false   # don't try to delete the (non-existent) zip file
  }
}

removed {
  from = archive_file.lambda_scribe_endpoint
  lifecycle {
    destroy = false
  }
}
```

After one successful apply these blocks can be deleted — the state entries
are gone.

Also remove the `hashicorp/archive` provider from `required_providers` once
no `archive_file` resources remain.

---

## Future Prevention

- **Build artefacts before Terraform runs.** Make the build a separate CI
  step that produces a zip file as a pipeline artefact. Terraform then
  references the zip via `local_existing_package`. This is the correct
  separation of concerns.

- **Never use `null_resource` + `archive_file` for Lambda packaging.**
  The pattern is fundamentally incompatible with ephemeral CI workspaces and
  Terraform's state model.

- **`create_package = true` is a workaround, not a solution.** It avoids
  the worst failure modes but still mixes build and deploy concerns inside
  Terraform. Prefer CI-step pre-build for any production workflow.

- **If you must use `null_resource` for a build step**, store the artefact in
  a location that persists across runs (S3, an artifact store) and have
  Terraform download it rather than rebuild it from scratch.

- **Recognise the warning signs during code review:**
  - `null_resource` with `local-exec` that writes files to a local path
  - `archive_file` whose `source_dir` or `source_file` is under a directory
    not committed to the repository
  - `depends_on` used to sequence `archive_file` after a build `null_resource`
  - Build commands (`pip install`, `npm ci`, `go build`) inside Terraform HCL

---

## Notes

- The `hashicorp/archive` provider (both `data` and `resource` forms of
  `archive_file`) reads the filesystem during Terraform's plan/refresh phase.
  This is a known characteristic of the provider, not a bug. See:
  [terraform-provider-archive #78](https://github.com/hashicorp/terraform-provider-archive/issues/78)

- `resource "archive_file"` was introduced specifically to address plan-time
  evaluation, but the `archive` provider's implementation still reads source
  paths during refresh of existing state entries. The distinction between
  `data` and `resource` is not meaningful for this failure mode.

- The `terraform-aws-modules/lambda/aws` module's internal `null_resource.archive`
  uses a `timestamp` trigger for a different reason — the module's packaging
  script is idempotent and the zip is stored in a `builds/` subdirectory
  keyed by content hash, so re-running it is always safe and cheap.

- When using `pip_requirements = true` in `source_path`, all packages from
  `requirements.txt` are installed. If a Lambda layer is intended to pin only
  a subset of packages (e.g., boto3/botocore only), consider maintaining a
  separate `requirements-layer.txt` and setting `pip_requirements =
  "${path.module}/requirements-layer.txt"`.
