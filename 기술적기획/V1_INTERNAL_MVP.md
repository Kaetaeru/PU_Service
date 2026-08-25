# StayOps V1 — Internal Operations MVP

> 목적: 판매 전에 **우리가 실제 숙소 운영에 매일 사용할 수 있는 최소 제품**의 범위를 고정한다.
> 기준 문서: `00_CANONICAL_SYSTEM.md`
> 구현 순서: `05_DEVELOPMENT_PLAN.md`
> 상태: V1 scope canonical for release-boundary decisions

---

## 1. V1의 정의

V1은 SaaS 판매판이 아니다.

V1의 목표는 현재 운영 중인 숙소의 공항/서울역 픽업·드랍오프 업무를 실제 게스트와 실제 기사에게 사용하면서, 정상 건에서 호스트가 메시지를 복사·재작성·재입력하지 않아도 한 건의 transfer가 끝나는 것이다.

```text
Guest Transfer Web
-> StayOps
-> Guest WhatsApp 접수 확인
-> Driver Kakao 배차 요청
-> signed Driver Response
-> DriverAssignment
-> Guest WhatsApp 배차 결과
-> 운행 완료
```

V1에서 호스트는 다음 역할만 맡는다.

- 배차 시작/기사 선택 같은 명시적 운영 승인
- 예외 확인 및 복구
- 게스트 자유 문의에 대한 수동 WhatsApp 답장
- 일정/항공편 변경, 취소, 재배차 같은 운영 판단
- 필요 시 예외적인 수동 보정

정상 플로우에서 차량번호, ETA, 기사정보 또는 게스트 안내문을 복사해서 다시 입력하지 않는다.

---

## 2. Plan-Critic으로 고정한 V1 경계

### Major A — "내부용"을 이유로 실패 복구를 빼면 실사용할 수 없음

실제 운영에서는 기사 무응답, 메시지 실패, 일정 변경, 취소가 반드시 발생한다.

**V1 수정:** exception dashboard와 최소 복구 액션을 V1 필수로 포함한다.

### Major B — Kakao 완전자동화를 V1의 선행조건으로 만들면 외부 공급자 때문에 제품 전체가 막힐 수 있음

기사 응답은 구조화되어야 하지만 기사 알림 자체는 내부 운영 단계에서 한 번의 operator action을 허용할 수 있다.

**V1 수정:**

- 선호: `DriverNotificationAdapter`가 Kakao 메시지를 자동 전송
- 허용 fallback: Host가 StayOps에서 생성된 배차 메시지를 **한 번의 share/send action**으로 기사에게 전달
- 금지: Host가 기사 메시지를 수동으로 재작성하거나 차량정보를 다시 입력

즉 V1의 핵심 자동화 경계는 "기사 알림을 누가 클릭해 보내는가"가 아니라 **기사 응답 이후 정보가 StayOps로 직접 들어와 Guest에게 자동 반환되는가**다.

### Major C — 내부 실사용에는 payment 책임 정보가 필요함

기사에게 누가 얼마를 받는지 불명확하면 실제 현장에서 다시 문의가 생긴다.

**V1 수정:** dispatch 전에 최소 payment instruction 상태를 가진다.

```text
payment_type = host | guest | split

guest_paid / guest_due
host_amount
```

정책 자동화가 아직 완성되지 않은 경우 Host가 dispatch 전 한 번 확인/수정할 수 있다. 이 값은 기사 메시지와 정산용 snapshot에 사용한다.

### Major D — 판매용 멀티호스트 기능을 V1에 넣으면 범위가 불필요하게 커짐

**V1 수정:** DB에는 `host_id`와 tenant isolation을 유지하지만, self-service host onboarding / billing / 범용 설정 UI는 V1 밖으로 둔다.

---

## 3. V1에서 실제로 사용하는 대상

### Guest

로그인하지 않는다.

사용:

- Guest Transfer Web
- WhatsApp

### Driver

로그인하지 않는다.

사용:

- KakaoTalk 배차 요청
- signed Driver Response Web

### Host / Operator

내부 운영 계정으로 로그인한다.

사용:

- StayOps Admin
- WhatsApp Business App

V1에는 일반 고객이 가입하는 계정 체계가 없다.

---

## 4. V1 Guest 입력 계약

### Guest contact — 필수

```text
guest_name
whatsapp_phone_e164
preferred_language
whatsapp_opt_in
whatsapp_opt_in_at
whatsapp_opt_in_policy_version
```

WhatsApp 번호가 Guest의 기본 연락 좌표다.

### Transfer — 필수/조건부

```text
direction = pickup | dropoff
service_date
service_time
flight_no
passenger_count
large_luggage_count
port_code
stay_code
child_seat_count
terminal_transfer
special_request
```

규칙:

