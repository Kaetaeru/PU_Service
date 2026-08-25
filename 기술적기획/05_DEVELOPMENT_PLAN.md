# StayOps Development Plan

> 기준 문서: `00_CANONICAL_SYSTEM.md`
> 목적: 첫 실운영 가능한 픽업/드랍오프 왕복 흐름을 가장 짧은 경로로 완성한 뒤 확장한다.

## 개발 원칙

1. 정상 플로우에서 호스트의 복사/재전달/재입력을 제거한다.
2. 첫 성공 기준은 "폼이 예쁘다"가 아니라 **Guest -> StayOps -> Driver -> StayOps -> Guest 왕복이 실제로 끝난다**다.
3. WhatsApp/Kakao 공급자 제약은 adapter 뒤에 둔다.
4. 멀티호스트는 나중에 판매하지만 `host_id`/tenant isolation은 처음부터 넣는다.
5. AI 자동 자유대화, 청소/세탁/iCal은 첫 왕복이 안정화되기 전에는 구현하지 않는다.
6. 각 단계는 다음 단계가 의존하는 위험을 먼저 제거한다.

---

# Phase 0 — External feasibility gates

## 목표

제품의 핵심이 외부 메시징 채널 때문에 막히지 않는지 실제 계정으로 확인한다.

## 해야 할 것

### WhatsApp

- 실제 WhatsApp Business 번호 1개로 API 연결 경로 확인
- 가능하면 Coexistence onboarding 성공
- Guest -> business inbound webhook 수신
- API -> Guest 테스트 메시지 발송
- Business App 수동 답장 -> message echo 수신 확인
- provider message id 기반 dedup 확인
- business-initiated template 발송 경로 확인

### Kakao

- 기사에게 선제적으로 보낼 실제 메시지 경로 결정
- 버튼/URL 포함 가능 여부 확인
- signed Driver Response URL이 기사 휴대폰에서 정상 열리는지 확인
- 자동 Kakao가 즉시 불가능하면 MVP fallback을 "한 번의 원클릭 share"로 명시하되 도메인/DB 흐름은 동일하게 유지

### Backend

- managed backend 후보의 최소 spike
- 1차 권장 Supabase에서 Postgres/Auth/Edge Function/RLS가 요구에 맞는지 검증

## 완료조건

- WhatsApp의 실제 inbound/outbound 경로가 동작한다.
- 호스트 수동 WhatsApp 운영과 API 자동화를 함께 쓸 수 있는 연결 모드가 확인되거나 명시적인 fallback이 선택된다.
- 기사 알림 전달 경로가 하나 선택된다.
- 이후 개발이 특정 공급자 미확정 때문에 다시 설계되지 않는다.

## 실패 시

이 단계를 우회해서 UI부터 대규모 개발하지 않는다. 외부 채널의 실제 제약에 맞춰 adapter/fallback만 수정한다.

---

# Phase 1 — Backend foundation and canonical data model

## 목표

모든 이후 화면/메시지가 의존할 영속 상태를 만든다.

## 구현 범위

- project/config/environment 분리
- database migrations
- `hosts`
- `stays`
- `transfer_requests`
- `drivers`
- `driver_dispatches`
- `driver_assignments`
- `notifications/message_events`
- `fare_rules`
- idempotency 저장
- audit timestamps/events
- tenant isolation/RLS

## 핵심 API/Function

```text
POST /transfer-requests
GET  /transfer-requests/{secure-reference}
```

내부 service:

```text
calculateFare()
createTransferRequest()
transitionTransferStatus()
```

## 요금

기존 HTML 요금표를 초기 seed로 옮기고 server authoritative calculation을 만든다.

예약 시:

```text
fare rule lookup
-> fare 계산
-> fare_snapshot 저장
```

과거 요청은 이후 요금표 수정으로 변경되지 않는다.

## 완료조건

