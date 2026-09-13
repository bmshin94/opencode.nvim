# opencode.nvim 분석 정리 (한국어)

> 이 문서는 `opencode.nvim` 레포를 분석하고 정리한 개인 학습/기획 노트입니다.
> 코드 기준 시점: `v1.0.1` (2026-09-11 릴리즈)

## 🔗 관련 링크

| 구분 | 주소 |
| --- | --- |
| 내 포크 (이 레포) | https://github.com/bmshin94/opencode.nvim |
| 원본 레포 (upstream) | https://github.com/nickjvandyke/opencode.nvim |
| OpenCode 공식 사이트 | https://opencode.ai/ |
| nixpkgs 등재 패키지명 | `pkgs.vimPlugins.opencode-nvim` |

> ℹ️ 이 레포는 원본을 **포크**한 것이며, 추가된 커밋은 `CLAUDE.md` 한 건뿐입니다.
> 원본이 업데이트되면 수동으로 동기화해야 합니다.

---

## 1. 이게 뭐야? — 한 줄 요약

**Neovim에서 AI 코딩 에이전트(OpenCode)를 쓰게 해주는 플러그인.**

AI한테 코드 물어보려고 브라우저/다른 앱으로 창을 옮겨다니는 왕복 과정을 없애줍니다.

| 기존 방식 | opencode.nvim |
| --- | --- |
| 1. 코드 보다가 막힘 | 1. 코드에 커서 두고 `Ctrl+X` |
| 2. 브라우저에서 AI 켬 | 2. 메뉴에서 "설명해줘" 고름 |
| 3. 코드 복사 → 붙여넣기 | **끝** |
| 4. "이거 XXX.py 40번째 줄인데요..." 설명 | |
| 5. 답 받고 다시 복사 | |
| 6. 에디터로 복귀 | |

### 규모

- Lua 코드 **약 2,000줄**, 총 **51개 파일**
- 하드 의존성 없음 (`AGENTS.md`: *"No hard Lua dependencies beyond Neovim itself"*)
- 라이선스: **MIT** (Copyright (c) 2025 Nick van Dyke)

---

## 2. 구조 & 동작 원리

```
Neovim (이 플러그인)  ──REST 요청──▶  opencode CLI 서버  ──▶ AI 모델
       ▲                                    │
       └────── SSE 실시간 이벤트 ◀───────────┘
```

### 폴더별 역할

| 경로 | 하는 일 |
| --- | --- |
| `lua/opencode.lua` | 공개 API 6개 (`ask`, `select`, `prompt`, `command`, `operator`, `statusline`) |
| `lua/opencode/server/` | 서버 자동 탐색 + REST/SSE 통신 (`curl` 래핑) |
| `lua/opencode/context/` | 에디터 컨텍스트 수집 (커서/선택영역/버퍼/진단) — 약 500줄 |
| `lua/opencode/ui/` | 입력창(`ask`) · 선택 메뉴(`select`) · 자동완성용 in-process LSP |
| `lua/opencode/events/` | OpenCode 이벤트를 Neovim `autocmd`로 변환 |
| `lua/opencode/promise/` | 의존성 없는 자체 Promise 구현 (`promise.nvim` 포크, 339줄) |
| `lua/opencode/config.lua` | 기본 설정값 (`vim.g.opencode_opts`로 덮어씀) |
| `lua/opencode/health.lua` | `:checkhealth opencode` 진단 |
| `plugin/events/` | Neovim 시작 시 자동 등록되는 autocmd 4종 |

### 핵심 기능 3개

#### ① 컨텍스트 플레이스홀더 (제일 유용)

프롬프트에 `@this`만 쓰면 → `"MyFile.py 파일의 40~55번째 줄"`로 자동 치환.
복사 붙여넣기도, 파일명 설명도 필요 없음.

| 플레이스홀더 | 들어가는 내용 |
| --- | --- |
| `@this` | 선택 영역, 없으면 커서 위치 |
| `@buffer` / `@buffers` | 현재 파일 / 열린 파일 전체 |
| `@diagnostics` | LSP 에러·경고 |
| `@marks` / `@quickfix` / `@visible` | 마크 / 퀵픽스 목록 / 화면에 보이는 부분 |

