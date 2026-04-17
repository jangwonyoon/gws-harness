---
name: gwh-triage
description: >
  Gmail 미읽음 메일을 중요도별로 분류하고 액션 아이템을 추출한다.
  2계정(work/personal) 병렬 fetch + merged 뷰 + 시간대 가중치(TZ=Asia/Seoul 기준
  평일 09-18 = work 상향, 저녁·주말 = personal 상향) 지원. 산출물은 per-account 파일 +
  merged 파일로 저장되고 모두 chmod 0600.
  Use when "메일 정리", "메일 트리아지", "메일 확인해줘", "gmail triage",
  "/gwh:triage", "안 읽은 메일", "메일 분류", "inbox zero".
version: 1.1.0
category: automation
tags: [google-workspace, gmail, triage, productivity, multi-account]
triggers:
  - "메일 정리"
  - "메일 트리아지"
  - "메일 확인해줘"
  - "gmail triage"
  - "안 읽은 메일"
dependencies: []
---

# Gmail Triage

미읽음 메일을 중요도별 4단계로 분류하고, 액션 아이템을 추출한다.
`~/.gwh/config.yml`에 등록된 모든 계정을 병렬 fetch하여 merged 뷰로 통합한다.

---

## Step 0: 사전 확인

### 0.1 gws CLI 확인

```bash
which gws
```
미설치 시 → 안내 후 중단: "gws CLI가 필요합니다. https://github.com/googleworkspace/cli"

### 0.2 config.yml 확인

```bash
test -f ~/.gwh/config.yml || {
  echo "❌ ~/.gwh/config.yml 이 없습니다. 먼저 /gwh:credential-init <name>으로 계정을 등록하세요."
  exit 1
}
```

### 0.3 캐시 확인

오늘 날짜의 merged 산출물이 이미 있는지 확인:

```bash
cat ~/.gwh/triage-$(date +%Y-%m-%d).md 2>/dev/null
```

- 존재 → 사용자에게 "오늘 이미 트리아지 결과가 있습니다. 새로 실행할까요?" AskUserQuestion
- 없음 → 진행

### 0.4 ACCOUNTS 확정 (enrollment 검증)

config.yml의 `accounts:` 를 순회하며 실제 인증 상태와 비교. 불일치하는 계정은 skip하고 상단에 배너.

```bash
# 등록된 계정 name 목록 추출
ACCOUNT_NAMES=()
if command -v yq &>/dev/null; then
  while IFS= read -r name; do ACCOUNT_NAMES+=("$name"); done < <(yq -f extract '.accounts | keys | .[]' ~/.gwh/config.yml)
else
  # yq 없으면 awk 폴백 — accounts 하위 2-space 들여쓰기 key 추출
  while IFS= read -r name; do ACCOUNT_NAMES+=("$name"); done < <(awk '
    /^accounts:/ {in_acc=1; next}
    in_acc && /^[a-z0-9]/ {in_acc=0}
    in_acc && /^  [a-z0-9][a-z0-9-]*:/ {gsub(/^  |:.*$/,""); print}
  ' ~/.gwh/config.yml)
fi

ACCOUNTS=()
ENROLLMENT_FAILS=()
for name in "${ACCOUNT_NAMES[@]}"; do
  # config.yml에서 verified_email 조회
  if command -v yq &>/dev/null; then
    EXPECTED=$(yq -f extract ".accounts.$name.verified_email" ~/.gwh/config.yml)
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

> **왜 enrollment 검증이 필요한가**: config.yml 수동 편집 또는 credential rotation 후 `verified_email`이 실제 OAuth 계정과 달라질 수 있다. 검증 없이 fetch하면 사용자가 의도한 것과 다른 계정의 메일이 라벨만 바뀌어 노출된다.

---

## Step 1: 2계정 병렬 fetch (D2 템플릿 inline)

```bash
# trap으로 임시 파일 cleanup 보장
trap 'rm -f /tmp/gwh-triage-*-$$.{json,err,exit}' EXIT

for a in "${ACCOUNTS[@]}"; do
  (
    GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$HOME/.config/gws-$a" \
    GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
    gws gmail +triage --max 50 --query 'is:unread' --format json \
      > "/tmp/gwh-triage-$a-$$.json" 2>"/tmp/gwh-triage-$a-$$.err"
    echo "$a:$?" > "/tmp/gwh-triage-$a-$$.exit"
  ) &
done
wait