- pickup의 time = scheduled flight landing time
- dropoff의 time = accommodation pickup time
- airport pickup에서는 flight number를 필수로 한다.
- passenger 기본 서비스 범위 = 1~7
- `large_luggage_count`를 별도 구조화 필드로 받는다.
- 서울역에서는 terminal transfer를 허용하지 않는다.
- client fare는 표시용이며 server fare snapshot이 권위값이다.

### V1에서 받지 않아도 되는 것

- 실시간 GPS 위치: 기본 V1 필수가 아님
- 여권 정보
- 이메일
- 별도 일반 전화번호

실시간 위치는 pilot에서 필요성이 확인되면 request-scoped location link로 추가한다.

---

## 5. V1 정상 플로우

### 5.1 Guest request

```text
Guest form submit
-> validation
-> authoritative fare calculation
-> TransferRequest 저장
-> request_id 생성
-> WhatsApp receipt
```

### 5.2 Dispatch 준비

StayOps는 다음을 기사에게 보낼 수 있는 상태여야 한다.

- 서비스 날짜/시간
- pickup/dropoff
- flight
- passenger / luggage
- route / stay
- add-ons
- guest display name
- payment instruction
- signed response URL

기사 선택은 V1에서 Host가 할 수 있다.

### 5.3 Driver response

```text
Accept / Reject
ETA
Vehicle plate
Driver name
Driver phone
Vehicle model/color optional
```

수락 후 Host가 이 정보를 재입력하지 않는다.

### 5.4 Guest return

DriverAssignment가 확정되면 StayOps가 Guest WhatsApp으로 자동 안내한다.

최소 내용:

- booking/reference
- driver name
- vehicle plate
- ETA 또는 확정 픽업시각
- meeting point
- driver contact (운영 정책상 허용 시)

### 5.5 Completion

Host가 운행 완료를 확인하거나 운영 규칙에 따라 `completed`로 마감한다.

V1에서는 복잡한 자동 trip tracking이 필요하지 않다.

---

## 6. V1 Host Admin — 최소 화면

V1 Admin은 CRM 전체가 아니라 **운영 및 예외 복구 화면**이다.

### 6.1 Today / Upcoming

표시:

- 오늘/내일 transfer
- Guest
- service time
- route
- transfer status
- driver/vehicle if assigned

### 6.2 Needs Attention

최우선 표시:

- driver not assigned
- dispatch rejected
- dispatch expired / no response
- WhatsApp send failed
- guest message needs host reply
- missing required information
- schedule conflict
- capacity warning
- cancelled/changed after dispatch

### 6.3 Request detail actions

V1 필수:

- Guest/route/request 상세 조회
- 시간/항공편 수정
- payment instruction 확인/수정
- 기사 선택 및 배차 요청
- 기사 재배차
- cancel
- notification retry
- WhatsApp conversation 열기
- 예외 상황에서 manual assignment 입력/수정
- completed 처리

`manual assignment`는 정상 경로가 아니라 비상 복구 기능이다.

---

## 7. V1 WhatsApp 범위

우리 내부 운영에 사용하는 WhatsApp Business 번호 1개를 먼저 연결한다.

V1 목표 모드:

```text
coexistence
```

필수 검증:

- Guest inbound webhook
- StayOps outbound operational message
- Host가 WhatsApp Business App에서 같은 대화를 읽음
- Host의 Business App 수동 답장
- 가능하면 message echo를 StayOps 이력에 반영
- provider message id dedup

### V1 자동 메시지

필수:

1. transfer request received
2. driver assigned / vehicle information
3. 운영자가 발생시킨 주요 schedule change 안내

그 외 체크인/체크아웃/후기 요청 자동화는 V1 밖이다.

### 자유 메시지

AI 자동 답변을 하지 않는다.

```text
Guest free-text
-> webhook/message event
-> needs_host_reply
-> Host가 WhatsApp Business App에서 수동 답장
```

---

## 8. V1 Driver / Kakao 범위

V1의 필수는 **기사 응답의 구조화**다.

필수:

- 배차 요청 메시지 생성
- signed response URL
- accept/reject
- ETA/vehicle 입력
- token expiry/invalid state 검사
- reject/timeout 후 redispatch
- one request -> one active assignment 보장

### Kakao 발송 수준

둘 중 하나면 V1 운영 가능:

A. 자동 Kakao outbound adapter

B. Host가 StayOps에서 누르는 one-click share/send fallback

B를 사용해도 메시지 내용은 StayOps가 생성하고, 기사 응답은 signed web으로 StayOps에 직접 돌아와야 한다.

---

## 9. V1 Backend / 데이터

관리형 backend를 사용한다. 첫 후보는 Supabase이며 Phase 0 spike로 확정한다.

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

필수 기술 동작:

