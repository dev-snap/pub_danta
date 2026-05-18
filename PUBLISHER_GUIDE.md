# Publisher (매체사) 연동 가이드

단타게임(Service DT) 매체사용 연동 문서. 매체사가 게임 페이지로 사용자를 진입시키고, 콜백을 수신하고, 운영용 REST API 를 호출할 때 필요한 사양을 한 곳에 모은다.

> ⚠️ **본 문서는 샘플 문서입니다.**
> 모든 예시 수치는 아래 기준값을 전제로 작성되었습니다.
> - **베팅 현금 금액**: `1,000원`
> - **매체 포인트 환율 (`point_rate`)**: `1 포인트 = 50,000원`
>
> 실제 연동 시에는 매체별로 합의된 값이 적용됩니다.

| 항목 | 값 |
|---|---|
| 작성일 | 2026-04-29 |
| 대상 | 매체사 개발자 / 인티그레이터 |

---

## §1. 참여 신청 (Onboarding)

매체사 자가가입 채널은 없다. 영업/운영팀 합의 후 단타 운영팀이 매체 정보를 등록하고 자격증명을 발급한다.

### 1-1. 등록 시 매체사가 운영팀에 전달할 정보

| 항목 | 필수 | 설명 |
|---|---|---|
| 매체사 운영명 | ✓ | 매체사 명칭 (운영 식별용) |
| `client_id` | ✓ | 매체사 그룹 코드 (운영팀 합의로 결정) |
| `state_callback_url` | 권장 | 라운드 이벤트 알림 수신 URL — 미설정 시 콜백 미발송 |
| `bet_callback_url` | ✓ | 베팅 동기 승인 수신 URL |
| `point_rate` | ✓ | 매체 포인트 환율 (예: 1 포인트 = 50000원이면 `50000`) |

운영팀 검수 후 **`app_key`** 와 **`callback_secret`** 이 발급된다.

### 1-2. 발급 자격증명

- **`app_key`** — 모든 요청에 첨부 (헤더/쿼리/바디 중 하나). 외부 노출 가능 (단순 식별자).
- **`callback_secret`** — HMAC 서명에 사용. **외부 노출 금지**.

### 1-3. 환경

| 환경 | 도메인 |
|---|---|
| 개발 (dev) | `https://dantadev.snapplay.io/` |
| 운영 (live) | 대기중… |

---

## §2. 사용자 진입

매체사는 자사 사이트에서 단타 게임 페이지로 **링크** 한다. 진입 URL 쿼리 파라미터로 사용자 식별 정보를 전달한다.

### 2-1. 진입 URL 형식

```
https://dantadev.snapplay.io/?user_id={USER_ID}&adid={ADID}&app_key={APP_KEY}
```

### 2-2. 쿼리 파라미터

| 파라미터 | 별칭 | 필수 | 설명 |
|---|---|---|---|
| `user_id` | `uid` | ✓ | 매체사가 부여한 유저 식별자 |
| `app_key` | `appKey`, `appkey` | ✓ | 매체사 인증 키 |
| `adid` | — | 선택 | 광고 식별자 (없으면 빈 문자열) |

---

## §3. 콜백 (Callback) 수신

단타 서버는 라운드/베팅 이벤트마다 매체사의 `state_callback_url` 또는 `bet_callback_url` 로 HTTP POST 를 발송한다. **본 절은 모든 콜백에 공통**이며, 이벤트별 페이로드 상세는 §4 에서 다룬다.

### 3-1. 공통 사양

| 항목 | 값 |
|---|---|
| 메서드 | `POST` |
| Content-Type | `application/json` |
| 인코딩 | UTF-8 |
| 타임아웃 | 10초 |
| 리트라이 | 이벤트별 차등 — 아래 §3-6 참조 |

**리트라이 정책 (2026-05-18~)**:
| 이벤트 | 정책 |
|---|---|
| `bet_confirmed` / `bet_canceled` | HTTP 2xx 외엔 **10초 간격으로 최대 3회 재시도** |
| `bet_request` | 동기 호출이라 재시도 없음 (응답 즉시 결정) |
| `round_started` / `round_closed` / `round_ended` / `round_canceled` | 1회 발송 (재시도 없음) |

> 매체사는 `order_id` (§4-9) 를 UNIQUE 키로 사용하면 중복 수신에도 멱등 처리 가능 — 같은 콜백이 재전송돼도 이중 차감/환불 없음. 자세한 가이드는 §6-1 참조.

### 3-2. 공통 헤더

| 헤더 | 값 | 설명 |
|---|---|---|
| `X-App-Key` | `{app_key}` | 매체 식별 |
| `X-Signature` | `sha256={hex}` | HMAC-SHA256(`callback_secret`, raw_body) |
| `Content-Type` | `application/json` | |

### 3-3. 공통 바디 형식

```json
{
  "event_type": "round_started",
  "app_key": "your_app_key",
  "data": { ... },
  "timestamp": "2026-04-29T11:43:29.000Z"
}
```

### 3-4. 서명 검증 (매체사 측 의무)

```js
// Node.js 예시
const crypto = require('crypto');
function verify(rawBody, signatureHeader, callbackSecret) {
  const expected = 'sha256=' + crypto.createHmac('sha256', callbackSecret)
    .update(rawBody, 'utf8').digest('hex');
  return signatureHeader === expected;
}
```

### 3-5. 응답 컨벤션

| HTTP 상태 | 의미 |
|---|---|
| `2xx` | 수신 성공 |
| 그 외 | 수신 실패 (재시도 없음 — 매체사가 §5 REST API 로 폴링하여 보완) |

> `bet_request` 만 응답 바디로 **승인 여부** 를 회신해야 한다. 다른 이벤트는 `2xx` = 수신 확인 의미.

### 3-6. 이벤트 타입 일람

| `event_type` | 채널 | 발송 시점 | 응답 의미 |
|---|---|---|---|
| `round_started` | `state_callback_url` | 라운드 OPEN | 수신 확인 |
| `round_closed` | `state_callback_url` | 베팅 마감 (closed) | 수신 확인 |
| `round_ended` | `state_callback_url` | 라운드 종료 + 정산 export 완료 | 수신 확인 |
| `round_canceled` | `state_callback_url` | 라운드 무효 처리 + 환불 export 완료 | 수신 확인 |
| `bet_request` | `bet_callback_url` | 베팅 시도 시 동기 호출 | **승인/거부 응답 필수** (재시도 없음) |
| `bet_confirmed` | `bet_callback_url` | 베팅 트랜잭션 커밋 직후 | 수신 확인 — **2xx 외엔 10초 × 3회 재시도** |
| `bet_canceled` | `bet_callback_url` | 라운드/베팅 취소로 환불 필요 | 수신 확인 + **매체 측 `media_amount` 환불 트리거** — **2xx 외엔 10초 × 3회 재시도** |

> `round_*` 콜백은 1회만 발송되며 리트라이가 없다. 누락 대비는 §5 REST API 폴링으로 보완할 것.
> `bet_confirmed` / `bet_canceled` 는 재시도되므로 매체사는 `order_id` UNIQUE 제약으로 멱등 처리할 것 (§3-7).

> **취소 이벤트 (`round_canceled` / `bet_canceled`) 의 의미**: 라운드가 정상 종료되지 않고 무효(환불) 처리됐다는 알림이다. 매체사는 `bet_request` 단계에서 차감해놓은 `media_amount` 를 다시 자사 DB 에 환불해야 한다. 트리거 조건은 다음과 같다:
> - **race condition**: 매체사 `bet_request` 승인 후 단타 서버 트랜잭션이 베팅 마감으로 실패한 경우 (단건 `bet_canceled` 만)
> - **engine 복구**: 라운드가 stuck 되어 무효 처리된 경우 (`round_canceled` + 참여자 모두에게 `bet_canceled`)
> - **운영 취소**: CMS 에서 게임을 비활성화한 사이 잔여 라운드가 있는 경우 (동일)
> - **orphan timeout**: bet_request 후 단타 서버 비정상 종료로 5분 이상 미확정 (`bet_canceled` `reason: "orphan_timeout"`)

### 3-7. 매체사 측 ORDER_ID 통합 가이드 (멱등성)

