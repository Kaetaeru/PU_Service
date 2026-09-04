# 07. WhatsApp 기반 배차 왕복 설계

> 이 문서는 `04_카카오_기사배차_왕복_설계.md`를 **대체한다.**
> 상위 규칙: `00_CANONICAL_SYSTEM.md`
> V1 범위: `V1_INTERNAL_MVP.md`

---

## 1. 결론

StayOps는 **모든 외부 메시지를 WhatsApp 하나로 보낸다.** KakaoTalk은 사용하지 않는다.

기사 쪽 연락 상대는 개별 운행 기사가 아니라 **운송회사의 배차 담당자(대표기사) 한 명**이다. 대표기사가 회사 내부에서 실제 운행 기사를 배정하고, 그 결과(기사 연락처·차량번호)를 StayOps에 입력한다.

```text
Guest Web
   |
   v
StayOps  ──────> Guest WhatsApp   (접수 확인)
   |     ──────> Host WhatsApp    (신규 요청 알림)
   |
   |  Host가 지불 확인 후 [배차 요청] 1회 클릭
   v
대표기사 WhatsApp  (배차 요청 + 입력 링크)
   |
   |  링크 탭
   v
signed Driver Response Web
   (수락/불가 · 기사 전화번호 · 차량번호)
   |
   v
StayOps  ──────> Guest WhatsApp   (배차 완료 안내)
         ──────> Host WhatsApp    (배차 완료 알림)
```

정상 건에서 Host가 메시지를 작성하거나 정보를 옮겨 적는 일은 없다. Host의 조작은 **지불 확인 후 버튼 1회**뿐이다.

---

## 2. 왜 이 구조인가

### 2.1 채널을 하나로 줄인 이유

- 검증·승인·온보딩 대상이 하나뿐이다. 카카오 파트너 계약과 알림톡 템플릿 심사가 통째로 빠진다.
- Guest 다리와 Driver 다리가 같은 인프라를 쓴다. 어댑터·outbox·webhook·중복처리 코드가 한 벌이다.
- 기존 카카오 Share 방식은 Host가 매 건 채팅방을 골라 공유해야 했다. WhatsApp 자동 발송은 그 조작을 없앤다.

### 2.2 대표기사 한 명으로 좁힌 이유

실제 운영은 운송회사에 배차를 맡기는 구조다. StayOps가 개별 기사를 알 필요가 없다.

이 결정으로 다음이 전부 불필요해진다.

- 기사 마스터 관리 UI
- Host의 기사 선택 단계
- 복수 기사 fan-out과 선착순 경쟁
- 동시 수락 충돌 처리
- 무응답 시 자동 순번 이관

또한 **WhatsApp을 써야 하는 사람이 한 명**으로 줄어든다. 국내 기사 다수가 WhatsApp을 쓰지 않을 위험이 실질적으로 해소된다.

### 2.3 기사 응답을 자유 텍스트로 받지 않는 이유

자유 텍스트를 파싱하면 오타·형식 불일치·누락이 그대로 운영 사고가 된다. 응답은 항상 **정해진 칸이 있는 웹 폼**으로 받는다. 이 원칙은 채널이 카카오에서 WhatsApp으로 바뀌어도 동일하다.

---

## 3. 대표기사 계약

### 3.1 DispatchPartner

StayOps가 아는 기사 측 주체는 이것 하나다.

```text
partner_id
host_id
name                        예: "○○투어 배차팀"
contact_name                대표기사 이름
whatsapp_phone_e164         필수
whatsapp_opt_in             필수 true
whatsapp_opt_in_at
whatsapp_opt_in_policy_version
preferred_language          기본 ko
notification_channel        whatsapp | sms | manual
is_active
```

대표기사도 business-initiated 메시지 수신 대상이므로 **opt-in 증거를 보존한다.** 계약서·서면·채팅 동의 중 하나로 확보하고 시점과 정책 버전을 남긴다.

`notification_channel`은 어댑터 선택값이다. V1 기본은 `whatsapp`이며, 대표기사가 WhatsApp을 쓸 수 없으면 설정만 바꿔 `sms`로 전환한다. 도메인 로직은 바뀌지 않는다.

### 3.2 개별 운행 기사

StayOps는 개별 기사를 **저장된 주체로 관리하지 않는다.** 대표기사가 배차 응답에 입력한 값이 해당 건의 기사 정보다.

