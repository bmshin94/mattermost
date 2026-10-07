# 📊 Mattermost 전수조사 분석 정리 (한국어)

> 작성: 카리나(Claude Code) 💖
> 작성일: 2026-10-07
> 분석 대상 커밋: `15833681`
> 분석 브랜치: `claude/awesome-goldberg-mlsyt0`

---

## 🔗 깃허브 주소

| 구분 | 주소 |
|---|---|
| **이 레포 (포크)** | https://github.com/bmshin94/mattermost |
| **원본 (upstream)** | https://github.com/mattermost/mattermost |
| 공식 Docker 배포 레포 | https://github.com/mattermost/docker |
| 공식 홈페이지 | https://mattermost.com |
| REST API 문서 | https://api.mattermost.com |
| 개발자 문서 | https://developers.mattermost.com |
| 제품 문서 | https://docs.mattermost.com |
| 플러그인 마켓플레이스 | https://mattermost.com/marketplace/ |
| 커뮤니티 서버 | https://community.mattermost.com |

---

## 1. 이게 뭐야? (한 줄 요약)

> **Slack(슬랙)의 오픈소스 · 자체호스팅(self-hosted) 버전.**
> 내 서버에 직접 설치해서 쓰는 기업용 메신저 + 업무 자동화 + AI 통합 플랫폼.

- **언어/스택**: Go (백엔드) + React 19 / TypeScript (프론트엔드)
- **DB**: PostgreSQL
- **배포 형태**: 단일 Linux 바이너리
- **릴리스**: 매월 16일, 컴파일 버전은 MIT 라이선스로 공개
- **현재 버전**: `12.0.0` (`server/public/model/version.go` 기준)
- **레포 크기**: 약 746MB

### 핵심 가치 제안
데이터가 외부로 나가면 안 되는 조직(금융 · 의료 · 공공 · 국방 · 제조, 한국의 **망분리 환경**)에서
Slack / MS Teams를 대체할 수 있는 **거의 유일한 현실적 선택지**.

---

## 2. 폴더 구조 전수조사

| 폴더 | 정체 | 규모 / 내용 |
|---|---|---|
| `server/` | 🦫 백엔드 (Go) | **Go 파일 2,281개**, `go 1.26.7` |
| `webapp/` | ⚛️ 프론트엔드 (React) | **TS/TSX 4,388개**, React `19.2.8`, TS `5.6.3` |
| `api/` | 📖 OpenAPI 스펙 | YAML **58개** → api.mattermost.com 문서 생성원 |
| `e2e-tests/` | 🧪 E2E 테스트 | Cypress + Playwright 2종 |
| `docs/` | 📚 개발자 문서 | api, develop, main, pdf, site, styles, vendor |
| `tools/` | 🔧 자체 개발도구 | `mattermost-govet`(커스텀 린터), `mmgotool`(i18n), `sharedchannel-test` |
| `i18n/` | 🌏 다국어 | 한국어 포함 수십개 언어 |
| `.github/` | 🤖 CI/CD | Actions 워크플로우 + 커스텀 액션 11개 + CodeQL + Dependabot |
| `.cursor/` | 🤖 Cursor Cloud Agent 설정 | Dockerfile, environment.json, cloud-agent-start.sh |

### 2-1. `server/` 상세

```
server/
├── channels/              # 핵심 비즈니스 로직
│   ├── api4/              # REST API v4 핸들러 (모든 엔드포인트)
│   ├── app/               # 애플리케이션 서비스 레이어
│   ├── store/             # DB 레이어 (PostgreSQL)
│   ├── web/               # 웹 라우팅
│   ├── wsapi/             # WebSocket API (실시간 메시지)
│   ├── jobs/              # 백그라운드 잡 스케줄러
│   ├── audit/             # 감사 로그
│   ├── db/                # 마이그레이션
│   ├── testlib/           # 테스트 헬퍼
│   └── manualtesting/
├── public/                # ⭐ 별도 Go 모듈 (외부 공개용!)
│   ├── model/             # 데이터 모델 315개 파일
│   ├── plugin/            # 플러그인 시스템 (hashicorp RPC 기반)
│   ├── pluginapi/         # 플러그인 개발용 헬퍼 SDK
│   ├── shared/
│   └── utils/
├── platform/              # 공통 서비스 (services, shared)
├── einterfaces/           # 엔터프라이즈 인터페이스 정의
├── enterprise/            # 유료 기능 (git submodule, 상용 라이선스)
├── fips/                  # FIPS 암호화 인증 빌드
├── cmd/                   # CLI 엔트리포인트 (mattermost, mmctl)
├── config/                # 설정
├── templates/             # 이메일 템플릿
├── scripts/ · build/      # 빌드 스크립트, Dockerfile
└── docker-compose.yaml    # 로컬 개발용 (postgres, minio, inbucket, openldap, elasticsearch)
    docker-compose.pgvector.yml   # ⭐ 벡터 검색(RAG) 준비됨!
```

