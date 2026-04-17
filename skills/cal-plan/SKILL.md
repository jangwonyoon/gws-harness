---
name: gwh-cal-plan
description: >
  Jira 스프린트 티켓을 시간 단위로 분해하여 Google Calendar에 타임블록으로 배치한다.
  2계정(work/personal) 읽기는 병렬 fetch + 1회 retry + 재실패 시 --force --account
  override 경로, 쓰기는 반드시 단일 계정 지정 (--account <X> 또는 AskUserQuestion).
  산출물은 per-account 파일에만 저장 (merged 경로 미갱신) — single-target write 정책.
  Use when "일정 짜줘", "타임블록", "캘린더 계획", "cal plan",
  "/gwh:cal-plan", "이번 주 계획", "시간 배분".
version: 1.1.0
category: automation
tags: [google-workspace, calendar, jira, timeblock, planning, productivity, multi-account]
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

> **중요 — 쓰기 정책**: cal-plan은 읽기(빈 시간 계산)는 2계정 union으로 하지만,
> 쓰기(이벤트 insert)는 **반드시 단일 계정**에 한정한다. `--account <X>` 미지정 시
> AskUserQuestion으로 강제 선택. 잘못된 계정에 이벤트 leak 방지.

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

> 이 블록은 `/gwh:triage` Step 0.4 및 `/gwh:brief`·`/gwh:digest` Step 0.2와 동일 — SKILL.md self-containment를 위해 인라인 복사. 설계 배경은 [`skills/shared/references/multi-account.md § 1`](../shared/references/multi-account.md) 참조.

### 0.3 태스크 소스 결정

| 소스 | 감지 방법 | 동작 |
|------|----------|------|
| Jira MCP | `mcp__atlassian__jira_get_user_profile` 성공 | 스프린트 티켓 자동 조회 |
| 수동 입력 | Jira 미연결 또는 사용자 선택 | AskUserQuestion으로 태스크 목록 입력 |

### 0.4 플래그 파싱

- `--account <X>`: 쓰기 대상 계정 (Step 5/6에 영향)
- `--force`: Step 3 재실패 시 override 경로 활성화

---

## Step 1: 태스크 수집

### Jira 경로 (계정 중립)

1. `mcp__atlassian__jira_get_agile_boards` → 활성 보드
2. `mcp__atlassian__jira_get_sprints_from_board` → 현재 스프린트
3. `mcp__atlassian__jira_get_sprint_issues` → 나에게 할당된 To Do / In Progress

각 티켓에서 추출: 키, 제목, Story Point, 우선순위(Highest > High > Medium > Low).

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

> **왜 SP→시간 변환이 필요한가**: Jira SP는 상대적 복잡도, 캘린더는 절대 시간. 팀별 속도가 다르므로 기본 테이블은 조정 필요. 사용자가 재조정 기회를 갖도록 명시적으로 출력.

---

## Step 3: 빈 시간 감지 — 2계정 병렬 + retry + fail-closed

### 3.1 2-branch 병렬 fetch (D2)

```bash
trap 'rm -f /tmp/gwh-calplan-*-$$.{json,err,exit}' EXIT

fetch_week() {
  for a in "${ACCOUNTS[@]}"; do
    (
      GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$HOME/.config/gws-$a" \
      GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
      gws calendar +agenda --week --format json \
        > "/tmp/gwh-calplan-$a-$$.json" 2>"/tmp/gwh-calplan-$a-$$.err"
      echo "$a:$?" > "/tmp/gwh-calplan-$a-$$.exit"
    ) &
  done
  wait
}

fetch_week
```

### 3.2 실패 계정 1회 retry (exponential backoff 2s)

```bash
FAILED=()
for a in "${ACCOUNTS[@]}"; do
  ec=$(cut -d: -f2 "/tmp/gwh-calplan-$a-$$.exit")
  [[ "$ec" != "0" ]] && FAILED+=("$a")
done

if [[ ${#FAILED[@]} -gt 0 ]]; then
  echo "⚠️  첫 fetch 실패: ${FAILED[*]} — 2초 후 재시도"
  sleep 2
  ACCOUNTS=("${FAILED[@]}") fetch_week
fi

# 최종 성공/실패 재집계
SUCCESS=()
FINAL_FAILED=()
for a in "${ACCOUNTS[@]}"; do
  ec=$(cut -d: -f2 "/tmp/gwh-calplan-$a-$$.exit")
  if [[ "$ec" == "0" ]]; then SUCCESS+=("$a"); else FINAL_FAILED+=("$a"); fi
done
```

### 3.3 재실패 시 fail-closed + override

```bash
if [[ ${#FINAL_FAILED[@]} -gt 0 ]]; then
  if [[ "$FORCE" == "true" && -n "$WRITE_ACCOUNT" ]]; then
    # override 경로: 성공 계정만으로 계산 진행
    echo "⚠️  --force 모드: ${FINAL_FAILED[*]} 일정 확인 불가."
    echo "   이벤트 설명에 [UNVERIFIED: ${FINAL_FAILED[*]}] 스탬프 추가됨."
    UNVERIFIED_STAMP="[UNVERIFIED: ${FINAL_FAILED[*]} 일정 확인 불가]"
  else
    echo "⛔ ${FINAL_FAILED[*]} 캘린더 fetch 재실패. 일부 계정 일정 미확인 상태로 이벤트 생성 금지."
    echo "   재시도: /gwh:cal-plan"
    echo "   강제 진행: /gwh:cal-plan --force --account <work|personal>"
    exit 1
  fi
fi
```

