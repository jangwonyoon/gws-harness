---
title: "feat: gws-harness 2계정 지원 (work + personal)"
type: feat
status: active
date: 2026-04-17
deepened: 2026-04-17
origin: docs/brainstorms/2026-04-17-gws-harness-multi-account-requirements.md
---

# feat: gws-harness 2계정 지원 (work + personal)

## Overview

gws-harness 플러그인에 **work + personal 2계정** 지원을 추가한다. gws-cli 0.22.5의 `GOOGLE_WORKSPACE_CLI_CONFIG_DIR` + `GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file` 환경변수 기반 프로세스 격리로 계정 순회를 구현하고, 기존 4개 읽기 스킬(`triage`/`brief`/`cal-plan`/`digest`)을 merged 뷰 기본으로 확장하며, 신규 `/gwh:credential-init` 스킬로 진입점을 제공한다. Security 기본(0600 권한 + 자동 Time Machine 제외 + atomic config write)을 모든 스킬에 inline 적용한다.

> **v2 재작성 메모**: v1 리뷰에서 지적된 (1) R8 dual-write 규칙 혼선, (2) N-account 구조 YAGNI, (3) Unit 1 speculative 추출, (4) Unit 7 보안 번들 과대, (5) D4 timezone 모호, (6) Migration 3-way 분기 부재, (7) atomic write 편의 위임, (8) 병렬 fetch 템플릿 부재 등을 반영. 8 units → 6 units. Retention/manifest HMAC/audit log/MDM 위협모델은 v1.2로 연기.

## Problem Frame

(see origin: `docs/brainstorms/2026-04-17-gws-harness-multi-account-requirements.md`)

현재 gws-harness는 `~/.config/gws/` 단일 경로만 사용 → 회사 Workspace admin의 OAuth 차단(오늘 발생)에 전면 취약. 사용자는 매일 아침 `/gwh:triage` + `/gwh:brief`, 매주 `/gwh:digest`, 수시 `/gwh:cal-plan` 사용 중이며 work + personal을 한 번에 통합 조회하면서도 회사 메일 본문이 섞인 로컬 캐시의 보안 경계를 명시적으로 관리해야 한다.

## Requirements Trace

Origin R1-R15 전체를 6 units으로 매핑:

- **R1-R3 (통합 실행)**: Unit B/C/D/E — 2계정 순회 + merged 뷰 + 계정 라벨 + 시간대 가중치
- **R4-R6 (계정 등록·식별)**: Unit A — `/gwh:credential-init`
- **R7-R8 (스토리지)**: Glossary + Unit B/C/D/E 공통 (inline chmod 포함)
- **R9 (단일 계정 플래그)**: Unit E만 필수, Unit B/C/D는 선택적(per-account 파일 직접 접근으로 대체 가능)
- **R10 (부분 실패)**: Unit B/C/D = partial merge + 구조화 메타데이터 / Unit E = compute retry + commit fail-closed + `--force` override
- **R11-R12 (MVP)**: 이 플랜 전체 = R11. `/gwh:mail-to-ticket`, AGENT.md 오케스트레이터 멀티계정 = R12(2차)
- **R13 (permissions)**: 각 Unit Step 5 inline `chmod 0600`
- **R14 (retention)**: **v1.2 연기** — MVP에서는 retention 없음. 14일 자동 삭제 패턴이 사용자 메모/미처리 캐시 파괴 위험으로 평가됨. 사용자가 필요 시 `rm ~/.gwh/*-{date}.md` 수동.
- **R15 (enrollment manifest)**: Unit A에서 config.yml에 `verified_email` 기입 + 각 스킬 Step 0에서 단순 비교. HMAC 서명은 v1.2.

## Scope Boundaries

- **2계정 하드코드** — N-account 구조 지원은 명시적 비목표. 3번째 계정 발생 시 별도 minor 작업.
- `/gwh:mail-to-ticket` 멀티계정 = 2차 (Jira/Notion 라우팅)
- AGENT.md 오케스트레이터 멀티계정 인식 = 2차 (라우팅 테이블만)
- Cal-plan **쓰기** 자동 라우팅 없음 — `--account <X>` 명시 필수
- 계정별 Gmail label/Calendar ID fine-grained 필터 = MVP 제외
- **v1.2 연기 (명시적 비목표 in MVP)**: 14일 retention 자동 tidy, enrollment manifest HMAC 서명, audit log, MDM/backup 위협모델 문서화

## Terminology Glossary

- **merged 파일** = 모든 활성 계정의 항목을 하나의 출력 파일에 담은 것. 경로는 `~/.gwh/{skill}-{date}.md`. 계정이 2개면 각 항목에 `[🏢 work]`/`[👤 personal]` 라벨 prefix, 1개면 prefix 생략(R8 호환).
- **per-account 파일** = 계정별 개별 파일. 경로는 `~/.gwh/{skill}-<name>-{date}.md`. 라벨 prefix 없음(이미 계정 확정).
- **dual write** = 읽기 스킬(triage/brief/digest)의 저장 정책 — 2계정 활성 시 per-account 2개 + merged 1개 모두 작성, 1계정 시 merged만. `--account <X>` 플래그는 per-account만 갱신.
- **single-target write** = cal-plan의 저장 정책 — per-account 파일만 작성(merged 경로 미갱신). 이벤트는 `--account <X>`로 명시된 단일 캘린더에만 insert.

## Context & Research

### Relevant Code and Patterns

