---
name: gwh-mail-to-ticket
description: >
  특정 메일에서 요구사항/버그를 추출하여 Jira 이슈 또는 Notion 페이지를 생성한다.
  Use when "이 메일 티켓으로", "메일에서 이슈 만들어", "mail to ticket",
  "/gwh:mail-to-ticket", "메일 티켓화", "이거 이슈로".
version: 1.0.0
category: automation
tags: [google-workspace, gmail, jira, notion, ticket, automation]
triggers:
  - "메일 티켓으로"
  - "메일에서 이슈 만들어"
  - "mail to ticket"
  - "메일 티켓화"
dependencies: []
---

# Mail to Ticket

특정 메일에서 요구사항/버그를 구조화하여 Jira 이슈 또는 Notion 페이지를 생성.

---

## Step 0: 사전 확인

### 0.1 gws CLI + 인증 확인
미설치/미인증 시 안내 후 중단.

### 0.2 대상 메일 식별

입력 방식 (우선순위):
1. **triage 결과에서 선택**: `~/.gwh/triage-{date}.md` 존재 시 → 🔴/🟡 목록에서 AskUserQuestion으로 선택
2. **메일 검색**: 사용자가 발신자/제목 키워드 제공 → `gws gmail +triage --query "from:xxx subject:xxx" --max 5`
3. **메일 ID 직접 지정**: 사용자가 ID 제공

### 0.3 대상 시스템 감지

| 시스템 | 감지 | 동작 |
|--------|------|------|
| Jira MCP | `mcp__atlassian__jira_get_user_profile` 성공 | Jira 이슈 생성 |
| Notion MCP | `mcp__notion-personal__API-get-self` 성공 | Notion 페이지 생성 |
| 둘 다 연결 | 양쪽 성공 | AskUserQuestion으로 대상 선택 |
| 둘 다 미연결 | 양쪽 실패 | 마크다운 출력 (수동 복사) |

---

## Step 1: 메일 본문 조회

```bash
gws gmail +read --id {MESSAGE_ID} --format json --headers
```

추출 대상:
- From, To, CC, Date
- Subject
- Body (text)
- 첨부파일 목록

---

## Step 2: 메일 분석 + 구조화

메일 유형을 자동 감지:

| 유형 | 감지 신호 | 추출 구조 |
|------|----------|----------|
| **버그 리포트** | "버그", "에러", "오류", "재현", "장애" | 제목, 설명, 재현 단계, 심각도, 환경 |
| **기능 요청** | "기능", "요청", "추가", "개선" | Epic + Story 분해 (제목, 설명, SP 추정) |
| **작업 요청** | "해주세요", "부탁", "확인", "처리" | 제목, 설명, 우선순위, 마감일 |
| **기타** | 위 패턴 없음 | 제목 + 원문 그대로 description |

---

## Step 3: 사용자 확인

구조화된 결과를 AskUserQuestion으로 보여주고 승인:

> **왜 자동 생성하지 않나**: LLM은 맥락 없이 "일단 티켓화"하는 경향이 있어 오버엔지니어링 위험 크다(예: 단순 린트 이슈도 Epic으로). 사용자가 "PR 직접 수정이 낫다" 같은 판단을 개입해야 하며, 실제 검증에서 CodeRabbit Autofix 존재 시 티켓 생성이 불필요함이 확인됨.

```markdown
📋 티켓 미리보기

**유형:** 버그 리포트
**제목:** [Bug] 로그인 후 리다이렉트 실패 (Chrome 126)
**심각도:** High
**설명:**
- 로그인 성공 후 대시보드로 이동하지 않음
- Chrome 126에서만 재현
**재현 단계:**
1. 로그인 페이지 접속
2. 계정 입력 후 로그인
3. 빈 화면 표시

생성할까요? [Jira / Notion / 마크다운 출력 / 수정 / 취소]
```

---

## Step 4: 티켓 생성

### Jira 경로

MCP로 이슈 생성:
```
mcp__atlassian__jira_create_issue
  project: {프로젝트 키}
  issueType: Bug / Story / Task
  summary: {제목}
```

> description에 포맷팅이 필요하면 curl + ADF JSON으로 덮어쓰기 (jira-analyzer 패턴):
> MCP로 생성 → curl로 ADF description 업데이트

원본 메일 링크를 코멘트로 추가.

### Notion 경로

```
mcp__notion-personal__API-post-page
  parent: {페이지/DB ID}
  properties: {제목}
  children: {구조화된 본문 블록}
```

### 마크다운 폴백

MCP 미연결 시 구조화된 결과를 마크다운으로 출력:
```markdown
## 📋 생성할 티켓

**제목:** {제목}
**유형:** {유형}
**설명:** {설명}

> 이 내용을 Jira/Notion에 수동으로 복사하세요.
```

---

## Step 5: 산출물 저장

결과를 `~/.gwh/ticket-{date}-{subject-slug}.md`에 저장:
- 원본 메일 요약
- 생성된 티켓 정보 (키, URL)
- 구조화된 내용
