# StayOps Phase 0 — External Validation Status

> 상태 기록일: 2026-09-04
> 목적: V1 Phase 0의 실제 외부 연동 검증 증거와 아직 하지 않은 작업을 분리해서 기록한다.
> 상위 구현 계약: `06_IMPLEMENTATION_READINESS.md`
> 배차 설계: `07_WhatsApp_배차_왕복_설계.md`
> 개발 순서: `05_DEVELOPMENT_PLAN.md`
>
> **2026-09-04 개편:** 배차 채널이 KakaoTalk에서 WhatsApp으로 바뀌었고, 연락 상대가 개별 기사에서 운송회사 대표기사 한 명으로 좁혀졌다. §1의 PASS 항목은 채널 변경과 무관하게 그대로 유효하다 (Guest 다리와 배차 다리가 같은 WhatsApp 인프라를 쓰기 때문). §2에 배차 관련 신규 검증 항목이 추가됐다.

---

## 0. 현재 판정

**Engineering-ready:** YES

**Phase 0 complete:** NO

**Real Guest pilot-ready:** NO

현재까지 **Meta 제공 Test WABA/Test Number를 이용한 WhatsApp Cloud API outbound + inbound webhook transport는 실제로 검증되었다.**

아직 실제 숙소 운영 WhatsApp Business 번호는 Meta/WABA에 추가하거나 Coexistence로 연결하지 않았다.

---

## 1. 완료된 WhatsApp 검증

### 1.1 Meta test assets

완료:

- Meta Developer App 생성
- WhatsApp use case/product 설정
- Meta 제공 Test WhatsApp Business Account 생성
- Meta 제공 Test Number 생성
- Test WABA ID 확인: `2098908274835715`

실제 숙소 운영번호는 이 단계에서 변경하지 않았다.

### 1.2 Outbound Cloud API

실제 WhatsApp 단말을 테스트 수신자로 등록하고 Meta Test Number에서 테스트 메시지를 전송했다.

결과:

```text
Meta Test Number
-> WhatsApp Cloud API
-> 실제 WhatsApp 단말
= PASS
```

### 1.3 Supabase webhook endpoint

Supabase project를 생성하고 Edge Function webhook endpoint를 배포했다.

```text
/functions/v1/whatsapp-webhook
```

Webhook verification용 `WHATSAPP_VERIFY_TOKEN`은 Supabase Edge Function Secret과 Meta Verify Token에 동일한 값을 사용한다.

직접 challenge 테스트에서:

```text
hub.mode=subscribe
hub.verify_token=<configured token>
hub.challenge=123456
```

에 대해 `123456`을 반환함을 확인했다.

결과:

```text
Meta webhook verification contract
= PASS
```

### 1.4 WABA app subscription

초기에는 Test WABA의 `subscribed_apps`에 StayOps 앱이 연결되어 있지 않아 실제 inbound event가 Supabase로 전달되지 않았다.

`/{WABA_ID}/subscribed_apps`를 확인하고 StayOps 앱을 Test WABA에 subscribe한 뒤 실제 webhook delivery가 시작되었다.

이 문제는 Phase 0 중 실제로 발견하고 해결한 외부 통합 조건으로 기록한다.

### 1.5 Inbound WhatsApp -> Meta -> Supabase

실제 WhatsApp 단말에서 Test Number로 다음 테스트 메시지를 보냈다.

```text
hello stayops
```

Supabase Edge Function Invocation에서:

```text
POST /functions/v1/whatsapp-webhook
HTTP 200
user-agent: facebookexternalua
```

를 확인했다.

Function log에서 실제 WhatsApp `messages` payload가 파싱되어 다음 정보가 존재함을 확인했다.

- `object = whatsapp_business_account`
- expected Test WABA entry
- `field = messages`
- Test Number metadata
- contact metadata
- text message object
- WhatsApp message id

결과:

```text
실제 WhatsApp 단말
-> Meta Test Number
-> Meta WABA webhook
-> Supabase Edge Function
-> JSON parse
= PASS
```

따라서 **테스트 번호 기준 Cloud API outbound/inbound transport feasibility는 검증 완료**다.

---

## 2. 아직 하지 않은 것

### 2.1 실제 숙소 WhatsApp Business 번호 Coexistence

**Status: NOT STARTED**

현재 실제 Guest 응대에 사용 중인 WhatsApp Business App 번호는:

- Meta Cloud API 번호로 migration하지 않음
- Business App 계정을 삭제하지 않음
- 기존 번호 연결 해제하지 않음
- Test WABA에 production number로 추가하지 않음
- Coexistence Embedded Signup 시작하지 않음

다음 검증의 목표는 **기존 WhatsApp Business App을 그대로 유지하면서 동일 번호에 Platform/API 자동화를 연결할 수 있는지 확인하는 것**이다.

필수 성공 조건:

```text
StayOps API -> 실제 운영번호 Guest outbound
Guest -> 실제 운영번호 -> StayOps webhook inbound
Host -> WhatsApp Business App manual reply 유지
가능하면 Business App sent-message echo -> StayOps
```

이 검증이 끝나기 전에는 실제 Guest pilot을 시작하지 않는다.

### 2.2 Production operational template

**Status: NOT STARTED**

실제 business-initiated 운영 메시지용 template의 생성/승인/실발송은 아직 검증하지 않았다.

최소 대상:

- request received
- driver assigned / vehicle information
- operator-confirmed major schedule change

