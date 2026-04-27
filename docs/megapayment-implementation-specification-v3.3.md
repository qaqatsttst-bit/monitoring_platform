# Megapayment Implementation Specification v3.3

## 1. Purpose

This document is the corrected implementation-ready specification for the Megapayment diagnostic module.

It replaces the weaker parts of the previous implementation specification and resolves the following architectural issues:

1. enum values must be protected by database constraints, not only by documentation;
2. `DiagnosticEvidenceLink` must not be an unsafe generic polymorphic table without integrity rules;
3. route-step evidence must have a direct and consistent database representation;
4. `TargetType` must be explicitly defined;
5. `operationId` must be safe by design;
6. ingestion must not confuse MVP-1 implementation scope;
7. diagnostic checks must not accidentally enter MVP-1 scope;
8. implementation instructions must prevent future milestones from entering MVP-1;
9. API DTOs must avoid leaking internal operation table identifiers;
10. route-step evidence must be retrievable without N+1 frontend calls;
11. route steps must expose evidence indicators for the timeline UI;
12. nullable string fields must not contain empty strings when present;
13. `external_operation_reference_hash` must not be returned by technician APIs.

The project must be implemented as an internal diagnostic control plane for bank technical users.

Megapayment must help technical users investigate:

- degraded services;
- degraded bank/provider adapters;
- degraded external providers;
- failed or slow business operations;
- failed or slow route steps;
- technical evidence links to metrics, logs, traces, dashboards, alerts, and runbooks.

Megapayment must not store or display:

- customer personal data;
- raw provider payloads;
- raw requests;
- raw responses;
- raw logs;
- raw traces;
- raw span attributes;
- stack traces;
- full exception objects.

---

## 2. Critical Implementation Decisions

### 2.1. Enum Persistence Decision

Use `VARCHAR` columns with PostgreSQL `CHECK` constraints.

Do not use native PostgreSQL enum types for MVP.

Reason:

- native PostgreSQL enums are stricter but harder to rename/change during iterative development;
- plain `VARCHAR` without constraints is too weak;
- `VARCHAR` plus `CHECK` constraints gives a strong and maintainable balance;
- C# application code must also validate enum values before persistence and before returning DTOs.

Therefore:

```text
Application layer validation + PostgreSQL CHECK constraints are mandatory.
```

### 2.2. Evidence Link Decision

Do not use only generic `target_type + target_id` without database integrity.

For MVP-1, `diagnostic_evidence_links` must support only evidence links for:

- operation diagnostics;
- operation route steps.

Therefore, MVP-1 evidence links must have direct nullable foreign keys:

```text
operation_diagnostic_id
operation_route_step_id
```

A CHECK constraint must enforce that exactly one target foreign key is present.

Future milestones may extend this table with:

```text
service_id
adapter_id
provider_id
incident_id
event_id
```

or introduce typed relation tables if needed.

### 2.3. Route Step Evidence Decision

The relationship:

```text
OperationRouteStep 1..N DiagnosticEvidenceLink
```

must be represented directly in the database through:

```text
diagnostic_evidence_links.operation_route_step_id
```

Do not rely on a string-only `targetId` for route step evidence in MVP-1.

### 2.4. TargetType Decision

`TargetType` is a required explicit enum.

Allowed values:

```text
OPERATION
ROUTE_STEP
SERVICE
ADAPTER
PROVIDER
INCIDENT
EVENT
```

MVP-1 may persist only:

```text
OPERATION
ROUTE_STEP
```

Future milestones may enable the remaining values.

### 2.5. Operation Identifier Decision

Do not store real banking operation identifiers directly unless they are formally approved as safe.

MVP-1 must use generated safe diagnostic identifiers.

Database field:

```text
diagnostic_operation_id
```

API field exposed to frontend:

```text
operationId
```

Mapping rule:

```text
operationId in API = diagnostic_operation_id in database
```

If a real external operation reference must be correlated later, store only:

```text
external_operation_reference_hash
```

The hash must be produced using HMAC-SHA256 with a secret application key or another approved one-way keyed hashing mechanism.

Do not store:

```text
externalOperationId
paymentId
cardOperationId
customerOperationId
rawOperationReference
```

unless the value is proven to be non-sensitive and approved.

### 2.6. MVP-1 Scope Decision

MVP-1 must implement only:

- operation list;
- operation details;
- operation route timeline;
- operation and route-step evidence links;
- backend seed data;
- backend APIs;
- Blazor UI pages/components for operations.

MVP-1 must not implement:

- ingestion endpoints;
- diagnostic checks;
- real provider integrations;
- real observability API queries;
- audit workflow;
- role management UI;
- service/provider CRUD unless already implemented;
- major solution restructuring.

### 2.7. Ingestion Decision

Ingestion belongs to MVP-3.

Ingestion may be documented as a future design section, but MVP-1 implementation must not include ingestion.

### 2.8. Diagnostic Check Decision

Diagnostic checks belong to MVP-3.

MVP-1 implementation must not include:

```text
POST /api/technician/path-elements/{pathElementKey}/checks
```

during MVP-1.


### 2.9. MVP-1 Technician API Read-Only Decision

All `/api/technician` endpoints in MVP-1 must be read-only.

MVP-1 technician API may include only `GET` endpoints.

The following methods are not allowed for MVP-1 technician API:

```text
POST
PUT
PATCH
DELETE
```

Exception:

There is no exception in MVP-1.

Write-side APIs, ingestion APIs, diagnostic check APIs, and admin mutation APIs belong to future milestones only.

### 2.10. External Operation Hash Decision for MVP-1

`external_operation_reference_hash` exists for future real integrations.

For MVP-1 seed/demo data, it may be `NULL`.

HMAC-SHA256 hashing is required only when a real external operation reference is imported, ingested, or otherwise correlated in future milestones.

Do not implement real external operation reference hashing in MVP-1 unless explicitly requested.

### 2.11. Evidence URL Scheme Decision

Production evidence URLs must use HTTPS.

HTTP may be allowed only in local development or demo mode through explicit configuration.

Application-level validation must reject non-HTTPS evidence URLs in production.

### 2.12. Public DTO Identifier Decision

Technician APIs must not expose `operation_diagnostics.id`.

Technician APIs must expose the safe public operation identifier:

```text
operationId = operation_diagnostics.diagnostic_operation_id
```

Route step identifiers may be exposed as UUIDs because they are internal technical diagnostic identifiers and are needed for route-step evidence lookup.

`external_operation_reference_hash` must not be returned by MVP-1 technician APIs.

### 2.13. Route Evidence Loading Decision

The operation details page must be able to load route-step evidence without one request per route step.

Therefore, MVP-1 must include an aggregate endpoint:

```text
GET /api/technician/operations/{operationId}/route/evidence
```

The per-step endpoint may remain available, but the aggregate endpoint is preferred for the operation details page.


### 2.14. Cursor Prompt Separation Decision

The implementation specification is the source of truth.

The Cursor prompt must be stored separately as an execution instruction.

Canonical Cursor prompt path:

```text
docs/cursor-prompts/mvp-1-implementation.md
```

