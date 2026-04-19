# gws-harness

Google Workspace 하네스 — `gws` CLI 기반 Claude Code 플러그인

Gmail 트리아지, 캘린더 타임블록, 아침 브리핑, 메일→티켓 변환, 주간 다이제스트를 대화형으로 제공합니다.

---

## 한눈에 보기

### 무엇을 해주나

매일 반복되는 Google Workspace 작업을 Claude Code에서 **자연어 한 줄**로 처리합니다.

- **"메일 정리해줘"** → 미읽음 메일을 긴급/액션/FYI/무시 4단계로 분류 + 액션 아이템 추출
- **"오늘 뭐해?"** → 메일 요약 + 캘린더 일정 + Jira 스프린트 상태를 한 장 브리핑
- **"이번 주 타임블록 짜줘"** → Jira 티켓을 스토리 포인트 기반 시간 블록으로 캘린더에 자동 배치
- **"이 메일 티켓으로"** → Gmail 메일에서 Jira 이슈 또는 Notion 페이지 자동 생성
- **"이번 주 정리해줘"** → 메일 송수신 / 미팅 비중 / 완료 티켓 집계 + 인사이트

### 누구에게 맞나

- 매일 아침 "오늘 뭐부터 할지" 5분 안에 정리하고 싶은 실무자
- **회사 + 개인 메일 2계정**을 한 번에 조회하고 싶은 사람 (v1.1.0+)
- Jira 티켓을 캘린더에 미리 박아두는 스프린트 플래너
- 주간 회고에 감이 아닌 숫자가 필요한 팀 리드 / PM
- 오픈소스 사용자 — Jira/Notion 없이 Gmail + Calendar만으로도 동작

### 차별점

- **2계정 병렬 처리**: 회사 + 개인 메일/캘린더를 동시에 조회하되 쓰기는 단일 계정에만 (사고 방지)
- **시간대 가중치**: 평일 09-18시엔 회사 메일 강조, 저녁/주말엔 개인 메일 강조 (시스템 시간대와 무관하게 한국 시간 기준)
- **캐시 재사용**: 같은 날 두 번째 호출은 4밀리초 (첫 호출 대비 358배 빠름)
- **쓰기 안전장치**: 메일 발송 / 캘린더 이벤트 생성 / 티켓 생성 등 모든 외부 쓰기는 사용자 승인 후 실행
- **보안 기본값**: credential과 메일 캐시 폴더는 자동으로 `chmod 0700`, Time Machine 제외, iCloud/Dropbox 동기화 차단

### 3단계로 시작하기

```
1. gws CLI 설치 + 인증
2. /plugin marketplace add jangwonyoon/gws-harness
3. /gwh:credential-init work   →  /gwh:triage
```

상세 사용 예시는 [docs/SCENARIOS.md](docs/SCENARIOS.md) 참조.

---

## 사전 요구사항

### 1. gws CLI 설치

```bash
# GitHub 릴리즈에서 설치
# https://github.com/googleworkspace/cli/releases
```

### 2. Google Workspace 인증

```bash
gws auth setup   # 최초 1회: OAuth 클라이언트 설정
gws auth login   # 로그인
gws auth status  # 인증 상태 확인
```

### 3. 플러그인 설치

Claude Code에서:

```
# 1) 마켓플레이스 등록 (최초 1회)
/plugin marketplace add jangwonyoon/gws-harness

# 2) 플러그인 설치
/plugin install gws-harness@gws-harness
```

설치 후 `/gwh:*` 명령이 활성화됩니다.

### 4. 계정 세팅 (v1.1.0+ 2계정 지원)

```
/gwh:credential-init work       # 회사 계정 등록
/gwh:credential-init personal   # 개인 계정 등록 (optional)
/gwh:triage                     # 모든 등록 계정 병렬 트리아지
```

- 기존 `~/.config/gws/` 단일 구조 사용자는 `/gwh:credential-init <name>` 실행 시 자동 migration (AskUserQuestion 승인 경유)
- 계정 하나만 등록해도 기존 사용법 그대로 동작 (merged 파일 = 라벨 없음)

## 스킬 목록

| 스킬 | 설명 | 트리거 |
|------|------|--------|
| `/gwh:credential-init` | 계정 등록 (work/personal) + 기존 gws/ auto-migrate | "계정 추가", "credential init" |
| `/gwh:triage` | Gmail 미읽음 메일 분류 + 액션 추출 (2계정 병렬) | "메일 정리", "메일 트리아지" |
| `/gwh:brief` | 아침 브리핑 (메일+캘린더+Jira, 2계정 통합) | "브리핑", "오늘 뭐해" |
| `/gwh:cal-plan` | Jira 티켓 → 캘린더 타임블록 (`--account <X>` 필수) | "일정 짜줘", "타임블록" |
| `/gwh:mail-to-ticket` | 메일 → Jira/Notion 티켓 | "메일 티켓으로" |
| `/gwh:digest` | 주간 활동 다이제스트 (2계정 통합) | "주간 요약", "이번 주 정리" |