#### ⭐ 레이어드 아키텍처 (설계 공부 포인트)

```
요청
 ▼
① api4/   → HTTP 핸들러. 파라미터 파싱 / 형식 검증 / 응답 직렬화
 ▼
② app/    → 비즈니스 로직. 권한 체크, 이벤트 발행, 트랜잭션
 ▼
③ store/  → 데이터 접근. SQL 쿼리만
 ▼
PostgreSQL
```

관심사 분리가 철저해서, DB를 바꿔도 ③만 수정하면 되는 구조.
대규모 Go 프로젝트 설계 레퍼런스로 교과서급.

### 2-2. `webapp/` 상세 (npm workspaces 모노레포)

```
webapp/
├── channels/              # 메인 React 앱
│   └── src/
│       ├── components/    # UI 컴포넌트
│       ├── actions/       # Redux 액션
│       ├── reducers/      # Redux 리듀서
│       ├── selectors/     # Redux 셀렉터
│       ├── client/ · hooks/ · plugins/ · sass/ · i18n/ · utils/
│       └── entry.tsx · root.tsx
└── platform/              # 공용 패키지 (workspace)
    ├── client/            # ⭐ Client4 — JS/TS용 공식 API SDK
    │   └── src/client4.ts · websocket.ts
    ├── components/        # 공용 UI 컴포넌트
    ├── mattermost-redux/  # Redux 스토어 로직
    ├── types/             # TypeScript 타입
    ├── shared/
    └── eslint-plugin/     # 자체 ESLint 플러그인
```

---

## 3. 주요 기능

- 💬 **채팅** — 채널 / DM / 스레드 / 반응 / 북마크 / 예약 발송 (`scheduled_post`)
- 📞 **음성통화 + 화면공유**
- 🤖 **AI 통합** — `agents`, `llmservices`, `recaps`, `scheduled_recaps` API
- ⚙️ **워크플로우 자동화** — Playbooks (인시던트 대응)
- 🔌 **확장** — Plugin / Webhook / Slash Command / Apps / Bot / OAuth
- 🏢 **엔터프라이즈** — LDAP · SAML · SSO, 컴플라이언스, 데이터 보존,
  Elasticsearch, IP 필터, 감사로그, 액세스 제어(ABAC), Shared Channels,
  자동번역(`autotranslation`), 콘텐츠 플래깅, CPA(Custom Profile Attributes)

### 공식 유즈케이스
1. **DevSecOps** — 개발/보안팀 협업, CI/CD 알림
2. **Incident Resolution** — 장애 대응 (Playbooks)
3. **IT Service Desk** — 사내 헬프데스크

---

## 4. 설치 및 사용법

### 4-1. 그냥 쓰고 싶으면 (Docker, 가장 쉬움)

```bash
git clone https://github.com/mattermost/docker
cd docker
cp env.example .env
mkdir -p ./volumes/app/mattermost/{config,data,logs,plugins,client/plugins,bleve-indexes}
sudo chown -R 2000:2000 ./volumes/app/mattermost
docker compose up -d
```

→ `http://localhost:8080` 접속 후 관리자 계정 생성

### 4-2. 소스로 개발하고 싶으면 (이 레포)

**필요 버전 (레포 파일에서 확인한 정확한 값)**

| 도구 | 버전 | 근거 파일 |
|---|---|---|
| Go | `1.26.7` 이상 | `server/go.mod` |
| Node.js | `24.11.1` (정확히) | `.nvmrc`, `mise.toml` |
| npm | `11.6.2` | `webapp/package.json` → engines |
| Docker | 최신 | `server/docker-compose.yaml` |

```bash
cd server
make start-docker      # postgres, minio, inbucket, openldap 등 기동
make run               # 서버 + 웹앱 (= run-server + run-client)
# 접속: http://localhost:8065

make stop-server
make stop-docker
```

**웹앱만 따로**

```bash
cd webapp
npm install
npm run dev-server     # 핫리로드
```

### 4-3. 자주 쓰는 Makefile 타겟

```bash
make help                  # 전체 명령어
make check-style           # plugin-checker + vet + golangci-lint + check-go-fix
make test-server           # Go 테스트
make mocks                 # mock 전체 재생성
make store-layers          # store 레이어 코드 생성
make pluginapi             # 플러그인 glue 코드 생성
make i18n-extract          # 번역 문자열 추출
make modules-tidy          # go mod tidy 대체
make new-migration name=x  # DB 마이그레이션 생성
make generated             # 모든 자동생성 자산 재생성
make run-pgvector          # pgvector PostgreSQL 이미지로 실행
```

### 4-4. 🚨 반드시 지켜야 할 규칙 (`server/AGENTS.md`)

1. ❌ `go mod tidy` 직접 실행 금지 → ✅ **`make modules-tidy`**
   (private enterprise import 때문에 tidy가 깨짐)
