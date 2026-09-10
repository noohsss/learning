# LangGraph Memory와 State

## State와 대화 기억

LLM 호출은 기본적으로 서로 독립적이다. 이전 대화를 계속 사용하려면 메시지와 실행 상태를 State에 넣고, 실행 사이에 저장·복원해야 한다.

```text
현재 대화 → State 갱신 → Checkpointer 저장
다음 호출 → thread_id 조회 → 이전 State 복원
```

ReAct Agent에서 `Human → AI → Tool → AI` 흐름이 하나의 대화처럼 이어지는 것도 하나의 그래프 실행 안에서 같은 `messages` State를 사용하기 때문이다.

## Checkpointer

Checkpointer는 그래프 시작 시와 노드 실행 후 State snapshot을 저장한다. 같은 `thread_id`로 그래프를 다시 호출하면 해당 대화의 State를 바탕으로 실행한다.

```python
memory = MemorySaver()
graph_with_memory = simple_builder.compile(checkpointer=memory)

config = {"configurable": {"thread_id": "session-1"}}

graph_with_memory.invoke(
    {"messages": [("human", "내 이름은 철수야. 반가워!")]},
    config=config,
)
graph_with_memory.invoke(
    {"messages": [("human", "내 이름이 뭐라고 했지?")]},
    config=config,
)
```

같은 Graph를 여러 번 호출해도 `thread_id`가 다르면 각각 독립된 대화다. 같은 `thread_id`를 사용해야 이전 메시지를 이어간다.

## State 조회

```python
state = graph_with_memory.get_state(config)
print(state.values)
print(f"저장된 메시지 수: {len(state.values['messages'])}")

history = list(graph_with_memory.get_state_history(config))
```

- `get_state(config)`: 해당 thread의 최신 State
- `state.values`: 실제 State 값
- `get_state_history(config)`: 과거 snapshot iterator

`MemorySaver`는 메모리에 저장하는 구현체라 프로세스가 종료되면 데이터가 사라진다. 운영 환경에서는 SQLite, PostgreSQL 등의 영속 Checkpointer를 고려한다.

## Checkpointer와 Store 비교

| 구분 | Checkpointer | Store |
|---|---|---|
| 구분 기준 | `thread_id` | `namespace`, `key` |
| 목적 | 같은 대화 이어가기 | 여러 대화에서 정보 공유 |
| 저장 대상 | 메시지, 현재 작업 단계, Tool 결과 | 사용자 프로필, 선호, 장기 설정 |
| 사용 범위 | 하나의 대화 스레드 | 사용자 또는 애플리케이션 범위 |

Long-term Memory는 반드시 영구 저장된다는 뜻이 아니라, 하나의 대화 스레드를 넘어 공유된다는 의미다.

## Store 기본 사용

```python
store = InMemoryStore()
user_id = "user-001"
user_namespace = ("users", user_id)

store.put(
    namespace=user_namespace,
    key="profile",
    value={"name": "철수", "interest": "Python"},
)

profile = store.get(user_namespace, "profile")
print(f"key={profile.key}, value={profile.value}")
```

주요 메서드는 다음과 같다.

- `put(namespace, key, value)`: 값 저장
- `get(namespace, key)`: 특정 값 조회
- `search(namespace)`: namespace 안의 데이터 목록 조회

## Store를 사용하는 Node

Store를 사용하는 Node는 `state`, `config`, `store`를 매개변수로 받을 수 있다.

```python
def memory_chatbot(
    state: MessagesState,
    config: RunnableConfig,
    store: BaseStore,
):
    user_id = config["configurable"]["user_id"]
    namespace = ("users", user_id)
    profile = store.get(namespace, "profile")
```

그래프를 컴파일할 때 Store를 연결한다.

```python
memory_graph = memory_builder.compile(
    checkpointer=MemorySaver(),
    store=store,
)
```

`thread_id`가 달라도 같은 `user_id`를 사용하면 Store의 프로필을 조회할 수 있다. 반대로 사용자별 namespace를 다르게 만들면 같은 `profile` key를 사용해도 데이터가 섞이지 않는다.

```text
("users", "user-001") / profile → 철수
("users", "user-002") / profile → 영희
```

namespace는 논리적인 데이터 분리 규칙이지 인증이나 권한 검사가 아니다. 클라이언트가 보낸 `user_id`를 그대로 신뢰하지 말고 인증된 사용자 정보로 namespace를 구성해야 한다.

## 대화 요약이 필요한 이유

대화가 길어지면 메시지가 누적되어 토큰 비용과 입력 길이가 증가한다. State를 무조건 삭제하면 중요한 정보가 사라질 수 있으므로, 오래된 대화를 요약하고 최근 메시지는 원문으로 유지하는 방식을 사용할 수 있다.

```text
START → chatbot → 토큰 초과 → summarize → END
                → 토큰 이내 →             END
```

## Summary State

`MessagesState`를 상속하고 `summary` 필드를 추가한다.

```python
class SummaryState(MessagesState):
    """최근 메시지와 오래된 대화의 요약을 관리하는 State."""

    summary: str
```

답변을 생성한 뒤 메시지 토큰 수를 계산하고 요약 여부를 결정한다.

```python
def route_after_chatbot(state: SummaryState):
    token_count = llm.get_num_tokens_from_messages(state["messages"])

    if token_count > MAX_TOKENS:
        return "summarize"
    return END
```

## `trim_messages`와 `RemoveMessage`

`trim_messages`는 토큰 기준으로 최근 메시지를 남긴다. 유지할 메시지와 요약할 메시지를 ID로 나눈 뒤, 요약할 메시지를 LLM에 전달한다.

`RemoveMessage`는 `add_messages` reducer가 특정 ID의 메시지를 현재 State에서 제거하도록 한다.

```python
return {
    "summary": summary_response.text,
    "messages": [
        RemoveMessage(id=message.id) for message in messages_to_summarize
    ],
}
```

기존 요약이 있으면 새로 요약할 대화와 기존 요약을 합쳐 summary를 갱신한다. 다음 LLM 호출에는 기존 요약을 합성한 메시지와 최근 messages를 함께 전달할 수 있다.

## 현재 State와 원본 대화

`RemoveMessage`로 삭제된 메시지는 현재 그래프 State에서는 사라진다. 하지만 Checkpointer가 저장한 과거 snapshot에는 이전 State가 남아 있을 수 있다.

원본 대화 전체를 보존해야 하는 서비스라면 별도의 DB나 로그 저장소를 사용해야 한다. Checkpointer는 그래프 실행을 복원하기 위한 저장소이지 원본 대화 보관소를 대신하지 않는다.

## 정보 저장 위치 선택

```text
현재 대화의 흐름·중간 결과 → State + Checkpointer
여러 대화에서 재사용할 정보 → Store
토큰이 커진 대화 → summary + 최근 messages
영구 보관이 필요한 원본 대화 → 별도 DB·로그 저장소
```

## 정리

- State는 현재 그래프 실행에 필요한 데이터를 관리한다.
- Checkpointer는 `thread_id`별 대화 State와 snapshot을 저장한다.
- Store는 namespace와 key를 기준으로 여러 대화에서 공유할 정보를 저장한다.
- 같은 `thread_id`는 대화를 이어가고, 다른 `thread_id`는 대화를 분리한다.
- 같은 `user_id`와 namespace를 사용하면 다른 thread에서도 Store 정보를 재사용할 수 있다.
- 긴 대화는 summary와 최근 messages를 함께 사용하는 방식으로 관리한다.
