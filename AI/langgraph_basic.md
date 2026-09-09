# LangGraph 기초

## LangGraph란

LangGraph는 LLM 애플리케이션의 실행 흐름을 그래프로 구성하는 프레임워크다. 상태를 공유하면서 여러 작업을 순서대로 실행하고, 조건에 따라 분기하거나 이전 단계로 돌아가 반복할 수 있다.

단순한 LLM 호출이나 직선형 파이프라인은 Chain으로 충분하지만, 검색 재시도·Tool 호출·검토와 재작성처럼 흐름이 복잡해지면 실행 과정을 그래프로 명시하는 편이 관리하기 쉽다.

```text
State → Node 실행 → State 갱신 → 다음 Node 선택
```

## Chain, Workflow, Agent

| 구분     | 다음 단계 결정 | 특징                            |
| -------- | -------------- | ------------------------------- |
| Chain    | 개발자         | 정해진 순서대로 실행            |
| Workflow | 개발자         | 조건 분기와 반복을 미리 정의    |
| Agent    | LLM            | 현재 상황에 따라 다음 행동 선택 |

LCEL은 직선형 파이프라인을 표현하는 데 적합하다. LangGraph는 개발자가 가능한 노드와 경로를 정의하고, 그 안에서 고정된 Workflow를 실행하거나 LLM이 다음 Tool과 노드를 선택하는 Agent를 만들 수 있다.

## 핵심 개념

### State

그래프의 모든 노드가 공유하는 데이터다. 질문, 답변, 검색 문서, 메시지, 시도 횟수 등을 저장한다.

```python
from typing import TypedDict

class State(TypedDict):
    question: str
    answer: str
```

`TypedDict`는 실행 시 일반 딕셔너리처럼 동작하지만, 상태 필드의 이름과 타입을 코드에 명시해 준다.

### Node

State를 입력으로 받아 하나의 작업을 수행하고, 변경된 값만 반환하는 함수다.

```python
def chatbot(state: State):
    response = llm.invoke(state["question"])
    return {"answer": response.content}
```

노드에는 LLM 호출, 검색, Tool 실행, 결과 검증, 질문 재작성 등의 역할을 넣을 수 있다. 하나의 노드가 너무 많은 일을 담당하면 테스트와 재시도가 어려워지므로 책임별로 나누는 것이 좋다.

### Edge

노드 사이의 실행 경로다.

- `START`: 시작점
- `END`: 종료점
- 일반 Edge: 항상 같은 노드로 이동
- Conditional Edge: State나 라우팅 결과에 따라 이동
- 순환 Edge: 이전 노드로 돌아가 반복

## 그래프 작성 순서

```python
from langgraph.graph import StateGraph, START, END

builder = StateGraph(State)
builder.add_node("chatbot", chatbot)
builder.add_edge(START, "chatbot")
builder.add_edge("chatbot", END)

graph = builder.compile()
result = graph.invoke({"question": "LangGraph가 뭐야?"})
```

1. `StateGraph(State)`로 그래프와 상태를 정의한다.
2. `add_node()`로 노드 이름과 함수를 등록한다.
3. `add_edge()`로 실행 순서를 연결한다.
4. `compile()`으로 실행 가능한 그래프를 만든다.
5. `invoke()`에 초기 State를 넣어 실행한다.

노드는 전체 State를 새로 만드는 대신 변경할 필드만 반환한다. 반환된 값은 State에 반영되고 다음 노드에 전달된다.

## 여러 단계의 Workflow

```python
class State(TypedDict):
    question: str
    plan: str
    answer: str

def make_plan(state: State):
    return {"plan": "질문 분석 후 답변 작성"}

def write_answer(state: State):
    return {"answer": f"계획: {state['plan']}"}

builder = StateGraph(State)
builder.add_node("plan", make_plan)
builder.add_node("write", write_answer)
builder.add_edge(START, "plan")
builder.add_edge("plan", "write")
builder.add_edge("write", END)
graph = builder.compile()
```

노드를 `계획 → 작성 → 검토`처럼 나누면 각 단계의 결과를 확인할 수 있고, 특정 단계만 다시 실행하거나 재시도하기도 쉽다.

## 조건부 분기

라우팅 함수가 반환한 값을 매핑해 다음 노드를 정한다.

```python
def route_by_language(state: State):
    return state["language"]

builder.add_conditional_edges(
    "detect_language",
    route_by_language,
    {"ko": "answer_ko", "en": "answer_en"},
)
```

```text
detect_language
  ├─ ko → answer_ko → END
  └─ en → answer_en → END
```

