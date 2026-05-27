# Cross-Account Role Policy — Rationale (Internal)

This document explains why each section and permission in
`MechanizedAI-CrossAccount-Role.json` is required. Intended for the team —
not for customer-facing distribution.

The customer creates a single IAM role (`MechanizedAI-CrossAccount-Role`) in
their AWS account. Our backend (in account `686992243604`) assumes that role
via STS with an external-ID condition and uses the granted permissions to
provision and manage every per-customer resource: VPC, ECS, Lambda, Step
Functions, OpenSearch, EFS, RDS, etc.

All resource provisioning is driven by Pulumi from `mlops-iac-user-account`.
The grep references below point at that repo unless otherwise noted.

---

## Statement 0 — Broad service access (`Resource: "*"`)

Permissions for the AWS services Pulumi creates and manages on behalf of the
customer. `*` resource scope is required because most of these are global,
region-wide, or created with names we don't know until runtime.

| Action | Why we need it | Where it's used |
|---|---|---|
| `acm:*` | Issue/manage TLS certificates for the per-customer ALBs that front TigerGraph, GraphRAG, the router, etc. | `ecs/graphrag.py`, `ecs/tigergraph.py`, `ecs/backend/router.py` (`aws.acm.Certificate`, `CertificateValidation`) |
| `apigateway:*` | Reserved for the customer-side API Gateway invocation roles (`MechanizedAI-APIGatewayInvoication`, `MechanizedAI-APIGatewayStepfunctionExecution`) and external API Gateway integrations driven by the backend; no `aws.apigateway.*` resources are currently created from IaC but the roles and downstream usage assume the permission is present | `iam/__init__.py` (API Gateway service-principal roles) |
| `route53:*` | DNS records for the per-customer endpoints (graphrag, tigergraph, router, etc.) | `aws.route53.*` in 4 files |
| `secretsmanager:ListSecrets` | List secrets owned by the customer account so the backend can enumerate them in the UI (LLM config page). Listing requires `*` scope; the broader `secretsmanager:*` below covers read/write but `*:ListSecrets` must be explicit because list operations don't accept resource-level scoping. | UI/backend secret enumeration |
| `application-autoscaling:*` | Target tracking / step scaling for ECS services (currently the mermaid renderer) | `ecs/backend/mermaid.py` (`aws.appautoscaling.Target`, `Policy`) |
| `cloudwatch:*` | Metrics, alarms and dashboards for the ECS services and Lambdas | `aws.cloudwatch.*` in 14 files (incl. `LogGroup`) |
| `states:*` | Step Functions — codemap, codegen, graphrag, source import pipelines | `aws.sfn.*` in 8 files |
| `logs:*` | CloudWatch Logs groups for every Lambda and ECS task. Required by Lambda + ECS task execution roles, and by Pulumi to create the LogGroup resources | `LogGroup` resources and `logs:PutLogEvents` in task definitions |
| `events:*` | Reserved for EventBridge — the `MechanizedAI-EventBridge` role is created in `iam/__init__.py` and the orphaned `stepfunctions/{datasync,dms,glue_headers}_status/` step functions still reference EventBridge buses; backend may use it for cross-account event routing. | Pulumi doesn't currently create `aws.events.*` resources in active code paths |
| `ec2:*` | VPC, subnets, route tables, security groups, NAT gateways, ENIs — used by every Lambda/ECS deployment | `aws.ec2.*` in 11 files |
| `ecs:*` | ECS clusters, services, task definitions — TigerGraph, GraphRAG, router, codegen, codemap, mermaid, imageinfo, etc. | `aws.ecs.*` in 13 files |
| `ecr:*` | ECR repos and pull permissions for the per-customer Lambda/ECS images (cross-account pulls from our backend ECR) | `aws.ecr.*` and Lambda image_uri references |
| `servicediscovery:*` | AWS Cloud Map for ECS service-to-service discovery (used by codemap/codegen ECS tasks) | `aws.servicediscovery.*` in 11 files |
| `aoss:*` | OpenSearch Serverless collections, security/access policies, encryption — backs file descriptions, source analysis, code chunks | `aws.opensearch.*` (incl. `serverlessSecurityPolicy`, `serverlessAccessPolicy`) |
| `kms:*` | KMS keys for envelope encryption (S3, Secrets Manager, EFS, RDS) | `aws.kms.*` in 5 files |
| `cloudformation:*` | Kept conservatively — Pulumi doesn't directly call CFN, but the Lambda role policy granted in `iam/__init__.py` references CFN actions (`cloudformation:CreateChangeSet`, etc.) for runtime workflows that interact with stacks | `iam/__init__.py` Lambda-role policy doc |
| `s3:ListBucket` | Listing buckets across the account (e.g. discovering customer-supplied source buckets in onboarding/UI flows). Resource-scoped `s3:*` below covers all writes on `mechanizedai-*` buckets we create | Backend bucket discovery |
| `cloudfront:UpdateDistribution` | Update CloudFront distributions for custom-domain support on API Gateway / Amplify — managed by the backend, not by Pulumi | Backend custom-domain wiring |
| `lambda:*` | Lambdas for codemap pipeline (summarizer, classifier, mermaid, imageinfo, etc.), source analysis, file descriptions, LLM-config registration, etc. Use `lambda:*` rather than enumerating actions to keep the policy compact as we add Lambdas | `aws.lambda_.*` in 3 files (with many per-file resources) |
| `rds:*` | RDS Aurora Postgres cluster that backs the LiteLLM router (stores model registrations, request logs) | `from usercomponents import rds` in `__main__.py` |
| `elasticache:*` | ElastiCache (Redis) used by the LiteLLM router for response caching and rate limiting | Router infra |
| `elasticloadbalancing:*` | ALBs / target groups for the per-customer ECS services (TigerGraph, GraphRAG, router) | `aws.lb.*` in 3 files |
| `elasticfilesystem:*` | EFS for TigerGraph graph store persistence | `aws.efs.*` in `ecs/tigergraph.py` |
| `ssm:*` | SSM Parameter Store — LLM config (`/mechanizedai/llm/active/*`), router host/master-key, per-workspace settings read by Lambdas and ECS containers at runtime | `aws.ssm.*` in 5 files |
| `secretsmanager:*` | Manage secrets (Vertex AI service-account JSONs, OpenAI/Anthropic keys, etc.) — created/updated/deleted from the backend when users save LLM config | `aws.secretsmanager.*` in 2 files; backend writes via cross-account |

