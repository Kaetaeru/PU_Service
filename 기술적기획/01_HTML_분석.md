# 01. TalkFile_pickup.html 기술 분석

## 1. 페이지 역할

페이지 제목은 `Stay Pachira · Airport Transfer Request`이며 고객이 공항/서울역과 숙소 사이의 픽업 또는 드랍오프를 요청하는 모바일 중심 단일 페이지다.

현재 구조는 HTML + CSS + 즉시 실행 JavaScript 한 파일로 완결되어 있다. 별도 프레임워크, 서버 호출, form action, fetch/XHR, 데이터베이스 연결은 없다.

## 2. 사용자가 입력하는 데이터

### 이동 방향

- `pickup`: AIRPORT → STAY
- `drop`: STAY → AIRPORT

방향에 따라 같은 날짜/시간 필드의 의미가 달라진다.

- pickup: 날짜 = Arrival date, 시간 = 항공편 착륙 시각
- drop: 날짜 = Departure date, 시간 = 운전자가 숙소에서 픽업해야 하는 시각

따라서 서버 모델에서 단순 `datetime` 하나로만 받기보다 방향과 시간 의미를 함께 보존해야 한다.

### 고객/운행 입력

| HTML ID | 의미 | 현재 UI 조건 |
|---|---|---|
| `g-name` | 대표 게스트 영문명 | 제출 JavaScript에서 필수 검증 |
| `g-date` | 도착일/출발일 | 제출 JavaScript에서 필수 검증 |
| `g-time` | 착륙 시각/숙소 픽업 시각 | 선택 |
| `g-flight` | 항공편 번호 | 선택, 대문자 변환 |
| `g-pax` | 인원 수 | 기본 2, HTML min 1/max 7 |
| `g-air` | 공항/서울역 코드 | ICN/GMP/SEOULSTN |
| `g-stay` | 숙소 코드 | PH1/PH2/PH3/SP5/SP6/WSR |
| `o-seat` | 카시트 사용 여부 | 선택 |
| `o-seat-n` | 카시트 수 | 기본 1, HTML min 1/max 4 |
| `o-term` | 인천/김포 터미널 T1↔T2 경유 | 서울역 선택 시 숨김 |
| `g-note` | 특이사항 | 선택 |

## 3. HTML 내부의 업무 마스터 데이터

### 숙소 코드

- `PH1`: Pachira House 1F / 이태원 / 이태원로16길 45
- `PH2`: Pachira House 2F / 이태원 / 이태원로16길 45
- `PH3`: Pachira House 3F / 이태원 / 이태원로16길 45
- `SP5`: Stay Pachira 5F / 이태원 / 보광로59길 36
- `SP6`: Stay Pachira 6F / 이태원 / 보광로59길 36
- `WSR`: Wausan-ro / 홍대 / HTML 상 상세 주소 공란

### 출도착 거점 코드

- `ICN`: Incheon Airport / 인천공항
- `GMP`: Gimpo Airport / 김포공항
- `SEOULSTN`: Seoul Station / 서울역

이 코드들은 HTML과 서버의 계약 키로 재사용할 수 있다. 다만 장기적으로 표시명·주소·지역은 서버측 master/config에서 관리하는 편이 안전하다.

## 4. 현재 요금 규칙

모든 금액은 KRW다.

### ICN ↔ 이태원

- pickup 1~4명: 90,000
- pickup 5~7명: 100,000
- drop 1~4명: 80,000
- drop 5~7명: 90,000

### ICN ↔ 홍대

- pickup 1~4명: 85,000
- pickup 5~7명: 90,000
- drop 1~4명: 75,000
- drop 5~7명: 80,000

### GMP

- 이태원/홍대, pickup/drop, 1~7명 모두 80,000

### SEOULSTN

- 이태원/홍대, pickup/drop, 1~7명 모두 50,000

### Add-on

- 카시트: 1개당 10,000
- T1 ↔ T2 경유: 20,000
- 서울역에서는 터미널 경유 옵션 비활성

