# Rerun Plan

## Project goal

Build `PU_Service` as the service layer for connecting accommodation operations with a pickup/transport service. The repository is new and contains no product implementation. Integration contracts must be established before architecture or implementation assumptions are made.

## Task graph

### TASK-001 — Define pickup-service integration contract

**Status:** active, waiting for user decisions (`needs_user` checkpoint)

**Goal**

Turn the current product intent into an implementation-ready contract for the accommodation pickup workflow.

**Evidence-backed work completed**

A provider-neutral discovery contract is recorded at `docs/pickup-integration-contract.md`. It documents:

- minimum logical service boundary;
- lifecycle semantics from intent receipt through validation, dispatch, provider acknowledgement, active service, completion, cancellation, and failure;
- provider-neutral entities/identifiers;
- idempotency and duplicate-protection requirements;
- retry/error classification;
- authentication/secret boundaries;
- observability and reconciliation requirements;
- architecture-affecting unresolved decisions.

**Unresolved dependencies**

The following must be confirmed before TASK-001 can be completed safely:

1. accommodation-side source system/repository/API and delivery mechanism;
2. exact pickup/transport use case and external provider/API, if any;
3. automatic versus manual/on-demand creation trigger;
4. scheduling/timezone/change semantics;
5. cancellation/modification ownership and allowed states;
6. accommodation-visible statuses and provider status delivery/query mechanism;
7. authentication/verification methods on both integration boundaries;
8. `PU_Service` ownership of persistence, API exposure, background processing, deployment, and gateway concerns;
9. retention/privacy requirements for guest/contact/location data.

**Acceptance criteria**

- The pickup request lifecycle is explicitly documented from accommodation booking/context through dispatch, status updates, completion, cancellation, and failure handling. **Partially satisfied:** provider-neutral semantics are documented; final transitions/status mapping depend on provider/business policy.
- Required domain entities and identifiers are listed without inventing provider-specific fields. **Satisfied at discovery level.**
- External integration boundaries and authentication requirements are identified or explicitly marked as unresolved user decisions. **Satisfied.**
- Error handling, idempotency, retry, observability, and data-retention expectations are captured at a level sufficient to choose an implementation architecture. **Partially satisfied:** all categories are documented, but retention and provider-specific retry/auth behavior remain unresolved.
- A concrete next implementation task can be selected without guessing missing provider or accommodation interfaces. **Not yet satisfied.**

**Validation**

- Mandatory Rerun read order was completed for this dispatch.
- Repository root was rechecked and contains only `.chatgpt-rerun` plus the TASK-001 discovery document added during this run; no product implementation exists.
- Every TASK-001 acceptance criterion was reviewed against repository/user evidence.
- Missing external contracts were not invented; they are explicitly enumerated for user resolution.

## Planned follow-up tasks

These remain provisional until TASK-001 is finalized:

- TASK-002 — Define service architecture and API/domain model.
- TASK-003 — Implement the agreed integration skeleton and configuration boundaries.
- TASK-004 — Add contract/unit/integration validation for pickup lifecycle behavior.

## Constraints

- Do not invent accommodation-system or pickup-provider APIs.
- Do not start TASK-002 while architecture-affecting TASK-001 dependencies remain unresolved.
- Preserve Rerun run identity, sequence, task, and validation history across reconciliation.
- Use `PLAN.md -> STATE.md -> control.json` for authoritative state transitions; `STATUS.md` remains a human-facing projection.