`bet_confirmed` / `bet_canceled` 가 재시도(10초 × 3회)되므로 같은 콜백이 2~3회 도착할 수 있다. 매체사 차감/환불이 이중 적용되지 않도록 `order_id` 기반 단일 키 매칭 + UNIQUE 제약을 둘 것.

#### 권장 스키마 (예시: MySQL)

```sql
ALTER TABLE PUBLIC_DANTA_USER_BET
  ADD COLUMN ORDER_ID VARCHAR(64) NULL,
  ADD UNIQUE INDEX uq_order_id (ORDER_ID);
-- MySQL 의 UNIQUE 는 다중 NULL 허용 — 이행기에 order_id 누락 row 가 있어도 충돌 X
```

#### bet_request 수신부

```sql
INSERT INTO PUBLIC_DANTA_USER_BET
       (ORDER_ID, APP_KEY, USER_ID, ROUND_ID, DIRECTION, BET_AMOUNT, MEDIA_BET_POINT, STATUS, ...)
VALUES (?, ?, ?, ?, ?, ?, ?, 'requested', ...)
ON DUPLICATE KEY UPDATE PID = PID;   -- 중복 콜백이면 no-op
```
→ 같은 `order_id` 가 재요청돼도 이중 차감 없음.

#### bet_confirmed / bet_canceled 수신부

```sql
-- 단일 키 매칭. bet_id / user_id 조합 매칭 불필요.
SELECT PID, STATUS FROM PUBLIC_DANTA_USER_BET WHERE ORDER_ID = ? LIMIT 1;

-- 이미 처리된 상태(예: STATUS='cancelled') 면 환불 건너뛰기 (멱등)
IF status IN ('settled', 'cancelled') THEN
  return 200 OK;
END IF;

UPDATE PUBLIC_DANTA_USER_BET
   SET STATUS = 'cancelled', SETTLED_AT = NOW()
 WHERE ORDER_ID = ?;
-- 환불 처리
```

#### 이행기 (점진 적용)

`order_id` 없이도 동작하도록 fallback 매칭을 유지하면 점진 배포가 가능하다 — 신규 콜백부터 `order_id` 가 항상 포함되므로, ORDER_ID 컬럼 정착 후엔 fallback 제거 권장.

```sql
-- 1차: order_id 매칭
SELECT ... WHERE ORDER_ID = ? LIMIT 1;

-- 2차 fallback (옛 콜백 호환)
SELECT ... WHERE APP_KEY=? AND USER_ID=? AND ROUND_ID=? AND DIRECTION=? AND BET_AMOUNT=? LIMIT 1;
```

---

## §4. 콜백 페이로드 상세

이벤트별 `data` 필드 스키마. 모든 페이로드는 §3-3 의 공통 봉투(`event_type` / `app_key` / `data` / `timestamp`) 안에 들어간다.

### 4-1. `round_started` — 라운드 시작

```json
{
  "event_type": "round_started",
  "app_key": "...",
  "data": {
    "game_id": 9,
    "game_title": "비트코인 단타",
    "game_type": "coin",
    "game_desc": "1분 단위 BTC 가격 예측",
    "round_id": 7007,
    "round_number": 1160
  },
  "timestamp": "2026-04-29T11:43:29.000Z"
}
```

### 4-2. `round_closed` — 베팅 마감

```json
{
  "event_type": "round_closed",
  "app_key": "...",
  "data": {
    "game_id": 9,
    "game_title": "...",
    "game_type": "coin",
    "game_desc": "...",
    "round_id": 7007,
    "round_number": 1160
  },
  "timestamp": "2026-04-29T11:48:00.000Z"
}
```

### 4-3. `round_ended` — 라운드 종료 + S3 정산 export

라운드 정산이 끝나고 매체별 결과 JSON 이 S3 에 업로드된 직후 발송. **참여자 0명 매체에도 동일하게 발송**되어 매체가 능동적으로 "참여 없음" 을 확인할 수 있다.

```json
{
  "event_type": "round_ended",
  "app_key": "...",
  "data": {
    "tr_id": "f903236b80e7cfb0200044d68cd31c21fbd92153fd120598",
    "game_id": 9,
    "game_title": "...",
    "game_type": "coin",
    "round_id": 7007,
    "round_number": 1160,
    "result": "down",
    "start_price": 113584000,
    "end_price": 113560000,
    "user_count": 4,
    "s3_url": "https://snap-service.s3.ap-northeast-2.amazonaws.com/danta/202604/games/9/rounds/7007/your_app_key.json"
  },
  "timestamp": "2026-04-29T11:48:00.000Z"
}
```

> S3 폴더의 `yyyymm` 부분은 **라운드 시작 시각의 KST(UTC+9) 기준** 이다 — 자정/월말을 걸쳐 종료되어도 시작 월 폴더에 일관 저장.

> **`result === "same"` (환불) 케이스**: `round_ended` 콜백으로도 환불 라운드가 통지될 수 있다 (예: 한쪽에만 베팅이 들어와 강제 무효 처리). 이때 S3 JSON 의 `round.result_reason` / `round.result_reason_label` 필드로 구체 사유를 확인할 수 있다 (§4-5 참조). UI 에서 "무효 처리" 안내 시 `result_reason_label` 을 그대로 노출 가능.

### 4-4. `round_canceled` — 라운드 무효 처리 + S3 환불 export

라운드 전체가 무효(환불) 처리되었음을 알리는 콜백. 채널은 `state_callback_url`. 매체별 환불 결과 JSON 이 S3 에 업로드된 직후 발송. **참여자 0명 매체에도 동일하게 발송**된다.

해당 라운드의 모든 베팅에 대해 별도로 `bet_canceled` 도 함께 발송되므로, 매체사는 보통 `bet_canceled` 로 단건 환불을 처리하고 `round_canceled` 는 라운드 단위 UI/통계 갱신에 사용한다.

```json
{
  "event_type": "round_canceled",
  "app_key": "...",
  "data": {
    "tr_id": "a4b1c8...",
    "game_id": 9,
    "game_title": "...",
    "game_type": "coin",
    "round_id": 7007,
    "round_number": 1160,
    "reason": "engine_recovery_stale",
    "user_count": 4,
    "s3_url": "https://snap-service.s3.ap-northeast-2.amazonaws.com/danta/202604/games/9/rounds/7007/your_app_key.json"
  },
  "timestamp": "2026-04-29T11:48:00.000Z"
}
```

S3 JSON 구조는 §4-5 정산 JSON 과 거의 동일하지만 다음이 다르다:

| 필드 | cancelled | settled (참고) |
|---|---|---|
| `status` | `"cancelled"` (신규 필드) | (없음) |
| `reason` | 취소 사유 (신규 필드, 콜백 envelope 와 동일 코드) | (없음) |
| `round.result` | `"void"` | `"up"` / `"down"` / `"same"` |
| `round.result_reason` | 사유 코드 (`"ENGINE_RECOVERY_STALE"` 등) | `null` 또는 환불 사유 코드 (예: `"ONE_SIDED_BET"`) |
| `round.result_reason_label` | 한글 안내 문구 | `null` 또는 환불 사유 안내 문구 |
| `users[].bet_result` | 모두 `"draw"` | `"win"` / `"lose"` / `"draw"` |
| `users[].payout_amount` | `bet_amount` 와 동일 (전액 환불) | 결과별 산정 |
| `summary.total_payout` | `total_bet` 과 동일 | 결과별 산정 |

라운드가 정상 종료되었는지 무효 처리되었는지는 **콜백의 `event_type` (`round_ended` vs `round_canceled`)** 또는 **S3 JSON 의 `status` 필드**로 구분 가능.