The prompt must tell Cursor what to implement, what not to implement, and which specification file is the source of truth.

The prompt must not replace this specification.

---

## 3. MVP-1 Functional Scope

### 3.1. In Scope

MVP-1 must include:

- backend-driven operation list;
- operation details page;
- ordered operation route steps;
- route timeline component;
- evidence links for operations;
- evidence links for route steps;
- safe demo seed data from backend;
- filtering by operation status, provider, service, operation type, error code, failed stage, and time range;
- pagination;
- sorting;
- safe DTOs only.

### 3.2. Out of Scope

MVP-1 must not include:

- ingestion endpoints;
- diagnostic check execution;
- external provider calls;
- real Prometheus/Loki/Tempo/Jaeger API queries;
- payment retry;
- operation replay;
- payment resubmission;
- customer data lookup;
- raw logs viewer;
- raw traces viewer;
- service/provider CRUD unless already present;
- full incident workflow;
- role management UI.

### 3.3. MVP-1 Required Pages

```text
/operations
/operations/{operationId}
```

### 3.4. MVP-1 Required Components

```text
OperationRouteTimeline
DiagnosticEvidencePanel
OperationStatusBadge
RouteStepStatusBadge
```

### 3.5. MVP-1 Required Backend Endpoints

```text
GET /api/technician/operations
GET /api/technician/operations/{operationId}
GET /api/technician/operations/{operationId}/route
GET /api/technician/operations/{operationId}/evidence
GET /api/technician/operations/{operationId}/route/evidence
GET /api/technician/route-steps/{routeStepId}/evidence
```

`routeStepId` is the UUID of the route step.

---

## 4. Enum Definitions

All enum values must use `UPPER_SNAKE_CASE`.

### 4.1. TargetType

```text
OPERATION
ROUTE_STEP
SERVICE
ADAPTER
PROVIDER
INCIDENT
EVENT
```

### 4.2. OperationType

```text
PAYMENT
TRANSFER
CARD_PAYMENT
WALLET_PAYMENT
PROVIDER_CALLBACK
STATUS_UPDATE
UNKNOWN
```

### 4.3. OperationTechnicalStatus

```text
PENDING
PROCESSING
SUCCEEDED
FAILED
TIMED_OUT
UNKNOWN
```

### 4.4. RouteStepType

```text
SERVICE
BANK_ADAPTER
PROVIDER_ADAPTER
EXTERNAL_PROVIDER
CALLBACK_HANDLER
STATUS_UPDATER
FRAUD_CHECK
AML_CHECK
INTERNAL_COMPONENT
```

### 4.5. RouteStepStatus

```text
OK
DEGRADED
FAILED
SKIPPED
TIMED_OUT
UNKNOWN
```

### 4.6. EvidenceType

```text
METRICS
LOGS
TRACE
DASHBOARD
ALERT
RUNBOOK
INTERNAL_LINK
```

### 4.7. EvidenceProviderType

```text
PROMETHEUS
GRAFANA
LOKI
TEMPO
JAEGER
ALERTMANAGER
INTERNAL_DASHBOARD
INTERNAL_RUNBOOK
```

### 4.8. ServiceStatus

```text
OK
DEGRADED
DOWN
NO_DATA
UNKNOWN
```

### 4.9. FreshnessStatus

```text
FRESH
STALE
EXPIRED
NO_DATA
UNKNOWN
```

### 4.10. AdapterType

```text
BANK_ADAPTER
PROVIDER_ADAPTER
INTERNAL_ADAPTER
OBSERVABILITY_ADAPTER
```

### 4.11. ProviderStatus

```text
OK
DEGRADED
DOWN
NO_DATA
UNKNOWN
```

### 4.12. DiagnosticCheckType

Future only. Do not implement in MVP-1.

```text
HEALTHCHECK
STATUS_CHECK
LATENCY_CHECK
AVAILABILITY_CHECK
SAFE_OPERATION_LOOKUP
SAFE_PROVIDER_STATUS_LOOKUP
SAFE_ADAPTER_STATUS_LOOKUP
OBSERVABILITY_SOURCE_REACHABILITY
```

### 4.13. DiagnosticCheckStatus

Future only. Do not implement in MVP-1.

```text
OK
DEGRADED
FAILED
TIMED_OUT
NOT_SUPPORTED
UNKNOWN
```

---

## 5. Error Code Catalog for MVP-1

`error_code` remains a flexible `VARCHAR(128)` field because real providers and adapters may introduce new technical codes.

However, MVP-1 seed data and UI examples should use the following recommended codes.

### 5.1. Recommended Operation Error Codes

```text
PROVIDER_TIMEOUT
PROVIDER_UNAVAILABLE
PROVIDER_RESULT_UNKNOWN
ADAPTER_TIMEOUT
ADAPTER_UNAVAILABLE
OPERATION_TIMEOUT
ROUTE_STEP_TIMEOUT
TRACE_NOT_FOUND
EVIDENCE_LINK_UNAVAILABLE
UNKNOWN_ERROR
```

### 5.2. Error Code Rules

`error_code` must be:

- technical;
- safe;
- non-PII;
- stable enough for filtering;
- suitable for displaying to internal technical users.

`error_code` must not contain:

- raw exception text;
- provider response body;
- customer data;
- card/payment personal data;
- stack trace fragments.

### 5.3. Database Constraint Decision for Error Codes

Do not add a database `CHECK` constraint for `error_code` in MVP-1.

Reason:

- providers and adapters may produce new safe technical codes;
- strict database constraints would slow future integration;
- application and seed data should still prefer the recommended catalog.

---

## 6. PostgreSQL Schema for MVP-1

### 6.1. Design Rules

MVP-1 schema must:

- use UUID primary keys;
- use `TIMESTAMPTZ` for timestamps;
- use `VARCHAR` enum columns with `CHECK` constraints;
- avoid raw payload columns;
- avoid PII columns;
- use generated safe operation identifiers;
- maintain referential integrity for route steps and evidence links;
- support pagination, filtering, and sorting.

### 6.2. Table: operation_diagnostics

Purpose:

Stores safe technical diagnostics for one operation.

