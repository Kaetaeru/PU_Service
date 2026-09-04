# Rerun Plan

## Project goal

Build StayOps first as an **internal-use V1** for real accommodation transfer operations, not as a sellable SaaS.

Canonical normal path:

```text
Guest Web
-> StayOps
-> Guest WhatsApp receipt + Host WhatsApp notice
-> Host confirms payment, clicks [배차 요청] once
-> DispatchPartner (lead driver) WhatsApp template + signed link
-> signed dispatch response Web (available/unavailable, driver phone, vehicle plate)
-> DriverAssignment
-> Guest WhatsApp assignment + Host WhatsApp notice
```

The Host must not write partner messages, re-enter driver phone/vehicle data, or rebuild Guest assignment notices on the normal path. The Host's only normal-path action is confirming payment and pressing one button.

## Canonical planning sources

Read before implementation:

1. `기술적기획/00_CANONICAL_SYSTEM.md`
2. `기술적기획/V1_INTERNAL_MVP.md`
3. `기술적기획/06_IMPLEMENTATION_READINESS.md`
4. `기술적기획/07_WhatsApp_배차_왕복_설계.md`
5. `기술적기획/08_요금표.md`
6. `기술적기획/09_UI_테마.md`
7. `기술적기획/05_DEVELOPMENT_PLAN.md`
8. `기술적기획/README.md`

Older documents are evidence/detail only when they conflict. `기술적기획/04_카카오_기사배차_왕복_설계.md` is **superseded** by `07`.

## V1 boundary

V1 = Development Plan Phases 0~6.

Post-V1: settlement automation, day tours, self-service multi-host productization, sales/billing, iCal, cleaning/laundry, AI Guest auto-replies.

## 2026-09-04 plan revision — WhatsApp only, lead-driver dispatch

A product-direction change was applied across the canonical documents. This did not reset the Rerun run identity.

### What changed

1. **KakaoTalk removed entirely.** All outbound and inbound messaging goes through one WhatsApp Business number: Guest, DispatchPartner, and Host operational alerts.
2. **Driver side collapsed to one contact.** StayOps talks to a transport company's lead driver (`DispatchPartner`), who assigns an actual driver internally and reports back. StayOps keeps no driver master, no driver accounts, no fan-out, no first-accept race.
3. **Dispatch send is automatic.** Host confirms payment and clicks once; StayOps sends the approved template. `DriverDispatch: pending -> awaiting_response` is triggered by send success, not by a Host share action.
4. **Payment model collapsed.** Payment happens outside the service. `payment_type` / `guest_due_krw` / `host_due_krw` / payment status tracking are dropped. Only `payment_confirmed` + who + when remain, and it gates dispatch.
5. **Dispatch response fields fixed.** available/unavailable, driver phone in E.164 (country code included so foreign guests can dial), vehicle plate. ETA is not required in V1.
6. **Host became an observer.** Host receives WhatsApp alerts for new request, assignment done, dispatch declined, no response, and send failure.
7. **Post-assignment vehicle/driver change is out of scope.** Handled manually; no re-entry flow, no long-lived response link.
8. **Fare table replaced and fully enumerated.** `08_요금표.md` lists all 16 confirmed origin/area/direction combinations. New origins ILSAN and EVERLAND are selectable in the Guest form. `ILSAN ↔ HONGDAE` and `EVERLAND ↔ HONGDAE` have no rule yet and are blocked in the form.
9. **Guests never see money.** No fare box, no live estimate, no amount in any Guest message or API response. Fare exists only for the Host and the DispatchPartner.
10. **Tax restored.** Listed fares are tax-exclusive supply amounts; tax = ROUND(supply × 0.1). Host and DispatchPartner see the tax-inclusive total. Major B's earlier "no tax in V1" conclusion is reversed because its premise (a Guest-visible amount rising 10%) no longer exists. This matches the real operating ledger.
11. **Day tours excluded from V1.** Nami Island / DMZ / Seoul city recorded in `08_요금표.md` §6.3 for later.
12. **UI theme fixed.** `09_UI_테마.md` captures the design system from the existing `ems-request.html` / `ems-admin.html` demos. Three screens share one stylesheet.

### New Phase-0 risk

Removing Kakao puts Meta template approval on the critical path for the driver leg as well as the guest leg. Mitigation: `PartnerNotificationAdapter` with `whatsapp` primary, `sms` fallback, `manual` last resort. All three send the same signed URL, so the domain never changes.

The largest open business risk is whether the lead driver uses WhatsApp at all. Confirming that is the first Phase-0 action on the dispatch side.

## TASK-001 — Initial pickup integration contract

**Status:** complete

