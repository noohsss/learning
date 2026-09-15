# Plan-and-Execute

## Plan-and-Execute란

Plan-and-Execute는 복잡한 요청을 먼저 여러 단계의 계획으로 나눈 뒤, 계획을 하나씩 실행하고, 중간 결과를 확인해 남은 계획을 수정하는 패턴이다.

~~~text
입력 → Planner → Executor → Replanner → Executor → ... → 최종 응답
~~~

한 번에 모든 작업을 LLM에게 맡기기보다 계획 수립, 개별 작업 실행, 진행 상황 판단을 분리한다. 조사·비교·일정 설계·보고서 작성처럼 여러 단계와 제약 조건이 있는 작업에 적합하다.

## ReAct와 비교

| 구분 | ReAct | Plan-and-Execute |
|---|---|---|
| 계획 범위 | 매 순간 다음 행동 결정 | 전체 계획을 먼저 수립 |
| 실행 방식 | LLM이 다음 Tool을 바로 선택 | Planner가 만든 단계의 첫 항목을 Executor가 실행 |
| 재검토 | Tool 실행 결과를 보고 다음 행동 결정 | Replanner가 남은 계획과 결과를 함께 검토 |
| 장점 | 단순하고 빠름 | 복잡한 목표와 제약 조건 처리에 유리 |
| 단점 | 전체 흐름과 의존 관계를 놓칠 수 있음 | Planner·Replanner 호출 비용 발생 |
| 적합한 작업 | 단일 조회, 간단한 질의 | 조사, 비교, 일정 설계, 보고서 작성 |

ReAct는 다음 행동을 짧은 주기로 결정한다. Plan-and-Execute는 먼저 전체 작업의 구조를 만들고, 실행 결과에 따라 그 구조를 수정한다. 두 패턴은 함께 사용할 수 있다. 예를 들어 Plan-and-Execute의 Executor 내부에서 ReAct Agent가 검색 Tool을 사용할 수 있다.

## 세 가지 역할

### Planner

Planner는 사용자 요청을 실행 가능한 단계로 나눈다.

- 조사 목표와 결과물 요구사항 파악
- 작업 사이의 의존 관계 반영
- 각 단계에 하나의 명확한 조사 목표 지정
- 실행하지 않은 결과를 미리 가정하지 않기
- 단계 수 제한

여행 계획이라면 교통, 관광지 운영 시간, 식당, 예산 검증처럼 서로 다른 조사 목표로 나눌 수 있다.

### Executor

Executor는 현재 plan의 첫 단계를 수행한다.

- 현재 단계만 집중해서 실행
- Search, Retriever, API, Tool 등을 사용
- 가격·시간·조건처럼 구체적인 결과 반환
- 출처 URL이나 문서 정보 기록
- 앞선 실행 결과가 필요하면 past_steps를 참고

Executor가 전체 계획을 다시 해석하게 하면 단계별 책임이 흐려지므로, 현재 단계와 필요한 맥락을 명확하게 전달하는 것이 좋다.

### Replanner

Replanner는 원래 요청, 남은 plan, past_steps를 비교해 다음 결정을 내린다.

- 필요한 정보가 부족하면 남은 조사 Plan 반환
- 검색 결과가 조건에 맞지 않으면 대안 조사 Plan 반환
- 모든 조건을 판단할 수 있으면 최종 Response 반환

~~~text
검색 결과 부족 → 추가 조사 계획
조건 불충족 → 대안 조사 계획
조건 충족 → 최종 응답
~~~

## State 설계

Plan-and-Execute는 실행 중간 정보가 많으므로 State 필드의 역할을 분리해야 한다.

~~~python
class PlanExecuteState(TypedDict):
    input: str
    plan: list[str]
    past_steps: Annotated[list[tuple[str, str]], operator.add]
    response: str
~~~

| 필드 | 역할 | 업데이트 방식 |
|---|---|---|
| input | 사용자의 원래 요청 | 초기 입력 유지 |
| plan | 아직 실행할 단계 | 새 계획으로 덮어쓰기 |
| past_steps | 실행 단계와 결과 | reducer로 누적 |
| response | 최종 응답 | 최종 결과 저장 |

