# Hermes 검색 관련 MCP / 설정 스냅샷

> 작성일: 2026-05-13  
> 대상 머신: `/home/pewpew/.hermes` 기본 프로필  
> 목적: Hermes에서 현재 검색/리서치에 쓰는 MCP 서버, 플러그인, Toolset 설정을 외부에서 빠르게 확인하기 위한 공유 문서  
> 보안: API Key, Token, SerpAPI MCP URL의 토큰성 path는 전부 `[REDACTED]` 처리

---

## 1. 현재 결론

현재 기본 Hermes 프로필에는 검색/리서치용으로 아래 구성이 켜져 있습니다.

- Built-in toolset
  - `web`: enabled — Web Search & Scraping
  - `browser`: enabled — Browser Automation
  - `session_search`: enabled — 과거 세션 검색

- MCP servers
  - `serpapi`: enabled — SerpApi MCP, remote HTTP
  - `serper`: enabled — Google/News/Images/Maps 등 Serper MCP, stdio via `npx`
  - `exa`: enabled — semantic web search/fetch MCP, stdio via `npx`
  - `perplexity-ask`: enabled — Perplexity Sonar 계열 질의 MCP, stdio via `npx`
  - `gbrain`: enabled — 로컬 GBrain MCP, stdio via bun

- Plugin
  - `web-search-plus`: enabled

---

## 2. Hermes MCP 상태

`hermes mcp list` 기준:

```text
MCP Servers:

  Name             Transport                      Tools        Status
  ──────────────── ────────────────────────────── ──────────── ──────────
  gbrain           /home/pewpew/.bun/bin/bun...   all          ✓ enabled
  serpapi          https://mcp.serpapi.com/[REDACTED]/mcp   all   ✓ enabled
  serper           npx -y go-serper-mcp-server    all          ✓ enabled
  exa              npx -y exa-mcp-server          all          ✓ enabled
  perplexity-ask   npx -y perplexity-mcp          all          ✓ enabled
```

`hermes tools list` 검색 관련 필터 기준:

```text
✓ enabled  web             🔍 Web Search & Scraping
✓ enabled  browser         🌐 Browser Automation
✓ enabled  session_search  🔎 Session Search

MCP servers:
  serpapi         all tools enabled
  serper          all tools enabled
  exa             all tools enabled
  perplexity-ask  all tools enabled
```

---

## 3. Sanitized `~/.hermes/config.yaml` 관련 섹션

### 3.1 Toolsets

```yaml
toolsets:
  - hermes-cli
```

메모:

- 현재 config의 `toolsets`는 `hermes-cli` 프리셋입니다.
- 실제 런타임에서는 `web`, `browser`, `session_search`, MCP tools가 활성화되어 있습니다.

### 3.2 MCP servers

```yaml
mcp_servers:
  gbrain:
    command: /home/pewpew/.bun/bin/bun
    args:
      - /home/pewpew/.bun/install/global/node_modules/gbrain/src/cli.ts
      - serve
    env:
      PATH: /home/pewpew/.bun/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    timeout: 120
    connect_timeout: 60

  serpapi:
    url: https://mcp.serpapi.com/[REDACTED]/mcp
    timeout: 120
    connect_timeout: 60

  serper:
    command: npx
    args:
      - -y
      - go-serper-mcp-server
      - -t
      - stdio
    env:
      SERPER_API_KEY: [REDACTED]
    timeout: 120
    connect_timeout: 60

  exa:
    command: npx
    args:
      - -y
      - exa-mcp-server
    env:
      EXA_API_KEY: [REDACTED]
    timeout: 120
    connect_timeout: 60

  perplexity-ask:
    command: npx
    args:
      - -y
      - perplexity-mcp
    env:
      PERPLEXITY_API_KEY: [REDACTED]
    timeout: 120
    connect_timeout: 60
```

### 3.3 Plugins

```yaml
plugins:
  enabled:
    - web-search-plus
```

### 3.4 Security

```yaml
security:
  allow_private_urls: false
  redact_secrets: true
  tirith_enabled: true
  tirith_path: tirith
  tirith_timeout: 5
  tirith_fail_open: true
  website_blocklist:
    enabled: false
    domains: []
    shared_files: []
```

검색/웹 도구를 쓸 때 참고:

- `allow_private_urls: false`라 사설망 URL 접근은 기본 차단 방향입니다.
- `redact_secrets: true`라 도구 에러/출력의 시크릿 노출을 줄이는 설정이 켜져 있습니다.

### 3.5 Agent / model 관련

```yaml
agent:
  max_turns: 150
  gateway_timeout: 1800
  tool_use_enforcement: auto
  disabled_toolsets: []

model:
  default: gpt-5.5
  provider: openai-codex
  base_url: https://chatgpt.com/backend-api/codex
  context_length: 400000
```

