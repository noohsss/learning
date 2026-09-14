# Router

## Router란

Router는 요청을 분류해 적절한 처리 경로로 전달한다.

~~~text
요청 → 빠르고 확실한 판단 → 전문 노드
              ↓ 애매함
           LLM 판단 → 전문 노드 또는 fallback
~~~

## 라우팅 방식

| 방식 | 적합한 판단 | 특징 |
|---|---|---|
| Deterministic | 형식·키워드처럼 확정 가능한 조건 | 빠르고 예측 가능 |
| LLM | 맥락 해석이 필요한 의도 | 유연하지만 비용과 지연 발생 |
| Semantic | 대표 문장과 의미가 가까운 요청 | threshold와 예시에 민감 |
| Hybrid | 확실한 요청과 애매한 요청이 섞인 환경 | 비용과 정확도의 균형 |

## Deterministic Routing

State의 값을 명시적 규칙으로 검사해 경로를 결정한다. 요청 유형, 주문 상태, 사용자 권한, 입력 형식처럼 확실하게 판별할 수 있는 조건에 적합하다.

키워드가 하나의 경로와 일치하면 해당 경로를 반환하고, 충돌하거나 일치하지 않으면 uncertain 또는 fallback 경로를 반환한다.

## LLM Routing

LLM이 요청의 의도를 분류할 때는 Structured Output으로 목적지를 제한한다. 어느 경로에도 확실히 속하지 않는 요청을 위해 uncertain을 포함하는 것이 안전하다.

분기에는 제한된 category를 사용하고, reason은 관찰과 디버깅에 사용한다. LLM이 생성한 확률값은 보정된 확률이 아니므로 confidence처럼 그대로 사용하지 않는다.

## Hybrid Routing과 Fallback

명확한 요청은 규칙으로 처리하고 애매한 요청만 LLM으로 보낸다. LLM이 uncertain을 반환하거나 호출에 실패하면 clarification 또는 fallback 경로로 보낸다.

~~~text
규칙 일치 → 전문 노드
규칙 불일치 → LLM 분류
LLM uncertain·실패 → fallback
~~~

규칙 기반 필터를 앞에 두면 불필요한 LLM 호출을 줄일 수 있고, LLM만 사용하는 것보다 latency와 비용을 관리하기 쉽다.

## Semantic Routing

route별 대표 문장을 임베딩해 Chroma에 저장하고, 사용자 요청과 가장 가까운 대표 문서의 metadata에서 경로를 가져온다.

~~~text
사용자 요청 → 임베딩 → 대표 문장 검색 → relevance score 확인
                                      ├─ threshold 이상 → 해당 route
                                      └─ threshold 미만 → LLM fallback
~~~

대표 문장과 threshold는 검증 데이터로 조정해야 한다. threshold가 너무 낮으면 잘못된 경로가 선택되고, 너무 높으면 fallback 비율이 높아진다.

## Flat과 Hierarchical Routing

~~~text
Flat: User → Router → Refund
                    → Shipping
                    → Account
                    → Product

Hierarchical:
User → Domain Router → Finance → SQL / RAG
                    → Support → Refund / FAQ
                    → Engineering → GitHub / Logs
~~~

목적지가 적으면 Flat Router가 단순하다. 목적지가 많고 domain 경계가 명확하면 Hierarchical Router가 관리에 유리하지만 여러 Router를 거치므로 latency가 늘 수 있다.

| 비교 | Flat | Hierarchical |
|---|---|---|
| Router 호출 | 보통 한 번 | 두 번 이상 가능 |
| 적합한 상황 | 목적지가 적음 | 목적지가 많고 계층이 명확함 |

## 평가 기준

- routing accuracy: 올바른 목적지를 선택한 비율
- fallback rate: 자동 분류하지 않고 보류한 비율
- unsafe routing rate: 위험한 잘못된 경로를 선택한 비율
- latency: 최종 처리까지 걸린 시간
- token cost: Router와 처리 노드가 사용한 전체 토큰

라우팅에서는 정확도만 보지 말고, 위험한 잘못된 경로를 선택하는 비율과 fallback 비율도 함께 확인해야 한다.
