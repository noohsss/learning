# ReAct Agent

## ReAct란

ReAct는 `Reasoning + Acting`의 조합이다. LLM이 답변을 한 번 생성하고 끝나는 것이 아니라, 현재 메시지를 보고 다음 Tool을 선택한 뒤 실행 결과를 다시 확인한다.

```text
Think → Act → Observe → 반복 → Answer
```

애플리케이션이 확인하는 것은 모델의 내부 추론 전체가 아니라 `tool_calls`, Tool 실행 결과인 `ToolMessage`, 최종 AI 메시지다.

## Tool의 역할

모델은 Tool 함수의 구현 코드를 직접 보는 것이 아니라 이름, docstring, 매개변수 타입으로 만들어진 schema를 보고 호출할 Tool과 인자를 선택한다.

```python
@tool(parse_docstring=True)
def get_current_time() -> str:
    """서버의 현재 로컬 날짜와 시간을 반환한다."""
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")
```

Tool 설명은 다음 내용을 포함하는 것이 좋다.

- 무엇을 처리하는 Tool인지
- 각 인자가 어떤 값인지
- 반환값의 의미
- 호출하면 안 되는 조건

파일·DB·네트워크를 사용하는 Tool은 모델의 판단과 별개로 함수 내부에서 경로, 확장자, 크기, 권한을 검사해야 한다.

## Tool 바인딩

```python
llm = ChatGoogleGenerativeAI(model=MODEL_NAME)
llm_with_tools = llm.bind_tools(tools)
```

`bind_tools()`는 LLM이 Tool schema를 참고해 `tool_calls`를 생성할 수 있게 한다. Tool을 바인딩했다고 함수가 바로 실행되는 것은 아니다. 실제 실행은 Agent 그래프의 `ToolNode`가 담당한다.

## MessagesState

`MessagesState`는 대화 메시지를 `messages` 키에 저장하는 LangGraph State다. Human, AI, Tool 메시지를 하나의 흐름으로 관리하고, 메시지 reducer를 통해 새 메시지를 기존 기록에 추가한다.

```python
def chatbot(state: MessagesState):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}
```

## ToolNode

`ToolNode`는 마지막 AI 메시지의 `tool_calls`를 읽고 다음 작업을 자동으로 처리한다.

1. Tool 이름과 인자를 파싱한다.
2. 등록된 함수를 실행한다.
3. 결과를 `ToolMessage`로 만든다.
4. Tool 호출 ID를 연결해 State에 추가한다.

서로 독립적인 Tool 호출은 한 AI 메시지에 여러 개 포함될 수 있다. 따라서 Tool은 전역 상태를 함부로 변경하지 않도록 설계하는 것이 안전하다.

## tools_condition

`tools_condition`은 마지막 AI 메시지를 보고 다음 경로를 선택한다.

```python
def should_continue(state: MessagesState):
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tools"
    return END
```

LangGraph에서 제공하는 `tools_condition`을 사용하면 위와 같은 분기 함수를 직접 작성하지 않아도 된다.

## ReAct 그래프

```python
graph_builder = StateGraph(MessagesState)
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", ToolNode(tools))

graph_builder.add_edge(START, "chatbot")
graph_builder.add_conditional_edges("chatbot", tools_condition)
graph_builder.add_edge("tools", "chatbot")

graph = graph_builder.compile()
```

실행 흐름은 다음과 같다.

```text
START → chatbot
          ├─ tool_calls 있음 → tools → chatbot
          └─ tool_calls 없음 → END
```

`chatbot`이 Tool 호출 메시지를 반환하면 `tools`가 실행되고, 결과가 다시 `chatbot`으로 전달된다. Tool 호출이 없는 AI 메시지가 반환되면 최종 답변으로 보고 종료한다.

## `create_agent()`와 직접 그래프 구성

```python
from langchain.agents import create_agent

agent = create_agent(model=llm, tools=tools)
result = agent.invoke({"messages": [("human", "지금 몇 시야?")]})
```

두 방식의 차이는 다음과 같다.

| 방식 | 특징 |
|---|---|
| 직접 `StateGraph` 구성 | Node, Edge, State, 분기 로직을 세밀하게 제어 |
| `create_agent()` | 일반적인 Tool 호출 Agent를 빠르게 구성 |

Agent 동작을 학습하거나 중간 상태를 직접 관리해야 한다면 그래프를 직접 구성하고, 일반적인 Agent가 필요하면 `create_agent()`를 사용할 수 있다.

## 로컬 파일 Agent

로컬 파일 Agent는 실제 파일 처리 함수와 모델에 노출할 Tool을 분리한다.

```text
질문 → 파일 검색 → 파일 읽기 → 내용 요약 → summary.txt 저장
```

파일 Tool에는 다음 제한을 둘 수 있다.

- 현재 작업 폴더 안의 파일만 허용
- `.txt` 파일만 허용
- 파일 크기 제한
- 저장할 내용의 길이 제한
- `../` 등을 이용한 상위 폴더 접근 차단

이 구조에서는 `_resolve_path()` 같은 안전성 검사 함수가 실제 파일 작업 전에 실행된다. Agent가 잘못된 파일명을 생성하더라도 Tool 내부에서 차단할 수 있다.

## 상품 조회 Agent

상품 검색과 상품 상세 조회를 서로 다른 Tool로 분리할 수 있다.

```text
상품명 질문 → search_products → 상품 ID 확인 → get_product → 가격·재고 응답
```

검색 Tool은 상품명과 ID 목록을 반환하고, 상세 Tool은 ID를 받아 가격과 재고를 반환한다. Agent는 첫 번째 Tool 결과를 보고 두 번째 Tool을 추가로 호출할 수 있다.

## 실행 결과 확인

최종 결과만 확인할 때는 `invoke()`를 사용한다. `stream()`을 사용하면 AI 메시지와 Tool 메시지가 추가되는 순서를 확인할 수 있다.

```python
for event in graph.stream({"messages": [("human", question)]}):
    for node_name, value in event.items():
        print(f"[{node_name}] {value}")
```

`create_agent()`로 만든 Agent도 메시지 State를 스트리밍할 수 있다. Tool 호출 이름을 출력하면 Agent가 어떤 순서로 Tool을 선택했는지 확인할 수 있다.

## 설계 시 확인할 점

1. Tool 이름과 설명만 보고도 용도를 알 수 있는가?
2. Tool 인자에 대한 입력 검증이 있는가?
3. Tool 실행 결과가 다음 LLM 호출에 충분한 정보인가?
4. Tool 호출이 계속 반복될 때 종료 조건이 있는가?
5. 외부 파일이나 DB를 변경하는 Tool의 권한 범위가 제한되어 있는가?
6. `stream()`으로 실제 실행 경로를 확인했는가?

## 정리

- ReAct는 LLM의 Tool 선택과 실행 결과 확인을 반복하는 패턴이다.
- `bind_tools()`는 LLM이 Tool 호출을 생성하도록 한다.
- `ToolNode`는 `tool_calls`를 실행하고 결과를 `ToolMessage`로 State에 추가한다.
- `tools_condition`은 Tool 호출이 있으면 Tool 노드로, 없으면 종료로 보낸다.
- 직접 그래프를 만들면 흐름을 세밀하게 제어할 수 있고, `create_agent()`는 구성이 간단하다.
