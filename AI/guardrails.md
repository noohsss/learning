# Guardrails

## Guardrails란

Guardrail은 Agent가 허용된 범위 안에서만 판단하고 행동하도록 만드는 안전·통제 장치다. Agent는 API 호출, 데이터 변경, 이메일 발송처럼 외부 시스템에 실제 영향을 줄 수 있으므로 프롬프트 지시만으로 통제하지 않고 여러 계층을 조합한다.

~~~text
User → Input Guardrail → Agent/LLM → Tool Guardrail → Tools/API
                                      ↓
                              Output Guardrail → User
~~~

| 계층 | 역할 |
|---|---|
| Input Guardrail | 빈 입력, 길이 제한, 유해 요청, 프롬프트 인젝션 검사 |
| Policy / Tool Guardrail | 권한, Tool 인자, 업무 규칙, 승인 여부 검사 |
| Output Guardrail | 민감정보, 금지 콘텐츠, 응답 형식 검사 |
| Runtime Guardrail | 호출 횟수, 시간, 비용 제한과 감사 기록 |

## Input Guardrails

업무 처리 노드 앞에서 입력을 검사한다.

~~~text
START → check_input → 안전? → handle_query → END
                       유해? → reject → END
~~~

빈 입력과 최대 길이처럼 확정적인 조건은 규칙 기반으로 먼저 검사하고, 안전성과 프롬프트 인젝션처럼 맥락이 필요한 조건은 LLM으로 판단할 수 있다.

유해 입력을 감지하면 이후 Agent나 Tool 노드를 실행하지 않고 거부 응답으로 분기한다. 명백한 공격은 키워드 규칙으로 빠르게 차단하고, 미묘한 공격은 LLM 기반 탐지로 보완하는 Hybrid 방식도 사용할 수 있다.

## 프롬프트 인젝션 방어

프롬프트 인젝션은 사용자가 시스템 지시를 무시하거나 숨겨진 지시를 노출하도록 유도하는 공격이다.

- 키워드 필터와 LLM 판단을 함께 사용하는 다층 방어
- 사용자 입력과 시스템 지시의 역할 분리
- Agent와 Tool에 최소 권한만 부여
- 검사 모델의 오류와 시간 초과에 대한 차단 정책
- Output Guardrail과 권한 검사를 함께 사용

완벽한 방어는 보장할 수 없으므로 하나의 탐지기만 신뢰하지 않는다.

## Output Guardrails

최종 응답을 반환하기 전에 검증 노드를 둔다.

~~~text
START → generate → validate_output → 안전? → pass_through → END
                                      민감정보? → sanitize → END
~~~

전화번호, API 키, 이메일 등 민감정보를 정규식으로 감지할 수 있다. 정규식은 기본 형식에는 유용하지만 난독화된 값이나 모든 국가별 형식을 보장하지 않는다. PII 탐지 라이브러리나 LLM 기반 검사를 추가로 사용할 수 있다.

민감정보를 단순히 마스킹하는 대신 응답을 다시 생성하는 루프를 만들 수도 있다.

~~~text
generate → validate → 민감정보 있음? → generate
                      안전? → END
~~~

재생성 루프에는 retry_count와 최대 횟수가 필요하다. 상한에 도달하면 안전한 고정 응답으로 종료한다. 같은 구조로 할루시네이션, 응답 포맷, 톤과 스타일도 검증할 수 있다.

## Input과 Output Schema

내부 State에는 raw_response나 검사 결과처럼 내부 처리용 필드가 포함될 수 있지만, 외부 호출자에게 모두 노출할 필요는 없다.

~~~python
StateGraph(
    InternalState,
    input_schema=InputSchema,
    output_schema=OutputSchema,
)
~~~

- 첫 번째 인자: Node 간 공유하는 내부 State
- input_schema: 그래프 입력 형태
- output_schema: 그래프 출력 형태

내부 실행 정보와 외부 API 응답을 분리하면 구현 세부 정보가 불필요하게 노출되는 것을 줄일 수 있다.

## 적용 원칙

1. 입력 검사는 실제 업무 노드보다 앞에 둔다.
2. 민감한 Tool은 권한과 인자를 코드로 검증한다.
3. 출력 검사는 사용자에게 반환하기 직전에 수행한다.
4. 재시도에는 최대 횟수와 실패 시 응답을 둔다.
5. Guardrail 실패도 로그와 감사 기록에 남긴다.
6. LLM의 판단만으로 보안 정책을 강제하지 않는다.
