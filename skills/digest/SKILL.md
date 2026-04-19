---
name: gwh-digest
description: >
  주간 다이제스트 — 이번 주 메일 송수신 + 캘린더 참석 + (optional) Jira 완료 티켓 요약.
  2계정(work/personal) 병렬 fetch 지원. 각 계정 송수신 통계 + 캘린더 참석 통계를
  집계하고 Jira 스프린트는 계정 중립 단일 쿼리로 처리한다. 산출물은
  ~/.gwh/digest-{YYYY}-W{WW}-<account>.md (per-account) + digest-{YYYY}-W{WW}.md (merged).
  Use when "이번 주 정리", "주간 요약", "weekly digest",
  "/gwh:digest", "이번 주 뭐 했지", "주간 리포트", "회고 자료".
version: 1.1.0
category: automation
tags: [google-workspace, gmail, calendar, jira, digest, retrospective, multi-account]
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

### 0.1 gws CLI 확인

```bash
which gws
```

### 0.2 config.yml + ACCOUNTS 확정

```bash
test -f ~/.gwh/config.yml || {
  echo "❌ ~/.gwh/config.yml 이 없습니다. 먼저 /gwh:credential-init <name>으로 계정을 등록하세요."
  exit 1
}

ACCOUNT_NAMES=()
if command -v yq &>/dev/null; then
  while IFS= read -r name; do ACCOUNT_NAMES+=("$name"); done < <(yq '.accounts | keys | .[]' ~/.gwh/config.yml)
else
  while IFS= read -r name; do ACCOUNT_NAMES+=("$name"); done < <(awk '
    /^accounts:/ {in_acc=1; next}
    in_acc && /^[a-z0-9]/ {in_acc=0}
    in_acc && /^  [a-z0-9][a-z0-9-]*:/ {gsub(/^  |:.*$/,""); print}
  ' ~/.gwh/config.yml)
fi

ACCOUNTS=()
ENROLLMENT_FAILS=()
for name in "${ACCOUNT_NAMES[@]}"; do
  if command -v yq &>/dev/null; then
    EXPECTED=$(yq ".accounts.$name.verified_email" ~/.gwh/config.yml)
  else
    EXPECTED=$(awk -v n="$name" '
      $0 ~ "^  " n ":" {in_n=1; next}
      in_n && /^  [a-z0-9]/ {in_n=0}
      in_n && /verified_email:/ {gsub(/.*verified_email: */,""); print; exit}
    ' ~/.gwh/config.yml)
  fi

  ACTUAL=$(GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$HOME/.config/gws-$name" \
           GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
           gws auth status 2>/dev/null | jq -r '.user // empty')

  if [[ -n "$ACTUAL" && "$ACTUAL" == "$EXPECTED" ]]; then
    ACCOUNTS+=("$name")
  else
    ENROLLMENT_FAILS+=("$name: expected=$EXPECTED actual=$ACTUAL")
  fi
done

if [[ ${#ACCOUNTS[@]} -eq 0 ]]; then
  echo "❌ 인증된 계정이 없습니다. /gwh:credential-init을 재실행하세요."
  exit 1
fi
```

> 이 블록은 `/gwh:triage` Step 0.4 및 `/gwh:brief`·`/gwh:cal-plan` Step 0.2와 동일 — SKILL.md self-containment를 위해 인라인 복사. 설계 배경은 [`skills/shared/references/multi-account.md § 1`](../shared/references/multi-account.md) 참조.

### 0.3 확장 기능 감지

Jira MCP 연결 여부 확인. 미연결 시 코어(메일+캘린더)만 실행.

### 0.4 기간 결정

기본: 이번 주 (월~오늘).
주 초(월/화) 실행 시 → AskUserQuestion: "데이터가 적습니다. 지난주 데이터를 포함할까요?"

