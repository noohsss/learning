# Human-in-the-Loop

## HITL이란

Human-in-the-Loop(HITL)는 AI가 모든 작업을 자동 처리하지 않고, 중요한 판단 지점에서 사람이 검토하거나 결정하도록 만드는 방식이다.

결제, 삭제, 이메일 발송처럼 되돌리기 어려운 작업이나, 정보가 부족해 사용자에게 추가 입력을 받아야 하는 작업에 적합하다.

~~~text
AI 작업 수행 → 사람의 판단이 필요한 지점에서 중단
            → 검토·입력 → 작업 재개
~~~

## interrupt()

interrupt(value)는 Node나 Tool 내부에서 그래프 실행을 중단하고 호출자에게 검토 정보를 반환한다. Checkpointer는 중단 당시의 State와 실행 위치를 저장한다.

~~~python
decision = interrupt(
    {
        "question": "이 결제를 실행하시겠습니까?",
        "tool": "send_payment",
        "args": {"amount": amount, "recipient": recipient},
    }
)
~~~

첫 번째 invoke는 interrupt payload를 반환한다. 사용자가 승인이나 거절을 결정하면 같은 thread_id에 Command(resume=결정)을 전달한다. 재개 시 중단된 Node나 Tool은 처음부터 다시 실행되며, interrupt는 resume 값을 반환하고 다음 코드로 진행한다.

## 중단에 사용되는 값

| 구분 | 역할 |
|---|---|
| 그래프 State | Node가 읽고 수정하는 내부 실행 상태 |
| interrupt payload | 사용자에게 보여줄 질문, Tool 이름과 인자 |
| resume value | 사용자가 반환하는 승인·거절 결정 |

payload와 resume value에는 문자열, 숫자, 목록, 딕셔너리처럼 JSON 직렬화 가능한 값을 사용한다.

## 부작용이 있는 작업

결제나 게시글 발행처럼 실제 외부 상태를 변경하는 코드는 interrupt 뒤에 둔다.

~~~python
decision = interrupt({"question": "실행할까요?"})

if decision["action"] == "reject":
    return "작업이 취소되었습니다."

return "작업을 실행했습니다."
~~~

interrupt 앞에서 외부 작업을 실행하면 재개 과정에서 같은 작업이 반복될 수 있다.

## Checkpointer와 thread_id

중단 후 재개하려면 Checkpointer와 thread_id가 필요하다.

~~~python
graph = builder.compile(checkpointer=MemorySaver())

config = {"configurable": {"thread_id": "payment-2"}}

result = graph.invoke({"messages": messages}, config)
request = result["__interrupt__"][0].value

result = graph.invoke(
    Command(resume={"action": "approve"}),
    config,
)
~~~

thread_id는 중단된 Graph 실행을 식별한다. 다른 thread_id를 사용하면 다른 State와 중단 지점을 조회한다.

## 승인·거절 흐름

승인 결정은 action만 전달할 수 있고, 거절 결정에는 reason을 추가할 수 있다.

~~~python
Command(resume={"action": "approve"})

Command(
    resume={
        "action": "reject",
        "reason": "금액을 다시 확인해야 합니다.",
    }
)
~~~

Tool은 거절 사유를 결과에 포함해 사용자에게 전달할 수 있다.

## Client와 Server 분리

실제 서비스에서는 첫 요청을 승인될 때까지 계속 연결해 두지 않는다.

~~~text
Client → POST /chat → Graph 실행 → interrupt
Client ← thread_id와 승인 질문 반환

사용자 승인·거절

Client → POST /resume
          thread_id + decision
       → Command(resume=decision)
       → Graph 재개
Client ← 최종 결과
~~~

첫 번째 API 응답에는 thread_id, 질문, Tool 이름, 인자를 포함할 수 있다. 두 번째 요청은 같은 thread_id와 결정값을 전달한다.

## 계획 검토 HITL

Plan-and-Execute의 Planner와 Executor 사이에 계획 검토 Node를 넣을 수 있다.

~~~text
Planner → 계획 검토
            ├─ 승인 → Executor
            └─ 거절 + 피드백 → Planner → 계획 검토
~~~

거절 피드백은 다음 Planner 호출에 전달하고, 수정된 계획을 다시 승인받는다. 계획이 승인된 뒤에만 외부 작업이나 Executor를 실행한다.

## 추가 정보 질문

사용자 요청에 필요한 정보가 부족하면 실행 전에 질문 Node에서 interrupt를 발생시킬 수 있다.

~~~text
요청 → 정보 충분성 확인
       ├─ 충분 → 실행
       └─ 불충분 → 질문 생성 → 사용자 답변 → 다시 확인
~~~

질문과 선택지를 Structured Output으로 만들고, 사용자의 답변은 reducer로 누적한다. 충분성 판단이 끝나면 실행 Node로 이동한다.

## Guardrail과 HITL 비교

| 구분 | Guardrail | HITL |
|---|---|---|
| 판단 주체 | 규칙·LLM·코드 | 사람 |
| 목적 | 허용 범위 밖의 입력·출력 차단 | 중요한 작업을 사람이 승인 |
| 처리 방식 | 자동 통과·거절·수정 | 중단 후 승인·거절·추가 입력 |
| 적합한 상황 | 반복적이고 명확한 정책 | 비용·위험이 크거나 애매한 작업 |

실무에서는 Input Guardrail로 명백한 위험을 차단하고, 통과한 위험 작업에는 HITL 승인을 추가할 수 있다.

## 설계 시 확인할 점

- interrupt 전에 외부 부작용이 실행되지 않는가?
- payload에 사용자가 판단할 정보가 충분한가?
- resume value를 검증하는가?
- 같은 thread_id로 재개하는가?
- 승인 대기 중 State를 안전하게 보관하는가?
- 사용자가 응답하지 않을 때 만료 정책이 있는가?
- 승인·거절과 Tool 실행을 감사 로그에 남기는가?
- 재개 시 Tool이 처음부터 실행된다는 점을 고려했는가?
