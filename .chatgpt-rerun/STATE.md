# Rerun State

- run_id: `rerun-7f3c6d7a-9a31-4a9d-b54c-4b7fbb1a4d2e`
- sequence: `0`
- task_id: `TASK-001`
- control_status: `complete`
- repository: `Kaetaeru/PU_Service`
- branch/ref: `main`

## Current checkpoint

TASK-001 remains complete and product implementation has not started.

The final pre-development plan-critic pass is complete. StayOps V1 is now both product-scoped and implementation-ready at the architecture/contract level.

Canonical read order:

1. `기술적기획/00_CANONICAL_SYSTEM.md`
2. `기술적기획/V1_INTERNAL_MVP.md`
3. `기술적기획/06_IMPLEMENTATION_READINESS.md`
4. `기술적기획/05_DEVELOPMENT_PLAN.md`
5. `기술적기획/README.md`

## Final gap corrections

1. Removed `guest_notified` from TransferRequest state; Notification is the only message-delivery source of truth.
2. Removed the unsupported V1 assumption that 10% VAT is automatically added to the original HTML fare. V1 authoritative Guest fare is current verified fare table + options.
3. Added `revision` binding across TransferRequest, DriverDispatch and Notification so old Driver links/messages cannot act on modified bookings.
4. Added `needs_reconfirmation` and assignment supersession for material edits after assignment.
5. Replaced ambiguous `guest_paid` dispatch semantics with `payment_type`, `guest_due_krw`, `host_due_krw`, and confirmed payment instruction.
6. Added conditional airport terminal field and child-seat age/note requirement.
7. Made WhatsApp inbound MessageEvent request association nullable when one phone maps to multiple active transfers.
8. Added append-only provider event dedup and out-of-order-safe Notification transitions.
9. Added stale/cancelled Notification revalidation immediately before send.
10. Marked WSR inactive until its exact address is known.
11. Explicitly reject passenger counts above 7 instead of silently applying the highest fare tier.
12. Added public endpoint security boundary: Guest/Driver browsers do not write directly to tables.
13. Added durable Notification outbox/retry worker architecture.
14. Fixed the V1 stack: Vite + React + TypeScript / Cloudflare Pages / Supabase Postgres + Auth + Edge Functions + RLS.
15. Fixed V1 Driver outbound baseline to Kakao Share one-click; automatic Kakao is optional and no longer blocks V1 engineering.

## Current readiness

**Engineering-ready:** yes.

Provider-independent implementation can start using fake/test messaging adapters.

**Pilot-ready:** no.

Before real Guests:

- actual internal WhatsApp Business inbound/outbound must be tested;
- Host manual reply/Coexistence or equivalent operating mode must be confirmed;
- required WhatsApp template and opt-in/privacy notice must be prepared;
- production backup/secrets and real Driver seed must be verified;
- real-device Kakao Share + Driver response link must pass.

## Next Exact Action

On fresh implementation authorization:

1. re-read mandatory Rerun files;
2. re-read canonical/V1/readiness docs;
3. create implementation branch `work/v1-internal` from current `main`;
4. start TASK-003 backend foundation:
   - Vite/React/TypeScript skeleton;
   - Supabase structure/migrations/seed;
   - canonical schema + RLS;
   - fare/validation unit tests;
   - transactional create-transfer-request boundary;
   - fake Notification adapter;
5. do not implement broad Admin, settlement, iCal, cleaning/laundry, AI chat, or SaaS onboarding.

Parallel external Phase-0 work should validate the actual WhatsApp number and real-device Kakao Share before TASK-006/internal pilot.