구현: `lua/opencode/context/builtins.lua` (164줄)
사용자가 `contexts` 옵션으로 **자기만의 플레이스홀더를 추가**할 수 있음 (확장성 설계 👍)

#### ② 내장 프롬프트 템플릿 8종

`Ctrl+X` → 메뉴에서 선택:
`explain` · `fix` · `review` · `test` · `optimize` · `document` · `implement` · `diagnostics`

#### ③ Vim 네이티브 diff로 AI 수정 승인 (킬러 기능)

AI가 파일을 고치려 하면 새 탭에서 `:diffpatch`로 좌우 비교 표시. 사람이 승인해야 적용됩니다.

| 키 | 동작 |
| --- | --- |
| `da` / `dr` | 전체 수락 / 거부 |
| `]c` / `[c` | 다음 / 이전 변경점 |
| `dp` / `do` | **커서 위 hunk만** 적용 / 거부 |

특히 `dp`가 강력 — AI가 5군데 고쳐도 **원하는 2군데만 골라 쓸 수 있음**.
구현: `lua/opencode/events/permissions/edits.lua`

---

## 3. 설치 및 사용법

### 준비물 3개

```bash
# 1. opencode 본체 (AI 담당) — 필수, 1.17 버전 이상
curl -fsSL https://opencode.ai/install | bash

# 2. curl — REST 통신용 (보통 이미 있음)
# 3. pgrep, lsof — 서버 자동 탐색용 (맥/리눅스 기본 탑재)
```

> 최소 버전 체크 로직: `lua/opencode/health.lua:38`

### Neovim 설치

```lua
-- 최신 Neovim (vim.pack)
vim.pack.add({
  {
    src = "https://github.com/nickjvandyke/opencode.nvim",
    version = vim.version.range("*"), -- 최신 안정 릴리즈
  },
})

-- ⚠️ setup() 호출이 아님! 전역 변수로 설정하는 게 이 플러그인 특징
---@type opencode.Opts
vim.g.opencode_opts = {}

-- 추천 키맵
vim.keymap.set({ "n", "x" }, "<C-a>", function() require("opencode").ask("@this: ") end)
vim.keymap.set({ "n", "x" }, "<C-x>", function() require("opencode").select() end)
vim.keymap.set({ "n", "x" }, "go",    function() return require("opencode").operator("@this ") end, { expr = true })
vim.keymap.set({ "n" },      "goo",   function() return require("opencode").operator("@this ") .. "_" end, { expr = true })
```

lazy.nvim 사용 시: `{ "nickjvandyke/opencode.nvim", version = "*" }`

> 설정을 `setup()` 대신 `vim.g`로 받는 이유: **더 단순한 UX + 빠른 시작 속도**
> (`lua/opencode/config.lua:9` 주석 참고)

### 실행

```bash
opencode --port    # ⚠️ --port 없으면 서버가 안 열려서 연결 불가!
```

그다음 Neovim에서 `:checkhealth opencode` 로 진단 → 초록불 확인.

### 사용법

| 방법 | 하는 일 |
| --- | --- |
| `Ctrl+A` (`ask`) | 직접 질문 입력 (`@this` 자동완성, `↑`로 이전 질문 재사용) |
| `Ctrl+X` (`select`) | 메뉴에서 프롬프트/명령/서버 선택 |
| `go` + 모션 (`operator`) | 그 범위를 AI에게 전달 (`.`으로 반복 가능) |
| `goo` | 현재 한 줄 전달 |

**꿀팁**
- 프롬프트 끝에 **띄어쓰기** → 전송하지 않고 쌓아둠
- 프롬프트 끝에 **`...`** → 입력창(`ask`)이 열림
- (구현: `lua/opencode/api/prompt.lua:10`)
- OpenCode는 참조된 파일을 **디스크에서** 읽으므로 **먼저 저장**해야 함

### 선택적 통합