2. `server/i18n/en.json` 수정 후 → 반드시 **`make i18n-extract`** (정렬 순서 재생성)
3. 요청 경로에서 로깅할 때는 **request-scoped logger** 선호.
   필요하면 메서드 시그니처에 `request.CTX` 추가 가능
4. **store 레이어에서는 `context.Context`를 시그니처에 쓰지 말 것.**
   `request.CTX`를 쓰고, 내부에서만 `rctx.Context()` 호출

### 4-5. AI 에이전트 친화 설정 (이 레포의 특징)

| 파일 | 역할 |
|---|---|
| `AGENTS.md` | 루트 AI 에이전트 규칙 (PR 템플릿 준수 등) |
| `server/AGENTS.md` | 서버 작업 규칙 (위 4-4) |
| `webapp/AGENTS.md` | 웹앱 작업 규칙 |
| `CLAUDE.OPTIONAL.md` | `webapp/`, `webapp/channels/src/`, `webapp/platform/` 에 존재 |
| `enable-claude-docs.sh` | `CLAUDE.OPTIONAL.md` → `CLAUDE.md` 복사 스크립트 |
| `.cursor/` | Cursor Cloud Agent 전용 환경 |

> ⚠️ `./enable-claude-docs.sh` 실행 시 기존 `CLAUDE.md`를 **덮어씀**.
> 이 레포 루트의 `CLAUDE.md`(카리나 페르소나)가 날아갈 수 있으니 주의.
> 또한 `.gitignore` 170~171행에서 `**/CLAUDE.md`를 무시하도록 설정되어 있음.

### 4-6. PR 작성 규칙 (`AGENTS.md`)

- `.github/PULL_REQUEST_TEMPLATE.md` 를 **정확히** 따를 것
- `<!-- -->` 주석은 전부 제거
- 해당 없는 섹션(Ticket Link, Screenshots)은 헤더까지 삭제 (N/A 쓰지 말 것)
- `#### Release Note` 헤더와 ` ```release-note ` 코드블록은 **항상 포함**.
  변경사항 없으면 `NONE` 기재

---

## 5. 플러그인? 스킬? MCP? → **플랫폼(본체)**

| 구분 | 정체 | 규모 | 비유 |
|---|---|---|---|
| **Mattermost** | 🏢 독립 실행 애플리케이션 | 746MB, Go + React | 안드로이드 **OS** |
| 플러그인 | 앱 확장 모듈 | 수MB | 안드로이드 **앱** |
| 스킬 (Claude Skill) | AI 지침서 (.md) | 수KB | **설명서** |
| MCP | AI ↔ 도구 연결 프로토콜 | 서버 1개 | **USB 규격** |

Mattermost는 **이 셋을 모두 품을 수 있는 땅**:

- **플러그인을 받는 쪽** — `server/public/plugin/` (hashicorp RPC 기반 콘센트)
- **MCP 서버가 될 수 있음** — REST API를 MCP로 감싸면 Claude가 메시지를 읽고 쓸 수 있음
- **스킬의 대상** — `CLAUDE.md` / `AGENTS.md` / `.cursor/` 가 이미 준비되어 있음

---

## 6. API 토큰, 꼭 써야 해?

### 인증 방식 4가지

| # | 방식 | 토큰 | 용도 |
|---|---|---|---|
| 1 | 세션 쿠키 | ❌ | 사람이 웹/앱 로그인 (자동) |
| 2 | Personal Access Token | ✅ | 내 스크립트 / 자동화 |
| 3 | **Bot Account Token** | ✅ | 봇 제작 ⭐ 권장 |
| 4 | OAuth 2.0 | ✅ | 외부 서비스 연동 |

### 토큰 발급

```
① System Console → Integrations → Integration Management
   → "Enable Personal Access Tokens" 활성화
② Integrations → Bot Accounts → Add Bot Account → 토큰 복사 (1회만 노출)
```

### 사용 예

**cURL**
```bash
curl -H "Authorization: Bearer <TOKEN>" \
     https://your-server.com/api/v4/users/me
```

**TypeScript** — 레포 내장 SDK `webapp/platform/client/src/client4.ts`
```typescript
import {Client4} from '@mattermost/client';

const client = new Client4();
client.setUrl('https://your-server.com');
client.setToken(process.env.MM_TOKEN!);

await client.createPost({
  channel_id: channelId,
  message: '안녕! 💖',
});
```

**Go** — 레포 내장 `server/public/model`
```go
import "github.com/mattermost/mattermost/server/public/model"

