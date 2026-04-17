# gws-harness 효용성 평가

이 디렉토리는 gws-harness의 각 스킬/에이전트를 **독립 Claude Code 세션에서 자율 평가**하기 위한 핸드오프 자료를 담는다.

## 구조

```
eval/
├── README.md                    # 이 파일
├── rubric.md                    # 5차원 평가 루브릭 (D1-D5)
├── handoff-prompt.md            # 새 세션에 붙여넣을 프롬프트 템플릿
├── briefs/                      # 컴포넌트별 평가 브리프
│   ├── gwh-triage.md
│   ├── gwh-brief.md
│   ├── gwh-cal-plan.md
│   ├── gwh-mail-to-ticket.md
│   ├── gwh-digest.md
│   └── agent-harness.md
└── results/                     # 평가 결과 (gitignore)
```

## 사용 흐름

### 1. 새 Claude Code 세션 열기

각 컴포넌트는 **독립 세션**에서 평가한다. 세션이 섞이면 맥락 오염으로 평가 품질 저하.

### 2. 핸드오프 프롬프트 붙여넣기

`eval/handoff-prompt.md`의 "프롬프트 템플릿" 섹션을 복사하여 평가 대상 이름으로 치환 후 붙여넣기.

예시:
```
당신은 gws-harness 플러그인의 `/gwh:triage` 효용성을 평가하는 독립 평가자입니다.
...
`eval/briefs/gwh-triage.md`부터 읽어주세요.
```

### 3. 세션이 자율 수행
- 브리프의 시나리오 실행
- 루브릭(D1-D5) 점수 부여
- `eval/results/{component}-{YYYY-MM-DD}.md`에 결과 저장
- 종합 점수 + 3가지 인사이트 보고

### 4. 결과 취합 (사용자가 여러 세션 완료 후)

```bash
# 총점 분포
grep "^total_score:" eval/results/*.md

# verdict 분포
grep "^verdict:" eval/results/*.md | sort | uniq -c

# 특정 차원의 점수 비교 (예: D5 채택 확률)
grep -A1 "D5" eval/results/*.md | grep -oE "[0-9]/5"
```

## 평가 대상 (6개)

| 컴포넌트 | 브리프 | 성격 |
|----------|--------|------|
| `/gwh:triage` | briefs/gwh-triage.md | 읽기 전용 (안전) |
| `/gwh:brief` | briefs/gwh-brief.md | 읽기 전용 (안전) |
| `/gwh:cal-plan` | briefs/gwh-cal-plan.md | **쓰기 (캘린더 이벤트 생성)** |
| `/gwh:mail-to-ticket` | briefs/gwh-mail-to-ticket.md | **쓰기 (Jira/Notion 생성)** |
| `/gwh:digest` | briefs/gwh-digest.md | 읽기 전용 (안전) |
| `AGENT.md` 하네스 | briefs/agent-harness.md | 오케스트레이터 메타 평가 |

## 주의사항

### 쓰기 스킬 평가 시
- 테스트 프로젝트/캘린더를 사용하거나
- 승인 단계에서 거절하고 산출물만 확인
- 실제 생성 후 수동 삭제 비용이 큼

### 실데이터 필수
빈 Gmail/Calendar로 평가하면 의미 없음. 최소한:
- 미읽음 메일 30+통
- 오늘 일정 2개+
- (선택) 활성 스프린트 티켓 3개+

### 평가자 간 비교성
여러 세션이 각기 다른 환경(회사/개인 계정, MCP 연결 상태 등)에서 평가하므로 **실행 컨텍스트 섹션에 환경을 명시**해야 결과 비교 가능.

## 결과가 쌓이면

- `results/`의 결과들이 다음 버전 개선 우선순위를 결정
- D5(채택 확률) 점수가 낮은 스킬은 재설계 후보
- 여러 평가자가 동일하게 지적한 약점은 즉시 수정

## 평가 결과 활용

평가 완료 후 사용자는:
1. 결과를 본인이 읽고 개선 이슈 판단
2. 낮은 점수 차원을 타겟으로 `feat/` 브랜치 + PR
3. 필요시 루브릭/브리프 자체도 개선
