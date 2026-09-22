# LangGraph Subgraph

Subgraph는 별도로 구성한 그래프를 다른 그래프의 Node처럼 포함하는 구조다. 복잡한 작업을 여러 그래프로 나누고, 부모 Graph는 전체 흐름을 관리하며 각 Subgraph는 특정 작업의 내부 흐름을 담당한다.

## 사용하는 이유

- **재사용**: 여러 Workflow에서 동일한 작업 Graph를 반복해서 활용할 수 있다.
- **역할 분리**: 조사, 분석, 생성처럼 서로 다른 작업을 별도의 Graph로 관리할 수 있다.
- **State 분리**: 내부 작업에 필요한 데이터를 Subgraph 안에서 관리하고 부모와 필요한 값만 교환할 수 있다.
- **독립적인 테스트**: Subgraph를 단독으로 실행해 내부 흐름을 검증할 수 있다.

단순한 한 단계 작업에는 일반 Node가 적합하다. 여러 Node와 분기, 도구 호출로 구성된 작업을 하나의 재사용 가능한 단위로 묶을 때 Subgraph의 장점이 커진다.

## State 공유

부모 Graph와 Subgraph가 같은 State 스키마를 사용하면 컴파일된 Graph를 부모 Graph의 Node로 직접 등록할 수 있다. Subgraph는 State 전체를 새로 반환하지 않고 변경된 필드만 반환하며, 반환하지 않은 필드는 기존 값이 유지된다.

공유 State를 사용할 때는 부모와 Subgraph가 어떤 필드를 읽고 쓰는지 명확히 정해야 한다. 공통 필드가 많아질수록 결합도가 높아질 수 있으므로 각 Graph의 책임과 수정 가능한 필드를 제한하는 것이 좋다.

## State가 다른 Graph 연결

부모와 Subgraph의 전체 State 구조가 달라도 서로 주고받는 필드의 이름과 형식이 호환되면 연결할 수 있다. 필드 이름이나 데이터 형식이 다르면 래퍼 함수가 변환 계층이 된다.

```text
부모 State → 입력 변환 → Subgraph State
Subgraph 결과 → 출력 변환 → 부모 State
```

래퍼 함수는 부모의 입력 중 Subgraph에 필요한 값만 추출하고, Subgraph의 결과를 부모가 사용하는 필드로 변환한다. 이 방식은 각 Graph가 자신의 State 설계를 독립적으로 유지할 수 있게 한다.

## 실행 스트리밍

부모 Graph를 기본 설정으로 스트리밍하면 부모 Node의 결과만 전달된다. `subgraphs=True`를 사용하면 부모 Graph와 내부 Subgraph의 이벤트가 함께 전달된다.

- `subgraphs=False`: 부모 Graph의 Node 결과만 반환
- `subgraphs=True`: 부모와 내부 Graph의 이벤트를 함께 반환
- `namespace`: 이벤트가 어느 Graph에서 발생했는지 나타내는 경로
- `chunk`: 해당 단계에서 변경된 State

`stream_mode="updates"`에서는 Node 이름과 해당 Node가 변경한 State를 확인할 수 있다. 내부 실행을 추적하면 복잡한 Workflow에서 어느 Subgraph의 어느 Node가 현재 실행 중인지 파악할 수 있다.

## 설계 시 고려할 점

- 부모와 Subgraph 사이의 입력·출력 계약을 문서화한다.
- 공유 State를 무분별하게 늘리지 않고 필요한 필드만 노출한다.
- 서로 다른 State를 사용할 때 변환 책임을 래퍼 함수에 모은다.
- 내부 Graph의 오류를 부모 Graph에서 어떻게 처리할지 정한다.
- 스트리밍과 디버깅에 필요한 `namespace`와 실행 식별자를 보존한다.
