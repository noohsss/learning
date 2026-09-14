# Reflection과 Evaluator-Optimizer

## Reflection

Reflection은 Agent가 자신의 출력 결과를 다시 읽고 피드백을 만든 뒤, 피드백을 반영해 다시 생성하는 패턴이다.

~~~text
START → generate → should_continue? → END
          ↑                │ continue
          └──── reflect ←──┘
~~~

State에는 생성 결과와 피드백을 메시지로 누적한다. generate는 이전 피드백을 참고하고, reflect는 현재 결과를 검토한다. 반복 횟수가 상한에 도달하면 최종 결과를 남기고 종료한다.

## Reflection의 장점과 한계

장점은 별도의 정답 데이터나 점수 모델 없이 출력의 누락·톤·길이·구조를 개선할 수 있다는 점이다. 반면 피드백이 자연어이므로 개선 정도를 정량화하기 어렵고, 합격 기준이 없으면 항상 고정 횟수까지 반복하게 된다.

반복이 길어질수록 표현만 미세하게 바뀌거나 원래 의도에서 벗어날 수 있다. 따라서 max_iterations를 두고, 실제 품질이 개선되는지 별도로 평가해야 한다.

## Evaluator-Optimizer

Evaluator-Optimizer는 평가 결과를 구조화하고, 합격 기준을 만족하지 못한 항목을 중심으로 개선하는 패턴이다.

~~~text
START → generate → evaluate → pass → END
          ↑            │ fail
          └── optimize ←┘
~~~

| 구분 | Reflection | Evaluator-Optimizer |
|---|---|---|
| 평가 결과 | 자연어 피드백 | 항목별 점수와 근거 |
| 종료 조건 | 주로 고정 횟수 | 합격 기준 또는 최대 횟수 |
| 개선 범위 | 전체적인 피드백 | 미달 항목 중심 |
| 주요 장점 | 구현이 단순함 | 종료 기준과 개선 방향이 명확함 |

평가자가 LLM이면 점수의 주관성과 실행별 변동성은 남는다. 생성 모델과 평가 모델을 분리하면 자기 평가 편향을 줄일 수 있다.

## 루브릭 설계

루브릭은 평가 항목과 점수 기준을 정리한 채점표다. 추상적인 표현보다 측정 가능한 기준을 작성한다.

| 나쁜 기준 | 좋은 기준 |
|---|---|
| 매력적인가? | 감정을 자극하는 단어가 1개 이상 포함되었는가? |
| 적절한 길이인가? | 50자 이상 150자 이하인가? |
| 좋은 카피인가? | 행동 유도 문구가 포함되었는가? |

마케팅 카피는 감정 자극, CTA, 글자 수를 평가할 수 있고, 기술 문서는 정확성, 재현 가능성, 코드 포함 여부를 평가할 수 있다.

## Structured Output 평가

Pydantic 모델로 점수와 근거를 제한하면 평가 결과를 코드에서 사용할 수 있다.

~~~python
class CriterionResult(BaseModel):
    score: int = Field(description="1~10 점수", ge=1, le=10)
    reason: str = Field(description="점수 근거 (1문장)")


class EvaluationResult(BaseModel):
    clarity: CriterionResult = Field(description="핵심 가치 전달 평가")
    emotion: CriterionResult = Field(description="타겟 공감과 변화 표현 평가")
    cta: CriterionResult = Field(description="행동 유도 평가")
    conciseness: CriterionResult = Field(description="길이와 중복 표현 평가")
    summary: str = Field(description="전체 평가 요약 (1~2문장)")
~~~

평가 결과를 코드에서 사용하려면 각 항목의 점수가 PASS_THRESHOLD 이상인지 확인한다.

~~~python
passed = all(
    evaluation[name]["score"] >= PASS_THRESHOLD
    for name in EVALUATION_CRITERIA
)
~~~

## 제품 리뷰 요약 흐름

제품 리뷰 요약은 장점, 단점, 추천 대상이 균형 있게 포함되어야 하는 작업이다.

Reflection 방식은 generate와 reflect를 반복하면서 자연어 피드백을 반영한다. Evaluator-Optimizer 방식은 completeness, accuracy, conciseness, recommendation 같은 항목을 점수화하고, 8점 미만 항목에 대한 수정 지시를 만들어 다시 생성한다.

## 구현 시 확인할 점

- 생성 결과와 평가 결과를 서로 다른 State 필드에 저장한다.
- 평가 결과는 코드가 읽을 수 있도록 구조화한다.
- 루브릭과 PASS_THRESHOLD를 명시한다.
- 반복 횟수 상한을 둔다.
- 생성 모델과 평가 모델을 분리할지 검토한다.
- 평가 점수를 절대적인 품질 보증값으로 해석하지 않는다.
- 원문에 없는 정보가 추가되지 않았는지 확인한다.
