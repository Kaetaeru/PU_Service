# Pickup Service Integration Contract

> **Superseded as an architecture document.** The intake boundary is now implemented as Supabase Edge Functions, not a standalone `PU_Service` HTTP API. Current contracts live in `기술적기획/00_CANONICAL_SYSTEM.md`, `기술적기획/06_IMPLEMENTATION_READINESS.md`, `기술적기획/07_WhatsApp_배차_왕복_설계.md` and `기술적기획/08_요금표.md`.
>
> The fare rules quoted below are replaced by `기술적기획/08_요금표.md`. KakaoTalk is not used at all.
>
> Still valid: the HTML-behaviour findings, the server-authority argument, idempotency semantics, the public-browser security boundary, and the provider-adapter separation.

## Status

TASK-001 integration discovery updated from the user-provided `TalkFile_pickup.html` and the technical planning documents under `기술적기획/`.

The accommodation-side intake interface is now sufficiently identified to define a concrete `HTML -> PU_Service` boundary without inventing an external transport-provider API.

## Confirmed accommodation-side interface

The source is a browser-based Stay Pachira airport transfer request page for guest/passenger transportation.

Confirmed behavior:

- user manually submits one transfer request;
- direction is `pickup` (airport/station -> stay) or `dropoff` (stay -> airport/station);
- guest name and date are explicitly validated by current JavaScript;
- passenger count is intended to be 1~7;
- route uses port codes `ICN`, `GMP`, `SEOULSTN` and stay codes `PH1`, `PH2`, `PH3`, `SP5`, `SP6`, `WSR`;
- optional fields include time, flight number, child seats, terminal transfer, and notes;
- pickup time means flight landing time entered by the guest;
- dropoff time means the requested accommodation collection time;
- the browser currently calculates a fare estimate from hard-coded rules;
- the current submit handler creates only a local result/ticket and driver message;
- there is no server request, persistence, request identifier, or status source of truth;
- the button labelled KakaoTalk uses browser share/clipboard behavior rather than a Kakao API integration.

## Confirmed first integration boundary

The first implementation boundary can be defined independently of the external transport provider:

```text
Stay Pachira transfer HTML
  -> POST /v1/transfer-requests
  -> PU_Service validation
  -> server-authoritative fare calculation
  -> stable request identity + persistence
  -> normalized status/lifecycle
  -> provider adapter boundary (provider implementation deferred)
```

Detailed design is recorded in:

- `기술적기획/01_HTML_분석.md`
- `기술적기획/02_PU_Service_연결_설계.md`
- `기술적기획/03_API_계약_초안.md`

## Required logical entities

### TransferRequest

Represents one guest transfer request.

Minimum concepts:

- stable internal `request_id`;
- source identifier (`stay-pachira-transfer-html` initially);
- direction;
- guest name and passenger count;
- service date/time and direction-derived time semantics;
- port code and stay code;
- optional flight number;
- add-ons;
- note;
- normalized status;
- server-authoritative fare snapshot;
- created/updated timestamps;
- idempotency/correlation identity.

### ProviderDispatch

Future external transport-provider correlation.

Minimum concepts:

- internal request id;
- provider identity;
- provider-side reference when available;
- dispatch attempt identity;
- provider state/outcome;
- synchronization timestamp.

No provider-specific schema is assumed yet.

### LifecycleEvent / IntegrationAttempt

Used for audit, retry and reconciliation.

Minimum concepts:

- request id;
- operation/event type;
- source;
- transition/outcome;
- retryability where relevant;
- timestamp;
- correlation identity.

## Server-side validation boundary

The browser is not authoritative.

`PU_Service` must validate at least:

- direction enum;
- non-empty guest name;
- passenger count 1~7;
- valid required date;
- valid optional time;
- known port code;
- known stay code;
- child-seat count 0~4;
- terminal-transfer rule (not valid for Seoul Station);
- provider-independent request consistency.

Maximum text lengths, past-date policy, advance-booking rules, and whether pickup flight/time becomes mandatory remain business-policy decisions.

## Fare authority

The existing HTML fare logic can remain as an immediate UI estimate during the first migration.

`PU_Service` must recompute the fare from server-owned policy and return the authoritative total. Client-supplied fare values must not be trusted.

The current source rules include:

- route/area/direction/passenger-tier base fare;
- child seat: KRW 10,000 each;
- terminal transfer: KRW 20,000;
- terminal transfer hidden for Seoul Station.

Longer term, fare/master configuration should move away from hard-coded browser constants toward a server-owned source of truth.

## Idempotency and duplicate protection

The HTML API submission requires safe retry semantics.

Required behavior:

- one stable internal request identity for one logical transfer;
- client-generated idempotency key per form submission intent;
- same key + same payload returns the original result;
- same key + conflicting payload is rejected;
- timeout/unknown result does not create a second logical request;
- future provider create/dispatch retries must reconcile unknown outcomes before repeating provider creation.

## Authentication and public-browser boundary

The intake HTML appears to be a guest-facing browser form, so long-lived secrets must not be embedded in it.

Initial architecture must support:

- HTTPS;
- explicit server validation;
- request/body limits and rate limiting;
- restricted CORS if the HTML and API use different origins;
- sensitive-data-aware logging;
- separate provider credential storage when a provider is added.

The exact hosting origin, API base URL, abuse-control mechanism, and provider authentication remain unresolved deployment/provider decisions.

## Normalized lifecycle

The initial intake can safely start at:

- `requested`

Future normalized phases may include:

- `confirmed`
- `dispatched`
- `completed`
- `cancelled`
- `failed`

The exact meaning and transitions for provider-facing phases are deferred until the transport provider/operating process is confirmed. Provider-native statuses should be preserved separately rather than guessed.

## Remaining unresolved decisions

The HTML evidence resolves the previous accommodation-source and pickup-use-case ambiguity. Remaining architecture/operations decisions are narrower:

1. Where is the HTML actually hosted, and what is its production origin/domain?
2. Where will `PU_Service` be deployed, and will the HTML call it through same-origin routing or cross-origin API access?
3. Which runtime/framework and persistence technology should `PU_Service` use?
4. What external transport/driver provider or manual operating process receives accepted requests?
5. What are confirmation, dispatch, cancellation and modification policies after intake?
6. What status information must guests/operators see after `requested`?
7. What provider authentication/callback verification is required once a provider is selected?
8. What retention/privacy policy applies to guest name, flight, location and note data?
9. The `WSR` detailed address is absent from the provided HTML master data and must be confirmed before production dispatch.

## TASK-001 acceptance checkpoint

Satisfied from current evidence:

- accommodation intake source and manual creation trigger identified;
- passenger airport/station transfer use case identified;
- direction-specific scheduling semantics identified;
- route/stay identifiers and fare rules extracted;
- provider-neutral lifecycle and domain identities documented;
- concrete HTML -> `PU_Service` API boundary selected;
- validation, server fare authority, idempotency, error and observability requirements defined;
- provider-specific concerns isolated behind an explicit adapter boundary rather than invented.

TASK-001 is therefore complete at the integration-contract level. Provider-specific dispatch remains a later dependency and does not block the first concrete implementation task: building the intake API/domain/persistence boundary for the supplied HTML.
