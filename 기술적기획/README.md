# StayOps 기술적 기획 — 시작점

StayOps의 핵심 목표는 **Host가 Guest와 Partner 사이의 수동 중계자가 되지 않게 하는 것**이다. 첫 제품은 공항/서울역 픽업·드랍오프이고, V1은 판매 전 우리가 실제 운영에 직접 쓰는 내부 MVP다.

## 반드시 먼저 읽을 것

새 기획/개발 작업은 아래 순서로 읽는다.

1. `00_CANONICAL_SYSTEM.md`
   - 제품 동작, 불변조건, canonical domain/state
2. `V1_INTERNAL_MVP.md`
   - 내부 V1 포함/제외 범위와 release gate
3. `06_IMPLEMENTATION_READINESS.md`
   - 최종 plan-critic 결과, 구현 계약, 보안/동시성/테스트/stack
4. `05_DEVELOPMENT_PLAN.md`
   - 단계별 구현 순서
5. 필요한 경우에만 이전 근거 문서

충돌 시 위 순서가 우선한다.

현재 실제 외부 연동 검증 진행상황은 `PHASE0_VALIDATION_STATUS.md`에서 별도로 추적한다. 이 파일은 canonical 요구사항을 바꾸는 문서가 아니라 **Phase 0 실행 증거/체크포인트**다.

---

## 현재 Phase 0 상태 — 2026-09-04

상세 증거: `PHASE0_VALIDATION_STATUS.md`

```text
[PASS] Meta Test WABA / Test Number 준비
[PASS] Test Number -> 실제 WhatsApp 단말 outbound
[PASS] Supabase Edge Function webhook verification
[PASS] StayOps app -> Test WABA subscription
[PASS] 실제 WhatsApp 단말 -> Meta -> Supabase inbound POST 200
[PASS] inbound messages payload parse

[TODO] production raw webhook logging 제거/축소
[TODO] 실제 숙소 WhatsApp Business 번호 Coexistence
[TODO] 실제 번호 outbound/inbound + Host manual reply
[TODO] production template
[TODO] opt-in/privacy 문구
[TODO] Kakao Share real-device Driver link test
```

**중요:** 실제 숙소 운영 WhatsApp Business 번호에는 아직 migration/disconnect/Coexistence 작업을 시작하지 않았다. 테스트 번호 검증과 실제 번호 검증을 혼동하지 않는다.

---

## 현재 큰 구조

```text
Guest Web + WhatsApp
        |
        v
      StayOps
  Supabase managed backend
        |
        +----> Driver Kakao Share/notification
        |           -> signed Driver Response Web
        |           -> accept/reject + ETA + vehicle
        |
        +----> Guest WhatsApp

Host
  - StayOps Admin: approval / exception recovery
  - WhatsApp Business App: Guest free-text manual reply
```

정상 transfer에서는 Host가 기사 메시지, 차량번호/ETA, Guest 배차안내를 다시 작성하지 않는다.

---

## V1 경계

V1 = `05_DEVELOPMENT_PLAN.md` Phase 0~6.

```text
external feasibility
-> backend/data
-> Guest request
-> Driver round trip
-> Guest WhatsApp return
-> Host exception UI
-> internal pilot
```

Post-V1:

- settlement automation/Excel replacement
- multi-host productization
- SaaS billing/sales
- iCal
- cleaning/laundry
- AI Guest replies

---

## 개발 직전 확정 사항

### Guest

- 기본 연락처: WhatsApp E.164 번호
- opt-in evidence 저장
- preferred language 저장
- passenger 1~7
- large luggage 구조화
- airport terminal 구조화
- child seat 사용 시 아동 연령 note 필수

### Fare

```text
base + options = total_fare_krw
```

- server authoritative
- 현재 HTML fare table을 seed
- V1에서 근거 없이 VAT 10%를 추가하지 않음
- fare snapshot 보존

### Payment

```text
pending | host | guest | split
guest_due_krw + host_due_krw = total_fare_krw
```

payment instruction confirmed 후에만 dispatch.

### Driver

- V1 baseline outbound = Kakao Share one-click
- signed opaque token response
- free-text Kakao 답장 파싱 안 함
- Host가 ETA/차량정보 재입력하지 않음

### State

Transfer / DriverDispatch / DriverAssignment / Notification을 분리한다.

`guest_notified`는 Transfer 상태가 아니다.

### Revision

Transfer material edit마다 revision을 올리고 old Driver token/Notification을 stale 처리한다.

### Backend

- Supabase Postgres/Auth/Edge Functions
- RLS
- public Guest/Driver write는 Edge Function만 사용
- Notification durable outbox + retry worker
- frontend: Vite + React + TypeScript
- hosting: Cloudflare Pages

---

## 현재 Source of Truth

- `00_CANONICAL_SYSTEM.md`
- `V1_INTERNAL_MVP.md`
- `06_IMPLEMENTATION_READINESS.md`
- `05_DEVELOPMENT_PLAN.md`

## 현재 실행 상태 / 검증 증거

- `PHASE0_VALIDATION_STATUS.md`

## 이전 근거 / 초안

- `01_HTML_분석.md`
- `02_PU_Service_연결_설계.md`
- `03_API_계약_초안.md`
- `04_카카오_기사배차_왕복_설계.md`
- `REQUIREMENTS.md`
- `MVP_SPEC.md`

이전 문서의 Host 차량정보 재입력, Guest Kakao 반환, `waiting/assigned/notified` 단일 상태, VAT 자동 가산 등은 canonical/V1/readiness 계약과 충돌하면 사용하지 않는다.

## 운영 근거

- `Stay_Pachira_픽업운영장부_2.xlsx`
- `pickup_2.html`

---

## 개발 시작 기준

**Engineering-ready:** `06_IMPLEMENTATION_READINESS.md` 기준으로 코드 작성 가능.

**Pilot-ready:** 실제 WhatsApp Business 번호의 inbound/outbound + Host manual reply 운영 경로, template/opt-in/privacy, production secret/backup, real-device 3자 왕복이 검증된 뒤.

다음 구현은 넓은 Dashboard가 아니라 provider-independent backend foundation과 Guest -> Driver -> Guest vertical slice부터 시작한다.