client := model.NewAPIv4Client("https://your-server.com")
client.SetToken(os.Getenv("MM_TOKEN"))
post, _, err := client.CreatePost(ctx, &model.Post{
    ChannelId: channelID,
    Message:   "안녕! 👋",
})
```

### 토큰 없이 되는 경우

- **Incoming Webhook** — URL 자체가 시크릿
  ```bash
  curl -X POST -d '{"text":"배포 완료 🚀"}' \
       https://your-server.com/hooks/<HOOK_ID>
  ```
- **플러그인 개발** — 서버 내부 실행이라 토큰 불필요. `pluginapi.Client` 직접 사용

### 보안 체크리스트

- [ ] 토큰을 git에 커밋하지 않기 (`.env` + `.gitignore`)
- [ ] 봇 토큰은 최소 권한만 부여 (System Admin 금지)
- [ ] 외부 노출 시 HTTPS 필수
- [ ] 토큰 주기적 로테이션

---

## 7. AI 에이전트 구축에 도움이 될까? → **매우 도움됨**

### 직접 만들면 7개월, Mattermost 쓰면 0일

| 필요 기능 | 자체 개발 | Mattermost |
|---|---|---|
| 채팅 UI (웹 · 모바일 · 데스크톱) | ~3개월 | ✅ 내장 |
| 로그인 / 권한 / 조직관리 | ~1개월 | ✅ 내장 |
| 대화 이력 DB | ~2주 | ✅ PostgreSQL |
| 실시간 스트리밍 | ~2주 | ✅ WebSocket |
| 파일 업로드 | ~1주 | ✅ 내장 |
| 알림 (푸시 · 이메일) | ~2주 | ✅ 내장 |
| 다국어 | ~1주 | ✅ i18n |
| LLM 연동 | 직접 | ✅ agents API |

### 내장 AI API (`api/v4/source/agents.yaml`, 서버 11.2+)

```
GET  /api/v4/agents            # 사용 가능한 에이전트 목록 (유저별 필터링)
GET  /api/v4/agents/status     # AI 플러그인 브릿지 상태 (가용성 + 사유 코드)
GET  /api/v4/llmservices       # 사용 가능한 LLM 서비스 목록
     /api/v4/recaps            # AI 대화 요약
     /api/v4/scheduled_recaps  # 예약 요약
```

관련 모델 파일: `server/public/model/agents.go`, `ai_recap_settings.go`,
`ai_recap_limits.go`, `ai_bridge_test_helper.go`, `autotranslation.go`

### 구축 난이도별 4가지 방법

**Lv.1 — Webhook 봇 (반나절)**
```
Outgoing Webhook → 외부 서버(PHP/Node/Python) → LLM API → Incoming Webhook
```

**Lv.2 — Bot Account + WebSocket (2~3일) ⭐ 추천**
```python
ws = connect('wss://server/api/v4/websocket', token=BOT_TOKEN)
for event in ws:
    if event.type == 'posted' and bot_mentioned(event):
        answer = call_llm(event.message)
        client.create_post(channel_id, answer)
```

**Lv.3 — Go 플러그인 (1~2주)**
```go
func (p *Plugin) MessageHasBeenPosted(c *plugin.Context, post *model.Post) {
    if !mentionsMe(post) { return }
    reply := p.callLLM(post.Message)
    p.client.Post.CreatePost(&model.Post{
        ChannelId: post.ChannelId,
        Message:   reply,
        RootId:    post.Id,   // 스레드 답장
    })
}
```

**Lv.4 — 내장 agents API 연동 (최신)**
Mattermost가 LLM 브릿지를 공식 관리. 플러그인 브릿지를 통해 접근.

### 멀티 에이전트 구성 예

```
#프로젝트 채널
 ├─ 🤖 코드리뷰봇   → PR 보안/품질 리뷰
 ├─ 🤖 번역봇       → 영↔한 자동 번역
 ├─ 🤖 요약봇       → 일일 대화 요약
 └─ 🤖 PM봇        → Jira 티켓 자동 생성
