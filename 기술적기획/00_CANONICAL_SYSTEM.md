# StayOps Canonical System

> 상태: 현재 제품/기술 기획의 **최우선 Source of Truth**
> 최종 정리: 2026-08-25

## 0. 문서 우선순위

기획 문서 사이에 충돌이 있으면 아래 순서로 해석한다.

1. `00_CANONICAL_SYSTEM.md` — 제품 동작과 불변조건
2. `V1_INTERNAL_MVP.md` — 판매 전 내부 V1의 포함/제외 범위
3. `06_IMPLEMENTATION_READINESS.md` — V1 구현 계약, 데이터/보안/테스트 세부 규칙
4. `05_DEVELOPMENT_PLAN.md` — 구현 순서와 단계별 완료조건
5. `REQUIREMENTS.md` — 전체 사업/운영 요구와 장기 아이디어
6. `MVP_SPEC.md`, `01~04` — 이전 초안과 세부 설계 근거

기존 문서의 Guest Kakao 반환, Host 차량정보 재입력, 단일 dispatch/message 상태, 암묵적 VAT 10% 추가 규칙은 이 문서와 `06_IMPLEMENTATION_READINESS.md`에 의해 대체된다.

---

## 1. 제품 목표와 V1

StayOps의 목적은 숙소 Host가 Guest와 Partner 사이에서 반복적으로 정보를 읽고 복사하고 재작성해 전달하는 **수동 중계자 역할**을 하지 않게 만드는 것이다.

핵심 불변조건:

> 정상적인 업무에서는 Host가 Driver 메시지, Driver ETA/차량정보, Guest 배차안내를 복사·재작성·재입력하지 않는다. Host는 승인, 예외 판단, 복구, 수동 대화에 집중한다.

첫 제품은 공항/서울역 픽업·드랍오프다.

V1은 판매용 SaaS가 아니라 **우리가 실제 숙소 운영에 직접 쓰는 내부 운영 MVP**다. 정확한 범위는 `V1_INTERNAL_MVP.md`를 따른다.

---

## 2. Actor와 채널

| Actor | 기본 채널 | 역할 |
|---|---|---|
| Guest | Guest Web + WhatsApp | 요청 입력, 운영 메시지 수신, Host와 자유 대화 |
| Driver | KakaoTalk + signed Driver Web | 배차 요청 수신, 수락/거절, ETA·차량정보 입력 |
| Host | StayOps Admin + WhatsApp Business App | 기사 선택/승인, 예외 복구, Guest 수동 답장 |
| StayOps | Managed Backend | 예약·배차·메시지·payment 상태의 Source of Truth |

WhatsApp과 KakaoTalk은 상태 저장소가 아니다. 업무 상태는 StayOps에 저장한다.

---

## 3. V1 배포 구조

직접 VPS를 관리하지 않는다.

```text
Guest Web -----------┐
Driver Response Web -┼----> Frontend
Host Admin ----------┘        |
                              v
                       Supabase Backend
                  Postgres / Auth / Edge Functions
                         |             |
                         v             v
                    WhatsApp      Kakao Share/Adapter
```

V1 구현 stack의 세부 결정은 `06_IMPLEMENTATION_READINESS.md`를 따른다.

---

## 4. Guest Contact 계약

Guest의 기본 연락 좌표는 **WhatsApp 번호**다.

```text
guest_name
whatsapp_phone_e164
preferred_language
whatsapp_opt_in
whatsapp_opt_in_at
whatsapp_opt_in_policy_version
```

규칙:

- UI는 국가코드 + 현지번호 형태를 허용하되 저장은 E.164로 정규화한다.
- 전화번호는 예약 PK가 아니다.
- `request_id`가 예약 identity이고 WhatsApp 번호는 contact/correlation key다.
- business-initiated 운영 메시지를 위한 opt-in 증거를 보존한다.
- 같은 번호에 여러 request가 존재할 수 있다.