**5개 기존 스킬 공통 패턴**:
- Step 0(사전 확인) → Step 1(fetch) → Step 2(분류/가공) → ... → Step N(저장) 구조
- frontmatter: `name: gwh-<short>`, `triggers:` 한/영 병기, `dependencies: []`
- 모든 gws 호출 `--format json` 통일 (`gws gmail +triage` 등 helper에만 해당 — `gws auth status`는 플래그 없이 기본 JSON)

**Triple 의존 엣지** (중요):
- `brief` Step 0.3/1: `cat ~/.gwh/triage-{date}.md` → `🔴/🟡` 파싱(358배 캐시 효과)
- `mail-to-ticket` Step 0.2: 동일 triage 캐시 파싱 → AskUserQuestion
- → R8 "단일 계정 시 라벨 생략"의 핵심 이유

**산출물 관리 단일 소스**:
- `AGENT.md` 113-124줄 표 — 모든 스킬 참조. Unit F에서 갱신.

### Institutional Learnings

- `docs/solutions/` 디렉토리 부재 — 이번이 `/ce:compound` 첫 기록 후보 (CONFIG_DIR 격리 패턴, Keychain=file 전환, atomic config write)
- `validation_plan.md` Phase C1: 358배 캐시 재사용 실측 → R8 하위호환 정당성
- gws-cli 0.22.5 ENVIRONMENT: `GOOGLE_WORKSPACE_CLI_CONFIG_DIR`, `GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND` (세션 내 실측 확인)
- `gws auth status` = 플래그 없이 기본 JSON 출력, stderr에 `Using keyring backend: keyring` 배너, `.user` 필드 존재(인증 상태일 때)

### External References

외부 리서치 스킵 — local patterns 강함 + gws env 실측 완료.

## Key Technical Decisions

### D1. 환경변수 기반 프로세스 격리

**결정**: 각 스킬이 `GOOGLE_WORKSPACE_CLI_CONFIG_DIR=~/.config/gws-<name> GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file gws <command>` 형태로 env 주입. 심볼릭 링크·rename 스왑 불필요.

**근거**: 세션 실측 확인(`gws --help | grep CONFIG_DIR`). 프로세스 격리 자연 성립.

### D2. 2계정 하드코드 병렬 fetch

**결정**: `(env1 gws ... &); (env2 gws ...); wait` 2-branch fork. `xargs -P`/`--sequential` 플래그 없음.

**근거**: MVP 스코프 2계정 고정(Scope Boundary). rate limit은 2계정에서 안전 범위. 3번째 계정 발생 시 별도 작업에서 일반화.

### D3. 정렬 규칙 = 긴급도 → (가중치 반영) 시간 → 계정

**결정**: 🔴→🟡→🟢→⚪ 계층 우선, 같은 계층 내 시간 최신 순, tiebreak로 계정 이름 알파벳.

### D4. R3 시간대 가중치 — `TZ=Asia/Seoul` 강제

**결정**:
- **타임존은 시스템 TZ와 무관하게 `TZ=Asia/Seoul date +%u%H`로 강제**
- 업무시간 = 월-금(`%u` 1-5) 09-18 → work 항목 1단계 상향 (🟡→🔴)
- 저녁·주말 → personal 항목 1단계 상향
- 🟢/⚪ 상향 없음 (노이즈 방지)
- 공휴일 미지원(MVP 제외)

**근거**: 해외 출장 등 시스템 TZ ≠ KST 케이스에서도 "KHC 업무시간" 일관 적용.

### D5. R6 라벨 추론 실패 fallback = credential dir `<name>`

**결정**: 도메인 매칭 실패 시 `<name>` 그대로 라벨로 사용. config.yml에서 수동 override 가능.

### D6. Migration 3-way 분기

**결정**: `/gwh:credential-init <name>` 실행 시 다음 분기 (상세 Unit A Step 1 참조):

1. **기존 `~/.config/gws/` 없음**: 신규 setup만
2. **기존 있음, 추론 target == command `<name>`**: 단일 AskUserQuestion 승인 → migrate
3. **기존 있음, 추론 target ≠ command `<name>`**: AskUserQuestion 2지선다 (추론 `<inferred>`로 migrate / command `<name>`로 override)
4. **Target dir `~/.config/gws-<target>/` 이미 존재**: AskUserQuestion 3지선다 (`.bak-{ts}`로 백업 후 migrate / 기존 dir 그대로 활용 / 취소)

추론은 `gws gmail users getProfile --params '{"userId":"me"}'`의 `.emailAddress` → 도메인 매칭.

### D7. 공유 helper 전략 = **inline + 한 번의 extract 뒤**

**결정**: Unit B(triage)를 레퍼런스로 삼아 env 주입 + 2-branch 병렬 fetch + chmod 패턴을 **인라인 구현**. Unit C/D/E는 triage 패턴 복사. 3번째 복사가 실제로 드러난 중복을 만들면 Unit F 또는 후속 작업에서 `skills/shared/references/multi-account.md`로 extract.

**근거**: v1 Unit 1(references 선행 생성)은 speculative. 실행 시 명확해진 중복만 추출하는 것이 Claude Code 플러그인 convention과 YAGNI에 부합.

### D8. Atomic config.yml write 의무화

**결정**: `~/.gwh/config.yml` 쓰기는 **항상 `write tmp → chmod 0600 → rename()` 패턴**. `flock ~/.gwh/config.yml.lock`으로 동시 `credential-init` 직렬화. "편의 선택" 아님.

**근거**: R15 enrollment manifest가 config.yml에 의존 → partial write 상태 허용 시 모든 스킬이 security control을 silently 우회.

### D9. Plugin version bump = minor (1.0.0 → 1.1.0)

