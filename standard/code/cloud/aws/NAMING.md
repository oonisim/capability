# Standards

Naming conventions and standards for AWS resources in smst-ai.

---

## AWS Resource Path Convention

All path-based AWS resources use a consistent hierarchical convention.

**Secrets Manager** names do NOT start with `/` (just a string, not a path):
```
{organisation}/{project}/{environment}/{system}[/{component}]/api/{version}/{type}
```

**SSM Parameter Store** paths start with `/` (AWS requirement for hierarchical parameters):
```
/{organisation}/{project}/{environment}/{system}[/{component}]/api/{version}/{type}
```

| Segment | Values | Example |
|---------|--------|---------|
| `organisation` | `esol` | Organisation that owns the resource |
| `project` | `smst-ai` | Project name |
| `environment` | `dev`, `prd` | Deployment environment |
| `system` | `courseloop`, `degreeworks` | Target system name |
| `component` | `scribe`, `document` | Sub-component (optional, omit if none) |
| `version` | `v1`, `v2` | API version |
| `type` | `key`, `config` | `key` for secrets, `config` for SSM parameters |

### Examples

| Type | Path |
|------|------|
| Secret (key) | `esol/smst-ai/dev/courseloop/api/v1/key` |
| Secret (key) | `esol/smst-ai/dev/degreeworks/scribe/api/v1/key` |
| Secret (key) | `esol/smst-ai/dev/degreeworks/document/api/v1/key` |
| SSM (config) | `/esol/smst-ai/dev/courseloop/api/v1/config` |
| SSM (config) | `/esol/smst-ai/dev/degreeworks/scribe/api/v1/config` |
| SSM (config) | `/esol/smst-ai/dev/degreeworks/document/api/v1/config` |
| SSM (metadata) | `/esol/smst-ai/dev/dw-scribe/deploy-metadata` |

### Rules

- SSM paths start with `/` (AWS requirement). Secrets Manager names do not.
- `organisation` is always included in both — prevents collisions if
  multiple organisations share an AWS account.
- `api` comes before `version` — the version qualifies the API, not the system.
- Multiple versions can coexist (e.g. v1 and v2 during migration).
- `deploy-metadata` is a special case — no system/api segments.

---

## AWS Resource Name Convention

Non-path resources (EC2, S3, IAM, SG, VPC endpoints) use a hyphenated
name prefix:

```
{organisation}-{project}-{environment}
```

Example: `esol-smst-ai-dev`

Each module appends a resource-specific suffix:

| Resource | Pattern | Example |
|----------|---------|---------|
| IAM role | `{prefix}-{app}-ec2-role` | `esol-smst-ai-dev-dw-scribe-ec2-role` |
| IAM policy | `{prefix}-{app}-{policy}` | `esol-smst-ai-dev-dw-scribe-bedrock` |
| S3 bucket | `{prefix}-{app}-data-{suffix}` | `esol-smst-ai-dev-dw-scribe-data-f91c5b6c` |
| Security group | `{prefix}-{app}-sg` | `esol-smst-ai-dev-dw-scribe-sg` |
| ALB frontend SG | `{prefix}-{app}-sg-alb-frontend` | `esol-smst-ai-dev-dw-scribe-sg-alb-frontend` |
| VPC endpoint | `{prefix}-common-vpce-{service}` | `esol-smst-ai-dev-common-vpce-ssm` |
| Organisation SG | `{prefix}-common-org` | `esol-smst-ai-dev-common-org` |
| Prefix list | `{prefix}-common-org-cidrs` | `esol-smst-ai-dev-common-org-cidrs` |
| Lambda function | `{prefix}-{app}-scribe-endpoint` | `esol-smst-ai-dev-dw-scribe-scribe-endpoint` |
| DynamoDB table | `{prefix}-{app}-scribe-job` | `esol-smst-ai-dev-dw-scribe-scribe-job` |
| ALB | `{prefix}-scribe` | `esol-smst-ai-dev-scribe` (32-char limit, penultimate segment stripped) |
| R53 hosted zone | `{project}-{env}.{parent}` | `smst-ai-dev.aws.monash.edu` |

