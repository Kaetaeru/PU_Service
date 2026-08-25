# Rerun Plan

## Project goal

Build StayOps first as an **internal-use V1** for real accommodation transfer operations, not as a sellable SaaS.

Canonical normal path:

```text
Guest Web/WhatsApp
-> StayOps
-> Driver Kakao Share/notification + signed response
-> StayOps DriverAssignment
-> Guest WhatsApp
```

The Host must not copy/rewrite Driver messages, re-enter Driver ETA/vehicle data, or rebuild Guest assignment notices on the normal path.

## Canonical planning sources

Read before implementation:

1. `기술적기획/00_CANONICAL_SYSTEM.md`
2. `기술적기획/V1_INTERNAL_MVP.md`
3. `기술적기획/06_IMPLEMENTATION_READINESS.md`
4. `기술적기획/05_DEVELOPMENT_PLAN.md`
5. `기술적기획/README.md`

Older documents are evidence/detail only when they conflict.

## V1 boundary

V1 = Development Plan Phases 0~6.

Post-V1: settlement automation, self-service multi-host productization, sales/billing, iCal, cleaning/laundry, AI Guest auto-replies.

## TASK-001 — Initial pickup integration contract

**Status:** complete

Preserved evidence:

- original Guest transfer HTML inspected;
- route/stay/fare/add-on rules extracted;
- browser-only behavior identified;
- server-authoritative validation/fare/idempotency/persistence required;
- Driver signed-response boundary established.

## TASK-002 — Feasibility + implementation architecture

**Status:** architecture/readiness contract complete; external account checks pending

### Decisions now fixed

- frontend: Vite + React + TypeScript, existing pickup UI/CSS reused;
- hosting: Cloudflare Pages;
- backend: Supabase Postgres + Auth + Edge Functions + RLS;
- notification durable outbox + scheduled retry worker;
- public Guest/Driver writes go through Edge Functions, not direct table DML;
- Host Admin uses Auth + RLS;
- Guest contact = WhatsApp E.164 + opt-in evidence;
- V1 Driver outbound baseline = Kakao Share one-click; automatic Kakao is optional;
- Driver response = opaque signed/bearer token page, token hash at rest;
- TransferRequest / DriverDispatch / DriverAssignment / Notification states separated;
- `guest_notified` removed from Transfer status;
- TransferRequest revision binds DriverDispatch and Notification, preventing stale accept/send;
- assignment material edits use `needs_reconfirmation` and supersede old assignment;
- V1 Guest fare = current verified fare table + options, no implicit VAT 10% addition;
- payment = pending|host|guest|split with guest_due + host_due = fare total;
- WSR inactive until exact address exists;
- inbound WhatsApp message may have nullable request association;
- webhook provider events deduplicated and handled append-only/out-of-order safe;
- Notification outbox rechecks request status/revision before sending.

Detailed contract: `기술적기획/06_IMPLEMENTATION_READINESS.md`.

### Remaining Phase-0 external checks

1. actual internal WhatsApp Business number:
   - inbound webhook;
   - outbound operational/template message;
   - Host manual reply operating mode;
   - Coexistence/message echo when available.
2. real-device Kakao Share:
   - Host selects intended Driver chat;
   - signed Driver URL opens correctly.
3. Pilot policy gate:
   - WhatsApp opt-in/privacy notice;
   - production secret/backup policy;
   - actual Driver seed/contact;
   - final real Guest fare confirmation.

These checks block **real Guest pilot**, not provider-independent code construction.

## TASK-003 — Backend foundation

**Status:** ready after development authorization

Follow `06_IMPLEMENTATION_READINESS.md` first, then Development Plan Phase 1.

First implementation scope:

- branch `work/v1-internal`;
- Vite/React/TS skeleton;
- Supabase migrations + seed;
- canonical tables/RLS;
- fare/validation tests;
- transactional `create_transfer_request`;
- fake/test Notification adapter.

Do not wait for production WhatsApp credentials to implement provider-independent schema/domain/tests.

## TASK-004 — Guest request + WhatsApp receipt

**Status:** blocked on TASK-003

## TASK-005 — Driver dispatch + signed response

**Status:** blocked on TASK-004

Kakao Share is acceptable V1 baseline.

## TASK-006 — Guest WhatsApp return + Host manual conversation

**Status:** blocked on TASK-005 and actual WhatsApp Phase-0 path

## TASK-007 — Host exception dashboard + internal pilot

**Status:** blocked on TASK-006

Completion is the V1 release gate.

## Constraints

- no long-lived browser secrets;
- client fare non-authoritative;
- no implicit V1 VAT 10% uplift;
- no passenger >7 automatic top-tier pricing;
- payment instruction confirmed before dispatch;
- stale request revision cannot be accepted/notified;
- message delivery is not assignment state;
- no Driver Kakao free-text parsing;
- no Guest AI free-text auto reply in V1;
- RLS/host isolation from first migration;
- fallbacks may require one operator click but must not require canonical data re-entry;
- preserve Rerun run identity and validation history.