> **참고**: `result_reason` 은 settled S3 (라운드가 `round_ended` 로 끝난 경우) 에도 등장한다 — 정상 결과(up/down)/자연 same 일 때는 `null`, 환불 처리되었을 때만 사유 코드가 들어온다. 즉 매체사는 `round.result_reason` 이 `null` 이 아니면 "무효(환불) 라운드" 로 안전하게 판단할 수 있다.
>
> **`result_reason` 코드 표**:
>
> | 코드 | 의미 | 발생 시점 |
> |---|---|---|
> | (`null`) | 정상 결과 또는 자연 same (`start_price == end_price`) | 일반 라운드 |
> | `ONE_SIDED_BET` | 한쪽에만 베팅이 들어와 강제 무효 처리 (전액 환불) | `round_ended` (result=`same`) |
> | `INVALID_START_PRICE` | 베팅 마감 시점 기준가 캡처 실패 — 강제 무효 처리 | `round_ended` (result=`same`) |
> | `ENGINE_RECOVERY_STALE` / `ENGINE_RECOVERY_ORPHAN` / `STARTUP_STALE_CLEANUP` / `FORCE_FINISH_FAILED` / `INACTIVE_GAME_CLEANUP` | engine 복구 / 운영 취소 (정의는 §4-8 `bet_canceled` 사유와 동일) | `round_canceled` |

### 4-5. S3 정산 JSON 구조 (`{app_key}.json`)

```json
{
  "tr_id": "f903...",
  "generated_at": "2026-04-29T11:48:00.000Z",
  "game": {
    "game_id": 9,
    "game_title": "비트코인 단타",
    "game_type": "coin",
    "game_desc": "..."
  },
  "round": {
    "round_id": 7007,
    "round_number": 1160,
    "result": "down",
    "start_price": 113584000,
    "end_price": 113560000,
    "started_at": "2026-04-29T11:43:00.000Z",
    "settled_at": "2026-04-29T11:48:00.000Z"
  },
  "media": {
    "media_id": 3,
    "app_key": "your_app_key",
    "point_rate": 50000.0
  },
  "summary": {
    "user_count": 4,
    "total_bet": 4000,
    "total_payout": 3948,
    "media_total_bet": 0.08,
    "media_total_payout": 0.07896
  },
  "users": [
    {
      "bet_id": 9911,
      "user_id": "u_01",
      "direction": "up",
      "bet_amount": 1000,
      "payout_amount": 0,
      "media_bet_amount": 0.02,
      "media_payout": 0.0,
      "bet_result": "lose"
    },
    {
      "bet_id": 9912,
      "user_id": "u_42",
      "direction": "down",
      "bet_amount": 1000,
      "payout_amount": 1316,
      "media_bet_amount": 0.02,
      "media_payout": 0.02632,
      "bet_result": "win"
    }
  ]
}
```

> **계산 검증** (pari-mutuel — 진 쪽 풀에서만 5% margin 차감):
> - W (down 3명) = 3000원, L (up 1명) = 1000원
> - distributable = L × (1 - margin) = 1000 × 0.95 = **950원** (winners 가 나눠 가질 딴돈)
> - bonus per winner = bet × distributable / W = 1000 × 950 / 3000 = **316원** (floor)
> - each_payout = 1000 + 316 = **1316원** (= 0.02632P, ≈ 1.316배)
> - 운영 마진 = 1000 - (3 × 316) = **52원** (≈ L × margin = 50원, 정수 floor 차이로 +2)
> - total_payout = 3 × 1316 = **3948원** (= 0.07896P)

#### S3 정산 JSON — 필드 사전

**최상위:**

| 필드 | 타입 | 설명 |
|---|---|---|
| `tr_id` | string (48자 hex) | 정산 트랜잭션 고유 ID. 매체사 측 대사 / 멱등 키로 사용 |
| `generated_at` | ISO 8601 | 본 JSON 생성 시각 (UTC) |
| `status` | `"cancelled"` 또는 (없음) | `"cancelled"` 는 환불 라운드. settled (정상 종료) 는 필드 자체 없음 |
| `reason` | string 또는 (없음) | cancelled 일 때 envelope reason 코드 (`"engine_recovery_stale"` 등) |

**`game` 객체:**

| 필드 | 타입 | 설명 |
|---|---|---|
| `game_id` | integer | 단타 서버 내부 게임 PK. 매체사 식별자로 사용 (2026-05-15 부 game_code 폐지) |
| `game_title` | string | 게임명 (예: `"비트코인 단타"`) |
| `game_type` | string | 템플릿 코드 (`"coin"` / `"stock_ko"` / `"stock_us"` / `"usd_krw"` / `"kospi"` / `"sp500"`) |
| `game_desc` | string | 게임 설명 (운영팀 작성) |

**`round` 객체:**

| 필드 | 타입 | 설명 |
|---|---|---|
| `round_id` | integer | 라운드 PK |
| `round_number` | integer | 게임 내 회차 번호 (1부터 증가) |
| `result` | `"up"` / `"down"` / `"same"` / `"void"` | 결과. `"same"` = 환불, `"void"` = cancelled 라운드 |
| `result_reason` | string 또는 `null` | 환불 사유 코드 (UPPER_SNAKE). 정상 결과는 `null`. 예: `"ONE_SIDED_BET"`, `"INVALID_START_PRICE"`, `"ENGINE_RECOVERY_STALE"` |
| `result_reason_label` | string 또는 `null` | 한글 안내 문구 (예: `"한쪽에만 베팅이 있어 무효 처리되었습니다"`). UI 에 그대로 노출 가능 |
| `start_price` | number | 베팅 마감 시점의 기준 가격 (원/포인트, 코인은 KRW, 환율은 KRW per USD 등) |
| `end_price` | number | 라운드 종료 시점의 가격 |
| `started_at` | ISO 8601 | 라운드 시작 (open) 시각 |
| `settled_at` | ISO 8601 | 정산 (포인트 지급) 완료 시각 |

**`media` 객체:**

| 필드 | 타입 | 설명 |
|---|---|---|
| `media_id` | integer | 매체 PK |
| `app_key` | string | 매체 인증 키 |
| `point_rate` | number | 매체 포인트 환율. `1 포인트 = point_rate 원` (예: 50000 = 1P 가 50,000원 가치) |

**`summary` 객체:**

| 필드 | 단위 | 설명 |
|---|---|---|
| `user_count` | 정수 | 본 매체의 베팅 row 수 (한 유저가 N번 베팅하면 N으로 카운트) |
| `total_bet` | 원(KRW) | 본 매체의 총 베팅 합계 (단타 서버 내부 통화) |
| `total_payout` | 원(KRW) | 본 매체의 총 지급 합계 |
| `media_total_bet` | 매체 포인트 | `total_bet ÷ point_rate` 환산값 |
| `media_total_payout` | 매체 포인트 | `total_payout ÷ point_rate` 환산값 |

**`users[]` 배열 (베팅 단위, win + lose + draw 모두 포함):**

| 필드 | 타입 | 설명 |
|---|---|---|
| `bet_id` | integer | 베팅 PK (정산 단위) |
| `user_id` | string | 매체사가 부여한 유저 식별자. 한 유저가 N번 베팅하면 N개 entry 로 등장 |
| `direction` | `"up"` / `"down"` | 베팅 방향 |
| `bet_amount` | 원(KRW) | 베팅 금액 (단타 내부 통화) |
| `payout_amount` | 원(KRW) | 지급 금액. lose=0, win=bet+딴돈, draw=bet (환불) |
| `media_bet_amount` | 매체 포인트 | `bet_amount ÷ point_rate` |
| `media_payout` | 매체 포인트 | `payout_amount ÷ point_rate` |
| `bet_result` | `"win"` / `"lose"` / `"draw"` | `draw` 는 환불 (same 또는 환불 라운드) |

> **참고**: `users[]` 는 매체별 정산 export 라 본 매체의 베팅만 포함. 다른 매체 데이터 노출 X.

> **`bet_id` 단위 처리**: 한 유저가 같은 라운드에 여러 번 베팅한 경우 entry 가 N개로 분리됨. 매체사는 `bet_id` 기준으로 환불/지급 처리해야 하며, `user_id` 로 GROUP BY 하면 안 됨 (베팅 단위가 정산 단위).

### 4-6. `bet_request` — 베팅 동기 승인

매체사가 포인트 원장을 보유하므로, 베팅이 발생할 때마다 **단타 서버가 매체 서버에게 동기 승인을 요청**한다. 채널은 `bet_callback_url`.

