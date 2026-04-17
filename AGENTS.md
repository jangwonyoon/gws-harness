# AGENTS.md — gws-harness 기여 가이드

AI 에이전트 또는 인간 기여자가 gws-harness에 새 스킬을 추가하거나 기존 스킬을 수정할 때 따라야 할 패턴과 원칙.

> **왜 이 문서가 필요한가**: SKILL.md는 "엔드 유저가 사용할 때의 동작"을 기술한다. 기여자는 그 뒤의 "왜 이렇게 설계했는지 + 새 스킬은 어떻게 추가하는지"를 알아야 일관성을 유지할 수 있다.

---

## 프로젝트 성격

gws-harness는 **Claude Code 플러그인**으로, Google Workspace CLI(`gws`)를 래핑하는 스킬 모음을 배포한다.

- **자율 실행 환경이 아님** — 모든 쓰기 작업은 `AskUserQuestion`으로 사용자 승인 후 실행
- **코어 + 확장 구조** — Gmail/Calendar는 `gws` CLI만으로 동작하는 코어, Jira/Notion은 MCP가 있을 때만 활성화되는 optional 확장
- **산출물 파일 기반 체이닝** — 스킬 간 데이터 공유는 `~/.gwh/` 마크다운 파일로

---

## 새 스킬 추가 절차

### 1. 디렉토리 + 파일
```
skills/{name}/SKILL.md
```

> **왜 디렉토리 구조인가**: 향후 `references/` 파일(긴 예제, 에러 코드 표 등)을 추가할 때 Progressive Disclosure를 유지하기 위함. SKILL.md는 <200줄 유지, 상세는 `references/` 이동.

### 2. Frontmatter 필수 필드
```yaml
---
name: gwh-{short-name}       # /gwh:* 네임스페이스 준수
description: >
  {1-2 문장 요약}. Use when "{한국어 트리거}", "{영어 트리거}",
  "/gwh:{short-name}".
version: 1.0.0
category: automation
tags: [google-workspace, {domain}, ...]
triggers:
  - "{한국어 트리거}"
  - "{영어 트리거}"
dependencies: []
---
```

> **왜 triggers를 한/영 모두 넣는가**: 장원님은 한국어 우선이지만 외부 기여자/공유 시 영어 사용자도 고려. 트리거 충돌 방지를 위해 `gwh-` 접두사 필수.

### 3. Step 0 사전 확인 블록 필수

모든 스킬은 `gws` CLI 설치 + 인증 상태를 먼저 체크한다:

```markdown
## Step 0: 사전 확인

### 0.1 gws CLI 확인
```bash
which gws
```
미설치 시 → 안내 후 중단.

### 0.2 인증 확인
```bash
gws auth status
```
미인증 시 → `gws auth login` 안내 후 중단.
```

> **왜 매번 체크하는가**: gws가 없거나 만료된 상태에서 스킬이 실행되면 의미 없는 API 에러가 쌓여 사용자 컨텍스트를 오염시킨다. 0.001초 비용으로 이 노이즈를 차단.

### 4. 산출물 저장 규칙

- 경로: `~/.gwh/{skill-name}-{date|week}.md`
- 형식: 구조화된 마크다운 (다른 스킬이 grep/read로 소비 가능)
- 네이밍: `triage-{YYYY-MM-DD}.md`, `digest-{YYYY}-W{WW}.md`

> **왜 파일로 저장하나**: (1) 캐시 재사용으로 API 비용 감소 — triage 캐시 재사용 시 358배 빠름(1.4s → 4ms), (2) 스킬 간 의존성을 파일 경로로 느슨하게 연결, (3) 감사/디버깅 가능성.

### 5. 쓰기 작업 승인 게이트

Gmail 쓰기(메일 보내기), Calendar 이벤트 생성, Jira 이슈 생성 등 **외부 시스템에 부작용을 일으키는 모든 작업**은 반드시:

1. 제안 내용을 사용자에게 보여줌
2. `AskUserQuestion`으로 승인 받음
3. 승인된 경우에만 실행

> **왜 승인 게이트인가**: AI가 잘못된 메일을 보내거나 빈 캘린더 이벤트 16개를 만드는 사고는 복구 비용이 비대칭적으로 크다. 승인 한 번 비용 < 실수 복구 비용.

---

## 코어 vs 확장 원칙

### 코어 (gws CLI만 있으면 동작해야 함)
- Gmail 읽기/트리아지
- Calendar 읽기
- (승인 후) Calendar 이벤트 생성

### 확장 (optional MCP)
- Jira 스프린트 조회 → Atlassian MCP
- Notion 페이지 생성 → Notion MCP

### 감지 패턴
```markdown
### 확장 기능 감지
| 확장 | 감지 | 활성화 시 |
|------|------|----------|
| Jira | `mcp__atlassian__jira_get_user_profile` 호출 시도 | 스프린트 섹션 |
| Notion | `mcp__notion-personal__API-get-self` 호출 시도 | 페이지 생성 |
```

> **왜 코어/확장 분리인가**: R6 요구사항(외부 공유용) — 회사 MCP를 안 가진 외부 사용자도 Gmail+Calendar만으로 즉시 가치를 얻어야 한다. MCP를 필수로 요구하면 진입 장벽이 커서 채택이 망한다.

---

## 네이밍 & 포맷 컨벤션

| 요소 | 규칙 |
|------|------|
| 스킬 디렉토리 | `skills/{kebab-case}/` |
| 스킬 name | `gwh-{kebab-case}` |
| 명령 호출 | `/gwh:{short-name}` |
| 산출물 파일 | `{skill}-{date}.md`, `{skill}-{YYYY}-W{WW}.md` |
| 이모지 섹션 | 📬 메일 / 📅 캘린더 / 🏃 스프린트 / 💡 인사이트 |
| 긴급도 | 🔴 긴급 / 🟡 액션 / 🟢 FYI / ⚪ 무시 |

---

## 테스트 & 검증

### 로컬 검증
```bash
# Frontmatter 필수 필드 확인
for f in skills/*/SKILL.md; do
  yq -f extract '.name, .description, .version, .triggers' "$f" > /dev/null || echo "BROKEN: $f"
done
```

### 실제 동작 확인
1. `gws auth status`로 인증 확인
2. 스킬을 수동 실행 (아직 플러그인 재설치 전이면 SKILL.md 지시사항을 따라 Bash로 시뮬레이션)
3. 산출물 파일이 예상 형식으로 생성되는지 확인

> **왜 수동 실행부터인가**: 스킬은 LLM이 문서를 읽고 따르는 방식이라, 통상의 유닛테스트가 없다. 실제 호출해보고 결과를 검수하는 게 가장 확실.

---

## 커밋 & 배포

- 레포: `jangwonyoon/gws-harness` (main 브랜치 = 배포 버전)
- 버전 관리: `.claude-plugin/plugin.json`의 `version` 필드
- 배포 채널: `/plugin marketplace add jangwonyoon/gws-harness`

> **왜 version 필드가 중요한가**: 플러그인은 마켓플레이스를 통해 자동 업데이트되므로 사용자의 `/gwh:*` 동작이 예기치 않게 바뀔 수 있다. Breaking change는 major bump.

---

## 참고

- 원본 아이디어이션: [Notion 페이지](https://www.notion.so/344de71aaa9f80798fcdf680991d19c5)
- 하네스 감사 리포트: `harness-report-{YYYY-MM-DD}.md`
- gws CLI 공식: https://github.com/googleworkspace/cli
