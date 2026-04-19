# gws-harness — 프로젝트 규약

Google Workspace CLI (gws) 자동화 플러그인. 작업 시 아래 규약 준수.

## Google Workspace CLI (gws) 사용 규칙

### stdout 오염 처리
- `gws` 명령은 stdout에 **"Using keyring" prefix**를 붙일 수 있음 → jq/json 파싱 전에 반드시 제거
- 래퍼 헬퍼 사용: `bin/gws-json <args>` (필터링된 JSON stdout만 반환)
- 직접 호출 시 패턴: `gws ... 2>/dev/null | sed '/^Using keyring/d' | jq .`

### 멀티 계정 실행
- 계정 격리는 **환경변수로만** (XDG 무시):
  - `GOOGLE_WORKSPACE_CLI_CONFIG_DIR=~/.gws/accounts/<alias>`
  - `KEYRING_BACKEND=file`
- `gws auth status` 호출 시 `--format json` **미지원** — stdout 파싱 필요
- OAuth 자격증명은 **Desktop app** 타입만 유효 (Web 타입은 실패)

### 단순 조회는 plain gws 우선
- 캘린더/메일 단순 조회는 raw `gws` CLI가 빠름. 하네스 스킬(`/gwh:*`) 호출은 **복합 워크플로우**에만

## 스킬 추가 (요약)

상세 절차는 **AGENTS.md** 참조. 최소 원칙만 여기 고정.

- 디렉토리: `skills/{kebab-case}/SKILL.md` (frontmatter `name: gwh-{kebab}` 필수)
- frontmatter `description`은 **"Use when ..." 트리거 한 줄** (워크플로우 단계 요약 금지 — Claude가 본문을 skip)
- 쉘 래퍼/헬퍼는 `bin/` 하위 (예: `bin/gws-json`)
- OAuth 자격증명 경로는 **env var로만** (`GOOGLE_WORKSPACE_CLI_CONFIG_DIR`) — 스킬/코드에 하드코딩 금지
- 쓰기 작업(메일 전송, 캘린더 생성 등)은 반드시 `AskUserQuestion` 승인 게이트 통과

## 스코프 가드

- 이데이션/eval 산출물은 `docs/brainstorms/`, `docs/plans/` 하위에만 커밋
- 실행 가능 코드(skills, bin, .github)는 검증 후 커밋
- 실제 CLI 동작 검증 전에 코드 스캐폴딩 금지 (역사적으로 feasibility 리뷰에서 P0 블로커 반복 발견)

## 커밋 & PR

- **Conventional Commits**: `feat:` / `fix:` / `refactor:` / `docs:` / `chore:` (breaking은 `feat!:`)
- main 직접 푸시 금지 — `feat/{name}` 브랜치 → PR → merge (CI `validate.yml` 통과 전제)
- 버전 bump는 `.claude-plugin/plugin.json` + `marketplace.json` 동기화

## CI / 마크다운 린트

- `.markdownlint.json` 설정 기반
- Edit/Write 후 자동 수정 훅 활성화 (`.claude/settings.json` PostToolUse)
- CI 실패 반복(3회+ 패턴) 방지용: push 전 로컬에서 `markdownlint-cli2 "**/*.md"` 실행

## 검증 우선 (Self-Validating Pipeline)

- 새 스킬/기능은 `VALIDATION_SPEC.md`에 실행 가능한 테스트 케이스 먼저 정의
- 멀티 계정 시나리오, stdout prefix 파싱, OAuth Desktop flow, 빈/변형 결과 fallback 4가지 최소 커버
- 테스트 녹색 전에 머지 금지

### 수동 드라이런

```bash
# 단일 스킬 실행 (플러그인 재설치 없이 SKILL.md 지시를 Bash로 시뮬)
claude -p "/gwh:triage"
claude -p "/gwh:brief"

# 인증 상태 확인 (스킬 Step 0과 동일)
gws auth status
GOOGLE_WORKSPACE_CLI_CONFIG_DIR=~/.gws/accounts/personal KEYRING_BACKEND=file gws auth status
```