검색/리서치 작업에 영향을 주는 부분:

- `max_turns: 150`: 검색 → fetch → 요약 → 재검색 루프를 꽤 길게 수행 가능
- `context_length: 400000`: 대량 문서 비교/요약에 유리

---

## 4. 검색 도구별 용도

### 4.1 Built-in `web`

기본 웹 검색/본문 추출에 사용.

- 빠른 일반 검색
- URL 본문 추출
- 단순 사실 확인

### 4.2 `web-search-plus` plugin

다중 검색 provider 라우팅용 플러그인.

적합:

- 뉴스/쇼핑/사실형: Serper/SerpApi 계열
- 리서치/분석형: Tavily/Exa/Perplexity 계열
- semantic discovery: Exa
- 직접 답변/추론형: Perplexity

현재 config상 plugin은 enabled 상태입니다.

### 4.3 SerpApi MCP

구조화된 SERP 데이터에 강함.

적합:

- Google SERP
- Google News
- Shopping
- Jobs
- Images
- Local
- Flights/Hotels 등 엔진별 구조화 결과

현재는 remote HTTP MCP로 연결됩니다.

### 4.4 Serper MCP

Google 검색 계열을 빠르게 호출할 때 사용.

적합:

- 일반 Google search
- News
- Images
- Maps/Places
- Shopping
- Scholar/Patents 등

현재는 `npx -y go-serper-mcp-server -t stdio`로 구동됩니다.

### 4.5 Exa MCP

semantic search / discovery / page fetch에 적합.

적합:

- 키워드보다 “이상적인 문서 설명”으로 찾기
- 특정 주제에 대한 블로그/문서/회사 페이지 발견
- URL 본문 fetch
- 인물/회사 semantic discovery

현재는 `npx -y exa-mcp-server`로 구동됩니다.

### 4.6 Perplexity Ask MCP

복잡한 리서치 질의, reasoning, deep research에 적합.

적합:

- 여러 출처를 종합해야 하는 질문
- 기술/시장/정책 비교
- 빠른 fact search
- 긴 deep research 보고서

현재는 `npx -y perplexity-mcp`로 구동됩니다.

### 4.7 GBrain MCP

검색 엔진이라기보다 로컬 지식베이스/브레인 검색 계층입니다.

적합:

- 저장된 page/chunk 검색
- graph link traversal
- raw data 저장/조회
- timeline/link/tag 기반 지식 검색

주의:

- 현재 이 세션에서 파일 업로드 도구 호출 시 `No database connection: connect() has not been called` 에러가 난 적이 있습니다. GBrain 자체 MCP 등록은 enabled지만, 특정 storage 기능은 DB 연결 상태 확인이 필요합니다.

---

## 5. 운영 추천 라우팅

### 빠른 현재 정보 확인

1. Built-in `web_search`
2. 필요 시 `web_extract`
3. SERP 구조가 필요하면 Serper/SerpApi

### 뉴스/트렌드/시장 레이더

1. Serper News 또는 SerpApi Google News
2. 원문 URL fetch
3. Exa로 관련 장문 분석/블로그 보강
4. Perplexity로 cross-source synthesis

### 기술 리서치/문헌/회사 조사

1. Exa semantic search
2. Perplexity reasoning/deep research
3. Serper/SerpApi로 최신성 검증
4. GBrain에 결과 저장/링크화

### Threads 콘텐츠 운영용 리서치

1. Serper/SerpApi로 최신 뉴스 신호 수집
2. Exa로 배경 문서/공식자료 탐색
3. Perplexity로 논점 정리
4. GBrain 또는 로컬 markdown에 Source / Claim / Insight 분리 저장

---

## 6. 재현/점검 명령어

```bash
# MCP 서버 목록
hermes mcp list

# 검색 관련 toolset 확인
hermes tools list | grep -Ei 'web|search|mcp|serp|exa|perplex|browser|fetch'

# config 위치
hermes config path

# config 전체 보기. 단, 외부 공유 전 반드시 secret redaction 필요
hermes config

# gateway 재시작 후 MCP 재발견 확인
hermes gateway restart
hermes mcp list
```

---

## 7. 공개 공유 시 보안 주의

공개 URL에 올릴 때 절대 포함하면 안 되는 값:

- `SERPER_API_KEY`
- `EXA_API_KEY`
- `PERPLEXITY_API_KEY`
- SerpAPI MCP URL의 긴 토큰 path
- OAuth token / refresh token
- Telegram bot token
- GitHub PAT / SSH private key
- `.env` 원문
- `auth.json` 원문

이 문서는 위 값을 모두 `[REDACTED]` 처리한 공유용 스냅샷입니다.