```json
{
  "event_type": "bet_request",
  "app_key": "...",
  "data": {
    "order_id": "dt_202605_a3f7b9c2e1d4f8a6b3c5e2d1f7a4b9c2e1d4f8a6b3c5e2d1",
    "request_ts": "2026-04-29T11:43:50.000Z",
    "game_id": 9,
    "game_title": "...",
    "game_type": "coin",
    "game_desc": "...",
    "round_id": 7007,
    "round_number": 1160,
    "user_id": "u_42",
    "direction": "up",
    "amount": 1000,
    "point_rate": 50000.0,
    "media_amount": 0.02
  },
  "timestamp": "2026-04-29T11:43:50.000Z"
}
```

#### 매체사 응답 (필수)

```json
{ "approved": true }
```
또는 거부:
```json
{ "approved": false, "reason": "포인트 부족" }
```

> 응답 바디의 `approved` 필드가 `true` 가 아니면 단타 서버는 **403** 으로 베팅을 거부한다.
> 매체사는 이 시점에 자사 DB 에서 `media_amount` 만큼 포인트를 차감해야 하며, `order_id` 를 UNIQUE 키로 저장해야 한다 (§6-1). 같은 `order_id` 가 재전송되면 ON CONFLICT 로 무시 → 멱등 차감.

### 4-7. `bet_confirmed` — 베팅 확정 알림 (비동기)

`bet_request` 승인 후 단타 서버가 트랜잭션을 커밋한 직후 발송. 채널은 `bet_callback_url`. **HTTP 2xx 외엔 10초 간격으로 최대 3회 재시도**.

```json
{
  "event_type": "bet_confirmed",
  "app_key": "...",
  "data": {
    "order_id": "dt_202605_a3f7b9c2e1d4f8a6b3c5e2d1f7a4b9c2e1d4f8a6b3c5e2d1",
    "request_ts": "2026-04-29T11:43:50.000Z",
    "game_id": 9,
    "game_title": "...",
    "game_type": "coin",
    "game_desc": "...",
    "round_id": 7007,
    "round_number": 1160,
    "user_id": "u_42",
    "bet_id": 9912,
    "direction": "up",
    "amount": 1000,
    "point_rate": 50000.0,
    "media_amount": 0.02
  },
  "timestamp": "2026-04-29T11:43:51.000Z"
}
```

> `order_id` 는 `bet_request` 와 동일한 값 — 매체사는 `WHERE order_id=?` 로 차감 row 의 상태를 `confirmed` 로 업데이트하면 된다. 재시도 시 같은 페이로드가 다시 도착할 수 있으나 멱등 처리 권장.

### 4-8. `bet_canceled` — 베팅 환불 트리거 (비동기)

라운드/베팅이 무효 처리되어 매체사가 자사 DB 의 `media_amount` 를 환불해야 함을 알리는 콜백. 채널은 `bet_callback_url`. **HTTP 2xx 외엔 10초 간격으로 최대 3회 재시도**.

발송 시점:
- **race condition** — 매체사 `bet_request` 승인 후 단타 서버 트랜잭션이 베팅 마감(`status='closed'`)으로 실패 (`reason: "round_closed"`)
- **engine 복구** — 라운드가 stuck 되어 무효 처리 (`reason: "engine_recovery_stale"` / `"engine_recovery_orphan"` / `"startup_stale_cleanup"` / `"force_finish_failed"`)
- **운영 취소** — CMS 에서 게임 비활성화 사이 잔여 라운드 (`reason: "inactive_game_cleanup"`)
- **orphan timeout** — bet_request 후 단타 서버 비정상 종료로 5분 이상 미확정 상태가 된 경우 (`reason: "orphan_timeout"`)

```json
{
  "event_type": "bet_canceled",
  "app_key": "...",
  "data": {
    "order_id": "dt_202605_a3f7b9c2e1d4f8a6b3c5e2d1f7a4b9c2e1d4f8a6b3c5e2d1",
    "game_id": 9,
    "game_title": "...",
    "game_type": "coin",
    "game_desc": "...",
    "round_id": 7007,
    "round_number": 1160,
    "user_id": "u_42",
    "bet_id": 9912,
    "direction": "up",
    "amount": 1000,
    "point_rate": 50000.0,
    "media_amount": 0.02,
    "reason": "round_closed"
  },
  "timestamp": "2026-04-29T11:43:51.000Z"
}
```

> **매칭은 `order_id` 단일 키로** — `bet_id` 는 race rejection 케이스에서 `null` 일 수 있다 (단타 서버에 베팅 INSERT 자체가 안 된 경우). `order_id` 는 항상 존재하며 `bet_request` 와 동일한 값이므로 매체사는 `WHERE order_id=?` 만으로 차감 row 를 찾을 수 있다.
> 매체사는 이 콜백 수신 시 자사 DB 에서 해당 유저에게 `media_amount` 를 환불해야 한다 (이전에 `bet_request` 승인 시 차감된 양과 동일). 재시도에 대비해 row 의 상태가 이미 `cancelled` 면 재처리하지 않도록 멱등성 보장.

### 4-9. 콜백 페이로드 공통 필드 사전

#### 모든 콜백 envelope 공통

| 필드 | 타입 | 설명 |
|---|---|---|
| `event_type` | string | `"round_started"` / `"round_closed"` / `"round_ended"` / `"round_canceled"` / `"bet_request"` / `"bet_confirmed"` / `"bet_canceled"` |
| `app_key` | string | 매체 인증 키 |
| `data` | object | 이벤트별 페이로드 (아래 표) |
| `timestamp` | ISO 8601 | 단타 서버 발송 시각 (UTC) |

#### `round_*` 콜백 (4개) `data` 필드

`round_started` / `round_closed` / `round_ended` / `round_canceled` 공통 base 필드:

| 필드 | 타입 | 발송 콜백 | 설명 |
|---|---|---|---|
| `game_id` | integer | 모두 | 게임 PK |
| `game_title` | string | 모두 | 게임명 |
| `game_type` | string | 모두 | 템플릿 코드 (`"coin"` 등) |
| `game_desc` | string | started/closed | 게임 설명 (ended/canceled 에서는 생략) |
| `round_id` | integer | 모두 | 라운드 PK |
| `round_number` | integer | 모두 | 회차 번호 |

`round_ended` / `round_canceled` 추가 필드:

| 필드 | 타입 | 발송 콜백 | 설명 |
|---|---|---|---|
| `tr_id` | string (48자 hex) | ended/canceled | 정산 트랜잭션 ID — S3 JSON 의 `tr_id` 와 동일 |
| `result` | `"up"` / `"down"` / `"same"` | ended | 라운드 결과. `"same"` = 환불 (자연 same 또는 강제 환불) |
| `start_price` | number | ended | 베팅 마감 시 기준가 |
| `end_price` | number | ended | 라운드 종료 시 가격 |
| `reason` | string | canceled | 취소 사유 코드 (`"engine_recovery_stale"` / `"inactive_game_cleanup"` 등) |
| `user_count` | integer | ended/canceled | 본 매체 베팅 row 수 (0 인 경우도 발송됨) |
| `s3_url` | string (URL) | ended/canceled | S3 정산/취소 JSON 다운로드 URL |

> `round_ended` 의 result=`"same"` + `s3_url` 의 round.result_reason 필드로 환불 사유 (`"ONE_SIDED_BET"` 등) 구분 가능.

#### `bet_*` 콜백 (3개) `data` 필드

`bet_request` / `bet_confirmed` / `bet_canceled` 공통 base 필드:

| 필드 | 타입 | 발송 콜백 | 설명 |
|---|---|---|---|
| `order_id` | string(58) | 모두 | **매칭 단일 키**. `dt_<yyyymm>_<48hex>` 형태 HMAC 해시 — 매체사 차감 row 의 UNIQUE 키로 사용 (§6-1) |
| `request_ts` | ISO 8601 | request / confirmed | 단타 서버가 `bet_request` 를 발송한 시각 (envelope `timestamp` 와 별개로 `data` 안에 명시 — 매체 측 replay/감사용) |
| `game_id` | integer | 모두 | 게임 PK |
| `game_title` | string | 모두 | 게임명 |
| `game_type` | string | 모두 | 템플릿 코드 |
| `game_desc` | string | 모두 | 게임 설명 |
| `round_id` | integer | 모두 | 라운드 PK |
| `round_number` | integer | 모두 | 회차 번호 (사람 친화 표시용) |
| `user_id` | string | 모두 | 매체 측 유저 식별자 |
| `direction` | `"up"` / `"down"` | 모두 | 베팅 방향 |
| `amount` | 원(KRW) | 모두 | 베팅 금액 (단타 서버 내부 통화) |
| `point_rate` | number | 모두 | 매체 환율 (1P = N 원) |
| `media_amount` | 매체 포인트 | 모두 | `amount ÷ point_rate` (매체사 포인트 차감/환불 단위) |