| 플러그인 | 효과 |
| --- | --- |
| `snacks.nvim` | `ask`에 `snacks.input`, `select`에 `snacks.picker` 적용 (미리보기 지원) |
| `blink.cmp` | `ask` 입력창에서 LSP 기반 자동완성 |
| `lualine.nvim` | 연결된 서버와 상태를 상태바에 표시 |

---

## 4. 플러그인? 스킬? MCP? → **순수 Neovim 플러그인**

| 구분 | 정체 | 실행 주체 | 해당? |
| --- | --- | --- | --- |
| **플러그인** | 프로그램에 기능 추가하는 확장 | 에디터(Neovim) | ✅ **이것** |
| **스킬** | AI에게 주는 작업 설명서(문서) | AI가 읽음 | ❌ |
| **MCP** | AI ↔ 외부도구 연결 표준 규격 | AI가 호출 | ❌ |

### 근거

- `lua/` + `plugin/` 폴더 구조 = Neovim 플러그인의 정석
- `plugin/events/`는 Neovim 시작 시 자동 실행 (autocmd 등록)
- 통신이 **평범한 HTTP REST + SSE** (`lua/opencode/server/init.lua:121`의 `curl` 함수)
  → MCP의 JSON-RPC 규격이 **아님**

### 방향이 반대라는 점이 핵심

```
MCP        = AI에게 손발을 달아주는 것 (AI가 도구를 사용)
이 플러그인 = AI에게 가는 리모컨      (사람이 AI를 사용)
```

AI를 확장하는 게 아니라, **에디터를 확장해서 AI를 편하게 부르는** 도구입니다.

---

## 5. API 토큰 필요해? → 두 개를 구분해야 함

### 🅰️ 이 플러그인 자체 — 토큰 불필요 ✅

플러그인은 `localhost:포트`로 요청을 보낼 뿐, AI와 직접 통신하지 않습니다.

다만 **옵션으로** Basic 인증을 넣을 수 있습니다 (원격 서버 연결 시):

```lua
vim.g.opencode_opts = {
  server = {
    url = "http://내서버:4096",
    username = "opencode",   -- OPENCODE_SERVER_USERNAME 환경변수도 읽음
    password = "비밀번호",     -- OPENCODE_SERVER_PASSWORD
  },
}
```

> 구현: `lua/opencode/config.lua:24`, `lua/opencode/server/init.lua:143`
> ⚠️ 이건 AI 요금과 무관한 **서버 문단속용 자물쇠**입니다.

### 🅱️ `opencode` 본체 — 여기는 인증 필요 💰

실제 AI 모델을 호출하는 건 `opencode`이므로:

```bash
opencode auth login
```

선택지 3가지:
- **본인 API 키** (Anthropic / OpenAI 등) — 사용량 과금
- **구독 연동** — 기존 구독 계정 연결
- **로컬 모델** (Ollama 등) — **완전 무료**

> 참고: `.github/workflows/opencode-review.yml:25`의 `OPENCODE_API_KEY`는
> **깃허브 자동리뷰 워크플로용**이며 로컬 설치와 무관합니다.

---

## 6. 왜 깃허브에서 유명할까? (⭐ 3.8k / 🍴 158)

> 2026-09-13 기준 원본 레포 확인값: 별 **3.8k**, 포크 **158**

| # | 이유 | 근거 |
| --- | --- | --- |
| ① | **타이밍** | OpenCode 인기 상승기에 "Neovim 유저용"이 없던 시점에 등장 |
| ② | **"새 UI를 안 만든다" 철학** | README: *"Rather than introduce yet another interaction model..."* — `:diffpatch`, `autoread`, `operatorfunc`+`g@` 등 Vim 기본 기능 재활용 → **새로 배울 게 없음** |
| ③ | **초경량** | Lua 2,000줄, 의존성은 `opencode` + `curl`뿐. 열어보면 다 읽히는 크기 → 신뢰 |
| ④ | **문서 품질** | README + `AGENTS.md`(AI 에이전트용 안내서!) + `CONTRIBUTING.md`(철학) + 이슈/PR 템플릿 + CI 3종 |
| ⑤ | **공신력** | **nixpkgs 공식 패키지 등재** (`pkgs.vimPlugins.opencode-nvim`) |
| ⑥ | **유지관리 신뢰** | v1.0.0 릴리즈 노트에 원작자가 직접 *"내 확신을 의미한다"*고 명시 + `release-please`로 릴리즈 자동화 |

