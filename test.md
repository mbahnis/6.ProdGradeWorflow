# Odaseva Panel — Technical Specification

## 1. Overview

Odaseva Panel is a serverless REST API for creating and querying job candidates and their CV files. It runs on AWS (API Gateway, Lambda, DynamoDB, S3) and is fully provisioned with Terraform. Access is restricted to machine clients authenticated through the OAuth2 Client Credentials flow, backed by Amazon Cognito.

## 2. Architecture

```
                 1. POST /oauth2/token
                    (client_id + client_secret)
   ┌────────┐ ──────────────────────────────────► ┌─────────┐
   │ Client │ ◄────────────────────────────────── │ Cognito │
   └────────┘            access token (JWT)       └─────────┘
       │
       │ 2. HTTPS request + "Authorization: Bearer <token>"
       ▼
┌──────────────────┐   JWT + scope   ┌────────────────────────────┐
│   API Gateway    │ ──────────────► │ Lambda: create_candidate   │──┐
│   (HTTP API)     │   validated     │ Lambda: get_candidate      │──┤
└──────────────────┘                 │ Lambda: list_candidates    │──┤
                                     └────────────────────────────┘  │
                                                                     ▼
                                               ┌──────────────────────────────┐
                                               │ DynamoDB (candidate records) │
                                               │ S3       (CV files)          │
                                               └──────────────────────────────┘
                                                  encrypted with a shared KMS key
```

| Route | Lambda | Required scope |
|---|---|---|
| `POST /candidates` | `create_candidate` | `candidates-api/write` |
| `GET /candidates/{candidateId}` | `get_candidate` | `candidates-api/read` |
| `GET /candidates?specialty=...` | `list_candidates` | `candidates-api/read` |

| Service | Role |
|---|---|
| API Gateway (HTTP API) | Public entry point; validates the Cognito JWT and its scope on each route, then invokes the matching Lambda (`AWS_PROXY`, payload format 2.0). |
| Lambda (Python 3.13) | One function per route (`src/<function>/handler.py`), packaged by Terraform `archive_file`. |
| DynamoDB | Stores candidate records in `odaseva-panel-candidates`; the `specialty-index` GSI serves listing by specialty. |
| S3 | Stores CV files in `odaseva-panel-cv` under `<specialty>/<candidateId>.<extension>`. |
| Cognito | OAuth2 authorization server: user pool, resource server `candidates-api` with `read`/`write` scopes, confidential app client. |
| KMS | Customer-managed key (`alias/odaseva-panel-<environment>`) encrypting DynamoDB and S3 data at rest. |
| IAM | One execution role per Lambda, restricted to the actions and resources that function uses. |
| CloudWatch Logs | Lambda logs (`/aws/lambda/<function>`) and API Gateway access logs (`/aws/apigateway/odaseva-panel`). |

## 3. Architecture and Security Decisions

### API Gateway (`infra/modules/api_gateway`)

| Decision | Alternative considered | Rationale |
|---|---|---|
| HTTP API (`protocol_type = "HTTP"`) | REST API | Native JWT authorizer (no custom Lambda authorizer or Cognito authorizer setup needed), lower cost and latency. REST API features (usage plans, request validation, direct WAF association) are not required for this scope. |
| JWT authorizer with `audience = [cognito_app_client_id]` and issuer `https://cognito-idp.<region>.amazonaws.com/<user_pool_id>` | Matching on the `aud` claim | Cognito access tokens issued through Client Credentials carry no `aud` claim. In that case API Gateway validates the `client_id` claim against the configured audience, so the app client ID is the correct value. |
| Per-route `authorization_scopes` (`write` for POST, `read` for GETs) | Authentication only, no scopes | Separates read and write privileges at the gateway, before any Lambda runs. |
| `$default` stage with `auto_deploy = true` | Named stages with explicit deployments | Single environment; every Terraform change is deployed without a separate deployment resource. |

### Cognito (`infra/modules/cognito`)