**결정**: merged 경로 유지 + auto-rename migration + 1.0.0 사용자가 API 호환 유지 → minor. Breaking change 없음(D8 보장).

**주의**: 외부 스크립트가 `~/.config/gws/` 경로를 하드코딩 참조하는 경우는 breaking. README에 upgrade note로 명시(Unit F).

## Open Questions

### Resolved During Planning

- **병렬 vs 순차 fetch**: 병렬 2-branch 하드코드(D2)
- **정렬 규칙**: 긴급도 → 시간 → 계정(D3)
- **시간대 가중치 세부**: `TZ=Asia/Seoul` 강제 + 평일 09-18 + ±1단계 + 🟢/⚪ 상향 없음(D4)
- **라벨 fallback**: credential dir `<name>`(D5)
- **Migration 분기**: 3-way(D6)
- **공유 helper**: Unit B 먼저 구현 후 extract(D7)
- **Atomic write**: 의무(D8)

### Deferred to Implementation

- OAuth flow 중 사용자 취소 시 rollback — gws-cli exit code 실측 후 Unit A 구현 중 결정
- `~/.gwh/ticket-proposal-*.md` 비규약 파일 처리 — Unit F에서 naming 규약 통일 결정(실제 파일 전수조사 후)
- 병렬 fetch 시 terminal stdout interleaving — trap cleanup + `/tmp/gwh-<account>-$$.json` 격리(D2 템플릿)

### Deferred to v1.2 (명시적 MVP 제외)

- R14 14일 retention 자동 tidy — 사용자 메모·미읽음 캐시 파괴 risk. MVP에서는 수동 `rm`
- R15 manifest HMAC 서명 — 현 scope 공격 모델(user-level config.yml tampering) 과대. verified_email vs `.user` 단순 비교만.
- Audit log (`~/.gwh/audit.log` JSONL) — 개인 단일 사용자 환경 필요성 불명확
- MDM/backup agent 위협모델 문서화 — corporate policy 조정이 더 빠른 경로
- R9 `--account <X>` 플래그를 읽기 스킬에도 적용 — cal-plan 외에는 per-account 파일 직접 cat으로 대체 가능. v1.2에서 실제 수요 확인 후 추가 여부 결정.

## High-Level Technical Design

> *This illustrates the intended approach and is directional guidance for review, not implementation specification.*

**읽기 스킬 (triage/brief/digest) 공통**:

```
[Skill Entry]
  ▼
[Step 0: 사전 확인]
  ├─ which gws + gws auth status (env 없이, 기존 패턴)
  ├─ cat ~/.gwh/config.yml → accounts: 목록 파싱
  └─ 각 <name>에 대해:
       env-injected gws auth status → .user 추출
       .user 가 config.yml accounts[name].verified_email 과 일치하는지 검증
       불일치 → 해당 계정 skip + ⚠️ 배너
     → ACCOUNTS 배열 확정
  │
  ▼
[Step 1: 2-branch 병렬 fetch]
  # Unit B 레퍼런스 구현, Unit C/D는 복사
  for a in "${ACCOUNTS[@]}"; do
    (GOOGLE_WORKSPACE_CLI_CONFIG_DIR=~/.config/gws-${a} \
     GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file \
     gws gmail +triage ... --format json > /tmp/gwh-${a}-$$.json 2>/tmp/gwh-${a}-$$.err
     echo "${a}:$?" > /tmp/gwh-${a}-$$.exit) &
  done
  wait
  trap 'rm -f /tmp/gwh-*-$$.{json,err,exit}' EXIT
  │
  ▼
[Step 2: 분류 + 시간대 가중치]
  KST=$(TZ=Asia/Seoul date +%u%H%M)   # e.g., 41030 = Thu 10:30
  # KST 1-5 * 0900-1759 → work 상향
  # KST 6-7 OR 1800-2359/0000-0859 → personal 상향
  # 🟢/⚪ 은 상향 없음
  │
  ▼
[Step 4: 출력 포맷]
  |ACCOUNTS| == 1 → 기존 포맷(라벨 없음)
  |ACCOUNTS| >= 2 → 각 아이템에 [🏢 work] / [👤 personal] prefix
  Merged 파일 상단 frontmatter:
    ---
    accounts_success: [work, personal]
    accounts_failed: []
    generated_at: 2026-04-17T09:30:00+09:00
    ---
  │
  ▼
[Step 5: dual write]
  mkdir -p ~/.gwh && chmod 0700 ~/.gwh
  for a in "${ACCOUNTS[@]}"; do
    write ~/.gwh/{skill}-${a}-{date}.md
    chmod 0600 ~/.gwh/{skill}-${a}-{date}.md
  done
  if [[ ${#ACCOUNTS[@]} -ge 2 ]]; then
    write ~/.gwh/{skill}-{date}.md   # merged
    chmod 0600 ...
  elif [[ ${#ACCOUNTS[@]} -eq 1 ]]; then
    write ~/.gwh/{skill}-{date}.md   # merged = 기존 포맷(라벨 없음)
    chmod 0600 ...
  fi
  # .sync-excluded 마커 없으면 생성
```

**cal-plan 예외 (Unit E)**:

```
[Step 3: 2-branch 병렬 fetch]
  # 실패 시 1회 auto-retry (exponential 2s)
  # 재실패 시:
  #   사용자가 --force --account <X> 명시했으면 진행(이벤트 설명에 [UNVERIFIED: <failed>] 스탬프)
  #   아니면 ⛔ 중단 + 재시도 안내
[Step 4: 빈 시간 계산]
  ACCOUNTS 캘린더 union → free slots
[Step 5: 쓰기 승인]
  --account <X> 없으면 AskUserQuestion 으로 선택 강제(work / personal)
[Step 6: 단일 계정 write]
  env-inject <X> → gws calendar +insert
[Step 7: single-target write]
  ~/.gwh/cal-plan-<X>-{date}.md (per-account only, merged 경로 미갱신)
```

**`/gwh:credential-init <name>` 흐름**:

```
[0] <name> 검증: ^[a-z0-9][a-z0-9-]*$ (leading hyphen 금지)
    reserved: config, shared, .., .  
    test ! -L ~/.config/gws-<name> && test ! -e ~/.config/gws-<name>
    macOS case-insensitive 충돌 체크 (lowercase normalize → ls -i)
[1] Migration 3-way 분기 (D6)
    ├─ ~/.config/gws/ 없음 → 신규
    ├─ 있음 + 추론 == <name> → 단일 승인 AskUserQuestion
    ├─ 있음 + 추론 != <name> → 2지선다 AskUserQuestion
    └─ target ~/.config/gws-<target>/ 존재 → 3지선다
    stat -f '%d' ~/.config/gws ~/.config 같은 device 확인(mv atomic 보장)
    trap: mv 실패 시 secure-unlink (rm -P)
[2] mkdir -p ~/.config/gws-<name>; chmod 0700
[3] gws auth setup(gcloud 필요) + gws auth login
    gcloud 미설치 감지 → setup 스킵하고 README 안내
    기존 ~/.config/gws-*/client_secret.json 있으면 cp로 재사용(동일 GCP project 전제)
[4] env-inject gws gmail users getProfile --params '{"userId":"me"}' → .emailAddress 추출
    도메인 매칭(kakaohealthcare.com → work, gmail.com → personal)
    실패 시 <name> 그대로 라벨
[5] Atomic config.yml write (D8):
    flock ~/.gwh/config.yml.lock
    write ~/.gwh/config.yml.tmp.$$
    chmod 0600 ~/.gwh/config.yml.tmp.$$
    rename() → ~/.gwh/config.yml
[6] tmutil addexclusion ~/.gwh (macOS, 없으면 추가; 이미 있으면 noop)
    realpath ~/.gwh 결과가 ~/Library/Mobile Documents / ~/Dropbox / ~/Library/CloudStorage / ~/OneDrive 하위면 REFUSE
[7] 완료 안내: 다음 단계 (추가 계정 or /gwh:triage)
```

## Implementation Units

- [ ] **Unit A: `/gwh:credential-init` 신규 스킬 (Migration 3-way + Security 초기화)**

**Goal:** 사용자가 계정 등록 + 기존 `~/.config/gws/` auto-rename + Security 기본(config.yml atomic write, tmutil addexclusion, cloud-parent 감지) 한 번에 수행.

**Requirements:** R4, R6, R11, R13(config.yml 0600), R15(verified_email), D6, D8

**Dependencies:** 없음 (레퍼런스 구현)

**Files:**
- Create: `skills/credential-init/SKILL.md`

**Approach:**
- frontmatter `name: gwh-credential-init`, triggers `"계정 추가"`, `"credential init"`, `/gwh:credential-init`
- Step 0 `<name>` 검증 (정규식 + reserved + symlink + case-insensitive)
- Step 1 Migration 3-way 분기 (D6)
  - `stat -f '%d' ~/.config/gws ~/.config` same-device 검증
  - `trap 'secure_cleanup' ERR` — 실패 시 target dir `rm -P` + source 복원
  - 추론은 `gws auth status` (환경변수 없음, 기존 경로) → `.user` 추출. 추론 target 계산 후 command `<name>`과 비교하여 2지/3지 AskUserQuestion
- Step 2 `mkdir -p ~/.config/gws-<name>; chmod 0700`
- Step 3 gcloud detect + setup/login (gcloud 미설치 시 setup 스킵 + client_secret.json 재사용 권고)
- Step 4 verified_email 추출 via `gws gmail users getProfile` (env-injected). 도메인 매칭 (D5)
- Step 5 Atomic config.yml write (D8): `flock` + tmp → rename, chmod 0600
- Step 6 `tmutil addexclusion ~/.gwh` (macOS, idempotent) + `realpath`로 cloud-parent 감지 → 실패 시 경고 및 refuse
- Step 7 다음 단계 안내

**Patterns to follow:**
- `skills/triage/SKILL.md` Step 0 패턴
- `skills/mail-to-ticket/SKILL.md` Step 3 AskUserQuestion 형식 (3지선다 활용)