```sql
CREATE TABLE operation_diagnostics (
    id UUID PRIMARY KEY,
    diagnostic_operation_id VARCHAR(128) NOT NULL UNIQUE,
    external_operation_reference_hash VARCHAR(256),
    operation_type VARCHAR(64) NOT NULL,
    technical_status VARCHAR(64) NOT NULL,
    provider_key VARCHAR(128),
    service_key VARCHAR(128),
    failed_stage VARCHAR(128),
    error_code VARCHAR(128),
    safe_error_message VARCHAR(500),
    latency_ms INTEGER,
    correlation_id VARCHAR(128),
    trace_id VARCHAR(128),
    started_at_utc TIMESTAMPTZ,
    completed_at_utc TIMESTAMPTZ,
    created_at_utc TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT chk_operation_type
        CHECK (operation_type IN (
            'PAYMENT',
            'TRANSFER',
            'CARD_PAYMENT',
            'WALLET_PAYMENT',
            'PROVIDER_CALLBACK',
            'STATUS_UPDATE',
            'UNKNOWN'
        )),

    CONSTRAINT chk_operation_technical_status
        CHECK (technical_status IN (
            'PENDING',
            'PROCESSING',
            'SUCCEEDED',
            'FAILED',
            'TIMED_OUT',
            'UNKNOWN'
        )),

    CONSTRAINT chk_diagnostic_operation_id_not_empty
        CHECK (length(trim(diagnostic_operation_id)) > 0),

    CONSTRAINT chk_external_operation_reference_hash_not_empty
        CHECK (
            external_operation_reference_hash IS NULL
            OR length(trim(external_operation_reference_hash)) > 0
        ),

    CONSTRAINT chk_operation_provider_key_not_empty
        CHECK (provider_key IS NULL OR length(trim(provider_key)) > 0),

    CONSTRAINT chk_operation_service_key_not_empty
        CHECK (service_key IS NULL OR length(trim(service_key)) > 0),

    CONSTRAINT chk_operation_failed_stage_not_empty
        CHECK (failed_stage IS NULL OR length(trim(failed_stage)) > 0),

    CONSTRAINT chk_operation_error_code_not_empty
        CHECK (error_code IS NULL OR length(trim(error_code)) > 0),

    CONSTRAINT chk_operation_correlation_id_not_empty
        CHECK (correlation_id IS NULL OR length(trim(correlation_id)) > 0),

    CONSTRAINT chk_operation_trace_id_not_empty
        CHECK (trace_id IS NULL OR length(trim(trace_id)) > 0),

    CONSTRAINT chk_operation_latency_non_negative
        CHECK (latency_ms IS NULL OR latency_ms >= 0),

    CONSTRAINT chk_operation_time_order
        CHECK (
            started_at_utc IS NULL
            OR completed_at_utc IS NULL
            OR completed_at_utc >= started_at_utc
        ),

    CONSTRAINT chk_operation_safe_message_not_empty
        CHECK (safe_error_message IS NULL OR length(trim(safe_error_message)) > 0),

    CONSTRAINT chk_operation_safe_message_length
        CHECK (safe_error_message IS NULL OR length(safe_error_message) <= 500)
);
```

Indexes:

```sql
CREATE INDEX ix_operation_diagnostics_status
    ON operation_diagnostics (technical_status);

CREATE INDEX ix_operation_diagnostics_provider
    ON operation_diagnostics (provider_key);

CREATE INDEX ix_operation_diagnostics_service
    ON operation_diagnostics (service_key);

CREATE INDEX ix_operation_diagnostics_type
    ON operation_diagnostics (operation_type);

CREATE INDEX ix_operation_diagnostics_error_code
    ON operation_diagnostics (error_code);

CREATE INDEX ix_operation_diagnostics_failed_stage
    ON operation_diagnostics (failed_stage);

CREATE INDEX ix_operation_diagnostics_started_at
    ON operation_diagnostics (started_at_utc DESC);

CREATE INDEX ix_operation_diagnostics_correlation_id
    ON operation_diagnostics (correlation_id);

CREATE INDEX ix_operation_diagnostics_trace_id
    ON operation_diagnostics (trace_id);
```

Forbidden columns:

```text
operation_id from external payment system
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

### 6.3. Table: operation_route_steps

Purpose:

Stores ordered technical route steps for an operation.

```sql
CREATE TABLE operation_route_steps (
    id UUID PRIMARY KEY,
    operation_diagnostic_id UUID NOT NULL REFERENCES operation_diagnostics(id) ON DELETE CASCADE,
    step_order INTEGER NOT NULL,
    step_name VARCHAR(200) NOT NULL,
    step_type VARCHAR(64) NOT NULL,
    service_key VARCHAR(128),
    adapter_key VARCHAR(128),
    provider_key VARCHAR(128),
    status VARCHAR(64) NOT NULL,
    latency_ms INTEGER,
    error_code VARCHAR(128),
    safe_error_message VARCHAR(500),
    correlation_id VARCHAR(128),
    trace_id VARCHAR(128),
    started_at_utc TIMESTAMPTZ,
    completed_at_utc TIMESTAMPTZ,
    created_at_utc TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT uq_operation_route_step_order
        UNIQUE (operation_diagnostic_id, step_order),

    CONSTRAINT chk_route_step_order_positive
        CHECK (step_order > 0),

    CONSTRAINT chk_route_step_name_not_empty
        CHECK (length(trim(step_name)) > 0),

    CONSTRAINT chk_route_step_service_key_not_empty
        CHECK (service_key IS NULL OR length(trim(service_key)) > 0),

    CONSTRAINT chk_route_step_adapter_key_not_empty
        CHECK (adapter_key IS NULL OR length(trim(adapter_key)) > 0),

    CONSTRAINT chk_route_step_provider_key_not_empty
        CHECK (provider_key IS NULL OR length(trim(provider_key)) > 0),

    CONSTRAINT chk_route_step_error_code_not_empty
        CHECK (error_code IS NULL OR length(trim(error_code)) > 0),

    CONSTRAINT chk_route_step_correlation_id_not_empty
        CHECK (correlation_id IS NULL OR length(trim(correlation_id)) > 0),

    CONSTRAINT chk_route_step_trace_id_not_empty
        CHECK (trace_id IS NULL OR length(trim(trace_id)) > 0),

    CONSTRAINT chk_route_step_type
        CHECK (step_type IN (
            'SERVICE',
            'BANK_ADAPTER',
            'PROVIDER_ADAPTER',
            'EXTERNAL_PROVIDER',
            'CALLBACK_HANDLER',
            'STATUS_UPDATER',
            'FRAUD_CHECK',
            'AML_CHECK',
            'INTERNAL_COMPONENT'
        )),

    CONSTRAINT chk_route_step_status
        CHECK (status IN (
            'OK',
            'DEGRADED',
            'FAILED',
            'SKIPPED',
            'TIMED_OUT',
            'UNKNOWN'
        )),

    CONSTRAINT chk_route_step_latency_non_negative
        CHECK (latency_ms IS NULL OR latency_ms >= 0),

    CONSTRAINT chk_route_step_time_order
        CHECK (
            started_at_utc IS NULL
            OR completed_at_utc IS NULL
            OR completed_at_utc >= started_at_utc
        ),

    CONSTRAINT chk_route_step_safe_message_not_empty
        CHECK (safe_error_message IS NULL OR length(trim(safe_error_message)) > 0),

    CONSTRAINT chk_route_step_safe_message_length
        CHECK (safe_error_message IS NULL OR length(safe_error_message) <= 500)
);
```

Indexes:

```sql
CREATE INDEX ix_route_steps_operation
    ON operation_route_steps (operation_diagnostic_id);

CREATE INDEX ix_route_steps_operation_order
    ON operation_route_steps (operation_diagnostic_id, step_order);

CREATE INDEX ix_route_steps_status
    ON operation_route_steps (status);

CREATE INDEX ix_route_steps_type
    ON operation_route_steps (step_type);

CREATE INDEX ix_route_steps_service
    ON operation_route_steps (service_key);

CREATE INDEX ix_route_steps_adapter
    ON operation_route_steps (adapter_key);

