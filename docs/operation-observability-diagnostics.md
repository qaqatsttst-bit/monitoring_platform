# Operation Observability Diagnostics

## 1. Purpose

Megapayment is not just a service catalog with status labels.

Megapayment is a **hybrid diagnostic control plane** for technical monitoring and diagnostics of:

- internal services;
- bank adapters;
- provider adapters;
- external providers;
- business operations;
- operation routes;
- incidents;
- events;
- observability evidence such as metrics, logs, traces, dashboards, and alerts.

The main goal is to help a technical user answer questions such as:

- Which service, adapter, or provider is currently degraded?
- Which business operations are failing?
- What happened to a specific operation?
- At which route step did the operation fail or become slow?
- Which service, adapter, or provider was involved?
- Which correlationId or traceId is related to the problem?
- Where should the user inspect metrics, logs, traces, dashboards, or alerts?
- Can a specific path element be checked safely without changing external state?

Megapayment must provide normalized technical diagnostics while avoiding exposure of customer personal data.

Megapayment must not become a replacement for Prometheus, Loki, Tempo, Jaeger, Grafana, or Alertmanager.

Instead, Megapayment stores a safe normalized diagnostic model and connects it to external observability systems through links, adapters, or safe summaries.

---

## 2. Architecture Overview

Megapayment should be designed as a hybrid diagnostic control plane with the following layers:

1. **Service-level monitoring**
2. **Adapter/provider monitoring**
3. **Operation-level diagnostics**
4. **Operation route/path diagnostics**
5. **Observability evidence layer**
6. **Safe diagnostic checks**
7. **Adapter integration layer**

### 2.1. Service-level monitoring

Service-level monitoring answers:

> What is the current technical status of a service?

Examples:

- payment-orchestrator is OK;
- provider-adapter is DEGRADED;
- callback-handler is DOWN;
- gateway latency is above threshold;
- a service has stale telemetry;
- error rate increased during the last time window.

This layer is represented by service status snapshots, metric snapshots, service drill-down pages, and service-level evidence links.

### 2.2. Adapter/provider monitoring

Adapter/provider monitoring answers:

> Is a bank adapter, provider adapter, or external provider working correctly?

Examples:

- bank internal adapter is reachable;
- provider adapter has high latency;
- external provider is not responding;
- provider status check failed;
- last successful check happened too long ago.

This layer must be separate from generic service monitoring because adapters and providers may have their own status semantics, check mechanisms, and integration restrictions.

### 2.3. Operation-level diagnostics

Operation-level diagnostics answers:

> What happened to a specific operation?

Examples:

- operation failed at provider authorization;
- operation timed out while waiting for provider response;
- operation succeeded but had high latency;
- operation failed with a safe technical error code;
- operation is linked to correlationId and traceId.

This layer must not be mixed into service-level monitoring entities.

### 2.4. Operation route/path diagnostics

Operation route diagnostics answers:

> At which route step did the operation fail, slow down, or become unclear?

An operation route is a sequence of technical path elements involved in processing an operation.

Examples of route elements:

- API Gateway;
- Payment Orchestrator;
- Bank Internal Adapter;
- Fraud/AML Service;
- Provider Adapter;
- External Provider;
- Callback Handler;
- Status Updater.

Each route step should have its own status, latency, error code, safe error message, and observability identifiers.

### 2.5. Observability evidence layer

The observability evidence layer answers:

> Where can the user inspect charts, logs, traces, dashboards, and alerts related to this problem?

Megapayment should provide evidence links to external systems such as:

- Prometheus;
- Grafana;
- Loki;
- Tempo;
- Jaeger;
- Alertmanager.

Megapayment may store safe evidence metadata and generated links, but it must not store raw logs, raw traces, raw span attributes, raw provider payloads, raw requests, or raw responses.

### 2.6. Safe diagnostic checks

Safe diagnostic checks answer:

> Can this path element be checked without changing external state?

