# StayOps V1 — Internal Operations MVP

> 목적: 판매 전에 **우리가 실제 숙소 운영에 매일 사용할 수 있는 최소 제품**의 범위를 고정한다.
> 상위 규칙: `00_CANONICAL_SYSTEM.md`
> 구현 계약: `06_IMPLEMENTATION_READINESS.md`
> 구현 순서: `05_DEVELOPMENT_PLAN.md`

---

## 1. V1 정의

V1은 SaaS 판매판이 아니다.

V1의 목표는 실제 Guest와 실제 Driver를 대상으로 공항/서울역 픽업·드랍오프 한 건을 다음 흐름으로 끝내는 것이다.

```text
Guest Transfer Web
-> StayOps
-> Guest WhatsApp receipt
-> Driver Kakao Share/notification
-> signed Driver Response
-> DriverAssignment
-> Guest WhatsApp assignment
-> completed
```

정상 건에서 Host가 하지 않아야 하는 일:

- 기사 메시지 수동 재작성
- 기사 ETA/차량번호 재입력
- Guest 배차안내 재작성/복사

Host가 V1에서 하는 일:

- 기사 선택과 dispatch 승인
- payment instruction 확인
- 일정 변경/취소/재배차
- 실패한 메시지 retry
- Guest 자유 문의에 WhatsApp Business App으로 수동 답장
- 예외 상황의 manual correction
- 운행 완료 처리

---

## 2. V1 범위

V1 = `05_DEVELOPMENT_PLAN.md` Phase 0~6.

```text
Phase 0  external feasibility
Phase 1  backend/data foundation
Phase 2  Guest request + WhatsApp receipt
Phase 3  Driver dispatch + signed response
Phase 4  Guest WhatsApp return + Host manual conversation
Phase 5  Host exception dashboard
Phase 6  internal pilot/hardening
```

V1 release gate는 Phase 6 완료다.

---

## 3. V1에서 제외

- 정산 자동화/Excel 대체
- Host self-service signup/onboarding
- SaaS billing/sales
- 범용 multi-host settings UI
- iCal
- 청소/세탁
- AI Guest auto-reply
- StayOps full WhatsApp inbox
- automatic flight tracking
- 복잡한 GPS/live tracking
- multi-driver fan-out marketplace

V1 동안 기존 운영 장부를 계속 사용한다. 단, 이후 이관을 위해 fare/payment snapshot은 V1부터 보존한다.

---

## 4. 사용자

### Guest

로그인 없음.

- Guest Transfer Web
- WhatsApp

### Driver

로그인 없음.

- KakaoTalk
- signed Driver Response Web

### Host / Operator

내부 계정으로 로그인.

- StayOps Admin
- WhatsApp Business App

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
port_code
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
- airport pickup이면 flight number 필수
- passenger_count = 1..7
- 8명 이상은 자동예약하지 않고 Host 문의
- ICN terminal = T1/T2/UNKNOWN
- GMP terminal = DOMESTIC/INTERNATIONAL/UNKNOWN
- child_seat_count > 0이면 아동 연령 정보가 포함된 notes 필수
- SEOULSTN에서는 terminal_transfer 금지
- timezone = Asia/Seoul

실시간 GPS는 V1 필수가 아니다.

---

## 6. Fare / Payment

### Fare

V1 Guest-facing 금액:

```text
base_fare_krw + option_fare_krw = total_fare_krw
```

- server가 authoritative fare를 계산
- 현재 검증된 HTML fare table을 초기 seed로 사용
- rule이 없으면 error
- 예약 당시 snapshot을 보존
- **V1에서 VAT 10%를 암묵적으로 추가하지 않음**

### Payment instruction

```text
payment_type = pending | host | guest | split
payment_instruction_status = pending | confirmed
guest_due_krw
host_due_krw
```

```text
guest_due_krw + host_due_krw = total_fare_krw
```

payment instruction이 confirmed가 아니면 dispatch하지 않는다.

실제 수금/입금 완료 관리는 Post-V1 정산 범위다.

---

## 7. 정상 플로우

### Guest request

```text
Guest form
-> validation
-> authoritative fare
-> TransferRequest revision=1
-> WhatsApp receipt Notification
```

### Dispatch preparation

Host가 확인:

- required info
- fare snapshot
- payment instruction
- Driver

그 뒤 DriverDispatch를 생성한다.

### Driver outbound

V1 baseline:

```text
StayOps dispatch message
-> Kakao Share one-click
-> Host가 기사 채팅방 선택
```

검증된 automatic Kakao adapter가 있으면 대체 가능하다.

Host가 메시지를 직접 재작성하지 않는다.

### Driver response

Driver Response Web:

```text
Accept / Reject
ETA
Vehicle plate
Vehicle model/color optional
```

Driver name/phone은 master를 prefill할 수 있다.

### Assignment

Driver accept는 transaction에서:

- token 유효성
- dispatch 상태
- request status/revision
- active assignment 부재

를 확인한 뒤 Assignment를 만든다.

### Guest return

active DriverAssignment 생성 후 Guest WhatsApp Notification을 queue한다.

최소 안내:

- booking/reference
- driver name
- vehicle plate
- ETA/pickup time
- meeting point
- driver contact if policy allows

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
- payment instruction

수정 시 `revision += 1`.

DriverDispatch는 생성 당시 `request_revision`을 저장한다. 오래된 revision의 token은 수락할 수 없다.

assignment 이후 material edit:

```text
assigned
-> needs_reconfirmation
-> old assignment superseded
-> new/reconfirmed dispatch
-> assigned
```

---

## 9. V1 WhatsApp

내부 운영 WhatsApp Business 번호 1개를 먼저 연결한다.

목표:

- Guest inbound webhook
- StayOps outbound operational message
- Host가 WhatsApp Business App에서 수동 답장
- 가능하면 Host sent-message echo 저장
- provider message/event dedup

필수 자동 메시지:

1. request received
2. driver assigned / vehicle information
3. Host가 확정한 주요 schedule change

Guest 자유 메시지에는 AI 답변을 하지 않는다.

여러 활성 transfer가 같은 번호에 있으면 message를 임의 예약에 연결하지 않는다.

실제 번호의 Coexistence/동등 경로는 Pilot 전 외부 게이트다.

---

## 10. V1 Driver / Kakao

필수:

- DriverDispatch 생성
- 기사 메시지 자동 구성
- signed response token
- accept/reject
- ETA/vehicle
- expiry/revocation
- stale revision 거부
- reject/timeout redispatch
- one request -> one active assignment

Kakao 자동 발송은 V1 필수가 아니다. 공식 Kakao Share 기반 one-click을 V1 baseline으로 사용할 수 있다.

---

## 11. V1 Host Admin

CRM 전체가 아니라 **운영/예외 복구 UI**다.

### Today / Upcoming

- 오늘/내일 transfer
- Guest
- time
- route
- transfer status
- Driver/vehicle

### Needs Attention

- payment pending
- missing required information
- terminal UNKNOWN
- driver not assigned
- dispatch rejected/expired
- needs_reconfirmation
- WhatsApp failed
- Guest message needs reply/association
- schedule conflict
- capacity warning
- cancelled/changed after dispatch

### Request detail actions

- request 상세
- schedule/flight/route/payment 수정
- Driver 선택/dispatch
- redispatch
- cancel
- notification retry
- WhatsApp conversation 열기
- exceptional manual assignment correction + reason
- completed

manual assignment는 정상 경로가 아니라 recovery다.

---

## 12. V1 Backend

관리형 backend 사용.

확정 구현 방향:

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
Driver
DriverDispatch
DriverAssignment
Notification / MessageEvent
FareRule
```

모든 tenant-owned row에 host_id를 둔다. V1 UI는 내부 Host 하나만 지원해도 되지만 RLS는 처음부터 검증한다.

Public Guest/Driver browser는 DB에 직접 write하지 않고 Edge Function을 통한다.

---

## 13. V1 Failure / Recovery

### WhatsApp fail

```text
Notification failed
-> Needs Attention
-> retry
-> 필요 시 Host manual WhatsApp
```

### Driver reject/timeout

```text
Dispatch rejected/expired
-> Host next Driver
-> new Dispatch
```

### Stale driver link

request revision이 바뀌면 old token accept를 거부한다.

### Cancellation

cancelled request는 새 assignment를 받을 수 없다. stale queued Notification은 보내지 않는다.

### WSR

정확한 주소가 없으므로 주소 확정 전 V1에서 inactive.

---

## 14. V1 Release Gate

### E2E

- 실제 Guest mobile form
- 실제 DB 저장
- 실제 Guest WhatsApp receipt
- 실제 Driver Kakao Share/notification
- 실제 Driver signed response
- Host 재입력 없이 Assignment
- 실제 Guest WhatsApp assignment 도착

### Host operation

- 정상 건은 복사/재작성 불필요
- reject/timeout redispatch 가능
- change/cancel/reconfirmation 가능
- WhatsApp failure retry 가능
- Guest free-text manual reply 가능

### Reliability

- duplicate Guest submit 방지
- stale/expired Driver token 방지
- simultaneous accept에서 active assignment 하나
- stale Notification 발송 방지
- webhook duplicate/out-of-order 안전
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
- 필요한 WhatsApp template
- opt-in/privacy notice
- production secrets/backups
- 실제 fare table 확인
- 실제 Driver 목록
- Kakao Share real-device 확인
- WSR inactive 또는 주소 확정

---

## 16. V1 이후

```text
V1 internal-use proven
-> settlement/operating tools
-> multi-host productization
-> external Host pilot
-> sales
-> iCal/cleaning/laundry/broader StayOps
```
