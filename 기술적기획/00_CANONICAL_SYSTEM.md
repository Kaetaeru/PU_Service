# StayOps Canonical System

> 상태: 현재 제품/기술 기획의 **최우선 Source of Truth**
> 최종 정리: 2026-09-04 (WhatsApp 단일 채널 · 대표기사 배차 구조로 개편)

## 0. 문서 우선순위

기획 문서 사이에 충돌이 있으면 아래 순서로 해석한다.

1. `00_CANONICAL_SYSTEM.md` — 제품 동작과 불변조건
2. `V1_INTERNAL_MVP.md` — 판매 전 내부 V1의 포함/제외 범위
3. `06_IMPLEMENTATION_READINESS.md` — V1 구현 계약, 데이터/보안/테스트 세부 규칙
4. `07_WhatsApp_배차_왕복_설계.md` — 배차 왕복 상세 설계
5. `08_요금표.md` — 요금에 관한 단일 기준
6. `09_UI_테마.md` — 화면 디자인 기준
7. `05_DEVELOPMENT_PLAN.md` — 구현 순서와 단계별 완료조건
8. `REQUIREMENTS.md` — 전체 사업/운영 요구와 장기 아이디어
9. `MVP_SPEC.md`, `01~04` — 이전 초안과 세부 설계 근거

### 이 문서가 대체하는 이전 규칙

- **KakaoTalk 관련 전부.** Kakao Share, 알림톡, 채널 챗봇, Kakao adapter는 사용하지 않는다. `04_카카오_기사배차_왕복_설계.md`는 `07_WhatsApp_배차_왕복_설계.md`로 대체됐다.
- Host의 차량정보 재입력
- Guest Kakao 반환
- `waiting/assigned/notified` 단일 dispatch/message 상태
- Guest에게 금액을 표시하던 규칙 전부 (요금 박스, 실시간 견적) — Guest는 금액을 보지 않는다
- "V1은 세금을 계산하지 않는다"는 이전 결정 — Guest 노출이 사라졌으므로 세금 계산을 복원한다 (`08_요금표.md` §6.2)
- `payment_type = pending|host|guest|split` 및 `guest_due_krw`/`host_due_krw` 분담 모델 (§6이 대체)
- 개별 기사 마스터 관리와 복수 기사 fan-out

---

## 1. 제품 목표와 V1

StayOps의 목적은 숙소 Host가 Guest와 Partner 사이에서 반복적으로 정보를 읽고 복사하고 재작성해 전달하는 **수동 중계자 역할**을 하지 않게 만드는 것이다.

핵심 불변조건:

> 정상적인 업무에서 Host는 기사 메시지, 기사 연락처·차량번호, Guest 배차안내를 복사·재작성·재입력하지 않는다. Host의 조작은 **지불 확인 후 [배차 요청] 클릭 1회**뿐이다.

첫 제품은 공항/역/거점과 숙소 사이의 픽업·드랍오프다.

V1은 판매용 SaaS가 아니라 **우리가 실제 숙소 운영에 직접 쓰는 내부 운영 MVP**다. 정확한 범위는 `V1_INTERNAL_MVP.md`를 따른다.

---

## 2. Actor와 채널

| Actor | 채널 | 역할 |
|---|---|---|
| Guest | Guest Web + WhatsApp | 요청 입력, 운영 메시지 수신, Host와 자유 대화 |
| DispatchPartner (대표기사) | WhatsApp + signed 배차 응답 Web | 배차 요청 수신, 가능/불가 회신, 기사 연락처·차량번호 입력 |
| Host | StayOps 운영 화면 + WhatsApp Business App + 개인 WhatsApp | 지불 확인, 배차 요청, 예외 복구, Guest 수동 답장, 상황 파악 |
| StayOps | Managed Backend | 예약·배차·메시지 상태의 Source of Truth |