# 성공/실패 집계
SUCCESS=()
FAILED=()
for a in "${ACCOUNTS[@]}"; do
  ec=$(cut -d: -f2 "/tmp/gwh-triage-$a-$$.exit")
  if [[ "$ec" == "0" ]]; then
    SUCCESS+=("$a")
  else
    err=$(cat "/tmp/gwh-triage-$a-$$.err" 2>/dev/null | head -3)
    FAILED+=("$a: $err")
  fi
done

if [[ ${#SUCCESS[@]} -eq 0 ]]; then
  echo "❌ 모든 계정 fetch 실패"
  printf '%s\n' "${FAILED[@]}"
  exit 1
fi
```

- `--max 50`: 계정당 최근 50통 (토큰·쿼터 절약)
- 2-branch 하드코드 (Scope Boundary: N-account 지원은 비목표)

> **왜 --max 50인가**: 50통 이상은 분류 품질 체감 향상이 거의 없는 반면 토큰 비용이 선형 증가한다. 대부분의 미읽음 200+통은 뉴스레터/자동알림이므로 최근 50통으로 시그널 대부분 포착 가능.

### ⚠️ resultSizeEstimate 주의사항

Gmail API의 `resultSizeEstimate`는 **정확한 카운트가 아닌 추정치**이며, 많은 결과 세트에서 **201에서 포화**되어 더 많은 실제 메시지가 있어도 201로 표시된다.

- 출력 시 "미읽음 **약** {N}+통" 형식 (`약`과 `+`로 추정임을 암시)
- 일반 트리아지에서는 추정치로 충분

---

## Step 2: 분류 + 시간대 가중치 (D4)

각 메일을 4단계로 분류한 후, 현재 KST를 기준으로 work/personal 항목의 중요도를 조정.

### 2.1 기본 분류

| 등급 | 기준 | 표시 |
|------|------|------|
| 🔴 긴급 | 장애/사고, 마감 임박(24h), 상위자 직접 요청 | 즉시 확인 필요 |
| 🟡 액션 필요 | PR 리뷰, 문서 승인, 회신 요청, 미팅 일정 확인 | 오늘 중 처리 |
| 🟢 FYI | 팀 공지, 변경 알림, 참고용 공유 | 읽기만 |
| ⚪ 무시 가능 | 자동 알림, 뉴스레터, 마케팅, 스팸 | 아카이브 후보 |

분류 근거:
- **발신자 패턴**: noreply@, notifications@, newsletter@ → ⚪
- **제목 키워드**: urgent, 긴급, ASAP, 장애 → 🔴 / review, 승인, 확인 → 🟡
- **수신 유형**: To(직접) vs CC(참조) → CC는 한 단계 낮춤

### 2.2 시간대 가중치 (D4) — `TZ=Asia/Seoul` 강제

```bash
# 시스템 TZ와 무관하게 KST 기준
KST_DOW=$(TZ=Asia/Seoul date +%u)    # 1=Mon .. 7=Sun
KST_HHMM=$(TZ=Asia/Seoul date +%H%M) # 0000..2359

IS_WORK_HOUR=false
# 평일(월-금) 09:00-17:59 → 업무시간
if [[ "$KST_DOW" -le 5 && "$KST_HHMM" -ge 0900 && "$KST_HHMM" -lt 1800 ]]; then
  IS_WORK_HOUR=true
fi
```

규칙:
- `IS_WORK_HOUR=true`: **work** 계정 항목의 🟡 → 🔴 상향
- `IS_WORK_HOUR=false` (저녁·주말): **personal** 계정 항목의 🟡 → 🔴 상향
- 🟢/⚪ 은 상향 없음 (노이즈 방지)
- 공휴일 미지원(MVP 제외)

> **왜 TZ=Asia/Seoul 강제인가**: 해외 출장 등 시스템 TZ ≠ KST 상황에서도 "KHC 업무시간" 일관 적용. macOS 기본 locale이 UTC/PST인 경우의 오판 방지.

---

## Step 3: 액션 아이템 추출

🔴/🟡 등급 메일에서 액션 아이템을 추출:

- **할 일**: "~해주세요", "확인 부탁", "리뷰 요청" 등
- **마감일**: 날짜 패턴 감지 (오늘, 내일, 이번 주, 특정 날짜)
- **대상**: Jira 티켓 번호, PR 번호, 문서 링크

---

## Step 4: 결과 출력 (R8 조건부 라벨)

### 4.1 단일 계정 (|ACCOUNTS| == 1): 기존 포맷 유지

```markdown
# 📬 Gmail Triage — {YYYY-MM-DD}

**미읽음 약 {N}+통 중 {M}통 분석 완료**

## 🔴 긴급 ({N}건)
1. **[발신자]** {제목}
   → 액션: {할 일} | 마감: {마감일}
```

라벨 prefix 없음 (기존 사용자 workflow 무변경).

### 4.2 2계정 (|ACCOUNTS| >= 2): merged 뷰 + 라벨

```markdown
---
accounts_success: [work, personal]
accounts_failed: []
generated_at: 2026-04-17T10:30:00+09:00
kst_weight: work_hours=true
---

# 📬 Gmail Triage — {YYYY-MM-DD} (통합)

**work 약 {N}+통 / personal 약 {M}+통 분석 완료**

## 🔴 긴급 ({N}건)
1. [🏢 work] **[발신자]** {제목}
   → 액션: {할 일}
2. [👤 personal] **[발신자]** {제목}
   → 액션: {할 일}

## 🟡 액션 필요 ({N}건)
1. [🏢 work] **[발신자]** {제목}
   ...
```

정렬 규칙 (D3): 🔴→🟡→🟢→⚪ 계층 우선, 같은 계층 내 시간 최신 순, tiebreak로 계정 이름 알파벳(personal < work).

### 4.3 Partial merge (한 계정만 성공)

실패 계정을 상단 배너로 경고:

```markdown
⚠️ personal: fetch 실패 (인증 만료). /gwh:credential-init personal 재실행 권장.

---
accounts_success: [work]
accounts_failed: [{name: personal, reason: "auth expired"}]
---

# 📬 Gmail Triage — {YYYY-MM-DD} (work만)
...
```

---

## Step 5: 산출물 저장 (dual write + chmod inline)

```bash
mkdir -p "$HOME/.gwh"
chmod 0700 "$HOME/.gwh"

# sync-excluded 마커 (문서화 신호)
[[ -f "$HOME/.gwh/.sync-excluded" ]] || {
  echo "# gws-harness cache — do NOT sync (contains decrypted creds in ../config/gws-*/)" \
    > "$HOME/.gwh/.sync-excluded"
  chmod 0600 "$HOME/.gwh/.sync-excluded"
}

DATE=$(date +%Y-%m-%d)

# 각 계정별 per-account 파일 (라벨 prefix 없음 — 이미 계정 확정)
for a in "${SUCCESS[@]}"; do
  PER="$HOME/.gwh/triage-$a-$DATE.md"
  write_per_account_file "$a" > "$PER"   # 구현 시 Step 4.1 포맷 사용
  chmod 0600 "$PER"
done

# Merged 파일
MERGED="$HOME/.gwh/triage-$DATE.md"
if [[ ${#SUCCESS[@]} -eq 1 ]]; then
  # 단일 성공: 기존 포맷(라벨 없음) — R8 하위호환 유지
  cp "$HOME/.gwh/triage-${SUCCESS[0]}-$DATE.md" "$MERGED"
else
  # 2계정 성공: frontmatter + 라벨 prefix 포함 merged
  write_merged_file > "$MERGED"
fi
chmod 0600 "$MERGED"
```

> **왜 dual write인가**: per-account 파일은 사용자가 특정 계정만 다시 살펴볼 때 사용 (`cat ~/.gwh/triage-work-$(date +%F).md`).
> merged 파일은 `/gwh:brief` Step 0.3의 358배 캐시 경로가 재사용 — 포맷이 라벨 포함이어도 파싱 가능해야 한다 (brief Unit C에서 처리).

### 5.1 R8 하위호환 보장

단일 계정 등록 사용자(예: migration 직후 work만)는 merged 파일 = per-account 파일 복사본 → 라벨 없음 → 기존 brief/mail-to-ticket 파싱 경로 그대로 동작.

---

## Verification Checklist (PR 머지 전 사용자 수동 실행)

- [ ] 단일 계정 상태에서 `/gwh:triage` → 기존 포맷(라벨 없음), `~/.gwh/triage-{date}.md` 생성
- [ ] 2계정 등록 후 `/gwh:triage` → per-account 2개 + merged 1개, merged frontmatter에 `accounts_success: [work, personal]`
- [ ] `stat -f '%p' ~/.gwh/triage-*.md | sort -u` → 모두 `100600`
- [ ] KST 평일 10:30 실행 → work `🟡` 메일이 `🔴`로 상향된 라인 존재
- [ ] KST 토요일 14:00 실행 → personal 상향, work 변동 없음
- [ ] `TZ=America/Los_Angeles /gwh:triage` → 서울 시간 기준 상향 규칙 동일 (D4 강제 확인)
- [ ] 한 계정 credential 수동 망가뜨림 → partial merge + 상단 배너, merged frontmatter `accounts_failed` 엔트리
- [ ] config.yml `verified_email` 수동 편집 → enrollment 불일치 배너, 해당 계정 skip