### CI 구성 (참고용으로 좋음)

| 워크플로 | 역할 |
| --- | --- |
| `lua-ls.yml` | 타입 체크 (lua-language-server) |
| `stylua.yml` | 포맷 검사 (column_width 120, indent 2, double quotes) |
| `release-please.yml` | 자동 릴리즈 + CHANGELOG 생성 |
| `opencode-review.yml` | PR에 `/review` 댓글 → AI 자동 코드리뷰 |

> ⚠️ 테스트 프레임워크는 없음 (`AGENTS.md`: *"No test framework"*). 검증은 `:checkhealth` 수동.

---

## 7. 로컬 에이전트 구축에 도움될까? → **강력 추천**

**"에이전트 클라이언트 교과서"** 수준. 훔쳐갈 만한 패턴 5개:

### ⭐ ① 서버 자동 탐색 (가장 실용적)

`lua/opencode/server/discovery/process/unix.lua`

```
pgrep -f "opencode.*--port"                 → 실행중인 프로세스 PID 찾기
lsof -Fpn -w -iTCP -sTCP:LISTEN -p <PIDs>   → 그 PID가 열어둔 포트 알아내기
```

포트 번호를 설정에 적지 않아도 자동 탐색. 현재 작업 폴더와 겹치는 서버만 필터링하고,
없으면 직접 띄운 뒤 5초간 폴링합니다.

**탐색 우선순위** (`server/discovery/init.lua`):
`이미 연결된 서버` → `설정된 URL` → `로컬 프로세스 스캔(CWD 겹침 필터)` → `자동 시작 + 폴링`

### ⭐⭐ ② 컨텍스트 주입 엔진 (핵심)

`lua/opencode/context/` — 에이전트 품질의 90%는 모델이 아니라 **"AI에게 뭘 보여주느냐"**.

배울 점:
- UI가 열리기 **전에** 커서/선택영역을 먼저 캡처 (입력창 열면 위치 정보가 날아가므로)
- `@this` → `"파일:줄번호"` 문자열로 렌더링
- 사용자 정의 플레이스홀더 추가 가능한 확장 구조

### ⭐ ③ Human-in-the-loop 권한 게이트

`plugin/events/permissions/init.lua`

```
AI: "이 파일 고쳐도 돼?"            (permission.asked 이벤트 수신)
  → 사람에게 diff 표시
  → POST /permission/{id}/reply     (승인 또는 거부 전송)
```

에이전트가 폭주하지 않게 막는 구조의 완성된 예시입니다.

### ④ SSE 이벤트 → 에디터 이벤트 브릿지

`GET /event`를 persistent 모드로 열고(`server/init.lua:314`), 들어오는 이벤트를
`OpencodeEvent:<타입>` User autocmd로 재배포:

```lua
vim.api.nvim_create_autocmd("User", {
  pattern = "OpencodeEvent:session.status",  -- 원하는 이벤트만 필터
  callback = function(args)
    local event = args.data.event
    local url = args.data.url
  end,
})
```

**플러그인 코드를 수정하지 않고도 사용자가 기능을 확장할 수 있는 설계** — 배울 만함.

### ⑤ 의존성 없는 Promise 구현

`lua/opencode/promise/init.lua` (339줄) — 체인 / 에러 전파 / 취소 포함.

### ⚠️ 한계

이건 **"몸"이 아니라 "리모컨"** 입니다. 진짜 에이전트 두뇌(LLM 호출 루프, 도구 실행,
대화 기억)는 전부 `opencode` 쪽에 있습니다. 두뇌를 배우려면 opencode 본체 소스를 봐야 함.

> 다만 에이전트에서 제일 어려운 건 두뇌가 아니라 **"컨텍스트 수집"과 "안전장치"** 이고,
> 그 두 개가 정확히 이 레포에 있습니다.

