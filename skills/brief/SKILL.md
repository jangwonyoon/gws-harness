---
name: gwh-brief
description: >
  아침 브리핑 — 오늘의 메일 요약 + 캘린더 일정 + (optional) Jira 스프린트 상태를 통합.
  2계정(work/personal) 병렬 fetch 지원. triage merged 캐시가 있으면 재사용(실측 358배),
  없으면 Gmail/Calendar 2-branch 병렬 fetch. merged 파일 frontmatter의
  accounts_success를 현재 ACCOUNTS와 비교하여 drift 배너를 띄운다.
  Use when "브리핑", "오늘 뭐해", "아침 요약", "morning brief",
  "/gwh:brief", "오늘 현황", "today summary".
version: 1.1.0
category: automation
tags: [google-workspace, gmail, calendar, jira, briefing, productivity, multi-account]
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

### 0.1 gws CLI 확인

```bash
which gws
```
미설치 시 → 안내 후 중단.

### 0.2 config.yml + ACCOUNTS 확정

`/gwh:triage` Step 0.4와 동일 로직. 등록된 모든 계정에 대해 env-inject `gws auth status`로 enrollment 검증 후 `ACCOUNTS` 배열 확정. 인증된 계정이 0개면 `/gwh:credential-init` 안내 후 중단.

> Unit B Step 0.4 코드 복사 (D7 — 3회+ 복사 시 references/multi-account.md로 extract 검토).

### 0.3 확장 기능 감지

| 확장 | 감지 | 결과 |
|------|------|------|
| Jira | `mcp__atlassian__jira_get_user_profile` 호출 | 성공 → 스프린트 섹션 활성 |

감지 실패해도 코어 기능(메일+캘린더)은 정상 동작.

### 0.4 triage 캐시 확인 + drift 감지

```bash
CACHE="$HOME/.gwh/triage-$(date +%Y-%m-%d).md"
USE_CACHE=false
DRIFT=false

if [[ -f "$CACHE" ]]; then
  # merged frontmatter의 accounts_success 추출
  CACHED_ACCS=$(awk '/^---$/{n++; next} n==1 && /^accounts_success:/{gsub(/.*\[|\].*/,""); print; exit}' "$CACHE")
  # 현재 ACCOUNTS 문자열과 비교 (정렬 후)
  CURRENT=$(printf '%s\n' "${ACCOUNTS[@]}" | sort | paste -sd, -)
  CACHED_SORTED=$(echo "$CACHED_ACCS" | tr ',' '\n' | sed 's/^ *//;s/ *$//' | sort | paste -sd, -)

  if [[ "$CURRENT" == "$CACHED_SORTED" ]]; then
    USE_CACHE=true
  else
    DRIFT=true
    echo "⚠️ triage 캐시는 [$CACHED_SORTED] 계정으로 생성되었습니다."
    echo "   현재 ACCOUNTS: [$CURRENT]"
    echo "   /gwh:triage를 재실행하여 캐시를 갱신하는 것을 권장합니다."
    # drift는 경고만, brief는 캐시 사용 계속 (사용자 선택)
  fi
fi
```

- `USE_CACHE=true` → Step 1에서 캐시 경로 (4ms±δ)
- 캐시 없음 또는 drift + 사용자 재실행 선택 → Step 1 fallback에서 2계정 병렬 fetch

> **왜 drift만 경고하고 캐시는 쓰는가**: brief는 빠른 확인 용도 — 매번 재계산하면 358배 성능 이점이 무력화된다. 사용자에게 상태 가시성만 제공하고 재실행 판단은 맡긴다.

---

## Step 1: 📬 메일 섹션

### 1.1 캐시 히트 경로

`$CACHE` 파일을 읽어 🔴/🟡 항목만 요약. 2계정 merged 포맷의 라벨 prefix (`[🏢 work]` / `[👤 personal]`)가 있으면 그대로 보존하여 표시.

### 1.2 캐시 미스 / drift 재실행 경로

Unit B Step 1의 2-branch 병렬 fetch 템플릿 복사:

```bash
trap 'rm -f /tmp/gwh-brief-*-$$.{json,err,exit}' EXIT

for a in "${ACCOUNTS[@]}"; do
  (
    GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$HOME/.config/gws-$a" \
    GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
    gws gmail +triage --max 30 --query 'is:unread' --format json \
      > "/tmp/gwh-brief-$a-$$.json" 2>"/tmp/gwh-brief-$a-$$.err"
    echo "$a:$?" > "/tmp/gwh-brief-$a-$$.exit"
  ) &
done
wait
```

캐시 경로보다 가볍게: `--max 30` (브리핑은 Top N만 필요).

> ⚠️ `resultSizeEstimate`는 추정치이며 201에서 포화된다. 출력 시 "약 {N}+통"으로 표기.

---