`bet_confirmed` / `bet_canceled` 추가 필드:

| 필드 | 타입 | 발송 콜백 | 설명 |
|---|---|---|---|
| `bet_id` | integer | confirmed (항상) / canceled (대부분) | 단타 서버 베팅 PK. `bet_canceled` 의 race rejection 케이스에서는 `null` — 이 때도 `order_id` 로 매칭 가능 |
| `reason` | string | canceled | 취소 사유 (`"round_closed"` / `"engine_recovery_stale"` / `"inactive_game_cleanup"` / `"orphan_timeout"` 등) |

#### `bet_request` 응답 스키마 (매체사 → 단타 서버)

| 필드 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `approved` | boolean | ✓ | `true` = 승인 (베팅 진행), `false` = 거부 (단타 서버 403 응답) |
| `reason` | string | 거부 시 권장 | 거부 사유 (예: `"포인트 부족"` / `"한도 초과"`). 단타 서버 로그/모니터링용 |

---

## §5. REST API

매체사 운영용 API. 두 prefix 모두 동일하게 동작한다.

- **권장**: `/api/pub/*`
- **호환 alias**: `/api/media/*`

### 5-1. 인증

모든 API 는 `app_key` 검증을 거친다. 아래 셋 중 **하나면 충분**.

| 위치 | 키 |
|---|---|
| HTTP 헤더 | `X-App-Key: {app_key}` |
| 쿼리 | `?app_key={app_key}` |
| 바디 (POST) | `{ "app_key": "..." }` |

| 상태 코드 | 의미 |
|---|---|
| 401 | `app_key` 누락 또는 무효 |
| 403 | `is_active = false` 인 매체 |

### 5-2. 응답 컨벤션

성공:
```json
{ "result": 1, "data": { ... } }
```
실패:
```json
{ "result": 0, "message": "...", "code": "OPTIONAL" }
```

### 5-3. 엔드포인트 일람

| Method | Path | 설명 |
|---|---|---|
| `GET` | `/api/pub/me` | 인증된 매체사 본인 정보 (`callback_secret` 제외) |
| `GET` | `/api/pub/games/active` | 진행중 게임 리스트 |
| `GET` | `/api/pub/games/recent?limit=50` | 최근 종료된 라운드 (전체 게임) |
| `GET` | `/api/pub/games/:gameId?historyLimit=50` | 게임 상세 (현재 라운드 + 최근 회차) |
| `GET` | `/api/pub/games/:gameId/rounds?page=1&size=50` | 게임별 라운드 히스토리 (페이지네이션) |
| `GET` | `/api/pub/games/:gameId/rounds/:roundId` | 회차 단건 (S3 정산 메타 포함) |
| `GET` | `/api/pub/games/:gameId/users/:userId/bets?page=1&size=50` | 매체 본인의 특정 유저-게임 베팅 이력 |
| `GET` | `/api/pub/rounds/:roundId` | 회차 단건 (`gameId` 불필요) |

### 5-4. `GET /api/pub/me`

```json
{
  "result": 1,
  "data": {
    "media_id": 3,
    "media_name": "Carry Gold",
    "client_id": "carrygold",
    "app_key": "...",
    "point_rate": 50000.0,
    "state_callback_url": "https://your.domain/dt/state",
    "bet_callback_url":   "https://your.domain/dt/bet",
    "is_active": true
  }
}
```

### 5-5. `GET /api/pub/games/active`

> 2026-05-15 부 모든 status (active / inactive / paused / ended) 게임 노출. 매체사 frontend 가 `status` 별 필터링 책임.

```json
{
  "result": 1,
  "data": [
    {
      "game_id": 9,
      "title": "비트코인 단타",
      "symbol": "KRW-BTC",
      "symbol_name": "비트코인",
      "icon_url": "https://...",
      "banner_url": null,
      "thumbnail_url": null,
      "status": "active",
      "current_round_id": 7007,
      "total_rounds": 1160,
      "started_at": "2026-04-28T05:51:55.657Z",
      "created_at": "2026-04-28T05:51:55.657Z",
      "round_duration": 300,
      "bet_cutoff": 30,
      "schedule_type": "24h",
      "margin": 0.05,
      "bet_amount": 1000,
      "template_code": "coin",
      "template_name": "코인",
      "market_open_time": "",
      "market_close_time": "",
      "market_open": true,
      "market_opens_at": null,
      "market_closes_at": null,
      "next_round_at": null,
      "market_status": {
        "schedule_type": "24h",
        "is_open": true,
        "next_open_at": null,
        "next_close_at": null
      },
      "currentRound": {
        "round_id": 7007,
        "round_number": 1160,
        "status": "open",
        "started_at": "2026-04-29T11:43:00.000Z",
        "bet_closed_at": null,
        "ended_at": null,
        "start_price": null,
        "end_price": null,
        "result_direction": null,
        "result_reason": null,
        "result_reason_label": null,
        "total_players": 0,
        "up_players": 0,
        "down_players": 0,
        "total_bet_amount": 0,
        "up_bet_amount": 0,
        "down_bet_amount": 0
      }
    }
  ]
}
```

#### 응답 필드 사전 — game (top-level)

| 필드 | 타입 | 설명 |
|---|---|---|
| `game_id` | integer | 게임 PK |
| `title` | string | 게임명 |
| `symbol` | string | 데이터 source 심볼 (`"KRW-BTC"` / `"005930"` / `"NVDA"` / `"SPY"` / `"KOSPI"` / `"KRW"` 등) |
| `symbol_name` | string | 사람이 읽는 이름 (`"비트코인"` / `"삼성전자"` 등) |
| `icon_url` / `banner_url` / `thumbnail_url` | string (URL) 또는 `null` | 게임 자산 이미지 (S3 절대 URL) |
| `status` | `"active"` / `"inactive"` / `"paused"` / `"ended"` | 게임 운영 상태. 매체사 frontend 가 필터링 |
| `current_round_id` | integer 또는 `null` | 현재 진행중 라운드 PK (참고용 — 본 응답의 `currentRound.round_id` 와 동일) |
| `total_rounds` | integer | 누적 회차 수 |
| `started_at` | ISO 8601 | 게임 첫 시작 시각 |
| `created_at` | ISO 8601 | DB row 생성 시각 |
| `round_duration` | integer (초) | 1 라운드 길이 (코인=60, 주식=3600, 코스피=86400, S&P500=300 등) |
| `bet_cutoff` | integer (초) | 라운드 종료 N초 전 베팅 마감 |
| `schedule_type` | `"24h"` / `"kr_market"` / `"us_market"` | 운영 스케줄 종류 |
| `margin` | decimal (소수) | 하우스 마진 (예: 0.05 = 5%) |
| `bet_amount` | 원(KRW) | 게임당 고정 판돈 (단타 서버 내부 통화) |
| `template_code` | string | 게임 템플릿 (`"coin"` / `"stock_ko"` / `"stock_us"` / `"usd_krw"` / `"kospi"` / `"sp500"`) |
| `template_name` | string | 템플릿 한글명 |
| `market_open_time` / `market_close_time` | string (`"HH:MM"`) | 시장 영업 시간 (24h 마켓은 빈 문자열) |
| `market_open` | boolean | 현재 시장 영업 여부 (코인/환율은 항상 true) |
| `market_opens_at` / `market_closes_at` | ISO 8601 또는 `null` | 다음 개장/마감 시각. 24h 는 `null` |
| `next_round_at` | ISO 8601 또는 `null` | 다음 라운드 시작 시각 (시장 닫혔거나 라운드 사이 갭일 때) |
| `market_status` | object | nested 호환 객체 (위 4개 필드와 동일 정보, 구버전 매체사 호환) |

