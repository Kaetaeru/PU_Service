# StayOps 기술적 기획 — 시작점

StayOps의 핵심 목표는 **Host가 Guest와 Partner 사이의 수동 중계자가 되지 않게 하는 것**이다. 첫 제품은 공항/역/거점 픽업·드랍오프이고, V1은 판매 전 우리가 실제 운영에 직접 쓰는 내부 MVP다.

**모든 외부 메시지는 WhatsApp 하나로 나간다. KakaoTalk은 사용하지 않는다.**

## 반드시 먼저 읽을 것

새 기획/개발 작업은 아래 순서로 읽는다.

1. `00_CANONICAL_SYSTEM.md` — 제품 동작, 불변조건, canonical domain/state
2. `V1_INTERNAL_MVP.md` — 내부 V1 포함/제외 범위와 release gate
3. `06_IMPLEMENTATION_READINESS.md` — 구현 계약, 보안/동시성/테스트/stack
4. `07_WhatsApp_배차_왕복_설계.md` — 배차 왕복 상세 설계
5. `08_요금표.md` — 요금 단일 기준
6. `09_UI_테마.md` — 화면 디자인 기준
7. `05_DEVELOPMENT_PLAN.md` — 단계별 구현 순서
8. 필요한 경우에만 이전 근거 문서

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
[TODO] Guest production template
[TODO] 대표기사 배차 요청 template
[TODO] 대표기사 연락처 / WhatsApp 사용 가능 여부 / opt-in
[TODO] 대표기사 실단말에서 URL 버튼 -> 배차 응답 Web 오픈
[TODO] Guest/Partner inbound 역할 라우팅 검증
[TODO] opt-in/privacy 문구
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
        +----> Host WhatsApp        신규 요청 / 배차 완료 / 이상 알림
        |
        +----> 대표기사 WhatsApp     배차 요청 template + URL 버튼
        |           -> signed 배차 응답 Web
        |           -> 가능/불가 + 기사 전화번호 + 차량번호
        |
        +----> Guest WhatsApp        접수 확인 / 배차 안내

Host
  - StayOps 운영 화면: 지불 확인 -> [배차 요청], 예외 복구
  - WhatsApp Business App: Guest 자유 대화 수동 답장
```

정상 transfer에서 Host의 조작은 **지불 확인 후 [배차 요청] 클릭 1회**뿐이다. 기사 메시지 작성, 차량번호·연락처 재입력, Guest 배차안내 작성을 하지 않는다.

---

## V1 경계

V1 = `05_DEVELOPMENT_PLAN.md` Phase 0~6.

```text
external feasibility
-> backend/data
-> Guest request
-> 대표기사 dispatch 왕복
-> Guest WhatsApp return
-> Host 운영 화면
-> internal pilot
```

Post-V1:

- 정산 자동화 / Excel 대체
- 당일 투어 상품
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
- 공항 terminal 구조화 (공항 거점일 때만)
- 거점 5곳: 인천공항 · 김포공항 · 서울역 · 일산 · 용인 에버랜드
- child seat 사용 시 아동 연령 note 필수
- **금액 비노출**

### Fare

```text
공급가 = 기본 + 옵션
세금   = ROUND(공급가 × 0.1)
총액   = 공급가 + 세금
```

- server authoritative
- 요금표 단일 기준은 `08_요금표.md` (표기 금액은 세금 별도 공급가)
- **Guest는 금액을 보지 않는다.** 폼·접수 확인·배차 안내 어디에도 없다
- **Host와 대표기사는 세금포함 총액을 본다**
- **요금표에 없는 노선 조합은 신청을 받지 않는다** (현재 일산↔홍대, 에버랜드↔홍대)
- 편도는 방향 무관 동일 금액. 인천공항만 방향별로 다름
- fare snapshot(공급가·세금·총액) 보존

### 지불

**지불은 이 서비스 바깥에서 이뤄진다.**

```text
payment_confirmed
payment_confirmed_by
payment_confirmed_at
```

`payment_confirmed = false`이면 배차 요청을 발송하지 않는다. 분담 계산·입금상태 관리·세금은 V1 범위 밖이다.

### 배차

- 연락 상대는 **운송회사 대표기사 한 명**
- 개별 운행 기사는 StayOps가 관리하지 않는다
- V1 baseline outbound = WhatsApp template 자동 발송
- signed opaque token 응답 웹
- 응답 항목: 가능/불가 · 기사 전화번호(국가번호 포함) · 차량번호
- 자유 텍스트 답장 파싱 안 함
- Host가 차량정보를 재입력하지 않음
- 배차 확정 후 차량·기사 변경은 시스템으로 처리하지 않음 (Host 수동 대응)

### State

Transfer / DriverDispatch / DriverAssignment / Notification을 분리한다.

`guest_notified`는 Transfer 상태가 아니다.

`DriverDispatch`의 `pending -> awaiting_response`는 **발송 성공**이 트리거다.

### Revision

Transfer material edit마다 revision을 올리고 old token/Notification을 stale 처리한다.

### Backend

- Supabase Postgres/Auth/Edge Functions
- RLS
- public Guest/배차응답 write는 Edge Function만 사용
- Notification durable outbox + retry worker
- 단일 WABA 번호 + inbound 역할 라우팅 (guest / partner / unknown)
- frontend: Vite + React + TypeScript
- hosting: Cloudflare Pages

### 화면

- 디자인 기준은 `09_UI_테마.md` (기존 `ems-request.html` / `ems-admin.html` 데모의 디자인 시스템)
- 손님 신청 폼 / 배차 응답 Web / Host 운영 화면이 같은 스타일시트를 공유
- UI 프레임워크·컴포넌트 라이브러리 추가하지 않음

---

## 현재 Source of Truth

- `00_CANONICAL_SYSTEM.md`
- `V1_INTERNAL_MVP.md`
- `06_IMPLEMENTATION_READINESS.md`
- `07_WhatsApp_배차_왕복_설계.md`
- `08_요금표.md`
- `09_UI_테마.md`
- `05_DEVELOPMENT_PLAN.md`

## 현재 실행 상태 / 검증 증거

- `PHASE0_VALIDATION_STATUS.md`

## 이전 근거 / 초안

- `01_HTML_분석.md`
- `02_PU_Service_연결_설계.md`
- `03_API_계약_초안.md`
- `04_카카오_기사배차_왕복_설계.md` — **폐기.** `07`이 대체
- `REQUIREMENTS.md`
- `MVP_SPEC.md`

이전 문서의 KakaoTalk 연동, Host 차량정보 재입력, Guest Kakao 반환, `waiting/assigned/notified` 단일 상태, VAT 자동 가산, `payment_type` 분담 모델, 개별 기사 마스터 등은 위 Source of Truth와 충돌하면 사용하지 않는다.

## 운영 근거

- `Stay_Pachira_픽업운영장부_2.xlsx`
- `pickup_2.html` — 입력 항목·문구 reference (시각 디자인 기준 아님)

---

## 개발 시작 기준

**Engineering-ready:** `06_IMPLEMENTATION_READINESS.md` 기준으로 코드 작성 가능.

**Pilot-ready:** 실제 WhatsApp Business 번호의 inbound/outbound + Host manual reply 운영 경로, Guest·대표기사 template, opt-in/privacy, production secret/backup, 대표기사 실단말 왕복이 검증된 뒤.

다음 구현은 넓은 운영 화면이 아니라 provider-independent backend foundation과 Guest -> 대표기사 -> Guest vertical slice부터 시작한다.