Allowed examples:

- healthcheck;
- status check;
- latency check;
- availability check;
- safe operation lookup;
- safe provider status lookup.

Forbidden examples:

- payment retry;
- operation replay;
- mutation request;
- state-changing provider call;
- fetching customer/payment personal data;
- storing raw provider response.

### 2.7. Adapter integration layer

The adapter integration layer is responsible for safe communication with external systems.

It should include separate adapter types for:

- bank diagnostics;
- provider diagnostics;
- operation lookup;
- path element checks;
- Prometheus metrics;
- Loki logs;
- Tempo/Jaeger traces;
- Grafana links.

Bank/provider diagnostic adapters must not be mixed with observability adapters.

---

## 3. Responsibility Separation

### 3.1. What Megapayment should store

Megapayment should store safe normalized technical data:

- services;
- bank adapters;
- provider adapters;
- external providers;
- operation diagnostics;
- operation route steps;
- service status snapshots;
- adapter/provider status snapshots;
- metric snapshots;
- diagnostic evidence links;
- events;
- incidents;
- safe diagnostic check results;
- timestamps;
- technical statuses;
- correlationId;
- traceId.

This data should be enough to build dashboards, operation details, route timelines, and investigation entry points.

### 3.2. What Megapayment must not store

Megapayment must not store:

- raw logs;
- raw traces;
- raw span attributes;
- raw provider payload;
- raw request;
- raw response;
- exception stack trace;
- full exception object;
- customer name;
- card number;
- masked card number;
- phone;
- email;
- passport;
- account number;
- client requisites;
- customer identifiers that can identify a real person;
- any customer personal data.

Megapayment must treat all raw external payloads as unsafe by default.

---

## 4. Difference Between Monitoring and Diagnostics

### 4.1. Service-level monitoring

Service-level monitoring focuses on the state of technical components.

It answers:

> What is the current status of a service, adapter, or provider?

Typical data:

- serviceKey;
- displayName;
- status;
- latencyMs;
- errorRate;
- availability;
- lastUpdatedAtUtc;
- freshnessStatus.

### 4.2. Operation-level diagnostics

Operation-level diagnostics focuses on a specific business operation.

It answers:

> What happened to this specific operation?

Typical data:

- operationId;
- operationType;
- technicalStatus;
- failedStage;
- errorCode;
- safeErrorMessage;
- latencyMs;
- correlationId;
- traceId.

### 4.3. Route/path diagnostics

Route/path diagnostics focuses on the technical route of an operation.

It answers:

> Which route step failed or became slow?

Typical data:

- stepOrder;
- stepName;
- stepType;
- serviceKey;
- adapterKey;
- providerKey;
- status;
- latencyMs;
- errorCode;
- safeErrorMessage;
- startedAtUtc;
- completedAtUtc.

### 4.4. Observability evidence

Observability evidence focuses on investigation entry points.

It answers:

> Where can the user inspect metrics, logs, traces, dashboards, or alerts?

Typical data:

- evidenceType;
- providerType;
- label;
- url;
- timeRangeFromUtc;
- timeRangeToUtc.

---

## 5. Core Concepts

### 5.1. Service

A service is an internal technical component of the platform.

Examples:

- API Gateway;
- Payment Orchestrator;
- Callback Handler;
- Status Updater.

A service can have:

- status;
- metrics;
- logs;
- traces;
- incidents;
- events;
- operation route steps.

### 5.2. BankAdapter

A BankAdapter is an integration element that communicates with internal bank systems.

It may expose safe diagnostic information such as:

- availability;
- latency;
- technical status;
- last check timestamp;
- safe error code;
- safe error message.

It must not expose customer personal data.

### 5.3. ProviderAdapter

A ProviderAdapter is an integration element that communicates with an external provider.

It may expose safe diagnostic information such as:

- provider status;
- technical error code;
- latency;
- timeout status;
- last successful check timestamp.

