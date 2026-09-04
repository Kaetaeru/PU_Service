# 03. HTML ↔ PU_Service API 계약 초안

> **이전 근거 문서.** 독립 HTTP API를 전제로 한 초안이다. 현재 구현 계약은 `06_IMPLEMENTATION_READINESS.md`, 입력 항목은 `00_CANONICAL_SYSTEM.md` §5, 요금은 `08_요금표.md`가 우선한다.
> 이 문서의 `port_code` + `direction` 조합은 `origin_code` / `destination_code` 쌍으로 대체됐고, 손님 연락처·언어·opt-in·수하물·터미널 필드가 이후 추가됐다.
> idempotency 규칙과 오류 응답 계약은 여전히 유효하다.

## 1. Create transfer request

### Request

```http
POST /v1/transfer-requests
Content-Type: application/json
Idempotency-Key: <client-generated-uuid>
```

### Body

```json
{
  "source": "stay-pachira-transfer-html",
  "direction": "pickup",
  "guest": {
    "name": "Julia Hussmann",
    "passenger_count": 2
  },
  "schedule": {
    "date": "2026-08-21",
    "time": "15:40"
  },
  "route": {
    "port_code": "ICN",
    "stay_code": "PH1"
  },
  "flight": {
    "number": "KE938"
  },
  "addons": {
    "child_seat_count": 1,
    "terminal_transfer": false
  },
  "note": "6 suitcases"
}
```

## 2. HTML field mapping

| HTML | API | 비고 |
|---|---|---|
| `dir` JS state | `direction` | `pickup` / `dropoff` |
| `g-name` | `guest.name` | 필수 |
| `g-pax` | `guest.passenger_count` | 1~7 |
| `g-date` | `schedule.date` | 필수 |
| `g-time` | `schedule.time` | 선택 |
| `g-air` | `route.port_code` | ICN/GMP/SEOULSTN |
| `g-stay` | `route.stay_code` | PH1/PH2/PH3/SP5/SP6/WSR |
| `g-flight` | `flight.number` | 선택, 서버에서 trim/uppercase normalize |
| `o-seat` + `o-seat-n` | `addons.child_seat_count` | 미선택이면 0, 선택 시 1~4 |
| `o-term` | `addons.terminal_transfer` | boolean |
| `g-note` | `note` | 선택 |

## 3. 시간 의미

클라이언트가 별도 의미 값을 임의로 보내지 않고 서버가 `direction`으로 의미를 결정하는 방식을 권장한다.

```text
pickup  -> schedule.time = flight arrival/landing time entered by guest
dropoff -> schedule.time = requested accommodation pickup time
```

서버 normalized response에서는 다음처럼 명시할 수 있다.

```json
{
  "schedule": {
    "date": "2026-08-21",
    "time": "15:40",
    "time_semantics": "flight_arrival",
    "timezone": "Asia/Seoul"
  }
}
```

## 4. Validation 초안

### 반드시 서버에서 검증

- `direction`: `pickup | dropoff`
- `guest.name`: 공백 제거 후 non-empty
- `guest.passenger_count`: integer 1~7
- `schedule.date`: 유효한 date, 필수
- `schedule.time`: 제공된 경우 유효한 time
- `route.port_code`: `ICN | GMP | SEOULSTN`
- `route.stay_code`: `PH1 | PH2 | PH3 | SP5 | SP6 | WSR`
- `addons.child_seat_count`: integer 0~4
- `addons.terminal_transfer`: boolean
- `SEOULSTN`일 때 `terminal_transfer=true`는 거부하거나 서버에서 false로 강제하지 말고 명시적인 validation error를 반환

### 아직 정책 결정이 필요한 validation

- guest name 최대 길이
- note 최대 길이
- 과거 날짜 허용 여부
- pickup에서 flight number/time을 필수로 만들지 여부
- dropoff에서 flight number가 의미 있는지 여부
- 출발 최소 사전 예약 시간
- 7명 초과 요청을 별도 문의로 처리할지 완전 차단할지

이 정책은 현재 HTML에서 확정되어 있지 않으므로 서버 구현 전에 결정한다.

## 5. Fare calculation