```

채널이 에이전트들의 공용 워크스페이스가 되고, 사람도 같은 채널에서 보므로
**Human-in-the-loop** 가 자연스럽게 성립.

### 플랫폼 비교

| 플랫폼 | 자체호스팅 | 무료 | 소스수정 | AI 내장 |
|---|---|---|---|---|
| Slack | ❌ | 제한 | ❌ | △ |
| Discord | ❌ | ✅ | ❌ | ❌ |
| MS Teams | ❌ | ❌ | ❌ | ✅ (유료) |
| **Mattermost** | ✅ | ✅ | ✅ | ✅ |

---

## 8. React / PHP 로 만들 수 있어?

### ⚛️ React — 완전 가능 (3가지 방법)

**① 웹앱 직접 수정**
`webapp/channels/src/components/` 수정. React 19.2.8 + TS 5.6 + Redux + react-intl.

**② 플러그인 웹앱 파트 ⭐ 추천**
```javascript
export default class Plugin {
    initialize(registry, store) {
        registry.registerRootComponent(MyPanel);
        registry.registerChannelHeaderButtonAction(icon, action, dropdownText);
        registry.registerPostTypeComponent('custom_ai', AIPostView);
        registry.registerRightHandSidebarComponent(Sidebar, title);
    }
}
```
본체를 건드리지 않고 React UI를 추가 — 가장 깔끔.

**③ 완전히 새 React 앱**
```typescript
import {Client4} from '@mattermost/client';
// Mattermost를 백엔드로만 쓰고 프론트는 Next.js 등으로 새로 제작
```

### 🐘 PHP — 되는 것 / 안 되는 것

| 작업 | PHP | 설명 |
|---|---|---|
| 서버 본체 재작성 | ❌ | Go 2,281 파일 — 비현실적 |
| 서버 플러그인 | ❌ | Go 전용 (hashicorp go-plugin RPC) |
| REST API 호출 | ✅ | Guzzle 등으로 쉽게 |
| **Webhook 봇** | ✅ ⭐ | PHP에 최적 |
| **Slash Command 처리** | ✅ ⭐ | POST 폼 받으면 끝 |
| OAuth 연동 | ✅ | |
| 웹앱 UI | ❌ | React 전용 |

```php
<?php
// Incoming Webhook (토큰 불필요)
$ch = curl_init('https://your-server.com/hooks/<HOOK_ID>');
curl_setopt_array($ch, [
    CURLOPT_POST => true,
    CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
    CURLOPT_POSTFIELDS => json_encode([
        'text'        => '🚀 PHP에서 보냈습니다',
        'username'    => 'deploy-bot',
        'icon_emoji'  => ':sparkles:',
    ]),
]);
curl_exec($ch);
```

```php
<?php
// Bot Token으로 REST API 호출 (Guzzle)
$client = new \GuzzleHttp\Client([
    'base_uri' => 'https://your-server.com/api/v4/',
    'headers'  => ['Authorization' => 'Bearer ' . getenv('MM_BOT_TOKEN')],
]);
$client->post('posts', ['json' => [
    'channel_id' => $channelId,
    'message'    => '라라벨에서 보낸 메시지',
]]);
```

### 추천 조합

```
Mattermost 서버 (Go, 수정 없이 그대로)
     ├── ⚛️ React 플러그인 UI
     └── 🐘 PHP(Laravel) 봇 서버 → 🧠 LLM API