---

## 5. Guest Transfer 입력 계약

### Guest

필수:

- 대표 Guest 영문명
- WhatsApp 번호
- preferred language
- WhatsApp 운영 메시지 opt-in

### Direction

```text
pickup  = AIRPORT/STATION -> STAY
dropoff = STAY -> AIRPORT/STATION
```

### Schedule

업무 timezone은 `Asia/Seoul`이다. 원래 입력 local date/time을 보존한다.

Pickup:

- arrival date
- scheduled landing time
- airport pickup이면 flight number 필수

Drop-off:

- service date
- accommodation pickup time
- flight number는 선택

### Route / capacity

```text
passenger_count       1..7
large_luggage_count   >= 0
port_code             ICN | GMP | SEOULSTN
stay_code             active Stay
terminal_code         conditional
```

terminal:

- ICN: `T1 | T2 | UNKNOWN`
- GMP: `DOMESTIC | INTERNATIONAL | UNKNOWN`
- SEOULSTN: null

8명 이상은 가장 높은 요금을 자동 적용하지 않는다. V1 자동예약 범위 밖으로 처리한다.

### Add-ons

```text
child_seat_count      0..4
child_seat_notes      required when child_seat_count > 0
terminal_transfer     boolean
special_request       optional
```

서울역에서는 terminal transfer를 허용하지 않는다.

### Location

실시간 GPS는 V1 기본 필수가 아니다. 필요성이 pilot에서 확인되면 request-scoped location flow를 추가한다.

---

## 6. Fare / Payment 불변조건

### Fare

V1의 Guest-facing authoritative fare는 최초 HTML에서 검증된 정책을 보존한다.

```text
base_fare_krw
+ option_fare_krw
= total_fare_krw
currency = KRW
```

- browser price는 estimate다.
- server fare snapshot이 권위값이다.
- fare rule이 없으면 0원으로 진행하지 않고 error다.
- 과거 snapshot은 rule 변경으로 재계산하지 않는다.
- **근거가 확정되지 않은 VAT 10%를 V1 Guest 금액에 자동 추가하지 않는다.**

세무용 tax breakdown은 Post-V1 정산 정책에서 별도로 정의한다.

### Payment instruction

```text
payment_type = pending | host | guest | split
payment_instruction_status = pending | confirmed
guest_due_krw
host_due_krw
```

불변조건:

```text
guest_due_krw + host_due_krw = total_fare_krw
```

payment instruction이 confirmed가 아니면 dispatch를 시작하지 않는다.

---

## 7. 정상 왕복 플로우

```text
1. Guest Form 제출
2. server validation + authoritative fare
3. TransferRequest 저장 + request_id/revision 생성
4. Guest WhatsApp 접수 확인
5. Host가 dispatch 준비 확인 / Driver 선택
6. DriverDispatch 생성
7. Host가 Kakao Share로 기사 채팅방에 전송
   또는 검증된 automatic adapter 사용
8. Driver가 signed Driver Web에서 accept/reject + ETA/vehicle 입력
9. StayOps transaction이 DriverAssignment 확정
10. Guest WhatsApp에 assignment 자동 반환
11. Host가 운행 완료 처리
```

Kakao 자동 outbound는 V1 필수가 아니다. one-click Kakao Share가 V1 baseline fallback이다.

---

## 8. WhatsApp 운영 모델

V1 목표는 내부 WhatsApp Business 번호에서 다음을 같이 만족하는 것이다.

- StayOps outbound operational message
- Guest inbound webhook
- Host의 WhatsApp Business App 수동 대화
- 가능한 경우 Host sent-message echo도 StayOps 이력에 반영

선호 연결 모드:

```text
coexistence
```

향후 판매 시 연결 모드는 명시적으로 관리한다.

```text
coexistence | api_only | manual_only
```

V1에서 AI 자유대화 자동답변은 하지 않는다.

