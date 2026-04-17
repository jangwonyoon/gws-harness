---
date: 2026-04-17
topic: gws-harness-multi-account
---

# gws-harness 멀티 계정 지원

## Problem Frame

현재 gws-harness는 gws-cli의 `~/.config/gws/` 단일 credential 경로를 그대로 사용하므로, 회사 계정(`diego.yoon@kakaohealthcare.com`)과 개인 계정(`yoonjangwon94@gmail.com`)을 동시에 사용할 수 없다. 회사 Workspace 관리자의 OAuth 앱 차단이 재발하면 업무 전체가 마비되고(오늘 발생), 개인 캘린더·메일을 회사 업무와 분리하고 싶을 때 우회책이 없다.

사용자는 매일 아침 `/gwh:triage`와 `/gwh:brief`, 매주 `/gwh:digest`, 수시로 `/gwh:cal-plan`을 사용하고 있으며, 회사·개인을 **한 번에 통합 조회**하는 경험이 필요하다.

## User Flow

```mermaid
flowchart TB
    A[세팅 1회: gws-credential-init work] --> B[세팅 1회: gws-credential-init personal]
    B --> C{일상 사용}
    C -->|아침| D[/gwh:triage]
    C -->|정오| E[/gwh:cal-plan]
    C -->|금요일| F[/gwh:digest]
    D --> G[내부: 각 계정 순회 fetch]
    E --> G
    F --> G
    G --> H[per-account 캐시 저장]
    H --> I[merged 캐시 저장]
    I --> J[통합 뷰 출력 + 계정 라벨]
```

## Requirements

**통합 실행**
- R1. 각 스킬 실행 시 기본 동작은 등록된 모든 계정을 순회해 merged 뷰를 출력한다.
- R2. Merged 뷰는 각 항목에 계정 라벨(예: `[🏢 work]`, `[👤 personal]`)을 붙여 출처를 명시한다.
- R3. 분류 우선순위(🔴/🟡/🟢/⚪)는 **계정 × 시간대 가중치**로 판정한다. 평일 업무시간(월–금 09–18 Asia/Seoul)에는 work 항목을 한 단계 상향, 저녁(18 이후)·주말에는 personal 항목을 한 단계 상향. 기본(키워드·발신자·수신유형) 분류 위에 컨텍스트 보정을 얹는 방식.

**계정 등록·식별**
- R4. 각 계정별 gws credential directory를 독립 생성하는 `/gwh:credential-init <name>` **슬래시 커맨드 스킬**을 제공한다. 내부 동작: `mkdir -p ~/.config/gws-<name>` → `GOOGLE_WORKSPACE_CLI_CONFIG_DIR=~/.config/gws-<name> GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file gws auth setup && gws auth login` 실행 → `~/.gwh/config.yml` 의 `accounts:` 섹션에 `verified_email` 자동 기입.
- R5. 존재하는 credential directory 목록으로 계정을 자동 감지한다 — 별도 `add` 명령 없이도 동작.
- R6. 계정 라벨은 이메일 도메인으로 기본 추론(`kakaohealthcare.com → work`, `gmail.com → personal`)하고, `~/.gwh/config.yml`에서 수동 재정의 가능.

**스토리지**
- R7. 각 스킬은 실행마다 per-account 캐시(`~/.gwh/{skill}-{account}-{date}.md`)와 merged 캐시(`~/.gwh/{skill}-{date}.md`)를 모두 저장한다.
- R8. 기존 단일 계정 사용자가 참조하던 파일명(`~/.gwh/{skill}-{date}.md`)은 **merged 파일** 경로와 동일하게 유지. 단, **활성 계정이 1개일 때는 계정 라벨 prefix(`[🏢 work]`)를 생략**하여 기존 소비자(brief의 triage-cache 파싱 등)의 포맷 가정과 호환. 2개 이상 or `--account X` 실행 시에만 라벨 prefix 삽입.

