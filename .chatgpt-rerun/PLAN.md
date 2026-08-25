# Rerun Plan

## Project goal

Build StayOps first as an **internal-use V1** for real accommodation operations, not as a sellable SaaS.

The first executable product is a complete airport/station transfer round trip:

```text
Guest Web/WhatsApp
-> StayOps
-> Driver Kakao + signed response
-> StayOps assignment
-> Guest WhatsApp
```

The normal path must not require the Host to copy, relay, or re-enter operational information. The Host is an exception manager and may directly answer Guest free-text messages from WhatsApp Business App when the selected WhatsApp connection mode supports coexistence.

## Canonical planning sources

Read these before implementation planning:

1. `기술적기획/00_CANONICAL_SYSTEM.md`
2. `기술적기획/V1_INTERNAL_MVP.md`
3. `기술적기획/05_DEVELOPMENT_PLAN.md`
4. `기술적기획/README.md`

Older `REQUIREMENTS.md`, `MVP_SPEC.md`, and `01~04` documents remain evidence/detail. If they conflict with the canonical/V1 documents, the newer canonical/V1 decisions win.

## V1 boundary

V1 means **we can operate real transfers ourselves before selling the product**.

V1 includes Development Plan Phases 0~6:

```text
Phase 0  external feasibility
Phase 1  backend/data foundation
Phase 2  Guest request + WhatsApp receipt
Phase 3  Driver dispatch + signed response
Phase 4  Guest WhatsApp return + Host manual conversation
Phase 5  Host exception dashboard
Phase 6  real pilot/hardening
```

V1 explicitly does not require:

- settlement automation / Excel replacement;
- self-service multi-host onboarding;
- SaaS billing or sales;
- iCal, cleaning, laundry;
- AI Guest auto-replies;
- a full WhatsApp inbox inside StayOps.

The existing operating ledger remains in use during V1. Fare/payment snapshots are still stored from V1 so settlement can be migrated later.

### V1 operational allowances

The following Host actions are allowed and do not violate the product invariant:

- selecting a Driver and starting dispatch;
- one-click Kakao share/send when automatic outbound is not yet available;
- editing schedule/flight/payment instruction;
- cancelling/redispatching;
- manually answering Guest free-text in WhatsApp Business App;
- exceptional manual assignment correction as a recovery path.

The following are not allowed on the normal path:

- manually rewriting the Driver message;
- re-entering Driver ETA/vehicle data after the Driver already supplied it;
- manually rebuilding the Guest assignment message.

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
- all tenant-owned data has `host_id` from the start, but V1 UI only needs the internal Host;
- WhatsApp coexistence and Kakao outbound delivery are external feasibility gates, not assumed universal capabilities;
- V1 must preserve payment instruction/snapshot data needed for actual Driver operations even though full settlement automation is Post-V1.

## TASK-002 — Feasibility + implementation architecture

**Status:** ready, not started

### Goal

Remove the external-integration risks and make the first V1 executable slice implementation-ready without building broad product UI first.

### Required outcomes

1. Validate the actual internal WhatsApp Business number/onboarding path:
   - inbound webhook;
   - outbound message/template;
   - coexistence + Business App manual reply/message echo when available;
   - explicit fallback when unavailable.
2. Select and validate the Driver Kakao notification route or the V1 one-click fallback.
3. Confirm managed backend/runtime choice.
4. Finalize canonical tables, RLS/tenant isolation, idempotency, fare/payment snapshot, signed Driver response token design.
5. Turn the current Guest input contract into an implementation-ready API/schema.

### Acceptance criteria

- no unverified messaging capability is required by the first implementation task;
- backend technology and deployment boundary are explicit;
- WhatsApp/Kakao adapters have concrete testable contracts;
- Guest -> StayOps -> Driver -> StayOps -> Guest can be implemented as the next vertical slice;
- V1 recovery paths are defined;
- no implementation of settlement, cleaning, laundry, iCal, sales onboarding, or AI inbox is required.

## TASK-003 — Backend foundation

**Status:** blocked on TASK-002

Follow `05_DEVELOPMENT_PLAN.md` Phase 1 and `V1_INTERNAL_MVP.md`.

## TASK-004 — Guest request + WhatsApp receipt

**Status:** blocked on TASK-003

Follow Phase 2.

## TASK-005 — Driver dispatch round trip

**Status:** blocked on TASK-004

Follow Phase 3.

## TASK-006 — Guest WhatsApp return + Host manual conversation sync

**Status:** blocked on TASK-005

Follow Phase 4.

## TASK-007 — Host exception dashboard + internal pilot

**Status:** blocked on TASK-006

Follow Phases 5-6. Completion of this task is the V1 release gate.

## Post-V1

After internal-use V1 is proven:

1. settlement/operating tools;
2. multi-host onboarding/productization;
3. external Host pilot and sales;
4. iCal/cleaning/laundry/broader StayOps.

## Constraints

- Do not put long-lived secrets in Guest/Driver browser code.
- Client fare is not authoritative.
- Preserve fare and payment snapshots for historical settlement.
- Do not treat message delivery as Driver assignment.
- Do not auto-parse Driver Kakao free-text as the canonical response path.
- Do not auto-reply to Guest free-text with AI in V1.
- Do not promise WhatsApp coexistence to every future Host before onboarding validation.
- Enforce tenant isolation at the database level even though V1 has only internal Host UI.
- Allow operational fallbacks only when they avoid copying/re-entering canonical data.
- Preserve Rerun run identity and validation history.
