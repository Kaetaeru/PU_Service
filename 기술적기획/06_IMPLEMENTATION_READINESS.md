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
3. Guest 운영 template 승인/발송 확인
4. 대표기사 배차 요청 template 승인 및 실단말 수신 확인
5. 대표기사 단말에서 URL 버튼 -> signed 배차 응답 Web 정상 오픈 확인
6. 대표기사 연락처 확보, WhatsApp 사용 가능 여부, opt-in 증거
7. Guest opt-in 문구/개인정보 안내 확정

**모든 외부 메시지는 WhatsApp 하나로 나간다. KakaoTalk은 사용하지 않는다.** V1 baseline은 Host가 지불을 확인하고 [배차 요청]을 누르면 StayOps가 대표기사에게 승인된 template을 자동 발송하는 방식이다. 기사 응답은 항상 signed 배차 응답 Web으로 StayOps에 직접 돌아온다. 상세는 `07_WhatsApp_배차_왕복_설계.md`.

전송 수단이 막히는 경우를 대비해 `PartnerNotificationAdapter`에 `sms` / `manual` fallback을 둔다. 세 경로 모두 같은 signed URL을 보내므로 도메인은 바뀌지 않는다.

---

## 1. 최종 Plan-Critic 발견 사항과 수정

> **2026-09-04 개편 주석**
> 아래 Major A~K는 카카오 전제 시점의 검수 이력이다. 결론 대부분은 그대로 유효하지만 두 항목은 이후 결정으로 대체됐다.
> - **Major B (VAT):** **결론이 뒤집혔다.** Guest가 금액을 아예 보지 않게 되면서 "손님이 보던 금액이 10% 올라 보인다"는 위험이 사라졌다. V1은 세금을 계산하며 Host와 대표기사에게 세금포함 총액을 보여준다. 요금표 단일 기준은 `08_요금표.md`다. 아래 Major B 본문 참조.
> - **Major E (payment):** `payment_type` / `guest_due_krw` / `host_due_krw` 모델은 **폐기**됐다. 지불이 서비스 바깥에서 이뤄지므로 V1은 `payment_confirmed` 플래그 하나만 기록한다. 아래 Major E 본문 참조.

### Major A — Transfer 상태와 메시지 상태가 다시 중복됨

기존 canonical에는 `TransferRequest.guest_notified`가 있었지만 Notification 상태를 별도로 관리한다는 원칙과 중복된다.

**수정:** TransferRequest에서 `guest_notified`를 제거한다. Guest 통지 여부는 Notification으로만 판단한다.

### Major B — 기존 HTML과 새 MVP 사이의 VAT 10% 규칙이 충돌함

최초 HTML은 route fare + option을 최종 표시 금액으로 사용한다. 이후 `MVP_SPEC.md`에서 근거가 확정되지 않은 상태로 `tax = supply * 10%`를 추가했다. 그대로 구현하면 Guest가 보던 금액이 10% 상승할 수 있다.

**최초 수정(뒤집힘):** V1 Guest-facing fare를 fare table + option 합계로 고정하고 세금을 추가하지 않는다.

**현재 결정(2026-09-04):** 이 우려의 전제는 "Guest가 보던 금액이 10% 올라 보인다"였다. **Guest가 금액을 아예 보지 않게 되면서 전제가 사라졌다.**

따라서 세금 계산을 복원한다.

```text
공급가 = base_fare_krw + option_fare_krw
세금   = ROUND(공급가 × 0.1)
총액   = 공급가 + 세금
```

- `08_요금표.md`의 표기 금액은 **세금 별도 공급가**다.
- **Guest에게는 어떤 금액도 표시하지 않는다.** 폼·접수 확인·배차 안내 어디에도 없고, 계산 결과를 클라이언트 응답에 담지도 않는다.
- Host 화면과 대표기사 배차 요청 메시지에는 **세금포함 총액**을 표시한다.

이 구조는 실제 운영 장부의 `공급가 / 세금(10%) / 세금포함 총액`과 일치한다. 첫 검토에서 발견됐던 "문서는 세금 없음, 장부는 10% 청구" 불일치가 이로써 해소된다.

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

