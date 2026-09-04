# StayOps V1 — Internal Operations MVP

> 목적: 판매 전에 **우리가 실제 숙소 운영에 매일 사용할 수 있는 최소 제품**의 범위를 고정한다.
> 상위 규칙: `00_CANONICAL_SYSTEM.md`
> 배차 설계: `07_WhatsApp_배차_왕복_설계.md`
> 요금: `08_요금표.md`
> 화면: `09_UI_테마.md`
> 구현 계약: `06_IMPLEMENTATION_READINESS.md`
> 구현 순서: `05_DEVELOPMENT_PLAN.md`

---

## 1. V1 정의

V1은 SaaS 판매판이 아니다.

V1의 목표는 실제 Guest와 실제 운송회사를 대상으로 픽업·드랍오프 한 건을 다음 흐름으로 끝내는 것이다.

```text
Guest Transfer Web
-> StayOps
-> Guest WhatsApp 접수 확인 + Host 알림
-> Host 지불 확인 + [배차 요청] 클릭
-> 대표기사 WhatsApp (요약 + 입력 링크)
-> signed 배차 응답 Web (가능/불가 · 기사 전화번호 · 차량번호)
-> DriverAssignment
-> Guest WhatsApp 배차 안내 + Host 알림
-> completed
```

정상 건에서 Host가 하지 않아야 하는 일:

- 기사 메시지 수동 작성/전달
- 기사 연락처·차량번호 재입력
- Guest 배차안내 재작성/복사

Host가 V1에서 하는 일:

- **지불 확인 후 [배차 요청] 클릭** (정상 건의 유일한 조작)
- 일정 변경/취소
- 배차 불가·무응답 시 다른 경로 수배
- 실패한 메시지 retry
- Guest 자유 문의에 WhatsApp Business App으로 수동 답장
- 예외 상황의 manual correction
- 운행 완료 처리

---

## 2. V1 범위

V1 = `05_DEVELOPMENT_PLAN.md` Phase 0~6.

```text
Phase 0  external feasibility (WhatsApp)
Phase 1  backend/data foundation
Phase 2  Guest request + WhatsApp receipt
Phase 3  대표기사 dispatch + signed response
Phase 4  Guest WhatsApp return + Host manual conversation
Phase 5  Host 운영 화면
Phase 6  internal pilot/hardening
```

V1 release gate는 Phase 6 완료다.

---

## 3. V1에서 제외

- KakaoTalk 연동 전부
- 정산 자동화/Excel 대체, 입금상태 관리
- 지불 분담(게스트지불/호스트부담) 계산
- Guest에게 금액 노출
- 개별 기사 마스터/계정, 다중 기사 fan-out
- 배차 확정 후 차량·기사 변경 흐름
- 당일 투어 상품
- Host self-service signup/onboarding
- SaaS billing/sales
- 범용 multi-host settings UI
- iCal
- 청소/세탁
- AI Guest auto-reply
- StayOps full WhatsApp inbox
- automatic flight tracking
- 복잡한 GPS/live tracking

V1 동안 기존 운영 장부를 계속 사용한다. 단, 이후 이관을 위해 **fare snapshot은 V1부터 보존한다.**

---

## 4. 사용자

### Guest

로그인 없음.

- Guest Transfer Web
- WhatsApp

### DispatchPartner (대표기사)

로그인 없음.

- WhatsApp
- signed 배차 응답 Web

운송회사 배차 담당자 한 명이다. 개별 운행 기사는 회사가 배정하며 StayOps가 관리하지 않는다.

### Host / Operator

내부 계정으로 로그인.

- StayOps 운영 화면
- WhatsApp Business App (Guest 자유 대화)
- 개인 WhatsApp (운영 알림 수신)

V1에는 외부 Host 가입 화면이 없다.

---

## 5. Guest 입력

### Contact — 필수

```text
guest_name
whatsapp_phone_e164
preferred_language
whatsapp_opt_in
whatsapp_opt_in_at
whatsapp_opt_in_policy_version
```

WhatsApp 번호가 기본 연락 좌표다.

### Transfer

```text
direction = pickup | dropoff
service_date
service_time
flight_no
passenger_count
large_luggage_count
origin_code / destination_code
terminal_code
stay_code
child_seat_count
child_seat_notes
terminal_transfer
special_request
```

규칙:

- pickup time = scheduled landing time
- dropoff time = accommodation pickup time
- 공항 픽업이면 flight number 필수
- passenger_count = 1..7
- 8명 이상은 자동예약하지 않고 Host 문의
- ICN terminal = T1/T2/UNKNOWN
- GMP terminal = DOMESTIC/INTERNATIONAL/UNKNOWN
- child_seat_count > 0이면 아동 연령 정보가 포함된 notes 필수
- terminal_transfer는 공항에서만 허용
- **요금 규칙이 없는 노선 조합은 신청을 받지 않는다** (`08_요금표.md`)
- timezone = Asia/Seoul

