---
name: gwh-credential-init
description: >
  gws-harness 계정(work/personal) 등록 진입점. 신규 Google Workspace 계정을
  `~/.config/gws-<name>/`에 격리 등록하고, 기존 `~/.config/gws/`가 있으면
  자동 migration을 수행한다. `~/.gwh/config.yml`에 `verified_email`과 라벨을
  atomic write로 기입하고, 보안 기본값(디렉토리 0700 · 파일 0600 ·
  Time Machine 제외 · 클라우드 동기화 경로 차단)을 설정한다.
  Use when "계정 추가", "credential init", "/gwh:credential-init", "계정 등록",
  "work 계정 세팅", "personal 계정 세팅".
version: 1.0.0
category: automation
tags: [google-workspace, credentials, multi-account, onboarding, security]
triggers:
  - "계정 추가"
  - "credential init"
  - "/gwh:credential-init"
  - "계정 등록"
  - "work 계정 세팅"
  - "personal 계정 세팅"
dependencies: []
---

# Credential Init

gws-harness의 2계정(work/personal) 구조 진입점.
`<name>`으로 지정된 credential 디렉토리를 생성·migrate하고, `~/.gwh/config.yml`에 등록한다.

> **전제**: 이 스킬은 2계정 하드코드 설계(`<name>` = `work` | `personal` 권장)를 따른다.
> N계정 일반화는 비목표이며, 3번째 계정이 실제 필요해질 때 별도 작업에서 다룬다.

---

## Step 0: 사전 확인

### 0.1 gws CLI 설치 확인

```bash
command -v gws &>/dev/null || {
  echo "❌ gws CLI가 설치되어 있지 않습니다."
  echo "   설치: https://github.com/googleworkspace/cli"
  exit 1
}
```

> **왜 `gws auth status`는 여기서 체크하지 않나**: credential-init의 목적은 **새 계정을 등록**하는 것이다. 기존 계정이 이미 로그인되어 있으면 Step 1의 migration 추론에서 활용되고, 아예 미인증이면 Step 3의 `gws auth login`이 처리한다. Step 0에서 인증 상태를 막으면 정상 플로우가 차단된다.

### 0.2 `<name>` 정규식 + reserved 검사

사용자가 제공한 `<name>` (예: `work`, `personal`)을 검증.

```bash
NAME="$1"   # 예: work

# 정규식: 소문자·숫자 시작, 하이픈 허용 (leading hyphen 금지)
if [[ ! "$NAME" =~ ^[a-z0-9][a-z0-9-]*$ ]]; then
  echo "❌ <name>은 소문자/숫자로 시작하고 [a-z0-9-]만 사용 가능합니다."
  exit 1
fi

# reserved
case "$NAME" in
  config|shared|.|..) echo "❌ '$NAME'은 예약어입니다."; exit 1 ;;
esac
```

### 0.3 symlink / 기존 비정상 엔트리 감지

```bash
TARGET="$HOME/.config/gws-$NAME"

if [[ -L "$TARGET" ]]; then
  echo "❌ $TARGET 이 심볼릭 링크입니다. 로컬 디렉토리여야 합니다."
  exit 1
fi
```

### 0.4 macOS case-insensitive 충돌 체크

```bash
# lowercase normalize 후 동일 inode에 매핑되는 다른 표기 dir 있는지 점검
LOWER=$(printf '%s' "$NAME" | tr '[:upper:]' '[:lower:]')
if [[ "$NAME" != "$LOWER" ]]; then
  echo "⚠️  macOS는 case-insensitive 기본입니다. '$NAME' 대신 '$LOWER'를 사용하세요."
  exit 1
fi
if [[ -d "$HOME/.config/gws-$LOWER" && "$NAME" != "$LOWER" ]]; then
  echo "❌ 기존 '$LOWER' 계정과 충돌합니다."
  exit 1
fi
```

---

## Step 1: Migration 3-way 분기 (D6)

`~/.config/gws/` (단일 계정 구조)가 있으면 자동 migration. 없으면 신규 setup.

### 1.1 기존 경로 확인

```bash
OLD="$HOME/.config/gws"
EXISTS_OLD=false
[[ -d "$OLD" && ! -L "$OLD" ]] && EXISTS_OLD=true
```

### 1.2 추론 (기존 경로가 있을 때만)

기존 계정의 이메일로 target 계정 추론:

```bash
INFERRED=""
if $EXISTS_OLD; then
  # 기존 경로는 환경변수 없이 기본 동작
  OLD_EMAIL=$(gws auth status 2>/dev/null | jq -r '.user // empty')
  case "$OLD_EMAIL" in
    *@kakaohealthcare.com) INFERRED="work" ;;
    *@gmail.com)           INFERRED="personal" ;;
    "")                    INFERRED="" ;;  # 추론 실패
    *)                     INFERRED="" ;;  # 알 수 없는 도메인
  esac
fi
```