| Decision | Alternative considered | Rationale |
|---|---|---|
| OAuth2 Client Credentials flow (`allowed_oauth_flows = ["client_credentials"]`) | Authorization code flow, API keys | Consumers are machine clients. API keys identify but do not authenticate a caller; user-based flows require human users. |
| `generate_secret = true` | Public client | Client Credentials requires a confidential client; the secret is the client's only proof of identity. |
| `allow_admin_create_user_only = true`, no password policy or MFA configuration | Default self sign-up enabled | The pool contains no users and only backs the OAuth2 token endpoint; self sign-up is disabled to guarantee it stays that way. Password policy and MFA only apply to users. |
| `allowed_oauth_scopes = aws_cognito_resource_server.candidates_api.scope_identifiers` | Hard-coded scope strings | Single source of truth for scopes, and an implicit dependency ensuring the resource server exists before the client. |

### DynamoDB (`infra/modules/dynamodb`)

| Decision | Alternative considered | Rationale |
|---|---|---|
| `candidateId` as partition key, no sort key | Composite key `specialty` + `candidateId` | Retrieval by ID is the primary access pattern and requires a single `GetItem`; random IDs distribute writes evenly across partitions. |
| GSI `specialty-index` (hash key `specialty`, `projection_type = "ALL"`) | `KEYS_ONLY` or `INCLUDE` projection | The list route returns all candidate attributes; `ALL` serves it from the index without additional reads. Items are small (no CV content), so duplicated storage is negligible. |
| Only key attributes declared (`candidateId`, `specialty`) | Declaring every business attribute | DynamoDB is schemaless outside keys; declaring non-key attributes is rejected by the API. |
| `billing_mode = "PAY_PER_REQUEST"` | Provisioned capacity | Unpredictable and low traffic; no capacity planning, cost proportional to usage. |
| `point_in_time_recovery { enabled = true }` | No backup, scheduled AWS Backup | Continuous backups with per-second restore over 35 days, no scheduling required. |
| SSE with the customer-managed KMS key | AWS-owned key (default) | Key usage is auditable in CloudTrail and controllable through IAM. |
| Conditional write `attribute_not_exists(candidateId)` before the S3 upload | Unconditional `PutItem` | Prevents an ID collision from overwriting an existing record or its CV object. |

### S3 (`infra/modules/s3`)

| Decision | Alternative considered | Rationale |
|---|---|---|
| SSE-KMS with the shared key, `bucket_key_enabled = true` | SSE-S3; SSE-KMS without bucket key | Same key and audit trail as DynamoDB. The bucket key reduces KMS requests (and their cost) by deriving object keys from a short-lived bucket-level key. |
| Public access block with all four settings set to `true` | Relying on bucket policy only | Account-independent guarantee that no ACL or policy can expose objects publicly. |
| Versioning `Disabled` | Versioning enabled | No CV update route exists, so there are no overwrites to protect against. Once enabled, versioning can only be suspended, not disabled. |
| Downloads through presigned `GetObject` URLs, 600-second expiry | Streaming the file through Lambda; public objects | Keeps the bucket private and avoids the Lambda response size limit. The URL is signed with the `get_candidate` role credentials, which bounds it to that role's permissions. |
| S3 client with `addressing_style = "virtual"` and explicit `region_name` | Default client configuration | The default configuration produces presigned URLs on the global host (`<bucket>.s3.amazonaws.com`) while signing for `eu-west-3`, which fails for recently created buckets. Virtual addressing uses the regional host. |
| `force_destroy = true` | Default (`false`) | Allows repeated `destroy`/`apply` cycles during the demo. Not suitable for production, as a destroy permanently deletes stored CVs. |

### IAM (`infra/modules/iam`)

