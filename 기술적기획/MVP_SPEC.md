# StayOps MVP — 개발 명세서

**범위**: 픽업 / 드랍오프 (공항 이동 서비스)
**작성일**: 2026-08
**문서 목적**: 개발 착수를 위한 기능·데이터·화면 명세

> 배경과 전체 구상은 `REQUIREMENTS.md` 참조. 이 문서는 MVP 1차 개발 범위만 다룬다.

---

## 1. 개요

### 1.1 해결하려는 문제

숙소 호스트가 게스트와 픽업 기사 사이에서 **수동 중계자** 역할을 하고 있다.

```
현재: 게스트 메시지 → 호스트가 읽음 → 요금 계산 → 기사용으로 재작성
      → 카톡 전송 → 배차정보 회신 → 게스트 언어로 번역 → 전송
      → 엑셀에 수기 기록 → 월말 정산

목표: 게스트 폼 입력 → 자동 저장 → 기사 메시지 자동 생성 → 전송
      → 배차정보 입력 → 게스트 안내 자동 생성 → 월별 정산 자동 집계
```

### 1.2 사용자 유형

| 유형 | 로그인 | 사용 화면 |
|---|---|---|
| 게스트 | 불필요 (링크 접속) | 예약 요청 폼 |
| 호스트 | 필요 | 대시보드·예약관리·정산 |
| 기사 | 불필요 (MVP) | 카톡/문자로 메시지 수신만 |

### 1.3 MVP 제외 범위

- 청소·세탁 파트너 (2차)
- iCal 자동 연동 (2차)
- 카카오 알림톡 자동 발송 (2차 — MVP는 원클릭 공유)
- 멀티 호스트 계정 (3차 — 단, 데이터 구조는 대비해서 설계)

---

## 2. 화면 명세

### 2.1 게스트 예약 요청 폼 `/request`

공개 페이지. 로그인 불필요. 모바일 우선.

**입력 항목**

| 필드 | 타입 | 필수 | 비고 |
|---|---|---|---|
| 이동 방향 | 라디오 | ✅ | 픽업 / 드랍오프 |
| 대표 게스트명 | text | ✅ | 영문. 기사 안내판용 |
| 날짜 | date | ✅ | |
| 시간 | time | | 픽업=착륙시각 / 드랍오프=숙소출발시각 |
| 항공편명 | text | | 자동 대문자 변환 |
| 인원 | number | ✅ | 1~7 |
| 공항/거점 | select | ✅ | ICN / GMP / 서울역 |
| 숙소(층) | select | ✅ | 아래 코드표 참조 |
| 카시트 | checkbox + 수량 | | 1~4개 |
| 터미널 경유 | checkbox | | 서울역 선택 시 숨김 |
| 특이사항 | textarea | | 아동 나이·캐리어 수 등 |

**동작**

- 입력 변경 시 요금 실시간 재계산 및 표시 (기본요금 + 옵션 항목별 분해)
- 제출 → DB 저장 → 확인 화면 전환
- 확인 화면에 기사 전달용 메시지 표시 + 복사/공유 버튼
- 다국어: EN(기본) / KO / CN — `?lang=en` 파라미터 또는 브라우저 언어 감지

**참고**: 현행 프로토타입 `pickup.html` 의 UI/UX를 기준으로 함. 요금 계산 로직도 동일.

---

### 2.2 호스트 대시보드 `/admin`

로그인 필요.

**상단 브리핑 카드**

- 오늘 / 내일 픽업·드랍오프 건수
- 가장 빠른 일정 (시각 + 게스트명)
- 처리 필요 알림 (아래 "경고 규칙" 참조)

**경고 규칙** (자동 판정)

