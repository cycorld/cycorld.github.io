---
layout: post
title: "Agent는 프롬프트가 아니라 구조로 좋아진다"
date: 2026-07-03 07:06:00 +0900
categories: ai agents claude
comments: true
---

> Anthropic Code w/ Claude 2026의 Agent Decomposition 세션을 보고 정리한 글입니다.  
> 핵심은 간단합니다. **Agent 개선은 긴 프롬프트가 아니라 `Eval → Skill → Tool → Subagent` 구조 재설계로 한다.**

## 원본 링크

- X 영상: [淘沙者(TheSandPicker) 공유 영상](https://x.com/etudecn/status/2072321319140782304)
- 공식 세션 페이지: [Tool, skill, or subagent? Decomposing an agent that outgrew its prompt](https://claude.com/code-with-claude/session/sf-ext-compose-multi-agent-systems-with-skills-and-mcp)
- 워크숍 코드: [anthropics/cwc-workshops — agent-decomposition](https://github.com/anthropics/cwc-workshops/tree/main/agent-decomposition)

※ 공개 검색 기준으로 YouTube 원본 URL은 확인하지 못했습니다. 현재 확인 가능한 공개 영상은 X에 업로드된 44분 38초 녹화본이고, 공식 원문은 Claude 세션 페이지와 GitHub 워크숍 저장소입니다. YouTube 공개본이 확인되면 이 글의 링크를 교체하겠습니다.

---

## 한 줄 요약

Agent가 복잡해져서 성능이 떨어질 때, 먼저 더 좋은 모델이나 더 긴 system prompt를 찾지 말고 다음 순서로 봐야 합니다.

```text
Eval 실행 → 실패 원인 분류 → Prompt 축소 → Skill 분리 → Tool 정리 → Subagent 재판단 → Eval 재실행
```

Anthropic 엔지니어는 `StockPilot`이라는 재고관리 Agent를 예제로 보여줍니다.

처음 상태는 이랬습니다.

| 항목 | Before |
|---|---:|
| System prompt | 402 lines |
| Tools | 12개 |
| Hardcoded subagents | 3개 |
| Eval score | 약 71% |
| F1 daily sweep | 488초, 102 tool calls |

개선 후에는 이렇게 바뀝니다.

| 항목 | After |
|---|---:|
| System prompt | 15 lines |
| 지식/정책 | 400 lines of Skills, 필요 시 로드 |
| Hardcoded subagents | 0개, delegation은 runtime decision |
| Eval score | 약 92% |
| F1 daily sweep | 약 100초, 3 scripts |

핵심은 지식을 버린 게 아닙니다. **항상 context에 넣던 지식을 필요할 때만 로드하도록 바꾼 것**입니다.

---

## 문제: 기능을 계속 붙이면 Agent는 망가진다

`StockPilot`은 야외용품 리테일러를 위한 재고관리 Agent입니다.

하는 일은 다음과 같습니다.

- 재고 부족 감지
- 수요 예측
- 공급업체 선택
- 발주서 작성
- 주간 리포트 작성

각 기능은 단독으로는 어렵지 않습니다. 문제는 시간이 지나며 기능이 계속 덧붙었다는 점입니다.

- 요구사항이 생길 때마다 system prompt에 추가
- 데이터 조회마다 custom tool 추가
- 복잡한 기능마다 subagent 추가
- 정책이 prompt 여기저기에 중복
- tool이 너무 많은 데이터를 context로 반환
- orchestrator와 subagent 사이에서 숫자/정책이 유실

결국 모델이 나빠서가 아니라 **Agent architecture가 노후화되어 eval이 하락**합니다.

---

## 1. Eval 없이 Agent를 고치지 않는다

가장 중요한 태도는 이것입니다.

> Agent 개선은 느낌으로 하지 않는다. Eval 점수로 hill climbing 한다.

워크숍에는 12개 eval task가 있습니다.

| ID | 의미 |
|---|---|
| R1-R5 | 기본 조회, 발주, lead time, 재고 조정 |
| R6-R8 | 재주문 추천, 14일 예측, 프로모션 월 예측 |
| R9 | 주간 리포트 |
| F1 | 일일 low-stock sweep |
| F2 | 프로모션 reorder + forecast |
| F3 | batch low-stock alerts |

Eval은 정답 여부만 보지 않습니다.

- pass/fail
- latency
- turn count
- tool call 수
- token usage
- 비용
- output format
- LLM-as-judge 품질 평가

즉, Agent 품질은 다음 네 가지를 같이 봐야 합니다.

```text
정확도 + 비용 + 속도 + 운영 안정성
```

---

## 2. System prompt는 짧아야 한다

나쁜 패턴은 이겁니다.

```text
새 요구사항 발생 → system prompt에 추가
새 예외 발생 → system prompt에 추가
새 보고서 형식 발생 → system prompt에 추가
새 정책 발생 → system prompt에 추가
```

이렇게 하면 system prompt가 업무 규칙 창고가 됩니다. 처음에는 편하지만 시간이 지나면 다음 문제가 생깁니다.

- 서로 다른 위치의 정책이 충돌
- 특정 작업에 필요 없는 정보까지 항상 context에 들어감
- 모델이 중요한 규칙을 놓침
- prompt 변경의 regression risk가 커짐

Anthropic이 권장하는 방향은 반대입니다.

### System prompt에 남길 것

- Agent의 역할
- 전역 안전 규칙
- 작업 원칙
- tool/skill/subagent를 선택하는 상위 기준

### Skill로 뺄 것

- 도메인 정책
- 계산 절차
- 보고서 템플릿
- 예외 처리 규칙
- 특정 업무에서만 필요한 지식

예시:

```text
항상 필요한 것 → system prompt
가끔 필요한 것 → skill
데이터 처리 → code execution / bash / file tools
독립 판단이 필요한 것 → subagent
```

이 방식을 Anthropic은 **progressive disclosure**라고 부릅니다. 필요한 정보만 그때그때 context에 공개하는 방식입니다.

---

## 3. Tool은 custom tool보다 primitive부터

영상에서 가장 실용적인 포인트입니다.

Agent에게 처음부터 업무별 custom tool을 많이 붙이지 말고, 인간이 컴퓨터에서 쓰는 기본 능력을 먼저 줍니다.

- 파일 읽기
- 파일 쓰기
- bash
- code execution
- web search
- todo list
- filesystem navigation

예를 들어 CSV 분석을 해야 한다고 합시다.

나쁜 방식:

```text
CSV 전체를 tool이 읽어서 모델 context에 반환한다.
```

좋은 방식:

```text
Agent가 Python script를 작성한다.
CSV를 로컬에서 계산한다.
필요한 결과만 context로 가져온다.
```

결과적으로 token usage, latency, cost가 내려갑니다.

워크숍에서도 F1 daily sweep이 `488초 / 102 tool calls`에서 `약 100초 / 3 scripts`로 줄었습니다.

핵심 원칙:

> 데이터를 모델 머리에 다 넣지 말고, 모델에게 컴퓨터를 쓰게 하라.

---

## 4. Subagent는 두 경우에만 쓴다

Subagent는 멋있지만 공짜가 아닙니다.

문제가 생기는 지점:

- orchestrator와 subagent 사이에서 정보가 유실됨
- 숫자, confidence, policy가 전달 중 왜곡됨
- logging과 observability가 어려움
- 단순 기능인데 subagent로 감싸 복잡도만 증가

영상에서 권장하는 subagent 사용처는 두 가지입니다.

### 1) 병렬로 많은 Claude를 던지고 싶을 때

- deep research
- web search
- codebase exploration
- 여러 후보안 탐색

### 2) fresh mind가 필요할 때

- 코드 리뷰
- 독립 검증
- forecasting처럼 기존 대화 context에 오염되면 안 되는 작업

반대로, subagent의 output이 단순 숫자 하나라면 보통 subagent가 아니라 tool이나 code execution이 맞습니다.

워크숍 README의 smell test가 좋습니다.

```text
Tool returns >2k tokens? → code execution instead.
Writing "always do X before Y" in the prompt? → skill.
Subagent's output is one number? → it shouldn't be a subagent.
```

---

## 5. Tool / Skill / Subagent 판단표

| 상황 | 선택 | 이유 |
|---|---|---|
| 항상 필요한 역할/원칙 | System prompt | 모든 작업에 적용 |
| 특정 작업에서만 필요한 절차 | Skill | 필요할 때만 context 로드 |
| 파일/CSV/계산/탐색 | Code execution / bash | token 절약, 정확도 상승 |
| 명확한 외부 API 호출 | Custom tool | I/O와 권한이 명확할 때 |
| 여러 Agent가 공유하는 표준 도구 | MCP | 재사용과 거버넌스 |
| 병렬 탐색 또는 독립 검증 | Subagent | 별도 context가 가치 있을 때 |
| 단순 기능 추가 | Subagent 금지 | 복잡도만 증가 |

---

## 6. 실제로 적용하는 절차

내 Agent가 느려졌거나 이상하게 틀리기 시작했다면, 다음 순서로 고칩니다.

### Step 1. 현재 구조 inventory

```md
# Agent Inventory

## Purpose
- 이 Agent의 핵심 목적:

## System Prompt
- 총 길이:
- 항상 필요한 규칙:
- 특정 작업에서만 필요한 규칙:

## Tools
| Tool | 용도 | 반환 token | 유지/삭제/대체 |
|---|---|---:|---|

## Skills
| Skill | 언제 필요한가 | 누가 호출하는가 |
|---|---|---|

## Subagents
| Subagent | 역할 | fresh mind? | parallelism? | 유지? |
|---|---|---|---|---|

## Evals
| Eval | 검증 대상 | 현재 결과 |
|---|---|---|
```

### Step 2. 최소 eval 10개 만들기

| 종류 | 개수 | 예시 |
|---|---:|---|
| 핵심 성공 경로 | 3 | 정상 요청 처리 |
| Regression | 3 | 예전에 되던 기능 |
| Failure mode | 2 | 모호한 요청, 충돌 정책 |
| 비용/속도 | 1 | tool call 수, token 수 |
| Output quality | 1 | 형식, tone, 완성도 |

### Step 3. 실패 원인을 분류

실패를 다음 범주로 나눕니다.

- system prompt 충돌
- skill 부재
- tool 반환 과다
- tool 선택 오류
- subagent 통신 문제
- 데이터 접근 방식 문제
- eval 자체 문제

### Step 4. Prompt를 줄이고 Skill로 뺀다

System prompt는 10~30줄 수준으로 유지하고, 업무 절차는 skill로 이동합니다.

### Step 5. Tool을 줄인다

각 tool에 대해 묻습니다.

```text
이 tool은 정말 필요한가?
아니면 bash/python/read/write/web search로 해결 가능한가?
```

### Step 6. Subagent를 재판단한다

다음 둘 중 하나가 아니면 제거 후보입니다.

- 병렬 탐색이 필요한가?
- fresh context가 필요한가?

### Step 7. Eval을 다시 돌린다

수정 후 반드시 다시 측정합니다.

```text
수정 → eval → 개선 확인 → merge
```

---

## 7. 내가 가져갈 운영 원칙

Agent 제품화에서 중요한 것은 “한 번 잘 되는 demo”가 아닙니다. 시간이 지나도 유지되는 구조입니다.

그래서 기본 architecture는 이렇게 잡는 게 좋겠습니다.

```text
짧은 System Prompt
+ 작업별 Skills
+ 인간형 Primitive Tools
+ 꼭 필요한 Subagents
+ Eval 기반 Hill Climbing
```

반대로 피해야 할 구조는 이겁니다.

```text
800줄 System Prompt
+ 기능별 Custom Tool 50개
+ Tool처럼 쓰는 Subagent 20개
+ Eval 없음
```

Agent가 커질수록 성능은 모델 하나로 해결되지 않습니다. 구조가 필요합니다.

---

## 바로 쓰는 체크리스트

- [ ] system prompt 길이를 잰다
- [ ] 항상 필요한 정보와 가끔 필요한 정보를 나눈다
- [ ] 가끔 필요한 정보는 skill로 뺀다
- [ ] custom tool을 primitive tool로 대체할 수 있는지 본다
- [ ] subagent는 병렬성/fresh mind가 없으면 제거한다
- [ ] 최소 10개 eval을 만든다
- [ ] token, latency, tool call 수를 같이 본다
- [ ] 수정할 때마다 eval을 다시 돌린다
- [ ] 좋아진 것만 merge한다

---

## 마무리

이 세션은 “AI assistant를 많이 만들어라”보다 더 중요한 메시지를 줍니다.

> Agent는 프롬프트로 커지는 것이 아니라, 구조로 성장해야 한다.

그리고 그 구조는 대략 이렇게 요약됩니다.

```text
System prompt는 짧게.
업무 지식은 Skill로.
데이터 처리는 Code execution으로.
Subagent는 꼭 필요할 때만.
개선은 Eval로 증명.
```