실시간 GPS는 V1 필수가 아니다.

---

## 6. Fare / 지불

### Fare

```text
공급가 = base_fare_krw + option_fare_krw
세금   = ROUND(공급가 × 0.1)
총액   = 공급가 + 세금
```

- server가 authoritative fare를 계산
- 요금표는 `08_요금표.md`
- **Guest는 금액을 보지 않는다.** 폼·접수 확인·배차 안내 어디에도 금액이 없고, 계산 결과를 클라이언트로 내려보내지도 않는다
- Host 화면과 대표기사 메시지에는 **세금포함 총액**을 표시한다
- 요금표에 없는 노선 조합은 신청을 받지 않는다
- 예약 당시 snapshot(공급가·세금·총액)을 보존한다

### 지불

**지불은 이 서비스 바깥에서 이뤄진다.** V1이 기록하는 것은 확인 사실 하나뿐이다.

```text
payment_confirmed
payment_confirmed_by
payment_confirmed_at
```

`payment_confirmed = false`이면 배차 요청을 발송하지 않는다.

실제 수금/입금/정산은 Post-V1 범위다.

---

## 7. 정상 플로우

### Guest request

```text
Guest form
-> validation
-> authoritative fare
-> TransferRequest revision=1
-> Guest WhatsApp 접수 확인
-> Host WhatsApp 신규 요청 알림
```

### 배차 요청

Host가 확인:

- 필수 정보
- fare snapshot
- **지불 확인**

그 뒤 [배차 요청]을 누르면 DriverDispatch가 생성되고 대표기사에게 자동 발송된다.

```text
StayOps
-> WhatsApp 템플릿 (요약 + URL 버튼)
-> 대표기사
```

Host가 메시지를 작성하거나 채팅방을 고르지 않는다. 발송 성공 시 dispatch가 `awaiting_response`로 넘어간다.

### 배차 응답

signed 배차 응답 Web:

```text
배차 가능 / 불가
기사 전화번호 (국가번호 포함)
차량번호
차량 종류/색상 optional
```

ETA는 V1 필수 항목이 아니다.

### Assignment

수락은 transaction에서:

- token 유효성
- dispatch 상태
- request status/revision
- active assignment 부재

를 확인한 뒤 Assignment를 만든다.

### Guest return

active DriverAssignment 생성 후 Guest WhatsApp Notification과 Host 알림을 queue한다.

Guest 안내 최소 정보:

- booking/reference
- 차량번호
- 기사 전화번호 (국가번호 포함)
- 일시
- 노선
- meeting point

### Completion

V1은 Host가 completed 처리할 수 있다. 자동 trip tracking은 필요 없다.

---

## 8. Revision과 변경

TransferRequest는 `revision`을 가진다.

material edit:

- date/time/direction
- route/stay/terminal
- flight
- passenger/luggage
- child seat/terminal transfer
- 지불 확인 취소

수정 시 `revision += 1`.

DriverDispatch는 생성 당시 `request_revision`을 저장한다. 오래된 revision의 token은 수락할 수 없다.

assignment 이후 material edit:

```text
assigned
-> needs_reconfirmation
-> old assignment superseded
-> new dispatch
-> assigned
```

**배차 확정 후 차량·기사만 바뀌는 경우는 시스템으로 처리하지 않는다.** Host가 수동 대응한다.

---

## 9. V1 WhatsApp

내부 운영 WhatsApp Business 번호 1개로 Guest·대표기사·Host를 모두 처리한다.

목표:

- Guest inbound webhook
- StayOps outbound operational message
- Host가 WhatsApp Business App에서 수동 답장
- 가능하면 Host sent-message echo 저장
- provider message/event dedup
- **inbound 역할 라우팅** (guest / partner / unknown)

필수 자동 메시지:

1. Guest 접수 확인
2. Guest 배차 완료 / 차량 정보
3. Guest 주요 일정 변경 (Host 확정분)
4. 대표기사 배차 요청
5. 대표기사 배차 취소
6. Host 운영 알림 (신규 요청 / 배차 완료 / 배차 불가 / 무응답 / 발송 실패)

Guest 자유 메시지에는 AI 답변을 하지 않는다. 대표기사 자유 텍스트는 파싱하지 않고 링크를 1회 재안내한다.

여러 활성 transfer가 같은 번호에 있으면 message를 임의 예약에 연결하지 않는다.

실제 번호의 Coexistence/동등 경로는 Pilot 전 외부 게이트다.

---

## 10. V1 배차

필수:

- DriverDispatch 생성
- 배차 요청 메시지 자동 구성
- WhatsApp 템플릿 자동 발송
- signed response token
- 가능/불가
- 기사 전화번호 + 차량번호 수집
- expiry/revocation
- stale revision 거부
- 무응답/불가 시 Host 알림
- one request -> one active assignment