Guest inbound free-text는 먼저 MessageEvent로 저장하고, 하나의 활성 request에 명확히 연결되는 경우에만 request_id를 붙인다. 여러 request가 후보면 자동 추정하지 않고 Host 확인 대상으로 둔다.

---

## 9. Driver / Kakao 운영 모델

기사 자유 텍스트를 파싱하지 않는다.

```text
DriverDispatch
-> Kakao Share/notification
-> signed Driver Response Web
-> accept/reject
-> ETA
-> vehicle plate
-> vehicle model/color optional
```

Driver 기본 name/phone은 Driver master에서 prefill할 수 있다.

Driver response token은 bearer credential이며 짧은 TTL, one-time/revocable, hashed-at-rest로 관리한다.

---

## 10. 상태 모델

메시지 전송과 업무 상태를 같은 enum으로 합치지 않는다.

### 10.1 TransferRequest

```text
requested
-> awaiting_driver
-> assigned
-> completed

assigned
-> needs_reconfirmation
-> awaiting_driver
-> assigned

terminal:
cancelled
failed
```

`guest_notified`는 Transfer 상태가 아니다.

### 10.2 DriverDispatch

```text
pending
-> awaiting_response
-> accepted | rejected | expired | cancelled
```

전송 성공/실패는 Notification이 담당한다.

### 10.3 DriverAssignment

```text
active
-> completed
-> superseded | cancelled
```

하나의 TransferRequest에는 active assignment가 최대 하나다.

### 10.4 Notification

```text
queued
-> sent
-> delivered

queued/sent -> failed
failed -> queued  # explicit retry
stale event -> skipped_stale
```

---

## 11. Revision / stale-data 불변조건

TransferRequest는 `revision`을 가진다.

- 최초 생성: 1
- dispatch 의미에 영향을 주는 material edit: +1
- DriverDispatch는 `request_revision`을 snapshot
- accept 시 현재 revision과 같아야 함
- Notification도 `request_revision`을 가진다
- 발송 직전 current status/revision을 재검증한다

material edit:

- date/time/direction
- route/stay/terminal
- flight
- passenger/luggage
- child seat/terminal transfer
- payment instruction

assignment 이후 material edit은 기존 assignment를 `superseded`로 만들고 request를 `needs_reconfirmation`으로 보낸다. stale dispatch/token으로 새 assignment를 만들 수 없다.

---

## 12. Canonical domain model

### Host

```text
host_id
brand_name
settings
whatsapp_connection_mode
```

### Stay

```text
stay_id
host_id
stay_code
name
address
area_code
is_active
```

WSR은 정확한 주소가 확정되기 전 V1에서 inactive다.

### TransferRequest

```text
request_id
host_id
stay_id
revision
idempotency_key

guest_name
whatsapp_phone_e164
preferred_language
whatsapp_opt_in_at
whatsapp_opt_in_policy_version

direction
service_date
service_time
time_semantics
flight_no
passenger_count
large_luggage_count
port_code
terminal_code
child_seat_count
child_seat_notes
terminal_transfer
special_request

fare_snapshot
payment_type
payment_instruction_status
guest_due_krw
host_due_krw
transfer_status
created_at
updated_at
```

### Driver

```text
driver_id
host_id
name
phone
notification_route
is_active
```

### DriverDispatch

```text
dispatch_id
request_id
request_revision
driver_id
status
expires_at
response_token_hash
responded_at
```

### DriverAssignment

```text
assignment_id
request_id
dispatch_id
driver_id
status
eta
vehicle_plate
vehicle_model
vehicle_color
assigned_at
completed_at
```

### Notification / MessageEvent

```text
message_id
request_id nullable
request_revision nullable
host_id
contact_key / provider wa_id
audience
channel
provider_message_id/provider_event_id
direction
message_type
status
dedupe_key
attempt_count
next_attempt_at
created_at
sent_at
delivered_at
last_error
```