### 1.3 분기 결정 (AskUserQuestion)

| 상황 | 분기 | 사용자 확인 |
|------|------|------------|
| `$EXISTS_OLD` == false | **신규 setup**만 진행 | 불필요 |
| `$EXISTS_OLD` && `$INFERRED` == `$NAME` | migrate | 단일 승인 AskUserQuestion: `예 / 취소` |
| `$EXISTS_OLD` && `$INFERRED` != `$NAME` (공백 포함) | migrate + 라벨 선택 | 2지선다 AskUserQuestion: `추론 [$INFERRED]로 migrate / command [$NAME]로 override` |
| `$EXISTS_OLD` && target dir 이미 존재 | 보호 | 3지선다 AskUserQuestion: `기존을 .bak-{ts}로 백업 후 migrate / 기존 dir 그대로 활용 / 취소` |

> Claude는 위 표를 따라 `AskUserQuestion` tool로 선택지를 제공하고, 사용자 응답에 따라 아래 1.4 / 1.5를 실행한다.

### 1.4 Same-device 검증 (migrate 분기에서만)

```bash
SRC_DEV=$(stat -f '%d' "$OLD")
DST_PARENT_DEV=$(stat -f '%d' "$HOME/.config")
if [[ "$SRC_DEV" != "$DST_PARENT_DEV" ]]; then
  echo "❌ cross-volume 상황입니다 (외부 SSD의 ~/.config 링크 등). 자동 migrate 불가."
  echo "   수동으로 디렉토리를 내부 볼륨으로 옮긴 후 재실행하세요."
  exit 1
fi
```

### 1.5 Migration 실행

```bash
# trap: migrate 실패 시 target 정리 + 원본 복원 시도
cleanup_migration() {
  if [[ -d "$TARGET" && ! -d "$OLD" ]]; then
    # rename 후 실패 → 원복 시도
    mv "$TARGET" "$OLD" 2>/dev/null || rm -P "$TARGET"/* 2>/dev/null
  fi
}
trap cleanup_migration ERR

# 백업 분기 (3지선다에서 선택된 경우)
if [[ -d "$TARGET" ]]; then
  mv "$TARGET" "$TARGET.bak-$(date +%s)"
fi

mv "$OLD" "$TARGET"
trap - ERR
```

> **동시 실행 방지**: `flock ~/.gwh/config.yml.lock` (Step 5에서 수행). migrate 자체의 동시 실행은 `~/.config/gws-*`가 inode-level에서 충돌하지 않는 한 OS의 `mv`가 직렬화한다.

### 1.6 SIGINT 핸들링

OAuth flow (Step 3)에서 사용자가 Ctrl-C → trap handler가 target dir 제거 + config.yml 미기입 보장.

```bash
trap 'rm -rf "$TARGET" 2>/dev/null; exit 130' INT TERM
```

---

## Step 2: 디렉토리 준비

```bash
mkdir -p "$TARGET"
chmod 0700 "$TARGET"
```

---

## Step 3: OAuth setup + login

### 3.1 gcloud 감지

```bash
if command -v gcloud &>/dev/null; then
  GCLOUD_OK=true
else
  GCLOUD_OK=false
fi
```

### 3.2 `gws auth setup` (옵션)

gcloud가 있으면:

```bash
GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$TARGET" \
GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
gws auth setup
```

없으면 setup 스킵하고 기존 GCP project의 `client_secret.json` 재사용 안내:

```bash
# 다른 credential dir에서 client_secret.json 있으면 복사 제안
EXISTING_SECRET=$(ls -1 "$HOME"/.config/gws-*/client_secret.json 2>/dev/null | head -1)
if [[ -n "$EXISTING_SECRET" ]]; then
  cp "$EXISTING_SECRET" "$TARGET/client_secret.json"
  chmod 0600 "$TARGET/client_secret.json"
  echo "♻️  기존 client_secret.json 재사용 (같은 GCP project 전제)"
else
  echo "⚠️  gcloud 미설치 + 재사용 가능한 client_secret.json 없음."
  echo "   https://github.com/googleworkspace/cli#setup 참조 후 재실행"
  exit 1
fi
```

> **재사용 실패 가능성**: 기존 `client_secret.json`의 OAuth client가 GCP Console에서 이미 삭제되었거나 invalid한 경우(401 `invalid_client`) Step 3.3에서 login이 실패한다. 이때는 `gws auth setup`을 재실행하여 새 client를 수동 생성해야 한다. Step 3.3 에러 핸들링 참조.

### 3.3 `gws auth login`

```bash
GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$TARGET" \
GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
gws auth login
```

