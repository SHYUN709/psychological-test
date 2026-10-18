# 나의 마음 지도

WPI(황상민)의 10개 축 개념을 빌린 50문항 자기이해 검사. 정식 WPI 검사가 아니며, 문항과 해석은 새로 작성했다.

- 자기평가 5축: 리얼리스트 · 로맨티스트 · 휴머니스트 · 아이디얼리스트 · 에이전트
- 타인평가 5축: 릴레이션 · 트러스트 · 매뉴얼 · 셀프 · 컬처
- 결과: 오각형 차트 2장 + 프로파일 해석 + 다른 AI 채팅에서 이어 쓸 수 있는 프롬프트 복사

빌드 없이 `index.html` 하나로 동작한다. GitHub Pages: Settings → Pages → Branch `main` / root.

## AI 심층 인터뷰가 동작하는 방식

| 열린 곳 | AI 호출 방법 |
|---|---|
| claude.ai 아티팩트 | `window.claude.use("sample")` (그대로) |
| GitHub Pages 등 그 밖 | `index.html`의 `AI_PROXY` 주소(Cloudflare Worker)로 `fetch` |
| `AI_PROXY`가 비어 있음 | AI 패널을 숨기고 "프롬프트 복사"만 남김 |

프록시 엔드포인트 (`worker/src/index.js`):

- `POST /questions` — `{answers: number[50]}` → `{questions:[{axes, q}]}` (질문 3개, JSON)
- `POST /interpret` — `{answers, interview:[{axes, q, a}]}` → SSE 스트림
  - `event: text` `{t}` 글 조각 / `event: done` `{truncated, refused}` / `event: error` `{code}`

프롬프트는 서버가 만든다. 브라우저는 문항 응답(1~5)과 인터뷰 답만 보내고, 점수도 서버가 다시 계산하므로 프록시를 아무 질문에나 쓰는 범용 AI로 돌려쓸 수 없어.

### 남용 방지

- **CORS / Origin 검사**: `ALLOWED_ORIGINS`에 있는 주소에서 온 요청만 받는다(그 밖은 403). curl처럼 Origin을 위조하는 요청은 이것만으로 못 막아서 아래 횟수 제한이 함께 필요해.
- **하루 호출 제한**: IP당 `DAILY_LIMIT_PER_IP`회(기본 10, 검사 1회 = 질문 1 + 해석 1 = 2회), 전체 `DAILY_LIMIT_GLOBAL`회(기본 300). 날짜는 UTC 기준이고, 카운터는 Workers KV에 48시간 저장돼.
- **토큰 상한**: 질문 `maxOutputTokens` 4000, 해석 8000 (`LIMITS`에서 조정). 요청 본문은 16KB, 질문 300자, 답변 800자까지만 받는다.

## 배포 (Cloudflare Workers)

준비물: Node.js 18+, Cloudflare 계정(무료 플랜 가능, 이메일 인증 필요), Gemini API 키([Google AI Studio](https://aistudio.google.com/apikey)에서 무료 발급).

> Windows PowerShell에서 `npx`가 "스크립트를 실행할 수 없다"며 막히면 `npx` 대신 `npx.cmd`를 쓰면 돼.

```bash
cd worker
npm install
npx wrangler login
```

**1. 레이트리밋용 KV 만들기** — 출력된 `id`를 `worker/wrangler.toml`의 `[[kv_namespaces]] id`에 넣어.

```bash
npx wrangler kv namespace create RATE_LIMIT
```

**2. 허용 도메인 확인** — `worker/wrangler.toml`의 `ALLOWED_ORIGINS`는 이미 `https://shyun709.github.io`로 돼 있어. Origin에는 경로(`/psychological-test/`)가 안 붙으니 그대로 두면 돼. 커스텀 도메인을 쓰게 되면 그 주소를 쉼표로 추가해.

**3. API 키 넣기** — 프롬프트가 나오면 키를 붙여넣어. 키는 Cloudflare에만 암호화돼 저장되고 저장소에는 남지 않아.

```bash
npx wrangler secret put GEMINI_API_KEY
```

**4. 배포**

```bash
npx wrangler deploy
```

출력되는 `https://wpi-ai-proxy.<계정>.workers.dev` 주소를 복사해.

**5. 페이지에 연결** — `index.html`에서 아래 줄을 찾아 주소를 넣고 커밋·푸시해. (이 저장소는 이미 `https://wpi-ai-proxy.wpi-ai-proxy.workers.dev`로 연결돼 있어.)

```js
const AI_PROXY = "https://wpi-ai-proxy.<계정>.workers.dev";
```

### 확인

```bash
curl -i -X POST https://wpi-ai-proxy.<계정>.workers.dev/questions -H "Origin: https://evil.example" -H "Content-Type: application/json" -d "{}"
```

`403 forbidden_origin`이 나오면 정상이야. 실제 동작은 GitHub Pages에서 검사를 끝내고 "질문 받기"를 눌러 봐. 로그는 `npx wrangler tail`로 볼 수 있어.

### 로컬에서 테스트

```bash
cd worker
cp .dev.vars.example .dev.vars
npx wrangler dev
```

`.dev.vars`에 키를 넣고, 다른 터미널에서 `python -m http.server 8080`으로 페이지를 띄운 뒤 `AI_PROXY`를 `http://127.0.0.1:8787`로 잠깐 바꿔서 테스트해. (커밋하기 전에 되돌리기)

### 설정 바꾸기

- 모델: `wrangler.toml`의 `MODEL` (기본 `gemini-3.8-flash`). 이 모델이 붐비거나(503) 한도(429)에 걸리면 한 번 더 시도한 뒤 `FALLBACK_MODEL`(기본 `gemini-3.5-flash-lite`)로 넘어가.
- 횟수: `DAILY_LIMIT_PER_IP`, `DAILY_LIMIT_GLOBAL`. 바꾼 뒤 `npx wrangler deploy`.
- 무료 등급에는 분당·하루 요청 한도가 있어서, 넘으면 페이지에 "요청이 너무 많아" 안내가 떠. 유료 등급으로 바꾸면 Google Cloud 결제 계정에서 예산 알림을 걸어 둬.

### 개인정보 안내

Gemini API 무료 등급은 Google 요금 안내상 입력 내용이 "Google 제품 개선에 사용"될 수 있어. 이 검사는 문항 응답과 인터뷰 답변을 보내므로, 민감한 내용을 쓰지 않도록 안내하거나 유료 등급으로 바꾸는 걸 고려해.

## 비밀값 관리

- API 키는 `wrangler secret`으로만 넣는다. `wrangler.toml`, `index.html`, 커밋 메시지 어디에도 쓰지 마.
- 로컬용 `.dev.vars`는 `.gitignore`에 들어 있어. 커밋 전에 `git status`로 한 번 더 확인해.
- 키가 노출됐다면 Google AI Studio에서 바로 삭제하고 새 키로 `wrangler secret put`을 다시 실행해.

## 프롬프트 수정할 때

질문·해석 프롬프트는 두 군데에 있어. 바꿀 땐 둘 다 맞춰 줘.

- `worker/src/index.js` — `questionsPrompt`, `interpretPrompt` (GitHub Pages용)
- `index.html` — `claudeBackend` (claude.ai 아티팩트용)

문항(`ITEMS`)이나 축 설명(`AXES`)을 바꾸면 `worker/src/index.js`의 사본도 같이 바꿔야 점수가 맞아.