### IAM SID Rules

All IAM policy SIDs must match `[0-9A-Za-z]*` only — no hyphens,
underscores, or special characters. Use CamelCase of the name prefix.

Example: `esol-smst-ai-dev` → SID prefix `EsolSmstAiDev`

---

## Terraform Output Naming Convention

All Terraform outputs — both at the root stack level and inside modules —
must begin with the AWS service abbreviation as a prefix, separated by
underscore:

```
{aws_service}_{descriptor}
```

### Rationale

Outputs are the primary data source for resource inventory management
(see `TODO.md` item 0). A consumer building a service-grouped inventory
tree must be able to determine which AWS service an output belongs to
from the name alone, without inspecting its value or description.

Without the prefix, outputs like `zone_id`, `instance_id`, `bucket_name`
are ambiguous across services — they cannot be categorised automatically.

### AWS service prefixes

| AWS Service | Prefix | Example output |
|---|---|---|
| EC2 (instance) | `ec2_` | `ec2_agent_instance_id`, `ec2_vector_db_private_ip` |
| IAM | `iam_` | `iam_role_arn`, `iam_policy_arn` |
| S3 | `s3_` | `s3_data_bucket_name`, `s3_artifacts_bucket_name` |
| Security Group | `sg_` | `sg_alb_frontend_id`, `sg_vpc_endpoint_id` |
| ALB | `alb_` | `alb_frontend_arn`, `alb_frontend_dns_name` |
| Lambda | `lambda_` | `lambda_function_arn`, `lambda_scribe_endpoint_alias_arn` |
| ECR | `ecr_` | `ecr_repository_urls`, `ecr_repository_arns` |
| Route 53 | `r53_` | `r53_public_zone_id`, `r53_name_servers` |
| VPC | `vpc_` | `vpc_id`, `vpc_cidr_block` |
| SSM Parameter Store | `ssm_` | `ssm_deploy_metadata_name` |
| Secrets Manager | `sm_` | `sm_scribe_api_key_arn` |
| KMS | `kms_` | `kms_customer_key_arn` |
| ACM | `acm_` | `acm_certificate_arn` |

### Rules

- **No unprefixed outputs.** An output name must make its service category
  self-evident without reading its `description`.
- **Multi-resource disambiguator after the prefix.** When multiple
  resources of the same service exist, add a logical role name between the
  prefix and the descriptor:
  `ec2_agent_instance_id`, `ec2_cv_instance_id`, `ec2_vector_db_private_ip`.
- **Module outputs omit the service prefix.** Inside a module the service
  is already implied by the module name (`module.r53.*`, `module.ec2_agent.*`).
  Adding the prefix would create redundant double-namespacing such as
  `module.r53.r53_zone_id`. Use the shortest unambiguous descriptor:
  `public_zone_id`, `record_cname_alb_fqdn`, `name_servers`.
- **Root outputs always carry the service prefix.** At the root stack level
  there is no surrounding module context, so the prefix is required:
  `r53_name_servers`, `ec2_agent_instance_id`, `s3_data_bucket_name`.
- **Existing outputs that predate this convention** (`data_bucket_name`,
  `instance_ids`, etc.) should be renamed when they are next touched —
  do not rename en masse to avoid churn on live resources.

---

## Utility Script Naming Convention

Scripts under `utility/` use a verb prefix that describes the script's intent:

| Prefix | Intent | Example |
|---|---|---|
| `monitor_` | Health check — read-only probe of a live resource | `monitor_alb.sh` |
| `invoke_` | Trigger or call a remote resource (Lambda, API) | `invoke_scribe_endpoint.sh` |
| `run_` | Execute a pipeline or multi-step operation | `run_pipeline_destroy_dev.sh` |
| `release_` | Promote code through environments | `release_feature.sh` |
| `generate_` | Produce a report, artefact, or derived output | `generate_cost_report.sh` |

### Rules

- **No side effects for `monitor_`** — scripts must be read-only; no writes, no mutations.
- **One resource per script** — keep scope narrow so scripts compose rather than sprawl.
