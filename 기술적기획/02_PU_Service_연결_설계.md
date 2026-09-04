# 02. PU_Service 연결 설계

> **이전 근거 문서.** 독립 `PU_Service` HTTP API를 전제로 쓰였으나, 현재 구현 계약은 Supabase Edge Functions 기반이다. `06_IMPLEMENTATION_READINESS.md`가 우선한다.
> validation·server fare 권위·idempotency·public browser 보안 경계에 관한 논지는 여전히 유효하며 현재 문서들에 승계됐다.

---

## 1. 목표

기존 `TalkFile_pickup.html`의 UI/UX를 유지하면서 `Request this transfer`를 실제 서버 예약 접수로 전환한다.

이번 설계의 핵심은 **HTML → PU_Service** 계약이다. 실제 기사/운송 provider API는 아직 제공되지 않았으므로 provider 연결은 adapter 경계로 분리한다.

## 2. 권장 전체 구조

```text
[Stay Pachira transfer HTML]
        |
        | POST /v1/transfer-requests
        v
[PU_Service HTTP API]
        |
        +--> Request validation
        +--> Fare policy
        +--> Transfer domain/service
        +--> Persistence
        +--> Audit / lifecycle events
        |
        +--> [Provider Adapter] ----> [운송/기사 시스템 - 추후 확정]
        |
        +--> [Operator/notification boundary - 추후 확정]
```

## 3. 책임 분리

### HTML이 계속 담당할 것

- pickup/drop-off 선택
- 고객 입력 UI
- 숙소/공항 선택 UI
- 카시트/터미널 경유 입력
- 즉시 예상 요금 표시
- 서버 제출 진행/성공/실패 UI
- 서버가 반환한 요청 번호와 상태 표시
- 필요하다면 기사 전달 문구 share/copy UI

### HTML에서 제거하거나 비권위화할 것

- 최종 요금 결정
- 실제 예약 생성 여부 판단
- 예약 상태의 source of truth
- provider credential
- 서버 secret/API key
- 중복 제출 판단

### PU_Service가 담당할 것

- request payload validation
- 숙소/거점 코드 validation
- 인원/카시트 범위 validation
- 방향별 schedule 의미 validation
- 서버측 fare calculation
- stable `request_id` 생성
- idempotency/중복 제출 보호
- 요청 및 lifecycle persistence
- normalized status 관리
- 기사/운송 업체 adapter 호출 경계
- correlation/audit log
- 필요 시 기사 전달용 메시지 생성

## 4. 1차 구현에서 권장하는 최소 경계

외부 운송 provider가 확정되기 전에도 아래 범위는 구현 가능하다.

### Endpoint

`POST /v1/transfer-requests`

역할:

1. HTML 요청을 접수한다.
2. validation 한다.
3. 서버 요금을 계산한다.
4. 요청을 저장한다.
5. `request_id`, normalized request, fare, current status를 반환한다.

이 단계에서는 status를 `requested`로 시작할 수 있다. 이후 provider adapter가 붙으면 `confirmed`, `dispatched` 등 내부 lifecycle로 확장한다.

### 조회 Endpoint

`GET /v1/transfer-requests/{request_id}`

HTML에서 즉시 필요하지 않더라도 운영/재조회와 추후 상태 페이지를 위해 일찍 계약을 잡아두는 편이 좋다.

## 5. Submit 흐름 변경

### 현재

```text
click submit
 -> name/date만 검사
 -> browser에서 fare 계산
 -> browser에서 ticket 생성
 -> browser에서 driver message 생성
 -> share/copy
```

### 변경 후

```text
click submit
 -> client 기본 validation
 -> client idempotency key 생성/유지
 -> POST /v1/transfer-requests
 -> PU_Service 전체 validation
 -> PU_Service fare 재계산
 -> request 저장
 -> response(request_id/status/fare/normalized data)
 -> HTML 결과 화면 표시
 -> 서버 응답 기반 share/copy
```

## 6. 요금 계산 전략

### 단기

HTML의 기존 계산 로직은 화면 estimate 용도로 유지한다.

서버에는 동일한 요금 정책을 별도로 구현하고 서버 결과를 최종값으로 사용한다.

응답 예:

```json
{
  "fare": {
    "currency": "KRW",
    "base": 90000,
    "addons": 10000,
    "total": 100000
  }
}
```

HTML estimate와 서버 total이 다르면 결과 화면은 서버 값을 표시한다.

### 중기

요금표를 HTML 상수에서 제거하고 `PU_Service`를 단일 source of truth로 만든다.

선택지는 두 가지다.

1. 폼 변경 시 quote API를 호출한다.
2. 초기 페이지 로드 시 fare/master config를 서버에서 받아 client가 estimate를 계산한다.

1차 연결에서는 복잡도를 줄이기 위해 기존 client estimate + submit 시 server authoritative calculation을 권장한다.

## 7. 숙소/거점 master 전략

현재 HTML의 코드값은 연결 contract로 활용하기 좋다.

### Port

- `ICN`
- `GMP`
- `SEOULSTN`

### Stay

- `PH1`
- `PH2`
- `PH3`
- `SP5`
- `SP6`
- `WSR`

서버는 표시 문자열이나 주소를 클라이언트가 보내는 값으로 신뢰하지 않고, code를 받아 서버 master에서 해석하는 구조를 권장한다.