Preserved evidence:

- original Guest transfer HTML inspected;
- route/stay/fare/add-on rules extracted;
- browser-only behavior identified;
- server-authoritative validation/fare/idempotency/persistence required;
- signed dispatch-response boundary established.

## TASK-002 — Feasibility + implementation architecture

**Status:** architecture/readiness contract complete; external account checks pending

### Decisions now fixed

- frontend: Vite + React + TypeScript, design system from `09_UI_테마.md`;
- hosting: Cloudflare Pages;
- backend: Supabase Postgres + Auth + Edge Functions + RLS;
- notification durable outbox + scheduled retry worker;
- public Guest/dispatch-response writes go through Edge Functions, not direct table DML;
- Host operations UI uses Auth + RLS;
- Guest contact = WhatsApp E.164 + opt-in evidence;
- **all messaging is WhatsApp; KakaoTalk is not used;**
- **dispatch contact is a single `DispatchPartner`; no driver master;**
- **dispatch send is automatic on Host click; send success drives `awaiting_response`;**
- dispatch response = opaque signed/bearer token page, token hash at rest;
- dispatch response fields = available/unavailable + driver phone (E.164) + vehicle plate;
- **payment is out of scope; only `payment_confirmed` gates dispatch;**
- single WABA number with inbound role routing (guest / partner / unknown);
- 24h service-window tracking via `wa_contacts` decides template vs freeform;
- TransferRequest / DriverDispatch / DriverAssignment / Notification states separated;
- `guest_notified` removed from Transfer status;
- TransferRequest revision binds DriverDispatch and Notification, preventing stale accept/send;
- assignment material edits use `needs_reconfirmation` and supersede old assignment;
- fare rules owned by server per `08_요금표.md`; **missing rule is an error, never 0;**
- **Guests never see any amount** — not in the form, not in any message, not in the API response;
- listed fares are tax-exclusive supply amounts; tax = ROUND(supply × 0.1); Host and DispatchPartner see the tax-inclusive total;
- one-way fares are direction-independent except Incheon Airport routes; rules store direction explicitly;
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
2. lead driver (DispatchPartner):
   - contact obtained;
   - **WhatsApp actually usable by that person;**
   - opt-in evidence;
   - dispatch request template approved;
   - real-device receipt and URL button opening the signed response Web;
   - quick-reply + URL button combination feasibility (for the 24h window).
3. inbound role routing verified on the shared number.
4. Pilot policy gate:
   - WhatsApp opt-in/privacy notice;
   - production secret/backup policy;
   - open items in `08_요금표.md` §5 resolved.

These checks block **real Guest pilot**, not provider-independent code construction.

## TASK-003 — Backend foundation

**Status:** ready after development authorization

Follow `06_IMPLEMENTATION_READINESS.md` first, then Development Plan Phase 1.

First implementation scope:

- branch `work/v1-internal`;
- Vite/React/TS skeleton using the `09_UI_테마.md` design system;
- Supabase migrations + seed;
- canonical tables/RLS (including `dispatch_partners`, `wa_contacts`);
- fare/validation tests against `08_요금표.md`;
- transactional `create_transfer_request`;
- fake/test Notification adapter.

Do not wait for production WhatsApp credentials to implement provider-independent schema/domain/tests.

## TASK-004 — Guest request + WhatsApp receipt

**Status:** blocked on TASK-003

Includes the Host new-request notification.

## TASK-005 — Dispatch request + signed response

**Status:** blocked on TASK-004

Host payment confirmation gate, automatic WhatsApp template send, signed response Web with driver phone + vehicle plate, assignment transaction.

## TASK-006 — Guest WhatsApp return + Host manual conversation

**Status:** blocked on TASK-005 and actual WhatsApp Phase-0 path

Includes inbound role routing.

## TASK-007 — Host operations screen + internal pilot

**Status:** blocked on TASK-006

Completion is the V1 release gate.

## Constraints

- WhatsApp only; no KakaoTalk;
- no long-lived browser secrets;
- client fare non-authoritative;
- **missing fare rule is an error, not 0 KRW;**
- no implicit V1 tax uplift;
- no passenger >7 automatic top-tier pricing;
- **payment must be confirmed before dispatch send;**
- stale request revision cannot be accepted/notified;
- message delivery is not assignment state;
- no free-text parsing of partner replies;
- no Guest AI free-text auto reply in V1;
- no driver master, no fan-out, no automatic redispatch in V1;
- no post-assignment vehicle/driver change flow in V1;
- RLS/host isolation from first migration;
- fallbacks may require one operator click but must not require canonical data re-entry;
- preserve Rerun run identity and validation history.
