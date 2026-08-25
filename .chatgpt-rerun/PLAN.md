# Rerun Plan

## Project goal

Build StayOps as a managed-backend hospitality operations system whose first executable product is a complete airport/station transfer round trip:

```text
Guest Web/WhatsApp
-> StayOps
-> Driver Kakao + signed response
-> StayOps assignment
-> Guest WhatsApp
```

The normal path must not require the host to copy, relay, or re-enter operational information. The host is an exception manager and may directly answer guest free-text messages from WhatsApp Business App when the selected WhatsApp connection mode supports coexistence.

## Canonical planning sources

Read these before implementation planning:

1. `기술적기획/00_CANONICAL_SYSTEM.md`
2. `기술적기획/05_DEVELOPMENT_PLAN.md`
3. `기술적기획/README.md`

Older `REQUIREMENTS.md`, `MVP_SPEC.md`, and `01~04` documents remain evidence/detail. If they conflict with the canonical documents, the canonical documents win.

## TASK-001 — Define initial pickup integration contract

**Status:** complete

Preserved validation:

- original Guest transfer HTML inspected;
- browser-only fare/share behavior identified;
- server-authoritative validation/fare/idempotency requirement established;
- stable request identity/persistence boundary established;
- route/stay codes and initial fare rules extracted;
- initial driver dispatch adapter boundary established.

## Planning refinement after TASK-001

Subsequent product decisions are now canonicalized:

- Guest contact coordinate = WhatsApp number in E.164 form + opt-in evidence;
- Guest operational notifications and inbound free-text = WhatsApp;
- Host manual Guest conversation = WhatsApp Business App when coexistence is supported;
- Driver operational channel = KakaoTalk + signed Driver Response URL;
- Driver reply data goes directly into `DriverAssignment`; Host must not re-enter vehicle/ETA data on the normal path;
- transfer, driver-dispatch, and notification states are separate;
- managed/serverless backend is preferred; Supabase-style Postgres/Auth/Edge Functions is the first architecture candidate to validate;
- all tenant-owned data has `host_id` from the start;
- WhatsApp coexistence and Kakao outbound delivery are external feasibility gates, not assumed universal capabilities.

## TASK-002 — Feasibility + implementation architecture

**Status:** ready, not started

### Goal

Remove the external-integration risks and make the first executable slice implementation-ready without building broad product UI first.

### Required outcomes

1. Validate a real WhatsApp Business onboarding path:
   - inbound webhook;
   - outbound message/template;
   - coexistence + Business App manual reply/message echo when available;
   - explicit fallback when unavailable.
2. Select and validate the Driver Kakao notification route or the temporary one-click fallback.
3. Confirm managed backend/runtime choice.
4. Finalize canonical tables, RLS/tenant isolation, idempotency, fare snapshot, signed driver response token design.
5. Turn the current Guest input contract into an implementation-ready API/schema.

### Acceptance criteria

- no unverified messaging capability is required by the first implementation task;
- backend technology and deployment boundary are explicit;
- WhatsApp/Kakao adapters have concrete testable contracts;
- Guest -> StayOps -> Driver -> StayOps -> Guest can be implemented as the next vertical slice;
- no implementation of cleaning/laundry/iCal/AI inbox is required.

## TASK-003 — Backend foundation

**Status:** blocked on TASK-002

Follow `05_DEVELOPMENT_PLAN.md` Phase 1.

## TASK-004 — Guest request + WhatsApp receipt

**Status:** blocked on TASK-003

Follow Phase 2.

## TASK-005 — Driver dispatch round trip

**Status:** blocked on TASK-004

Follow Phase 3.

## TASK-006 — Guest WhatsApp return + Host manual conversation sync

**Status:** blocked on TASK-005

Follow Phase 4.

## TASK-007 — Host exception dashboard + pilot

**Status:** blocked on TASK-006

Follow Phases 5-6.

## Later

Settlement, multi-host productization, iCal, cleaning, laundry and other StayOps modules follow only after the transfer round trip is proven in real operations.

## Constraints

- Do not put long-lived secrets in Guest/Driver browser code.
- Client fare is not authoritative.
- Preserve fare snapshots for historical settlement.
- Do not treat message delivery as driver assignment.
- Do not auto-parse Driver Kakao free-text as the canonical response path.
- Do not auto-reply to Guest free-text with AI in MVP.
- Do not promise WhatsApp coexistence to every Host before onboarding validation.
- Enforce tenant isolation at the database level.
- Preserve Rerun run identity and validation history.
