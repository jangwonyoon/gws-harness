# khc-gws-harness

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

```bash
claude plugin install khc-gws-harness
```

## 스킬 목록

| 스킬 | 설명 | 트리거 |
|------|------|--------|
| `/gwh:triage` | Gmail 미읽음 메일 분류 + 액션 추출 | "메일 정리", "메일 트리아지" |
| `/gwh:brief` | 아침 브리핑 (메일+캘린더+Jira) | "브리핑", "오늘 뭐해" |
| `/gwh:cal-plan` | Jira 티켓 → 캘린더 타임블록 | "일정 짜줘", "타임블록" |
| `/gwh:mail-to-ticket` | 메일 → Jira/Notion 티켓 | "메일 티켓으로" |
| `/gwh:digest` | 주간 활동 다이제스트 | "주간 요약", "이번 주 정리" |

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

산출물은 스킬 간 공유됩니다 (예: brief가 triage 결과를 재사용).