| 경고 | 조건 |
|---|---|
| 일정 중복 | 같은 날 · 같은 공항 · 시각 차 60분 이내 2건 이상 |
| 정원 초과 우려 | 인원 7명 이상 또는 특이사항에 캐리어 5개 이상 언급 |
| 배차 미완료 | 이동일 D-1인데 `dispatch_status != '배차완료'` |
| 정보 누락 | 시간·지불방식·층 중 하나라도 비어 있음 |

**예약 목록**

- 기본 정렬: 날짜 오름차순, 오늘 이후 우선
- 필터: 기간 / 구분(픽업·드랍) / 배차상태 / 입금상태
- 각 행에서 바로 수정 가능해야 함 (특히 시간 — 항공편 지연 대응)

---

### 2.3 예약 상세 `/admin/booking/{id}`

**표시**: 게스트 입력 전체 + 자동 계산 요금

**호스트 입력 항목**

| 필드 | 설명 |
|---|---|
| 지불 방식 | 호스트전액 / 게스트전액 / 분담 |
| 게스트 지불액 | 분담 시 입력. 호스트부담 = 총액 − 게스트지불 |
| 차량번호 | 배차 확정 시 |
| 기사 연락처 | 배차 확정 시 |
| 배차 상태 | 배차대기 / 배차완료 / 안내완료 |
| 입금 상태 | 미입금 / 입금예정 / 입금완료 |
| 메모 | 내부용 |

**액션 버튼**

1. **기사 메시지 생성** → 아래 템플릿으로 생성 → 복사 / 카톡공유
2. **게스트 안내 생성** → 배차정보 + 공항 미팅 안내 → 언어 선택(EN/KO) → 복사 / 공유
3. **구글 캘린더 등록** → 일정 등록 링크 생성

---

### 2.4 정산 `/admin/settlement`

**월별 집계 표**

| 컬럼 |
|---|
| 월 / 픽업건수 / 드랍건수 / 공급가 / 세금 / 세금포함총액 / 게스트지불 / 호스트부담 |

**입금 현황**

- 입금완료 / 입금예정 / 미입금 각각 건수·금액
- "이번 달 지급할 금액" 강조 표시

**집계 축**

- 월별 / 숙소별 / 층별 / 노선별

**내보내기**

- Excel(.xlsx) 다운로드 — 현행 장부와 동일 컬럼 구조
- 세무 신고용 TSV 복사

---

## 3. 데이터 모델

### 3.1 `bookings`

```
id                  PK
host_id             FK → hosts (MVP는 단일값, 확장 대비)
direction           ENUM('pickup','dropoff')       -- 픽업/드랍오프
guest_name          VARCHAR(100)                    -- 영문 대표게스트명
travel_date         DATE
travel_time         TIME NULL                       -- 픽업=착륙, 드랍=숙소출발
flight_no           VARCHAR(20) NULL
pax                 TINYINT
port_code           ENUM('ICN','GMP','SEOULSTN')
stay_code           VARCHAR(10)                     -- PH1,PH2,PH3,SP5,SP6,WSR
car_seat_qty        TINYINT DEFAULT 0
terminal_transfer   BOOLEAN DEFAULT 0
guest_note          TEXT NULL

-- 금액 (입력 시점 스냅샷으로 저장. 요금표 변경돼도 과거 건 불변)
base_fare           INT                             -- 기본요금
option_fare         INT                             -- 카시트+경유
supply_amount       INT                             -- base + option (공급가)
tax_amount          INT                             -- supply * 0.1
total_amount        INT                             -- supply + tax
guest_paid          INT DEFAULT 0
host_amount         INT                             -- total - guest_paid

payment_type        ENUM('host','guest','split')
payment_status      ENUM('unpaid','pending','paid') DEFAULT 'unpaid'

vehicle_no          VARCHAR(30) NULL
driver_phone        VARCHAR(30) NULL
dispatch_status     ENUM('waiting','assigned','notified') DEFAULT 'waiting'

admin_memo          TEXT NULL
created_at          DATETIME
updated_at          DATETIME
```