특히 `WSR`은 HTML에 상세 주소가 없으므로 실제 배차 전에 주소 master를 확정해야 한다.

## 8. 시간 모델

단순한 `datetime` 필드만 사용하면 현재 UI 의미를 잃는다.

권장 모델:

```text
direction = pickup | dropoff
service_date
service_time (nullable)
time_semantics = flight_arrival | accommodation_pickup
business_timezone = Asia/Seoul
```

규칙:

- pickup → `flight_arrival`
- dropoff → `accommodation_pickup`

날짜/시간을 server datetime으로 변환할 때는 서울 기준 business timezone을 명시하고, 향후 flight provider가 붙더라도 원래 고객 입력값을 보존한다.

## 9. Idempotency

HTML의 submit을 API로 바꾸면 네트워크 timeout이나 double click 때문에 중복 예약이 생길 수 있다.

권장:

- HTML이 폼 인스턴스별 UUID를 생성한다.
- `Idempotency-Key` header로 전송한다.
- PU_Service는 동일 key + 동일 logical request의 반복 요청에 같은 결과를 반환한다.
- provider dispatch가 붙은 뒤에도 internal `request_id`와 provider job id를 1:1 또는 명시적인 dispatch-attempt 관계로 기록한다.

브라우저가 요청 결과를 못 받았다고 해서 create를 무조건 다시 provider에 보내면 안 된다.

## 10. Public browser API의 인증/보호

현재 페이지는 고객이 직접 입력하는 성격이므로 브라우저 코드 안에 장기 secret을 넣으면 안 된다.

### 권장 기본선

- HTTPS only
- strict JSON validation
- rate limiting
- request/body size limit
- known web origin에 대한 CORS 제한(다른 origin일 경우)
- server-side logging에서 guest/name/note를 불필요하게 노출하지 않기

### 배포 선택

#### A. 같은 origin/reverse proxy — 권장

```text
https://stay.example/transfer
https://stay.example/api/transfer-requests -> PU_Service
```

장점: CORS가 단순하고 배포/보안 경계가 명확하다.

#### B. 별도 API origin

```text
HTML: https://stay.example
API:  https://pickup-api.example
```

이 경우 `PU_Service`에 명시적인 `ALLOWED_ORIGINS` 설정이 필요하다.

실제 HTML hosting URL은 아직 제공되지 않았으므로 어느 방식을 쓸지는 배포 단계에서 확정한다.

## 11. 상태 모델

HTML이 현재 보장하는 상태는 사실상 화면 문구 `requested` 하나뿐이다.

서버 내부 normalized lifecycle 초안:

```text
requested
  -> confirmed
  -> dispatched
  -> completed

requested/confirmed/dispatched
  -> cancelled

requested/confirmed/dispatched
  -> failed
```

`confirmed`와 `dispatched`의 실제 의미는 기사/provider 운영 방식이 정해진 뒤 확정한다. provider 고유 status는 별도 필드로 보존하고 무리하게 1:1 enum으로 맞추지 않는다.

## 12. 기사 전달 메시지 처리

현재 HTML은 브라우저에서 `lastMsg`를 만든다.

1차 연결에서는 두 방식이 가능하다.

### 권장

서버가 normalized request와 fare를 바탕으로 `driver_message`를 생성해 응답한다.

장점:

- 운영 메시지 형식이 한 곳에서 관리된다.
- client와 운영 데이터가 불일치하지 않는다.
- 추후 Kakao/문자/provider adapter가 붙을 때 재사용 가능하다.

HTML의 `navigator.share`/copy는 이 서버 메시지를 공유하는 UI로 남긴다.

## 13. 오류 UI 계약

HTML은 API 실패 시 결과 화면으로 넘어가면 안 된다.

최소 분류:

- `400/422`: 필드/business validation 오류 → 해당 입력 수정 안내
- `409`: idempotency/request conflict → 기존 request 결과 표시 가능
- `429`: 과도한 요청 → 잠시 후 재시도
- `5xx/network`: 예약 성공 여부가 불명확할 수 있음 → 동일 idempotency key로 상태 확인/재시도

특히 timeout 시 새 idempotency key를 만들어 다시 제출하지 않는다.

## 14. 단계별 연결 계획

### Phase 1 — Intake contract

- `POST /v1/transfer-requests`
- validation
- fare policy
- request persistence
- idempotency
- response contract

### Phase 2 — HTML wiring

- local form values → API payload adapter
- submit loading/disable
- server validation 오류 표시
- response 기반 ticket/message 표시
- request_id 노출

### Phase 3 — Operations

- request 조회
- 운영자 확인/상태 변경 방식
- driver notification 경계
- cancellation/change flow

### Phase 4 — Provider integration

- provider API/operating process 확정
- adapter 구현
- auth/secrets
- outbound dispatch
- callback/polling
- reconciliation/retry

## 15. 현재 권장 결론

가장 먼저 구현할 연결점은 **HTML의 submit handler → `PU_Service POST /v1/transfer-requests`** 이다.

이때 기존 HTML의 UI, 숙소/공항 code, client estimate는 최대한 유지하되 최종 validation·fare·request identity·persistence는 서버로 옮긴다.

외부 운송 업체가 아직 확정되지 않았다는 이유로 이 접수 계층까지 미룰 필요는 없다. provider adapter를 분리하면 HTML 접수 시스템과 provider 연결을 독립적으로 개발할 수 있다.
