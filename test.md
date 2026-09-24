# Odaseva Panel — Technical Specification

## 1. Overview

Odaseva Panel is a serverless REST API for creating and querying job candidates
and their CV files. It runs entirely on AWS (API Gateway, Lambda, DynamoDB, S3)
and is provisioned with Terraform. Access is restricted to machine clients
authenticated through the OAuth2 Client Credentials flow, backed by Cognito.

See the [README](../README.md) for the architecture diagram, the full service
list, and deployment/testing instructions.

## 2. Architecture and Security Decisions

### Authentication and API exposure

The API is exposed through an HTTP API (API Gateway), chosen over a REST API
for its native JWT authorizer support, which fits directly with Cognito's
OAuth2 Client Credentials flow without a custom Lambda authorizer.

| Route | Lambda | Required scope |
|---|---|---|
| `POST /candidates` | `create_candidate` | `candidates-api/write` |
| `GET /candidates/{candidateId}` | `get_candidate` | `candidates-api/read` |
| `GET /candidates?specialty=...` | `list_candidates` | `candidates-api/read` |

Cognito is configured purely as an OAuth2 authorization server: no end users,
self sign-up disabled, a single confidential app client authenticating with a
client secret. Each route requires a specific scope, enforced by the API
Gateway authorizer before a Lambda ever runs.

### Data storage

DynamoDB stores candidate records, keyed by `candidateId` (the main access
pattern), with a secondary index on `specialty` to support the listing route.
S3 stores the CV files, referenced from DynamoDB by their key. Both are
encrypted at rest with a shared, customer-managed KMS key, which keeps key
usage auditable and centrally controlled rather than relying on AWS default
encryption. The S3 bucket blocks all public access; CVs are only reachable
through presigned URLs valid for 10 minutes.

### Least privilege

Each Lambda has its own IAM role, scoped to exactly the actions and resources
it needs (for example, the listing function can only query the index, never
read or write S3). KMS permissions are further restricted so the key can only
be used through DynamoDB or S3, not called directly. API Gateway's permission
to invoke each Lambda is scoped to this specific API, not to API Gateway in
general, to avoid a compromised or misconfigured API elsewhere from reaching
these functions.

### Reliability details

Candidate creation writes to DynamoDB before uploading the CV, using a
conditional write to avoid overwriting an existing record if two requests
generate the same random ID (retried automatically). Lambda logs and API
Gateway access logs are kept separate, the latter also capturing requests
rejected before reaching a Lambda (missing or invalid token).

## 3. Operating Constraints and Limitations

- **CV size**: capped at 4.5 MB. The file travels base64-encoded in the
  request body, and Lambda's synchronous payload limit (6 MB) is the binding
  constraint, not S3 itself.
- **Response time**: no formal load test was run. Manual testing (console and
  curl) showed sub-second responses end to end, including cold starts.
- **Candidate ID**: a random 6-digit identifier with automatic collision
  retry. Adequate for a demo; a larger ID space or a sequence would be needed
  at production volume.
- **DynamoDB**: on-demand billing, so no capacity planning is required; it
  scales automatically within standard AWS account quotas.
- **Consistency**: if the CV upload fails after the record is written, the API
  returns an error but the record is not automatically rolled back. Acceptable
  for this scope; a production version would clean up the orphaned record.

## 4. Deployment and Testing

The infrastructure is deployed with `terraform init/plan/apply` from `infra/`.
Testing is done with curl against the deployed API: obtain an access token
from the Cognito token endpoint, then call the three routes, and confirm that
a request without a token is rejected. Full commands are in the
[README](../README.md).

## 5. Known Limitations and Possible Improvements

Left out of scope for this exercise, with the reasoning behind each:

- **WAF / CloudFront**: not attachable directly to an HTTP API; would require
  a CloudFront distribution in front of it.
- **Remote Terraform state**: local state is enough for a single operator;
  a team or CI setup would need an S3 backend with locking.
- **CI/CD**: the exercise calls for manual control of destroy/apply during the
  live demo, which an automated pipeline would work against.
- **CV updates**: no update route is in scope, so S3 versioning is disabled.
