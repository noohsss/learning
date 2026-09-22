# Agent Handoff

Handoff는 현재 Agent가 다른 Agent에게 대화의 제어권을 넘기고, 전환된 Agent가 이후 사용자 요청에 계속 응답하는 Multi-Agent 패턴이다. Supervisor처럼 중앙 Agent가 모든 작업 결과를 회수하는 구조와 달리, Handoff에서는 현재 대화의 담당자 자체가 변경된다.

```text
사용자 → Sales Agent
             └─ 기술 문의 → Support Agent
                                └─ 구매 문의 → Sales Agent
```

## Supervisor와의 차이

| 구분 | Supervisor | Handoff |
|---|---|---|
| 제어 구조 | 중앙 Agent가 작업을 위임 | 현재 Agent가 담당권을 넘김 |
| 결과 처리 | 위임 결과가 Supervisor로 돌아옴 | 전환된 Agent가 사용자에게 응답 |
| 중심 개념 | 작업 조율 | 대화 담당자 전환 |
| 상태 | 작업 결과와 전체 Context 중심 | 현재 활성 Agent 유지 중심 |

작업을 여러 전문가에게 나누고 중앙에서 결과를 종합해야 하면 Supervisor가 적합하다. 여러 턴에 걸쳐 특정 전문가가 대화를 이어가야 하면 Handoff를 고려할 수 있다.

## Command를 이용한 전환

LangGraph의 `Command`는 State 변경과 다음 실행 위치를 하나의 반환값으로 표현한다.

```python
return Command(
    update={"category": "support"},
    goto="support",
)
```

- `update`: 현재 State에 반영할 값
- `goto`: 다음에 실행할 Node
- `graph=Command.PARENT`: 현재 내부 Graph가 아닌 부모 Graph의 Node로 이동

일반 Node는 State를 반환하고 Edge가 다음 Node를 결정한다. 반면 Command는 Node 안에서 State 변경과 Routing을 함께 결정한다. 상태 변경과 이동이 하나의 제어 동작일 때 유용하다.

## Handoff Tool

Handoff Tool은 일반 Tool처럼 업무 결과를 반환하는 대신 다른 Agent로 이동하는 `Command`를 반환한다. Tool 실행 시 필요한 `ToolRuntime`은 현재 State와 Tool 호출 정보를 제공한다.

- `runtime.state`: 현재 Agent의 State
- `runtime.tool_call_id`: 현재 Tool 호출을 식별하는 ID
- `Command.PARENT`: 부모 Graph로 제어권을 전달

Handoff가 발생할 때는 현재 Agent의 마지막 `AIMessage`와 Handoff Tool 실행 결과를 부모 State의 메시지 목록에 반영해야 한다. `ToolMessage.tool_call_id`에 같은 Tool 호출 ID를 사용하면 모델의 Tool Call과 실행 결과를 연결할 수 있다.

## 활성 Agent State

부모 State에 `active_agent`를 저장하면 현재 대화의 담당자를 명시적으로 관리할 수 있다. Graph 시작 시 `active_agent`를 기준으로 실행할 Agent를 선택하고, Handoff Tool은 이 값을 새로운 Agent 이름으로 갱신한다.

```text
START → active_agent 선택 → 현재 Agent 실행
                         ├─ 일반 응답 → END
                         └─ Handoff → active_agent 갱신 → 새 Agent 실행
```

같은 `thread_id`로 다음 사용자 메시지를 처리하면 이전 실행에서 저장된 `active_agent`가 유지된다. 따라서 새로운 요청마다 담당자를 처음부터 분류하지 않고, 현재 담당 Agent에서 대화를 이어갈 수 있다.

## Handoff 설계 시 주의점

- 각 Agent가 직접 처리할 수 있는 범위와 전환 조건을 명확히 한다.
- 하나의 요청에서 Handoff Tool을 중복 호출하지 않도록 제한한다.
- 전환 시 부모 State에 필요한 메시지와 활성 Agent를 함께 갱신한다.
- Agent 전환이 반복되는 순환 경로를 방지한다.
- 동일 Thread의 장기 대화에서 활성 Agent가 의도치 않게 남지 않는지 확인한다.
- 전환 대상 Agent가 필요한 Context와 권한만 전달받도록 한다.