**Test scenarios:**
- Happy path: 기존 gws/ 있음(work) + `credential-init work` → 추론 일치 → 단일 승인 migrate → config.yml work 엔트리
- Happy path: 위 이후 `credential-init personal` → 신규 OAuth → config.yml personal 엔트리 추가
- Happy path: 2계정 등록 후 즉시 `/gwh:triage` 실행 → merged 뷰 성공
- Edge case: 기존 gws/가 실제로는 personal email → 추론 `personal`, command `work` → 2지선다 AskUserQuestion → 사용자 'override'로 진행 → `work` 라벨로 migrate
- Edge case: `~/.config/gws-work/` 이미 존재 → 3지선다 AskUserQuestion → `.bak-{ts}` 선택 → 기존 dir 백업 후 신규 migrate
- Edge case: `<name>` = `-foo` (leading hyphen) → 거부
- Edge case: `<name>` = `Work` + macOS case-insensitive → 기존 `work`와 충돌 경고 → lowercase 안내
- Edge case: `~/.gwh`가 `~/Library/Mobile Documents/com~apple~CloudDocs/gwh` 심볼릭 → REFUSE + 사용자에게 로컬 경로로 이동 안내
- Error path: cross-volume mv (예: 외부 SSD의 `~/.config` 링크) → `stat -f '%d'` 불일치 감지 → abort + 수동 안내
- Error path: OAuth flow 사용자 중단 (SIGINT) → trap handler가 `~/.config/gws-<name>/` 제거
- Error path: `gws auth setup`에 gcloud 없음 → setup 스킵 + README 링크 안내, `gws auth login`만 수행(기존 client_secret.json 재사용)
- Error path: flock 획득 실패 (다른 credential-init 진행 중) → "다른 세팅이 진행 중입니다" 메시지 후 exit
- Integration scenario: migration 후 `gws gmail users messages list` (env-inject) 정상 동작 확인 (keyring=file 전환 검증)

**Verification:**
- `~/.config/gws-work/` 존재, `credentials.enc` + encryption key file 모두 dir 내부
- `~/.gwh/config.yml` 권한 0600, `accounts:` 섹션에 `verified_email` 기입
- `tmutil isexcluded ~/.gwh` → true
- Cloud-parent 감지: `realpath ~/.gwh`가 known sync root 밖
- Migration 후 `gws auth status`(env-inject) → `.user`가 config.yml verified_email과 일치

---

- [ ] **Unit B: `/gwh:triage` 2계정 확장 (공유 패턴 레퍼런스)**

**Goal:** 2계정 병렬 fetch + merged 뷰 + 시간대 가중치 + partial merge 메타데이터. Unit C/D가 복사할 레퍼런스 구현.

**Requirements:** R1, R2, R3, R5, R7, R8 (라벨 조건부), R10 (partial + metadata), R13 (chmod inline), D2, D4, D7

**Dependencies:** Unit A (config.yml accounts: 읽어야 함)

**Files:**
- Modify: `skills/triage/SKILL.md`

**Approach:**
- Step 0 유지 + **Step 0.4 (신규)**:
  - `cat ~/.gwh/config.yml` 파싱 → accounts 목록
  - 각 계정에 env-inject `gws auth status` → `.user` 추출
  - config.yml `accounts[name].verified_email` 과 비교, 불일치 skip + 상단 배너
  - ACCOUNTS 배열 확정
- Step 1 2-branch 병렬 fetch (D2 템플릿 inline):
  - `(GOOGLE_WORKSPACE_CLI_CONFIG_DIR=~/.config/gws-work GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file gws gmail +triage --max 50 --query 'is:unread' --format json > /tmp/gwh-triage-work-$$.json 2>/tmp/gwh-triage-work-$$.err; echo "work:$?" > /tmp/gwh-triage-work-$$.exit) &`
  - personal 동일 블록
  - `wait`
  - `trap 'rm -f /tmp/gwh-triage-*-$$.{json,err,exit}' EXIT`
- Step 2 분류 + 시간대 가중치 (D4):
  - `KST=$(TZ=Asia/Seoul date +%u%H%M)` — `%u` 1-5 = 평일, `%H%M` 0900-1759 = 업무시간
  - work 항목: 업무시간이면 `🟡→🔴` 상향, 🟢/⚪ 상향 없음
  - personal 항목: 업무시간 밖이면 `🟡→🔴` 상향
- Step 3 액션 아이템 추출 (기존)
- Step 4 R8 조건부 라벨 prefix (Glossary 참조)
- Step 5 dual write + chmod:
  - `|ACCOUNTS| == 1`: `~/.gwh/triage-{date}.md` (merged 경로, 라벨 없음)
  - `|ACCOUNTS| >= 2`: per-account 2개 + merged 1개 (라벨 포함)
  - Merged 파일 상단 frontmatter: `accounts_success: [...], accounts_failed: [...], generated_at: ...`
  - 각 저장 후 `chmod 0600`, `~/.gwh` 디렉토리 `chmod 0700`
  - `~/.gwh/.sync-excluded` 없으면 생성

**Patterns to follow:**
- 기존 `skills/triage/SKILL.md` Step 1-5 뼈대 유지
- frontmatter 시간대 가중치 예시 텍스트는 D4 문구 인용

**Test scenarios:**
- Happy path (단일 계정 환경): 기존 사용자 업그레이드 직후 → 라벨 없음, 경로 `~/.gwh/triage-{date}.md`, frontmatter에 `accounts_success: [work]` 포함
- Happy path (2계정): merged + per-account 둘 다 생성. 각 라인에 라벨 prefix 확인
- Edge case (시간대 가중치 KST 10:30 평일): work `🟡` 메일이 `🔴` 상향. `KST=$(TZ=Asia/Seoul date +%u%H%M)` 실제 호출 검증
- Edge case (KST 22:00): work 상향 없음, personal `🟡`이 `🔴`으로 상향
- Edge case (KST 토요일 14:00): personal 상향, work 상향 없음
- Edge case (enrollment mismatch): config.yml work verified_email 가짜 수동 편집 → work skip, `⚠️ work: enrollment 불일치` 배너, merged는 personal만
- Error path (work credential invalid + personal 정상): partial merge, per-account file은 personal만 생성, merged frontmatter `accounts_failed: [{name: work, reason: 인증 실패}]` + 상단 배너 1줄
- Error path (양쪽 다 실패): 빈 merged + frontmatter로 실패 내역 기록, exit code 1
- Integration scenario: triage 완료 후 `/gwh:brief` 실행 → merged 파일 cat + 라벨 포함 포맷 파싱 정상 동작