## Step 2: 📅 캘린더 섹션 (2-branch 병렬)

```bash
trap 'rm -f /tmp/gwh-brief-cal-*-$$.{json,err,exit}' EXIT

for a in "${ACCOUNTS[@]}"; do
  (
    GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$HOME/.config/gws-$a" \
    GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
    gws calendar +agenda --today --format json \
      > "/tmp/gwh-brief-cal-$a-$$.json" 2>"/tmp/gwh-brief-cal-$a-$$.err"
    echo "$a:$?" > "/tmp/gwh-brief-cal-$a-$$.exit"
  ) &
done
wait
```

각 계정 캘린더 일정을 시간순 병합:
- 시간순 일정 나열 (시작-종료, 제목, 참석자 수, **계정 라벨** — |ACCOUNTS| >= 2일 때)
- 미팅 사이 빈 시간대 = 양 계정 union의 free slots → "집중 작업 가능"
- 양쪽 모두 0개면 → "오늘은 미팅 없음. 집중 개발 가능 🎯"

---

## Step 3: 🏃 스프린트 섹션 (optional, 계정 중립)

Jira MCP가 연결된 경우에만 실행. **계정과 무관** (Jira는 단일 쿼리).

1. `mcp__atlassian__jira_get_agile_boards` → 활성 보드
2. `mcp__atlassian__jira_get_sprints_from_board` → 현재 스프린트
3. `mcp__atlassian__jira_get_sprint_issues` → 티켓 목록

요약:
- 스프린트 D-{N}: {완료}/{전체} 완료
- 블로커: {N}건
- 나에게 할당된 In Progress: {N}건

---

## Step 4: 결과 출력

### 단일 계정 (|ACCOUNTS| == 1)

기존 포맷 유지 (라벨 없음):

```markdown
# ☀️ Morning Brief — {YYYY-MM-DD} ({요일})

## 📬 메일 ({N}통 미읽음)
- 🔴 긴급 {N}건: {요약}
- 🟡 액션 {N}건: {요약}

## 📅 오늘 일정
| 시간 | 제목 | 참석자 |
|------|------|--------|
| 10:00-11:00 | 스프린트 플래닝 | 6명 |

**빈 시간:** 09-10시, 11-14시 (총 {N}h)
```

### 2계정 (|ACCOUNTS| >= 2)

```markdown
---
accounts_success: [work, personal]
accounts_failed: []
generated_at: 2026-04-17T09:30:00+09:00
source: cache | fresh
---

# ☀️ Morning Brief — {YYYY-MM-DD} ({요일}) (통합)

## 📬 메일
- 🔴 긴급 {N}건
  - [🏢 work] {요약}
  - [👤 personal] {요약}
- 🟡 액션 {N}건

## 📅 오늘 일정 (work + personal union)
| 시간 | 계정 | 제목 | 참석자 |
|------|------|------|--------|
| 10:00-11:00 | 🏢 work | 스프린트 플래닝 | 6명 |
| 19:00-20:00 | 👤 personal | 운동 | - |

**빈 시간:** 11:00-18:00, 20:00 이후

## 🏃 스프린트 (D-{N}) — 계정 중립
...
```

Jira 미연결 시 🏃 섹션 생략.

---

## Step 5: 산출물 저장 (dual write + chmod inline)

Unit B Step 5 패턴 복사:

```bash
mkdir -p "$HOME/.gwh"
chmod 0700 "$HOME/.gwh"

DATE=$(date +%Y-%m-%d)

for a in "${SUCCESS[@]}"; do
  PER="$HOME/.gwh/brief-$a-$DATE.md"
  write_per_account_brief "$a" > "$PER"
  chmod 0600 "$PER"
done

MERGED="$HOME/.gwh/brief-$DATE.md"
if [[ ${#SUCCESS[@]} -eq 1 ]]; then
  cp "$HOME/.gwh/brief-${SUCCESS[0]}-$DATE.md" "$MERGED"
else
  write_merged_brief > "$MERGED"
fi
chmod 0600 "$MERGED"
```

---

## Verification Checklist (PR 머지 전 사용자 수동 실행)

- [ ] 2계정 triage 선행 후 brief → 캐시 히트 경로, gws API 호출 0회
- [ ] triage 없이 brief → 2계정 병렬 fetch, merged 브리핑 생성
- [ ] 단일 계정 상태에서 triage → personal 추가 → brief → drift 배너 출력
- [ ] 캐시 히트 vs fresh fetch 지연 비교 — 캐시가 확연히 빠름(~100배+)
- [ ] 한 계정 calendar 만료 상태 → partial merge + 배너, 다른 계정 일정은 정상 표시
- [ ] `stat -f '%p' ~/.gwh/brief-*.md` → 모두 `100600`
- [ ] 2계정 merged 파일 frontmatter 파싱 가능 (`awk '/^---$/{n++}n==1'`)