**외부 메시지는 전부 WhatsApp 하나로 나간다.** KakaoTalk은 사용하지 않는다.

WhatsApp은 상태 저장소가 아니다. 업무 상태는 StayOps에 저장한다.

### 대표기사 구조

기사 측 연락 상대는 개별 운행 기사가 아니라 **운송회사의 배차 담당자 한 명**이다. 대표기사가 회사 내부에서 실제 운행 기사를 배정하고, 그 결과를 StayOps에 입력한다.

StayOps는 개별 운행 기사를 저장된 주체로 관리하지 않는다. 기사 마스터·기사 계정·기사 로그인은 없다.

---

## 3. V1 배포 구조

직접 VPS를 관리하지 않는다.

```text
Guest Web ------------┐
배차 응답 Web ---------┼----> Frontend (Cloudflare Pages)
Host 운영 화면 --------┘             |
                                     v
                            Supabase Backend
                       Postgres / Auth / Edge Functions
                                     |
                                     v
                            WhatsApp Cloud API
                          (Guest · 대표기사 · Host)
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
pickup  = 거점 -> STAY
dropoff = STAY -> 거점
```

### Schedule

업무 timezone은 `Asia/Seoul`이다. 원래 입력한 local date/time을 보존한다.

Pickup:

- arrival date
- scheduled landing time
- 공항 픽업이면 flight number 필수

Drop-off:

- service date
- accommodation pickup time
- flight number는 선택

### Route / capacity

```text
passenger_count       1..7
large_luggage_count   >= 0
origin/destination    거점 코드 (아래)
stay_code             active Stay
terminal_code         conditional
```

거점 코드:

```text
ICN        인천공항        공항
GMP        김포공항        공항
SEOULSTN   서울역
ILSAN      일산
EVERLAND   용인 에버랜드
```

다섯 거점 모두 손님이 신청 폼에서 고를 수 있다.

terminal:

- ICN: `T1 | T2 | UNKNOWN`
- GMP: `DOMESTIC | INTERNATIONAL | UNKNOWN`
- 그 외: null

공항이 아닌 거점(`SEOULSTN` / `ILSAN` / `EVERLAND`)에서는 항공편·터미널·터미널경유 항목을 표시하지 않는다.

8명 이상은 가장 높은 요금을 자동 적용하지 않는다. V1 자동예약 범위 밖으로 거부하고 Host 문의로 전환한다.

거점·노선의 유효 조합은 `08_요금표.md` §2를 따른다. **요금 규칙이 없는 조합은 신청을 받지 않는다.** 현재 `일산 ↔ 홍대`, `에버랜드 ↔ 홍대`가 여기에 해당하며 폼에서 선택할 수 없게 막는다.

### Add-ons

```text
child_seat_count      0..4
child_seat_notes      required when child_seat_count > 0
terminal_transfer     boolean
special_request       optional
```

터미널 경유는 공항(`ICN`/`GMP`)에서만 허용한다.

### Location

실시간 GPS는 V1 기본 필수가 아니다. 필요성이 pilot에서 확인되면 request-scoped location flow를 추가한다.

---

## 6. Fare / 지불 불변조건

### Fare

요금 규칙은 `08_요금표.md`가 단일 기준이다.

```text
공급가 = base_fare_krw + option_fare_krw
세금   = ROUND(공급가 × 0.1)
총액   = 공급가 + 세금
currency = KRW
```

**요금은 Host와 대표기사에게만 쓰이는 내부 지표다.**

- **Guest는 금액을 보지 않는다.** 신청 폼, 접수 확인 메시지, 배차 안내 메시지 어디에도 금액이 없다. 계산 결과를 클라이언트로 내려보내지도 않는다.
- Host 운영 화면과 대표기사 배차 요청 메시지에는 **세금포함 총액**을 표시한다.
- server fare snapshot이 권위값이다.
- **요금표에 없는 노선 조합은 계산하지 않고 신청을 거부한다.** 0원으로 진행하지 않는다.
- 인원이 1~7을 벗어나면 상위 구간을 조용히 적용하지 않고 거부한다.
- 과거 snapshot은 rule 변경으로 재계산하지 않는다.
- 편도 요금은 방향에 무관하게 같다. **인천공항 노선만 방향별로 다르다.** 규칙은 방향까지 명시해 저장하고 코드에 암묵 규칙을 두지 않는다.