---

## Statement 1 — IAM read/write (`Resource: ["*"]`)

We create the customer-side IAM roles that AWS services (Lambda, ECS, Step
Functions, etc.) assume in order to operate. Pulumi needs to create, attach
policies to, tag, and eventually tear down these roles. `Resource: *` is
because role/policy ARNs aren't known until they are created.

| Action | Why |
|---|---|
| `iam:GetRole`, `GetPolicy`, `GetRolePolicy`, `GetPolicyVersion`, `ListAttachedRolePolicies`, `ListInstanceProfilesForRole`, `ListPolicyVersions`, `ListRolePolicies`, `ListEntitiesForPolicy` | Pulumi refresh / state reconciliation — reading current state before each `pulumi up` |
| `iam:CreateRole`, `CreatePolicy`, `CreatePolicyVersion`, `CreateServiceLinkedRole`, `AttachRolePolicy`, `PutRolePolicy`, `UpdateAssumeRolePolicy` | Creating the per-customer IAM roles (`MechanizedAI-LambdaRole`, `MechanizedAI-ECSTaskRole`, `MechanizedAI-SFNRole`, etc.) and attaching policies |
| `iam:DeleteRole`, `DeletePolicy`, `DeletePolicyVersion`, `DeleteRolePolicy`, `DetachRolePolicy` | Tear-down when a workspace is removed or when policies are re-versioned (AWS keeps 5 versions max) |
| `iam:TagRole`, `UntagRole`, `TagPolicy`, `UntagPolicy` | Resource tagging — every Pulumi-managed resource gets the `aws-apn-id` tag and per-workspace labels |

Could be scoped to `arn:aws:iam::*:role/MechanizedAI-*` and
`arn:aws:iam::*:policy/MechanizedAI-*` for least-privilege; left at `*` because
`iam:CreateServiceLinkedRole` requires `*` and a few list/get actions don't
support resource-level conditions.