**유연성**
- R9. 단일 계정만 실행하는 `--account <name>` 플래그를 지원 — per-account 캐시만 생성·갱신.
- R10. **읽기 스킬(triage/brief/digest)**은 한 계정이 실패해도 나머지는 정상 진행 — partial merge 출력 + 상단에 `⚠️ {account}: {사유}` 표시. **쓰기 스킬(cal-plan)**은 모든 계정 캘린더 fetch 성공 시에만 진행(fail-closed) — 실패 시 `⛔ fetch 실패한 계정: {N}. 일정 충돌 위험으로 중단.` 표시하고 이벤트 생성 중단.

**Security**
- R13. `~/.gwh/` 의 모든 파일 0600 / 디렉토리 0700 권한. `~/.gwh/.sync-excluded` 마커 생성 + README에 iCloud Drive·Dropbox·Time Machine 제외 설정 문서화.
- R14. Cache retention — `~/.gwh/*-{date}.md` 파일은 생성 후 14일 경과 시 자동 삭제(매 실행 시 tidy 단계). Config(`~/.gwh/config.yml`)는 예외.
- R15. Account enrollment manifest — `~/.gwh/config.yml`의 `accounts:` 섹션에 등록된 `verified_email` 필드와 `gws auth status`의 실제 `.user` 필드가 일치하는 credential dir만 사용. 미등록·불일치 디렉토리는 자동 감지에서 제외(신뢰 경계 보호).

**MVP 범위**
- R11. 1차 배포 대상: `/gwh:triage`, `/gwh:brief`, `/gwh:cal-plan`, `/gwh:digest`, `/gwh:credential-init` (onboarding 진입점).
- R12. `/gwh:mail-to-ticket`, `AGENT.md` 오케스트레이터는 2차로 연기.

## Success Criteria

- 아침에 `/gwh:triage` 한 번 실행하면 회사·개인 메일이 하나의 🔴/🟡/🟢/⚪ 뷰에 계정 라벨과 함께 등장한다.
- 회사 Workspace admin 차단이 재발해도 `gws-credential-init personal`만으로 `/gwh:triage --account personal` 동작하여 **일상 워크플로우 중단 없음**.
- 기존 단일 계정 사용자의 `~/.gwh/triage-{date}.md` 소비 코드는 수정 없이 동일하게 동작 (merged 경로 = 기존 경로).
- 새 세팅 시 credential init 2회 + config.yml 수정 없이(도메인 자동 추론으로) `/gwh:triage` 즉시 통합 뷰 출력.

## Scope Boundaries

- 3개 이상 계정 동시 운용은 **명시적 비목표** — 구조는 N개 지원이지만 MVP는 2개만 검증.
- `/gwh:mail-to-ticket`의 계정별 타깃 시스템 라우팅(work→Jira, personal→Notion)은 2차 연기.
- `AGENT.md` 오케스트레이터의 멀티계정 통합은 2차 연기.
- 캘린더 **쓰기** 동작(`cal-plan`의 이벤트 생성) 시 어느 계정에 쓸지는 `--account <name>` 명시 필수 — 자동 라우팅 규칙 없음.
- 계정별 다른 Gmail label/Calendar ID 필터링 같은 fine-grained 설정은 MVP 제외.

## Key Decisions