CREATE INDEX ix_route_steps_provider
    ON operation_route_steps (provider_key);
```

### 6.4. Table: diagnostic_evidence_links

Purpose:

Stores safe evidence links for operations and route steps.

This table must not be generic without integrity. In MVP-1 it must use typed nullable foreign keys.

```sql
CREATE TABLE diagnostic_evidence_links (
    id UUID PRIMARY KEY,

    target_type VARCHAR(64) NOT NULL,

    operation_diagnostic_id UUID NULL REFERENCES operation_diagnostics(id) ON DELETE CASCADE,
    operation_route_step_id UUID NULL REFERENCES operation_route_steps(id) ON DELETE CASCADE,

    evidence_type VARCHAR(64) NOT NULL,
    provider_type VARCHAR(64) NOT NULL,
    label VARCHAR(200) NOT NULL,
    url TEXT NOT NULL,
    time_range_from_utc TIMESTAMPTZ,
    time_range_to_utc TIMESTAMPTZ,
    created_at_utc TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT chk_evidence_target_type_mvp1
        CHECK (target_type IN (
            'OPERATION',
            'ROUTE_STEP'
        )),

    CONSTRAINT chk_evidence_exactly_one_target_mvp1
        CHECK (
            (
                operation_diagnostic_id IS NOT NULL
                AND operation_route_step_id IS NULL
                AND target_type = 'OPERATION'
            )
            OR
            (
                operation_diagnostic_id IS NULL
                AND operation_route_step_id IS NOT NULL
                AND target_type = 'ROUTE_STEP'
            )
        ),

    CONSTRAINT chk_evidence_type
        CHECK (evidence_type IN (
            'METRICS',
            'LOGS',
            'TRACE',
            'DASHBOARD',
            'ALERT',
            'RUNBOOK',
            'INTERNAL_LINK'
        )),

    CONSTRAINT chk_evidence_provider_type
        CHECK (provider_type IN (
            'PROMETHEUS',
            'GRAFANA',
            'LOKI',
            'TEMPO',
            'JAEGER',
            'ALERTMANAGER',
            'INTERNAL_DASHBOARD',
            'INTERNAL_RUNBOOK'
        )),

    CONSTRAINT chk_evidence_time_order
        CHECK (
            time_range_from_utc IS NULL
            OR time_range_to_utc IS NULL
            OR time_range_to_utc >= time_range_from_utc
        ),

    CONSTRAINT chk_evidence_label_not_empty
        CHECK (length(trim(label)) > 0),

    CONSTRAINT chk_evidence_url_not_empty
        CHECK (length(trim(url)) > 0)
);
```

URL scheme and forbidden parameter validation must be enforced in application code because it is environment-dependent.

Indexes:

```sql
CREATE INDEX ix_evidence_links_operation
    ON diagnostic_evidence_links (operation_diagnostic_id)
    WHERE operation_diagnostic_id IS NOT NULL;

CREATE INDEX ix_evidence_links_route_step
    ON diagnostic_evidence_links (operation_route_step_id)
    WHERE operation_route_step_id IS NOT NULL;

CREATE INDEX ix_evidence_links_target_type
    ON diagnostic_evidence_links (target_type);

CREATE INDEX ix_evidence_links_evidence_type
    ON diagnostic_evidence_links (evidence_type);

CREATE INDEX ix_evidence_links_provider_type
    ON diagnostic_evidence_links (provider_type);