- migrations
- auth for internal Host
- `host_id`
- tenant isolation/RLS
- server-authoritative fare
- fare snapshot
- idempotent Guest create
- signed Driver token
- audit timestamps/events
- notification dedup/retry state

### V1의 멀티호스트 범위

DB 구조와 RLS는 멀티호스트를 견딜 수 있어야 하지만 UI는 **우리 내부 Host만 운영 가능하면 충분**하다.

---

## 10. V1 실패/복구 규칙

### WhatsApp send failed

```text
notification failed
-> Needs Attention
-> retry
-> 필요 시 Host가 WhatsApp App에서 직접 안내
```

### Driver reject

```text
Dispatch A rejected
-> Host가 다른 Driver 선택
-> Dispatch B
```

### Driver timeout

```text
Dispatch expired
-> Needs Attention
-> redispatch
```

### Guest schedule change

Host가 request를 수정한다.

이미 assignment가 있다면:

- 기존 assignment를 조용히 덮어쓰지 않는다.
- change/audit event를 남긴다.
- 기사 재확인/재배차 필요 상태를 표시한다.
- Guest에게 변경 확정 내용을 다시 안내한다.

### Cancellation

- cancelled request는 새 assignment를 받을 수 없다.
- 이미 기사에게 보낸 dispatch가 있으면 취소 사실을 운영자에게 명확히 표시한다.

---

## 11. V1에 포함하지 않는 것

다음은 판매 전 내부 MVP의 필수 조건이 아니다.

- Host self-service signup/onboarding
- SaaS billing/subscription
- host-specific public settings UI 전체
- 여러 WhatsApp 계정의 범용 onboarding UI
- AI Guest auto-reply
- StayOps 자체 full WhatsApp inbox
- iCal 자동 수집
- 청소/세탁 자동화
- 체크인/체크아웃 자동 메시지 전체
- 자동 flight tracking
- 복잡한 GPS/live tracking
- 기사 다중 fan-out marketplace
- 정산 자동화 전체
- Excel 대체

V1 동안 월말 정산은 기존 운영 장부를 계속 사용한다.

단, 이후 정산 이관을 위해 fare/payment snapshot 데이터는 V1부터 보존한다.

---

## 12. V1과 개발 Phase 매핑

`05_DEVELOPMENT_PLAN.md` 기준:

```text
Phase 0  External feasibility      -> V1 필수
Phase 1  Backend foundation        -> V1 필수
Phase 2  Guest Request             -> V1 필수
Phase 3  Driver round trip         -> V1 필수
Phase 4  Guest WhatsApp return     -> V1 필수
Phase 5  Host exception dashboard  -> V1 필수
Phase 6  Pilot hardening           -> V1 release gate

Phase 7  Settlement                -> Post-V1
Phase 8  Multi-host productization -> Post-V1 / 판매 준비
Phase 9  Broader StayOps           -> Post-V1
```

따라서 **V1 = Phase 0~6**이다.

---

## 13. V1 Release Gate

V1은 코드가 존재한다고 완료가 아니다.

다음을 만족해야 내부 실사용 V1로 간주한다.

### End-to-end

- 실제 Guest mobile form 제출
- 실제 DB 저장
- 실제 Guest WhatsApp 접수 확인
- 실제 Driver가 Kakao/one-click 전달을 통해 배차 요청 수신
- 실제 Driver가 signed page에서 응답
- Assignment가 Host 재입력 없이 저장
- 실제 Guest WhatsApp으로 차량/ETA 안내 도착

### Host operations

- Host가 정상 요청에서 데이터를 복사/재작성하지 않아도 됨
- driver reject/timeout을 재배차 가능
- 일정 변경/취소 가능
- WhatsApp failure를 발견하고 복구 가능
- Guest free-text에 WhatsApp Business App에서 수동 답장 가능

### Reliability

- duplicate Guest submit이 중복 예약을 만들지 않음
- expired/duplicate Driver response가 assignment를 중복 생성하지 않음
- notification retry가 동일 운영 메시지를 무제한 중복 발송하지 않음
- audit trail로 주요 변경 원인을 확인할 수 있음

### Pilot

목표:

- 실제 운영 transfer 약 20건 이상을 V1 후보로 처리
- P0/P1급 미해결 운영 장애가 없음
- 정상 transfer의 대부분이 Host의 복사/재입력 없이 완료
- 남은 수동 작업은 "의사결정/예외 대응"인지 "불필요한 중계"인지 구분해 기록

---

## 14. V1 이후

V1이 실제 운영에서 안정화되면 다음 순서로 확장한다.

```text
V1 internal-use proven
-> Settlement / operating tools
-> Multi-host onboarding / productization
-> External host pilot
-> Sales
-> iCal / cleaning / laundry / broader StayOps
```

판매 준비는 V1과 같은 프로젝트가 아니라 **검증된 내부 운영 제품을 범용화하는 다음 단계**로 취급한다.
