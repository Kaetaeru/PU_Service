# StayOps Canonical System

> 상태: 현재 제품/기술 기획의 **최우선 Source of Truth**
> 최종 정리: 2026-08-25

## 0. 문서 우선순위

이 문서는 StayOps의 큰 틀, 채널 역할, 핵심 상태, MVP 경계를 정의한다.

기획 문서 사이에 충돌이 있으면 아래 순서로 해석한다.

1. `00_CANONICAL_SYSTEM.md` — 현재 제품 동작과 불변조건
2. `05_DEVELOPMENT_PLAN.md` — 구현 순서와 단계별 완료조건
3. `REQUIREMENTS.md` — 전체 사업/운영 요구와 장기 확장 아이디어
4. `MVP_SPEC.md` — 기존 MVP 상세 초안
5. `01~04` 문서 — HTML/API/기사배차에 대한 분석 및 세부 설계 근거

기존 문서에서 고객 리턴 채널을 KakaoTalk으로 적었거나 호스트가 차량번호를 수동 재입력하도록 적은 부분은 이 문서에 의해 대체된다.

---

## 1. 제품 목표

StayOps의 목적은 숙소 호스트가 게스트와 파트너 사이에서 반복적으로 메시지를 읽고, 복사하고, 재작성하고, 다시 전달하는 **수동 중계자 역할**을 하지 않게 만드는 것이다.

핵심 제품 불변조건:

> 정상적인 업무에서는 호스트가 정보를 복사·재전달·재입력하지 않는다. 호스트는 예외 확인, 판단, 승인, 수동 대화와 정산에 집중한다.

첫 번째 실제 제품 범위는 공항/서울역 픽업·드랍오프다. 청소, 세탁, iCal 자동화는 이 흐름이 실운영에서 안정화된 뒤 확장한다.

---

## 2. Actor와 기본 채널

| Actor | 기본 채널 | 역할 |
|---|---|---|
| Guest | WhatsApp + Guest Web | 요청 입력, 자동 안내 수신, 필요 시 호스트와 직접 대화 |
| Driver | KakaoTalk + signed Driver Web | 배차 요청 수신, 수락/거절, ETA·차량정보 입력 |
| Host | StayOps Admin + WhatsApp Business App | 예외 관리, 수동 게스트 응답, 운영 수정, 정산 |
| StayOps | Managed Backend | 예약·배차·메시지·정산 상태의 Source of Truth |

WhatsApp과 KakaoTalk은 상태 저장소가 아니다. 메시지 전송/수신 채널이며 업무 상태는 StayOps에 저장한다.

---

## 3. 권장 배포 구조

직접 VPS나 상시 실행 서버를 관리하지 않는 managed/serverless 구조를 기본으로 한다.

```text
Guest Web -----------┐
Driver Response Web -┼----> StayOps Frontend
Host Admin ----------┘             |
                                  v
                           Managed Backend
                         (1차 권장: Supabase)
                    ┌──────────┬───────────┐
                    |          |           |
                 Postgres     Auth    Edge Functions
                    |                      |
                    |             ┌────────┴─────────┐
                    |             v                  v
                    |      WhatsApp Platform    Kakao Adapter
                    |             |                  |
                    └─────────────┼──────────────────┘
                                  v
                         audit / message state
```

기술 선택은 구현 단계에서 검증하지만, 제품 설계는 특정 SDK에 직접 종속시키지 않는다.

---

## 4. Guest Contact 계약

게스트의 기본 연락 좌표는 **WhatsApp 번호**다. 별도 일반 전화번호와 WhatsApp 번호를 중복 입력받지 않는다.

### 필수 연락 필드

```text
guest_name
whatsapp_phone_e164
preferred_language
whatsapp_opt_in
whatsapp_opt_in_at
whatsapp_opt_in_policy_version
```

규칙:

- 전화번호는 UI에서 국가코드 + 현지번호로 입력받을 수 있지만 저장은 E.164 형태로 정규화한다.
- 예: `+821012345678`, `+31612345678`.
- WhatsApp 운영 메시지를 발송하기 위한 명시적 opt-in 문구와 동의 시점을 보존한다.
- 동일 전화번호가 여러 예약을 가질 수 있으므로 전화번호 자체를 예약 PK로 사용하지 않는다.
- `transfer_request_id`가 예약의 영속 식별자이고 WhatsApp 번호는 연락/대화 상관관계 키다.

---

## 5. Guest Transfer 입력값

### 5.1 Guest

필수:

- 대표 게스트 영문명
- WhatsApp 번호
- 선호 언어
- WhatsApp 운영 안내 수신 동의

### 5.2 Transfer direction

```text
pickup  = AIRPORT/STATION -> STAY
dropoff = STAY -> AIRPORT/STATION
```

### 5.3 Schedule

#### Pickup

필수 권장 계약:

- arrival date
- scheduled landing time
- flight number (공항 픽업에서는 필수)
- port code
- terminal/meeting point는 확보 가능한 경우 구조화해서 저장

시간 의미는 `flight_arrival`이다.

#### Drop-off

필수:

- service date
- accommodation pickup time
- destination port code

항공편 번호는 운영상 도움이 되지만 초기 계약에서는 선택으로 둘 수 있다.

시간 의미는 `accommodation_pickup`이다.

### 5.4 Capacity / route

- passenger_count: 1~7 기본 서비스 범위
- large_luggage_count: 명시 입력 권장
- stay_code
- port_code: ICN / GMP / SEOULSTN

수하물과 인원 조합은 차량 용량 경고에 사용하되, 구체적인 경고 임계값은 설정 데이터로 둔다.

### 5.5 Add-ons

- child_seat_count: 0~4
- terminal_transfer: boolean
- special_request: optional text

서울역에서는 terminal transfer를 허용하지 않는다.

### 5.6 Location

실시간 위치는 최초 예약 폼의 필수값으로 받지 않는다.

필요한 경우 픽업 직전 WhatsApp으로 위치공유 링크를 보내고 게스트의 명시적 브라우저 권한으로 수집한다.

```text
latitude
longitude
accuracy_m
captured_at
expires_at
```

공항 실내 위치는 부정확할 수 있으므로 terminal/gate/meeting point와 함께 사용한다.

---

## 6. 정상 픽업 왕복 플로우

```text
1. Guest가 Transfer Form 제출
        |
        v
2. StayOps가 validation + authoritative fare 계산
        |
        v
3. TransferRequest 저장 + request_id 생성
        |
        v
4. Guest WhatsApp으로 접수 확인
        |
        v
5. DriverDispatch 생성
        |
        v
6. 기사 KakaoTalk으로 배차 요청 + signed response link
        |
        v
7. 기사: 수락/거절 + ETA + 차량번호/차량정보 입력
        |
        v
8. StayOps가 DriverAssignment를 transaction으로 확정
        |
        v
9. Guest WhatsApp으로 배차정보 자동 반환
        |
        v
10. 운행 완료 -> 정산 대상 반영
```

호스트는 정상 플로우에서 메시지 복사/재전달/차량번호 재입력을 하지 않는다.

---

## 7. WhatsApp 운영 모델

### 7.1 목표

각 호스트는 자신의 WhatsApp Business 번호를 계속 직접 관리할 수 있어야 한다.

선호 연결 모델은 **WhatsApp Business App + WhatsApp Business Platform/Cloud API Coexistence**다.

이 경우 기대 동작:

- StayOps가 API로 자동 운영 메시지를 전송한다.
- 게스트의 WhatsApp 답장은 webhook으로 StayOps에 들어온다.
- 호스트는 기존 WhatsApp Business App에서 같은 대화를 읽고 직접 답장할 수 있다.
- Business App에서 호스트가 보낸 메시지는 message echo를 통해 StayOps가 대화 이력에 반영할 수 있다.

### 7.2 판매 시 중요한 제약

Coexistence를 모든 호스트에게 무조건 가능하다고 가정하지 않는다.

호스트 온보딩에서 다음을 **실제 연결 테스트로 검증**해야 한다.

```text
whatsapp_connection_mode =
  coexistence
  | api_only
  | manual_only
```

MVP/SaaS의 선호 모드는 `coexistence`지만 계정·지역·온보딩 경로·공급자 조건에 따라 가능 여부가 달라질 수 있으므로 판매 약속 전에 검증한다.

### 7.3 자동과 수동의 경계

MVP에서 StayOps는 자유 대화를 AI로 자동 답변하지 않는다.

자동화 대상:

- 예약 접수
- 배차 완료
- 차량번호/ETA
- 일정 변경
- 미팅 위치
- 기타 명확한 상태 기반 운영 알림

사람 판단이 필요한 게스트 자유 메시지:

```text
Guest message -> webhook -> StayOps에 기록 + needs_host_reply
                              |
                              v
                    Host가 WhatsApp Business App에서 답장
                              |
                              v
                    message echo -> StayOps 이력 반영
```

