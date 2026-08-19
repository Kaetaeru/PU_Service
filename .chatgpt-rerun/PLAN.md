# Rerun Plan

## Project goal

Build `PU_Service` as the service layer for connecting accommodation operations with a pickup/transport service. The repository is currently new and contains no product implementation, so the first work must establish the integration contract before implementation assumptions are made.

## Task graph

### TASK-001 — Define pickup-service integration contract

**Goal**

Turn the current product intent into an implementation-ready contract for the accommodation pickup workflow.

**Dependencies**

- Confirm the accommodation-side source of booking/guest/pickup data.
- Confirm the external pickup/transport provider or API, if one exists.
- Confirm required pickup lifecycle states, cancellation rules, scheduling fields, and authentication expectations.
- Confirm whether this repository owns only backend/service logic or also deployment/API gateway concerns.

**Acceptance criteria**

- The pickup request lifecycle is explicitly documented from accommodation booking/context through dispatch, status updates, completion, cancellation, and failure handling.
- Required domain entities and identifiers are listed without inventing provider-specific fields.
- External integration boundaries and authentication requirements are identified or explicitly marked as unresolved user decisions.
- Error handling, idempotency, retry, observability, and data-retention expectations are captured at a level sufficient to choose an implementation architecture.
- A concrete next implementation task can be selected without guessing missing provider or accommodation interfaces.

**Validation**

- Re-read repository instructions and Rerun state in mandatory order.
- Verify every acceptance criterion is either satisfied with repository/user evidence or recorded as an unresolved dependency.
- Do not begin implementation until unresolved external contracts that materially affect architecture are resolved.

## Planned follow-up tasks

These are provisional and must be refined after TASK-001:

- TASK-002 — Define service architecture and API/domain model.
- TASK-003 — Implement the agreed integration skeleton and configuration boundaries.
- TASK-004 — Add contract/unit/integration validation for pickup lifecycle behavior.

## Constraints

- Do not invent accommodation-system or pickup-provider APIs.
- Preserve Rerun run identity and validation history on future reconciliations.
- Use `PLAN.md -> STATE.md -> control.json` for authoritative state transitions.