```text
DriverAssignment.driver_phone_e164
DriverAssignment.vehicle_plate
```

기사 마스터, 기사 계정, 기사 로그인은 V1에 없다.

---

## 4. 배차 요청 발송

### 4.1 발송 조건

Host가 아래를 만족한 상태에서 [배차 요청]을 눌렀을 때만 발송한다.

```text
필수 손님 정보 유효
일정 유효
활성 숙소/노선 유효
인원 1..7
지불 확인 완료 (payment_confirmed = true)
요청이 취소되지 않음
```

**지불은 이 서비스 바깥에서 이뤄진다.** StayOps는 금액을 정산하지 않고, Host가 "확인했다"고 표시한 사실만 기록한다. 자세한 것은 `00_CANONICAL_SYSTEM.md` §6.

### 4.2 메시지 구성

승인된 WhatsApp 템플릿으로 발송한다. **요약은 메시지에, 상세는 링크 뒤에** 둔다.

```text
[Stay Pachira 배차 요청]
8/20(수) 15:40 · 공항 픽업
인천공항 T1 → 이태원 (파키라하우스 1층)
6명 / 캐리어 4개
카시트 1개 필요
경유: T1 ⇄ T2

요금(세금포함): 143,000원
  기본 100,000 + 카시트 10,000 + 경유 20,000 + 세금 13,000

[ 배차 정보 입력 ]     ← URL 버튼
```

**요금은 세금포함 총액으로 넣는다.** 요금표는 `08_요금표.md`가 기준이며, 표기 금액은 공급가이므로 10%를 더한 총액을 보여준다. 항목 분해는 옵션이 있을 때만 붙인다.

메시지에 넣지 않는 것:

- 손님 전화번호
- 손님 전체 이름 (기사 안내판용 표기는 링크 뒤 페이지에)
- 항공편·특이사항 등 상세 (링크 뒤 페이지에 표시)

이유:

1. 템플릿 변수가 적을수록 Meta 승인이 빠르고 반려가 적다.
2. 손님 개인정보가 Meta 인프라와 대표기사 단말 채팅 이력에 영구 잔존하지 않는다.
3. 예약이 수정되면 이미 보낸 채팅 메시지는 옛 내용으로 남지만, 링크 뒤 페이지는 항상 현재 revision을 보여준다.

**Guest에게 가는 메시지에는 금액이 들어가지 않는다.** 요금은 Host와 대표기사에게만 쓰이는 내부 지표다.

### 4.3 상태 전환

```text
Host [배차 요청] 클릭
  -> DriverDispatch 생성 (status = pending, request_revision 스냅샷, 응답 토큰 발급)
  -> Notification(audience=partner) outbox 적재
  -> 발송 성공(sent)
  -> DriverDispatch: pending -> awaiting_response
```

카카오 방식에서는 Host가 실제 share를 실행한 뒤에 `awaiting_response`로 넘어갔다. WhatsApp에서는 **발송 성공이 전환 트리거**다.

`notification_channel = manual`인 경우에만 Host의 "전달 완료" 확인으로 전환한다.

---

## 5. 배차 응답 웹

### 5.1 링크

```text
https://<frontend>/driver#t=<opaque-token>
```

- 최소 128-bit 난수 opaque token
- DB에는 hash만 저장
- `expires_at` 보유
- fragment(`#`)로 전달해 HTTP request/referrer 로그 노출을 줄인다
- 페이지 초기화 후 주소 표시줄에서 token을 제거한다

### 5.2 입력 항목

```text
[필수] 배차 가능 / 배차 불가
[필수] 기사 전화번호     국가번호 포함 E.164로 저장
[필수] 차량번호
[선택] 차량 종류/색상
[선택] 메모
```

전화번호는 **국가번호를 포함해 저장한다.** 손님이 해외 로밍 단말로 직접 전화를 걸 수 있어야 하기 때문이다. 입력 UI는 국가번호 선택 + 현지번호 형태를 허용하되 저장 전 국제전화번호 파서로 E.164 정규화한다.

`도착 예정 시각(ETA)`은 V1 필수 항목이 아니다. 픽업 시각은 항공편 착륙 시각 기준으로 이미 확정돼 있고, 현행 운영 장부도 ETA를 기록하지 않는다.

### 5.3 페이지에 표시할 상세

링크를 연 대표기사에게는 운행에 필요한 전체 정보를 보여준다.