### FareRule

route/area/direction/passenger tier와 option price의 server-owned policy. request 생성 시 snapshot을 보존한다.

---

## 13. Concurrency / idempotency

### Guest submit

```text
unique(host_id, idempotency_key)
```

- same key + same payload -> original result
- same key + different payload -> conflict

### Driver accept

한 transaction에서 token/expiry/revision/request status/active assignment를 잠그고 검사한 뒤 assignment + status + Guest Notification outbox를 함께 만든다.

두 기사가 동시에 수락해도 active assignment는 하나만 성공한다.

### Webhook

- provider event는 append-only 저장
- `provider_event_id` unique
- duplicate no-op
- out-of-order callback이 delivered를 sent로 되돌리지 못함

---

## 14. 보안 경계

- Guest browser는 DB table에 직접 write하지 않는다. public Edge Function을 통한다.
- Host Admin은 Supabase Auth + RLS를 사용한다.
- privileged function은 client가 보낸 host_id를 신뢰하지 않고 authenticated Host ownership을 확인한다.
- secret/service key/provider token은 browser bundle에 넣지 않는다.
- public request id만으로 PII를 조회할 수 없다.
- Guest endpoint는 validation/body limit/rate limit을 적용한다.
- webhook signature/verification을 provider 지원 방식에 맞춰 검증한다.

Driver link 권장 형태:

```text
https://<frontend>/driver#t=<opaque-random-token>
```

URL fragment를 이용해 token이 일반 HTTP request/referrer 로그에 노출되는 범위를 줄인다.

---

## 15. Notification outbox

외부 메시지 전송을 DB transaction 성공 조건으로 묶지 않는다.

```text
business transaction
-> Notification queued
-> commit
-> worker send
-> provider result/webhook
-> Notification state update
```

Notification은 durable outbox 역할을 하며 retry/dedupe/stale-skip을 지원한다.

---

## 16. Failure / fallback

### WhatsApp fail

```text
Notification failed
-> Needs Attention
-> retry
-> 필요 시 Host manual WhatsApp
```

### Driver reject / timeout

```text
Dispatch rejected/expired
-> Host selects next Driver
-> new Dispatch
```

### Schedule/material change

```text
revision +1
-> old pending dispatch invalid
-> active assignment superseded
-> needs_reconfirmation
```

### Cancel

cancelled request는 새 assignment를 받을 수 없다. queued stale notification은 발송하지 않는다.

### Kakao automatic outbound unavailable

V1은 Kakao Share one-click을 사용한다. Host가 메시지를 직접 재작성하지 않는다.

---

## 17. V1에서 하지 않는 것

- AI Guest 자유대화 자동응답
- StayOps 자체 full WhatsApp inbox
- 자동 flight tracking
- 복잡한 live GPS tracking
- 다중 기사 fan-out marketplace
- 정산 자동화/Excel 대체
- self-service multi-host onboarding/billing
- iCal/청소/세탁

---

## 18. Pilot 전 외부 게이트

실손님에게 사용하기 전 반드시 확정한다.

1. 실제 WhatsApp Business 번호 inbound/outbound
2. Host manual reply와 Coexistence 또는 동등 경로
3. 필요한 WhatsApp template
4. opt-in/privacy notice
5. production backup/secret
6. 실제 fare table 청구금액
7. Driver seed/연락처
8. Kakao Share real-device E2E
9. WSR는 주소 확정 전 inactive 유지

---

## 19. 최종 구현 기준

구체적인 기술 stack, transaction boundary, repository layout, 테스트 목록은 `06_IMPLEMENTATION_READINESS.md`를 따른다.

`06_IMPLEMENTATION_READINESS.md`의 Engineering-ready 조건은 코드 작성 가능 여부를 의미하고, Pilot-ready 조건은 실제 Guest 운영 가능 여부를 의미한다.