### 지불

**지불은 이 서비스 바깥에서 이뤄진다.** StayOps는 금액을 정산하지 않고 수금·입금을 관리하지 않는다.

V1이 기록하는 것은 하나뿐이다.

```text
payment_confirmed            boolean
payment_confirmed_by
payment_confirmed_at
```

불변조건:

> `payment_confirmed = false`이면 배차 요청을 발송하지 않는다.

이전 문서의 `payment_type`, `payment_instruction_status`, `guest_due_krw`, `host_due_krw`, 입금상태 3단계는 V1에서 사용하지 않는다. 정산은 Post-V1 범위이며, 그때를 위해 **fare snapshot은 V1부터 보존한다.**

---

## 7. 정상 왕복 플로우

```text
 1. Guest Form 제출
 2. server validation + authoritative fare
 3. TransferRequest 저장 (request_id, revision = 1)
 4. Guest WhatsApp 접수 확인 발송
 5. Host WhatsApp 신규 요청 알림 발송
 6. Host가 지불을 확인하고 payment_confirmed 표시
 7. Host가 [배차 요청] 클릭
 8. DriverDispatch 생성 (request_revision 스냅샷, 응답 토큰 발급)
 9. 대표기사에게 WhatsApp 템플릿 발송 (URL 버튼)
10. 발송 성공 -> DriverDispatch: pending -> awaiting_response
11. 대표기사가 링크에서 가능/불가 + 기사 전화번호 + 차량번호 입력
12. StayOps transaction이 DriverAssignment 확정
13. Guest WhatsApp에 배차 안내 자동 발송
14. Host WhatsApp에 배차 완료 알림 발송
15. Host가 운행 완료 처리
```

6~7번이 Host의 유일한 조작이다. 상세는 `07_WhatsApp_배차_왕복_설계.md`.

---

## 8. WhatsApp 운영 모델

### 8.1 단일 번호

Guest와 대표기사가 **같은 WABA 번호**와 대화한다. Host 알림은 같은 번호에서 Host 개인 번호로 발송한다.

V1 목표는 내부 WhatsApp Business 번호 하나에서 다음을 같이 만족하는 것이다.

- StayOps outbound operational message (Guest / 대표기사 / Host)
- inbound webhook 수신
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

번호를 Guest용/Partner용으로 분리하는 방안은 Post-V1에서 검토한다.

### 8.2 역할 라우팅

들어오는 메시지의 역할을 먼저 판정한다.

```text
inbound message (provider wa_id)
  -> E.164 정규화
  -> 역할 판정
       활성 DispatchPartner 번호와 일치       -> role = partner
       활성 TransferRequest 손님 번호와 일치  -> role = guest
       둘 다 일치                             -> role = partner (+ 충돌 플래그)
       어느 쪽도 아님                         -> role = unknown
  -> MessageEvent 저장 (role, request_id nullable)
```

Guest inbound free-text는 하나의 활성 request에 명확히 연결되는 경우에만 `request_id`를 붙인다. 0건이거나 복수면 자동 추정하지 않고 Host 확인 대상으로 둔다.

대표기사가 링크 대신 자유 텍스트로 답장하면 고정 문구로 링크를 1회 재안내한다. **응답 내용을 파싱하지 않는다.**

V1에서 AI 자유대화 자동답변은 하지 않는다.

### 8.3 24시간 창

상대가 메시지를 보내야 24시간 자유 텍스트 창이 열린다. URL 버튼 탭은 메시지가 아니므로 창이 열리지 않는다.