It must not expose raw provider payload or customer data.

### 5.4. ExternalProvider

An ExternalProvider is an external system involved in processing operations.

Examples:

- payment provider;
- transfer provider;
- card processing provider;
- wallet provider.

Megapayment should monitor its technical availability and status through safe adapters or imported telemetry.

### 5.5. OperationDiagnostic

OperationDiagnostic is a safe technical diagnostic record for a specific operation.

It must describe the technical state of the operation without storing customer personal data.

### 5.6. OperationRoute

OperationRoute is the ordered technical path of an operation.

It is composed of route steps.

### 5.7. OperationRouteStep

OperationRouteStep describes one technical step in the operation route.

A route step may represent:

- internal service;
- bank adapter;
- provider adapter;
- external provider;
- callback handler;
- status updater.

### 5.8. PathElement

PathElement is a generic term for any technical element that can participate in an operation route.

Examples:

- service;
- bank adapter;
- provider adapter;
- external provider;
- observability source.

### 5.9. DiagnosticEvidenceLink

DiagnosticEvidenceLink connects a target object with investigation evidence.

Targets may include:

- service;
- adapter;
- provider;
- operation;
- route step;
- incident;
- event.

Evidence may include:

- metrics;
- logs;
- traces;
- dashboard;
- alert.

### 5.10. DiagnosticCheck

DiagnosticCheck is a safe read-only check of a technical path element.

It must never change state in external systems.

### 5.11. MetricSnapshot

MetricSnapshot is a short local metric snapshot stored by Megapayment.

It should be used for:

- current status cards;
- summary dashboards;
- short history;
- service drill-down;
- degraded/down detection.

Detailed time-series analysis should be delegated to Prometheus/Grafana.

### 5.12. ObservabilityDataSource

ObservabilityDataSource is an external observability system used by Megapayment.

Examples:

- Prometheus;
- Grafana;
- Loki;
- Tempo;
- Jaeger;
- Alertmanager.

### 5.13. CorrelationId

CorrelationId is a technical identifier used to correlate logs, events, operation diagnostics, and route steps.

It must not contain customer personal data.

### 5.14. TraceId

TraceId is a technical identifier used to open distributed traces in tracing systems such as Tempo or Jaeger.

It must not contain customer personal data.

---

## 6. Example Operation Route

A safe example operation route:

1. API Gateway
2. Payment Orchestrator
3. Bank Internal Adapter
4. Fraud/AML Service
5. Provider Adapter
6. External Provider
7. Callback Handler
8. Status Updater

Each route step may expose only safe technical data:

- stepOrder;
- stepName;
- stepType;
- serviceKey;
- adapterKey;
- providerKey;
- status;
- latencyMs;
- errorCode;
- safeErrorMessage;
- correlationId;
- traceId;
- startedAtUtc;
- completedAtUtc.

Example route step:

```text
stepOrder: 5
stepName: Provider Adapter
stepType: ProviderAdapter
serviceKey: provider-adapter-main
adapterKey: provider-adapter-default
providerKey: provider-main
status: Failed
latencyMs: 15000
errorCode: PROVIDER_TIMEOUT
safeErrorMessage: Provider did not respond within timeout
correlationId: corr-technical-id
traceId: trace-technical-id
startedAtUtc: 2026-04-27T10:00:02Z
completedAtUtc: 2026-04-27T10:00:17Z
```

This example intentionally contains no customer personal data.

---

## 7. Operation Data

### 7.1. Allowed operation fields

Operation-level diagnostics may include:

- operationId;
- operationType;
- technicalStatus;
- providerKey;
- serviceKey;
- failedStage;
- errorCode;
- safeErrorMessage;
- latencyMs;
- correlationId;
- traceId;
- startedAtUtc;
- completedAtUtc;
- createdAtUtc.

### 7.2. Forbidden operation fields

Operation-level diagnostics must not include:

- customerName;
- cardNumber;
- maskedCardNumber;
- phone;
- email;
- passport;
- accountNumber;
- clientRequisites;
- rawRequest;
- rawResponse;
- providerPayload;
- exception;
- stackTrace;
- fullErrorMessage.

### 7.3. Safe error message rule

`safeErrorMessage` must be a sanitized technical message.

It must not be:

- raw exception text;
- raw provider response;
- stack trace;
- raw validation payload;
- customer-visible banking message;
- data copied from logs without redaction.

Good examples:

```text
Provider did not respond within timeout
Adapter returned unavailable status
Route step exceeded latency threshold
Trace was not found in configured time window
```

Bad examples:

```text
Raw exception object
Raw provider response body
Full HTTP request body
Full HTTP response body
Stack trace
Customer-related message
```

---

## 8. Status Model

### 8.1. Service status

A service status view should show:

- serviceKey;
- displayName;
- status;
- latencyMs;
- errorRate;
- availability;
- lastUpdatedAtUtc;
- freshnessStatus.

Recommended status values:

- OK;
- DEGRADED;
- DOWN;
- NO_DATA;
- UNKNOWN.

### 8.2. Adapter status

An adapter status view should show:

- adapterKey;
- adapterType;
- status;
- latencyMs;
- errorRate;
- lastCheckAtUtc;
- lastSuccessfulCheckAtUtc;
- safeErrorMessage.

Recommended adapter types:

- BankAdapter;
- ProviderAdapter;
- InternalAdapter;
- ObservabilityAdapter.

### 8.3. Provider status

A provider status view should show:

- providerKey;
- providerType;
- status;
- latencyMs;
- availability;
- lastCheckAtUtc;
- lastKnownIncident.

Recommended provider statuses:

- OK;
- DEGRADED;
- DOWN;
- NO_DATA;
- UNKNOWN.

### 8.4. Operation technical status

An operation diagnostic view should show:

- operationId;
- operationType;
- technicalStatus;
- failedStage;
- errorCode;
- safeErrorMessage;
- latencyMs;
- correlationId;
- traceId.

Recommended operation technical statuses:

- PENDING;
- PROCESSING;
- SUCCEEDED;
- FAILED;
- TIMED_OUT;
- UNKNOWN.

---

## 9. Charts, Metrics, Logs, and Traces

### 9.1. Service-level charts

Service-level charts may include:

- latency over time;
- error rate over time;
- availability over time;
- request count;
- timeout count;
- failed operation count;
- provider error count.

Megapayment may use local `MetricSnapshot` data for short summaries and status cards.

For detailed time-series charts, Megapayment should link to or query Prometheus/Grafana.

### 9.2. Operation-level timeline

Operation-level diagnostics should not be represented only as a standard line chart.

For a specific operation, the most useful view is a route timeline:

```text
API Gateway              OK        40 ms
Payment Orchestrator     OK        120 ms
Bank Internal Adapter    OK        200 ms
Fraud/AML Service        OK        300 ms
Provider Adapter         FAILED    15000 ms
External Provider        UNKNOWN   -
Callback Handler         SKIPPED   -
Status Updater           FAILED    30 ms
```

The timeline should highlight:

- failed step;
- slowest step;
- total latency;
- error code;
- safe error message;
- evidence links.

### 9.3. Metrics

Metrics can be handled through two approaches:

1. Local snapshots stored in Megapayment.
2. Detailed external metrics through Prometheus/Grafana.

Megapayment should not become its own time-series database.

Recommended approach:

- use `MetricSnapshot` for current state and short history;
- use Prometheus/Grafana for detailed charts and long-range analysis;
- store evidence links to detailed charts;
- avoid duplicating large time-series datasets.

### 9.4. Logs

In the MVP, logs should be opened through links to Loki or another external log system.

Megapayment should build log links using safe technical parameters:

- correlationId;
- traceId;
- serviceKey;
- providerKey;
- time range.