### 2.3 Guest opt-in / privacy notice

**Status: NOT FINALIZED**

Guest Form에서 저장할:

```text
whatsapp_opt_in
whatsapp_opt_in_at
whatsapp_opt_in_policy_version
```

의 실제 사용자 문구와 개인정보 안내를 production pilot 전에 확정해야 한다.

### 2.4 대표기사 배차 왕복

**Status: NOT STARTED**

2026-09-04 개편으로 배차 채널이 KakaoTalk에서 WhatsApp으로 바뀌었다. 연락 상대는 개별 기사가 아니라 **운송회사 대표기사 한 명**이다. 상세는 `07_WhatsApp_배차_왕복_설계.md`.

검증해야 할 것:

```text
대표기사 연락처 확보
대표기사가 WhatsApp을 쓸 수 있는지 확인      ← 전제가 무너지면 SMS 어댑터로 전환
대표기사 opt-in 증거 확보
배차 요청 template 생성/승인
승인된 template을 대표기사 실단말로 발송
URL 버튼 -> signed 배차 응답 Web 오픈
가능/불가 + 기사 전화번호 + 차량번호 회신
같은 번호에서 Guest/Partner inbound 역할 라우팅
```

추가 확인 항목: 하나의 template에 quick-reply 버튼과 URL 버튼을 함께 넣을 수 있는지. 가능하면 24시간 자유 텍스트 창을 확보할 수 있어 운영 유연성이 올라간다. **V1 baseline은 URL 버튼 단독**으로 잡는다.

### 2.6 카카오 관련 항목 폐기

이전 Phase 0에 있던 `Kakao Share real-device round trip`은 폐기됐다. KakaoTalk은 사용하지 않는다.

### 2.5 Production logging hardening

**Status: TODO BEFORE REAL GUEST DATA**

현재 Phase 0 테스트용 `whatsapp-webhook` 함수는 webhook payload 전체를 debug log에 출력하는 상태다.

실제 Guest 데이터를 받기 전 반드시 raw payload logging을 제거/축소한다.

Production log에는 원칙적으로 다음 정도만 남긴다.

```text
waba_id
field
phone_number_id
has_message
message_type
message_id
request_id / execution status
```

Guest 이름, 전화번호/wa_id, 실제 message body 등은 일반 function log에 평문으로 남기지 않는다.

실제 메시지 데이터 보존이 필요한 경우 이후 canonical `MessageEvent` storage policy에 따라 DB에 저장한다.

---

## 3. Phase 0 현재 체크리스트

```text
[PASS] Meta Developer App / WhatsApp test assets
[PASS] Test Number outbound -> actual WhatsApp device
[PASS] Supabase Edge Function webhook endpoint
[PASS] Meta callback verification
[PASS] messages webhook field configured
[PASS] StayOps app subscribed to Test WABA
[PASS] actual WhatsApp inbound -> Meta -> Supabase POST 200
[PASS] inbound messages payload JSON parsing

[TODO] remove/reduce raw webhook payload logging before real Guest data
[TODO] actual accommodation WhatsApp Business number Coexistence onboarding
[TODO] actual-number API outbound
[TODO] actual-number Guest inbound webhook
[TODO] Host manual Business App reply while API remains connected
[TODO] sent-message echo if supported by selected Coexistence path
[TODO] Guest production template send
[TODO] Guest opt-in/privacy copy

[TODO] 대표기사 연락처 확보
[TODO] 대표기사 WhatsApp 사용 가능 여부 확인
[TODO] 대표기사 opt-in 증거 확보
[TODO] 배차 요청 template 승인
[TODO] 대표기사 실단말 수신 + URL 버튼 -> 배차 응답 Web 오픈
[TODO] quick-reply + URL 버튼 조합 가능 여부 확인
[TODO] Guest/Partner inbound 역할 라우팅 검증
[TODO] Host 운영 알림 발송 경로 확인
```

---

## 4. 다음 Exact Action

다음 Phase 0 작업은 **실제 숙소 WhatsApp Business 번호의 Coexistence 연결 검증**이다.

진행 전 안전 규칙:

```text
DO NOT delete the current WhatsApp Business App account.
DO NOT disconnect the current production number.
DO NOT migrate the number to API-only mode by accident.
DO NOT treat the Meta test number as the production number.
```

그 전에 또는 동시에 테스트 Edge Function의 raw payload logging을 제거/축소한다.

Coexistence가 성공하면 같은 Supabase webhook endpoint를 사용해 실제 번호에서 outbound/inbound/manual-reply 경로를 재검증한다.

---

## 5. Phase 0 완료 조건

Phase 0 WhatsApp 부분은 아래가 모두 실제 운영번호에서 검증된 뒤 완료로 본다.

```text
real Business number remains operational
+ StayOps outbound works
+ Guest inbound webhook works
+ Host manual reply path works
+ required production template works
+ no production PII raw logging
```

배차 부분은 아래가 검증되면 V1 baseline external feasibility를 충족한다.

```text
대표기사 template 승인
+ 대표기사 실단말 수신
+ URL 버튼 -> signed 배차 응답 Web 오픈
+ 가능/불가 + 기사 전화번호 + 차량번호 회신
+ Guest/Partner inbound 역할 분리
```

대표기사가 WhatsApp을 쓸 수 없으면 `PartnerNotificationAdapter`를 `sms`로 전환한다. 같은 signed URL을 보내므로 도메인 구현은 영향받지 않는다.