따라서 business-initiated 메시지는 승인된 템플릿이어야 한다. 창 상태를 `wa_contacts`로 추적하고 발송 직전 템플릿/자유 텍스트를 결정한다.

---

## 9. 배차 운영 모델

대표기사의 자유 텍스트를 파싱하지 않는다.

```text
DriverDispatch
-> WhatsApp 템플릿 (요약 + URL 버튼)
-> signed 배차 응답 Web
-> 가능 / 불가
-> 기사 전화번호 (국가번호 포함 E.164)
-> 차량번호
-> 차량 종류/색상 optional
```

응답 토큰은 bearer credential이며 짧은 TTL, 폐기 가능, hashed-at-rest로 관리한다.

`ETA`는 V1 필수 항목이 아니다. 픽업 시각은 항공편 착륙 시각 기준으로 이미 확정돼 있다.

**배차 확정 후 차량·기사가 바뀌는 경우는 V1에서 시스템으로 처리하지 않는다.** 이례적 상황이므로 Host가 수동 대응한다.

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

`pending -> awaiting_response`의 트리거는 **Notification 발송 성공(sent)**이다. `notification_channel = manual`인 경우에만 Host의 전달 완료 확인으로 전환한다.

전송 성공/실패 자체는 Notification이 담당한다.

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
- 지불 확인 취소

assignment 이후 material edit은 기존 assignment를 `superseded`로 만들고 request를 `needs_reconfirmation`으로 보낸다. stale dispatch/token으로 새 assignment를 만들 수 없다.

---

## 12. Canonical domain model

### Host

```text
host_id
brand_name
settings
whatsapp_connection_mode
host_notify_phone_e164        운영 알림 수신용 개인 번호
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
origin_code
destination_code
terminal_code
child_seat_count
child_seat_notes
terminal_transfer
special_request

fare_snapshot
  base_fare_krw
  option_fare_krw
  supply_amount_krw
  tax_amount_krw
  total_amount_krw
  fare_rule_version
  calculated_at

payment_confirmed
payment_confirmed_by
payment_confirmed_at

transfer_status
created_at
updated_at
```

### DispatchPartner

```text
partner_id
host_id
name
contact_name
whatsapp_phone_e164
whatsapp_opt_in
whatsapp_opt_in_at
whatsapp_opt_in_policy_version
preferred_language
notification_channel          whatsapp | sms | manual
is_active
```

### DriverDispatch

```text
dispatch_id
request_id
request_revision
partner_id
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
partner_id
status
driver_phone_e164
vehicle_plate
vehicle_model
vehicle_color
assigned_at
completed_at
```

개별 기사는 마스터가 아니라 **이 레코드의 값**으로만 존재한다.

### Notification / MessageEvent