plan은 Replanner가 남은 계획 전체를 바꿀 수 있으므로 일반 필드로 둔다. past_steps는 여러 Executor 결과를 보존해야 하므로 operator.add reducer를 사용한다.

## 구조화된 계획과 결정

Planner 출력은 문자열 그대로 받기보다 Pydantic 모델로 제한하는 편이 안전하다.

~~~python
class Plan(BaseModel):
    steps: list[str] = Field(description="순서대로 수행할 단계")


class Response(BaseModel):
    response: str = Field(description="사용자에게 전달할 최종 응답")


class ReplanDecision(BaseModel):
    decision: Union[Plan, Response]
~~~

Replanner는 한 번의 호출에서 남은 계획 또는 최종 응답 중 하나만 반환한다. 두 타입을 Union으로 묶으면 출력 형식을 코드에서 구분할 수 있다.

~~~python
if isinstance(result.decision, Response):
    return {"response": result.decision.response}

return {"plan": result.decision.steps}
~~~

Plan과 Response를 하나의 문자열로 반환하면 다음 동작을 코드로 구분하기 어렵다. 구조화된 결과를 사용하면 종료와 계속 실행을 명확하게 라우팅할 수 있다.

## Planner Prompt 설계

Planner Prompt에는 다음 내용을 넣는다.

~~~text
- 사용자의 목표
- 결과물의 형식
- 지켜야 할 제약 조건
- 조사 가능한 단계로 분리하라는 규칙
- 단계 수의 상한
- 아직 확인하지 않은 내용을 사실로 가정하지 말라는 규칙
~~~

단계가 너무 크면 Executor가 한 번에 많은 일을 처리하게 되고, 너무 작으면 LLM 호출 수가 늘어난다. 한 단계는 하나의 조사 목표를 가지되, 독립적으로 실행 가능한 크기로 정한다.

## Executor와 이전 결과

Executor가 이전 결과를 참고해야 하는 경우 past_steps를 문자열로 변환해 Prompt에 포함한다.

~~~python
past_context = ""
if state.get("past_steps"):
    past_context = "\n\n지금까지 확인한 결과:\n" + "\n".join(
        f"- {step}: {result}"
        for step, result in state["past_steps"]
    )
~~~

이전 결과를 전달할 때는 전체 내용을 무제한으로 넣지 않고, 필요한 정보만 요약하거나 길이를 제한하는 것이 좋다. 결과가 길어지면 입력 토큰과 비용이 증가하고, 현재 단계와 관계없는 정보가 판단을 방해할 수 있다.

## Replanner의 계획 수정

Replanner Prompt에는 완료된 단계를 반복하지 않도록 명시한다.

~~~text
- 이미 완료한 단계는 새 계획에서 제외한다.
- 검색 결과가 부족하면 필요한 추가 조사만 반환한다.
- 조건을 만족하지 못한 항목은 대안 조사 단계로 교체한다.
- 근거가 충분하면 최종 응답을 반환한다.
- 최종 응답은 조사 결과에 있는 정보만 사용한다.
~~~

완료된 단계를 다시 계획에 넣으면 같은 검색이 반복된다. 계획을 수정할 때는 남은 단계만 반환하도록 한다.

## 기본 그래프

~~~text
START → planner → executor → replanner
                         ↑          ├─ continue → executor
                         └──────────└─ end → END
~~~

~~~python
graph_builder = StateGraph(PlanExecuteState)

graph_builder.add_node("planner", plan_step)
graph_builder.add_node("executor", execute_step)
graph_builder.add_node("replanner", replan_step)

graph_builder.add_edge(START, "planner")
graph_builder.add_edge("planner", "executor")
graph_builder.add_edge("executor", "replanner")
graph_builder.add_conditional_edges(
    "replanner",
    should_continue,
    {
        "continue": "executor",
        "end": END,
    },
)

graph = graph_builder.compile()
~~~

Executor와 Replanner가 반복되므로 종료 조건이 반드시 필요하다. response가 State에 저장되면 end를 반환하고, 그렇지 않으면 continue를 반환한다.

## Replanner 없는 구조

초기 계획을 실행 중에 수정할 필요가 없다면 Replanner를 제거할 수 있다. Executor가 완료한 단계를 plan에서 제거하고, 남은 plan이 있는지 확인한다.

