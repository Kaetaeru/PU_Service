# Rerun State

- run_id: `rerun-7f3c6d7a-9a31-4a9d-b54c-4b7fbb1a4d2e`
- sequence: `0`
- task_id: `TASK-001`
- control_status: `complete`
- repository: `Kaetaeru/PU_Service`
- branch/ref: `main`

## Current checkpoint

TASK-001 remains complete. No product implementation has been started by this planning update.

The repository has been reconciled around a clear version boundary: **V1 is the internal operations MVP we use ourselves before any SaaS sale/productization work.**

Current canonical planning sources:

1. `기술적기획/00_CANONICAL_SYSTEM.md`
2. `기술적기획/V1_INTERNAL_MVP.md`
3. `기술적기획/05_DEVELOPMENT_PLAN.md`
4. `기술적기획/README.md`

Older planning documents remain evidence/detail rather than being deleted.

## V1 boundary now fixed

V1 = Development Plan Phases 0~6:

- external WhatsApp/Kakao/backend feasibility;
- managed backend/data model;
- Guest request + WhatsApp receipt;
- Driver dispatch + signed response;
- Guest WhatsApp assignment return + Host manual WhatsApp conversation;
- Host exception/recovery dashboard;
- real internal pilot/hardening.

Post-V1:

- settlement automation / Excel replacement;
- self-service multi-host onboarding;
- SaaS billing/sales;
- iCal;
- cleaning/laundry;
- AI Guest auto-replies;
- broader StayOps modules.

The existing operating ledger remains the settlement tool during V1. Fare/payment snapshots are still stored so later settlement migration does not require reconstructing historical values.

## V1 operational rules

Normal path invariant:

> Host does not manually rewrite Driver messages, re-enter Driver ETA/vehicle data, or rebuild Guest assignment notices.

Allowed Host actions:

- Driver selection and dispatch approval;
- one-click Kakao share/send fallback when automatic outbound is unavailable;
- schedule/flight/payment instruction edits;
- cancellation/redispatch;
- manual Guest free-text reply from WhatsApp Business App;
- exceptional manual assignment correction as recovery only.

V1 must contain recovery for common real-world failures. An internal tool that works only on the happy path is not considered usable V1.

## Canonical product decisions

- Product invariant: normal operations must not require the Host to copy/relay/re-enter information; Host is an exception manager.
- First product scope: airport/station pickup and drop-off round trip.
- Guest default contact coordinate: WhatsApp number normalized to E.164 plus opt-in evidence and preferred language.
- Guest automated operational messages and inbound free-text: WhatsApp.
- Host manual Guest replies: existing WhatsApp Business App when the chosen WhatsApp connection mode supports coexistence.
- V1 does not AI-auto-reply to Guest free-text.
- Driver channel: KakaoTalk notification plus signed Driver Response Web for accept/reject, ETA and vehicle data.
- Driver vehicle/ETA data writes directly into `DriverAssignment`; Host does not re-enter it on the normal path.
- TransferRequest, DriverDispatch and Notification states are separate.
- Managed/serverless backend preferred; Supabase-style Postgres/Auth/Edge Functions is the first candidate to validate.
- All tenant-owned data has `host_id` from the beginning, but V1 UI only needs the internal Host.
- WhatsApp coexistence is preferred but not promised universally; future Host onboarding must test actual eligibility/connection path.
- Kakao outbound delivery remains an external feasibility gate behind `DriverNotificationAdapter`.
- V1 includes payment instruction/snapshot data required to tell the Driver who pays, while full settlement automation is deferred.

## Plan-critic findings incorporated into V1

1. **Major — Happy-path-only internal MVP would fail in real operation.**
   - Added cancel, redispatch, notification retry, schedule change and manual exception recovery to V1.
2. **Major — Kakao full automation could block the entire V1 on an external dependency.**
   - V1 permits a one-click Host share/send fallback while preserving direct Driver response into StayOps.
3. **Major — Payment responsibility was needed before real dispatch.**
   - V1 stores payment instruction and fare/payment snapshots; full settlement is Post-V1.
4. **Major — Sales/multi-host UI would over-expand V1.**
   - Keep `host_id`/RLS in the data model but defer self-service onboarding, billing and generic settings UI.
5. **Major — Previous canonical findings remain in force:** Guest WhatsApp contact, state separation, Coexistence feasibility, and Kakao adapter boundary.

## Preserved validation history

- actual repository verified as `Kaetaeru/PU_Service`, branch `main`;
- original Guest transfer HTML inspected;
- no original backend/persistence existed;
- fare and route/stay rules were extracted;
- client fare was classified as non-authoritative;
- stable request identity, idempotency and persistence were identified as required;
- Driver signed-response design was established;
- newer StayOps requirements and MVP documents were reviewed rather than silently overwritten;
- canonical Guest WhatsApp + Driver Kakao architecture recorded before V1 scoping.

## Next Exact Action

Do not start sales onboarding, settlement automation, cleaning/iCal, or broad dashboard work.

On fresh `continue` authorization, execute TASK-002 against the V1 boundary:

1. validate the internal WhatsApp Business number for inbound/outbound and Host manual-reply mode;
2. validate/select automatic Driver Kakao outbound or the one-click V1 fallback;
3. validate the managed backend choice;
4. finalize V1 schema/API/RLS/token/idempotency/fare-payment snapshot contracts;
5. only then advance to backend implementation.