## 5. 현재 제출 처리

`Request this transfer` 버튼 클릭 시 다음 일만 브라우저 안에서 실행된다.

1. 대표 게스트명과 날짜가 비었는지 확인한다.
2. 현재 폼 값을 읽는다.
3. HTML 내 `FARE` 상수로 기본 요금을 계산한다.
4. 카시트/터미널 경유 비용을 더한다.
5. 결과 ticket HTML을 만든다.
6. 기사 전달용 한국어 문자열 `lastMsg`를 만든다.
7. 입력 폼을 숨기고 결과 화면을 보여준다.

즉, 버튼 문구는 "Request"이지만 실제로 어떤 서버나 기사 시스템에도 예약이 생성되지 않는다.

## 6. 공유 처리

`Send via KakaoTalk` 버튼은 Kakao SDK/API를 호출하지 않는다.

- `navigator.share`가 있으면 OS/browser share sheet를 연다.
- 없으면 텍스트를 clipboard에 복사한다.

따라서 현재 KakaoTalk은 전달 채널의 UI 표현일 뿐 시스템 통합점이 아니다.

## 7. PU_Service 연결에 필요한 핵심 변경

가장 중요한 변경점은 현재 `submit` handler 중 "로컬 결과 생성" 앞에 서버 접수를 추가하는 것이다.

권장 흐름:

1. HTML 입력 → 정규화된 JSON payload 생성
2. `POST /v1/transfer-requests`
3. `PU_Service`가 필드 검증
4. `PU_Service`가 서버측 요금 재계산
5. 요청 저장 + `request_id` 발급
6. 서버 결과를 HTML에 표시
7. 기사 전달 메시지도 가능하면 서버 응답 기준으로 생성

## 8. 현재 상태에서 발견되는 기술적 주의점

### 클라이언트 요금을 신뢰하면 안 됨

요금표가 JavaScript에 있기 때문에 사용자가 개발자 도구로 값을 변경할 수 있다. 브라우저 요금은 UX용 estimate로 유지할 수 있지만 서버가 같은 규칙으로 반드시 재검증해야 한다.

### HTML min/max만으로는 서버 검증이 되지 않음

`g-pax`는 min 1/max 7, 카시트는 min 1/max 4이지만 JavaScript submit 단계에서 범위를 명시적으로 거부하지 않는다. 조작된 DOM/직접 API 호출까지 고려해 `PU_Service`에서 범위를 검증해야 한다.

### 현재 요청 식별자가 없음

중복 클릭, 네트워크 재시도, 고객 재제출을 구분할 수 있는 ID가 없다. 서버에서 `request_id`를 만들고 클라이언트 idempotency 키를 받을 구조가 필요하다.

### 현재 영속 상태가 없음

브라우저 새로고침 후 요청을 복구하거나 운영자가 조회할 수 없다. 예약 접수 서비스라면 서버측 저장이 필요하다.

### `WSR` 상세 주소가 비어 있음

현 HTML은 홍대 `WSR`에 상세 주소 문자열이 없다. 실제 기사 배차에 정확한 주소가 필요하다면 master data 확정이 필요하다.

### 시간 의미가 방향별로 다름

pickup의 time은 landing time이고 drop의 time은 실제 vehicle pickup time이다. 서버에서 같은 필드를 받아도 `direction`에 따른 의미 검증이 필요하다.

## 9. 분석 결론

이 HTML은 새로운 프론트엔드를 다시 만드는 대상이 아니라, 이미 완성된 **고객 입력 UI + 견적 UI**로 보는 것이 적합하다.

`PU_Service`의 첫 역할은 이 페이지 뒤에 서버측 예약 접수 계층을 붙여서 다음을 보장하는 것이다.

- 신뢰 가능한 validation
- 서버 authoritative fare
- request identity
- persistence
- lifecycle status
- 중복 방지
- 이후 기사/운송 provider 연결을 위한 adapter boundary