Megapayment must not store raw logs.

Megapayment must not display raw log previews unless a separate redaction policy and safe preview mechanism are implemented.

### 9.5. Traces

In the MVP, traces should be opened through links to Tempo, Jaeger, or another external tracing system.

Megapayment should build trace links using:

- traceId;
- correlationId;
- serviceKey;
- time range.

Megapayment must not store raw traces or raw span attributes.

Future safe trace summaries may include:

- total duration;
- slowest span;
- failed span;
- involved services;
- technical error code.

Such summaries must be sanitized and must not contain PII.

### 9.6. Alerts

Alerts may be linked through Alertmanager or another alerting system.

Megapayment may store:

- alert link;
- alert label;
- technical status;
- time range;
- affected service/provider/operation.

Megapayment must not store raw alert payloads if they may contain sensitive data.

---

## 10. Diagnostic Evidence Links

DiagnosticEvidenceLink connects a diagnostic target with external evidence.

### 10.1. Supported target types

Supported target types may include:

- Service;
- Adapter;
- Provider;
- Operation;
- RouteStep;
- Incident;
- Event.

### 10.2. Supported evidence types

Supported evidence types may include:

- Metrics;
- Logs;
- Trace;
- Dashboard;
- Alert.

### 10.3. Supported provider types

Supported provider types may include:

- Prometheus;
- Grafana;
- Loki;
- Tempo;
- Jaeger;
- Alertmanager;
- InternalDashboard.

### 10.4. Recommended fields

DiagnosticEvidenceLink should contain:

- targetType;
- targetId;
- evidenceType;
- providerType;
- url;
- label;
- timeRangeFromUtc;
- timeRangeToUtc;
- createdAtUtc.

### 10.5. Safety rules

Evidence links must not include PII in URL parameters.

Evidence links should be generated from safe technical identifiers only.

Safe examples:

- serviceKey;
- providerKey;
- correlationId;
- traceId;
- time range.

Unsafe examples:

- customer name;
- card number;
- phone;
- email;
- account number;
- raw payload data.

---

## 11. Safe Diagnostic Checks

### 11.1. Purpose

A diagnostic check allows the user to inspect a path element safely.

It must help answer:

- Is this service reachable?
- Is this adapter responding?
- Is this provider available?
- Is latency acceptable?
- Can this operation be found through a safe technical lookup?
- Is the observability source available?

### 11.2. Allowed checks

Allowed check types:

- healthcheck;
- status check;
- latency check;
- availability check;
- safe operation lookup;
- safe provider status lookup;
- safe adapter status lookup;
- observability source reachability check.

### 11.3. Forbidden checks

Forbidden check types:

- payment retry;
- operation replay;
- payment resubmission;
- mutation request;
- state-changing external call;
- customer data lookup;
- raw payment data lookup;
- raw provider payload fetch;
- raw response storage.

### 11.4. Mandatory safety requirements

Every diagnostic check must have:

- timeout;
- rate limit;
- audit log;
- safe result DTO;
- no PII;
- no raw payload;
- no external state mutation;
- clear target type;
- clear check type;
- clear result status.

### 11.5. Recommended check result fields

A safe check result may include:

- checkId;
- targetType;
- targetKey;
- checkType;
- status;
- latencyMs;
- errorCode;
- safeErrorMessage;
- checkedAtUtc;
- checkedByUserId;
- correlationId;
- traceId.

It must not include raw request, raw response, raw provider payload, customer data, or stack trace.

---

## 12. Adapter Integration Layer

The adapter integration layer should contain separate adapter contracts for different responsibilities.

### 12.1. Bank/provider diagnostic adapters

These adapters are responsible for safe technical diagnostics of bank and provider integration elements.

Recommended contracts:

- BankAdapterDiagnosticClient;
- ProviderDiagnosticClient;
- OperationLookupAdapter;
- PathElementCheckAdapter.

Responsibilities:

- fetch safe status;
- fetch safe operation diagnostic summary;
- run safe read-only checks;
- return only sanitized technical data.

They must not return:

- raw customer data;
- raw payment data;
- raw provider response;
- raw request;
- raw response;
- stack trace.

### 12.2. Observability adapters

These adapters are responsible for observability evidence.

Recommended contracts:

- PrometheusClient;
- LokiClient;
- TempoClient;
- GrafanaLinkBuilder;
- AlertmanagerClient.

Responsibilities:

- build safe metrics links;
- build safe logs links;
- build safe trace links;
- build safe dashboard links;
- optionally return safe summaries.

Observability adapters must not be mixed with bank/provider diagnostic adapters.

### 12.3. Push ingestion

Push ingestion can be used for dev/demo or controlled integration scenarios.

Examples:

- service sends operational snapshot;
- adapter sends safe status snapshot;
- provider integration sends safe status snapshot;
- operation processing pipeline sends safe operation diagnostic event.

Push ingestion must accept only safe normalized payloads.

It must not accept raw provider payloads, raw requests, raw responses, or customer personal data.

---

## 13. MVP Scope

### 13.1. In scope for MVP

The MVP should include:

- service status dashboard;
- adapter/provider status summary;
- operation list;
- operation details;
- operation route steps;
- route timeline;
- metric snapshots;
- diagnostic evidence links to metrics/logs/traces;
- safe fields only;
- backend-driven data for operations and routes;
- gradual replacement of frontend mock data with backend APIs.

### 13.2. Out of scope for MVP

The MVP should not include:

- storing raw logs;
- storing raw traces;
- storing raw provider payload;
- retrying payments;
- replaying operations;
- mutation checks against external systems;
- real integrations with all providers;
- complex automatic correlation of all distributed traces;
- full built-in log viewer;
- full built-in trace viewer;
- storing PII;
- complex incident workflow;
- full alert management workflow.

### 13.3. MVP user flow

A minimal MVP user flow should be:

1. User opens the dashboard.
2. User sees degraded services, adapters, providers, and failed operation count.
3. User opens the operations page.
4. User filters failed operations by time range, provider, service, status, or error code.
5. User opens a specific operation.
6. User sees operation technical status and safe error details.
7. User sees operation route timeline.
8. User identifies failed or slow route step.
9. User opens metrics/logs/traces evidence links.
10. User may run a safe read-only check for a path element if supported.

---

## 14. Relation to the Current Project

The current project already contains several building blocks that should be preserved.

### 14.1. ServiceOperationalState

`ServiceOperationalState` should remain a service-level model.

It should not be expanded into operation diagnostics.

It answers:

> What is the current operational state of a service?

It should not answer:

> What happened to a specific business operation?

### 14.2. MetricSnapshot

`MetricSnapshot` should remain a short service-level metric model.

It can support:

- dashboard cards;
- service status;
- short history;
- freshness checks;
- service drill-down.

It should not become a replacement for Prometheus.

### 14.3. DataSource and ServiceSourceBinding

`DataSource` and `ServiceSourceBinding` should be used for observability sources and links.

Examples:

- Grafana dashboard source;
- Loki logs source;
- Tempo traces source;
- Prometheus metrics source.

They must not be overloaded to represent every bank/provider integration concept unless explicitly modeled and constrained.

### 14.4. OperationDiagnostic

`OperationDiagnostic` should be a separate model.

It should describe a specific operation using safe technical fields only.

It should not contain PII or raw payloads.

### 14.5. OperationRouteStep

`OperationRouteStep` should be a separate model.

It should represent one step in an operation route.

It should be linked to an operation diagnostic and optionally to service/adapter/provider keys.

### 14.6. Events and incidents

Events and incidents should be connected to services, adapters, providers, operations, and route steps over time.

However, complex incident workflow can remain out of MVP.

### 14.7. Frontend mock data

Frontend mock data should be gradually replaced with backend-driven data.