**최초 수정(폐기됨):** `payment_type = pending | host | guest | split` + `guest_due_krw` / `host_due_krw` 분담 모델.

**현재 결정(2026-09-04):** 위 모델을 **전부 폐기한다.**

지불은 이 서비스 바깥에서 이뤄진다. StayOps는 금액을 정산하지 않고 수금·입금을 관리하지 않는다. V1이 기록하는 것은 Host가 확인했다는 사실 하나뿐이다.

```text
payment_confirmed            boolean
payment_confirmed_by
payment_confirmed_at
```

불변조건:

```text
payment_confirmed = false 이면 배차 요청을 발송하지 않는다.
```

Host가 이 확인을 하고 [배차 요청]을 누르는 것이 정상 건에서 Host의 유일한 조작이다.

실제 입금/수금/분담 정산은 Post-V1 settlement 영역이다. 그때를 위해 fare snapshot은 V1부터 보존한다.

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

`pending -> awaiting_response`의 트리거는 **대표기사 Notification의 발송 성공(sent)**이다. Host의 추가 조작이 필요 없다.

`notification_channel = manual`인 경우에만 Host의 "전달 완료" 확인으로 전환한다.

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
route valid (fare rule 존재)
passenger_count 1..7
fare snapshot exists
payment_confirmed = true
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
origin_code                ICN | GMP | SEOULSTN | ILSAN | EVERLAND | STAY
destination_code           STAY | ICN | GMP | SEOULSTN | ILSAN | EVERLAND
terminal_code              conditional (공항일 때만)
stay_code                  required active stay
child_seat_count           0..4
child_seat_notes           required when child_seat_count > 0
terminal_transfer          boolean, 공항에서만 허용
special_request            optional, max 1000
```

거점 코드와 유효 노선 조합, 미확정 항목은 `08_요금표.md`를 따른다. 요금 규칙이 없는 조합은 신청을 받지 않는다.

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
supply_amount_krw       base + option
tax_amount_krw          ROUND(supply × 0.1)
total_amount_krw        supply + tax
fare_rule_version / source
calculated_at
```

- **Guest 응답에 금액을 담지 않는다.** 손님 브라우저는 요금을 계산하지도 받지도 않는다.
- server가 fare rule을 조회한다.
- **요금표에 없는 조합은 0원으로 진행하지 않고 validation error.**
- 인원 1~7을 벗어나면 상위 구간을 적용하지 않고 거부한다.
- 과거 request의 snapshot은 fare rule 변경으로 재계산하지 않는다.
- fare rule은 방향(`pickup`/`dropoff`)까지 명시해 저장한다. "방향이 없으면 같은 값" 같은 암묵 규칙을 코드에 두지 않는다.

### 지불 확인

```text
payment_confirmed            boolean
payment_confirmed_by
payment_confirmed_at
```

지불 자체는 서비스 바깥에서 이뤄진다. StayOps는 Host가 확인했다는 사실만 기록하고, 확인 전에는 배차 요청을 발송하지 않는다.

**요금은 대표기사에게 보내는 배차 요청 메시지에 넣지 않는다.**

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
디자인 기준: 09_UI_테마.md (ems 데모 디자인 시스템)
Cloudflare Pages 배포
```

V1에서 UI framework/component library는 추가하지 않는다. `09_UI_테마.md`의 CSS 변수와 컴포넌트 값을 그대로 쓰면 충분하다.

세 화면(손님 신청 폼 / 배차 응답 Web / Host 운영 화면)이 같은 스타일시트를 공유한다.

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
    create-dispatch/
    respond-dispatch/
    whatsapp-webhook/
    process-notifications/

public/

기술적기획/
```

`src/app/driver/`는 대표기사용 배차 응답 화면이다.

`pickup_2.html`은 입력 항목과 문구의 reference로 남긴다. **시각 디자인의 기준은 `09_UI_테마.md`이며 `pickup_2.html`이 아니다.** 구현 후에도 즉시 삭제하지 않는다.

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
5. DriverAssignment 생성 (기사 전화번호 E.164, 차량번호)
6. DriverDispatch accepted
7. TransferRequest assigned
8. Guest 배차 안내 Notification outbox 생성
9. Host 배차 완료 알림 Notification outbox 생성

