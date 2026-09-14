# LangGraph 병렬화

## 병렬 처리란

서로 독립적인 여러 Node를 같은 단계에 연결해 동시에 실행하는 방식이다.

~~~text
START ─┬→ legal_analysis     ─┐
       ├→ financial_analysis ─┼→ summarize → END
       └→ technical_analysis ─┘
~~~

하나의 문서를 법률, 재무, 기술 관점으로 각각 분석한다면 순차 처리보다 전체 대기 시간을 줄일 수 있다. 전체 시간은 각 작업 시간의 합이 아니라 가장 오래 걸린 작업 시간과 병합 시간에 가까워진다.

## Fan-out과 Fan-in

- Fan-out: 하나의 시작점에서 여러 Node로 분기
- Fan-in: 여러 Node가 완료된 뒤 하나의 Node로 병합

각 병렬 Node가 같은 State 필드를 갱신하면 reducer가 필요하다.

~~~python
class AnalysisState(TypedDict):
    document: str
    analyses: Annotated[list, operator.add]
    summary: str
~~~

operator.add reducer는 각 Node가 반환한 리스트를 기존 리스트에 이어 붙인다. 병렬 실행 결과의 완료 순서는 보장되지 않는다.

## 결과 순서 보장

결과 순서가 중요하면 결과에 순번이나 식별자를 포함한다.

~~~python
return {"analyses": [(0, "[법률] 분석 결과")]}
~~~

병합 Node에서 식별자를 기준으로 정렬한 뒤 LLM에 전달한다.

~~~python
sorted_analyses = sorted(state["analyses"], key=lambda x: x[0])
all_analyses = "\n\n".join(content for _, content in sorted_analyses)
~~~

## Voting 패턴

하나의 질문에 대해 여러 후보를 동시에 생성하고 Judge LLM이 최선의 답변을 선택하는 방식이다.

~~~text
START ─┬→ generate_1 ─┐
       ├→ generate_2 ─┼→ vote → END
       └→ generate_3 ─┘
~~~

후보 생성용 LLM과 평가용 LLM을 분리하면 평가 기준을 독립적으로 관리할 수 있다. Structured Output으로 best_id와 선택 이유를 받으면 후보 선택을 코드에서 처리하기 쉽다.

## 병렬 처리 기준

병렬화는 각 작업이 다른 작업의 결과 없이 시작할 수 있을 때 적용한다.

적합한 예:

- 문서를 여러 관점으로 분석
- 문장을 여러 언어로 번역
- 여러 답변 후보 생성
- 여러 독립 API 조회

주의할 점:

- 병렬 요청이 많으면 Rate Limit에 도달할 수 있다.
- 완료 순서가 달라질 수 있다.
- 병렬화해도 가장 오래 걸리는 작업보다 짧아질 수는 없다.
- 작업 수와 입력이 단순하면 한 Node에서 batch()를 사용하는 편이 간단할 수 있다.