**Verification:**
- `stat -f '%p' ~/.gwh/triage-*.md | head -1` → `100600`
- 단일 계정 시 기존 포맷 파일 diff (라벨 prefix 없음 확인)
- 2계정 시 merged frontmatter 파싱 가능 (`yq` 또는 jq)
- 시간대 가중치: Asia/Seoul 고정 확인을 위해 `TZ=America/Los_Angeles gws gmail` 환경에서도 상향 로직이 서울 시간 기준으로 동작

---

- [ ] **Unit C: `/gwh:brief` 2계정 확장 (triage 캐시 소비 + drift 감지)**

**Goal:** brief가 merged triage 캐시 재사용(358배 성능) + 캐시 미스 시 2계정 fetch fallback + 단일→멀티 같은날 format drift 감지.

**Requirements:** R1, R2, R7, R8, R10

**Dependencies:** Unit A, Unit B (triage merged 포맷 표준 참조)

**Files:**
- Modify: `skills/brief/SKILL.md`

**Approach:**
- Step 0.3 `cat ~/.gwh/triage-{date}.md` 유지 (성능 유지)
- Merged 파일 상단 frontmatter 파싱 → `accounts_success` 개수 확인
- **Drift 감지**: 현재 ACCOUNTS와 frontmatter `accounts_success`가 다르면 상단에 `⚠️ triage 캐시가 {N}개 계정으로 생성됨. 현재 ACCOUNTS: {M}개` 배너 + 재실행 권유
- Step 1 캐시 미스 시: Unit B Step 1 병렬 패턴 복사 (D7: 3번째 복사 시 extract 재검토)
- Step 2 캘린더도 2-branch 병렬 fetch
- Step 3 출력 포맷 기존 유지 + 필요 시 라벨 prefix
- Step 4 dual write + chmod (Unit B 패턴 복사)

**Patterns to follow:**
- Unit B Step 0.4/1/5 패턴 복사
- 기존 brief Step 0.3 성능 로직 유지

**Test scenarios:**
- Happy path (캐시 히트): 2계정 triage 선행 + brief → gws API 호출 0회, 4ms±δ 지연
- Happy path (캐시 미스): triage 없이 brief → 2계정 병렬 fetch, merged 브리핑
- Edge case (drift): 단일 계정 triage 생성 후 personal 추가하고 brief 실행 → drift 배너 + 재실행 권유
- Error path (한 계정 calendar 만료): partial merge + 배너
- Integration scenario: brief 캐시 히트 경로에서 merged triage의 라벨 prefix 포함 포맷 파싱 시 요약 품질 유지

**Verification:**
- 캐시 히트 지연 4ms±δ
- Drift 감지가 `accounts_success` 기반으로 정확히 동작
- 2계정 fallback 병렬 실행이 순차 대비 ~2배 빠름(실측)

---

- [ ] **Unit D: `/gwh:digest` 2계정 확장**

**Goal:** 주간 송수신 집계 N(=2)계정 병렬 fetch. Jira는 계정 중립.

**Requirements:** R1, R2, R7, R10

**Dependencies:** Unit A, Unit B (fetch 패턴 복사)

**Files:**
- Modify: `skills/digest/SKILL.md`

**Approach:**
- Step 0.4 Unit B 패턴 복사 (ACCOUNTS 확정)
- Step 1/2 송수신 쿼리 각각 2-branch 병렬 fetch
- Step 3 Jira는 `assignee = currentUser()` 단일 쿼리 — **계정 중립 명시** ("🏃 스프린트 (계정 통합)" 섹션 헤더)
- Step 4 집계 + 라벨링 (R8 조건부)
- Step 5 dual write: `~/.gwh/digest-{YYYY}-W{WW}-<account>.md` + `~/.gwh/digest-{YYYY}-W{WW}.md`
- Step 6 chmod 0600 + sync-excluded 확인

**Patterns to follow:**
- Unit B 패턴 전체 복사
- Jira 중립 섹션은 기존 digest 구조 재활용

**Test scenarios:**
- Happy path (2계정): 각 계정 송수신 통계 + Jira 공유 섹션
- Happy path (단일 계정): 기존 포맷 유지
- Edge case (한 계정 송신 0건): "송신 0건" 명시 (누락이 아닌 0 값)
- Error path (한 계정 전체 실패): partial merge + frontmatter 실패 기록
- Integration scenario: 매주 실행 시 같은 주차 파일 overwrite 정책 확인

**Verification:**
- 2계정 섹션 모두 등장 + Jira 중립 섹션 1개
- 파일 권한 0600

---

- [ ] **Unit E: `/gwh:cal-plan` 2계정 확장 (retry + fail-closed + override)**

**Goal:** 읽기는 2계정 병렬 fetch + 1회 retry + 재실패 시 `--force --account <X>` override 경로. 쓰기는 single-target (per-account 파일만, merged 미갱신).

**Requirements:** R1, R2, R7, R8, R10 (fail-closed 변형)

**Dependencies:** Unit A, Unit B (fetch 패턴)

**Files:**
- Modify: `skills/cal-plan/SKILL.md`

**Approach:**
- Step 0.4 Unit B 패턴 복사
- Step 3 (week fetch): 2-branch 병렬 fetch
  - 한 계정이라도 실패 시 **1회 auto-retry** (exponential 2s)
  - 재실패 시:
    - `--force --account <X>` 있으면 **override path**: 성공 계정만으로 계산 진행, 이벤트 설명란에 `[UNVERIFIED: {failed-account} 일정 확인 불가]` 스탬프
    - `--force` 없으면 ⛔ 중단 + override 안내 (`/gwh:cal-plan --force --account work`)
