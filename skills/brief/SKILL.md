---
name: gwh-brief
description: >
  아침 브리핑 — 오늘의 메일 요약 + 캘린더 일정 + (optional) Jira 스프린트 상태를 통합 브리핑.
  Use when "브리핑", "오늘 뭐해", "아침 요약", "morning brief",
  "/gwh:brief", "오늘 현황", "today summary".
version: 1.0.0
category: automation
tags: [google-workspace, gmail, calendar, jira, briefing, productivity]
triggers:
  - "브리핑"
  - "오늘 뭐해"
  - "아침 요약"
  - "morning brief"
  - "오늘 현황"
dependencies: []
---

# Morning Brief

메일 요약 + 오늘 캘린더 일정 + (optional) Jira 스프린트 상태를 하나의 브리핑으로 통합.

---

## Step 0: 사전 확인

### 0.1 gws CLI + 인증 확인

triage 스킬과 동일. `which gws` → `gws auth status`. 미설치/미인증 시 안내 후 중단.

### 0.2 확장 기능 감지

| 확장 | 감지 | 결과 |
|------|------|------|
| Jira | `mcp__atlassian__jira_get_user_profile` 호출 | 성공 → 스프린트 섹션 활성 |

감지 실패해도 코어 기능(메일+캘린더)은 정상 동작.

> **왜 런타임 감지인가**: Jira MCP가 없는 외부 사용자도 이 스킬을 쓸 수 있어야 한다(R6). 설정 파일 대신 런타임에 감지하면 사용자 설정 복잡도 0, 의존성 부재 시 graceful degradation.

### 0.3 triage 캐시 확인

```bash
cat ~/.gwh/triage-$(date +%Y-%m-%d).md 2>/dev/null
```

- 존재 → Gmail API 재호출 없이 캐시 사용
- 없음 → Step 1에서 Gmail 조회

> **왜 캐시 재사용이 필수인가**: brief는 triage와 동일한 Gmail 쿼리를 재실행할 가능성이 크다. 실측 358배 속도 차(4ms vs 1,432ms) + API 쿼터 절감. 하루 단위 캐시는 triage 결과가 하루 중 크게 변하지 않는다는 가정에 근거.

---

## Step 1: 📬 메일 섹션

### 캐시가 있는 경우
`~/.gwh/triage-{date}.md` 파일을 읽어 🔴/🟡 항목만 요약.

### 캐시가 없는 경우
```bash
gws gmail +triage --max 30 --query 'is:unread' --format json
```

> ⚠️ `resultSizeEstimate`는 추정치이며 201에서 포화된다. 출력 시 "약 {N}+통"으로 표기.

간소화된 분류 (triage 스킬보다 가볍게):
- 🔴 긴급 + 🟡 액션 필요 항목만 나열
- 총 미읽음 수 + FYI/무시 가능 건수는 숫자만

---

## Step 2: 📅 캘린더 섹션

```bash
gws calendar +agenda --today --format json
```

JSON 결과에서:
- 시간순 일정 나열 (시작-종료, 제목, 참석자 수)
- 미팅 사이 빈 시간대 식별 → "집중 작업 가능" 표시
- 오늘 일정이 0개면 → "오늘은 미팅 없음. 집중 개발 가능 🎯"

---

## Step 3: 🏃 스프린트 섹션 (optional)

Jira MCP가 연결된 경우에만 실행.

1. `mcp__atlassian__jira_get_agile_boards` → 활성 보드 확인
2. `mcp__atlassian__jira_get_sprints_from_board` → 현재 스프린트
3. `mcp__atlassian__jira_get_sprint_issues` → 티켓 목록

요약:
- 스프린트 D-{N}: {완료}/{전체} 완료
- 블로커: {N}건
- 나에게 할당된 In Progress: {N}건

---

## Step 4: 결과 출력

```markdown
# ☀️ Morning Brief — {YYYY-MM-DD} ({요일})

## 📬 메일 ({N}통 미읽음)
- 🔴 긴급 {N}건: {요약}
- 🟡 액션 {N}건: {요약}
- 나머지: FYI {N} / 무시 {N}

## 📅 오늘 일정
| 시간 | 제목 | 참석자 |
|------|------|--------|
| 10:00-11:00 | 스프린트 플래닝 | 6명 |
| 14:00-14:30 | 1:1 | 2명 |

**빈 시간:** 09-10시, 11-14시, 14:30-18시 (총 {N}h)

## 🏃 스프린트 (D-{N})
{완료}/{전체} 완료 | 블로커 {N}건
내 In Progress: {티켓 목록}

---
💡 **오늘 추천:** {빈 시간 기반 작업 제안}
```

Jira 미연결 시 🏃 섹션 생략, 💡 추천에서도 Jira 관련 제안 제외.

---

## Step 5: 산출물 저장

결과를 `~/.gwh/brief-{YYYY-MM-DD}.md`에 저장.
