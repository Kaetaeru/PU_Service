# StayOps V1 — Implementation Readiness

> 목적: `plan-critic` 최종 검수 결과를 반영해 V1을 **코드 작성 가능한 상태**로 고정한다.
> 적용 범위: 판매 전 내부 운영 V1
> 상위 제품 규칙: `00_CANONICAL_SYSTEM.md`
> V1 범위: `V1_INTERNAL_MVP.md`
> 구현 순서: `05_DEVELOPMENT_PLAN.md`

---

## 0. 최종 판정

**Engineering-ready:** 예. 아래 계약을 기준으로 backend/frontend 구현을 시작할 수 있다.

**Pilot-ready:** 아직 아님. 실제 Guest를 받기 전 아래 외부 게이트가 반드시 완료되어야 한다.

1. 내부에서 실제 사용할 WhatsApp Business 번호의 API inbound/outbound 확인
2. 같은 번호에서 Host 수동 대화를 유지할 수 있는 Coexistence 또는 동등 운영 경로 확인
3. 실제 business-initiated 운영 template 발송 확인
4. Guest opt-in 문구/개인정보 안내 확정

Kakao 자동발송은 V1 release blocker가 아니다. V1의 baseline은 StayOps가 dispatch를 만들고 **Kakao Share one-click**으로 Host가 기사 채팅방을 선택해 보내는 방식이다. 기사 응답은 항상 signed Driver Response Web으로 StayOps에 직접 돌아온다.

---

## 1. 최종 Plan-Critic 발견 사항과 수정

### Major A — Transfer 상태와 메시지 상태가 다시 중복됨

기존 canonical에는 `TransferRequest.guest_notified`가 있었지만 Notification 상태를 별도로 관리한다는 원칙과 중복된다.

**수정:** TransferRequest에서 `guest_notified`를 제거한다. Guest 통지 여부는 Notification으로만 판단한다.

### Major B — 기존 HTML과 새 MVP 사이의 VAT 10% 규칙이 충돌함

최초 HTML은 route fare + option을 최종 표시 금액으로 사용한다. 이후 `MVP_SPEC.md`에서 근거가 확정되지 않은 상태로 `tax = supply * 10%`를 추가했다. 그대로 구현하면 Guest가 보던 금액이 10% 상승할 수 있다.

**수정:** V1의 Guest-facing authoritative fare는 **현재 검증된 fare table + option 합계**로 고정한다.

```text
base_fare_krw
+ option_fare_krw
= total_fare_krw
```

V1에서는 세금을 암묵적으로 10% 추가하지 않는다. 세무/정산용 tax breakdown은 실제 회계 정책이 확정된 뒤 Post-V1에서 별도 필드로 추가할 수 있으나 `total_fare_krw`를 소급 변경하지 않는다.

### Major C — 일정 변경 뒤 오래된 기사 링크가 수락될 수 있음

현재 request가 수정되어도 이미 발급한 dispatch token이 어떤 버전의 예약을 기준으로 하는지 알 수 없었다.

**수정:** 모든 TransferRequest에 `revision`을 둔다.

- 최초 생성: `revision = 1`
- dispatch 의미에 영향을 주는 수정 시 `revision += 1`
- DriverDispatch는 `request_revision`을 저장
- Driver accept 시 `dispatch.request_revision == transfer.revision`이어야 함
- 불일치하면 `stale_dispatch`로 거부

material edit:

- service date/time
- direction
- route/stay/terminal
- flight number
- passenger/luggage
- child-seat/terminal-transfer option
- payment instruction

### Major D — Assignment 후 일정 변경의 상태가 없음

**수정:** TransferRequest에 `needs_reconfirmation`을 둔다.

assignment 이후 material edit이 발생하면 기존 assignment를 조용히 유지하지 않는다.

```text
assigned
-> material edit
-> needs_reconfirmation
-> new/reconfirmed dispatch
-> assigned
```

기존 DriverAssignment는 삭제하지 않고 `superseded`로 보존한다.

### Major E — payment의 `guest_paid`가 지불 예정액인지 실제 지불액인지 모호함

**수정:** V1 dispatch용 payment instruction은 아래로 명확히 분리한다.

```text
payment_type = pending | host | guest | split
guest_due_krw
host_due_krw
```

불변조건:

```text
guest_due_krw + host_due_krw = total_fare_krw
```

`payment_type = pending`이면 dispatch를 시작할 수 없다.

실제 입금/수금 완료 여부는 Post-V1 settlement 영역이다.

### Major F — 공항 터미널 정보가 기사 운행에 충분히 구조화되지 않음

**수정:** `terminal_code`를 conditional field로 둔다.