#### 응답 필드 사전 — `currentRound` 객체

| 필드 | 타입 | 설명 |
|---|---|---|
| `round_id` | integer 또는 `null` | 라운드 PK. 라운드 시작 전이면 `null` |
| `round_number` | integer | 게임 내 회차 번호 |
| `status` | `"open"` / `"closed"` / `"finished"` / `"settled"` / `"cancelled"` | 라운드 상태. open=베팅 가능, closed=마감, finished=결과 결정, settled=정산 완료, cancelled=무효 |
| `started_at` | ISO 8601 | 라운드 시작 시각 |
| `bet_closed_at` | ISO 8601 또는 `null` | 베팅 마감 시각 (closed 전환 시점) |
| `ended_at` | ISO 8601 또는 `null` | 라운드 종료 시각 (결과 결정 시점) |
| `start_price` | number 또는 `null` | 베팅 마감 시 기준가. open 단계에선 `null` |
| `end_price` | number 또는 `null` | 라운드 종료 시 가격 |
| `result_direction` | `"up"` / `"down"` / `"same"` 또는 `null` | 결과 방향. `"same"` = 환불 |
| `result_reason` | string (UPPER_SNAKE) 또는 `null` | 환불 사유 코드. 정상 결과는 `null`. (예: `"ONE_SIDED_BET"`) |
| `result_reason_label` | string 또는 `null` | 한글 안내 문구 |
| `total_players` / `up_players` / `down_players` | integer | 베팅 row 수 (한 유저 N번이면 N으로 카운트) |
| `total_bet_amount` / `up_bet_amount` / `down_bet_amount` | 원(KRW) | 누적 베팅 합계 (단타 내부 통화) |

> **`market_status` 필드 설명** (2026-05-11 추가)
>
> - 게임의 데이터 소스 시장이 현재 거래 가능한지 알림
> - `schedule_type` — `24h` (코인/환율) / `kr_market` (한국주식 09:00~15:30 KST) / `us_market` (미국주식 09:30~16:00 ET, 서머타임 자동)
> - `is_open` — 현재 시장 영업 여부 (휴장일 자동 제외)
> - `next_open_at` — 닫혀있을 때 다음 개장 시각 (ISO 8601). 열려있으면 `null`
> - `next_close_at` — 열려있을 때 오늘 마감 시각 (ISO 8601). 24h 마켓이면 `null`
> - **`is_open=false` 면 `current_round` 도 거의 없습니다** — 매체사 UI 에서 "현재 LIVE 아님" 같은 안내 + 다음 개장 시각 표시 권장

> **정산 모델 — pari-mutuel ("원금 + 딴돈", 진 쪽 풀에서만 margin 차감)** (2026-05-12 부 fixed-odds 폐기)
>
> - **`margin`** — 하우스 마진을 **소수**로 표기. 예: `0.05` = 5%
> - 이긴 사람이 받는 액수 = **원금 + 딴돈**
>   - 원금 = 본인 베팅 (그대로 돌려받음)
>   - 딴돈 = 진 쪽 풀의 (1 − margin) 을 이긴 쪽 베팅 비율로 분배
> - **마진은 "딴돈(진 쪽 풀)의 5%" 만 차감** — 이긴 쪽 풀은 절대 손대지 않음
> - ⇒ **이긴 사람은 항상 ≥ 본인 베팅 보장** (원금 손실 X)
> - ⇒ 운영 손익 = `L × margin` (진 쪽 풀에서만, 이긴 쪽 적게 걸렸어도 운영 손실 X)
>
> **공식**:
> ```
>   W = 이긴 쪽 총 베팅 (winners pool)
>   L = 진 쪽 총 베팅 (losers pool)
>   distributable = L × (1 - margin)              // 진 쪽 풀의 (1 - margin) = 분배 가능 딴돈
>   bonus         = bet × distributable / W       // 이 사람이 받는 딴돈
>   each_payout   = bet + bonus                   // 원금 + 딴돈
> ```
>
> **양방향 예상 배당 배수** (pool 응답에서 동적 제공):
>
> "1단위 베팅 → 정산 시 받게 될 총액" (원금 포함). 즉 `each_payout = bet × multiplier`.
> ```
>   up_payout_multiplier   = 1 + down_bet × (1 - margin) / up_bet     (up 이 이겼을 때)
>   down_payout_multiplier = 1 + up_bet   × (1 - margin) / down_bet   (down 이 이겼을 때)
> ```
> - 한쪽 풀이 0 이면 해당 방향 multiplier 는 `null` (베팅 없음 → 배당 정의 불가). UI 는 "−" 처리 권장.
> - **베팅 추가될 때마다 multiplier 가 변동**. 매체사 UI 는 `pool_update` WS / 또는 폴링으로 최신 값 사용.
> - 정산 시점의 multiplier 가 **실제 적용**됨 (베팅한 시점 multiplier 와 다를 수 있음 — pari-mutuel 의 본질).
>
> **예시 — `margin = 0.05`, up 0.6 (3명 × 0.2) / down 0.4 (2명 × 0.2) / down 이김**:
> ```
>   distributable = 0.6 × (1 - 0.05) = 0.57          (진 쪽 풀에서 5% 차감)
>   down 베터 1명: bonus = 0.2 × 0.57 / 0.4 = 0.285
>                  each_payout = 0.2 + 0.285 = 0.485   (원금 + 딴돈)
>   운영 마진 = 0.6 × 0.05 = 0.03                     (= L × margin, "딴돈의 5%")
>   풀 검증: 2 × 0.485 + 0.03 = 1.0 ✓
> ```
>
> **극단 케이스 — `margin = 0.05`, up 1.0 / down 0.01 (한쪽 풀 100배 차)**:
> ```
>   up 베터 (1.0 베팅, up 이김): each = 1.0 × (1 + 0.01 × 0.95 / 1.0) = 1.0095
>   ✓ 본인 베팅 1.0 보다 항상 ≥ 받음 (원금 손실 절대 X)
> ```
>
> **deprecated**: `pool.payout_multiplier` 단일 필드는 항상 `null` 반환. 매체사는 `up_payout_multiplier` / `down_payout_multiplier` 양방향 사용 필수.

#### `pool` 객체 필드 사전 (게임 상세 응답 / pool_update WS 메시지 공통)

| 필드 | 타입 | 설명 |
|---|---|---|
| `total_bet_amount` | 원(KRW) | 라운드 누적 총 베팅 |
| `up_bet_amount` / `down_bet_amount` | 원(KRW) | 방향별 누적 베팅 |
| `total_players` / `up_players` / `down_players` | integer | 방향별 베팅 row 수 |
| `margin` | decimal | 하우스 마진 (예: 0.05) |
| `up_payout_multiplier` | decimal 또는 `null` | up 이 이겼을 때 1단위당 받을 총액 (원금 포함). `1 + down_bet × (1 - margin) / up_bet`. up_bet=0 이면 `null` |
| `down_payout_multiplier` | decimal 또는 `null` | down 이 이겼을 때 동일 공식. down_bet=0 이면 `null` |
| `payout_multiplier` | `null` (deprecated) | 양방향 동일 가정 폐기. 항상 `null` 반환 — 매체사는 위 양방향 multiplier 사용 |

### 5-6. `GET /api/pub/games/recent?limit=50`

```json
{
  "result": 1,
  "data": [
    {
      "round_id": 7006,
      "game_id": 9,
      "round_number": 1159,
      "status": "settled",
      "result_direction": "down",
      "start_price": 113584000,
      "end_price": 113560000,
      "started_at": "2026-04-29T11:38:00.000Z",
      "ended_at": "2026-04-29T11:43:00.000Z"
    }
  ]
}
```

### 5-7. `GET /api/pub/games/:gameId/rounds?page=1&size=50`

```json
{
  "result": 1,
  "data": {
    "game": { "game_id": 9, "title": "...", "template_code": "coin" },
    "page": 1,
    "size": 50,
    "total": 1160,
    "rounds": [
      {
        "round_id": 7006, "round_number": 1159, "status": "settled",
        "started_at": "...", "bet_closed_at": "...", "ended_at": "...", "settled_at": "...",
        "start_price": 113584000, "end_price": 113560000, "result_direction": "down",
        "total_players": 4, "up_players": 1, "down_players": 3,
        "total_bet_amount": 4000, "up_bet_amount": 1000, "down_bet_amount": 3000
      }
    ]
  }
}
```