- 조작된 client 가격을 보내도 서버 가격만 저장됨
- 중복 submit/retry가 하나의 logical request만 생성
- 다른 host 계정으로 tenant 데이터를 읽지 못함
- validation/fare 단위 테스트 통과

---

# Phase 2 — Guest Request MVP

## 목표

실제 게스트가 모바일에서 요청을 생성할 수 있게 한다.

## Guest UI

기존 `pickup_2.html`의 UX를 기반으로 다음을 확정한다.

### Guest

- Main guest name *
- Country code + WhatsApp number *
- Preferred language *
- WhatsApp operational messaging opt-in *

### Transfer

- Pickup / Drop-off *
- Date *
- Time *
- Flight number (pickup at airport: required)
- Passenger count *
- Large luggage count

### Route

- ICN / GMP / Seoul Station
- Accommodation
- terminal/meeting point when applicable

### Options

- Child seat count
- Terminal transfer
- Special request

## Submit

```text
Guest form
-> server validation
-> server fare
-> TransferRequest
-> confirmation page
```

## WhatsApp receipt

요청 생성 후 Guest에게 접수 확인 메시지를 보낸다.

실제 자동발송이 Phase 0에서 막힌 경우에도 notification queue/state는 구현하고 test adapter로 검증한다.

## 완료조건

- 실제 휴대폰에서 Guest form 제출 가능
- request id 생성
- DB 저장
- 서버 요금 일치
- WhatsApp 연락처 E.164 정규화
- opt-in evidence 저장
- receipt notification 결과 추적

---

# Phase 3 — Driver dispatch round trip

## 목표

호스트가 차량번호를 다시 입력하지 않아도 기사 응답이 StayOps에 직접 들어오게 한다.

## DriverDispatch

Host/운영 규칙이 특정 기사에게 dispatch를 만든다.

```text
TransferRequest
-> DriverDispatch
-> DriverNotificationAdapter
```

## 기사 메시지

포함 정보:

- 일시
- pickup/dropoff
- guest display name
- passenger/luggage
- flight
- route
- add-ons
- signed response button/link

민감정보는 운영상 필요한 최소 범위만 노출한다.

## Driver Response Web

로그인 없는 token 기반 모바일 페이지.

```text
Accept / Reject
ETA
Vehicle plate
Driver name
Driver phone
Model/color optional
```

## 확정 transaction

수락 시:

```text
validate token + dispatch
-> ensure request not cancelled/already assigned
-> create DriverAssignment
-> mark dispatch accepted
-> mark transfer assigned
```

## 재배차

- reject -> 새 dispatch 가능
- timeout -> expired -> 새 dispatch 가능
- 동시 중복 assignment는 DB constraint/transaction으로 차단

## 완료조건

- 기사 휴대폰에서 링크로 수락/거절 가능
- 차량번호/ETA가 자동으로 request와 연결
- 중복/만료 token 거부
- 하나의 request에 assignment 하나만 확정

---

# Phase 4 — Guest WhatsApp return + host manual conversation

## 목표

배차 결과가 자동으로 Guest에게 돌아가고, 자유 문의는 호스트가 기존 WhatsApp Business App에서 직접 처리한다.

## Assignment notification

`DriverAssignment` 생성 이벤트:

```text
-> Guest WhatsApp notification
```

메시지 최소 정보:

- booking/reference
- driver name
- vehicle plate
- ETA
- meeting point
- driver contact if policy allows

## Incoming guest messages

```text
WhatsApp webhook
-> normalize contact
-> associate active/recent transfer(s)
-> MessageEvent 저장
-> needs_host_reply 표시
```

애매하게 여러 활성 예약이 매칭되면 자동으로 임의 예약에 붙이지 않고 운영자 확인 대상으로 둔다.

## Host manual reply

Coexistence 모드에서는 Host가 WhatsApp Business App에서 직접 답장한다.

```text
Business App reply
-> message echo
-> MessageEvent 저장
```

MVP에서 AI free-text auto reply는 하지 않는다.

## 완료조건