이렇게 하면 StayOps가 별도 완전한 채팅 클라이언트를 MVP에 만들 필요가 없다.

### 7.4 메시지 정책

- business-initiated 메시지는 WhatsApp 정책상 필요한 경우 승인된 template을 사용한다.
- 고객 서비스 window/메시지 정책은 WhatsApp adapter가 담당하며 도메인 상태와 분리한다.
- inbound/outbound/message echo는 provider message id로 중복 제거한다.
- `message sent`와 `guest notified`는 실제 전송 결과를 구분한다.

---

## 8. Driver / KakaoTalk 운영 모델

기사에게 자유 텍스트 카카오 답장을 파싱하는 구조에 의존하지 않는다.

```text
Kakao notification
  -> signed single-use Driver Response URL
  -> accept / reject
  -> ETA
  -> vehicle plate
  -> driver name/phone
  -> optional model/color/memo
```

서버는 dispatch token, 만료시각, 현재 dispatch state를 검증한다.

기사 A가 거절하거나 timeout이면 새로운 DriverDispatch를 만든다. 하나의 요청에 대한 동시 중복 assignment는 transaction/unique constraint로 막는다.

Kakao 자동 발송의 실제 공급자/API 경로는 아직 외부 의존성이다. 도메인은 `DriverNotificationAdapter` 뒤에 두고, 실운영 전 반드시 발송 경로를 검증한다.

---

## 9. 상태 모델

하나의 enum에 모든 것을 합치지 않는다.

### 9.1 TransferRequest

```text
requested
-> awaiting_driver
-> assigned
-> guest_notified
-> completed

예외:
cancelled
failed
```

### 9.2 DriverDispatch

```text
created
-> sent
-> awaiting_response
-> accepted | rejected | expired
```

### 9.3 Notification

```text
queued
-> sent
-> delivered

또는
failed
```

불변조건:

- `DriverDispatch.sent`는 배차완료가 아니다.
- `TransferRequest.assigned`는 유효한 DriverAssignment가 저장된 뒤에만 가능하다.
- 취소/만료된 dispatch token으로 assignment를 만들 수 없다.
- 동일 논리적 이벤트의 알림 재시도는 중복 게스트 메시지를 만들지 않아야 한다.

---

## 10. Canonical domain model

### Host

```text
host_id
brand_name
settings
whatsapp_connection_mode
```

### Stay

```text
stay_id / stay_code
host_id
name
address
area_code
is_active
```

### TransferRequest

```text
request_id
host_id
stay_id

guest_name
whatsapp_phone_e164
preferred_language
whatsapp_opt_in_at

direction
service_date
service_time
time_semantics
flight_no
passenger_count
large_luggage_count
port_code
addons
special_request

fare_snapshot
payment_state
transfer_status
created_at
updated_at
```

### Driver

```text
driver_id
host_id or provider_id
name
phone
notification_route
is_active
```

### DriverDispatch

```text
dispatch_id
request_id
driver_id
status
sent_at
responded_at
expires_at
response_token_hash
```

### DriverAssignment

```text
request_id
dispatch_id
driver_id
eta
vehicle_plate
vehicle_model
vehicle_color
driver_name
driver_phone
assigned_at
```

### GuestLocation

```text
request_id
latitude
longitude
accuracy_m
captured_at
expires_at
```

### MessageEvent / Notification

```text
message_id
request_id
host_id
audience
channel
provider_message_id
direction
message_type
status
created_at
sent_at
delivered_at
failed_reason
```

### FareRule

숙소/노선/방향/인원 구간에 따른 서버 권위 요금 설정. 예약 생성 시 금액 snapshot을 TransferRequest에 보존한다.

---

## 11. Multi-host / 판매 구조

처음부터 모든 tenant-owned 데이터에 `host_id`를 둔다.

```text
Host A
  - WhatsApp Business A
  - Stay A1/A2
  - Driver list A
  - Fare rules A

Host B
  - WhatsApp Business B
  - Stay B1
  - Driver list B
  - Fare rules B
```

인증된 호스트는 다른 host의 예약/게스트/기사/정산 데이터를 읽을 수 없어야 한다. DB 레벨 tenant isolation(RLS 등)을 필수로 검증한다.

SaaS URL 예:

```text
/request/{host_slug}
/admin
/driver/dispatch/{token}
```

---

## 12. 보안·개인정보