> **왜 fail-closed인가**: 한 계정 일정을 못 본 상태로 이벤트를 insert하면 실제 미팅과 충돌하는 타임블록이 생성된다. `--force` override는 사용자가 의식적으로 위험을 감수할 때만 허용하고, 이벤트 설명에 스탬프를 박아 나중에 식별 가능하게 한다.

### 3.4 빈 시간 계산 (성공 계정 union)

JSON 결과에서:
- 각 계정의 일정을 union → 합친 busy 목록
- 업무 시간대(기본 09:00-18:00) 내 빈 슬롯 계산
- 30분 미만 빈 시간은 무시
- 점심시간(12:00-13:00) 자동 제외

---

## Step 4: 타임블록 배치 제안

배치 규칙:
1. **우선순위 높은 티켓 먼저** (Highest → High → Medium → Low)
2. **큰 블록 먼저 배치** (연속 3h+ 슬롯에 큰 태스크)
3. **같은 태스크는 연속 배치** (컨텍스트 스위칭 최소화)
4. **SP 8+ 태스크는 반드시 분할** (하루 최대 4h 연속)

출력:

```markdown
## 📅 타임블록 배치 제안 (union: work + personal 기준)

### 월요일 (4/17)
| 시간 | 태스크 | 예상 |
|------|--------|------|
| 09:00-12:00 | DPPM-91 hex 감사 (1/2) | 3h |
| 14:00-17:00 | DPPM-91 hex 감사 (2/2) | 3h |

⚠️ 목/금 미팅 4개로 가용시간 3h만 남음 → 우선순위 조정 필요
```

---

## Step 5: 쓰기 대상 계정 결정

### 5.1 `--account <X>` 명시된 경우

곧바로 Step 6.

### 5.2 플래그 없음

AskUserQuestion으로 **단일 선택**:

```
이벤트를 어느 계정 캘린더에 생성할까요?

- 🏢 work (kakaohealthcare.com)
- 👤 personal (gmail.com)
- 취소
```

> **왜 자동 라우팅하지 않나**: 태스크가 Jira 티켓이라도 업무 외 시간에 배치되면 personal에 넣는 게 맞을 수도 있고, 반대도 가능. 안전한 기본값은 없으므로 항상 사용자 확인.

### 5.3 배치 승인 AskUserQuestion

- **승인** → Step 6
- **수정** → 변경사항 반영 후 재제안
- **취소** → Step 7만 실행 (per-account md만 저장, 캘린더 미반영)

> **왜 승인 게이트가 필수인가**: 한 번 호출로 최대 16개 캘린더 이벤트 생성 가능. 잘못 배치된 이벤트 수동 삭제 비용(~30분)이 승인 비용(~10초)보다 수백 배 크다.

---

## Step 6: 캘린더 이벤트 생성 (단일 계정)

```bash
for block in "${APPROVED_BLOCKS[@]}"; do
  DESC="Story Point: $SP | Priority: $PRIO"
  [[ -n "$UNVERIFIED_STAMP" ]] && DESC="$DESC\n$UNVERIFIED_STAMP"

  GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$HOME/.config/gws-$WRITE_ACCOUNT" \
  GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
  gws calendar +insert \
    --summary "[$KEY] $TITLE" \
    --start "$START" \
    --end "$END" \
    --description "$DESC"
done
```

각 이벤트 생성 후 성공/실패 보고.

> **이벤트 leak 방지**: `GOOGLE_WORKSPACE_CLI_CONFIG_DIR`는 `$WRITE_ACCOUNT`에 고정.
> ACCOUNTS 반복문 안에서 실행하지 않음 — single-target write 강제.

---

## Step 7: 산출물 저장 (single-target write — per-account만)

```bash
mkdir -p "$HOME/.gwh"
chmod 0700 "$HOME/.gwh"

DATE=$(date +%Y-%m-%d)
PER="$HOME/.gwh/cal-plan-$WRITE_ACCOUNT-$DATE.md"
write_cal_plan_output > "$PER"
chmod 0600 "$PER"
```

- **merged 경로(`cal-plan-{date}.md`) 미갱신** — 이벤트가 한 계정에만 있으므로 merged 의미 없음.
- 파일 내용: 배치된 블록, 미배치 태스크, 가용 시간 통계, `UNVERIFIED_STAMP` (있으면).

---

## Verification Checklist (PR 머지 전 사용자 수동 실행)

- [ ] 2계정 fetch + `--account work` 쓰기 → 이벤트는 work 캘린더에만, personal 캘린더 변동 없음
- [ ] `--account` 미지정 → AskUserQuestion 단일 선택 강제
- [ ] 단일 계정 환경에서 `--account` 없이 실행 → 해당 계정에 자동(AskUserQuestion 선택지가 1개)
- [ ] 첫 fetch 실패 → 2초 후 재시도 → 성공 시 정상 진행
- [ ] 재실패 + `--force` 없음 → 이벤트 생성 0건, 명확한 중단 메시지
- [ ] 재실패 + `--force --account work` → 이벤트 설명에 `[UNVERIFIED: personal 일정 확인 불가]` 스탬프 실제 포함
- [ ] 사용자 AskUserQuestion 취소 → 이벤트 미생성, per-account md만 저장
- [ ] 산출물 파일 `~/.gwh/cal-plan-<account>-{date}.md` 권한 `100600`, merged `cal-plan-{date}.md` 미생성/미갱신
- [ ] personal 일정과 work 스프린트 미팅 overlap → 빈 시간 계산에서 둘 다 busy로 처리