- Driver assignment 후 Guest에게 실제 메시지 도착
- Guest 답장이 webhook으로 저장
- Host가 Business App에서 수동 답장 가능
- echo가 중복 없이 StayOps에 기록
- 자동 상태 알림과 사람 수동 답장이 서로를 덮어쓰지 않음

---

# Phase 5 — Host exception dashboard

## 목표

호스트가 정상 건을 처리하는 화면이 아니라 **문제가 있는 건을 빠르게 발견·해결하는 화면**을 만든다.

## 첫 화면 우선순위

- awaiting driver
- driver rejected/expired
- WhatsApp failed
- guest needs reply
- missing information
- schedule conflict
- capacity warning
- upcoming today/tomorrow

## 필요한 조작

- request 상세 조회
- 시간/항공편 수정
- 기사 재배차
- 취소
- 메시지 재시도
- WhatsApp conversation 열기
- payment/settlement metadata 수정

## 완료조건

실운영자가 정상 요청은 건드리지 않고 예외 요청만 처리할 수 있다.

---

# Phase 6 — Pilot hardening

## 목표

Stay Pachira/관련 실제 숙소에서 사람이 쓰는 시스템으로 검증한다.

## 검증 항목

- 20~50건 실제 요청
- duplicate submit
- guest wrong number
- WhatsApp unavailable
- driver reject
- driver timeout
- guest schedule change
- flight delay
- notification failure/retry
- cancellation after dispatch
- simultaneous updates
- tenant isolation

## 운영 측정

핵심 KPI:

```text
host manual relay count per transfer
request -> driver assigned time
driver response timeout rate
WhatsApp delivery failure rate
manual exception rate
```

가장 중요한 KPI는 **호스트가 한 예약당 몇 번 중계를 해야 했는가**다.

## 완료조건

정상 예약의 대부분이 호스트의 복사/재입력 없이 끝난다.

---

# Phase 7 — Settlement and operating tools

## 목표

검증된 픽업 데이터를 현행 엑셀 정산을 대체할 수 있는 운영 데이터로 확장한다.

## 범위

- payment type
- guest paid
- host amount
- tax/supply snapshot
- partner settlement
- monthly summary
- Excel/TSV export

기존 운영 장부를 regression fixture로 사용한다.

---

# Phase 8 — Multi-host productization

## 목표

내부 운영 도구를 다른 호스트가 스스로 가입해 사용할 수 있는 상품으로 만든다.

## onboarding

```text
Sign up
-> Host 생성
-> Stay 등록
-> Fare rules
-> Driver contacts
-> WhatsApp connection test
-> Guest request link 발급
```

## 필수

- host-specific branding
- tenant isolation
- settings UI
- WhatsApp connection state
- driver/provider settings
- usage/operational logs

과금 기능은 실제 판매 검증 뒤에 추가한다.

---

# Phase 9 — StayOps broader automation

픽업 왕복이 안정화된 뒤에만 확장한다.

후보:

- iCal/reservation ingestion
- cleaning schedule
- laundry
- check-in/check-out messages
- reminders
- partner settlement expansion
- multilingual templates

각 모듈은 `Reservation/Event -> Partner task -> Partner response -> Guest/Host output -> Settlement` 패턴을 재사용한다.

---

# 권장 실제 착수 순서

```text
0. WhatsApp/Kakao feasibility
1. DB + fare + tenant foundation
2. Guest form + WhatsApp receipt
3. Driver dispatch + signed response
4. Guest WhatsApp automatic return + host manual reply sync
5. Exception dashboard
6. Real pilot
7. Settlement
8. Multi-host productization
9. Cleaning/iCal/etc.
```

가장 먼저 만들 UI가 Host dashboard일 필요는 없다. 가장 먼저 증명해야 하는 제품 가치는 **한 건의 실제 transfer가 게스트 입력부터 기사 배차와 게스트 반환까지 사람의 중계 없이 왕복하는 것**이다.
