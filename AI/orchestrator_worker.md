# Orchestrator-worker

## 패턴이란

Orchestrator가 요청을 분석해 작업 목록을 만들고, 작업마다 Worker를 동적으로 실행한 뒤 결과를 하나의 보고서로 합치는 패턴이다.

~~~text
START → Orchestrator ─┬→ Worker
                      ├→ Worker → 결과 종합 → END
                      └→ Worker
~~~

고정 Fan-out은 그래프를 만들 때 작업 종류와 수가 정해진다. Orchestrator-worker는 실행 시점에 작업 목록과 Worker 수가 결정된다.

## Send API

Send(노드이름, state)는 지정한 Worker Node를 새로운 입력 State로 실행한다. 라우팅 함수가 Send 객체의 리스트를 반환하면 리스트 길이만큼 동적 Fan-out이 발생한다.

~~~python
def assign_workers(state: ReportState):
    return [
        Send(
            "analyze_task",
            {
                "request": state["request"],
                "task_id": task_id,
                "title": task.title,
                "description": task.description,
            },
        )
        for task_id, task in enumerate(state["tasks"])
    ]
~~~

## 전체 State와 Worker State

ReportState는 원본 요청, 작업 목록, Worker 결과, 최종 보고서를 저장하는 전체 State다. WorkerState는 Worker 하나가 담당 작업을 처리하는 데 필요한 값만 가진다.

Worker들은 같은 WorkerState schema를 사용하지만 Send가 전달하는 값은 서로 독립적이다. Worker가 전체 공유 State에서 자신의 작업을 다시 찾을 필요 없이 전달받은 값을 바로 사용한다.

task_id는 병렬 실행 결과가 어느 작업에서 나온 것인지 식별하고, 원래 계획 순서로 정렬하기 위한 값이다.

## 구조화된 작업 계획

Orchestrator가 LLM의 자유 형식 응답을 반환하면 작업 목록을 코드에서 사용하기 어렵다. Pydantic과 with_structured_output으로 작업 계획을 구조화한다.

~~~python
class AnalysisTask(BaseModel):
    title: str = Field(description="분석 항목 제목")
    description: str = Field(description="분석할 내용")


class AnalysisPlan(BaseModel):
    tasks: list[AnalysisTask] = Field(description="보고서 작성에 필요한 분석 작업")
~~~

Orchestrator는 요청을 3~5개 작업으로 나누고, assign_workers가 작업마다 Send를 만든다.

## Worker 결과 병합

여러 Worker가 같은 results 필드를 갱신하므로 reducer를 사용한다.

~~~python
class ReportState(TypedDict):
    request: str
    tasks: list[AnalysisTask]
    results: Annotated[list, operator.add]
    final_report: str
~~~

병렬 완료 순서는 계획 순서와 다를 수 있으므로 make_report에서 task_id를 기준으로 정렬한다.

~~~python
sorted_results = sorted(
    state["results"],
    key=lambda item: item["task_id"],
)
~~~

## Rate Limiting

Orchestrator가 여러 작업을 만들면 짧은 시간에 다수의 LLM 요청이 발생한다.

- RPM: 분당 요청 수
- TPM: 분당 처리 토큰 수
- RPD: 일일 요청 수
- 동시 요청 수

max_concurrency는 한 번의 그래프 실행에서 동시에 수행할 Worker 수를 제한한다. 전체 Worker 수를 줄이는 설정은 아니며, 실행 중인 Worker가 끝나면 대기 중인 Worker가 시작된다.

~~~python
result = rate_limit_graph.invoke(
    {"topics": topics},
    config={"max_concurrency": 3},
)
~~~

max_concurrency만으로 RPM과 TPM을 직접 계산하지는 않는다. 재시도, exponential backoff, 요청 속도 제한을 함께 고려해야 한다.

## Worker 설계 기준

- Worker 입력 State를 작게 유지한다.
- 모든 Worker가 같은 함수 구조를 사용하게 한다.
- 결과에 task_id나 order를 포함한다.
- reducer로 결과를 누적한다.
- 최종 병합 단계에서 순서를 정렬한다.
- Worker 수와 동시 실행 수를 제한한다.
- 실패한 Worker를 재시도하거나 부분 결과를 처리할 정책을 둔다.