- browser에 provider secret/API key를 넣지 않는다.
- guest phone과 정밀 location은 로그에 평문으로 불필요하게 남기지 않는다.
- signed driver token은 짧은 TTL 또는 1회성으로 사용한다.
- 공개 request id만으로 개인정보 조회가 가능하면 안 된다.
- 위치 정보는 운행 완료/취소 뒤 짧은 보존 정책을 적용한다.
- WhatsApp opt-in 증거와 적용 문구 version을 보존한다.
- tenant isolation을 DB 정책으로 검증한다.
- 예약 금액은 생성 시 snapshot으로 보존해 요금표 변경이 과거 정산을 바꾸지 않게 한다.

---

## 13. Failure / fallback

### WhatsApp 자동 발송 실패

- Notification을 failed로 기록
- Host admin에 예외 노출
- 필요 시 manual WhatsApp 또는 SMS fallback은 후속 adapter로 추가

### WhatsApp Coexistence 불가

- 호스트 온보딩을 차단하거나 `api_only/manual_only` 모드로 명시적으로 전환
- Coexistence가 된 것처럼 UI에서 가장하지 않는다.

### 기사 응답 없음

```text
Dispatch A expired
-> Dispatch B 생성
-> 새 기사에게 요청
```

### 기사 중복 수락

assignment 확정은 atomic transaction으로 한 번만 성공한다.

### 게스트가 일정 변경

수정된 서비스 시간은 기존 assignment/notification과 함께 audit event를 남기고 필요하면 재배차 또는 기사 재통지를 발생시킨다.

---

## 14. MVP에서 하지 않는 것

첫 왕복 플로우가 안정화되기 전에는 아래를 구현 범위에 넣지 않는다.

- AI 자유대화 자동응답
- 자체 WhatsApp full inbox UI
- 청소/세탁 전체 자동화
- iCal 완전 자동동기화
- 복잡한 파트너 marketplace
- 다중 기사 동시 fan-out 배차
- SaaS 과금 시스템

---

## 15. Plan-Critic 결정 사항

### Major 1 — 기존 MVP가 호스트를 다시 중계자로 만듦

기존 `MVP_SPEC.md`는 기사 회신 후 호스트가 차량번호/연락처를 수동 입력하고 게스트 안내를 다시 생성하는 흐름이 있었다.

**수정:** 기사 signed response -> DriverAssignment -> Guest WhatsApp 자동 리턴을 canonical 정상 플로우로 확정한다.

### Major 2 — 게스트를 다시 찾을 안정적 연락 좌표가 없었음

기존 폼/모델에는 비동기 결과를 보낼 guest contact가 빠져 있었다.

**수정:** `whatsapp_phone_e164 + opt-in`을 필수 Guest Contact 계약으로 확정한다.

### Major 3 — Coexistence를 모든 판매 고객에게 보장할 수 없음

같은 WhatsApp Business App 번호와 API를 함께 쓰는 경로는 존재하지만 실제 계정/온보딩 조건을 확인하지 않고 모든 호스트에게 가능하다고 약속하면 판매 단계에서 실패할 수 있다.

**수정:** `coexistence/api_only/manual_only` 연결 모드와 온보딩 feasibility gate를 둔다.

### Major 4 — 배차 상태와 메시지 상태가 섞여 있었음

`waiting/assigned/notified` 한 enum만으로는 기사에게 메시지는 갔지만 수락하지 않은 상태, 고객 알림 실패 등을 표현할 수 없다.

**수정:** TransferRequest / DriverDispatch / Notification 상태를 분리한다.

### Major 5 — Kakao 발송 경로가 아직 미검증

기사 signed response 방식은 정의됐지만 StayOps가 어떤 공식/상용 경로로 기사에게 Kakao 메시지를 선제 발송할지는 확정되지 않았다.

**수정:** `DriverNotificationAdapter`로 격리하고 개발 초기 feasibility 단계에서 실제 발송 경로를 검증한다.

---

## 16. 아직 남은 제품 결정

아래는 구현 전에 또는 pilot 중 실제 운영 근거로 확정한다.

1. WhatsApp Coexistence의 실제 onboarding provider/경로
2. 기사 Kakao 자동 발송 경로와 비용/템플릿 제약
3. 기사 응답 timeout 기본값
4. 수하물 기반 차량용량 경고 규칙
5. 게스트에게 기사 전화번호를 언제 공개할지
6. guest/location/message 데이터 보존기간
7. `WSR` 상세 주소
8. 호스트가 서비스 변경을 승인해야 하는 예외 범위

이 결정들은 핵심 구조를 막지 않으며 adapter/config/policy 경계에 둔다.