> **keyring=file 배경**: 기본 Keychain backend는 계정별 격리가 어렵다.
> file 백엔드는 `$TARGET/credentials.enc`와 암호화 키 파일을 모두 디렉토리 내부에 둔다.
> Keychain 대비 보안이 하락하므로 Step 2의 `chmod 0700`과 Step 6의 `tmutil addexclusion`이 필수.

> **Migration 후 재로그인 강제**: Step 1.5에서 기존 `~/.config/gws/`를 migrate한 경우, 그 안의 `credentials.enc`는 대개 Keychain 백엔드로 생성된 것이어서 `KEYRING_BACKEND=file`에서 **재해독 불가 → `auth_method: none`으로 나타남**. `gws auth login`을 반드시 재실행하여 file 백엔드로 refresh token을 다시 발급받는다.

> **에러 핸들링 (401 `invalid_client`)**: `gws auth login`이 401 `invalid_client` 또는 "No OAuth client configured" 로 실패하면 client_secret.json의 OAuth client가 GCP Console에서 삭제/disabled 됐을 가능성이 크다. 다음 순서로 복구:
>
> 1. `https://console.cloud.google.com/apis/credentials?project=<project>` 에서 client 존재/활성 확인
> 2. OAuth consent screen("대상" 탭) 게시 상태 확인 — Testing이면 현재 유저가 테스터 리스트에 있는지
> 3. 없으면 새 **Desktop** OAuth client 생성 → JSON 다운로드 → `$TARGET/client_secret.json` 덮어쓰기 (`chmod 0600`) → `gws auth login` 재실행

---

## Step 4: verified_email 추출 + 라벨 결정 (D5)

### 4.1 이메일 조회

```bash
EMAIL=$(GOOGLE_WORKSPACE_CLI_CONFIG_DIR="$TARGET" \
         GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
         gws gmail users getProfile --params '{"userId":"me"}' \
         | jq -r '.emailAddress // empty')

if [[ -z "$EMAIL" ]]; then
  echo "❌ 이메일 조회 실패. Step 3 login이 완료되었는지 확인하세요."
  exit 1
fi
```

### 4.2 라벨 매칭 (fallback = `<name>`)

```bash
case "$EMAIL" in
  *@kakaohealthcare.com) LABEL="work" ;;
  *@gmail.com)           LABEL="personal" ;;
  *)                     LABEL="$NAME" ;;  # D5 fallback
esac
```

---

## Step 5: Atomic config.yml write (D8)

`~/.gwh/config.yml` 손상 허용 불가 — partial write 상태는 모든 스킬의 security control을 silently 우회.

### 5.1 yq hard dependency 확인

config.yml write는 중복 엔트리/들여쓰기 실수 없이 atomic해야 하므로 `yq`(mikefarah)를 필수 의존성으로 요구한다:

```bash
command -v yq &>/dev/null || {
  echo "❌ yq가 필요합니다 (mikefarah/yq v4+)."
  echo "   macOS: brew install yq"
  echo "   기타:  https://github.com/mikefarah/yq/#install"
  exit 1
}
```

> **왜 읽기는 awk 폴백 허용하고 쓰기만 yq 필수인가**: 읽기(triage/brief/digest/cal-plan의 Step 0.2)는 config.yml의 평문 구조를 단순 파싱 — awk로 충분하고 오작동해도 "해당 계정 skip" 정도로 degradation. 쓰기는 merge/upsert가 필요하고 실수 시 config.yml 손상 → 모든 스킬이 동시 고장. yq idempotent upsert(`yq -i '.accounts["name"] = {...}'`)가 중복/들여쓰기 모두 안전하게 처리한다.

### 5.2 lockdir + atomic rename

```bash
mkdir -p "$HOME/.gwh"
chmod 0700 "$HOME/.gwh"

LOCK="$HOME/.gwh/config.yml.lock.d"
CONFIG="$HOME/.gwh/config.yml"
TMP="$CONFIG.tmp.$$"

# POSIX mkdir 원자성으로 cross-platform 직렬화 (macOS는 flock 미지원)
# 10초 timeout으로 deadlock 방지
TIMEOUT=10
ELAPSED=0
while ! mkdir "$LOCK" 2>/dev/null; do
  if (( ELAPSED >= TIMEOUT )); then
    echo "❌ config.yml lockdir 획득 실패 ($LOCK). 다른 credential-init이 진행 중이거나 stale lock."
    echo "   stale 판단 후: rmdir '$LOCK' 수동 제거"
    exit 1
  fi
  sleep 0.2
  ELAPSED=$((ELAPSED + 1))
done
# lockdir 자동 정리 (정상/에러 모두)
trap 'rmdir "$LOCK" 2>/dev/null; rm -f "$TMP" 2>/dev/null' EXIT

# 기존 config 읽거나 초기값
if [[ -f "$CONFIG" ]]; then
  cp "$CONFIG" "$TMP"
else
  printf 'version: 1\naccounts:\n' > "$TMP"
fi

# yq idempotent upsert — 동일 $NAME 재등록 시 엔트리 교체(중복 방지)
yq -i ".accounts[\"$NAME\"] = {\"verified_email\": \"$EMAIL\", \"label\": \"$LABEL\", \"config_dir\": \"$TARGET\", \"added_at\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}" "$TMP"

chmod 0600 "$TMP"
mv "$TMP" "$CONFIG"   # atomic rename

if [[ ! -f "$CONFIG" ]]; then
  echo "❌ config.yml write 실패"
  exit 1
fi
```