---

## 8. React / PHP로 만들 수 있어? → **가능**

통신 방식이 **평범한 HTTP REST + SSE**뿐이므로 어떤 언어로든 구현 가능합니다.

### 실제 사용되는 엔드포인트 전체 목록

> 출처: `lua/opencode/server/init.lua` 전수 조사

| Method | 엔드포인트 | 용도 |
| --- | --- | --- |
| `GET` | `/global/health` | 서버 생존 확인 |
| `GET` | `/path` | 작업 디렉토리 조회 |
| `GET` | `/session` | 세션 목록 |
| `GET` | `/agent` | 사용 가능한 에이전트 목록 |
| `POST` | `/tui/publish` | **프롬프트 텍스트 입력** ⭐ |
| `POST` | `/tui/execute-command` | 명령 실행 (전송/취소/undo 등) |
| `POST` | `/tui/select-session` | 세션 전환 |
| `POST` | `/permission/{id}/reply` | **승인/거부 응답** ⭐ |
| `GET` | `/event` | **SSE 실시간 이벤트 스트림** ⭐ |

> 💡 프롬프트 전송 방식이 특이합니다: `/tui/publish`로 **텍스트를 입력창에 "붙여넣고"**
> → `/tui/execute-command`로 **"엔터를 누르는"** 2단계
> (`lua/opencode/api/prompt.lua:16-20`). 사람의 타이핑을 그대로 흉내내는 구조.

### ⚛️ React — 잘 맞음

```js
// SSE는 브라우저 내장 API로 처리
const es = new EventSource("http://localhost:4096/event");
es.onmessage = (e) => setEvents((prev) => [...prev, JSON.parse(e.data)]);

// 프롬프트 전송
await fetch("http://localhost:4096/tui/publish", {
  method: "POST",
  body: JSON.stringify({
    type: "tui.prompt.append",
    properties: { text: "이 코드 설명해줘" },
  }),
});
```

챙겨야 할 것:
- `@this`를 쓰려면 에디터가 필요 → **Monaco** 또는 **CodeMirror** 통합
- diff 화면 → `react-diff-viewer` 등으로 구현
- **CORS 주의** — 브라우저에서 직접 localhost 호출이 막힐 수 있음 → 얇은 프록시 권장

### 🐘 PHP — 가능하지만 주의점

- ✅ **REST 호출은 쉬움** — `curl` / Guzzle로 처리. `/event` 외엔 전부 짧은 요청
- ❌ **SSE(`/event`)가 문제** — 연결을 장시간 유지해야 하는데 일반 PHP-FPM은
  "요청 받고 즉시 종료" 구조라 워커가 계속 점유됨

해결책:
1. **SSE만 Node.js로 분리** (가장 현실적) — PHP는 나머지 담당
2. **ReactPHP / Swoole / FrankenPHP** 사용 (상주 프로세스)
3. **폴링으로 대체** — 실시간성 일부 포기 (권한 승인 UX가 답답해짐)

### 추천 조합

```
React (프론트) + Node.js (SSE 중계) + PHP (비즈니스 로직 / DB)
```

### 포팅의 진짜 난관 — Vim이 공짜로 주던 것들

| Vim에선 기본 제공 | 웹에선 직접 구현 |
| --- | --- |
| `:diffpatch` 좌우 비교 | diff 뷰어 + hunk 단위 선택 로직 |
| `autoread` 파일 자동 갱신 | 파일 감시(watcher) + 웹소켓 푸시 |
| `dp`/`do` hunk 부분 적용 | 패치 파싱/적용 엔진 |
| 커서/선택영역 정보 | 에디터 컴포넌트에서 추출 |

> 이 플러그인이 2,000줄로 끝난 건 **Vim이 무거운 일을 대신 해줬기 때문**입니다.
> 웹으로 옮기면 코드량이 몇 배로 늘어납니다.

---

## 9. 수익화 아이디어

### 🧊 전제: 플러그인 자체로는 돈이 안 됨