---

## Statement 2 — S3 buckets we create (`s3:::mechanizedai-*`)

All buckets Pulumi creates for the workspace follow the `mechanizedai-*`
naming convention (data bucket per workspace, source storage, codegen output,
etc.). Granting `s3:*` only on this prefix prevents the role from touching
unrelated customer S3 data.

---

## Statement 3 — DynamoDB tables we create (`table/mechanizedai-*`)

Same idea as S3 — Pulumi-managed DynamoDB tables follow the `mechanizedai-*`
naming convention (source analysis, code processing, file processing, business
rules, user-project restrictions, etc.).

---

## Statement 4 — `iam:PassRole`

When Pulumi creates a Lambda function, ECS task definition, Step Function,
etc., it must pass an existing role to that service. AWS requires the
explicit `iam:PassRole` permission, scoped to the roles being passed.

* `arn:aws:iam::*:role/aws-service-role/ecs.application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling_ECSService`
  — service-linked role required by ECS Application Auto Scaling (mermaid scaling).
* `arn:aws:iam::*:role/MechanizedAI-*` — wildcards over every role we create
  (`MechanizedAI-LambdaRole`, `MechanizedAI-ECSTaskRole`, `MechanizedAI-ECSTaskExecutionRole`,
  `MechanizedAI-SFNRole`, `MechanizedAI-EventBridge`, `MechanizedAI-llm-configs-lambda-role`, etc.)

---

## Why `*` on the resource for Statement 0

For most AWS services the resource type either doesn't support ARN-level
permission scoping at all (`ec2:*`, `kms:*` for key creation, `cloudwatch:*`,
`states:*`, etc.) or we don't know the name until Pulumi creates it. Scoping
each action would either be a no-op or require pre-computing every resource
ARN.

For S3 and DynamoDB the resource is scoped — those statements are separated
out (Statements 2 and 3) because we own the naming convention and can enforce
the prefix.

---

## Dropped from previous versions

* `sagemaker:*`, `glue:*`, `emr-serverless:*`, `elasticmapreduce:*`,
  `datasync:*`, `athena:*`, `dms:*` — services for a product line we no
  longer ship. The orphaned `stepfunctions/{datasync,dms,glue_headers}_status/`
  files in `mlops-iac-user-account` are dead code (not imported).
* `s3:::sagemaker-*`, `s3:::jumpstart-*` — bucket patterns tied to the same
  retired SageMaker product line.
* Individual `lambda:*` actions (`lambda:CreateFunction`,
  `lambda:InvokeFunction`, etc.) — collapsed into `lambda:*`. Also fixed the
  `Lambda:RemovePermission` capital-L typo as a side effect.
* Dedicated secretsmanager block (`secretsmanager:GetSecretValue`,
  `DescribeSecret`, `CreateSecret`, `UpdateSecret`, `DeleteSecret` on
  `mechanizedai*` secrets) — fully covered by Statement 0's `secretsmanager:*`.
* Dedicated elasticloadbalancing scoped block (added at some point for a
  specific target group) — fully covered by Statement 0's
  `elasticloadbalancing:*`.

---

## Locations that must stay in sync

When the policy changes, update **all** of these:

1. `mlops-cross-account-template/MechanizedAI-CrossAccount-Role.json` — the
   CloudFormation template customers download.
2. `mlops-documentation/pages/role.mdx` — the manual setup instructions
   published at docs.mechanized.ai.
3. `mlops-iac-user-account/helper-scripts/update_role.py` — the script we run
   in-place to update existing roles on customer accounts.
4. `MechanizedAI - Account Setup` Word doc (in shared drive / Downloads) — the
   PDF distributed during onboarding.

The live policy in any customer account can be retrieved with:

```
aws iam get-policy-version \
  --policy-arn arn:aws:iam::<account>:policy/MaquinaAI-Access-Policy \
  --version-id $(aws iam get-policy --policy-arn ... --query 'Policy.DefaultVersionId' --output text) \
  --profile <profile> --query 'PolicyVersion.Document' --output json
```

Use `profile=dummy-dev` for the dev account (`170469460161`) as the canonical
source of truth.