```text
message_id
request_id nullable
request_revision nullable
host_id
contact_key / provider wa_id
audience                      guest | partner | host
role                          guest | partner | host | unknown
channel                       whatsapp | sms | manual
send_mode                     template | freeform
template_name nullable
template_language nullable
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

### wa_contacts

```text
phone_e164 / wa_id
host_id
role                          guest | partner | unknown
linked_partner_id nullable
last_inbound_at
window_expires_at
```

### FareRule

`08_요금표.md` §6의 형태를 따른다. request 생성 시 snapshot을 보존한다.

---

## 13. Concurrency / idempotency

### Guest submit

```text
unique(host_id, idempotency_key)
```

- same key + same payload -> original result
- same key + different payload -> conflict

### 배차 응답

한 transaction에서 token/expiry/revision/request status/active assignment를 잠그고 검사한 뒤 assignment + status + Guest Notification outbox + Host Notification outbox를 함께 만든다.

응답 endpoint는 멱등이다. 같은 응답을 반복하면 최초 결과를 반환한다.

대표기사가 한 명이므로 복수 수락 경쟁은 발생하지 않지만, 중복 제출과 Host edit 동시 발생을 막기 위해 트랜잭션 보호는 유지한다.

### Webhook

- provider event는 append-only 저장
- `provider_event_id` unique
- duplicate no-op
- out-of-order callback이 delivered를 sent로 되돌리지 못함

---

## 14. 보안 경계

- Guest browser는 DB table에 직접 write하지 않는다. public Edge Function을 통한다.
- 배차 응답 Web도 마찬가지로 Edge Function을 통한다.
- Host 운영 화면은 Supabase Auth + RLS를 사용한다.
- privileged function은 client가 보낸 host_id를 신뢰하지 않고 authenticated Host ownership을 확인한다.
- secret/service key/provider token은 browser bundle에 넣지 않는다.
- public request id만으로 PII를 조회할 수 없다.
- Guest/배차응답 endpoint는 validation/body limit/rate limit을 적용한다.
- webhook signature/verification을 provider 지원 방식에 맞춰 검증한다.

배차 응답 링크 형태:

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

### WhatsApp 발송 실패

```text
Notification failed
-> Host 확인 필요 목록
-> retry
-> 필요 시 Host manual 전달
```

### 대표기사 배차 불가 / 무응답

```text
Dispatch rejected/expired
-> Host에게 알림
-> Host가 다른 경로 수배 (V1에서 자동화하지 않음)
```

### Schedule/material change

```text
revision +1
-> old pending dispatch invalid
-> active assignment superseded
-> needs_reconfirmation
```

### Cancel

cancelled request는 새 assignment를 받을 수 없다. 대표기사에게 취소 통보를 발송하고, queued stale notification은 발송하지 않는다.

### 전송 수단 불가

`PartnerNotificationAdapter`를 `sms` 또는 `manual`로 전환한다. 세 경로 모두 같은 signed URL을 보내므로 도메인은 바뀌지 않는다. fallback은 운영자 클릭 1회를 요구할 수 있으나 **canonical 데이터 재입력을 요구해서는 안 된다.**

---

## 17. V1에서 하지 않는 것

- KakaoTalk 연동 전부
- AI Guest 자유대화 자동응답
- StayOps 자체 full WhatsApp inbox
- 자동 flight tracking
- 복잡한 live GPS tracking
- 개별 기사 마스터/계정, 다중 기사 fan-out
- 배차 확정 후 차량·기사 변경 흐름
- 정산 자동화/Excel 대체, 입금상태 관리
- Guest에게 금액 노출
- self-service multi-host onboarding/billing
- iCal/청소/세탁
- 당일 투어 상품 (`08_요금표.md` §6.3)

---

## 18. Pilot 전 외부 게이트

실손님에게 사용하기 전 반드시 확정한다.

1. 실제 WhatsApp Business 번호 inbound/outbound
2. Host manual reply와 Coexistence 또는 동등 경로
3. Guest 템플릿 승인 (접수 확인 / 배차 완료 / 일정 변경)
4. 대표기사 템플릿 승인 및 실단말 수신
5. 대표기사 단말에서 URL 버튼 -> signed 응답 Web 정상 오픈
6. 대표기사 연락처 확보와 WhatsApp 사용 가능 여부 확인
7. 대표기사 opt-in 증거 확보
8. opt-in/privacy notice
9. production backup/secret
10. `08_요금표.md` 미확정 항목 확정
11. Guest/Partner inbound 역할 라우팅 검증
12. WSR는 주소 확정 전 inactive 유지

---

## 19. 최종 구현 기준

구체적인 기술 stack, transaction boundary, repository layout, 테스트 목록은 `06_IMPLEMENTATION_READINESS.md`를 따른다.

`06_IMPLEMENTATION_READINESS.md`의 Engineering-ready 조건은 코드 작성 가능 여부를 의미하고, Pilot-ready 조건은 실제 Guest 운영 가능 여부를 의미한다.
