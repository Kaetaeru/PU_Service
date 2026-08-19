# Pickup Service Integration Contract

## Status

This document records the evidence-backed integration contract for `PU_Service` at TASK-001. It deliberately avoids provider-specific field names, endpoints, and lifecycle assumptions that are not present in the repository or user requirements.

## Confirmed project intent

`PU_Service` is intended to act as a service layer connecting accommodation operations with a pickup/transport service.

At this checkpoint:

- no accommodation-side system, API, event source, or schema has been identified;
- no pickup/transport provider or provider API has been identified;
- no product implementation exists in this repository;
- no authentication, cancellation, scheduling, status-mapping, persistence, or deployment contract has been supplied.

These missing external contracts materially affect implementation architecture and must not be guessed.

## Provider-neutral responsibility boundary

Subject to the unresolved decisions below, the minimum responsibility of `PU_Service` is expected to be:

1. accept or derive a pickup intent from an accommodation-side source;
2. validate that the intent contains enough information to request transport;
3. assign or retain a stable internal request identity and correlation context;
4. submit the request through a provider adapter or equivalent external boundary;
5. correlate provider acknowledgement/rejection with the internal request;
6. ingest later provider status changes by the mechanism the provider actually supports (for example callback, polling, or another integration mechanism);
7. expose or propagate normalized request state back to the accommodation-side consumer;
8. handle cancellation, terminal failure, duplicate delivery, retryable transport errors, and reconciliation without creating duplicate provider jobs.

This is a logical service boundary, not a commitment to HTTP, webhooks, queues, polling, or any particular deployment model.

## Minimum logical entities and identifiers

The integration needs the following concepts regardless of provider-specific schema. Exact field names remain implementation decisions.

### PickupRequest

An internal record representing one requested transport operation.

Minimum identity/context requirements:

- stable internal request identifier;
- accommodation/source-system identifier;
- source record/reference identifier;
- requested pickup schedule or scheduling constraint;
- origin and destination information at the precision required by the selected provider;
- normalized lifecycle state;
- created/updated timestamps;
- correlation metadata needed to trace calls and events.

The transported subject, passenger/load details, capacity requirements, and contact details are unresolved because the user has not yet defined the pickup use case precisely.

### ProviderDispatch

Represents the relationship between one internal pickup request and one external provider request/job.

Minimum concepts:

- internal pickup request identifier;
- provider identity;
- provider-side job/reference identifier once one exists;
- dispatch attempt identity;
- current provider-facing outcome/status;
- last synchronization timestamp.

### IntegrationAttempt

Records an outbound or inbound integration attempt sufficiently to diagnose duplicate delivery and retry behavior.

Minimum concepts:

- correlation/request identity;
- operation type;
- attempt number or unique attempt identity;
- success/failure classification;
- retryability classification;
- timestamp.

### LifecycleEvent

Records a meaningful pickup lifecycle transition or integration event for audit and reconciliation.

Minimum concepts:

- pickup request identifier;
- previous and resulting state where applicable;
- event source;
- event timestamp;
- correlation metadata.

## Minimum lifecycle semantics

Exact enum names and provider status mappings cannot be finalized until the external provider is known. The service must nevertheless support these semantic phases:

1. **Intent received** — accommodation-side pickup intent is received or detected.
2. **Validation** — required business/integration data is accepted or rejected before dispatch.
3. **Dispatch pending/attempted** — a provider request is prepared and submitted.
4. **Provider accepted or rejected** — the external side either creates/accepts the job or returns a business rejection.
5. **Active service** — later non-terminal provider states are correlated to the same request and normalized without inventing a provider-independent status vocabulary prematurely.
6. **Completed** — transport reaches its successful terminal state.
7. **Cancelled** — cancellation is recorded and, when required, propagated across the integration boundary.
8. **Failed** — unrecoverable dispatch, synchronization, or business failure is recorded with a reason suitable for reconciliation.

Cancellation may branch from any non-terminal state allowed by the actual business/provider policy. The exact allowed transitions remain unresolved.

## Idempotency and duplicate protection

Implementation must prevent duplicate external jobs when the same accommodation-side intent or retry is delivered more than once.

Required behavior:

- use one stable internal request identity for one logical pickup request;
- persist correlation between the source reference, internal request, and provider job/reference;
- make retries distinguishable from new business requests;
- treat provider idempotency support as optional until its API is known;
- never blindly retry a request whose provider outcome is unknown without first reconciling whether an external job was created.

The exact idempotency key format cannot be defined until the source and provider contracts are known.

## Retry and error classification

At minimum, errors must be separated into:

- validation/business rejection — not automatically retried;
- authentication/authorization failure — blocked until credentials/configuration are corrected;
- transient transport/provider failure — retryable under a bounded policy;
- rate limiting — retried according to the provider contract;
- timeout/unknown outcome — reconciled before a create request is repeated;
- malformed or unmappable provider event — quarantined/logged for investigation rather than silently discarded.

Retry counts, delay/backoff policy, dead-letter handling, and provider-specific error mapping remain unresolved.

## Authentication and secret boundary

Authentication methods are currently unknown. Architecture must leave explicit boundaries for:

- accommodation-side caller/source authentication;
- outbound provider authentication;
- inbound provider callback/event verification if the selected provider uses callbacks;
- secret storage and rotation.

No credential type, token format, signing algorithm, or secret store is assumed here.

## Observability and reconciliation requirements

The service should be able to answer, for each logical pickup request:

- which accommodation/source record caused it;
- whether a provider job was created and which one;
- the last known normalized and provider state;
- what integration attempts occurred and why they failed or retried;
- whether the request is terminal or still requires reconciliation.

Logs and metrics must use correlation identifiers and must avoid exposing sensitive guest/contact data unnecessarily. Retention duration and compliance requirements remain unresolved.

## Architecture-affecting unresolved decisions

TASK-001 cannot become implementation-ready until the following are answered:

1. **Accommodation source** — What system/repository/API creates the booking/stay/pickup context, and how should `PU_Service` receive it?
2. **Pickup service meaning/provider** — What exact service is being connected? Is the transported subject guests/passengers, luggage/cargo, or another truck/transport use case, and is there an existing external provider/API?
3. **Creation trigger** — Is pickup created automatically from a reservation/stay event, manually/on demand, or both?
4. **Scheduling contract** — Exact timestamp or time window? Which timezone is authoritative? What happens on schedule changes or delays?
5. **Cancellation/change policy** — Which states may be cancelled or modified, and which side is authoritative after provider acceptance?
6. **Status interface** — What statuses must the accommodation side see, and how does the provider publish/query status?
7. **Authentication** — How do the accommodation side and provider authenticate, and how are inbound provider events verified if applicable?
8. **Ownership/deployment boundary** — Does `PU_Service` own persistence, public/internal API exposure, background processing, deployment, and API-gateway concerns, or only domain/integration logic?
9. **Retention/privacy** — What guest/contact/location data may be stored and for how long?

## Acceptance checkpoint for TASK-001

Completed from current evidence:

- provider-neutral lifecycle semantics documented;
- minimum logical entities and identifiers documented without provider field invention;
- integration/authentication boundaries identified;
- idempotency, retry, observability, and reconciliation requirements documented;
- architecture-affecting unresolved decisions enumerated explicitly.

Not yet satisfied:

- provider-specific and accommodation-side contracts are still unknown;
- therefore a concrete implementation architecture/task cannot be selected safely.

The next valid transition is to obtain the unresolved decisions above, then resume TASK-001 and finalize the contract before starting TASK-002.