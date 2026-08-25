# Rerun State

- run_id: `rerun-7f3c6d7a-9a31-4a9d-b54c-4b7fbb1a4d2e`
- sequence: `0`
- task_id: `TASK-001`
- control_status: `complete`
- repository: `Kaetaeru/PU_Service`
- branch/ref: `main`

## Current checkpoint

TASK-001 remains complete. No product implementation has been started by this planning update.

The repository later expanded from the original `PU_Service` intake concept into the broader StayOps hospitality-operations product. A plan-critic review reconciled the newer `REQUIREMENTS.md` / `MVP_SPEC.md`, the earlier HTML/API analysis, the Kakao driver round-trip design, and the latest Guest WhatsApp + Host manual-reply direction.

Two canonical planning documents now define the current system:

1. `기술적기획/00_CANONICAL_SYSTEM.md`
2. `기술적기획/05_DEVELOPMENT_PLAN.md`

`기술적기획/README.md` is the entry point and explicitly declares the canonical read order. Older planning documents are retained as evidence/detail rather than deleted.

## Canonical product decisions

- Product invariant: normal operations must not require the Host to copy/relay/re-enter information; Host is an exception manager.
- First product scope: airport/station pickup and drop-off round trip.
- Guest default contact coordinate: WhatsApp number normalized to E.164 plus opt-in evidence and preferred language.
- Guest automated operational messages and inbound free-text: WhatsApp.
- Host manual Guest replies: existing WhatsApp Business App when the chosen WhatsApp connection mode supports coexistence.
- MVP does not AI-auto-reply to Guest free-text.
- Driver channel: KakaoTalk notification plus signed Driver Response Web for accept/reject, ETA and vehicle data.
- Driver vehicle/ETA data writes directly into `DriverAssignment`; Host does not re-enter it on the normal path.
- TransferRequest, DriverDispatch and Notification states are separate.
- Managed/serverless backend preferred; Supabase-style Postgres/Auth/Edge Functions is the first candidate to validate.
- All tenant-owned data has `host_id` from the beginning.
- WhatsApp coexistence is preferred but not promised universally; Host onboarding must test actual eligibility/connection path.
- Kakao outbound delivery remains an external feasibility gate behind `DriverNotificationAdapter`.

## Plan-critic material findings resolved

1. **Major — Host remained a manual relay in old MVP.**
   - Corrected normal path to Driver response -> Assignment -> Guest WhatsApp automatic return.
2. **Major — No stable asynchronous Guest contact.**
   - Added WhatsApp contact/opt-in contract.
3. **Major — Coexistence was at risk of being treated as universal.**
   - Added explicit `coexistence | api_only | manual_only` connection-mode concept and onboarding feasibility gate.
4. **Major — Dispatch and message states were conflated.**
   - Split TransferRequest / DriverDispatch / Notification state machines.
5. **Major — Kakao proactive delivery path is still unverified.**
   - Isolated behind an adapter and moved validation to TASK-002/Phase 0.

## Preserved validation history

- actual repository verified as `Kaetaeru/PU_Service`, branch `main`;
- original Guest transfer HTML inspected;
- no original backend/persistence existed;
- fare and route/stay rules were extracted;
- client fare was classified as non-authoritative;
- stable request identity, idempotency and persistence were identified as required;
- driver signed-response design was established;
- newer StayOps requirements and MVP documents were reviewed rather than silently overwritten.

## External capability notes

Current research supports WhatsApp Business App + Cloud API coexistence as a real capability with message mirroring/message echoes through supported onboarding paths, but eligibility/provider/onboarding details must be tested per Host before it becomes a sales promise.

## Next Exact Action

Do not jump directly to broad UI or cleaning/iCal work.

On fresh `continue` authorization, execute TASK-002 using `기술적기획/00_CANONICAL_SYSTEM.md` and `기술적기획/05_DEVELOPMENT_PLAN.md`:

1. validate WhatsApp inbound/outbound + coexistence/manual-reply echo with a real test business number or selected provider path;
2. validate/select the Driver Kakao outbound route or explicit one-click fallback;
3. validate the managed backend choice;
4. finalize implementation-ready schema/API/RLS/token/idempotency contracts;
5. only then advance to backend implementation.
