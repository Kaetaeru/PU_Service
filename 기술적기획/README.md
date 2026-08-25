# StayOps 기술적 기획 — 시작점

이 폴더는 StayOps의 제품/기술 기획 Source of Truth를 관리한다.

StayOps의 핵심 목표는 **호스트가 게스트와 파트너 사이의 수동 중계자가 되지 않게 하는 것**이다. 첫 번째 실제 제품 범위는 공항/서울역 픽업·드랍오프 왕복 자동화다.

## 반드시 먼저 읽을 것

새 작업이나 기획 검토를 시작할 때는 아래 순서로 읽는다.

1. `00_CANONICAL_SYSTEM.md`
2. `V1_INTERNAL_MVP.md` — V1/내부 실사용 범위를 판단할 때 필수
3. `05_DEVELOPMENT_PLAN.md`
4. 필요한 경우에만 아래 세부/근거 문서를 읽는다.

`00_CANONICAL_SYSTEM.md`는 전체 제품 동작과 불변조건의 최우선 Source of Truth다.

`V1_INTERNAL_MVP.md`는 **판매 전 우리가 직접 사용하는 V1의 포함/제외 범위와 release gate**를 정의한다. V1 범위를 판단할 때 기존 `MVP_SPEC.md`보다 이 문서가 우선한다.

---

## 현재 확정된 큰 구조

```text
Guest
  - Transfer Web
  - WhatsApp
        |
        v
      StayOps
  - managed backend
  - request / dispatch / message / payment state
        |
        +----> Driver KakaoTalk
        |        -> signed response web
        |        -> accept/reject + ETA + vehicle
        |
        +----> Guest WhatsApp
                 -> automated operational updates
                 -> guest free-text inbound

Host
  - StayOps Admin: operation / exception recovery
  - WhatsApp Business App: direct manual guest conversation
```

정상 transfer에서는 호스트가 메시지를 복사하거나 차량번호를 재입력하지 않는다.

---

## 현재 버전 경계

### V1 — Internal Operations MVP

판매용 SaaS가 아니라 **우리 숙소 운영에서 실제 Guest/Driver를 상대로 매일 쓸 수 있는 버전**이다.

V1 = `05_DEVELOPMENT_PLAN.md`의 Phase 0~6.

```text
Guest Form
-> StayOps
-> Guest WhatsApp receipt
-> Driver dispatch
-> signed Driver response
-> DriverAssignment
-> Guest WhatsApp return
-> Host exception recovery
-> real pilot
```

V1에서 허용되는 Host 작업:

- 기사 선택/배차 시작 같은 운영 승인
- 일정 변경, 취소, 재배차
- 실패한 메시지 복구
- Guest 자유 문의에 WhatsApp Business App으로 수동 답장
- 예외 상황의 manual assignment 보정

V1에서 금지되는 정상 작업:

- 기사 메시지 수동 재작성
- 기사 차량번호/ETA를 Host가 다시 입력
- Guest 안내문 복사/재작성

### Post-V1

- 정산 자동화/Excel 대체
- 멀티호스트 self-service onboarding
- SaaS 판매/과금
- iCal
- 청소/세탁
- AI Guest 자동답변
- broader StayOps

V1 동안 정산은 기존 장부를 계속 사용하되 fare/payment snapshot은 V1부터 저장한다.

---

## Canonical 결정 요약

### Guest

- 기본 연락처 = WhatsApp 번호
- `whatsapp_phone_e164` + opt-in을 필수 연락 계약으로 사용
- 선호 언어 저장
- 실시간 위치는 예약 시 필수가 아니라 픽업 직전 필요 시 별도 공유

### WhatsApp

- Guest 자동 운영 메시지 + inbound webhook
- Host는 가능하면 동일 번호를 WhatsApp Business App에서 계속 직접 사용
- V1 목표 연결 방식 = Business App + API Coexistence
- Coexistence는 실제 내부 번호로 Phase 0에서 검증
- V1에서는 자유대화 AI 자동응답을 하지 않음

### Driver

- 기본 채널 = KakaoTalk
- 자유 텍스트 답장을 파싱하지 않음
- signed Driver Response URL에서 수락/거절, ETA, 차량번호 입력
- 자동 Kakao outbound가 즉시 불가능하면 V1은 StayOps가 생성한 메시지의 one-click share/send를 허용
- Host의 수동 재작성/차량정보 재입력은 허용하지 않음

### StayOps

- 예약/배차/메시지/payment 상태의 Source of Truth
- managed/serverless backend 사용
- 1차 권장 방향 = Supabase 계열 Postgres/Auth/Edge Functions 구조를 검증
- DB에는 처음부터 `host_id`와 tenant isolation을 넣되 V1 UI는 우리 내부 Host만 지원하면 충분

---

## 문서 지도

### 현재 Source of Truth

- `00_CANONICAL_SYSTEM.md`
  - 제품 목표
  - actor/channel 역할
  - Guest 입력 계약
  - WhatsApp/기사 왕복
  - 상태 모델
  - canonical domain model
  - multi-host/보안/fallback
  - plan-critic으로 발견된 주요 문제와 수정

- `V1_INTERNAL_MVP.md`
  - 판매 전 내부 실사용 V1의 정확한 scope
  - Guest/Driver/Host의 V1 기능
  - 허용 fallback
  - failure/recovery
  - Post-V1 제외범위
  - V1 release gate

- `05_DEVELOPMENT_PLAN.md`
  - 실제 구현 순서
  - 단계별 dependency
  - 완료조건
  - pilot 및 SaaS화 순서
  - Phase 0~6이 V1을 구성

### 세부 분석 / 이전 초안

- `01_HTML_분석.md`
  - 최초 transfer HTML 구조/요금/제출 분석

- `02_PU_Service_연결_설계.md`
  - HTML -> backend 경계의 초기 설계

- `03_API_계약_초안.md`
  - 초기 transfer request API contract

- `04_카카오_기사배차_왕복_설계.md`
  - 기사 Kakao -> signed response -> assignment 구조의 상세 근거
  - 단, Guest 리턴 채널은 canonical 결정에 따라 WhatsApp이 우선

- `REQUIREMENTS.md`
  - StayOps 전체 사업/운영 문제와 장기 모듈 아이디어

- `MVP_SPEC.md`
  - 기존 MVP 상세 초안
  - V1 범위나 Host 수동 차량정보 재입력 등 canonical/V1 문서와 충돌하는 부분은 더 이상 우선하지 않음

### 운영 근거

- `Stay_Pachira_픽업운영장부_2.xlsx`
  - 실운영 요금/정산 검증 자료

- `pickup_2.html`
  - 현재 Guest transfer UI 프로토타입

---

## 개발 착수 기준

구현은 `05_DEVELOPMENT_PLAN.md`의 순서를 따르고, V1 완료 여부는 `V1_INTERNAL_MVP.md`의 release gate로 판단한다.

V1의 첫 제품 성공 기준은 다음 왕복이다.

```text
Guest Form
-> StayOps 저장
-> Driver 요청
-> Driver response
-> StayOps assignment
-> Guest WhatsApp return
```

여기에 실제 운영에 필요한 실패 복구와 Host exception UI까지 포함해 Phase 0~6을 통과해야 V1로 본다.

이 V1이 실제 운영에서 안정화되기 전에는 정산 자동화, 멀티호스트 판매, 청소, 세탁, iCal, AI 채팅으로 범위를 넓히지 않는다.
