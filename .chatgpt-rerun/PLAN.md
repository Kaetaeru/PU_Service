# Rerun Plan

## Project goal

Build `PU_Service` as the server-side service layer behind the Stay Pachira airport transfer request interface, then connect accepted requests to the eventual transport/driver provider through an isolated adapter boundary.

## Task graph

### TASK-001 — Define pickup-service integration contract

**Status:** complete

**Goal**

Turn the product intent into an implementation-ready contract for the accommodation transfer intake workflow without inventing an external provider API.

**Evidence incorporated**

The user supplied `TalkFile_pickup.html`, which resolved the accommodation-side intake ambiguity. The page is a guest-facing airport/station transfer form with manual submit, direction-specific schedule semantics, stable port/stay codes, local fare calculation, and browser share/copy output. It has no backend request or persistence.

Technical planning artifacts were created:

- `기술적기획/README.md`
- `기술적기획/01_HTML_분석.md`
- `기술적기획/02_PU_Service_연결_설계.md`
- `기술적기획/03_API_계약_초안.md`

The consolidated contract at `docs/pickup-integration-contract.md` was updated with this evidence.

**Acceptance criteria**

- Pickup request lifecycle documented from intake through future provider dispatch/completion/cancellation/failure. **Satisfied at normalized/provider-neutral level.**
- Required domain entities and identifiers listed without inventing provider fields. **Satisfied.**
- External integration and authentication boundaries identified or explicitly deferred. **Satisfied.**
- Error handling, idempotency, server-side fare authority, retry/reconciliation and sensitive-data considerations captured sufficiently to choose the first architecture boundary. **Satisfied.**
- Concrete next implementation task selectable without guessing the accommodation interface or provider API. **Satisfied:** first implementation can build the HTML intake API/domain/persistence boundary while provider dispatch remains behind an adapter.

**Validation**

- Rerun documents were re-read in mandatory order before work.
- Repository state was checked before adding planning artifacts.
- Uploaded HTML was inspected directly and its form controls, client fare data, submit handler and share/copy behavior were analyzed.
- No server call/provider API was inferred where none exists.
- Browser fare and validation were explicitly treated as non-authoritative.
- External provider-specific behavior remains deferred rather than invented.

### TASK-002 — Select implementation architecture and finalize intake domain/API

**Status:** ready, not started

**Goal**

Turn the technical planning contract into an implementation blueprint for the first `PU_Service` executable slice.

**Starting scope**

- choose runtime/framework;
- choose persistence technology and migration strategy;
- define source tree/module boundaries;
- finalize `POST /v1/transfer-requests` and `GET /v1/transfer-requests/{id}`;
- finalize domain/value objects and fare policy ownership;
- define idempotency persistence strategy;
- decide same-origin versus CORS deployment boundary based on actual hosting coordinates;
- define tests for the extracted HTML fare/validation rules;
- keep transport-provider adapter as an interface until provider details exist.

**Dependencies / decisions still needed during TASK-002**

- actual HTML production origin/domain;
- desired/available runtime or hosting constraints for `PU_Service`;
- persistence/deployment preference if one already exists;
- `WSR` production address before real dispatch;
- provider/driver process is not required for intake implementation but is required before dispatch implementation.

**Acceptance criteria**

- technology choices are explicit and justified;
- API/domain/storage contracts are implementation-ready;
- fare rules have an authoritative server representation and test vectors;
- HTML integration path and deployment/CORS boundary are explicit;
- provider adapter interface is defined without provider-specific fabrication;
- TASK-003 can implement a minimal end-to-end intake path.

### TASK-003 — Implement HTML intake service slice

**Status:** blocked on TASK-002

Provisional scope:

- service skeleton/config;
- transfer request domain;
- fare calculation;
- persistence;
- idempotent create API;
- request read API;
- validation/errors;
- automated tests.

### TASK-004 — Wire HTML and add operational/provider integration

**Status:** blocked on TASK-003 and provider/operations decisions

Provisional scope:

- replace local-only submit with API call;
- server-response ticket/driver message;
- loading/retry/idempotency UX;
- provider/operator adapter and lifecycle integration when available.

## Constraints

- Do not embed long-lived secrets in guest HTML.
- Client-side fare is estimate only; server fare is authoritative.
- Do not invent transport-provider endpoints/status/auth.
- Preserve Rerun run identity and validation history.
- Use `PLAN.md -> STATE.md -> control.json` for authoritative state transitions; `STATUS.md` is projection only.