```text
일시 / 구분 (픽업·드랍오프)
출발지 → 목적지 (숙소 정확한 주소 포함)
공항 터미널
항공편 번호
손님 이름 (기사 안내판용)
인원 / 대형수하물 수
카시트 수 및 아동 연령 메모
터미널 경유 여부
특이사항
요금 (세금포함 총액 + 항목 분해)
```

손님 전화번호는 표시하지 않는다.

### 5.4 확정 트랜잭션

수락 처리는 하나의 트랜잭션에서 끝낸다.

1. token hash 조회 및 잠금
2. 만료·폐기 여부 확인
3. 요청 상태와 revision 확인 (`dispatch.request_revision == transfer.revision`)
4. 이미 active assignment가 없는지 확인
5. `DriverAssignment` 생성
6. `DriverDispatch` → `accepted`
7. `TransferRequest` → `assigned`
8. Guest 배차 안내 Notification outbox 적재
9. Host 알림 Notification outbox 적재

중복 제출은 멱등 처리한다. 같은 응답을 반복하면 최초 결과를 반환한다.

### 5.5 배차 불가

```text
DriverDispatch -> rejected
TransferRequest -> awaiting_driver 유지
Host에게 알림
```

대표기사가 배차 불가로 회신하면 회사 차원에서 못 받는다는 뜻이다. 자동 재발송은 하지 않고 Host가 다른 경로를 찾는다. 자주 발생하는 상황이 아니므로 V1에서 자동화하지 않는다.

### 5.6 무응답

`expires_at`을 넘기면 `expired`로 전환하고 Host에게 알린다. 만료 시간은 설정값이며 초기값 후보는 발송 후 60분이다.

---

## 6. 손님 안내 반환

active `DriverAssignment`가 생기면 Guest WhatsApp Notification을 큐에 넣는다.

포함 정보:

```text
예약 참조번호
차량번호
기사 전화번호 (국가번호 포함)
일시
노선
미팅 포인트
```

Host는 이 문구를 작성하지 않는다. 손님 `preferred_language` 템플릿이 있으면 사용하고 없으면 English로 fallback한다.

---

## 7. 단일 번호와 역할 라우팅

Guest와 대표기사가 **같은 WABA 번호**와 대화한다. 따라서 들어오는 메시지의 역할을 먼저 판정해야 한다.

```text
inbound message (provider wa_id)
  -> E.164 정규화
  -> 역할 판정
       활성 DispatchPartner 번호와 일치      -> role = partner
       활성 TransferRequest 손님 번호와 일치 -> role = guest
       둘 다 일치                            -> role = partner (+ 충돌 플래그)
       어느 쪽도 아님                        -> role = unknown
  -> MessageEvent 저장 (role, request_id nullable)
  -> 후속 처리
       partner : 활성 dispatch가 있으면 입력 링크 1회 재안내, Host에 표시
       guest   : 활성 request가 정확히 1건일 때만 연결.
                 0건 또는 복수면 자동 추정하지 않고 needs_association
       unknown : Host 확인 대상
```

대표기사가 링크 대신 자유 텍스트로 답장하는 경우, 고정 문구로 링크를 한 번 다시 안내한다. AI 응답이 아니라 결정적 재안내이므로 "V1에서 AI 자유대화 자동응답 금지" 원칙과 충돌하지 않는다. **응답 내용을 파싱하지는 않는다.**

번호를 Guest용/Partner용으로 분리하는 방안은 Post-V1에서 검토한다. 월 20건 규모에서는 온보딩·템플릿 승인 비용이 두 배가 되는 쪽이 손해다.

---

## 8. 24시간 창

WhatsApp은 상대가 메시지를 보내야 24시간 자유 텍스트 창이 열린다. URL 버튼 탭은 메시지가 아니므로 창이 열리지 않는다.

따라서 **대표기사에게 보내는 모든 business-initiated 메시지는 승인된 템플릿이어야 한다.**

`wa_contacts`로 창 상태를 추적하고, 발송 직전 템플릿/자유 텍스트를 결정한다.

```text
wa_id / phone_e164   PK
host_id
role                 guest | partner | unknown
linked_partner_id    nullable
last_inbound_at
window_expires_at    last_inbound_at + 24h
```

### Phase 0 확인 항목

