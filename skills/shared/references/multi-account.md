# Multi-Account References

> gws-harness 2계정(work/personal) 구조의 공통 패턴 참조. 이 문서는 SKILL.md들이
> 반복적으로 구현하는 로직의 **근거와 표준 구현**을 단일 소스로 기록한다.
> 각 SKILL.md는 self-contained하게 자체 구현을 인라인하되, 설계 배경과
> 수정 시 주의사항은 이 문서를 참조한다.

Related skills:
- `skills/credential-init/` — entrypoint, 이 패턴들의 정합성을 보장
- `skills/triage/` — 2계정 읽기 레퍼런스 (dual write)
- `skills/brief/` — triage 캐시 재사용 + drift 감지
- `skills/digest/` — 주간 집계 2-branch 병렬
- `skills/cal-plan/` — 읽기 병렬 + 쓰기 single-target

---

## § 1. Enrollment Verification (Step 0.4 공통)

### 배경

`~/.gwh/config.yml`의 `accounts.<name>.verified_email`은 `/gwh:credential-init`
실행 당시의 Google OAuth 결과로 기록된다. 이후 다음 상황에서 실제 인증 상태와
어긋날 수 있다:

1. 사용자가 config.yml을 수동 편집
2. `gws auth logout` 후 다른 Google 계정으로 재로그인
3. credential rotation 중 실패로 partial 상태

검증 없이 fetch하면 **사용자가 의도한 계정과 다른 계정의 메일이 "work" 라벨로 노출**되는 위험이 있다.

### 표준 구현

```bash
# 1. config.yml에서 계정 name 목록 추출 (yq 또는 awk 폴백)
ACCOUNT_NAMES=()
if command -v yq &>/dev/null; then
  # NOTE: `-f extract`는 markdown frontmatter 추출용 플래그. ~/.gwh/config.yml은
  # plain YAML이므로 플래그 없이 순수 표현식만 전달한다.
  while IFS= read -r name; do ACCOUNT_NAMES+=("$name"); done \
    < <(yq '.accounts | keys | .[]' ~/.gwh/config.yml)
else
  while IFS= read -r name; do ACCOUNT_NAMES+=("$name"); done < <(awk '
    /^accounts:/ {in_acc=1; next}
    in_acc && /^[a-z0-9]/ {in_acc=0}
    in_acc && /^  [a-z0-9][a-z0-9-]*:/ {gsub(/^  |:.*$/,""); print}
  ' ~/.gwh/config.yml)
fi

# 2. 각 계정에 대해 env-inject gws auth status → 실제 .user와 config 비교
ACCOUNTS=()
ENROLLMENT_FAILS=()
for name in "${ACCOUNT_NAMES[@]}"; do
  EXPECTED=$(yq ".accounts.$name.verified_email" ~/.gwh/config.yml 2>/dev/null)
  ACTUAL=$(GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$HOME/.config/gws-$name" \
           GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
           gws auth status 2>/dev/null | jq -r '.user // empty')

  if [[ -n "$ACTUAL" && "$ACTUAL" == "$EXPECTED" ]]; then
    ACCOUNTS+=("$name")
  else
    ENROLLMENT_FAILS+=("$name: expected=$EXPECTED actual=$ACTUAL")
  fi
done
```

### 사용 스킬

- `triage` Step 0.4 (레퍼런스)
- `brief` Step 0.2
- `digest` Step 0.2
- `cal-plan` Step 0.2

### 수정 시 주의

- v1.2에서 HMAC 서명 도입 시 `verified_email` 단순 비교 → 서명 검증으로 교체. 이때 4개 스킬 모두 갱신 필요.
- `jq`가 stderr에 배너를 출력하는 문제 있으면 `2>/dev/null` 유지 (배너는 `.user` 추출에 영향 없음).

---

## § 2. 2-Branch Parallel Fetch (Step 1 공통 템플릿, D2)

### 배경

2계정 하드코드 (Scope Boundary). `xargs -P`나 `--sequential` 플래그 없이
bash `&` + `wait`로 단순 병렬화. rate limit은 2계정 수준에서 안전.

### 표준 템플릿

```bash
trap 'rm -f /tmp/gwh-<skill>-*-$$.{json,err,exit}' EXIT

for a in "${ACCOUNTS[@]}"; do
  (
    GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$HOME/.config/gws-$a" \
    GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
    gws <command> --format json \
      > "/tmp/gwh-<skill>-$a-$$.json" 2>"/tmp/gwh-<skill>-$a-$$.err"
    echo "$a:$?" > "/tmp/gwh-<skill>-$a-$$.exit"
  ) &
done
wait
```