을 완료한다.

---

## 12. V1 자동 테스트 필수 목록

### Fare / validation

- `08_요금표.md`의 모든 확정 노선 / passenger tier
- child-seat / terminal-transfer option
- passenger 0 / 8 거부
- 공항이 아닌 거점에서 terminal transfer 거부
- **요금표에 없는 노선 조합 거부** (0원으로 진행하지 않음)
- inactive/unknown stay 거부
- missing fare rule은 error (일산↔홍대, 에버랜드↔홍대 포함)
- 세금 = ROUND(공급가 × 0.1), 총액 = 공급가 + 세금
- 인천공항은 방향별 금액이 다르고, 나머지 거점은 양방향 동일
- **Guest 응답 payload에 금액 필드가 없음**

### Request identity

- duplicate submit same key
- same key different payload conflict

### Driver

- valid accept
- reject
- expired token
- revoked token
- duplicate accept (멱등 — 최초 결과 반환)
- 동시 중복 제출에도 active assignment 하나
- request cancelled before accept
- request revision changed before accept
- payment_confirmed = false 상태에서 dispatch 생성 거부
- 기사 전화번호가 E.164로 정규화되어 저장됨

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
- inbound 역할 라우팅: 대표기사 번호는 partner, 손님 번호는 guest, 미등록 번호는 unknown
- 대표기사 자유 텍스트는 파싱되지 않고 링크 재안내 1회만 발생
- 24시간 창이 닫힌 상태에서 자유 텍스트 발송을 시도하지 않음

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

### DispatchPartner

V1의 대표기사는 **한 명**이며 seed/config로 관리한다. 기사 관리 UI는 필요 없다.

개별 운행 기사는 저장된 주체가 아니다. 대표기사가 배차 응답에 입력한 전화번호·차량번호가 해당 건의 기사 정보다.

---

## 14. 실손님 Pilot 전 필수 체크

코드가 완성되어도 아래가 없으면 real Guest pilot을 시작하지 않는다.

- [ ] WhatsApp 실제 번호 inbound/outbound 확인
- [ ] Host manual reply 운영 방식 확인
- [ ] Guest template 승인/발송 확인
- [ ] 대표기사 배차 요청 template 승인/발송 확인
- [ ] 대표기사 단말에서 URL 버튼 -> signed 배차 응답 Web 오픈 확인
- [ ] 대표기사 연락처 확보 / WhatsApp 사용 가능 여부 확인
- [ ] 대표기사 opt-in 증거 확보
- [ ] Guest/Partner inbound 역할 라우팅 검증
- [ ] opt-in 문구와 privacy notice 준비
- [ ] production secret 분리
- [ ] production DB backup 정책 확인
- [ ] 실제 Stay address 확인; WSR는 미확정이면 inactive
- [ ] `08_요금표.md` 미확정 항목 확정
- [ ] Guest/대표기사/Host 실제 휴대폰 3자 왕복 테스트

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
10. 실제 WhatsApp adapter 연결 (Guest / 대표기사 / Host)
11. Host 운영 화면
12. real-device E2E
13. internal pilot
```

중요: WhatsApp Coexistence와 template 승인이 확인되기 전에도 2~9는 구현 가능하지만, **실손님 Pilot은 시작하지 않는다.**

---

## 16. 남아 있는 외부 결정 — 코드 구조를 막지 않음

다음은 adapter/config 경계에 있어 backend foundation을 다시 설계할 이유가 없다.

- 실제 WhatsApp provider/onboarding path
- Coexistence/message echo 지원 수준
- WhatsApp template 이름/승인 상태
- 한 template에 quick-reply 버튼 + URL 버튼 조합 가능 여부 (24시간 창 확보용)
- Guest용/Partner용 번호 분리 여부 (V1은 단일 번호)
- 배차 응답 timeout 값 (초기 후보 60분)
- 개인정보 보존기간의 최종 정책
- `08_요금표.md` §5의 미확정 항목

이 항목은 Phase 0 또는 Pilot 전 gate에서 확정한다.