전송 수단은 `PartnerNotificationAdapter`로 추상화한다. `whatsapp`이 기본이고 `sms` / `manual` fallback이 있다. 세 경로 모두 같은 signed URL을 보낸다.

---

## 11. V1 Host 운영 화면

CRM 전체가 아니라 **운영/예외 복구 UI**다.

### Today / Upcoming

- 오늘/내일 transfer
- Guest
- time
- route
- 현재 단계
- 차량번호/기사 연락처
- 요금 (공급가 · 세금 · 세금포함 총액)

### 확인 필요

- **지불 확인 대기** (여기서 확인하고 배차 요청을 누른다)
- missing required information
- terminal UNKNOWN
- 배차 요청 미발송
- 대표기사 무응답
- 대표기사 배차 불가
- needs_reconfirmation
- WhatsApp 발송 실패
- Guest message needs reply/association
- schedule conflict
- capacity warning
- cancelled/changed after dispatch

### Request detail actions

- request 상세
- schedule/flight/route 수정
- 지불 확인 체크
- **[배차 요청]**
- 재발송
- cancel
- notification retry
- WhatsApp conversation 열기
- exceptional manual assignment correction + reason
- completed

manual assignment는 정상 경로가 아니라 recovery다.

화면 디자인은 `09_UI_테마.md`를 따른다.

---

## 12. V1 Backend

관리형 backend 사용.

```text
Supabase Postgres
Supabase Auth
Supabase Edge Functions
RLS
Notification durable outbox + scheduled retry worker
```

필수 domain:

```text
Host
Stay
TransferRequest
DispatchPartner
DriverDispatch
DriverAssignment
Notification / MessageEvent
wa_contacts
FareRule
```

모든 tenant-owned row에 host_id를 둔다. V1 UI는 내부 Host 하나만 지원해도 되지만 RLS는 처음부터 검증한다.

Public Guest/배차응답 browser는 DB에 직접 write하지 않고 Edge Function을 통한다.

---

## 13. V1 Failure / Recovery

### WhatsApp 발송 실패

```text
Notification failed
-> 확인 필요 목록
-> retry
-> 필요 시 Host manual 전달
```

### 대표기사 배차 불가 / 무응답

```text
Dispatch rejected/expired
-> Host 알림
-> Host가 다른 경로 수배
```

V1에서 자동 재발송/자동 이관은 하지 않는다.

### Stale link

request revision이 바뀌면 old token accept를 거부한다.

### Cancellation

cancelled request는 새 assignment를 받을 수 없다. 대표기사에게 취소 통보를 보내고 stale queued Notification은 보내지 않는다.

### WSR

정확한 주소가 없으므로 주소 확정 전 V1에서 inactive.

---

## 14. V1 Release Gate

### E2E

- 실제 Guest mobile form
- 실제 DB 저장
- 실제 Guest WhatsApp 접수 확인
- 실제 Host 알림
- Host 지불 확인 + [배차 요청] 1회
- 실제 대표기사 WhatsApp 수신
- 실제 대표기사 signed response
- Host 재입력 없이 Assignment
- 실제 Guest WhatsApp 배차 안내 도착

### Host operation

- 정상 건은 복사/재작성 불필요
- 배차 불가/무응답 시 상황 파악 가능
- change/cancel/reconfirmation 가능
- WhatsApp failure retry 가능
- Guest free-text manual reply 가능

### Reliability

- duplicate Guest submit 방지
- stale/expired token 방지
- 중복 응답 제출에도 active assignment 하나
- stale Notification 발송 방지
- webhook duplicate/out-of-order 안전
- Guest/Partner inbound 역할 분리 정확
- audit trail 확인 가능

### Pilot

- 실제 transfer 약 20건 이상
- P0/P1급 미해결 운영 장애 없음
- 정상 transfer 대부분이 Host 중계 없이 완료
- 남은 수동 작업을 의사결정/예외와 불필요 중계로 구분해 기록

---

## 15. Pilot 전 외부 체크

- 실제 WhatsApp Business inbound/outbound
- Host manual reply 운영 경로
- Guest 템플릿 승인
- 대표기사 템플릿 승인 및 실단말 수신
- 대표기사 연락처 / WhatsApp 사용 가능 여부 / opt-in 증거
- opt-in/privacy notice
- production secrets/backups
- `08_요금표.md` 미확정 항목 확정
- 실제 Stay address 확인 (WSR 포함)

---

## 16. V1 이후

```text
V1 internal-use proven
-> settlement/operating tools
-> 당일 투어 상품
-> multi-host productization
-> external Host pilot
-> sales
-> iCal/cleaning/laundry/broader StayOps
```