### 사용 스킬

- `triage` Step 1 (Gmail triage)
- `brief` Step 1.2 (Gmail triage, `--max 30`), Step 2 (Calendar agenda today)
- `digest` Step 1 (Gmail 수신/송신 각각), Step 2 (Calendar agenda week)
- `cal-plan` Step 3.1 (Calendar agenda week)

### 수정 시 주의

- 쿼리/플래그가 스킬마다 다르므로 **완전 공유 함수화는 지양** (콜백 복잡도 증가).
- `/tmp/gwh-<skill>-<account>-$$` prefix는 skill이름을 반드시 포함 (동시 실행 시 충돌 방지).
- trap cleanup은 `EXIT` 시그널에만 걸어 정상/비정상 모두 처리.

---

## § 3. R8 Conditional Label Output

### 배경

단일 계정 사용자(업그레이드 직후, personal 미등록) workflow를 보전. 라벨 prefix
(`[🏢 work]` / `[👤 personal]`)가 있으면 기존 brief의 triage 캐시 파싱 로직과
mail-to-ticket의 triage 참조가 영향을 받는다 (358배 캐시 성능 보전).

### 규칙

- `|ACCOUNTS| == 1` (성공 계정 기준): 기존 포맷, 라벨 없음, merged 파일 = per-account 파일 복사본
- `|ACCOUNTS| >= 2`: merged 파일에 라벨 prefix + frontmatter (`accounts_success`, `accounts_failed`, `generated_at`)
- per-account 파일은 항상 라벨 없음 (이미 계정 확정)

### 사용 스킬

- `triage` Step 4
- `brief` Step 4
- `digest` Step 5
- `cal-plan` — 예외(single-target write, per-account만)

---

## § 4. Dual Write Policy

### 배경

`~/.gwh/<skill>-<date>.md` merged 경로를 기존 brief/mail-to-ticket이 파싱하므로
**merged 파일은 반드시 존재**해야 한다 (R8 하위호환). per-account 파일은 사용자가
특정 계정만 확인할 때 편의.

### 규칙

| 스킬 | 정책 | 경로 |
|------|------|------|
| `triage` | dual write | per-account + merged |
| `brief` | dual write | per-account + merged |
| `digest` | dual write | per-account + merged (주차 단위) |
| `cal-plan` | **single-target** | per-account만 (merged 미갱신) |

### Cal-plan이 예외인 이유

이벤트는 단일 캘린더에 insert되므로 merged 파일에 두 계정의 타임블록을
합쳐 저장해도 **실제 캘린더와 불일치**하는 사실 파일이 된다. per-account 파일이
진실의 유일한 소스.

---

## § 5. Security Inline (Step 5 공통)

### 배경

keyring=file 백엔드로 credentials.enc와 암호화 키가 `~/.config/gws-<name>/`에
평문 파일로 저장된다. `~/.gwh/`에는 메일 본문 스니펫·캘린더 상세가 캐시된다.
양쪽 모두 Time Machine/iCloud/Dropbox 동기화 대상이 되면 보안 경계가 무너진다.

### 필수 적용

```bash
mkdir -p "$HOME/.gwh"
chmod 0700 "$HOME/.gwh"
# 파일 생성 직후
chmod 0600 "$PATH_TO_FILE"
```

`tmutil addexclusion` (user-level xattr 기반, sudo 불필요) 와 cloud-parent REFUSE는 `/gwh:credential-init` Step 6에서 **1회 실행** (idempotent). 개별 스킬에서 재실행 불필요.

### 사용 스킬

- `triage` Step 5
- `brief` Step 5
- `digest` Step 6
- `cal-plan` Step 7
- `mail-to-ticket` Step 5 (Unit F에서 적용)

---

## § 6. Deferred to v1.2

MVP 비목표 (명시적):
- 14일 retention 자동 tidy
- enrollment manifest HMAC 서명
- audit log (`~/.gwh/audit.log` JSONL)
- MDM/backup agent 위협모델 문서화
- R9 `--account <X>` 플래그를 읽기 스킬에도 적용 (현재는 cal-plan만)

이 항목들이 구현될 때 이 문서의 §1 ~ §5가 업데이트 지점이다.