| 걸림돌 | 내용 |
| --- | --- |
| **MIT 라이선스** | 누구나 복사·수정·상업적 판매 자유 → 방어력 0 |
| **니치 시장** | Neovim 유저는 소수이고, 유료 플러그인 저항이 가장 큰 집단 |
| **원작자도 안 함** | `.github/FUNDING.yml`에 GitHub Sponsors(`nickjvandyke`) 하나뿐 |
| **API 불안정** | README 명시: 이벤트 스키마는 *"stable API 계약이 아니라 best-effort"* |

### 💡 핵심 통찰

> **개발자는 도구에 돈을 안 쓴다. 하지만 "회사"는 리스크에 돈을 쓴다.**

```
"AI가 고친 코드 diff로 보여주는 도구"   → 개발자에게: 0원
"AI 코드 변경 감사 추적 + 정책 통제"    → 보안/컴플라이언스팀에: 유료 가능
```

**똑같은 코드인데 구매 결정자만 바뀐 것.**

### 🔍 숨은 보석 — `opencode-review.yml:28`

```json
{ "bash": { "*": "deny", "gh*": "allow",
            "gh pr review*": "deny",
            "gh pr review --comment*": "allow" } }
```

**"AI가 실행할 수 있는 명령어를 화이트리스트로 통제하는 정책"** — 기본 전면 차단(`deny`),
필요한 것만 허용.

- 지금은 **YAML에 손으로 쓴 한 줄**
- 하지만 회사 입장에선 **"AI 에이전트 보안 정책"** 그 자체
- 이걸 **관리·배포·위반 기록**하는 도구는 시장에 거의 없음

> 🎯 팔 수 있는 건 플러그인이 아니라 **"AI 에이전트에게 씌우는 안전벨트 + 블랙박스"**.

### 아이디어 랭킹

| 순위 | 아이디어 | 첫 매출 | 잠재 규모 | 난이도 | 방어력 |
| --- | --- | --- | --- | --- | --- |
| 🥇 1 | AI 거버넌스 & 감사 플랫폼 | 6~12개월 | 🔥🔥🔥 | 높음 | 강함 |
| 🥈 2 | JetBrains/VSCode 포팅 (마켓플레이스) | 2~4개월 | 🔥🔥 | 중간 | 약함 |
| 🥉 3 | 컨텍스트 팩 / 최적화 SaaS | 3~6개월 | 🔥🔥 | 중간 | 중간 |
| 4 | 특화 코드리뷰 GitHub App | 2~3개월 | 🔥 | 낮음 | 매우 약함 |
| 5 | 자체 호스팅 팀 에이전트 서버 | 6~12개월 | 🔥🔥🔥 | 매우 높음 | 강함 |
| 6 | 콘텐츠 · 교육 · 컨설팅 | **2~6주** | 🔥 | 낮음 | 중간 |

> ⚠️ 위 표는 확정 수치가 아닌 **판단/추정**이며, 실제 시장 반응으로 검증 필요.

#### 🥇 1위 상세 — AI 거버넌스 & 감사 플랫폼 ("AI 블랙박스")

**문제:** 회사가 AI 코딩 도구를 도입했지만 —
AI가 어떤 파일을 읽었는지(`.env`? DB 접속정보?), 어떤 명령을 실행했는지,
누가 승인했는지 **기록이 없음**. 사고 나면 *"AI가 그랬어요"* 밖에 할 말이 없음.

**해결:**

```
      AI 에이전트
          ↓ permission.asked
   ┌──────────────────────┐
   │  ① 정책 검사          │  ← .env 등 민감 경로 차단
   │  ② 사람에게 승인 요청  │  ← 위험한 건 2인 승인
   │  ③ 전부 기록          │  ← 감사 로그
   └──────────────────────┘
          ↓ POST /permission/{id}/reply
      승인 또는 거부
```

**티어 구성 (Open-core):**

| 티어 | 내용 |
| --- | --- |
| 🆓 무료 (OSS) | 에디터 플러그인 + 로컬 로그 |
| 💼 팀 | 중앙 정책 배포, 대시보드, 슬랙 알림 |
| 🏢 기업 | SSO, 감사 리포트, 온프레미스, 승인 워크플로 |

