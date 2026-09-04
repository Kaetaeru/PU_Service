# StayOps Development Plan

> 기준 문서: `00_CANONICAL_SYSTEM.md`
> 목적: 첫 실운영 가능한 픽업/드랍오프 왕복 흐름을 가장 짧은 경로로 완성한 뒤 확장한다.

## 개발 원칙

1. 정상 플로우에서 호스트의 복사/재전달/재입력을 제거한다.
2. 첫 성공 기준은 "폼이 예쁘다"가 아니라 **Guest -> StayOps -> Driver -> StayOps -> Guest 왕복이 실제로 끝난다**다.
3. WhatsApp 공급자 제약은 adapter 뒤에 둔다. 전송 수단이 막혀도 도메인은 멈추지 않는다.
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

### 대표기사 (WhatsApp)

- 대표기사 연락처 확보
- **대표기사가 WhatsApp을 쓸 수 있는지 확인** — 이 전제가 무너지면 즉시 SMS 어댑터로 전환
- 대표기사 opt-in 증거 확보
- 배차 요청 template 생성/승인
- 승인된 template을 대표기사 실단말로 발송
- URL 버튼 -> signed 배차 응답 Web이 실단말에서 정상 열리는지 확인
- 한 template에 quick-reply 버튼 + URL 버튼 조합 가능 여부 확인 (24시간 창 확보용)
- 같은 번호에서 Guest/Partner inbound 역할 라우팅 확인

### Backend

- managed backend 후보의 최소 spike
- 1차 권장 Supabase에서 Postgres/Auth/Edge Function/RLS가 요구에 맞는지 검증

## 완료조건

- WhatsApp의 실제 inbound/outbound 경로가 동작한다.
- 호스트 수동 WhatsApp 운영과 API 자동화를 함께 쓸 수 있는 연결 모드가 확인되거나 명시적인 fallback이 선택된다.
- 대표기사에게 배차 요청이 실제로 도달하고 응답 링크가 열린다.
- Guest와 대표기사의 inbound가 역할로 구분된다.
- 이후 개발이 특정 공급자 미확정 때문에 다시 설계되지 않는다.

## 실패 시 대비

대표기사가 WhatsApp을 쓸 수 없거나 template 승인이 지연되면 `PartnerNotificationAdapter`를 `sms` 또는 `manual`로 전환한다. 세 경로 모두 같은 signed URL을 보내므로 Phase 1~4 구현은 그대로 진행할 수 있다.

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
- `dispatch_partners`
- `driver_dispatches`
- `driver_assignments`
- `notifications/message_events`
- `wa_contacts`
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

`08_요금표.md` §2의 16개 조합을 방향까지 명시해 seed하고 server authoritative calculation을 만든다.

예약 시:

```text
fare rule lookup (origin · area · direction · pax)
-> 공급가 = 기본 + 옵션
-> 세금 = ROUND(공급가 × 0.1)
-> 총액 = 공급가 + 세금
-> fare_snapshot 저장
```

- 요금표에 없는 조합은 0원이 아니라 error다.
- 과거 요청은 이후 요금표 수정으로 변경되지 않는다.
- **계산 결과를 Guest 응답에 담지 않는다.** 요금은 Host와 대표기사에게만 쓰인다.

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

- 인천공항 / 김포공항 / 서울역 / 일산 / 용인 에버랜드
- Accommodation (이태원 5곳 · 홍대 1곳)
- terminal/meeting point — 공항일 때만
- **금액을 표시하지 않는다.** 손님은 행선지와 옵션만 고른다
- 요금표에 없는 조합(일산↔홍대, 에버랜드↔홍대)은 선택 불가

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
- 서버가 공급가·세금·총액을 계산해 snapshot 저장
- **Guest 화면과 응답 어디에도 금액이 없음**
- WhatsApp 연락처 E.164 정규화
- opt-in evidence 저장
- receipt notification 결과 추적

---

# Phase 3 — Driver dispatch round trip

## 목표

호스트가 차량번호를 다시 입력하지 않아도 기사 응답이 StayOps에 직접 들어오게 한다.

## DriverDispatch

Host가 지불을 확인하고 [배차 요청]을 누르면 dispatch가 만들어지고 대표기사에게 자동 발송된다.

```text
TransferRequest (payment_confirmed = true)
-> DriverDispatch
-> PartnerNotificationAdapter (whatsapp)
-> 발송 성공 시 awaiting_response
```

Host가 채팅방을 고르거나 메시지를 작성하지 않는다.

## 배차 요청 메시지

승인된 WhatsApp template. **요약은 메시지에, 상세는 링크 뒤에** 둔다.

메시지에 넣는 것:

- 일시
- pickup/dropoff
- 노선 (거점 · 터미널 · 숙소)
- 인원/수하물
- 카시트 필요 여부
- **요금 (세금포함 총액 + 항목 분해)**
- signed response 버튼

메시지에 넣지 않는 것:

- 손님 전화번호
- 손님 전체 이름
- 항공편·특이사항 등 상세 (링크 뒤 페이지에 표시)

## 배차 응답 Web

로그인 없는 token 기반 모바일 페이지. 상단에 운행 상세를 읽기 전용으로 보여주고, 입력은 세 가지뿐이다.

```text
배차 가능 / 불가
기사 전화번호      국가번호 포함 E.164 정규화
차량번호
차량 종류/색상 optional
```

ETA는 V1 필수가 아니다.

## 확정 transaction

수락 시:

```text
validate token + dispatch
-> ensure request not cancelled/already assigned
-> create DriverAssignment
-> mark dispatch accepted
-> mark transfer assigned
```

## 배차 불가 / 무응답

- 배차 불가 -> Host에게 알림. V1에서 자동 재발송하지 않는다
- timeout -> expired -> Host에게 알림
- 중복 제출과 동시 assignment는 DB constraint/transaction으로 차단

## 완료조건

- 대표기사 휴대폰에서 링크로 배차 가능/불가 회신 가능
- 기사 전화번호와 차량번호가 자동으로 request와 연결
- 전화번호가 E.164로 정규화되어 저장
- 중복/만료 token 거부
- payment_confirmed = false 상태에서 dispatch 생성 거부
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
- 차량번호
- 기사 전화번호 (국가번호 포함)
- 일시
- 노선
- meeting point

동시에 Host에게도 배차 완료 알림을 보낸다.

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
0. WhatsApp feasibility (Guest + 대표기사)
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