| Decision | Alternative considered | Rationale |
|---|---|---|
| One role per Lambda (`create_candidate`, `get_candidate`, `list_candidates`) | A shared role for all functions | Least privilege: each role grants only the calls its function makes (`PutItem` + `PutObject`; `GetItem` + `GetObject`; `Query` on the GSI ARN only). |
| `kms:Decrypt` with `kms:ViaService = dynamodb.<region>.amazonaws.com` on all three roles | No KMS permission for DynamoDB access | With a customer-managed key, DynamoDB decrypts the table key on behalf of the caller, so each role needs `kms:Decrypt`. The condition prevents direct KMS calls. |
| Separate S3 statements: `KMSGenerateDataKeyViaS3` (create) and `KMSDecryptViaS3` (get), conditioned on `s3.<region>.amazonaws.com` | Single unconditioned KMS statement; `kms:Encrypt` | SSE-KMS uploads only call `GenerateDataKey`. Keeping one statement per service keeps each key usage auditable. |
| `aws_lambda_permission` with `source_arn = "<api execution_arn>/*/*"` | No `source_arn`, or a wildcard | Without a source ARN, any API Gateway API (including in other accounts) could invoke the functions and bypass the JWT authorizer (confused deputy). The ARN restricts invocation to this API. |
| Managed policy `AWSLambdaBasicExecutionRole` for logging | Inline logging permissions | Standard, minimal CloudWatch Logs permissions. |

### KMS (`infra/modules/kms`)

| Decision | Alternative considered | Rationale |
|---|---|---|
| Single customer-managed symmetric key `aws_kms_key.data_encryption`, shared by DynamoDB and S3 | AWS-managed keys; one key per service | Centralized control and auditing; one key is sufficient for a single-application data scope. |
| `enable_key_rotation = true` | No rotation | Yearly automatic rotation of key material; key ID, ARN and alias remain unchanged and previous material stays available for decryption. |
| `deletion_window_in_days = 7` | 30 days (maximum) | Minimum window, suited to frequent destroy/recreate cycles; still allows cancelling an accidental deletion. |
| Alias `alias/${project_name}-${environment}` | Referencing the key UUID | Readable, stable identifier. |

### Lambda (`infra/modules/lambda`)

| Decision | Alternative considered | Rationale |
|---|---|---|
| Single generic module instantiated three times | One module per function | No duplicated Terraform; functions differ only by name, source directory, role and environment variables. |
| `memory_size = 128`, `timeout = 10` (module defaults) | Higher memory, default 3 s timeout | Lightweight processing (JSON parsing, base64 decoding, two AWS calls). 10 s covers cold start plus a multi-megabyte upload and stays below the 30 s HTTP API integration timeout. |
| Packaging with `archive_file` and `source_code_hash` | Manually built zip, CI build step | Reproducible build; code changes trigger a redeployment automatically. No third-party dependencies (boto3 is provided by the runtime). |
| Resource names passed through environment variables (`DYNAMODB_TABLE_NAME`, `S3_BUCKET_NAME`, `GSI_NAME`) | Hard-coded names | Code is independent of naming conventions and environments. |

### Logging

| Decision | Alternative considered | Rationale |
|---|---|---|
| Explicit Lambda log groups `/aws/lambda/<function>` with `depends_on` from the function | Log groups auto-created by Lambda | Auto-created groups have unlimited retention and are not removed by `terraform destroy`; `depends_on` ensures Terraform creates the group first. |
| Separate API Gateway access log group `/aws/apigateway/odaseva-panel` (JSON: `requestId`, `ip`, `requestTime`, `httpMethod`, `routeKey`, `status`, `responseLength`) | Lambda logs only | Captures requests rejected before reaching a Lambda (401/403 from the authorizer, 404 on unknown routes). |
| `retention_in_days = 14` on all log groups | Unlimited retention | Sufficient for debugging, bounded storage cost. |

## 4. Operating Constraints and Limitations

### Response time

No load test was performed. The following figures are point observations from manual tests (Lambda console and curl), not measured percentiles:

| Scenario | Observed order of magnitude |
|---|---|
| Lambda cold start (Python 3.13, 128 MB) | ~500 ms – 1 s |
| Warm invocation, end-to-end | ~200 – 500 ms |

Latency depends on the CV size (`create_candidate`), the number of result pages (`list_candidates`) and memory allocation: Lambda CPU scales with `memory_size`, so increasing it is the first lever if latency becomes a concern.