질문 유형 분류, 검색 여부 결정, Tool 호출 여부, 검토 결과에 따른 통과·재작성 등에 사용할 수 있다.

## 반복과 재시도

이전 노드로 연결하고 조건부 엣지에서 종료 여부를 판단하면 반복을 만든다.

```python
class CountState(TypedDict):
    count: int

def increment(state: CountState):
    return {"count": state["count"] + 1}

def route_by_count(state: CountState):
    return "end" if state["count"] >= 5 else "continue"

builder.add_node("increment", increment)
builder.add_edge(START, "increment")
builder.add_conditional_edges(
    "increment",
    route_by_count,
    {"continue": "increment", "end": END},
)
```

반복은 검색 결과가 부족할 때 질문을 다시 작성하는 RAG, 답변 검토 후 재작성하는 Workflow, Tool 결과를 확인하며 다시 판단하는 Agent에 사용할 수 있다. 반드시 종료 조건과 최대 시도 횟수를 두어 무한 루프를 막아야 한다.

## Node를 나누는 기준

- 하나의 명확한 책임을 가질 때
- State가 의미 있게 바뀔 때
- 분기나 재시도의 기준이 달라질 때
- 실행 시간이나 실패 가능성이 다를 때
- 단계별 로그와 디버깅이 필요할 때

예를 들어 `질문 재작성`, `문서 검색`, `검색 결과 평가`, `답변 생성`은 서로 다른 책임을 가지므로 별도 노드로 구성하는 것이 좋다.

## State 업데이트와 Reducer

기본 State 필드는 새 값으로 덮어쓴다.

```python
class State(TypedDict):
    name: str

# 기존 name이 새 값으로 교체된다.
return {"name": "새 이름"}
```

메시지나 로그처럼 여러 노드의 결과를 누적하려면 reducer를 지정한다.

```python
import operator
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

class ReducerState(TypedDict):
    name: str
    messages: Annotated[list, add_messages]
    logs: Annotated[list, operator.add]
```

| 필드       | 동작                                     |
| ---------- | ---------------------------------------- |
| `name`     | 새 값으로 덮어쓰기                       |
| `messages` | `add_messages` 규칙으로 메시지 누적·관리 |
| `logs`     | `operator.add`로 리스트 연결             |

항상 최신 값만 필요하면 일반 필드를 사용하고, 대화 기록이나 실행 로그처럼 누적이 필요할 때 reducer를 사용한다.

## `invoke()`와 `stream()`

`invoke()`는 전체 그래프가 끝난 뒤 최종 State를 반환한다.

```python
result = graph.invoke({"question": "질문"})
```

그래프의 `stream()`은 노드가 실행되는 중간 결과를 단계별로 전달한다.

```python
for update in graph.stream({"question": "질문"}):
    print(update)

for state in graph.stream(
    {"question": "질문"},
    stream_mode="values",
):
    print(state)
```

기본 `stream()`은 노드별 업데이트를 확인할 때 사용하고, `stream_mode="values"`는 각 단계의 전체 State를 확인할 때 사용한다.

`llm.stream()`과 혼동하면 안 된다. LLM의 `stream()`은 토큰이나 콘텐츠 조각을 실시간으로 받는 기능이고, 그래프의 `stream()`은 노드 실행 및 State 변화 과정을 확인하는 기능이다.

## 그래프 시각화

### Jupyter Notebook에서 확인

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

### `.py` 파일에서 PNG로 저장

```python
png_data = graph.get_graph().draw_mermaid_png()
with open("graph.png", "wb") as file:
    file.write(png_data)
```

PNG 생성 기능은 실행 환경에 Mermaid 렌더링에 필요한 의존성이 설치되어 있어야 한다.

시각화하면 시작점과 종료점, 조건부 분기, 반복 경로가 올바르게 연결되었는지 확인할 수 있다.

## 실전에서 확인할 점

1. 모든 경로가 `END`에 도달하는가?
2. 반복 노드에 종료 조건과 최대 횟수가 있는가?
3. 하나의 노드가 너무 많은 책임을 가지고 있지 않은가?
4. State 필드가 덮어쓰기인지 누적인지 명확한가?
5. 중간 State를 `stream()`으로 확인할 수 있는가?
6. 그래프 시각화로 의도하지 않은 경로를 확인했는가?

## 정리

LangGraph의 핵심은 LLM 호출 자체가 아니라 LLM 애플리케이션의 상태와 흐름을 명시적으로 관리하는 것이다. 단순한 Chain은 그대로 사용하고, 조건 분기·반복·재시도·상태 공유가 필요한 부분에 LangGraph를 적용하면 Workflow와 Agent를 안정적으로 구성할 수 있다.