### 5-8. `GET /api/pub/games/:gameId/rounds/:roundId`

응답에 **S3 정산 메타** (`settlement.s3_url` 등) 가 포함되어, 매체사가 콜백을 놓친 경우에도 동일한 데이터를 직접 가져올 수 있다.

```json
{
  "result": 1,
  "data": {
    "round_id": 7006,
    "round_number": 1159,
    "game_id": 9,
    "status": "settled",
    "result": "down",
    "result_reason": null,
    "result_reason_label": null,
    "start_price": 113584000,
    "end_price": 113560000,
    "started_at": "...",
    "ended_at": "...",
    "settled_at": "...",
    "point_rate": 50000.0,
    "settlement": {
      "tr_id": "f903...",
      "s3_url": "https://snap-service.s3.../202604/games/9/rounds/7006/your_app_key.json",
      "s3_key": "danta/202604/games/9/rounds/7006/your_app_key.json",
      "s3_bucket": "snap-service",
      "user_count": 4,
      "total_bet": 4000,
      "total_payout": 3948,
      "media_total_bet": 0.08,
      "media_total_payout": 0.07896,
      "created_at": "..."
    },
    "results": [
      {
        "bet_id": 9911, "user_id": "u_01", "direction": "up",
        "bet_amount": 1000, "payout_amount": 0, "bet_result": "lose",
        "media_bet_amount": 0.02, "media_payout": 0.0, "settled_at": "..."
      },
      {
        "bet_id": 9912, "user_id": "u_42", "direction": "down",
        "bet_amount": 1000, "payout_amount": 1316, "bet_result": "win",
        "media_bet_amount": 0.02, "media_payout": 0.02632, "settled_at": "..."
      }
    ]
  }
}
```

> `settlement` 가 `null` 이면 아직 정산 export 가 완료되지 않은 상태.

#### 응답 필드 사전 — round detail (top-level)

| 필드 | 타입 | 설명 |
|---|---|---|
| `round_id` / `round_number` / `game_id` | integer | 식별자 |
| `status` | `"open"` / `"closed"` / `"finished"` / `"settled"` / `"cancelled"` | 라운드 상태 |
| `result` | `"up"` / `"down"` / `"same"` 또는 `null` | 결과 방향 |
| `result_reason` | string (UPPER_SNAKE) 또는 `null` | 환불 사유 코드. 정상 결과는 `null` |
| `result_reason_label` | string 또는 `null` | 한글 안내 문구 |
| `start_price` / `end_price` | number | 기준가 / 종료가 |
| `started_at` / `ended_at` / `settled_at` | ISO 8601 | 라운드 라이프사이클 시각 |
| `point_rate` | decimal | 호출자 매체의 환율 (1P = N 원) |
| `settlement` | object 또는 `null` | S3 정산 export 메타. 정산 진행 중이면 `null` |
| `results[]` | array | 본 매체의 유저별 베팅 결과 (베팅 row 단위) |

#### `settlement` 객체 필드 사전

| 필드 | 타입 | 설명 |
|---|---|---|
| `tr_id` | string (48자 hex) | 정산 트랜잭션 ID. callback `round_ended` 의 `tr_id` 와 동일 |
| `s3_url` | string (URL) | S3 정산 JSON 다운로드 URL |
| `s3_key` | string | S3 객체 key (`/YYYYMM/games/{game_id}/rounds/{round_id}/{media_id}.json`) |
| `s3_bucket` | string | S3 버킷 (`"snap-service"`) |
| `user_count` | integer | 본 매체 베팅 row 수 |
| `total_bet` / `total_payout` | 원(KRW) | 본 매체 총 베팅 / 총 지급 (단타 내부 통화) |
| `media_total_bet` / `media_total_payout` | 매체 포인트 | `÷ point_rate` 환산값 |
| `created_at` | ISO 8601 | 정산 export 완료 시각 |

#### `results[]` 배열 필드 사전

| 필드 | 타입 | 설명 |
|---|---|---|
| `bet_id` | integer | 베팅 PK (정산 단위) |
| `user_id` | string | 매체 측 유저 식별자 |
| `direction` | `"up"` / `"down"` | 베팅 방향 |
| `bet_amount` | 원(KRW) | 베팅 금액 |
| `payout_amount` | 원(KRW) | 지급 금액. lose=0, win=bet+딴돈, draw=bet (환불) |
| `bet_result` | `"win"` / `"lose"` / `"draw"` | 결과 |
| `media_bet_amount` | 매체 포인트 | `bet_amount ÷ point_rate` |
| `media_payout` | 매체 포인트 | `payout_amount ÷ point_rate` |
| `settled_at` | ISO 8601 | 정산 완료 시각 |

### 5-9. `GET /api/pub/games/:gameId/users/:userId/bets?page=1&size=50`

매체 본인이 등록한 유저(=`(client_id, app_key, user_id)`) 의 베팅만 노출한다. 다른 매체 데이터는 절대 노출되지 않는다.

---

## §6. 점검 체크리스트 (인티그레이션 완료 전)