요청 body에 최종 요금을 받지 않는다.

서버는 다음 값만으로 요금을 계산한다.

```text
direction
port_code
stay_code -> area
passenger_count
child_seat_count
terminal_transfer
```

### Response fare 예시

```json
{
  "currency": "KRW",
  "base": 90000,
  "items": [
    {
      "code": "child_seat",
      "quantity": 1,
      "unit_amount": 10000,
      "amount": 10000
    }
  ],
  "total": 100000
}
```

클라이언트의 현재 fare 계산은 화면 estimate일 뿐 최종 금액의 source of truth가 아니다.

## 6. Success response

### HTTP 201

```json
{
  "request_id": "tr_01J...",
  "status": "requested",
  "direction": "pickup",
  "guest": {
    "name": "Julia Hussmann",
    "passenger_count": 2
  },
  "schedule": {
    "date": "2026-08-21",
    "time": "15:40",
    "time_semantics": "flight_arrival",
    "timezone": "Asia/Seoul"
  },
  "route": {
    "port_code": "ICN",
    "port_name": "Incheon Airport",
    "stay_code": "PH1",
    "stay_name": "Pachira House 1F"
  },
  "flight": {
    "number": "KE938"
  },
  "addons": {
    "child_seat_count": 1,
    "terminal_transfer": false
  },
  "fare": {
    "currency": "KRW",
    "base": 90000,
    "items": [
      {
        "code": "child_seat",
        "quantity": 1,
        "unit_amount": 10000,
        "amount": 10000
      }
    ],
    "total": 100000
  },
  "driver_message": "...",
  "created_at": "2026-08-19T05:55:00Z"
}
```

`request_id` 형식은 예시일 뿐 아직 구현 선택 사항이다. 중요한 것은 client-visible stable unique identifier가 있다는 점이다.

## 7. Validation error response

### HTTP 422

```json
{
  "error": {
    "code": "validation_error",
    "message": "Request contains invalid fields",
    "fields": {
      "guest.passenger_count": "must be between 1 and 7"
    }
  }
}
```

HTML은 전체 toast 하나만 띄우는 것보다 field별 error를 연결할 수 있도록 설계하는 것이 좋다.

## 8. Idempotency behavior

동일한 `Idempotency-Key`로 동일 요청이 반복되면 새 예약을 만들지 않고 최초 결과를 반환한다.

권장 규칙:

- 같은 key + 같은 payload → 최초 성공 response 재사용
- 같은 key + 다른 payload → `409 idempotency_conflict`
- timeout 후 retry → 반드시 기존 key 재사용

## 9. Request 조회

```http
GET /v1/transfer-requests/{request_id}
```

### Response 최소 항목

- `request_id`
- current normalized `status`
- normalized form data
- server fare
- created/updated timestamp
- provider reference가 생긴 경우 provider correlation 정보(고객에게 노출 가능한 범위만)

## 10. 향후 API — 지금은 구현 대상 아님

provider/운영 정책이 확정된 뒤 아래를 검토한다.

```text
POST /v1/transfer-requests/{id}/cancel
PATCH /v1/transfer-requests/{id}
POST /internal/transfer-requests/{id}/confirm
POST /internal/provider-events/...
```

현재 HTML 근거만으로 cancellation/change/provider callback 계약을 확정하지 않는다.

## 11. Frontend submit pseudo-flow

```text
on submit:
  validate basic required fields
  disable submit button
  build payload from current form
  reuse/create idempotency key
  POST to PU_Service

  if 201:
    render request_id
    render server fare
    render normalized route/schedule
    use server driver_message for share/copy

  if 422:
    show field errors

  if network/5xx:
    keep same idempotency key
    allow safe retry

  finally:
    enable submit when appropriate
```

## 12. 확정이 필요한 외부 좌표

HTML → PU_Service 계약은 위 초안으로 진행할 수 있지만 실제 연결 전에 아래 두 값은 필요하다.

1. HTML이 실제 배포되는 origin/domain
2. `PU_Service`가 배포될 API base URL 또는 same-origin reverse proxy 경로

외부 기사/운송 provider는 HTML 접수 API 구현과 분리해서 나중에 확정할 수 있다.