### DynamoDB

- On-demand mode: no capacity management; the table scales automatically. According to AWS documentation, on-demand tables instantly absorb up to twice their previous traffic peak, and are bounded by default per-table throughput quotas; sudden spikes beyond that can be throttled.
- Maximum item size: 400 KB (DynamoDB limit). Candidate items only hold a few short attributes and the S3 key (`cvS3Key`), not the CV itself, so this limit is not a practical constraint.
- The GSI is partitioned by `specialty`: a single dominant specialty concentrates index traffic on one partition key.
- `list_candidates` follows `LastEvaluatedKey` and returns all results in one response, with no pagination exposed to the client; very large result sets are bounded by the Lambda timeout and the 6 MB response limit.

### S3 and CV size

- Maximum CV size: **4.5 MB** after base64 decoding, enforced by `MAX_FILE_SIZE_BYTES = 4_500_000` in `create_candidate` (HTTP 400 above it).
- This limit is imposed by the transport, not by S3: the CV is sent base64-encoded in the request body, synchronous Lambda invocation payloads are capped at 6 MB, and base64 adds ~33% overhead. S3 itself supports objects up to 5 TB.
- Presigned download URLs expire after 600 seconds, or earlier if the signing role's temporary credentials expire first.

### Candidate ID

- Format `CA-XXXXXX` (6 random digits, `secrets.randbelow`), i.e. 1,000,000 possible values.
- Collisions are detected by the conditional write and retried with a new ID, up to `MAX_ID_ATTEMPTS = 3`; after three collisions the API returns HTTP 500.
- The collision probability grows with volume (birthday bound: roughly 40% chance of at least one collision among 1,000 candidates), which makes the retry necessary; the ID space is appropriate for a demo, not for large volumes.

### Scalability

- Lambda scales automatically with concurrency. All functions share the account's regional concurrency quota (1,000 by default; new accounts may start lower). This is the main scaling constraint to monitor; no reserved concurrency is configured.
- API Gateway HTTP API scales natively, within the account-level throttling quotas (10,000 requests per second with a burst of 5,000 by default, per region).
- S3 and Cognito scale independently; Cognito token requests are subject to per-account request rate quotas, so clients should cache and reuse access tokens until expiry (1 hour by default).

## 5. Deployment and Testing

The infrastructure is deployed from `infra/` with `terraform init`, `terraform plan` and `terraform apply` (Terraform >= 1.9, AWS provider `~> 5.0`, archive provider `~> 2.0`). Lambda packages are built during the plan. Testing is done with curl: obtain an access token from the Cognito token endpoint using the `cognito_app_client_id` and `cognito_app_client_secret` Terraform outputs, then call the three routes with it and confirm that a request without a token returns 401. The complete commands are in the [README](../README.md).

## 6. Known Limitations and Future Improvements

| Not implemented | Justification |
|---|---|
| WAF and CloudFront | AWS WAF cannot be attached directly to an HTTP API; it would require a CloudFront distribution in front of the API. Out of scope for the MVP; rate limiting relies on API Gateway default throttling. |
| Remote Terraform state (S3 backend with DynamoDB locking) | Solo project with a single operator; the local state avoids bootstrapping extra resources. Required as soon as several people or a pipeline apply changes. |
| Automated CI/CD | The exercise requires manual control of `destroy`/`apply` during the live demo; automatic continuous deployment would conflict with that. A CI running `terraform fmt -check`, `terraform validate` and `plan` would be a low-risk first step. |
| Explicit KMS key policy | The default key policy delegates access control to IAM, which is sufficient given the per-role `kms:ViaService` conditions. An explicit key policy restricted to the three roles would add defense in depth. |
| CV update | No update route is in scope; S3 versioning is therefore disabled. Supporting updates would call for enabling versioning and a `PUT`/`PATCH` route. |
| Consistency between DynamoDB and S3 | The record is written before the CV upload; if the upload fails, the record references a missing object and the API returns 500. Automatic cleanup would require `dynamodb:DeleteItem` on the create role. |