Mock data must be clearly treated as demo data and must not define the final architecture.

### 14.8. PushSender

PushSender can be used as demo/dev ingestion.

It must not be treated as a full bank/provider integration.

Production-grade integrations should be represented through explicit adapter contracts and safe payload rules.

---

## 15. Recommended Future Backend Model

This section is a target direction, not an immediate implementation requirement.

### 15.1. OperationDiagnostic

Recommended fields:

- Id;
- OperationId;
- OperationType;
- TechnicalStatus;
- ProviderKey;
- ServiceKey;
- FailedStage;
- ErrorCode;
- SafeErrorMessage;
- LatencyMs;
- CorrelationId;
- TraceId;
- StartedAtUtc;
- CompletedAtUtc;
- CreatedAtUtc.

### 15.2. OperationRouteStep

Recommended fields:

- Id;
- OperationDiagnosticId;
- StepOrder;
- StepName;
- StepType;
- ServiceKey;
- AdapterKey;
- ProviderKey;
- Status;
- LatencyMs;
- ErrorCode;
- SafeErrorMessage;
- CorrelationId;
- TraceId;
- StartedAtUtc;
- CompletedAtUtc.

### 15.3. DiagnosticEvidenceLink

Recommended fields:

- Id;
- TargetType;
- TargetId;
- EvidenceType;
- ProviderType;
- Url;
- Label;
- TimeRangeFromUtc;
- TimeRangeToUtc;
- CreatedAtUtc.

### 15.4. DiagnosticCheckResult

Recommended fields:

- Id;
- TargetType;
- TargetKey;
- CheckType;
- Status;
- LatencyMs;
- ErrorCode;
- SafeErrorMessage;
- CheckedAtUtc;
- CheckedByUserId;
- CorrelationId;
- TraceId.

---

## 16. Recommended Future API Surface

This section is a target direction, not an immediate implementation requirement.

### 16.1. Operation diagnostics API

Recommended endpoints:

```text
GET /api/technician/operations
GET /api/technician/operations/{operationId}
GET /api/technician/operations/{operationId}/route
GET /api/technician/operations/{operationId}/evidence
```

### 16.2. Service diagnostics API

Recommended endpoints:

```text
GET /api/technician/services/{serviceKey}/status
GET /api/technician/services/{serviceKey}/metrics
GET /api/technician/services/{serviceKey}/evidence
```

### 16.3. Adapter/provider diagnostics API

Recommended endpoints:

```text
GET /api/technician/adapters/{adapterKey}/status
GET /api/technician/providers/{providerKey}/status
GET /api/technician/providers/{providerKey}/evidence
```

### 16.4. Safe diagnostic checks API

Recommended endpoints:

```text
POST /api/technician/path-elements/{pathElementKey}/checks
GET /api/technician/diagnostic-checks/{checkId}
```

Diagnostic check endpoints must be read-only in effect.

They must never trigger payment retry, operation replay, or any external state-changing action.

---

## 17. Recommended Frontend Views

This section is a target direction, not an immediate implementation requirement.

### 17.1. Dashboard

The dashboard should show:

- service status counts;
- adapter/provider status counts;
- failed operation count;
- degraded services;
- degraded providers;
- active incidents;
- latest critical events;
- links to metrics/logs/traces/dashboards.

### 17.2. Service drill-down

Service drill-down should show:

- service status;
- current metrics;
- short metric history;
- related events;
- related incidents;
- affected operations;
- evidence links.

### 17.3. Operations page

The operations page should show:

- operationId;
- operationType;
- technicalStatus;
- providerKey;
- serviceKey;
- failedStage;
- errorCode;
- latencyMs;
- startedAtUtc;
- completedAtUtc.

Filters should include:

- status;
- provider;
- service;
- operation type;
- time range;
- error code;
- failed stage.

### 17.4. Operation details page

Operation details should show:

- operation summary;
- technical status;
- failed stage;
- safe error message;
- correlationId;
- traceId;
- route timeline;
- failed step;
- slowest step;
- evidence links.

### 17.5. Path element details page

Path element details should show:

- target type;
- target key;
- current status;
- latest check result;
- related operations;
- related incidents;
- metrics/logs/traces evidence links;
- safe read-only check action if supported.

---

## 18. Security and Privacy Rules

### 18.1. No PII rule

Megapayment must not store or display customer personal data.

Forbidden data includes:

- customer name;
- card number;
- masked card number;
- phone;
- email;
- passport;
- account number;
- client requisites;
- customer address;
- raw customer payload;
- any customer-identifying data.

### 18.2. No raw payload rule

Megapayment must not store:

- raw request;
- raw response;
- raw provider payload;
- raw logs;
- raw traces;
- raw span attributes;
- raw exception;
- stack trace.

### 18.3. Safe identifiers rule

Allowed identifiers must be technical identifiers only.

Examples:

- operationId;
- serviceKey;
- adapterKey;
- providerKey;
- correlationId;
- traceId.

These identifiers must not contain customer personal data.

### 18.4. Safe links rule

Evidence links must be generated from safe technical parameters.

Links must not include customer data in query strings.

### 18.5. Safe check rule

Diagnostic checks must be read-only.

They must not trigger:

- payment retry;
- operation replay;
- resubmission;
- provider mutation;
- external state change.

### 18.6. Auditability rule

Manual diagnostic checks should be auditable.

A check result should include:

- who initiated the check;
- when it was initiated;
- what target was checked;
- what check type was used;
- what safe result was returned.

---

## 19. Architectural Constraints

The following constraints are mandatory:

1. Do not mix service monitoring and operation diagnostics into one entity.
2. Do not store PII.
3. Do not store raw provider payload.
4. Do not store raw logs.
5. Do not store raw traces.
6. Do not build Megapayment as a replacement for Prometheus, Loki, Tempo, Jaeger, or Grafana.
7. Do not implement diagnostic checks that change state in external systems.
8. Do not add customer/payment personal fields to DTOs.
9. Do not add full exception or stack trace to operation diagnostics.
10. Do not treat PushSender as a production bank/provider integration.
11. Do not expose raw observability data without redaction policy.
12. Do not make frontend mock data the source of truth.
13. Do not introduce provider-specific logic directly into UI components.
14. Do not mix bank/provider diagnostic adapters with observability adapters.

---

## 20. Implementation Direction

Implementation should proceed in small controlled steps.

Recommended order:

1. Finalize this architecture document.
2. Add safe backend models for operation diagnostics and operation route steps.
3. Add read-only backend API for operation list, operation details, and operation route.
4. Add safe ingestion for operation diagnostics.
5. Add frontend operation list and operation details pages.
6. Add diagnostic evidence links for operation/service/route step.
7. Replace frontend mock data with backend-driven APIs.
8. Add adapter/provider status summaries.
9. Add safe read-only path element checks.
10. Add real adapter contracts gradually.

Each step must preserve the security and privacy constraints defined in this document.

---

## 21. Acceptance Criteria for This Architecture

This architecture is valid only if the following conditions are preserved:

1. Megapayment is treated as a hybrid diagnostic control plane.
2. Service monitoring, adapter/provider monitoring, operation diagnostics, route diagnostics, and observability evidence are separate concerns.
3. Megapayment stores a safe normalized technical model.
4. Megapayment does not store raw logs, raw traces, raw provider payload, raw requests, or raw responses.
5. Megapayment does not store or display PII.
6. Operation diagnostics are based on safe technical fields only.
7. Operation routes are represented as ordered technical steps.
8. Metrics, logs, traces, dashboards, and alerts are connected through evidence links or safe summaries.
9. Diagnostic checks are read-only and auditable.
10. The MVP remains focused and does not attempt to replace observability tools.
