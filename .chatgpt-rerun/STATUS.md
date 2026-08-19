# Rerun Status

Human-facing projection only. Do not use this file as reconciliation source of truth.

- Connection: active
- Run: `rerun-7f3c6d7a-9a31-4a9d-b54c-4b7fbb1a4d2e`
- Sequence: `0`
- Status: `needs_user`
- Current task: `TASK-001 — Define pickup-service integration contract`
- Implementation started: no
- Discovery artifact: `docs/pickup-integration-contract.md`
- Last meaningful checkpoint: provider-neutral integration responsibilities, lifecycle semantics, identifiers, idempotency/retry/error handling, auth boundaries, and observability were documented. TASK-001 is waiting on the accommodation-side source contract and the exact pickup/transport use case/provider/API plus related scheduling, cancellation, status, auth, ownership, and retention decisions.

When the missing decisions are supplied and control returns to `continue`, resume TASK-001 on the same sequence; do not skip directly to implementation.