- ICN: `T1 | T2 | UNKNOWN`
- GMP: `DOMESTIC | INTERNATIONAL | UNKNOWN`
- SEOULSTN: `null`

`UNKNOWN`은 요청 생성은 허용하지만 Host의 Needs Attention에 표시한다.

### Major G — 카시트 수량만으로 실제 준비 정보가 부족함

**수정:** `child_seat_count > 0`이면 `child_seat_notes`를 필수로 한다. 최소한 아동 연령 정보를 입력하도록 Guest UI에서 안내한다.

### Major H — WhatsApp inbound를 무조건 하나의 예약에 연결할 수 없음

한 Guest 번호에 동시에 여러 transfer가 있을 수 있다.

**수정:** `MessageEvent.request_id`는 nullable이다.

- provider `wa_id` / normalized phone으로 Guest contact를 식별
- 활성 transfer가 정확히 하나면 연결 가능
- 0개 또는 여러 개면 임의 연결하지 않고 `needs_host_reply / needs_association`으로 둔다

### Major I — webhook 중복/순서 역전이 현재 상태를 되돌릴 수 있음

**수정:** provider webhook은 append-only event로 먼저 저장한다.

- `provider_event_id` unique
- 동일 event 재수신은 no-op
- notification current state는 허용된 단방향 transition만 적용
- 오래된 `sent` 이벤트가 이미 `delivered`인 상태를 되돌릴 수 없음

### Major J — 취소/수정 뒤 queue에 남아 있던 메시지가 발송될 수 있음

**수정:** Notification에도 `request_revision`과 logical `dedupe_key`를 저장한다.

발송 직전 worker가 현재 request status/revision을 재검증한다. stale/cancelled event는 보내지 않고 `skipped_stale`로 마감한다.

### Major K — WSR의 실제 주소가 비어 있음

**수정:** WSR은 정확한 주소가 입력되기 전까지 V1 seed에서 `is_active = false`로 둔다. 주소 없이 실제 dispatch를 허용하지 않는다.

---

## 2. V1 최종 상태 모델

### 2.1 TransferRequest

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

`Guest notified`는 Transfer 상태가 아니다.

### 2.2 DriverDispatch

운영상태만 가진다. 전송 성공 여부는 Notification에 둔다.

```text
pending
-> awaiting_response
-> accepted | rejected | expired | cancelled
```

Kakao Share fallback에서는 Host가 실제 share action을 실행한 뒤 `awaiting_response`로 전환한다.

### 2.3 DriverAssignment

```text
active
-> completed
-> superseded | cancelled
```

하나의 TransferRequest에는 `active` assignment가 최대 하나다.

### 2.4 Notification

```text
queued
-> sent
-> delivered

queued/sent
-> failed
failed -> queued   # explicit retry

stale/cancelled event:
-> skipped_stale
```

---

## 3. Dispatch-ready 불변조건

TransferRequest가 `awaiting_driver`로 전환되려면 모두 만족해야 한다.

```text
required guest contact valid
required schedule valid
active stay/address valid
route valid
passenger_count 1..7
fare snapshot exists
payment_type != pending
payment amounts reconcile
child seat info complete when requested
request not cancelled
```

공항 terminal이 `UNKNOWN`이면 dispatch는 Host가 경고를 확인하고 명시적으로 진행할 수 있다.

8명 이상은 자동 상위 요금 구간을 적용하지 않는다. V1 자동예약 범위 밖으로 거부하고 Host 문의로 전환한다.

---

## 4. V1 Guest 입력 최종 계약

### Guest

```text
guest_name                 required, max 100
whatsapp_phone_e164        required
preferred_language         required, BCP-47-like short code
whatsapp_opt_in            required true
whatsapp_opt_in_at         server timestamp
whatsapp_opt_in_policy_version
```

UI에서는 국가코드 + 현지번호를 받을 수 있으나 저장 전 국제전화번호 파서를 사용해 E.164로 정규화한다.

### Transfer

```text
direction                  pickup | dropoff
service_date               required
service_time               required
flight_no                  required for airport pickup; optional otherwise
passenger_count            required, 1..7
large_luggage_count        required, >= 0
port_code                  ICN | GMP | SEOULSTN
terminal_code              conditional
stay_code                  required active stay
child_seat_count           0..4
child_seat_notes           required when child_seat_count > 0
terminal_transfer          boolean, not allowed for SEOULSTN
special_request            optional, max 1000
```

시간의 업무 기준 timezone은 `Asia/Seoul`이다. 원래 입력한 local date/time을 보존한다.

### Language behavior