- **쓰는 사람:** 개발자 / **돈 내는 사람:** 보안팀 · CTO · 컴플라이언스
- **좋은 진입점:** 규제 산업 (금융, 의료, 공공) — AI 도입하고 싶은데 감사 때문에 못 하는 곳

**MVP (2~3주):**
1. `GET /event` SSE 구독 → `permission.asked` 전부 SQLite 기록
2. 웹 대시보드 1페이지 (승인/거부 타임라인, 파일별 통계)
3. `policy.yml` — 경로 패턴 차단 하나만
4. 개발자 5명에게 보여주고 *"이거 회사에 필요해?"* 검증

### 🎯 추천 전략: "6번으로 벌면서 1번을 만든다"

```
지금        6️⃣ 콘텐츠/컨설팅      → 현금 + 고객 인터뷰 + 브랜드
  ↓
3~6개월     🥇 감사 도구 (무료 OSS) → 유저 확보
  ↓
6~12개월    💼 팀 플랜 유료화      → 사업화
```

**90일 플랜 (각 단계에 통과 기준을 두고, 못 넘으면 진행 안 함)**

| 기간 | 할 일 | 통과 기준 |
| --- | --- | --- |
| 0~2주 | **검증만** (코드 X). 개발자 10명 + 보안담당 3명 인터뷰 | 절반 이상이 "문제다" |
| 3~6주 | 못생긴 MVP (SSE 로거 + 대시보드 + policy.yml) | 본인이 매일 쓰게 됨 |
| 7~10주 | 공개 (GitHub, r/neovim, HN) + 한국어 콘텐츠 1편 | 별 100개 또는 실유저 10명 |
| 11~13주 | 유료화 실험 (팀 플랜 대기명단 + 가격 노출) | 5개 팀이 이메일 남김 |

### ⚠️ 리스크 4개

| 리스크 | 대응 |
| --- | --- |
| **플랫폼 리스크** (이벤트 스키마 변경 — README에 명시됨) | 처음부터 **어댑터 패턴**으로 설계, 여러 에이전트 지원 |
| **흡수 리스크** (에이전트 벤더가 감사 기능 내장) | **"멀티 벤더 통합 감사"** 를 무기로 — 벤더는 경쟁사를 감사해주지 않음 |
| **오픈소스 딜레마** (MIT라 복사 가능) | 코드가 아니라 **호스팅·데이터·통합·신뢰**를 판매 |
| **타이밍 리스크** (아직 위기감이 없을 수 있음) | 0~2주 인터뷰로 "지금 필요" vs "내년 필요" 확인 |

---

## 10. 최종 요약

| 항목 | 결론 |
| --- | --- |
| 🔌 **정체** | Neovim 플러그인 (MCP도 스킬도 아님) |
| 💰 **비용** | 플러그인 무료·토큰 불필요 / AI 모델 값은 별도 (로컬 모델 쓰면 무료) |
| ⭐ **인기 비결** | 새 UI 안 만들고 Vim 기능 재활용 + 초경량 + 문서 최상급 |
| 🤖 **에이전트 학습** | 서버 자동탐색 / 컨텍스트 주입 / 권한 게이트 → 강력 추천 |
| 🌐 **포팅** | REST + SSE라 React/PHP 모두 가능 (SSE만 주의) |
| 💵 **수익화** | "AI 변경 감사·거버넌스 도구"가 가장 유망 |

### ⚠️ 이 레포 사용 시 주의사항

1. 포크 상태이므로 **원본 업데이트는 수동 동기화** 필요
2. `opencode`는 반드시 **`--port`** 로 실행
3. 설정은 `setup()`이 아니라 **`vim.g.opencode_opts`**
4. 이벤트 payload는 **안정 API가 아님** (OpenCode 버전 따라 변경 가능)
5. OpenCode는 파일을 **디스크에서** 읽으므로 프롬프트 전에 **저장** 필요
6. `vim.g`는 정수/문자열 혼합 키를 지원하지 않아 일부 snacks 옵션 제약 →
   `require("opencode.config").opts` 직접 수정으로 우회
