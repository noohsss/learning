# Supervisor Pattern

Supervisor 패턴은 중앙 Supervisor가 사용자 요청을 분석하고 여러 전문 Agent에게 작업을 위임한 뒤, 결과를 종합해 최종 응답을 만드는 Multi-Agent 구조다.

```text
사용자 → Supervisor
             ├─ Research Agent
             ├─ Writer Agent
             └─ Reviewer Agent
```

## Multi-Agent를 사용하는 이유

하나의 Agent가 모든 역할, Tool, Context를 담당하면 Prompt가 복잡해지고 Tool 선택 오류와 지시 충돌이 증가할 수 있다. 역할별 Agent를 분리하면 각 Agent가 자신의 작업에 필요한 지시와 정보에 집중할 수 있다.

- 역할과 Context 분리
- Agent별 Tool과 권한 제한
- 역할별 Prompt와 모델 설정
- Agent별 독립적인 평가 기준
- 요청과 중간 결과에 따른 동적 작업 위임

다만 Agent 수가 늘어나면 모델 호출 횟수, 지연 시간, 비용, 실패 지점도 증가한다. 실행 순서가 단순하고 고정되어 있다면 일반 Workflow나 단일 Agent가 더 적합할 수 있다.

## Supervisor의 책임

Supervisor는 직접 모든 세부 작업을 수행하기보다 다음 역할을 담당한다.

1. 사용자 요청과 현재 Context를 분석한다.
2. 필요한 전문 Agent를 선택한다.
3. 선택한 Agent에 작업 목적과 필요한 정보를 전달한다.
4. 반환된 결과를 대화 Context에 반영한다.
5. 추가 작업이 필요한지 판단한다.
6. 충분한 결과를 바탕으로 최종 응답을 생성한다.

## Subagent를 Tool로 감싸는 구조

LangChain에서는 전문 Agent를 Tool 함수 안에서 호출하고, Supervisor에게는 이 Tool만 제공하는 방식으로 구성할 수 있다. 각 Tool은 담당 작업을 설명하는 입력을 받아 Subagent를 실행하고, Subagent의 최종 응답을 Supervisor에게 반환한다.

이 구조의 장점은 Supervisor가 각 Agent의 내부 구현을 알 필요 없이 Tool 이름과 설명을 기준으로 위임할 수 있다는 점이다. Subagent는 자신의 Prompt, Tool, Context를 독립적으로 가질 수 있다.

## ReAct와 Supervisor

Supervisor는 요청을 분석하고 Tool을 호출한 뒤 결과를 관찰하여 다음 작업을 결정하는 ReAct 흐름으로 동작할 수 있다.

```text
판단 → Subagent Tool 호출 → 결과 관찰 → 다음 작업 판단 → 최종 응답
```

자료 조사가 먼저 필요한지, 작성 결과를 검토해야 하는지, 검토 결과에 따라 다시 작성해야 하는지를 Supervisor가 판단한다. 따라서 고정된 순서보다 작업 상태와 중간 결과에 따라 흐름이 달라지는 업무에 적합하다.

## Checkpointer 위치

대화의 전체 흐름을 관리하는 최상위 Supervisor에 Checkpointer를 두면 사용자 대화와 위임 결과를 하나의 Thread에서 관리할 수 있다. Subagent마다 별도의 영속 상태가 필요한 경우에는 Agent별 Checkpointer 정책을 추가로 설계할 수 있다.

## Supervisor 설계 기준

- Agent의 책임 범위를 명확히 정의한다.
- Tool 설명에 호출 조건과 반환 결과를 구체적으로 작성한다.
- Supervisor가 불필요하게 같은 Agent를 반복 호출하지 않도록 종료 조건을 둔다.
- Agent 간에 전달할 Context의 범위를 최소화한다.
- Agent별 실패와 Supervisor 전체 실패를 구분해 처리한다.
- 비용과 지연 시간을 고려해 병렬화나 호출 횟수 제한을 적용한다.