V1 자동 template은 지원된 언어가 있으면 사용하고, 없으면 English로 fallback한다. `preferred_language`를 받았다는 이유로 모든 언어의 자동 번역을 V1에 약속하지 않는다.

---

## 5. V1 Fare / Payment 최종 계약

### Fare snapshot

```text
currency = KRW
base_fare_krw
option_fare_krw
total_fare_krw
fare_rule_version / source
calculated_at
```

- client estimate는 권위값이 아니다.
- server가 fare rule을 조회한다.
- rule이 없으면 0원으로 진행하지 않고 validation/config error.
- 과거 request의 snapshot은 fare rule 변경으로 재계산하지 않는다.
- V1에서 근거 없이 VAT 10%를 추가하지 않는다.

### Payment instruction

```text
payment_type
payment_instruction_status = pending | confirmed
guest_due_krw
host_due_krw
confirmed_by
confirmed_at
```

Driver message는 confirmed payment instruction만 사용한다.

---

## 6. Stale-data / concurrency 규칙

### Guest duplicate submit

Guest form이 `Idempotency-Key`를 생성하고 안전한 retry에서 같은 key를 재사용한다.

DB unique:

```text
(host_id, idempotency_key)
```

- same key + same payload -> original result
- same key + different payload -> conflict

### Driver double response

accept/reject endpoint는 idempotent하다.

- 동일 response 반복 -> 기존 결과 반환
- 이미 다른 assignment 확정 -> conflict
- expired/cancelled/stale revision -> reject

### Host edit vs Driver accept

assignment 확정 transaction에서 request revision/status를 다시 확인한다. Host edit과 Driver accept가 동시에 발생해도 하나의 일관된 결과만 commit된다.

---

## 7. 보안 경계

### Public Guest

Guest browser가 Supabase table에 직접 INSERT하지 않는다.

```text
Guest Web
-> public Edge Function
-> server validation/rate limit
-> privileged DB transaction
```

### Host Admin

- Supabase Auth
- V1은 내부 Host account만 필요
- normal reads/writes는 user JWT + RLS
- admin function이 secret/service role을 쓸 때는 caller의 Host ownership을 function에서 다시 검증

### Driver token

Driver response token은 bearer credential이다.

권장 V1:

- 최소 128-bit cryptographically random opaque token
- DB에는 hash만 저장
- expires_at
- used_at/revoked_at
- dispatch/request revision binding

가능하면 링크는 fragment로 전달한다.

```text
https://<frontend>/driver#t=<opaque-token>
```

fragment는 HTTP request/referrer에 포함되지 않는다. Driver Web이 token을 읽어 response Edge Function에 POST하고, 페이지 초기화 후 주소 표시에서 token을 제거한다.

### Webhook

- provider가 제공하는 signature/verification을 검증
- raw secret은 Edge Function secret에 저장
- browser bundle에 provider secret/service key 금지

---

## 8. Notification outbox / retry

외부 WhatsApp 호출을 DB transaction 안에서 성공해야만 하는 구조로 만들지 않는다.

1. 업무 transaction이 상태 변경 + Notification(outbox) 생성
2. commit
3. worker가 queued Notification을 발송
4. provider result 저장
5. webhook delivery event 반영

V1 implementation:

- `notifications` table을 durable outbox로 사용
- 즉시 발송 시도 가능
- 실패/미처리 건은 scheduled worker가 재시도
- `attempt_count`, `next_attempt_at`, `last_error`
- logical event `dedupe_key` unique

Supabase를 선택하면 Cron/Edge Function 조합으로 worker를 실행할 수 있다.

---

## 9. V1 기술 스택 결정

별도 VPS를 운영하지 않는다.

### Frontend

```text
Vite
React
TypeScript
existing pickup UI/CSS 재사용
Cloudflare Pages 배포
```

V1에서 UI framework/component library는 추가하지 않는다.

### Backend

```text
Supabase Postgres
Supabase Auth
Supabase Edge Functions (TypeScript/Deno)
Postgres migrations / RLS
Supabase Cron for retry worker
```

### 최소 frontend dependencies

```text
react
react-dom
@supabase/supabase-js
libphonenumber-js
```

테스트는 `Vitest`를 기본으로 한다. V1 release 직전 핵심 실제 브라우저 flow는 Playwright 또는 수동 real-device E2E로 검증한다.

---

## 10. 권장 repository layout

구현 시작 시 root에 아래 구조를 만든다.

```text
src/
  app/
    request/
    driver/
    admin/
  components/
  lib/
  styles/

supabase/
  migrations/
  seed.sql
  functions/
    _shared/
    create-transfer-request/
    create-driver-dispatch/
    respond-driver-dispatch/
    whatsapp-webhook/
    process-notifications/

public/

기술적기획/
```

