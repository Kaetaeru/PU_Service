# StayOps 기술적 기획 — 시작점

이 폴더는 StayOps의 제품/기술 기획 Source of Truth를 관리한다.

StayOps의 핵심 목표는 **호스트가 게스트와 파트너 사이의 수동 중계자가 되지 않게 하는 것**이다. 첫 번째 실제 제품 범위는 공항/서울역 픽업·드랍오프 왕복 자동화다.

## 반드시 먼저 읽을 것

새 작업이나 기획 검토를 시작할 때는 아래 순서로 읽는다.

1. `00_CANONICAL_SYSTEM.md`
2. `05_DEVELOPMENT_PLAN.md`
3. 필요한 경우에만 아래 세부/근거 문서를 읽는다.

두 canonical 문서와 기존 초안이 충돌하면 canonical 문서가 우선한다.

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
  - request / dispatch / message / settlement state
        |
        +----> Driver KakaoTalk
        |        -> signed response web
        |        -> accept/reject + ETA + vehicle
        |
        +----> Guest WhatsApp
                 -> automated operational updates
                 -> guest free-text inbound

Host
  - StayOps Admin: exception/operation/settlement
  - WhatsApp Business App: direct manual guest conversation
```

정상 transfer에서는 호스트가 메시지를 복사하거나 차량번호를 재입력하지 않는다.

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
- 선호 연결 방식 = Business App + API Coexistence
- Coexistence 가능 여부는 호스트 onboarding 때 실제 검증
- MVP에서는 자유대화 AI 자동응답을 하지 않음

### Driver

- 기본 채널 = KakaoTalk
- 자유 텍스트 답장을 파싱하지 않음
- signed Driver Response URL에서 수락/거절, ETA, 차량번호 입력
- Kakao 실제 발송 공급자는 adapter 뒤에 두고 feasibility 단계에서 검증

### StayOps

- 예약/배차/메시지/정산의 Source of Truth
- managed/serverless backend 사용
- 1차 권장 방향 = Supabase 계열 Postgres/Auth/Edge Functions 구조를 검증
- 모든 tenant-owned 데이터에 처음부터 `host_id`

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

- `05_DEVELOPMENT_PLAN.md`
  - 실제 구현 순서
  - 단계별 dependency
  - 완료조건
  - pilot 및 SaaS화 순서

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
  - 호스트 수동 차량정보 재입력 등 canonical과 충돌하는 부분은 더 이상 우선하지 않음

### 운영 근거

- `Stay_Pachira_픽업운영장부_2.xlsx`
  - 실운영 요금/정산 검증 자료

- `pickup_2.html`
  - 현재 Guest transfer UI 프로토타입

---

## 개발 착수 기준

구현은 `05_DEVELOPMENT_PLAN.md`의 순서를 따른다.

첫 제품 성공 기준은 다음 하나다.

```text
Guest Form
-> StayOps 저장
-> Driver 요청
-> Driver response
-> StayOps assignment
-> Guest WhatsApp return
```

이 한 건이 호스트의 복사/재입력 없이 실제 휴대폰에서 끝나기 전에는 청소, 세탁, iCal, AI 채팅, SaaS 과금으로 범위를 넓히지 않는다.
