# gws-harness

Google Workspace 하네스 — `gws` CLI 기반 Claude Code 플러그인

Gmail 트리아지, 캘린더 타임블록, 아침 브리핑, 메일→티켓 변환, 주간 다이제스트를 대화형으로 제공합니다.

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