```bash
WEEK_START=$(date -v-monday +%Y/%m/%d 2>/dev/null || date -d "last monday" +%Y/%m/%d)
WEEK_END=$(date +%Y/%m/%d)
YEAR_WEEK=$(date +%Y-W%V)
```

---

## Step 1: 📬 메일 통계 (2-branch 병렬)

수신 + 송신을 각각 2계정 병렬로 집계.

```bash
trap 'rm -f /tmp/gwh-digest-*-$$.{json,err,exit}' EXIT

# 수신
for a in "${ACCOUNTS[@]}"; do
  (
    GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$HOME/.config/gws-$a" \
    GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
    gws gmail +triage --query "after:$WEEK_START before:$WEEK_END" --max 200 --format json \
      > "/tmp/gwh-digest-in-$a-$$.json" 2>"/tmp/gwh-digest-in-$a-$$.err"
    echo "$a:$?" > "/tmp/gwh-digest-in-$a-$$.exit"
  ) &
done
wait

# 송신
for a in "${ACCOUNTS[@]}"; do
  (
    GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$HOME/.config/gws-$a" \
    GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
    gws gmail +triage --query "from:me after:$WEEK_START before:$WEEK_END" --max 200 --format json \
      > "/tmp/gwh-digest-out-$a-$$.json" 2>"/tmp/gwh-digest-out-$a-$$.err"
    echo "$a:$?" > "/tmp/gwh-digest-out-$a-$$.exit"
  ) &
done
wait
```

계정별 집계:
- **수신:** 총 N통 (카테고리별 분류)
- **송신:** M통 (수신자별 그룹핑 — 어떤 팀/사람과 가장 많이 소통했는지)
- 송신 0건은 "송신 없음"이 아닌 **"송신 0건"** 으로 명시 (누락 아닌 0 값)

---

## Step 2: 📅 캘린더 통계 (2-branch 병렬)

```bash
for a in "${ACCOUNTS[@]}"; do
  (
    GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$HOME/.config/gws-$a" \
    GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
    gws calendar +agenda --days 7 --format json \
      > "/tmp/gwh-digest-cal-$a-$$.json" 2>"/tmp/gwh-digest-cal-$a-$$.err"
    echo "$a:$?" > "/tmp/gwh-digest-cal-$a-$$.exit"
  ) &
done
wait
```

집계:
- **미팅 수:** N회 (계정별 + 합계)
- **미팅 시간:** 총 Mh
- **미팅 비중:** work 업무시간(40h) 대비 % (personal 미팅은 별도 집계, 업무시간에 포함 안 함)
- **참석자 빈도:** 가장 자주 만난 사람 Top 3 (계정별)
- **전주 대비:** 미팅 ±N회, 시간 ±Mh

---

## Step 3: 🏃 Jira 통계 — 계정 중립 (optional)

Jira MCP 연결 시 **단일 쿼리** (사용자 1명 = Jira 계정 1개 전제):

1. `mcp__atlassian__jira_search_issues` — `assignee = currentUser() AND status changed to Done DURING (startOfWeek(), now())`
2. 집계:
   - **완료 티켓:** N건 (SP 합계 M)
   - **티켓 유형별:** Bug N / Story N / Task N
   - **전주 대비:** ±N건, ±M SP

> **왜 계정 중립인가**: Jira 계정은 회사 SSO로 단일화되어 있다. gws 2계정이어도 Jira는 1개 — 중복 쿼리 불필요.

---

## Step 4: 💡 인사이트 도출

자동 인사이트:
- **코딩 가용시간:** 40h - work 미팅시간 - (work 송신 추정시간: 1통당 ~5분)
- **미팅 과부하 경고:** work 미팅 비중 > 30% 시 경고
- **소통 편중 경고:** 특정 사람과 메일 50%+ 시 알림 (계정별 독립 판정)
- **생산성 추정:** 완료 SP / 코딩 가용시간 = SP/h 효율