```

### 6.5. Future Evidence Target Expansion

When MVP-2/MVP-3 introduces service/provider/adapter/incident evidence, use one of these approaches.

Preferred approach:

```sql
ALTER TABLE diagnostic_evidence_links
ADD COLUMN service_id UUID NULL REFERENCES services(id) ON DELETE CASCADE,
ADD COLUMN adapter_id UUID NULL REFERENCES adapters(id) ON DELETE CASCADE,
ADD COLUMN provider_id UUID NULL REFERENCES providers(id) ON DELETE CASCADE,
ADD COLUMN incident_id UUID NULL REFERENCES incidents(id) ON DELETE CASCADE,
ADD COLUMN event_id UUID NULL REFERENCES events(id) ON DELETE CASCADE;
```

Then replace the MVP-1 target constraint with an expanded exactly-one-target constraint.

Alternative approach:

Use separate typed relation tables:

```text
operation_evidence_links
route_step_evidence_links
service_evidence_links
provider_evidence_links
incident_evidence_links
```

Do not use string-only polymorphic links if referential integrity matters.

---

## 7. DTO Specification for MVP-1

### 7.1. OperationListItemDto

```csharp
public sealed class OperationListItemDto
{
    public required string OperationId { get; init; } // Maps to diagnostic_operation_id
    public required string OperationType { get; init; }
    public required string TechnicalStatus { get; init; }
    public string? ProviderKey { get; init; }
    public string? ServiceKey { get; init; }
    public string? FailedStage { get; init; }
    public string? ErrorCode { get; init; }
    public int? LatencyMs { get; init; }
    public string? CorrelationId { get; init; }
    public string? TraceId { get; init; }
    public DateTimeOffset? StartedAtUtc { get; init; }
    public DateTimeOffset? CompletedAtUtc { get; init; }
}
```

### 7.2. OperationDetailsDto

```csharp
public sealed class OperationDetailsDto
{
    public required string OperationId { get; init; } // Maps to diagnostic_operation_id
    public required string OperationType { get; init; }
    public required string TechnicalStatus { get; init; }
    public string? ProviderKey { get; init; }
    public string? ServiceKey { get; init; }
    public string? FailedStage { get; init; }
    public string? ErrorCode { get; init; }
    public string? SafeErrorMessage { get; init; }
    public int? LatencyMs { get; init; }
    public string? CorrelationId { get; init; }
    public string? TraceId { get; init; }
    public DateTimeOffset? StartedAtUtc { get; init; }
    public DateTimeOffset? CompletedAtUtc { get; init; }
    public DateTimeOffset CreatedAtUtc { get; init; }
}
```

### 7.3. OperationRouteStepDto

```csharp
public sealed class OperationRouteStepDto
{
    public required Guid RouteStepId { get; init; }
    public required int StepOrder { get; init; }
    public required string StepName { get; init; }
    public required string StepType { get; init; }
    public string? ServiceKey { get; init; }
    public string? AdapterKey { get; init; }
    public string? ProviderKey { get; init; }
    public required string Status { get; init; }
    public int? LatencyMs { get; init; }
    public string? ErrorCode { get; init; }
    public string? SafeErrorMessage { get; init; }
    public string? CorrelationId { get; init; }
    public string? TraceId { get; init; }
    public DateTimeOffset? StartedAtUtc { get; init; }
    public DateTimeOffset? CompletedAtUtc { get; init; }
    public required bool HasEvidence { get; init; }
    public required int EvidenceCount { get; init; }
}
```

`RouteStepId` is required because route-step evidence links are now directly related to route steps.

### 7.4. DiagnosticEvidenceLinkDto

```csharp
public sealed class DiagnosticEvidenceLinkDto
{
    public required Guid EvidenceLinkId { get; init; }
    public required string TargetType { get; init; }
    public string? OperationId { get; init; } // Maps to diagnostic_operation_id when TargetType = OPERATION
    public Guid? RouteStepId { get; init; } // Maps to operation_route_steps.id when TargetType = ROUTE_STEP
    public required string EvidenceType { get; init; }
    public required string ProviderType { get; init; }
    public required string Label { get; init; }
    public required string Url { get; init; }
    public DateTimeOffset? TimeRangeFromUtc { get; init; }
    public DateTimeOffset? TimeRangeToUtc { get; init; }
}
```


### 7.5. RouteStepEvidenceGroupDto

```csharp
public sealed class RouteStepEvidenceGroupDto
{
    public required Guid RouteStepId { get; init; }
    public required IReadOnlyList<DiagnosticEvidenceLinkDto> EvidenceLinks { get; init; }
}
```

### 7.6. PagedResult

```csharp
public sealed class PagedResult<T>
{
    public required IReadOnlyList<T> Items { get; init; }
    public required int Page { get; init; }
    public required int PageSize { get; init; }
    public required int TotalCount { get; init; }
}
```

### 7.7. OperationFilterRequest

```csharp
public sealed class OperationFilterRequest
{
    public string? Status { get; init; }
    public string? ProviderKey { get; init; }
    public string? ServiceKey { get; init; }
    public string? OperationType { get; init; }
    public string? ErrorCode { get; init; }
    public string? FailedStage { get; init; }
    public DateTimeOffset? FromUtc { get; init; }
    public DateTimeOffset? ToUtc { get; init; }
    public int Page { get; init; } = 1;
    public int PageSize { get; init; } = 20;
    public string SortBy { get; init; } = "startedAtUtc";
    public string SortDirection { get; init; } = "desc";
}
```

Allowed `SortBy` values:

```text
startedAtUtc
completedAtUtc
latencyMs
technicalStatus
operationType
providerKey
serviceKey
errorCode
```

Allowed `SortDirection` values:

```text
asc
desc
```

Default:

```text
sortBy=startedAtUtc
sortDirection=desc
```


Validation rules:

```text
page must be >= 1
pageSize must be between 1 and 100
sortBy must be one of the allowed SortBy values
sortDirection must be asc or desc
fromUtc must be earlier than or equal to toUtc when both are provided
```

Recommended default SQL ordering:

```sql
ORDER BY started_at_utc DESC NULLS LAST, created_at_utc DESC
```

Recommended latency ordering:

```sql
ORDER BY latency_ms DESC NULLS LAST, created_at_utc DESC
```

---

## 8. API Specification for MVP-1

### 8.1. Common Rules

All endpoints must:

- return safe DTOs only;
- use ISO-8601 UTC timestamps;
- use `UPPER_SNAKE_CASE` enum values;
- avoid raw payloads;
- avoid PII;
- return consistent error responses.


MVP-1 technician API is read-only:

```text
Only GET /api/technician/* endpoints are allowed in MVP-1.
No POST, PUT, PATCH, or DELETE technician endpoint is allowed in MVP-1.
```


Public identifier rule:

```text
operation_diagnostics.id must not be returned by technician APIs
external_operation_reference_hash must not be returned by technician APIs
operationId in API must map to diagnostic_operation_id
```

### 8.2. Common Error Response

```json
{
  "errorCode": "VALIDATION_FAILED",
  "message": "Request validation failed",
  "details": [
    {
      "field": "status",
      "reason": "Unsupported status value"
    }
  ],
  "correlationId": "corr-demo-001"
}
```

### 8.3. GET /api/technician/operations

Purpose:

Returns paginated operation diagnostics.

Query parameters:

| Parameter | Type | Required | Description |
|---|---|---:|---|
| status | string | no | OperationTechnicalStatus |
| providerKey | string | no | Provider filter |
| serviceKey | string | no | Service filter |
| operationType | string | no | OperationType |
| errorCode | string | no | Error code filter |
| failedStage | string | no | Failed stage filter |
| fromUtc | datetime | no | Start time |
| toUtc | datetime | no | End time |
| page | integer | no | Default 1 |
| pageSize | integer | no | Default 20, max 100 |
| sortBy | string | no | Default startedAtUtc |
| sortDirection | string | no | Default desc |

Response:

```json
{
  "items": [
    {
      "operationId": "op-demo-001",
      "operationType": "TRANSFER",
      "technicalStatus": "FAILED",
      "providerKey": "provider-main",
      "serviceKey": "payment-orchestrator",
      "failedStage": "PROVIDER_AUTHORIZATION",
      "errorCode": "PROVIDER_TIMEOUT",
      "latencyMs": 15420,
      "correlationId": "corr-demo-001",
      "traceId": "trace-demo-001",
      "startedAtUtc": "2026-04-27T10:00:02Z",
      "completedAtUtc": "2026-04-27T10:00:17Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 1
}
```

### 8.4. GET /api/technician/operations/{operationId}

`operationId` is the safe generated diagnostic operation identifier.

It maps to:

```text
operation_diagnostics.diagnostic_operation_id
```

Response:

```json
{
  "operationId": "op-demo-001",
  "operationType": "TRANSFER",
  "technicalStatus": "FAILED",
  "providerKey": "provider-main",
  "serviceKey": "payment-orchestrator",
  "failedStage": "PROVIDER_AUTHORIZATION",
  "errorCode": "PROVIDER_TIMEOUT",
  "safeErrorMessage": "Provider did not respond within timeout",
  "latencyMs": 15420,
  "correlationId": "corr-demo-001",
  "traceId": "trace-demo-001",
  "startedAtUtc": "2026-04-27T10:00:02Z",
  "completedAtUtc": "2026-04-27T10:00:17Z",
  "createdAtUtc": "2026-04-27T10:00:18Z"
}
```

### 8.5. GET /api/technician/operations/{operationId}/route

Returns route steps ordered by `stepOrder`.

Response:

```json
[
  {
    "routeStepId": "9fdfe90f-a9a5-4e62-93c1-a89a63761a17",
    "stepOrder": 1,
    "stepName": "API Gateway",
    "stepType": "SERVICE",
    "serviceKey": "api-gateway",
    "adapterKey": null,
    "providerKey": null,
    "status": "OK",
    "latencyMs": 40,
    "errorCode": null,
    "safeErrorMessage": null,
    "correlationId": "corr-demo-001",
    "traceId": "trace-demo-001",
    "startedAtUtc": "2026-04-27T10:00:02Z",
    "completedAtUtc": "2026-04-27T10:00:02.040Z",
    "hasEvidence": true,
    "evidenceCount": 2
  }
]
```

### 8.6. GET /api/technician/operations/{operationId}/evidence

Returns operation-level evidence links.

Only links with:

```text
target_type = OPERATION
operation_diagnostic_id = current operation id
```

must be returned.

Response:

```json
[
  {
    "evidenceLinkId": "5302ac77-5ec6-4c87-9d4b-3f0b5a520201",
    "targetType": "OPERATION",
    "operationId": "op-demo-001",
    "routeStepId": null,
    "evidenceType": "TRACE",
    "providerType": "TEMPO",
    "label": "Open distributed trace",
    "url": "https://tempo.example.local/trace/trace-demo-001",
    "timeRangeFromUtc": "2026-04-27T09:55:00Z",
    "timeRangeToUtc": "2026-04-27T10:05:00Z"
  }
]
```

### 8.7. GET /api/technician/operations/{operationId}/route/evidence

Returns route-step evidence grouped by route step for the selected operation.

This endpoint is preferred by the operation details page because it avoids N+1 frontend calls.

Response:

```json
[
  {
    "routeStepId": "9fdfe90f-a9a5-4e62-93c1-a89a63761a17",
    "evidenceLinks": [
      {
        "evidenceLinkId": "f62d1c6c-d78c-4e82-bf55-dfc5ee923df1",
        "targetType": "ROUTE_STEP",
        "operationId": null,
        "routeStepId": "9fdfe90f-a9a5-4e62-93c1-a89a63761a17",
        "evidenceType": "METRICS",
        "providerType": "GRAFANA",
        "label": "Open provider adapter latency dashboard",
        "url": "https://grafana.example.local/d/provider-adapter-latency?providerKey=provider-main",
        "timeRangeFromUtc": "2026-04-27T09:55:00Z",
        "timeRangeToUtc": "2026-04-27T10:05:00Z"
      }
    ]
  }
]
```

### 8.8. GET /api/technician/route-steps/{routeStepId}/evidence

Returns route-step-level evidence links.

Only links with:

```text
target_type = ROUTE_STEP
operation_route_step_id = routeStepId
```

must be returned.

Response:

```json
[
  {
    "evidenceLinkId": "f62d1c6c-d78c-4e82-bf55-dfc5ee923df1",
    "targetType": "ROUTE_STEP",
    "operationId": null,
    "routeStepId": "9fdfe90f-a9a5-4e62-93c1-a89a63761a17",
    "evidenceType": "METRICS",
    "providerType": "GRAFANA",
    "label": "Open provider adapter latency dashboard",
    "url": "https://grafana.example.local/d/provider-adapter-latency?providerKey=provider-main",
    "timeRangeFromUtc": "2026-04-27T09:55:00Z",
    "timeRangeToUtc": "2026-04-27T10:05:00Z"
  }
]
```

---

## 9. Validation Rules

### 9.1. Application-Level Enum Validation

Application code must validate enum values before persistence.

Invalid values must produce:

```text
400 VALIDATION_FAILED
```

Do not rely only on database exceptions.

### 9.2. Operation Validation

Required:

- `diagnosticOperationId`;
- `operationType`;
- `technicalStatus`;
- `createdAtUtc`.

Rules:

- `diagnosticOperationId` must be generated safe identifier;
- `diagnosticOperationId` must be unique;
- `externalOperationReferenceHash` is optional and must not expose original operation ID;
- `externalOperationReferenceHash` may be null for MVP-1 seed/demo data;
- `operationType` must be valid;
- `technicalStatus` must be valid;
- `latencyMs` must be null or non-negative;
- `completedAtUtc` must not be earlier than `startedAtUtc`;
- `safeErrorMessage` must be sanitized;
- nullable string fields must be either null or non-empty after trimming;
- `externalOperationReferenceHash` must not be returned by technician APIs.

### 9.3. Route Step Validation

Required:

- `operationDiagnosticId`;
- `stepOrder`;
- `stepName`;
- `stepType`;
- `status`.

Rules:

- `stepOrder` must be greater than 0;
- `(operationDiagnosticId, stepOrder)` must be unique;
- `stepType` must be valid;
- `status` must be valid;
- `latencyMs` must be null or non-negative;
- `completedAtUtc` must not be earlier than `startedAtUtc`;
- nullable string fields must be either null or non-empty after trimming;

### 9.4. Evidence Link Validation

Required:

- `targetType`;
- exactly one target foreign key;
- `evidenceType`;
- `providerType`;
- `label`;
- `url`.

Rules:

- `targetType = OPERATION` requires `operationDiagnosticId`;
- `targetType = ROUTE_STEP` requires `operationRouteStepId`;
- exactly one target foreign key must be non-null;
- `evidenceType` must be valid;
- `providerType` must be valid;
- URL must be safe;
- URL must not contain PII;
- URL must not contain raw payload data;
- time range must be valid.

### 9.5. Safe URL Validation

Reject evidence URLs containing query parameters or path fragments that indicate PII or raw data.

Forbidden parameter names include:

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

MVP may allow placeholder demo links, but they still must not contain forbidden parameters.


Identifier safety rule:

```text
correlationId and traceId must be generated technical identifiers
correlationId and traceId must not contain customer data
correlationId and traceId must not be copied from raw payloads without validation
```


Production URL scheme rule:

```text
https is required in production
http is allowed only in local development or explicitly configured demo mode
javascript:, data:, file:, and other executable/local schemes are forbidden
```

---

## 10. Frontend Specification for MVP-1

### 10.1. Operations Page

Route:

```text
/operations
```

Required behavior:

- load operation list from backend;
- support filters;
- support pagination;
- support sorting;
- display safe fields only;
- open operation details page.

Required columns:

```text
Operation ID
Type
Status
Provider
Service
Failed Stage
Error Code
Latency
Started At
Completed At
Actions
```

Forbidden UI fields:

```text
customerName
cardNumber
maskedCardNumber
phone
email
accountNumber
rawRequest
rawResponse
providerPayload
stackTrace
exception
```

### 10.2. Operation Details Page

Route:

```text
/operations/{operationId}
```

Required sections:

1. operation summary;
2. technical status;
3. failure summary;
4. correlation identifiers;
5. route timeline;
6. operation evidence links;
7. route-step evidence links if available.

### 10.3. OperationRouteTimeline Component

Required display:

- route step ID hidden or available only for internal actions;
- step order;
- step name;
- step type;
- status;
- latency;
- error code;
- safe error message;
- evidence link indicator.

Required highlighting:

- failed step;
- timed-out step;
- slowest step;
- skipped step.


MVP-1 calculation rule:

```text
Frontend may compute failed step and slowest step from the loaded route steps.
Backend route summary fields may be added later if needed.
```

Example:

```text
1 API Gateway              OK         40 ms
2 Payment Orchestrator     OK         120 ms
3 Bank Internal Adapter    OK         200 ms
4 Fraud/AML Service        OK         300 ms
5 Provider Adapter         FAILED     15000 ms
6 External Provider        UNKNOWN    -
7 Callback Handler         SKIPPED    -
8 Status Updater           OK         30 ms
```

### 10.4. DiagnosticEvidencePanel Component

Required display:

- label;
- evidence type;
- provider type;
- time range;
- open link action.

The component must support:

- operation-level evidence;
- route-step-level evidence.

Operation details page should load route-step evidence using:

```text
GET /api/technician/operations/{operationId}/route/evidence
```

The per-route-step evidence endpoint should be used only for direct drill-down or fallback behavior.

---

## 11. Seed Data Requirements

MVP-1 must include backend seed data.

Seed data must contain no PII and no raw payloads.

### 11.1. Demo Operation 1: Failed Provider Timeout

Operation:

```text
diagnosticOperationId: op-demo-001
operationType: TRANSFER
technicalStatus: FAILED
providerKey: provider-main
serviceKey: payment-orchestrator
failedStage: PROVIDER_AUTHORIZATION
errorCode: PROVIDER_TIMEOUT
safeErrorMessage: Provider did not respond within timeout
latencyMs: 15420
correlationId: corr-demo-001
traceId: trace-demo-001
```

Route:

```text
1 API Gateway              OK         40 ms
2 Payment Orchestrator     OK         120 ms
3 Bank Internal Adapter    OK         200 ms
4 Fraud/AML Service        OK         300 ms
5 Provider Adapter         FAILED     15000 ms
6 External Provider        UNKNOWN    -
7 Callback Handler         SKIPPED    -
8 Status Updater           OK         30 ms
```

Operation evidence:

```text
TRACE     TEMPO      Open distributed trace
LOGS      LOKI       Open related logs by correlationId
```

Route-step evidence for Provider Adapter:

```text
METRICS   GRAFANA    Open provider adapter latency dashboard
DASHBOARD GRAFANA    Open provider adapter dashboard
```

### 11.2. Demo Operation 2: Slow Success

Operation:

```text
diagnosticOperationId: op-demo-002
operationType: PAYMENT
technicalStatus: SUCCEEDED
providerKey: provider-wallet
serviceKey: payment-orchestrator
failedStage: null
errorCode: null
safeErrorMessage: null
latencyMs: 9800
correlationId: corr-demo-002
traceId: trace-demo-002
```

Route must contain one `DEGRADED` slow step and final success.

### 11.3. Demo Operation 3: Unknown Provider Result

Operation:

```text
diagnosticOperationId: op-demo-003
operationType: CARD_PAYMENT
technicalStatus: UNKNOWN
providerKey: card-provider-main
serviceKey: payment-orchestrator
failedStage: EXTERNAL_PROVIDER
errorCode: PROVIDER_RESULT_UNKNOWN
safeErrorMessage: Provider result was not received in the expected time window
latencyMs: null
correlationId: corr-demo-003
traceId: trace-demo-003
```

### 11.4. Demo Operation 4: Timed Out

Operation:

```text
diagnosticOperationId: op-demo-004
operationType: TRANSFER
technicalStatus: TIMED_OUT
providerKey: transfer-provider-main
serviceKey: payment-orchestrator
failedStage: EXTERNAL_PROVIDER
errorCode: OPERATION_TIMEOUT
safeErrorMessage: Operation exceeded configured timeout
latencyMs: 30000
correlationId: corr-demo-004
traceId: trace-demo-004
```

### 11.5. Demo Operation 5: Success

Operation:

```text
diagnosticOperationId: op-demo-005
operationType: PAYMENT
technicalStatus: SUCCEEDED
providerKey: provider-main
serviceKey: payment-orchestrator
failedStage: null
errorCode: null
safeErrorMessage: null
latencyMs: 850
correlationId: corr-demo-005
traceId: trace-demo-005
```

---

## 12. Repository and Architecture Rules

### 12.1. Do Not Restructure the Whole Solution

The implementation must not perform full solution restructuring during MVP-1.

Use the existing repository structure.

Before making changes, inspect the current project structure, existing models, database configuration, migrations, services, controllers, pages, components, and tests.

Reuse existing patterns instead of creating a parallel architecture.

Do not delete existing documentation, code, pages, components, migrations, or tests unless they are clearly obsolete and replaced by the MVP-1 implementation.

Add folders/classes only where needed.

If the existing project is layered, follow existing layers.

If the existing project is simpler, implement MVP-1 with minimal structural changes.

### 12.2. Recommended File Types

Exact paths must be adapted to the current repository, but MVP-1 should add files equivalent to:

```text
Models/OperationDiagnostic.cs
Models/OperationRouteStep.cs
Models/DiagnosticEvidenceLink.cs

Dtos/OperationListItemDto.cs
Dtos/OperationDetailsDto.cs
Dtos/OperationRouteStepDto.cs
Dtos/DiagnosticEvidenceLinkDto.cs
Dtos/RouteStepEvidenceGroupDto.cs
Dtos/PagedResult.cs
Dtos/OperationFilterRequest.cs

Services/OperationDiagnosticsService.cs
Services/EvidenceLinkService.cs

Repositories/OperationDiagnosticRepository.cs
Repositories/OperationRouteStepRepository.cs
Repositories/DiagnosticEvidenceLinkRepository.cs

Controllers/TechnicianOperationsController.cs
Controllers/TechnicianRouteStepEvidenceController.cs
Controllers/TechnicianOperationRouteEvidenceController.cs

Alternatively, implement route evidence as a method inside TechnicianOperationsController if that matches the existing project structure.

Pages/Operations.razor
Pages/OperationDetails.razor

Components/OperationRouteTimeline.razor
Components/DiagnosticEvidencePanel.razor
Components/OperationStatusBadge.razor
Components/RouteStepStatusBadge.razor

Migrations/AddOperationDiagnosticsMvp1.cs
Seed/OperationDiagnosticsSeedData.cs
```

Do not create all files blindly if the existing project already has equivalent structures.

---

## 13. Implementation Plan for MVP-1

### Step 1: Add Database Migration

Implement:

- `operation_diagnostics`;
- `operation_route_steps`;
- `diagnostic_evidence_links`.

Acceptance criteria:

- migration applies successfully;
- enum CHECK constraints exist;
- evidence target integrity constraint exists;
- no forbidden columns exist;
- route steps have direct FK to operation diagnostics;
- evidence links have direct FK to operation diagnostics or route steps.

### Step 2: Add Domain Models and Enum Constants

Implement domain models and enum constants.

Acceptance criteria:

- enums match this specification;
- `operationId` API maps to `diagnostic_operation_id`;
- no raw/PII fields exist.

### Step 3: Add Repositories

Implement read repositories for:

- operation list;
- operation details;
- route steps;
- operation evidence links;
- route-step evidence links;
- grouped route-step evidence links.

Acceptance criteria:

- filtering works;
- pagination works;
- sorting works;
- route steps are ordered by `step_order`;
- DTOs contain safe fields only.

### Step 4: Add Application Services

Implement:

- `OperationDiagnosticsService`;
- `EvidenceLinkService`.

Acceptance criteria:

- service validates filters;
- service validates enum values;
- service returns DTOs only;
- service handles not found correctly;
- service does not expose database entities directly.

### Step 5: Add API Endpoints

Implement:

```text
GET /api/technician/operations
GET /api/technician/operations/{operationId}
GET /api/technician/operations/{operationId}/route
GET /api/technician/operations/{operationId}/evidence
GET /api/technician/operations/{operationId}/route/evidence
GET /api/technician/route-steps/{routeStepId}/evidence
```

Acceptance criteria:

- invalid filters return 400;
- unknown operation returns 404;
- unknown route step returns 404;
- endpoint responses match DTOs;
- no future MVP-3 endpoints are implemented.

### Step 6: Add Seed Data

Add safe demo operations and route steps.

Acceptance criteria:

- at least 5 demo operations exist;
- at least one failed operation exists;
- at least one timed-out operation exists;
- at least one slow successful operation exists;
- at least one unknown provider result exists;
- evidence links exist for operation and route step;
- no PII or raw payload exists.

### Step 7: Add Operations UI

Implement:

```text
/operations
```

Acceptance criteria:

- data comes from backend API;
- filters work;
- pagination works;
- sorting works;
- statuses are visually distinguishable;
- row action opens operation details.

### Step 8: Add Operation Details UI

Implement:

```text
/operations/{operationId}
```

Acceptance criteria:

- operation details load from backend;
- route timeline loads from backend;
- operation evidence links load from backend;
- route-step evidence links are visible if available;
- failed step is highlighted;
- slowest step is highlighted;
- no forbidden fields are displayed.

### Step 9: Add Tests

Minimum tests:

- migration applies;
- enum constraints reject invalid values;
- evidence target constraint rejects invalid target combinations;
- operation list endpoint works;
- operation details endpoint works;
- route endpoint returns ordered steps;
- operation evidence endpoint works;
- grouped route-step evidence endpoint works;
- route-step evidence endpoint works;
- evidence URL scheme validation works;
- filters work;
- sorting works;
- unknown operation returns 404;
- forbidden fields are absent from DTOs.

---

## 14. Testing Requirements

### 14.1. Database Tests

Required:

- invalid operation status is rejected;
- invalid operation type is rejected;
- invalid route step status is rejected;
- invalid route step type is rejected;
- invalid evidence type is rejected;
- invalid evidence provider type is rejected;
- evidence link with no target is rejected;
- evidence link with multiple targets is rejected;
- route step cannot exist without operation diagnostic.

### 14.2. API Tests

Required:

- operation list returns page;
- operation list filters by status;
- operation list filters by provider;
- operation list sorts by startedAtUtc desc;
- operation details returns by safe operationId;
- unknown operation returns 404;
- route returns ordered steps;
- operation evidence returns operation links;
- grouped route-step evidence returns route-step links by routeStepId;
- route-step evidence returns route-step links;
- evidence URL scheme validation rejects unsafe schemes.

### 14.3. UI Manual Acceptance

Required manual checks:

1. Open `/operations`.
2. Confirm operation list loads from backend.
3. Filter by `FAILED`.
4. Open `op-demo-001`.
5. Confirm Provider Adapter is failed.
6. Confirm route timeline is ordered.
7. Confirm operation evidence links are visible.
8. Confirm route-step evidence links are visible.
9. Confirm no customer data is displayed.
10. Confirm no raw request/response/provider payload/log/trace is displayed.

---

## 15. Future MVP-3 Design: Ingestion

This section is future design only.

Do not implement it in MVP-1.

### 15.1. Future Endpoint

```text
POST /api/ingestion/operation-diagnostics
```

### 15.2. Future Rules

When implemented, ingestion must:

- be internal-only;
- require internal authentication;
- validate all enum values;
- reject PII;
- reject raw payloads;
- be idempotent;
- persist only safe normalized data.

### 15.3. Future Idempotency Rule

Ingestion should be idempotent by:

```text
diagnostic_operation_id
```

or by an internal event id if the design becomes event-based.

For MVP-1, ingestion does not exist.

---

## 16. Future MVP-3 Design: Diagnostic Checks

This section is future design only.

Do not implement it in MVP-1.

### 16.1. Future Endpoint

```text
POST /api/technician/path-elements/{pathElementKey}/checks
```

### 16.2. Future Rules

Diagnostic checks must:

- be read-only;
- have timeout;
- have rate limit;
- write audit log;
- return safe result DTO;
- not retry payment;
- not replay operation;
- not mutate external provider state;
- not fetch customer data;
- not store raw request/response.

For MVP-1, diagnostic checks do not exist.

---

## 17. Definition of Done for MVP-1

MVP-1 is done only when:

1. `operation_diagnostics` table exists.
2. `operation_route_steps` table exists.
3. `diagnostic_evidence_links` table exists.
4. PostgreSQL enum CHECK constraints exist.
5. Evidence links have direct FK to operation or route step.
6. Evidence links enforce exactly one target.
7. API uses safe `operationId` mapped to `diagnostic_operation_id`.
8. Operation list is backend-driven.
9. Operation details are backend-driven.
10. Route timeline is backend-driven.
11. Operation evidence links are backend-driven.
12. Route-step evidence links are backend-driven.
13. Grouped route-step evidence endpoint is implemented to avoid N+1 frontend calls.
14. Route steps expose `hasEvidence` and `evidenceCount`.
15. `external_operation_reference_hash` is not returned by technician APIs.
16. Frontend operation mock data is removed or isolated.
17. Seed data contains at least 5 safe operations.
18. No PII fields exist in models, DTOs, database, seed data, or UI.
19. No raw payload fields exist in models, DTOs, database, seed data, or UI.
20. Ingestion endpoints are not implemented.
21. Diagnostic check endpoints are not implemented.
22. Tests cover enum constraints, evidence target constraints, APIs, sorting, filtering, and route ordering.
23. `diagnostic_operation_id`, route step `step_name`, evidence `label`, and evidence `url` cannot be empty.
24. Evidence URL validation enforces HTTPS in production.
25. `page`, `pageSize`, `sortBy`, and `sortDirection` validation exists.
26. `external_operation_reference_hash` is not required for MVP-1 seed/demo data.
27. README or project docs explain how to run MVP-1.
28. Grouped route-step evidence endpoint is covered by API tests.
29. Evidence URL scheme validation is covered by unit tests.
30. Technician APIs do not return `operation_diagnostics.id`.
31. Cursor prompt is stored separately under `docs/cursor-prompts/mvp-1-implementation.md`.
32. The main specification contains only a short Cursor Usage section, not the full prompt.

---

## 18. Cursor Usage

The long Cursor implementation prompt is stored separately and must be copied into Cursor chat when implementation starts.

Prompt file:

```text
docs/cursor-prompts/mvp-1-implementation.md
```

Source of truth:

```text
docs/megapayment-implementation-specification-v3.3.md
```

Rules:

- this specification defines what must be implemented;
- the Cursor prompt defines how Cursor should start implementation;
- the Cursor prompt must not override this specification;
- if the prompt and specification conflict, this specification wins;
- the prompt must instruct Cursor to implement only MVP-1;
- the prompt must explicitly forbid MVP-2/MVP-3 scope.

---

## 19. Final Acceptance Criteria

The implementation is acceptable when Megapayment can be demonstrated as a real diagnostic module:

- user opens operations page;
- user sees failed, slow, unknown, successful operations;
- user opens a failed operation;
- user sees ordered technical route;
- user identifies the failed provider adapter step;
- user sees safe error message;
- user opens safe evidence links;
- no customer data is shown;
- no raw payload is shown;
- no future MVP-3 scope was accidentally implemented;
- MVP-1 technician API remains read-only;
- evidence links are type-safe and backed by foreign keys;
- grouped route-step evidence avoids N+1 frontend calls;
- internal operation table UUID and external operation hash are not exposed through technician APIs.
