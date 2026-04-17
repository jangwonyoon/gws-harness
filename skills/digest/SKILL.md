---
name: gwh-digest
description: >
  주간 다이제스트 — 이번 주 메일 송수신 + 캘린더 참석 + (optional) Jira 완료 티켓을 요약 리포트로 생성.
  Use when "이번 주 정리", "주간 요약", "weekly digest",
  "/gwh:digest", "이번 주 뭐 했지", "주간 리포트", "회고 자료".
version: 1.0.0
category: automation
tags: [google-workspace, gmail, calendar, jira, digest, retrospective]
triggers:
  - "이번 주 정리"
  - "주간 요약"
  - "weekly digest"
  - "주간 리포트"
  - "이번 주 뭐 했지"
dependencies: []
---

# Weekly Digest

이번 주 메일 송수신 + 캘린더 참석 + (optional) Jira 완료 티켓 요약.

---

## Step 0: 사전 확인

### 0.1 gws CLI + 인증 확인
미설치/미인증 시 안내 후 중단.

### 0.2 확장 기능 감지
Jira MCP 연결 여부 확인. 미연결 시 코어(메일+캘린더)만 실행.

### 0.3 기간 결정

기본: 이번 주 (월~오늘).
주 초(월/화) 실행 시 → AskUserQuestion: "데이터가 적습니다. 지난주 데이터를 포함할까요?"

---

## Step 1: 📬 메일 통계

```bash
gws gmail +triage --query "after:{YYYY/MM/DD} before:{YYYY/MM/DD}" --max 200 --format json
```

집계:
- **수신:** 총 N통 (카테고리별 분류)
- **송신:** `gws gmail +triage --query "from:me after:..." --format json`으로 별도 조회
- **송신 분석:** 수신자별 그룹핑 (어떤 팀/사람과 가장 많이 소통했는지)

---

## Step 2: 📅 캘린더 통계

```bash
gws calendar +agenda --days 7 --format json
```

> 또는 gws 내장 헬퍼 활용:
> ```bash
> gws workflow +weekly-digest --format json
> ```

집계:
- **미팅 수:** N회
- **미팅 시간:** 총 Mh
- **미팅 비중:** 전체 업무시간(40h) 대비 M%
- **참석자 빈도:** 가장 자주 만난 사람 Top 3
- **전주 대비:** 미팅 ±N회, 시간 ±Mh

---

## Step 3: 🏃 Jira 통계 (optional)

Jira MCP 연결 시:

1. `mcp__atlassian__jira_search_issues` — `assignee = currentUser() AND status changed to Done DURING (startOfWeek(), now())`
2. 집계:
   - **완료 티켓:** N건 (SP 합계 M)
   - **티켓 유형별:** Bug N / Story N / Task N
   - **전주 대비:** ±N건, ±M SP

---

## Step 4: 💡 인사이트 도출

자동 인사이트:
- **코딩 가용시간:** 40h - 미팅시간 - (메일 추정시간: 송신 1통당 ~5분)
- **미팅 과부하 경고:** 미팅 비중 > 30% 시 경고
- **소통 편중 경고:** 특정 사람과 메일 50%+ 시 알림
- **생산성 추정:** 완료 SP / 코딩 가용시간 = SP/h 효율

> **왜 30% 임계인가**: Paul Graham의 Maker's Schedule vs Manager's Schedule — 미팅 30% 초과 시 개발 몰입 시간이 1시간 이하로 파편화되어 복잡한 작업 불가. 업계 경험칙 근거.

> **왜 주간 단위인가**: 일간은 노이즈(회의 몰린 날), 월간은 피드백 루프 지연. 스프린트 주기(1-2주)와 일치해 회고/계획에 즉시 활용 가능.

---

## Step 5: 결과 출력

```markdown
# 📊 Weekly Digest — {YYYY}-W{WW} ({시작일}~{종료일})

## 📬 메일
- 수신: {N}통 | 송신: {M}통
- 주요 소통: {팀/사람} ({N}통), {팀/사람} ({M}통)

## 📅 미팅
- {N}회 ({M}h) — 전주 대비 {±변화}
- 미팅 비중: {M}% {⚠️ 과부하 경고 if > 30%}
- 빈도 Top 3: {사람1}, {사람2}, {사람3}

## 🏃 스프린트 (Jira 연결 시)
- 완료: {N}건 (SP {M})
- Bug {N} / Story {N} / Task {N}
- 전주 대비: {±변화}

## 💡 인사이트
- 코딩 가용시간: {N}h (전체 {M}%)
- SP 효율: {N} SP/h
- {추가 인사이트}

---
```

---

## Step 6: 산출물 저장

결과를 `~/.gwh/digest-{YYYY}-W{WW}.md`에 저장.
