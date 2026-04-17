# Handoff Prompt — gws-harness 효용성 평가

> 이 프롬프트를 **새 Claude Code 세션**에 붙여넣어 특정 스킬/에이전트의 효용성을 평가한다.
> 각 평가는 독립 세션에서 수행하며, 결과는 `eval/results/{component}-{YYYY-MM-DD}.md`에 저장한다.

---

## 평가 대상 선택

아래 중 하나를 선택하여 프롬프트에 포함:

- `/gwh:triage` → `eval/briefs/gwh-triage.md`
- `/gwh:brief` → `eval/briefs/gwh-brief.md`
- `/gwh:cal-plan` → `eval/briefs/gwh-cal-plan.md`
- `/gwh:mail-to-ticket` → `eval/briefs/gwh-mail-to-ticket.md`
- `/gwh:digest` → `eval/briefs/gwh-digest.md`
- `gws-harness AGENT.md` → `eval/briefs/agent-harness.md`

---

## 프롬프트 템플릿 (복사해서 쓰세요)

```
당신은 gws-harness 플러그인의 `{COMPONENT_NAME}` 효용성을 평가하는 독립 평가자입니다.

## 사전 준비 (이미 되어있다고 가정)
- gws CLI 설치됨 (`which gws` 확인)
- Google Workspace 인증됨 (`gws auth status`)
- (선택) Atlassian MCP / Notion MCP 연결

## 수행 절차

1. `eval/briefs/{COMPONENT_FILE}`을 읽는다
2. `eval/rubric.md` 평가 루브릭을 숙지한다
3. 브리프의 시나리오를 **실제로 실행**한다 (시뮬레이션/모킹 금지)
4. 각 시나리오별로 관찰 사항 기록
5. `eval/rubric.md`의 5가지 차원(D1-D5)에 각 1-5점 부여, 근거 포함
6. 결과를 `eval/results/{COMPONENT}-{YYYY-MM-DD}.md`에 저장 (형식은 아래 참조)
7. 사용자에게 종합 점수 + 3가지 주요 인사이트 보고

## 평가 원칙
- **실데이터 필수**: 빈 메일함/빈 캘린더로 평가 금지
- **솔직하게**: 점수 부풀리지 말 것 (D5 특히)
- **구체적 근거**: "유용했다" ❌ → "N통 중 M통을 액션 가능한 형태로 분류" ✅
- **실패도 가치**: 작동하지 않은 부분, 오해한 부분 그대로 기록

## 결과 파일 형식

`eval/results/{COMPONENT}-{YYYY-MM-DD}.md`:

---
component: {COMPONENT_NAME}
evaluator: {session context, e.g. "session-on-2026-04-17"}
date: {YYYY-MM-DD}
total_score: {X}/25
verdict: {필수 | 강력 추천 | 유용 | 개선 필요 | 재설계 필요}
---

# Eval: {COMPONENT_NAME}

## 실행 컨텍스트
- 실제 데이터: 미읽음 N통 / 미팅 M개 / Jira 티켓 K개
- 환경 특이사항: (회사/개인 계정, MCP 연결 상태 등)

## 시나리오별 관찰
{브리프에 명시된 시나리오 각각}

## 루브릭 점수
| 차원 | 점수 | 근거 |
|------|:---:|------|
| D1 문제 해결력 | X/5 | ... |
| D2 신호/노이즈 | X/5 | ... |
| D3 컨텍스트 절약 | X/5 | ... |
| D4 재사용성 | X/5 | ... |
| D5 채택 확률 | X/5 | ... |
| **합계** | **X/25** | |

## 3가지 인사이트
1. (가장 강한 점)
2. (가장 약한 점)
3. (개선 제안)

## 스킬 개선 아이디어
- (있으면 기록)
---

이제 시작합니다. `eval/briefs/{COMPONENT_FILE}`부터 읽어주세요.
```

## 사용 예시

새 Claude Code 세션에서:
```
사용자: "당신은 gws-harness 플러그인의 /gwh:triage 효용성을 평가하는 독립 평가자입니다. ..."
```

`{COMPONENT_NAME}`과 `{COMPONENT_FILE}` 플레이스홀더를 실제 값으로 치환한 뒤 붙여넣으면 됩니다.

## 결과 취합

여러 세션이 개별 평가를 마치면 `eval/results/`에 누적됩니다. 취합 시:

```bash
# 종합 점수 집계
grep "^total_score:" eval/results/*.md

# verdict 분포
grep "^verdict:" eval/results/*.md | sort | uniq -c
```