~~~text
START → planner → executor
                    ├─ plan 남음 → executor
                    └─ plan 없음 → responder → END
~~~

~~~python
def execute_step(state: PlanExecuteState):
    current_step = state["plan"][0]
    result = search_llm.invoke(current_step)

    return {
        "past_steps": [(current_step, result.text)],
        "plan": state["plan"][1:],
    }


def should_continue(state: PlanExecuteState):
    return "continue" if state.get("plan") else "end"
~~~

이 구조는 호출 흐름이 단순하고 예측 가능하다. 하지만 실행 결과에 따라 추가 조사나 대안 탐색을 할 수 없으므로, 계획이 고정된 작업에 적합하다.

## recursion_limit

recursion_limit은 한 번의 그래프 실행에서 허용하는 최대 Node 실행 횟수다. 반복되는 Node도 실행할 때마다 횟수에 포함된다.

~~~python
config = {"recursion_limit": 20}
graph.invoke(inputs, config=config)
~~~

상한을 초과하면 GraphRecursionError가 발생한다. recursion_limit은 무한 반복을 막는 안전장치이지, Planner가 좋은 계획을 만들도록 보장하는 설정은 아니다.

recursion_limit 외에도 다음 종료 장치를 둘 수 있다.

- 최대 조사 단계 수
- 최대 Replanner 호출 횟수
- 같은 단계 반복 감지
- 검색 결과가 일정 길이 이상이면 중단
- API 오류가 반복되면 fallback 응답

## stream으로 실행 과정 확인

Plan-and-Execute는 여러 단계가 반복되므로 stream으로 State 변화를 확인하면 디버깅하기 쉽다.

~~~python
for mode, event in graph.stream(
    inputs,
    config=config,
    stream_mode=["updates", "values"],
):
    if mode == "values":
        continue

    for node_name, value in event.items():
        print(node_name, value)
~~~

- updates: 각 Node가 갱신한 값 확인
- values: 각 단계의 전체 State 확인

updates에서는 Planner가 만든 plan, Executor가 추가한 past_steps, Replanner가 반환한 plan 또는 response를 구분해서 출력할 수 있다.

## 출처와 불확실성 관리

최신 정보를 조사하는 Executor라면 결과에 다음 정보를 포함하는 것이 좋다.

- 자료 제목
- 발행 주체
- 발행 시점
- URL
- 확인된 사실
- 해석 또는 불확실성

Replanner와 최종 응답 Node에는 조사 결과에 없는 내용을 추측하지 않도록 지시한다. 출처가 충돌하면 충돌 사실을 숨기지 않고 표시한다.

## 오류와 비용 관리

Plan-and-Execute는 Planner, Executor, Replanner 호출이 반복되므로 단순한 단일 LLM 호출보다 비용이 크다.

- Planner 단계 수를 제한한다.
- Executor 결과 길이를 제한한다.
- Replanner가 불필요한 반복을 만들지 않게 한다.
- API 재시도와 exponential backoff를 설정한다.
- 검색 결과가 충분하면 조기 종료한다.
- recursion_limit과 별도로 비용 상한을 둔다.

## Plan-and-Execute를 선택하는 기준

다음 조건이 많으면 Plan-and-Execute가 적합하다.

- 하나의 요청에 여러 조사 단계가 필요함
- 작업 사이에 순서나 의존 관계가 있음
- 중간 결과에 따라 다음 계획이 달라질 수 있음
- 최종 결과에 여러 출처와 조건을 반영해야 함
- 실행 과정을 State로 추적해야 함

반대로 단순한 Tool 하나 호출이나 짧은 질의는 ReAct 또는 일반 Chain이 더 간단하다.

## 정리

Plan-and-Execute는 계획과 실행을 분리해 복잡한 작업의 흐름을 관리하는 패턴이다. Planner는 단계 목록을 만들고, Executor는 현재 단계를 수행하며, Replanner는 결과를 바탕으로 남은 계획을 수정하거나 최종 응답을 반환한다. plan은 교체하고 past_steps는 누적하는 State 설계가 핵심이며, 반복 구조에는 종료 조건과 recursion_limit이 필요하다.