> **중요**: 금액 필드는 계산값이 아니라 **저장값**이어야 한다. 요금표가 바뀌어도 과거 예약의 정산 금액이 변하면 안 된다.

### 3.2 `stays` (숙소 마스터)

```
stay_code    PK      -- PH1, PH2, PH3, SP5, SP6, WSR
host_id      FK
name_ko              -- 파키라하우스 3층 (이연공방 앞)
name_en              -- Pachira House 3F
address_ko           -- 이태원로16길 45
area_code            -- ITAEWON / HONGDAE  (요금 구간 결정용)
is_active
```

### 3.3 `fare_rules` (요금표)

**하드코딩 금지.** 관리자 화면에서 수정 가능해야 함.

```
id           PK
host_id      FK
port_code            -- ICN, GMP, SEOULSTN
area_code            -- ITAEWON, HONGDAE
direction            -- pickup / dropoff
pax_min      TINYINT
pax_max      TINYINT
fare         INT      -- 공급가 (세금 별도)
```

### 3.4 `option_prices`

```
option_code  PK   -- CAR_SEAT, TERMINAL_TRANSFER
host_id      FK
price        INT
unit              -- per_item / per_booking
```

---

## 4. 비즈니스 규칙

### 4.1 요금 계산

```
base_fare = fare_rules 조회 (port_code, area_code, direction, pax 구간 매칭)
option_fare = (car_seat_qty × CAR_SEAT 단가)
            + (terminal_transfer ? TERMINAL_TRANSFER 단가 : 0)
supply_amount = base_fare + option_fare
tax_amount    = ROUND(supply_amount × 0.1)
total_amount  = supply_amount + tax_amount
host_amount   = total_amount − guest_paid
```

**제약**

- 터미널 경유는 `port_code IN ('ICN','GMP')` 일 때만 적용. 서울역은 옵션 자체를 숨김
- `pax` 범위를 벗어나면 가장 가까운 상위 구간 적용
- 요금 규칙이 없는 조합은 **0원 반환이 아니라 에러 표시** (설정 누락을 감춰선 안 됨)

### 4.2 현행 요금표 (초기 데이터)

단위: 원, 세금 별도

| port | area | direction | 1~4명 | 5~7명 |
|---|---|---|---|---|
| ICN | ITAEWON | pickup | 90,000 | 100,000 |
| ICN | ITAEWON | dropoff | 80,000 | 90,000 |
| ICN | HONGDAE | pickup | 85,000 | 90,000 |
| ICN | HONGDAE | dropoff | 75,000 | 80,000 |
| GMP | ITAEWON | pickup | 80,000 | 80,000 |
| GMP | ITAEWON | dropoff | 80,000 | 80,000 |
| SEOULSTN | ITAEWON | pickup | 50,000 | 50,000 |

**옵션**: 카시트 10,000원/개 · 터미널 경유(T1⇄T2) 20,000원/건

### 4.3 숙소 초기 데이터

| stay_code | name_ko | address | area |
|---|---|---|---|
| PH1 | 파키라하우스 1층 (이연공방 앞) | 이태원로16길 45 | ITAEWON |
| PH2 | 파키라하우스 2층 (이연공방 앞) | 이태원로16길 45 | ITAEWON |
| PH3 | 파키라하우스 3층 (이연공방 앞) | 이태원로16길 45 | ITAEWON |
| SP5 | 스테이파키라 5층 (CU건물) | 보광로59길 36 | ITAEWON |
| SP6 | 스테이파키라 6층 (CU건물) | 보광로59길 36 | ITAEWON |
| WSR | 와우산로 (홍대) | — | HONGDAE |

---

## 5. 메시지 템플릿

DB 또는 설정 파일로 관리. 하드코딩 금지.

### 5.1 기사 전달용 (한국어)