> **왜 flock 대신 mkdir lockdir**: `flock(1)`은 Linux util-linux 번들. macOS 기본 bash/zsh에는 미설치(brew install flock 필요). `mkdir`는 POSIX 필수 커맨드이고 원자적으로 실패하므로 cross-platform 직렬화에 적합. 10초 timeout + trap EXIT cleanup으로 stale lockdir 위험 제어.

---

## Step 6: Time Machine 제외 + 클라우드 동기화 경로 차단

### 6.1 cloud-parent 감지 (REFUSE 조건)

```bash
REAL=$(realpath "$HOME/.gwh")
for cloud in \
  "$HOME/Library/Mobile Documents" \
  "$HOME/Library/CloudStorage" \
  "$HOME/Dropbox" \
  "$HOME/OneDrive"
do
  case "$REAL" in
    "$cloud"*)
      echo "⛔ ~/.gwh 이 $cloud 하위에 있습니다."
      echo "   암호화 키 · 메일 본문 캐시가 클라우드로 동기화될 위험."
      echo "   로컬 경로(예: ~/.gwh)로 이동한 후 재실행하세요."
      exit 1
      ;;
  esac
done
```

### 6.2 Time Machine 제외 (idempotent)

```bash
if command -v tmutil &>/dev/null; then
  # -p(path-based)는 sudo 필요 → user-level(xattr 기반)로 먼저 시도
  tmutil addexclusion "$HOME/.gwh" 2>/dev/null || true
  # 확인
  if ! tmutil isexcluded "$HOME/.gwh" 2>&1 | grep -q 'Excluded'; then
    echo "⚠️  tmutil addexclusion 실패 — xattr fallback 시도"
    xattr -w com.apple.metadata:com_apple_backup_excludeItem 'com.apple.backupd' \
      "$HOME/.gwh" 2>/dev/null || \
      echo "⚠️  xattr fallback도 실패. 수동 실행 필요: sudo tmutil addexclusion -p '$HOME/.gwh'"
  fi
fi
```

> **왜 `-p` 플래그 제거**: `tmutil addexclusion -p` (sticky path-based)는 root 권한 필요 — 일반 사용자 실행 시 silently 무시되어 `[Included]` 상태로 남는다. `tmutil addexclusion` (no flag)은 `com.apple.metadata:com_apple_backup_excludeItem` xattr을 디렉토리에 붙이는 **user-level exclusion**으로 sudo 불필요. path-based보다 약하지만(디렉토리가 이동되면 해제) `~/.gwh`는 고정 경로라 실무 영향 없음.

---

## Step 7: 완료 안내

```markdown
✅ 계정 '{NAME}' 등록 완료

- Email: {EMAIL}
- Label: {LABEL}
- Config: ~/.config/gws-{NAME}/ (0700)
- Keyring: file backend (Keychain 대비 보안 하락 — `~/.gwh` 0700 + tmutil 제외로 보완)
- Registry: ~/.gwh/config.yml (0600)

다음 단계:
- 추가 계정: `/gwh:credential-init <다른-name>`  (권장: `work` 없으면 `work`, 있으면 `personal`)
- 트리아지: `/gwh:triage` — 등록된 모든 계정을 병렬 fetch
- 캘린더: `/gwh:cal-plan --account {NAME}` — 명시적 계정 지정 필수
```

---

## Verification Checklist (PR 머지 전 사용자 수동 실행)

- [ ] `ls -ld ~/.config/gws-work` → `drwx------` (0700)
- [ ] `stat -f '%p' ~/.gwh/config.yml` → `100600`
- [ ] `~/.config/gws-<name>/credentials.enc` + 암호화 키 파일 모두 디렉토리 내부
- [ ] `tmutil isexcluded ~/.gwh` → `Excluded` 포함 (user-level xattr 기반이므로 `TRUE/FALSE` 대신 `Excluded`/`Included` 문자열 확인)
- [ ] `realpath ~/.gwh`가 known sync root 밖
- [ ] Migration 후 `GOOGLE_WORKSPACE_CLI_CONFIG_DIR=~/.config/gws-<name> GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file gws auth status | jq -r .user` 결과가 config.yml의 `verified_email`과 일치