```

Go를 몰라도 React + PHP만으로 충분히 구현 가능.

---

## 9. 유튜브 강의 제작 가능성 → **가능, 블루오션**

### 장점
- 오픈소스이므로 화면 녹화 / 강의 제작 자유
- **한국어 콘텐츠가 거의 없음** (영어도 적음)
- "슬랙 대안", "망분리 메신저", "사내 메신저 구축" 검색 수요 꾸준
- B2B 시청자 → 외주 · 컨설팅 문의로 직결 (고가치)
- 난이도 스펙트럼이 넓음 (설치 ~ Go 플러그인)

### 단점 / 대응
- 백엔드·인프라 주제라 대중성 낮음 → 조회수보다 전환율로 승부
- 버전 업데이트가 빨라 영상이 빨리 낡음 → **버전 명시 + 깃허브 완성 코드 제공**
- 환경 구축 영상은 "안 돼요" 문의 폭주 → 트러블슈팅 편 별도 제작

### 커리큘럼 (25편)

**시즌 1 — 입문 (5편)**
1. 슬랙 월 100만원? 공짜로 우리 회사 메신저 만들기 (10분)
2. Docker로 10분 설치 따라하기 (15분)
3. 관리자 콘솔(System Console) 완전정복 (20분)
4. HTTPS + 도메인 연결 (Nginx 리버스 프록시) (15분)
5. 백업 · 복구 · 업그레이드 실무 (15분)

**시즌 2 — 연동 (5편)**
6. Webhook으로 깃허브 알림 받기 (코드 없이)
7. Slash Command 만들기 (`/날씨`)
8. PHP로 봇 만들기
9. JS SDK (Client4) 완전정복
10. OAuth 2.0 외부 서비스 연동

**시즌 3 — AI 에이전트 (6편) 🔥**
11. 사내 ChatGPT 만들기 (WebSocket + LLM API)
12. 사내 문서를 읽는 RAG 챗봇 (pgvector)
13. agents API로 LLM 공식 연동
14. AI 코드리뷰 봇 (PR 자동 리뷰)
15. 멀티 AI 에이전트 협업 시스템
16. MCP 서버 만들어 Claude에 연결

**시즌 4 — 개발 심화 (5편)**
17. 소스 빌드 + 개발환경 (`make run`)
18. React로 UI 플러그인 만들기
19. Go 플러그인 입문
20. 코드 구조 해부 (api4 → app → store)
21. 마켓플레이스에 플러그인 등록하기

**시즌 5 — 기업 실무 (4편)**
22. LDAP / Active Directory 연동
23. 망분리 환경 구축 (금융 · 공공)
24. Kubernetes / Helm 대규모 배포
25. Slack → Mattermost 데이터 마이그레이션

### 수익 구조 (중요도 순)
1. 💎 **외주 / 컨설팅 유입** — 건당 수백~수천만원 (가장 큼)
2. 🏫 기업 사내교육 강의 — 일 100~300만원
3. 🎓 유료 강의 (인프런 등)
4. 💬 멤버십 / 1:1 멘토링
5. 🔌 제작 플러그인 판매
6. 📺 애드센스 광고 (가장 작음)

> 유튜브는 광고 수익이 아니라 **영업 깔때기(funnel)** 로 접근할 것.

### 제목 전략
| ❌ 나쁜 예 | ✅ 좋은 예 |
|---|---|
| Mattermost 설치 방법 | 슬랙 월 100만원 아끼는 법 |
| Mattermost 소개 | 회사 데이터, 미국 서버에 보내지 마세요 |
| AI 봇 만들기 | ChatGPT를 우리 회사 메신저에 넣어봤다 |
| 코드 분석 | Go 파일 2,281개 전부 분석해봤습니다 |

---

## 10. 수익화 아이디어 (상세)

### 🥇 ① SI 구축 + 컨설팅 — ★★★★★ (가장 현실적)

**시장 배경 (한국 특화)**
망분리 규제로 클라우드 메신저 사용이 금지된 영역이 광범위:
전자금융감독규정(금융), 국정원 보안지침(공공), 의료법(의료), 보안측정(방산).
→ Slack / Teams 사용 불가 → **온프레미스가 유일한 선택**

**단가표 (업계 통상)**

| 서비스 | 단가 |
|---|---|
| 기본 구축 (Docker, ~100명) | 300 ~ 800만원 |
| K8s HA 구축 (1,000명+) | 2,000 ~ 5,000만원 |
| LDAP / SSO 연동 | 300 ~ 1,000만원 |
| Slack 데이터 마이그레이션 | 500 ~ 1,500만원 |
| 커스텀 플러그인 개발 | 500 ~ 3,000만원 |
| **연간 유지보수** ⭐ | **구축비의 15~20% / 년** |

> 핵심은 **유지보수 계약**. 고객 10곳 × 연 500만원 = 연 5,000만원 안정 수입.

**진입 전략**
1. 내 서버에 구축 → 과정을 블로그/유튜브로 공개
2. 포트폴리오 사이트 + 무료 상담 CTA
3. 소기업 1~2곳 저가 구축 (레퍼런스 확보)
4. 레퍼런스로 중견기업 영업
5. 공공 조달(나라장터) 진입

**⚠️ 라이선스 주의**
AGPL이므로 "고객 서버에 설치해 주는 것"은 문제없음.
그러나 **코드를 수정해 SaaS로 제공하면 수정분 소스 공개 의무** 발생.
→ 커스터마이징은 반드시 **플러그인 형태**로 분리할 것.

---

### 🥈 ② 플러그인 개발 · 판매 — ★★★★☆

**유통 채널**
1. 공식 마켓플레이스 (700개 이상 등록, 노출 좋음)
2. 직접 판매 (Gumroad, Lemon Squeezy, 자체 사이트)
3. GitHub Sponsors (오픈소스 + 후원)

**아이디어 (한국 특화 포함)**

| 플러그인 | 수요 근거 | 예상 가격 |
|---|---|---|
| 🤖 AI 어시스턴트 (Claude/GPT) | 수요 폭발 | $10~30 / 월 / 유저 |
| 📋 전자결재 · 승인 워크플로 | 🇰🇷 한국 기업 필수 | 500만원 (온프레) |
| 🗓️ 근태 · 출퇴근 관리 | 주 52시간 대응 | $5 / 월 / 유저 |
| 📊 AI 주간보고 자동생성 | 전 직장인 공감 | $200 / 월 / 팀 |
| 🔒 DLP (정보유출방지) | 금융권 필수 | 1,000만원+ |
| 🌏 AI 실시간 번역 | 해외지사 보유 기업 | $8 / 월 / 유저 |
| 🎫 Jira / Redmine 심화연동 | 개발팀 | $300 / 월 |
| 📞 카카오 알림톡 연동 | 🇰🇷 한국만 가능 | $100 / 월 |
| 🏢 국내 그룹웨어 연동 (더존·한컴) | 🇰🇷 틈새 독점 | 협의 |
| 📝 회의록 자동작성 (STT + AI) | 수요 큼 | $15 / 월 / 유저 |

**1순위 추천 — "AI 주간보고 생성기"**
- 한국 직장인 전원이 공감하는 페인포인트
- 기술 난이도 낮음 (대화 수집 + LLM 요약)
- 효과 즉시 체감 → 결제 전환율 높음
- 데모 영상 제작 쉬움 → 유튜브 바이럴 가능

**가격 설계**
```
Free        : 1채널, 월 10회            → 유입
Pro   $9/월 : 무제한, 커스텀 템플릿
Team $29/월 : 팀 통계, 관리자 기능
Enterprise  : 온프레미스 라이선스 (연 500만원~)  ← 실제 수익원
```

---

### 🥉 ③ AI 에이전트 SaaS — ★★★★☆ (최대 포텐)

**컨셉: "회사 전용 AI 비서, 데이터는 외부로 나가지 않음"**

```
고객사 내부망
 ┌──────────────────────────┐
 │  Mattermost              │
 │       ↕                  │
 │  🤖 AI 에이전트 (상품)    │
 │       ↕                  │
 │  📚 사내 문서 RAG         │
 │  (위키 · PDF · 코드)      │
 └──────────────────────────┘
   데이터 외부 유출 없음