> **왜 30% 임계인가**: Paul Graham의 Maker's Schedule vs Manager's Schedule — 미팅 30% 초과 시 개발 몰입 시간이 1시간 이하로 파편화되어 복잡한 작업 불가.

> **왜 주간 단위인가**: 일간은 노이즈(회의 몰린 날), 월간은 피드백 루프 지연. 스프린트 주기(1-2주)와 일치해 회고/계획에 즉시 활용.

---

## Step 5: 결과 출력 (R8 조건부 라벨)

### 단일 계정 (|ACCOUNTS| == 1)

기존 포맷 유지 (라벨 없음, 섹션 구조 동일).

### 2계정 (|ACCOUNTS| >= 2)

```markdown
---
accounts_success: [work, personal]
accounts_failed: []
generated_at: 2026-04-17T18:00:00+09:00
week: 2026-W16
---

# 📊 Weekly Digest — {YYYY}-W{WW} ({시작일}~{종료일}) (통합)

## 📬 메일

### 🏢 work
- 수신: {N}통 | 송신: {M}통
- 주요 소통: {사람} ({N}통)

### 👤 personal
- 수신: {N}통 | 송신: {M}통 (또는 "송신 0건")

## 📅 미팅

| 계정 | 횟수 | 시간 | 업무시간 대비 |
|------|------|------|--------------|
| 🏢 work | {N}회 | {M}h | {%} |
| 👤 personal | {N}회 | {M}h | (참고) |

빈도 Top 3 (계정별):
- 🏢 work: {사람1}, {사람2}, {사람3}
- 👤 personal: ...

## 🏃 스프린트 (계정 통합)
- 완료: {N}건 (SP {M})
- Bug {N} / Story {N} / Task {N}
- 전주 대비: {±변화}

## 💡 인사이트
- work 코딩 가용시간: {N}h
- work 미팅 비중: {M}% {⚠️ if > 30%}
- SP 효율: {N} SP/h
```

### Partial merge (한 계정 실패)

상단 배너 + frontmatter `accounts_failed`에 실패 사유 기록. 섹션은 성공한 계정만 포함.

---

## Step 6: 산출물 저장 (dual write + chmod inline)

```bash
mkdir -p "$HOME/.gwh"
chmod 0700 "$HOME/.gwh"

# per-account
for a in "${SUCCESS[@]}"; do
  PER="$HOME/.gwh/digest-$YEAR_WEEK-$a.md"
  write_per_account_digest "$a" > "$PER"
  chmod 0600 "$PER"
done

# merged
MERGED="$HOME/.gwh/digest-$YEAR_WEEK.md"
if [[ ${#SUCCESS[@]} -eq 1 ]]; then
  cp "$HOME/.gwh/digest-$YEAR_WEEK-${SUCCESS[0]}.md" "$MERGED"
else
  write_merged_digest > "$MERGED"
fi
chmod 0600 "$MERGED"
```

> **주차 overwrite 정책**: 같은 주차 재실행 시 기존 파일 덮어쓰기 (변경 감지/append 없음). 주 중간 갱신 빈도 낮음.

---

## Verification Checklist (PR 머지 전 사용자 수동 실행)

- [ ] 단일 계정: 기존 포맷 유지, `~/.gwh/digest-{YYYY}-W{WW}.md` 생성
- [ ] 2계정: per-account 2개 + merged 1개, merged에 work/personal 섹션 모두 등장 + Jira 단일 섹션
- [ ] 한 계정 송신 0건 → "송신 0건" 명시적 출력 (누락 아님)
- [ ] 한 계정 fetch 실패 → partial merge + frontmatter `accounts_failed` 엔트리 + 상단 배너
- [ ] 같은 주차 재실행 → 기존 파일 overwrite
- [ ] 모든 산출물 파일 권한 `100600`
- [ ] Jira 미연결 시 🏃 섹션 생략, 코어 섹션만 출력
