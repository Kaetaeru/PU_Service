# Rerun Status

Human-facing projection only. Do not use this file as reconciliation source of truth.

- Connection: active
- Run: `rerun-7f3c6d7a-9a31-4a9d-b54c-4b7fbb1a4d2e`
- Sequence: `0`
- Control status: `complete`
- Current control task: `TASK-001 — complete`
- Implementation started: no
- Last plan revision: 2026-09-04 — WhatsApp only, lead-driver dispatch

## Canonical planning

- `기술적기획/00_CANONICAL_SYSTEM.md`
- `기술적기획/V1_INTERNAL_MVP.md`
- `기술적기획/06_IMPLEMENTATION_READINESS.md`
- `기술적기획/07_WhatsApp_배차_왕복_설계.md`
- `기술적기획/08_요금표.md`
- `기술적기획/09_UI_테마.md`
- `기술적기획/05_DEVELOPMENT_PLAN.md`

## 2026-09-04 direction change

```text
Messaging   WhatsApp only. KakaoTalk removed entirely.
Dispatch    One DispatchPartner (transport company lead driver).
            No driver master, no fan-out, no automatic redispatch.
Send        Automatic on Host's single click. Send success drives awaiting_response.
Payment     Out of scope. Only payment_confirmed gates dispatch.
Response    available/unavailable + driver phone (E.164) + vehicle plate.
Host        Observer with WhatsApp alerts, not an approver of message content.
Fare        08_요금표.md is the single source, all 16 combinations enumerated.
            Listed amounts are tax-exclusive; tax = ROUND(supply x 0.1).
            Guests never see any amount. Host and DispatchPartner see the total.
            Ruleless routes are rejected, never priced at 0.
Origins     ICN, GMP, SEOULSTN, ILSAN, EVERLAND — all selectable by the Guest.
Tours       Excluded from V1.
UI          09_UI_테마.md, derived from existing ems-* demos.
```

## Current Phase 0 execution evidence

Detailed checkpoint: `기술적기획/PHASE0_VALIDATION_STATUS.md`

As of 2026-09-04:

```text
[PASS] Meta Developer App + WhatsApp Test WABA/Test Number
[PASS] Test Number outbound -> actual WhatsApp device
[PASS] Supabase Edge Function callback verification
[PASS] messages webhook configuration
[PASS] StayOps app subscribed to Test WABA
[PASS] actual WhatsApp device -> Meta -> Supabase inbound POST 200
[PASS] inbound WhatsApp messages payload parsed by Edge Function

[TODO] remove/reduce raw webhook payload logging before real Guest data
[NOT STARTED] actual accommodation WhatsApp Business number Coexistence
[TODO] actual-number API outbound/inbound
[TODO] Host manual Business App reply while API remains connected
[TODO] Guest production template send
[TODO] Guest opt-in/privacy copy

[TODO] lead driver contact + WhatsApp usability confirmation
[TODO] lead driver opt-in evidence
[TODO] partner dispatch template approval
[TODO] real-device receipt + URL button -> signed response Web
[TODO] quick-reply + URL button combination feasibility
[TODO] inbound role routing verification on the shared number
```

The PASS rows are unaffected by the channel change: the Guest leg and the dispatch leg now share the same WhatsApp transport that was already validated.

Important boundary:

- only Meta test assets have been connected and exercised;
- the existing real accommodation WhatsApp Business App account/number has **not** been migrated, disconnected, or enrolled into Coexistence yet;
- test-number feasibility must not be interpreted as real-number Coexistence completion.

## Readiness

- **Engineering-ready: YES**
- **Phase 0 complete: NO**
- **Real Guest pilot-ready: NO**

## Open decisions

One fare gap remains: `ILSAN ↔ HONGDAE` and `EVERLAND ↔ HONGDAE` have no price and are blocked in the Guest form until one is supplied. Everything else in `08_요금표.md` §2 is confirmed.

## Largest current risks

1. Removing Kakao puts Meta template approval on the critical path for the dispatch leg as well as the guest leg. Mitigated by `sms` / `manual` adapter fallbacks that send the same signed URL.
2. The lead driver may not use WhatsApp. This is a business check, not a technical one, and it is the first Phase-0 action on the dispatch side.

## V1 stack

```text
Frontend: Vite + React + TypeScript -> Cloudflare Pages
Backend:  Supabase Postgres/Auth/Edge Functions/RLS
Messaging: WhatsApp Cloud API (single WABA number, role-routed inbound)
Design:   09_UI_테마.md
```

Next Phase 0 action: harden the test webhook logging, then validate the existing accommodation WhatsApp Business number through a Coexistence-safe onboarding path without deleting/disconnecting/migrating it to API-only mode. In parallel, confirm the lead driver's WhatsApp usability.

Next implementation authorization may still create `work/v1-internal` and start TASK-003 backend foundation/provider-independent vertical slice in parallel.