```

**차별점**
ChatGPT Enterprise는 데이터가 OpenAI로 전송됨 → 금융권 통과 불가.
온프레미스 LLM(Llama, Qwen) 또는 프라이빗 엔드포인트 사용 →
**"보안 심사를 통과하는 AI"** 포지셔닝.

**가격**

| 플랜 | 가격 |
|---|---|
| 스타트업 (50명) | 월 100만원 |
| 중견 (500명) | 월 500만원 |
| 대기업 온프레미스 | 연 5,000만원~ |

**기술 스택**
```
프론트  : Mattermost 플러그인 (React)
백엔드  : PHP Laravel 또는 Node.js
AI      : Claude API 또는 온프레미스 Llama
벡터DB  : pgvector  ← server/docker-compose.pgvector.yml 이미 존재!
```

---

### ④ 매니지드 호스팅 — ★★★☆☆

중소기업 타겟 "설치·운영 몰라도 되는 Mattermost"

| 플랜 | 가격 | 내용 |
|---|---|---|
| Starter (20명) | 월 5만원 | 공유 인프라 |
| Business (100명) | 월 20만원 | 전용 인스턴스, 자동 백업 |
| Dedicated | 월 50만원+ | 전용서버, SLA, 전화지원 |

```
고객 50개 × 월 20만원 = 월 1,000만원
인프라 원가 ≈ 월 300만원
→ 순익 월 700만원 (자동화 시 거의 패시브)
```

**⚠️** AGPL — 수정 없이 그대로 호스팅은 문제없음. 수정 시 공개 의무 발생.

---

### ⑤ 교육 · 콘텐츠 — ★★★★☆ (초기자본 0원)

| 상품 | 가격 |
|---|---|
| 유튜브 애드센스 | 월 10~100만원 |
| 인프런 / 클래스101 강의 | 10~15만원 × 수강생 |
| 전자책 (PDF) | 3~5만원 |
| **기업 사내교육** ⭐ | 일 100~300만원 |
| 1:1 멘토링 | 시간당 10~20만원 |
| 세미나 · 웨비나 | 회당 50~200만원 |

**선순환 구조**
```
유튜브 영상 → 신뢰 획득 → 블로그/깃허브 포트폴리오 → 문의 유입
 → 외주 구축(수백~수천만원) → 실전 경험 축적 → 더 좋은 영상 → 반복