## 코어 vs 확장

| 기능 | 필수 | 선택 |
|------|------|------|
| Gmail 트리아지 | gws CLI | — |
| 캘린더 조회/생성 | gws CLI | — |
| Jira 스프린트 연동 | — | Atlassian MCP |
| Notion 페이지 생성 | — | Notion MCP |

**Gmail + Calendar만으로 핵심 기능이 모두 동작합니다.** Jira/Notion MCP가 연결되면 자동으로 확장 기능이 활성화됩니다.

## 산출물

모든 스킬의 결과는 `~/.gwh/`에 저장됩니다:

```
~/.gwh/
├── triage-2026-04-17.md        # 메일 트리아지 결과
├── brief-2026-04-17.md         # 아침 브리핑
├── cal-plan-2026-04-17.md      # 타임블록 배치
├── ticket-2026-04-17-*.md      # 생성된 티켓 정보
└── digest-2026-W16.md          # 주간 다이제스트
```

산출물은 스킬 간 공유됩니다 (예: brief가 triage 결과를 재사용 — 실측 358배 캐시).

2계정 환경에서는 per-account 파일(`triage-work-{date}.md`)과 merged 파일(`triage-{date}.md`)이 함께 생성됩니다.

## Security / Privacy

v1.1.0부터 계정별 credential이 `~/.config/gws-<name>/`에 격리되고, `~/.gwh/` 캐시 디렉토리는 메일 본문 스니펫을 포함하므로 다음 보안 조치를 자동 적용합니다:

| 조치 | 대상 | 적용 시점 |
|------|------|----------|
| `chmod 0700` | `~/.gwh/`, `~/.config/gws-*/` | `/gwh:credential-init`, 모든 스킬의 산출물 저장 Step |
| `chmod 0600` | 모든 산출물 파일 + `config.yml` | 파일 생성 직후 inline |
| `tmutil addexclusion` (user-level) | `~/.gwh/` | `/gwh:credential-init` Step 6 (idempotent, sudo 불필요) |
| Cloud-parent REFUSE | `~/.gwh/` 경로 | `/gwh:credential-init` 실행 시 `realpath` 검사 |

### 수동 설정이 필요한 경우

`~/.gwh/`를 아래 경로 하위에 두면 자동 차단되므로, **로컬 경로**로 옮긴 뒤 재실행하세요:

- `~/Library/Mobile Documents/` (iCloud Drive)
- `~/Library/CloudStorage/` (Dropbox/OneDrive 통합)
- `~/Dropbox/`, `~/OneDrive/`

### keyring backend 관련

2계정 지원을 위해 keyring은 **file 백엔드**(`GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file`)를 사용합니다. 이는 기본 Keychain 대비 보안이 하락하는 trade-off입니다:

- credential 암호화 키가 `~/.config/gws-<name>/` 내부 평문 파일로 저장됨
- 위 `chmod 0700` + `tmutil addexclusion` + cloud-parent REFUSE로 보완
- 기업 MDM/backup agent 환경에서는 별도 정책 조정 권장

## Upgrade Notes (1.0.0 → 1.1.0)

- **자동 migration**: 기존 `~/.config/gws/` 단일 경로 사용자는 `/gwh:credential-init <name>` 실행 시 3-way 분기로 안전 migration (AskUserQuestion 승인 경유, same-device 검증, SIGINT trap).
- **경로 하드코딩 주의**: 외부 스크립트가 `~/.config/gws/`를 참조하고 있으면 migration 후 경로 조정 필요 (`~/.config/gws-<name>/`).
- **merged 파일 호환**: 단일 계정 사용 시 기존 `~/.gwh/triage-{date}.md` 포맷 그대로 유지 (R8). 2계정 등록 시부터 라벨 prefix + frontmatter 포함.

## 더 읽기

| 문서 | 대상 독자 | 내용 |
|------|----------|------|
| [docs/USE_CASES.md](docs/USE_CASES.md) | 처음 써보는 사람 / 플러그인 검토자 | 페르소나 6개 × "누가 / 액션 플랜 / 아웃풋" 정리 |
| [docs/SCENARIOS.md](docs/SCENARIOS.md) | 실제 사용 흐름이 궁금한 사람 | 월요일 아침부터 금요일 회고까지 구체적 대화 예시 6개 |
| [AGENTS.md](AGENTS.md) | 기여자 / 새 스킬 추가하려는 사람 | 스킬 추가 절차, 네이밍 규칙, PR 워크플로우, 2계정 패턴 재사용 |
| [AGENT.md](AGENT.md) | 오케스트레이터 동작 이해 | 자연어 요청이 어떤 `/gwh:*` 스킬로 라우팅되는지, 코어 vs 확장 구분 |
| [skills/shared/references/multi-account.md](skills/shared/references/multi-account.md) | 2계정 구조 내부 동작 궁금한 사람 | 계정 인증 검증, 병렬 fetch, dual write, 보안 기본값 설계 배경 |