하나의 템플릿에 quick-reply 버튼과 URL 버튼을 함께 넣을 수 있는지 확인한다. 가능하면 `[확인] + [배차 정보 입력]` 구성으로 바꾼다. 대표기사가 [확인]을 누르는 순간 24시간 창이 열리고, 이후 변경 통지·재촉을 템플릿 없이 자유 텍스트로 보낼 수 있어 운영 유연성이 크게 올라간다.

Meta의 버튼 조합 규칙은 변동이 있으므로 **V1 baseline은 URL 버튼 단독**으로 잡고, 확인되면 업그레이드한다.

---

## 9. 템플릿 목록

| # | 이름 | 대상 | 언어 | 용도 |
|---|---|---|---|---|
| 1 | `guest_request_received` | Guest | en / ko | 접수 확인 |
| 2 | `guest_driver_assigned` | Guest | en / ko | 배차 완료 (차량번호·기사 연락처·미팅포인트) |
| 3 | `guest_schedule_changed` | Guest | en / ko | Host가 확정한 주요 일정 변경 |
| 4 | `partner_dispatch_request` | 대표기사 | ko | 배차 요청 + 입력 링크 |
| 5 | `partner_dispatch_cancelled` | 대표기사 | ko | 배차 취소 통보 |
| 6 | `host_notice` | Host | ko | 신규 요청 / 배차 완료 / 배차 불가 / 무응답 |

6번은 Host 개인 번호로 보내는 운영 알림이다. Host가 화면을 보고 있지 않아도 상황을 알 수 있게 한다.

월 20건 기준 예상 대화량은 약 60~80건으로, 비용은 고려 대상이 아니다.

---

## 10. 전송 어댑터

전송 수단이 막혀도 도메인이 멈추지 않도록 어댑터 뒤에 둔다.

```text
PartnerNotificationAdapter
  ├─ whatsapp   V1 기본
  ├─ sms        대표기사가 WhatsApp 미사용 / 템플릿 미승인 시
  └─ manual     최후: Host가 생성된 메시지와 링크를 임의 채널로 전달
```

세 경로 모두 **같은 signed URL**을 보낸다. 응답은 언제나 웹으로 돌아오므로 `DriverDispatch` / `DriverAssignment` / 확정 트랜잭션은 전송 수단과 무관하다.

`manual` 경로는 기존 설계의 원칙을 그대로 지킨다.

> fallback은 운영자 클릭 1회를 요구할 수 있으나, canonical 데이터 재입력을 요구해서는 안 된다.

---

## 11. 예약이 변경·취소될 때

### 일정 등 주요 항목 변경

```text
TransferRequest.revision += 1
  -> 기존 dispatch의 토큰은 수락 불가 (stale_dispatch)
  -> 배차 전이면 새 dispatch 발송
  -> 배차 후면 needs_reconfirmation
  -> 큐에 남은 옛 Notification은 발송 직전 검사에서 skipped_stale 처리
```

### 취소

```text
TransferRequest -> cancelled
  -> 대표기사에게 취소 통보 템플릿 발송
  -> 큐에 남은 Notification 발송 안 함
  -> 이후 assignment 생성 불가
```

### 배차 확정 후 차량·기사가 바뀌는 경우

**V1에서 시스템으로 처리하지 않는다.** 이례적인 상황이므로 Host가 그때그때 수동으로 대응한다. 응답 링크를 장기 유효화하거나 재입력 흐름을 만들지 않는다.

---

## 12. 04번 문서에서 승계한 것과 버린 것

### 승계

- 기사 자유 텍스트를 파싱하지 않는다
- signed 응답 링크로 구조화된 데이터를 받는다
- 배차 요청에 손님 개인정보를 최소로 넣는다
- 응답 토큰은 짧은 TTL, 폐기 가능, hash-at-rest
- 전송 수단을 어댑터로 추상화한다
- 메시지 전송 성공은 배차 완료가 아니다
- 재배차 시 새 dispatch를 만든다

### 버림

- KakaoTalk 알림톡 / 채널 메시지 / 챗봇 Skill 전체
- Kakao Share one-click
- 카카오 채널 Webhook 관련 논의
- 복수 기사 fan-out과 first-accept 경쟁
- `driver_notified` / `guest_notified`를 포함한 단일 enum 상태 머신 (`00_CANONICAL_SYSTEM.md` §10이 대체)
- GuestLocation 실시간 위치 공유 (V1 필수 아님)
- 기사 마스터 관리