`pickup_2.html`은 UI reference로 남기고, 구현 후 즉시 삭제하지 않는다.

---

## 11. Backend transaction boundary

불변조건이 걸린 write는 client의 여러 요청으로 분해하지 않는다.

권장 Postgres transaction/RPC:

```text
create_transfer_request(...)
accept_driver_dispatch(...)
edit_transfer_material_fields(...)
cancel_transfer(...)
```

특히 `accept_driver_dispatch`는 한 transaction에서:

1. token hash lookup/lock
2. expiry/revocation 확인
3. request status + revision 확인
4. active assignment 없음 확인
5. DriverAssignment 생성
6. DriverDispatch accepted
7. TransferRequest assigned
8. Guest assignment Notification outbox 생성

을 완료한다.

---

## 12. V1 자동 테스트 필수 목록

### Fare / validation

- 모든 현재 route/direction/passenger tier
- child-seat/terminal-transfer option
- passenger 0 / 8 거부
- SEOULSTN terminal transfer 거부
- inactive/unknown stay 거부
- missing fare rule은 error
- VAT 10%가 암묵적으로 추가되지 않음

### Request identity

- duplicate submit same key
- same key different payload conflict

### Driver

- valid accept
- reject
- expired token
- revoked token
- duplicate accept
- two drivers simultaneous accept
- request cancelled before accept
- request revision changed before accept

### Change/cancel

- assigned request material edit -> needs_reconfirmation
- old assignment superseded
- old dispatch cannot accept
- cancellation prevents later assignment
- queued stale notification not sent

### Messaging

- duplicate provider webhook no-op
- out-of-order delivery webhook does not regress status
- failed notification retry
- dedupe prevents duplicate logical Guest message
- inbound Guest message with ambiguous active request remains unassociated

### Security

- anonymous direct table write denied
- Host A cannot read/write Host B rows
- public request reference alone cannot expose PII

---

## 13. Seed / pilot rules

### Stay master

- PH1/PH2/PH3/SP5/SP6 seed 가능
- WSR는 정확한 주소가 들어오기 전 `is_active=false`

### Capacity

V1에서 capacity rule은 **warning only**다. 명확한 차량별 capacity data가 생기기 전 자동 거절하지 않는다.

초기 warning 후보:

- passenger_count >= 7
- large_luggage_count >= 5

### Driver

V1의 실제 기사 목록은 seed/config로 관리해도 된다. 범용 Driver management UI는 필요 없다.

---

## 14. 실손님 Pilot 전 필수 체크

코드가 완성되어도 아래가 없으면 real Guest pilot을 시작하지 않는다.

- [ ] WhatsApp 실제 번호 inbound/outbound 확인
- [ ] Host manual reply 운영 방식 확인
- [ ] 필요한 WhatsApp template 승인/발송 확인
- [ ] opt-in 문구와 privacy notice 준비
- [ ] production secret 분리
- [ ] production DB backup 정책 확인
- [ ] 실제 Stay address 확인; WSR는 미확정이면 inactive
- [ ] fare table의 실제 청구 금액 확인
- [ ] 실제 Driver 연락처/기사 메시지 확인
- [ ] Kakao Share 링크/버튼을 기사 휴대폰에서 E2E 확인
- [ ] Guest/Driver/Host 실제 휴대폰 3자 왕복 테스트

---

## 15. 개발 시작 순서

외부 계정 설정을 기다리는 동안에도 provider-independent 코드는 시작할 수 있다.

```text
1. branch 생성: work/v1-internal
2. Vite + React + TypeScript skeleton
3. Supabase project/migrations skeleton
4. canonical schema + RLS
5. fare/validation + tests
6. create_transfer_request transaction + API
7. Guest Form wiring
8. fake/test Notification adapter
9. DriverDispatch + signed response transaction
10. 실제 WhatsApp/Kakao adapter 연결
11. exception admin
12. real-device E2E
13. internal pilot
```

중요: WhatsApp Coexistence가 확인되기 전에도 2~9는 구현 가능하지만, **실손님 Pilot은 시작하지 않는다.**

---

## 16. 남아 있는 외부 결정 — 코드 구조를 막지 않음

다음은 adapter/config 경계에 있어 backend foundation을 다시 설계할 이유가 없다.

- 실제 WhatsApp provider/onboarding path
- Coexistence/message echo 지원 수준
- WhatsApp template 이름/승인 상태
- Kakao 완전자동 outbound 여부
- driver response timeout 값
- 개인정보 보존기간의 최종 정책
- guest에게 driver phone을 노출하는 정확한 시점

이 항목은 Phase 0 또는 Pilot 전 gate에서 확정한다.
