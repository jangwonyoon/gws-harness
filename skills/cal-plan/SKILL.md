---
name: gwh-cal-plan
description: >
  Jira 스프린트 티켓을 시간 단위로 분해하여 Google Calendar에 타임블록으로 배치한다.
  Use when "일정 짜줘", "타임블록", "캘린더 계획", "cal plan",
  "/gwh:cal-plan", "이번 주 계획", "시간 배분".
version: 1.0.0
category: automation
tags: [google-workspace, calendar, jira, timeblock, planning, productivity]
triggers:
  - "일정 짜줘"
  - "타임블록"
  - "캘린더 계획"
  - "cal plan"
  - "이번 주 계획"
dependencies: []
---

# Calendar Planner

Jira 스프린트 티켓(또는 수동 입력 태스크)을 시간 단위로 분해하여 Google Calendar 빈 시간에 타임블록으로 배치.

---

## Step 0: 사전 확인

### 0.1 gws CLI + 인증 확인
triage 스킬과 동일. 미설치/미인증 시 안내 후 중단.

### 0.2 태스크 소스 결정

| 소스 | 감지 방법 | 동작 |
|------|----------|------|
| Jira MCP | `mcp__atlassian__jira_get_user_profile` 성공 | 스프린트 티켓 자동 조회 |
| 수동 입력 | Jira 미연결 또는 사용자 선택 | AskUserQuestion으로 태스크 목록 입력 |

---

## Step 1: 태스크 수집

### Jira 경로

1. `mcp__atlassian__jira_get_agile_boards` → 활성 보드
2. `mcp__atlassian__jira_get_sprints_from_board` → 현재 스프린트
3. `mcp__atlassian__jira_get_sprint_issues` → 나에게 할당된 To Do / In Progress 티켓

각 티켓에서 추출:
- 키 (DPPM-91)
- 제목
- Story Point (SP)
- 우선순위 (Highest > High > Medium > Low)

### 수동 경로

AskUserQuestion으로 태스크 목록 입력 요청:

```
이번 주 할 작업을 알려주세요.
예: "hex 감사 (3h), 버그픽스 (1.5h), 컴포넌트 마이그레이션 (5h)"
```

---

## Step 2: 시간 산출

SP → 시간 변환 기본 테이블:

| SP | 예상 시간 | 비고 |
|----|----------|------|
| 1 | 1.5h | 간단한 수정 |
| 2 | 3h | 표준 태스크 |
| 3 | 5h | 중간 복잡도 |
| 5 | 8h (1일) | 하루 집중 |
| 8 | 13h (1.5일) | 대형 — 분할 추천 |

SP 없는 티켓은 Medium 우선순위에서 3h로 기본 추정.
사용자에게 산출 결과를 보여주고 조정 기회 제공.

---

## Step 3: 빈 시간 감지

```bash
gws calendar +agenda --week --format json
```

JSON 결과에서:
- 기존 일정(미팅, 이벤트)을 파싱
- 업무 시간대(기본 09:00-18:00) 내 빈 슬롯 계산
- 30분 미만 빈 시간은 무시 (의미 있는 작업 불가)
- 점심시간(12:00-13:00) 자동 제외

결과: 요일별 가용 시간 목록

---

## Step 4: 타임블록 배치 제안

배치 규칙:
1. **우선순위 높은 티켓 먼저** (Highest → High → Medium → Low)
2. **큰 블록 먼저 배치** (연속 3h+ 슬롯에 큰 태스크)
3. **같은 태스크는 연속 배치** (컨텍스트 스위칭 최소화)
4. **SP 8+ 태스크는 반드시 분할** (하루 최대 4h 연속)

출력:

```markdown
## 📅 타임블록 배치 제안

### 월요일 (4/17)
| 시간 | 태스크 | 예상 |
|------|--------|------|
| 09:00-12:00 | DPPM-91 hex 감사 (1/2) | 3h |
| 14:00-17:00 | DPPM-91 hex 감사 (2/2) | 3h |

### 화요일 (4/18)
| 시간 | 태스크 | 예상 |
|------|--------|------|
| 09:00-10:30 | DPPM-95 버그픽스 | 1.5h |
| 10:30-12:00 | DPPM-88 마이그레이션 (1/3) | 1.5h |

⚠️ **경고:** 목/금은 미팅 4개로 가용시간 3h만 남음
→ 우선순위 조정 또는 태스크 이월 필요
```

---

## Step 5: 사용자 확인

AskUserQuestion으로 배치 승인:

- **승인** → Step 6으로
- **수정** → 변경사항 반영 후 재제안
- **취소** → 산출물만 저장, 캘린더 미반영

---

## Step 6: 캘린더 이벤트 생성

승인된 블록만 생성:

```bash
gws calendar +insert \
  --summary "[DPPM-91] hex 감사 (1/2)" \
  --start "2026-04-17T09:00:00" \
  --end "2026-04-17T12:00:00" \
  --description "Story Point: 3 | Priority: High"
```

각 이벤트 생성 후 성공/실패 보고.

---

## Step 7: 산출물 저장

결과를 `~/.gwh/cal-plan-{YYYY-MM-DD}.md`에 저장.
- 배치된 블록 목록
- 미배치 태스크 (시간 부족)
- 가용 시간 통계