- **통합 뷰 (merged)**: 사용자가 하루를 "계정별"이 아니라 "중요도별"로 시작해야 한다. 스위치 UX는 마찰이 크고 per-account 스위칭의 일상 가치가 불분명.
- **MVP 4개 스킬**: `mail-to-ticket`은 타깃 시스템 라우팅 복잡도가 별도 스코프. 읽기 중심 4개부터 안정화.
- **자동 감지 + 재정의**: `add` 명령 없이도 `~/.config/gws-*/` 존재로 계정 인식. 라벨이 틀리면 `config.yml`로 정정. 세팅 마찰 최소화.
- **단일 실행 플래그 지원**: per-account 캐시가 어차피 중간 산출물이므로 `--account X` 추가 비용 거의 없음. 부분 실패 partial merge도 같은 구조에서 자연 파생.
- **기존 파일명 하위호환**: merged 경로 = 기존 경로(`~/.gwh/{skill}-{date}.md`)로 두어 brief→triage 캐시 read 등 기존 소비 코드 수정 불필요. 단일 계정 시 라벨 생략(R8)으로 포맷 호환까지 확보.
- **시간대 가중치 priority (R3)**: account-blind 정책은 평일 업무시간에 personal 뉴스레터가 회사 PM 요청 위로 surface되는 failure mode가 분명. 시간·요일 컨텍스트 보정은 MVP에 포함해야 R5 merged view의 '오늘 1순위' 약속이 성립.
- **cal-plan fail-closed**: 읽기 partial은 정보 누락이지만 cal-plan 쓰기 partial은 실제 일정 충돌(affirmative harm). 동일 partial 정책을 universal하게 적용하지 않고 R10에서 읽기/쓰기 분리.
- **Security 명시적 정책 (R13-R15)**: 회사·개인 메일 본문이 섞인 merged cache는 DLP 리스크. Permissions·sync 배제·retention·enrollment manifest 4종 세트로 신뢰 경계 방어선 구축. `~/.config/gws-*` 존재만으로 계정 인식 시 공격 표면 생성 방지.

## Dependencies / Assumptions

- gws-cli 0.22.5+가 `GOOGLE_WORKSPACE_CLI_CONFIG_DIR` 환경변수로 per-account config 디렉토리 지정을 지원함(실측 확인). 각 스킬은 실행 시 env 주입만으로 N 계정 순회 가능 — 심볼릭 링크/디렉토리 rename 같은 스왑 레이어 불필요.
- gws-cli 0.22.5+가 `GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file` 옵션을 지원하여 암호화 키를 config dir 내부 파일로 저장 가능(실측 확인). macOS Keychain(기본 `keyring` 백엔드)은 service 이름이 `gws-cli` 하드코딩이라 계정별 덮어쓰기 발생 — MVP는 `file` 백엔드로 전환해 디렉토리별 자연 격리.
- `gws auth status`(플래그 없이)가 stdout에 JSON을 출력하며 `.user` 필드로 현재 활성 credential의 이메일을 반환함. keyring 안내 메시지는 stderr로 분리됨.

## Alternatives Considered

- **독립 실행 (gws-switch 스크립트)**: `gws-switch work && /gwh:triage` 두 번 실행 → 2개 파일 따로. 제외 사유: 매일 아침 두 번 커맨드 부담 + 통합 우선순위 비교 불가.
- **플래그만, 통합 없음**: `/gwh:triage --account X` 각각 실행. 제외 사유: 통합 뷰 원칙과 상충(플래그는 R9로 흡수).
- **명시적 `/gwh:account add` 커맨드**: 투명도 높지만 세팅 마찰. 자동감지 + 재정의 패턴이 거의 동일 투명도를 더 낮은 비용으로 제공.

## Outstanding Questions

### Resolve Before Planning
(없음)

### Deferred to Planning
- [Affects R1, R10][Technical] 계정 순회 fetch 전략 — 순차 vs 병렬. `GOOGLE_WORKSPACE_CLI_CONFIG_DIR` 기반 프로세스 격리가 성립하면 병렬이 기본값. keyring=file 전제에서 race 없음.
- [Affects R2][Technical] Merged 출력에서 항목 정렬 규칙 — 중요도 > 시간 > 계정? 혹은 중요도 > 계정 > 시간?
- [Affects R6][Needs research] 라벨 자동 추론이 실패하는 경우(custom domain, alias) 기본 라벨 규칙.
- [Affects R3][Technical] 시간대 가중치 구현 세부 — timezone 고정(Asia/Seoul) vs 사용자 로컬 시각, 공휴일 처리, 상향 단계 수(1단계 vs 2단계).

## Next Steps
→ `/ce:plan` for structured implementation planning
