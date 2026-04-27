# Cursor Prompt: Megapayment MVP-1 Implementation v1.1

## Source of Truth

Read and follow this specification:

```text
docs/megapayment-implementation-specification-v3.3.md
```

The specification is the source of truth. This prompt is only an execution instruction.

If this prompt and the specification conflict, follow the specification.

---

## Before Making Changes

First inspect the current project structure.

Check existing:

- solution/project layout;
- models/entities;
- DbContext or database access layer;
- migrations;
- services;
- repositories;
- controllers/API endpoints;
- Blazor pages/components;
- seed data approach;
- tests.

Reuse existing patterns.

Do not create a parallel architecture if the project already has a working structure.

Do not delete existing documentation, code, pages, components, migrations, or tests unless they are clearly obsolete and replaced by the MVP-1 implementation.

---

## Task

Implement only MVP-1 of the Megapayment internal diagnostic module.

MVP-1 scope:

- operation list;
- operation details;
- operation route timeline;
- operation-level evidence links;
- route-step-level evidence links;
- grouped route-step evidence endpoint;
- backend seed data;
- backend APIs;
- Blazor UI pages/components for operations.

---

## Implementation Order

Work in this order:

1. inspect current structure and identify existing patterns;
2. add database migration and models/entities;
3. add repositories/data access;
4. add application services;
5. add API endpoints;
6. add safe seed data;
7. add/update Blazor pages and components;
8. add/update tests;
9. run build/tests/checks available in the repository;
10. summarize what changed and what was not implemented.

Do not move to MVP-2 or MVP-3 after MVP-1 is done.

---

## Do Not Implement

Do not delete existing documentation, code, pages, components, migrations, or tests unless they are clearly obsolete and replaced by the MVP-1 implementation.

Do not implement:

- ingestion endpoints;
- diagnostic checks;
- real provider integrations;
- real observability API queries;
- audit workflow;
- role management UI;
- service/provider CRUD unless already implemented;
- MVP-2 scope;
- MVP-3 scope;
- full solution restructuring.

Use the existing repository structure. Add folders/classes only where needed. Reuse existing patterns.

---

## MVP-1 Technician API Rule

Only `GET /api/technician/*` endpoints are allowed in MVP-1.

Do not add technician API endpoints using:

```text
POST
PUT
PATCH
DELETE
```

---

## External Operation Identifier Rule

Use safe generated diagnostic operation identifiers.

Database field:

```text
diagnostic_operation_id
```

API field:

```text
operationId
```

Mapping:

```text
operationId = operation_diagnostics.diagnostic_operation_id
```

Do not store or return:

```text
externalOperationId
paymentId
cardOperationId
customerOperationId
rawOperationReference
customerId
clientId
operation_diagnostics.id
external_operation_reference_hash
```

`external_operation_reference_hash` may remain `NULL` in MVP-1 seed/demo data.

Do not implement real external operation reference hashing unless explicitly requested.

---

## Database Requirements

Use PostgreSQL.

Add the following MVP-1 tables:

```text
operation_diagnostics
operation_route_steps
diagnostic_evidence_links
```

Requirements:

- use UUID primary keys;
- use `TIMESTAMPTZ`;
- store enum values as `VARCHAR` with PostgreSQL `CHECK` constraints;
- add application-level enum validation;
- add not-empty constraints for required strings and nullable strings when present;
- `operation_route_steps` must reference `operation_diagnostics`;
- `diagnostic_evidence_links` must have nullable FKs:
  - `operation_diagnostic_id`;
  - `operation_route_step_id`;
- `diagnostic_evidence_links` must enforce exactly one target;
- do not use string-only `targetId` polymorphic evidence links in MVP-1;
- do not add PII/raw payload columns.

Forbidden database columns:

```text
external_operation_id
payment_id
customer_id
client_id
customer_name
card_number
masked_card_number
phone
email
account_number
raw_request
raw_response
provider_payload
raw_provider_payload
raw_logs
raw_traces
stack_trace
exception
full_exception
full_error_message
```

---

## Required Endpoints

Implement:

```text
GET /api/technician/operations
GET /api/technician/operations/{operationId}
GET /api/technician/operations/{operationId}/route
GET /api/technician/operations/{operationId}/evidence
GET /api/technician/operations/{operationId}/route/evidence
GET /api/technician/route-steps/{routeStepId}/evidence
```

The grouped route-step evidence endpoint is required to avoid N+1 frontend calls:

```text
GET /api/technician/operations/{operationId}/route/evidence
```

---

## Required DTO Behavior

Route steps must expose:

```text
hasEvidence
evidenceCount
```

Evidence links must expose:

```text
evidenceLinkId
targetType
operationId
routeStepId
evidenceType
providerType
label
url
timeRangeFromUtc
timeRangeToUtc
```

Evidence links must not expose:

```text
operation_diagnostics.id
external_operation_reference_hash
```

---

## Required Frontend

Implement or update:

```text
/operations
/operations/{operationId}
OperationRouteTimeline
DiagnosticEvidencePanel
OperationStatusBadge
RouteStepStatusBadge
```

Behavior:

- backend APIs are the source of truth;
- filters work;
- pagination works;
- sorting works;
- route steps are ordered by `stepOrder`;
- failed step is highlighted;
- slowest step is highlighted;
- operation evidence links are shown;
- route-step evidence links are loaded through the grouped route evidence endpoint where possible;
- route-step evidence links are shown;
- route steps expose `hasEvidence` and `evidenceCount`.

---

## Hard Safety Constraints

Do not store or display:

```text
PII
raw logs
raw traces
raw request
raw response
raw provider payload
stack traces
full exception objects
customer fields
```

Do not implement:

```text
payment retry
operation replay
payment resubmission
provider mutation
external state-changing calls
```

Do not turn `MetricSnapshot` into a time-series database.

Do not expand `ServiceOperationalState` into operation diagnostics.

---

## Evidence URL Safety

Evidence URL validation must:

- require `https` in production;
- allow `http` only in local development or explicitly configured demo mode;
- reject `javascript:`, `data:`, `file:`, and other executable/local schemes;
- reject URLs containing PII/raw payload query parameters.

Forbidden URL parameter names include:

```text
customerName
cardNumber
maskedCardNumber
phone
email
passport
accountNumber
clientId
customerId
rawPayload
providerPayload
rawRequest
rawResponse
paymentData
personalData
```

Allowed technical parameters include:

```text
serviceKey
adapterKey
providerKey
correlationId
traceId
fromUtc
toUtc
dashboardUid
panelId
```

---

## Seed Data

Add at least 5 safe demo operations:

1. failed provider timeout;
2. slow success;
3. unknown provider result;
4. timed out;
5. success.

Seed data must include:

- ordered route steps;
- operation evidence links;
- route-step evidence links;
- no PII;
- no raw payloads.

---

## Testing

Add or update tests for:

- enum constraints;
- evidence exactly-one-target constraint;
- operation list API;
- operation details API;
- route ordering;
- operation evidence API;
- grouped route-step evidence API;
- route-step evidence API;
- filtering;
- sorting;
- evidence URL scheme validation;
- route steps expose `hasEvidence` and `evidenceCount`;
- `operation_diagnostics.id` is not returned by technician APIs;
- `external_operation_reference_hash` is not returned by technician APIs;
- forbidden fields are absent from DTOs.

---

## Completion Criteria

Stop when MVP-1 is implemented and tests/checks pass.

Do not continue into MVP-2 or MVP-3.

Do not implement ingestion.

Do not implement diagnostic checks.

Do not restructure the whole solution.