- [ ] `app_key` / `callback_secret` 발급 완료, 안전한 위치에 저장
- [ ] HMAC-SHA256 서명 검증 로직 구현 (모든 콜백)
- [ ] `state_callback_url` — `round_started` / `round_closed` / `round_ended` / `round_canceled` 수신 처리
- [ ] `bet_callback_url` — `bet_request` 동기 승인 (`{approved}` 응답)
- [ ] `bet_confirmed` 수신 시 자사 DB 와 일치 확인
- [ ] **`bet_canceled` 수신 시 자사 DB 에서 `media_amount` 환불 처리** (이전 `bet_request` 차감의 역연산)
- [ ] **`round_canceled` 수신 시 라운드 단위 UI/통계 갱신** (해당 라운드 결과 표시 = "무효")
- [ ] 매체사 사이트의 진입 링크에 `?user_id=...&app_key=...` 쿼리 파라미터 부착
- [ ] 콜백 endpoint 가 외부에서 접근 가능한지 (방화벽/whitelist) 확인
- [ ] 콜백은 1회만 발송되므로 누락 대비 폴링 로직 준비
- [ ] `round_ended` / `round_canceled` 누락 대비: 주기적으로 [§5-7 라운드 히스토리](#5-7-get-apipubgamesgameidroundspage1size50) 또는 [§5-8 회차 단건](#5-8-get-apipubgamesgameidroundsroundid) API 폴링
- [ ] S3 JSON 파싱 로직: `users[]` 배열 + `summary` 집계 + **`status` 필드로 settled vs cancelled 구분**
- [ ] `result === 'same'` (환불) 케이스 매체 UI/포인트 처리

---

## §7. FAQ / 주의사항

**Q. `app_key` 가 노출되어도 되나요?**
A. URL 쿼리/HTTP 헤더로 평문 전달되므로 외부 노출 가능. 단순 식별자 역할이며, 인증의 핵심은 `callback_secret` 입니다.

**Q. 콜백 URL 이 응답 안 하면 어떻게 되나요?**
A. 10초 타임아웃으로 실패 처리. **`bet_confirmed` / `bet_canceled` 는 10초 간격으로 최대 3회 재시도** (2026-05-18~). 그 외 `round_*` 이벤트는 1회만 발송되니 누락 대비는 §5 REST API 폴링으로 보완하세요. 재시도 멱등 처리 가이드는 §3-7 참조.

**Q. `round_ended` 가 안 와요.**
A. 다음을 확인하세요:
1. `state_callback_url` 이 정확한지
2. 응답이 `2xx` 였는지
3. 매체가 `is_active = true` 인지

누락된 경우 [§5-8](#5-8-get-apipubgamesgameidroundsroundid) 으로 동일 데이터 회수가 가능합니다.

**Q. `start_price` 가 `null` 인 라운드가 있어요.**
A. 라운드가 `open` 단계 (베팅 받는 중) — `start_price` 는 베팅 마감 시점(`closed`) 에 캡처됩니다.

**Q. `result === 'same'` 의 의미는?**
A. 시작가와 종료가가 동일 → **환불 처리**. 모든 베팅이 `bet_result = 'draw'` 가 되며 `payout_amount === bet_amount`.

**Q. 환불 라운드의 구체 사유를 유저에게 안내하고 싶어요.**
A. S3 JSON 의 `round.result_reason_label` 필드를 그대로 노출하면 됩니다 (예: "한쪽에만 베팅이 있어 무효 처리되었습니다"). 코드 기반으로 자체 분기하고 싶다면 `round.result_reason` (UPPER_SNAKE) 을 사용하세요. 정상 결과(up/down)/자연 same 일 때는 두 필드 모두 `null` 로 응답됩니다.

**Q. 내가 받은 액수가 베팅한 액수 × `2 - margin` 보다 적어요.**
A. 정산이 **fixed-odds 가 아닌 pari-mutuel ("원금 + 딴돈")** 이기 때문입니다 (2026-05-12 부 변경). 이긴 쪽 인원/금액이 많을수록 1인당 딴돈이 적어지고, 이긴 쪽이 적을수록 1인당 딴돈이 많아집니다. **단, 이긴 사람은 항상 본인 베팅 이상을 받습니다 (원금 손실 X)**.

> **예시 — `margin = 0.05`, up 0.6 (3명 × 0.2) / down 0.4 (2명 × 0.2) / down 이김**
> ```
>   distributable = 0.6 × (1 - 0.05) = 0.57          (진 쪽 풀에서 5% 차감)
>   각 down 베터 (0.2 베팅):
>     bonus       = 0.2 × 0.57 / 0.4 = 0.285         (딴돈)
>     each_payout = 0.2 + 0.285 = 0.485              (원금 + 딴돈)
>   운영 마진 = 0.6 × 0.05 = 0.03 (= 진 쪽 풀의 5%)
> ```
>
> 만약 같은 시나리오에서 **up 이 이겼다면**: `bonus = 0.2 × (0.4×0.95) / 0.6 ≈ 0.1267` → each = 0.2 + 0.1267 ≈ **0.3267**. 이긴 쪽이 많이 걸려있으면 1인당 딴돈이 적은 게 pari-mutuel 의 본질.
>
> UI 표시 — pool 응답의 `up_payout_multiplier` / `down_payout_multiplier` 를 그대로 "예상 배당률" 로 노출하세요. multiplier 의 의미는 "1단위 베팅 → 받게 될 총액 (원금 포함)". 베팅이 추가될 때마다 값이 변동하니 `pool_update` WS 로 실시간 갱신 권장.

---

## §8. 변경 이력

| 일자 | 내용 |
|---|---|
| 2026-04-29 | 초기 발행 |
| 2026-05-08 | **Breaking** — `payout_rate` (예: 1.9) → `margin` (예: 0.05) 로 필드 교체. 의미: 이긴 사람의 딴 돈에서 차감하는 하우스 마진 (소수). 지급 배수는 `2 - margin` 으로 산출. `pool` 응답 객체의 `up_payout_rate` / `down_payout_rate` 도 제거되고 단일 `payout_multiplier` 로 통합 (= `2 - margin`, 양방향 동일). |
| 2026-05-12 | **Additive** — 라운드 무효(환불) 사유 표준화. `round.result_reason` (UPPER_SNAKE 코드) + `round.result_reason_label` (한글 안내 문구) 필드 추가. S3 정산/취소 JSON, REST `GET /api/pub/games/:gameId/rounds/:roundId` 응답에 모두 포함. 기존 동작 변경 없음 — 정상 결과는 두 필드 모두 `null` 반환. 한쪽에만 베팅이 들어온 경우 `ONE_SIDED_BET` 코드로 강제 환불 (이전엔 단순 `result='same'` 만 노출되어 사유 구분 불가했음). |
| 2026-05-12 | **Breaking** — 정산 모델을 **fixed-odds (`bet × (2 - margin)`) → pari-mutuel ("원금 + 딴돈")** 로 변경. 이긴 사람은 원금 그대로 + 진 쪽 풀의 (1 - margin) 을 베팅 비례 딴돈으로 받음. 마진은 **진 쪽 풀의 5%** 만 차감 (이긴 쪽 손대지 않음) → **이긴 사람 원금 손실 절대 없음**. pool 응답에 `up_payout_multiplier` / `down_payout_multiplier` 양방향 동적 배수 추가 (multiplier = "1단위 베팅 → 받을 총액(원금 포함)"). `payout_multiplier` 단일 필드는 deprecated (`null` 반환). 매체사 UI 는 양방향 multiplier 별도 표시 필수. 기존 fixed-odds 로 정산된 라운드는 그대로 유지 — 신규 라운드부터 적용. |
| 2026-05-15 | **Breaking** — `game_code` (16자 hex hash) 식별자 완전 제거. 모든 콜백 envelope / S3 정산 JSON / REST 응답의 `game_code` 필드가 **`game_id` (정수 PK)** 로 일원화. 신규 라운드부터 S3 path 가 `games/{game_id}/rounds/...` 형식. 기존 hex path 의 historical 정산 JSON 은 S3 / `round_settlements.s3_key` 에 그대로 보존되므로 fallback API 정상 동작. **매체 수신부** (Service_API/carrygold 등) 도 `data.game_code` → `data.game_id` 로 destructure 변경 필요. DB 마이그레이션: [20260515_1_drop_games_game_code.sql](sql/20260515_1_drop_games_game_code.sql) — 코드 배포 후 적용. |
| 2026-05-15 | **Fix** — 콜백 페이로드의 `game_id` / `round_id` / `round_number` / `bet_id` 필드가 일부 이벤트에서 **문자열로 직렬화되던 버그 수정** (예: `"round_id": "232"` → `"round_id": 232`). PostgreSQL `bigint` 컬럼이 pg 드라이버 기본값으로 string 반환되던 것을 `Number()` 명시 변환. 영향 이벤트: `round_started` / `round_closed` / `round_ended` / `round_canceled` / `bet_request` / `bet_confirmed` / `bet_canceled`, 그리고 매체별 정산 S3 JSON. 본 문서는 발행 초부터 `integer` 타입을 명시해왔으므로 문서 변경은 없으나, 만약 매체사 수신부가 `"232"` 형태(string)를 가정해 파싱하고 있었다면 numeric 으로 처리하도록 점검 필요. JS `Number` safe range 는 `2^53 − 1` (≈9×10¹⁵) 이라 PK 수명 동안 정밀도 문제는 없음. |
| 2026-05-18 | **Breaking + Additive** — `bet_*` 콜백에 **`order_id`** (HMAC-SHA256 해시 식별자) / **`request_ts`** / **`round_number`** 3개 필드 추가, `bet_confirmed` / `bet_canceled` 는 **2xx 외엔 10초 × 3회 재시도** 정책 도입 (이전: 1회 발송). 신규 베팅 주문은 단타 서버 `bet_orders` 테이블 (월별 파티션) 에 추적되고 시도별 이력은 `order_callback_attempts` 에 보존됨. 매체사는 §3-7 에 따라 `ORDER_ID` UNIQUE 컬럼을 추가해 멱등 처리 (재시도 이중 차감/환불 방지). race rejection 시 `bet_id=null` 이어도 `order_id` 로 단일 키 매칭 가능 → 기존 (`user_id`+`round_id`+`direction`+`amount`) fallback 매칭은 이행기 호환용으로 유지. 신규 사유 추가: `orphan_timeout` (단타 서버 비정상 종료 후 5분 초과 미확정 베팅의 자동 환불 트리거). 마이그레이션: [20260518_1_create_bet_orders.sql](sql/20260518_1_create_bet_orders.sql), [20260518_2_create_order_callback_attempts.sql](sql/20260518_2_create_order_callback_attempts.sql), [20260518_3_alter_callback_logs_add_order_id.sql](sql/20260518_3_alter_callback_logs_add_order_id.sql) — 코드 배포 전 모두 적용 + 단타 서버 환경변수 `DT_ORDER_HMAC_SECRET` (32+ bytes) 설정 필수. |