- Step 4 빈 시간 계산: 성공 계정 캘린더 union
- Step 5 쓰기 승인: `--account <X>` 없으면 AskUserQuestion (work/personal 단일 선택)
- Step 6 env-inject 대상 계정 → `gws calendar +insert`
- Step 7 single-target write: `~/.gwh/cal-plan-<X>-{date}.md` (per-account only), merged 미갱신
- chmod 0600 + sync-excluded 확인

**Patterns to follow:**
- Unit B Step 0.4/1 패턴
- 기존 cal-plan Step 3-7 골조 유지

**Test scenarios:**
- Happy path (2계정 읽기 + `--account work` 쓰기): 빈 시간 = work+personal union, 이벤트는 work 캘린더만
- Happy path (`--account` 미지정): AskUserQuestion 단일 선택 후 진행
- Edge case (1계정 환경): 기존 동작 유지, `--account` 없어도 해당 계정에 자동
- Edge case (retry 성공): 첫 fetch 실패 → 2초 backoff → 재fetch 성공 → 정상 진행
- Error path (재실패 + no `--force`): ⛔ 배너 + 재시도 안내, Step 5-6 건너뜀, 저장 없음
- Error path (재실패 + `--force --account work`): work 성공분으로 계산, personal 일정 미반영 경고, 이벤트 설명 `[UNVERIFIED: personal 일정 확인 불가]` 스탬프
- Error path (승인 거부): 사용자가 AskUserQuestion 취소 → Step 6 건너뜀, 읽기 결과만 per-account 파일에 저장
- Integration scenario: personal 일정과 work 스프린트 overlap 케이스 → 빈 시간 계산 정확도

**Verification:**
- `--force` 없이 fetch 재실패 → 이벤트 생성 0건, 명확한 에러 메시지
- `--force` override 시 이벤트 설명란에 `[UNVERIFIED: ...]` 실제 스탬프 확인
- Per-account 파일만 생성, merged cal-plan 경로 미갱신
- 이벤트가 지정된 계정 캘린더에만 존재 (다른 계정 leak 없음)

---

- [ ] **Unit F: Finalization (문서 + AGENT.md + version bump + mail-to-ticket security inline)**

**Goal:** 문서 일관성 + 버전 bump + `/gwh:mail-to-ticket`에도 최소 security(chmod + sync-excluded) 적용. 필요 시 D7 extract 결정.

**Requirements:** R11 (mail-to-ticket security), R13 (mail-to-ticket 포함), D9

**Dependencies:** Unit A-E 완료

**Files:**
- Modify: `AGENT.md` — "산출물 관리" 섹션에 per-account + merged 패턴, Glossary 반영
- Modify: `AGENTS.md` — 새 스킬 추가 절차에 "2계정 패턴 참조" 짧게 추가(향후 contributor 가이드)
- Modify: `README.md` — "계정 세팅" 섹션(3줄 onboarding) + "Security / Privacy" 섹션(`tmutil addexclusion` 자동화 안내 + iCloud/Dropbox 수동 설정 가이드 + 기존 `~/.config/gws` 사용자 auto-rename 설명)
- Modify: `skills/mail-to-ticket/SKILL.md` — Step 5 저장 절에 `chmod 0600` + sync-excluded 확인 추가 (멀티계정 확장은 2차)
- Modify: `.claude-plugin/plugin.json` — `version: 1.0.0 → 1.1.0`
- Modify: `.claude-plugin/marketplace.json` — 같은 bump
- (조건부) Create: `skills/shared/references/multi-account.md` — Unit B/C/D/E 구현 후 실제 중복이 드러나면 extract. 드러나지 않으면 skip.

**Approach:**
- AGENT.md 산출물 표 재작성: per-account 컬럼 + merged 컬럼 + 단일 계정 시 라벨 생략 주석
- README 예제: `/gwh:credential-init work && /gwh:credential-init personal && /gwh:triage` + upgrade note (외부 스크립트가 `~/.config/gws` 하드코딩 시 조정 필요)
- plugin.json 변경 주석 없음(JSON), PR 설명에 이유 기재
- D7 extract 판단: Unit B/C/D/E 완료 후 Step 0.4/Step 1이 **3회 이상 동일 복사**되었는지 점검. 2회라면 현 플랜 유지(inline), 3회+면 `references/multi-account.md` 생성

**Patterns to follow:**
- 기존 AGENT.md 113-124줄 표 포맷
- 기존 README 구조(설치 → 스킬 → 예제)

**Test expectation:** none — 문서/version 변경, 행동 변화 없음.

**Verification:**
- CI (`validate.yml`) frontmatter + markdownlint 통과
- `gh pr create` 후 CI 초록
- README 3줄 onboarding + Security 섹션 + upgrade note 모두 존재
- mail-to-ticket 파일 권한 0600 확인
- D7 extract 결정 근거 PR 설명에 기재

## System-Wide Impact

**Interaction graph:**
- `/gwh:credential-init` → `~/.gwh/config.yml` + `~/.config/gws-<name>/` → 모든 읽기 스킬 Step 0
- `/gwh:triage` merged → `/gwh:brief` Step 0.3 + `/gwh:mail-to-ticket` Step 0.2 파싱 (R8 호환)
- `tmutil addexclusion` (Unit A Step 6) = 1회 실행 이후 idempotent