```

---

### ⑥ 테마 · 템플릿 · 소규모 상품 — ★★☆☆☆

| 상품 | 가격 |
|---|---|
| 커스텀 테마 팩 | $20 ~ 50 |
| 이모지 · 스티커 팩 | $5 ~ 15 |
| Playbook 템플릿 (장애대응, 온보딩) | $30 ~ 100 |
| 원클릭 설치 스크립트 | $20 ~ 50 |
| Bot 보일러플레이트 | $30 ~ 80 |

금액은 작지만 한 번 제작하면 지속 판매 (패시브 인컴).

---

### 종합 비교

| 아이디어 | 초기비용 | 난이도 | 수익규모 | 수익화 속도 | 추천도 |
|---|---|---|---|---|---|
| ① SI 구축 · 컨설팅 | 거의 0 | 중 | 💰💰💰💰 | 빠름 | ⭐⭐⭐⭐⭐ |
| ② 플러그인 판매 | 낮음 | 중상 | 💰💰💰 | 중간 | ⭐⭐⭐⭐ |
| ③ AI 에이전트 SaaS | 중간 | 상 | 💰💰💰💰💰 | 느림 | ⭐⭐⭐⭐ |
| ④ 매니지드 호스팅 | 높음 | 중 | 💰💰💰 | 중간 | ⭐⭐⭐ |
| ⑤ 교육 콘텐츠 | 0 | 하 | 💰💰 | 느림 | ⭐⭐⭐⭐ |
| ⑥ 테마 · 템플릿 | 0 | 하 | 💰 | 빠름 | ⭐⭐ |

---

### 실행 로드맵 (React + PHP 기준)

**1~3개월 — 기반 다지기** (수익 0원, 투자 기간)
- [ ] Docker로 Mattermost 설치 완주
- [ ] PHP Webhook 봇 제작
- [ ] React 플러그인 "Hello World" 성공
- [ ] 유튜브 3편 업로드 (설치 / Webhook / 봇)
- [ ] 깃허브 코드 공개 + 블로그 1편

**4~6개월 — 첫 수익** (월 50~300만원)
- [ ] "AI 주간보고 생성기" 플러그인 완성
- [ ] Gumroad 판매 시작 ($9/월)
- [ ] 유튜브 10편 돌파 → 첫 외주 문의
- [ ] 소기업 1곳 저가 구축 (레퍼런스)

**7~12개월 — 본격화** (월 500~2,000만원)
- [ ] 플러그인 2~3개로 확장
- [ ] 공식 마켓플레이스 등록
- [ ] 중견기업 구축 1~2건 (건당 500~1,000만원)
- [ ] 인프런 강의 오픈
- [ ] AI 에이전트 SaaS MVP

**1년 이후 — 스케일** (월 2,000만원+)
- [ ] 유지보수 계약 10곳 (연 5,000만원 안정수입)
- [ ] AI SaaS 유료 고객 확보
- [ ] 공공 조달(나라장터) 진입
- [ ] 팀 구성 / 법인 설립

**최종 추천:** ① SI 구축 + ⑤ 유튜브를 **동시에** 시작.
유튜브는 초기자본 0원으로 신뢰 자산을 쌓고, 그 신뢰가 외주로 전환되며,
외주 경험이 플러그인 아이디어가 되고, 플러그인이 AI SaaS로 확장되는 구조.

---

## 11. 라이선스 주의사항 ⚖️

| 범위 | 라이선스 |
|---|---|
| 대부분의 소스 | **AGPL-3.0** 및 Apache 2.0 혼합 (`LICENSE.txt`) |
| `server/enterprise/` | **Mattermost Enterprise 상용 라이선스** (submodule, `LICENSE.enterprise`) |
| 매월 16일 컴파일 릴리스 | MIT |
| 서드파티 의존성 | `NOTICE.txt` (679KB) 참조 |

**AGPL 핵심**
네트워크를 통해 서비스로 제공하는 경우에도 **수정한 소스를 공개할 의무**가 발생.
→ 상업 서비스 시:
- 코드 수정 없이 그대로 배포/호스팅 → 안전
- 커스터마이징은 **플러그인(별도 바이너리)** 으로 분리 → 안전한 경로
- 본체 포크 수정 후 SaaS 제공 → **공개 의무 발생**
- 불확실하면 Mattermost 상용 라이선스 문의 또는 법률 검토 권장

---

## 12. 분석 요약 (수치)

| 항목 | 값 |
|---|---|
| 레포 크기 | 746 MB |
| Go 파일 | 2,281개 |
| TS / TSX 파일 | 4,388개 |
| OpenAPI 스펙 파일 | 58개 |
| 데이터 모델 파일 | 315개 |
| Go 버전 | 1.26.7 |
| Node 버전 | 24.11.1 |
| npm 버전 | 11.6.2 |
| React 버전 | 19.2.8 |
| TypeScript 버전 | 5.6.3 |
| 제품 버전 | 12.0.0 |
| 마켓플레이스 통합 | 700개+ |
| GitHub 커스텀 액션 | 11개 |

---

## 13. 참고 문서 링크

### 공식
- 개발환경 설정: https://developers.mattermost.com/contribute/server/developer-setup
- 웹앱 개발 문서: https://developers.mattermost.com/contribute/more-info/webapp/
- 플러그인 개발: https://developers.mattermost.com/integrate/plugins/
- REST API 레퍼런스: https://api.mattermost.com
- 배포 가이드: https://docs.mattermost.com/guides/deployment.html
- Docker 설치: https://docs.mattermost.com/install/install-docker.html
- Kubernetes / Helm: https://docs.mattermost.com/install/install-kubernetes.html
- 기여 가이드: https://developers.mattermost.com/contribute/getting-started/
- 보안 공지 구독: https://mattermost.com/security-updates/#sign-up

### 레포 내부
- `README.md` — 프로젝트 개요
- `CONTRIBUTING.md` — 기여 절차
- `SECURITY.md` — 보안 취약점 신고
- `AGENTS.md`, `server/AGENTS.md`, `webapp/AGENTS.md` — AI 에이전트 작업 규칙
- `webapp/README.md`, `webapp/STYLE_GUIDE.md` — 웹앱 가이드
- `api/README.md`, `api/CONTRIBUTING.md` — API 문서 빌드
- `e2e-tests/README.md` — E2E 테스트 실행
- `tools/README.md` — 자체 개발도구
- `.github/e2e-tests-workflows.md` — CI E2E 워크플로우
- `.cursor/README.md` — Cursor Cloud Agent

---

*이 문서는 `bmshin94/mattermost` 레포지토리를 전수조사하여 작성되었습니다.*
*분석/작성: 카리나 (Claude Code) 💖*
