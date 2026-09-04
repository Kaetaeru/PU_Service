# Rerun State

- run_id: `rerun-7f3c6d7a-9a31-4a9d-b54c-4b7fbb1a4d2e`
- sequence: `0`
- task_id: `TASK-001`
- control_status: `complete`
- repository: `Kaetaeru/PU_Service`
- branch/ref: `main`

## Current checkpoint

TASK-001 remains complete and product implementation has not started.

On 2026-09-04 a **product-direction revision** was applied across the canonical documents: WhatsApp-only messaging, lead-driver dispatch, payment out of scope, fare table replaced, UI theme fixed. Run identity, sequence, task, and validation history are preserved.

Canonical read order:

1. `기술적기획/00_CANONICAL_SYSTEM.md`
2. `기술적기획/V1_INTERNAL_MVP.md`
3. `기술적기획/06_IMPLEMENTATION_READINESS.md`
4. `기술적기획/07_WhatsApp_배차_왕복_설계.md`
5. `기술적기획/08_요금표.md`
6. `기술적기획/09_UI_테마.md`
7. `기술적기획/05_DEVELOPMENT_PLAN.md`
8. `기술적기획/README.md`

## Earlier gap corrections (still in force)

1. Removed `guest_notified` from TransferRequest state; Notification is the only message-delivery source of truth.
2. No implicit tax uplift on the Guest-facing fare.
3. `revision` binding across TransferRequest, DriverDispatch and Notification so old links/messages cannot act on modified bookings.
4. `needs_reconfirmation` and assignment supersession for material edits after assignment.
5. Conditional airport terminal field and child-seat age/note requirement.
6. WhatsApp inbound MessageEvent request association is nullable when one phone maps to multiple active transfers.
7. Append-only provider event dedup and out-of-order-safe Notification transitions.
8. Stale/cancelled Notification revalidation immediately before send.
9. WSR inactive until its exact address is known.
10. Passenger counts above 7 explicitly rejected instead of silently applying the highest fare tier.
11. Public endpoint security boundary: Guest/dispatch-response browsers do not write directly to tables.
12. Durable Notification outbox/retry worker architecture.
13. V1 stack fixed: Vite + React + TypeScript / Cloudflare Pages / Supabase Postgres + Auth + Edge Functions + RLS.

## 2026-09-04 revision

1. **KakaoTalk removed entirely.** One WhatsApp Business number carries Guest, DispatchPartner and Host messaging. `04_카카오_기사배차_왕복_설계.md` superseded by `07_WhatsApp_배차_왕복_설계.md`.
2. **Driver side is a single `DispatchPartner`** (transport company lead driver). No driver master, no driver accounts, no fan-out, no first-accept race, no automatic redispatch.
3. **Dispatch send is automatic** on the Host's single click; `pending -> awaiting_response` is triggered by send success.
4. **Payment model collapsed** to `payment_confirmed` + who + when. Payment occurs outside the service. `payment_type` / `guest_due_krw` / `host_due_krw` / payment status are dropped. Dispatch is gated on the confirmation.
5. **Dispatch response fields:** available/unavailable, driver phone in E.164 with country code, vehicle plate. ETA not required.
6. **Host receives WhatsApp operational alerts** (new request, assignment done, declined, no response, send failure).
7. **Inbound role routing** (guest / partner / unknown) and 24h window tracking via `wa_contacts` added.
8. **Post-assignment vehicle/driver change is out of V1 scope**, handled manually.
9. **Fare table replaced and fully enumerated** by `08_요금표.md`: 16 confirmed origin/area/direction combinations. New origins ILSAN and EVERLAND are selectable in the Guest form. `ILSAN ↔ HONGDAE` and `EVERLAND ↔ HONGDAE` have no rule and are blocked. Routes without a rule are rejected, never priced at 0.
10. **Guests never see money.** No fare box, no live estimate, no amount in any Guest message or API response. Fare is an internal indicator for the Host and the DispatchPartner only.
11. **Tax restored.** Listed fares are tax-exclusive supply amounts; tax = ROUND(supply × 0.1); Host and DispatchPartner see the tax-inclusive total. This reverses Major B, whose premise (a Guest-visible amount rising 10%) no longer exists, and matches the real operating ledger.
12. **One-way fares are direction-independent except Incheon Airport routes.** Rules store direction explicitly; no implicit "same both ways" logic in code.
13. **Day tours excluded from V1** (recorded in `08_요금표.md` §6.3).
14. **UI theme fixed** by `09_UI_테마.md`, derived from the existing `ems-request.html` / `ems-admin.html` demos.

## Current readiness

**Engineering-ready:** yes.

Provider-independent implementation can start using fake/test messaging adapters.

**Pilot-ready:** no.

Before real Guests:

- actual internal WhatsApp Business inbound/outbound must be tested;
- Host manual reply/Coexistence or equivalent operating mode must be confirmed;
- Guest templates and the partner dispatch template must be approved;
- the lead driver's contact, WhatsApp usability and opt-in must be secured;
- real-device receipt + URL button -> signed response Web must pass;
- inbound role routing must be verified on the shared number;
- opt-in/privacy notice, production backup/secrets must be ready;
- open items in `08_요금표.md` §5 must be resolved.

## Remaining fare gap

Only one is left: **`ILSAN ↔ HONGDAE` and `EVERLAND ↔ HONGDAE` have no rule.** Those combinations are blocked in the Guest form until a price is supplied. Everything else in `08_요금표.md` §2 is confirmed.

## Next Exact Action

On fresh implementation authorization:

1. re-read mandatory Rerun files;
2. re-read canonical/V1/readiness/dispatch/fare/theme docs;
3. create implementation branch `work/v1-internal` from current `main`;
4. start TASK-003 backend foundation:
   - Vite/React/TypeScript skeleton on the `09_UI_테마.md` design system;
   - Supabase structure/migrations/seed;
   - canonical schema + RLS including `dispatch_partners` and `wa_contacts`;
   - fare/validation unit tests against `08_요금표.md`, including tax calculation, direction-specific Incheon pricing, rejection of ruleless routes, and absence of any amount in the Guest response;
   - transactional create-transfer-request boundary;
   - fake Notification adapter;
5. do not implement broad admin, settlement, day tours, iCal, cleaning/laundry, AI chat, or SaaS onboarding.

Parallel external Phase-0 work should validate the actual WhatsApp number, the lead driver's WhatsApp usability, and the partner dispatch template before TASK-006/internal pilot.