**Error propagation:**
- 한 계정 auth 실패: 읽기 = partial merge + frontmatter 기록 + 상단 배너 / cal-plan = retry → `--force` override or 중단
- enrollment mismatch: 해당 계정 skip + 배너
- config.yml write 실패(flock 획득 불가, 디스크 풀): credential-init abort, trap이 credential dir도 정리

**State lifecycle risks:**
- Migration 중 SIGINT: trap `rm -P target + 원본 복원`
- Same-day format drift: Unit C의 drift 배너가 감지
- Merged 파일 overwrite: frontmatter `accounts_success` 개수가 많은 쪽 우선(구현 결정) — 동일한 날 re-run 시 첫 실행이 partial이었어도 두 번째가 full이면 full로 덮어쓰기

**API surface parity:**
- `/gwh:mail-to-ticket` 멀티계정 미지원, Unit F에서 chmod + sync-excluded만 적용 (security parity)
- AGENT.md 오케스트레이터는 라우팅 테이블만 유지 (2차)

**Integration coverage:**
- cross-layer 필수 커버: (1) Unit B merged → Unit C 캐시 히트 포맷 호환, (2) Unit E 쓰기 시 work 캘린더에만 insert (leak 없음), (3) Unit A migration 후 즉시 `/gwh:triage` 실행 성공
- Unit B/C/E Test scenarios의 Integration 항목이 이를 커버

**Unchanged invariants:**
- 5개 기존 스킬 사용자-facing trigger 유지
- gws CLI 호출 인자 구조 변경 없음
- `~/.gwh/{skill}-{date}.md` 경로 유지 (R8)
- brief 358배 캐시 성능 유지

## Risks & Dependencies

| Risk | Mitigation |
|------|------------|
| `KEYRING_BACKEND=file`로 암호화 키가 config dir 내부 평문 파일 저장(Keychain 대비 보안 하락) | `/gwh:credential-init` Step 6 "keyring=file 전환 안내" 출력 + dir 0700 / 파일 0600 / tmutil 자동 제외 / README Security 섹션 투명 문서화 |
| Migration `mv` 중 cross-volume 실패 시 키링 평문 파일 두 곳 잔존 | Step 1 `stat -f '%d'` same-device 검증 + trap `rm -P` secure-unlink + 실패 시 config.yml 엔트리 기입 안 함 |
| Migration target dir 이미 존재 → BSD mv가 기존 dir 안으로 source 이동 | 사전 `test -e ~/.config/gws-<target>` → 3지선다 AskUserQuestion (`.bak-{ts}`/기존 활용/취소) |
| 추론 target ≠ command `<name>` 오라벨 위험 | 2지선다 AskUserQuestion (추론/override)로 사용자 확인 |
| `.sync-excluded` 마커는 자동 차단 없음 (문서화 신호) | `tmutil addexclusion ~/.gwh` 자동 실행 + `realpath`로 known cloud parent(iCloud/Dropbox/OneDrive/CloudStorage) 감지 → REFUSE |
| 2계정 Gmail API rate limit | 2계정은 안전 범위, 구현 중 rate limit 감지 시 `--sequential` 플래그 보강 여부 재평가(Deferred to Implementation) |
| 외부 스크립트가 `~/.config/gws/` 경로 하드코딩 시 D6 migration이 silent breakage | README upgrade note에 명시 + `plugin.json` minor bump(1.1.0)로 눈에 띄게 |
| config.yml 손상 시 모든 스킬 fail | Atomic write(D8) + flock 직렬화 + Unit A 실패 시 원자적 rollback |
| OAuth flow SIGINT rollback 미완전 | Unit A Step 1/2의 trap handler로 credential dir 제거 + config.yml 미기입 |

## Documentation / Operational Notes

- **Onboarding (README)**: `/gwh:credential-init work` → `/gwh:credential-init personal` → `/gwh:triage` 3단계
- **Upgrade path**: merged 경로 유지 + auto-migration → 기존 사용자 workflow 무변경. `~/.config/gws/` 외부 참조 시 조정 필요 note
- **Security 가이드 섹션 (README)**: `tmutil addexclusion` 자동화 설명 + iCloud Drive / Dropbox / OneDrive 수동 설정 가이드. "본 디렉토리는 sync 도구로 backup 하지 말 것" 원칙
- **v1.2 Roadmap 힌트**: 14일 retention, enrollment manifest HMAC, audit log, MDM 위협모델 — 실제 사용 중 문제 감지 시 추가
- **Changelog**: 플러그인에 CHANGELOG.md 관례 부재 → PR 설명 + git commit 메시지

## Sources & References

- **Origin document**: [docs/brainstorms/2026-04-17-gws-harness-multi-account-requirements.md](docs/brainstorms/2026-04-17-gws-harness-multi-account-requirements.md)
- Existing skills: `skills/{triage,brief,cal-plan,mail-to-ticket,digest}/SKILL.md`
- Orchestrator: `AGENT.md` (Unit F에서 표 업데이트)
- Contributor guide: `AGENTS.md`
- Plugin manifest: `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`
- gws-cli 0.22.5+ ENVIRONMENT: `GOOGLE_WORKSPACE_CLI_CONFIG_DIR`, `GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND` (세션 내 실측)
- gws helpers: `gws auth status` (플래그 없이 JSON), `gws gmail users getProfile` (verified_email 추출)
- Prior harness audit: `harness-report-2026-04-17.md`
- v1 review synthesis (이 문서 재작성 근거): coherence 11 / feasibility 7 / security 8 / scope-guardian 6 / adversarial 7 = 39 findings → 반영 후 6 units