```
🚐 {픽업|드랍오프} 예약 안내
━━━━━━━━━━━━━
{도착일시|출발일시}: {M/D(요일) HH:MM}
항공편: {flight_no}
인원: {pax}명
대표게스트: {guest_name}
경로: {port_ko} → {area_ko}
{도착지|출발지}: {stay.name_ko}, {stay.address_ko}
⚠️ 카시트 {n}개 필요           ← car_seat_qty > 0 일 때만
경유: T1 ⇄ T2 (+20,000원)      ← terminal_transfer 일 때만
특이사항: {guest_note}          ← 값 있을 때만
━━━━━━━━━━━━━
요금: {total_amount}
  (기본 {base_fare} + 카시트 {n} + 경유 {m})   ← 옵션 있을 때만
지불: {지불방식 문구}
```

**지불방식 문구 분기**

| payment_type | 문구 |
|---|---|
| host | 호스트 정산예정 (게스트 지불 없음) |
| guest | 게스트가 기사에게 직접 지불 |
| split | 게스트 직접 {guest_paid} + 호스트 {host_amount}<br>※ 기사: 게스트에게 {guest_paid}만 수령 |

### 5.2 게스트 배차 안내용 (영문)

```
Your driver has been assigned!

🚐 DRIVER ASSIGNED
━━━━━━━━━━━━━
Vehicle number: {vehicle_no}
Driver contact: {driver_phone}
Arrival: {date}, {time} ({flight_no})
Route: {port_en} → {area_en}
━━━━━━━━━━━━━

How to meet your driver:
Your driver tracks your flight and allows time for immigration
and baggage claim, so please don't worry if you're running late.
He will be waiting at your exit gate with a sign showing your name.

Two important notes:
1. Please exit through the designated gate only — leaving through
   a different one may cause you to miss each other.
2. Incheon Airport has two terminals (T1 and T2) and they are far
   apart. Everyone in your group must gather at the one designated
   terminal. ★ If the driver has to stop at the other terminal,
   an additional transfer fee applies.
```

한국어 버전도 동일 구조로 제공 (기존 운영 문구 사용).

---

## 6. 비기능 요구사항

| 항목 | 요구 |
|---|---|
| 모바일 | 게스트 폼은 모바일 우선. 375px 기준 정상 동작 |
| 다국어 | EN / KO / CN. 문구는 외부 파일 분리 |
| 개인정보 | 게스트명·연락처·항공편 저장 → 개인정보처리방침 필요, 수집 동의 체크박스 |
| 보관 | 예약 데이터 최소 5년 (세무 증빙) |
| 확장 | 모든 테이블에 `host_id`. 숙소·요금·문구는 설정 데이터로 분리 |
| 백업 | 일 1회 DB 백업 |

---

## 7. 개발 순서 제안

| 단계 | 내용 | 산출물 |
|---|---|---|
| 1 | DB 스키마 + 요금 계산 로직 | 마이그레이션, 단위테스트 |
| 2 | 게스트 폼 → 저장 | `/request` 동작 |
| 3 | 호스트 로그인 + 예약 목록·상세 | `/admin` 기본 |
| 4 | 메시지 생성 + 복사/공유 | 기사·게스트 템플릿 |
| 5 | 대시보드 브리핑 + 경고 규칙 | 알림 |
| 6 | 정산 집계 + 엑셀 내보내기 | `/admin/settlement` |

---

## 8. 참고 자료

동봉 파일:

- `pickup.html` — 게스트 폼 프로토타입 (UI/UX 및 요금 로직 기준)
- `alerts.html` — 알림 프로토타입 (브리핑 화면 참고)
- `Stay_Pachira_픽업운영장부.xlsx` — 현행 정산 장부 (엑셀 내보내기 목표 형태)
- `REQUIREMENTS.md` — 전체 서비스 구상 (MVP 이후 로드맵)

**실운영 데이터** (2026년 7~8월, 19건): 요금 계산·정산 로직 검증용 테스트 케이스로 활용 가능.
