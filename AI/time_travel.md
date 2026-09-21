# LangGraph Time Travel

LangGraph의 Time Travel은 Checkpointer에 저장된 과거 실행 상태를 조회하고, 특정 시점부터 실행을 재개하거나 State를 변경해 새로운 실행 경로를 만드는 기능이다. 실행 결과를 단순히 덮어쓰는 것이 아니라 checkpoint와 부모 관계를 보존하므로, 실행 이력을 추적하고 여러 결과를 비교할 수 있다.

## Checkpoint와 State

Checkpointer는 그래프가 실행되는 동안 각 단계의 State를 저장한다. 하나의 실행 이력은 `thread_id`로 구분하며, 해당 이력 안의 각 시점은 `checkpoint_id`로 구분한다.

```text
thread_id: 하나의 대화·작업 실행 이력
checkpoint_id: 해당 실행 이력의 특정 State 시점
StateSnapshot: 특정 시점의 값과 다음 실행 정보를 가진 snapshot
```

StateSnapshot에는 일반적으로 다음 정보가 포함된다.

- `values`: 해당 시점에 저장된 State 값
- `next`: 해당 State에서 다음에 실행할 Node
- `config`: 현재 checkpoint를 다시 가리키는 설정
- `parent_config`: 현재 checkpoint가 파생된 부모 checkpoint 정보

`get_state(config)`는 특정 thread의 최신 State를 조회하고, `get_state_history(config)`는 해당 thread에 저장된 checkpoint 목록을 최신순으로 반환한다.

## Replay

Replay는 과거 checkpoint의 State를 변경하지 않고, 선택한 시점부터 그래프를 다시 실행하는 방식이다. 과거 checkpoint의 `config`를 사용해 입력 없이 실행을 재개하면 해당 State의 `next`에 지정된 Node부터 진행된다.

```text
A → B → C → D
    ↘ Replay → C' → D'
```

Replay의 특징은 다음과 같다.

- 선택한 과거 State의 값은 그대로 유지
- 이미 완료된 이전 Node는 다시 실행하지 않음
- 기존 checkpoint는 삭제하거나 덮어쓰지 않음
- 선택한 checkpoint를 부모로 하는 새로운 실행 경로 생성
- 같은 State 값이라도 새 실행에는 새로운 `checkpoint_id` 부여

Replay는 특정 단계부터 동일한 조건으로 재실행하거나, 장애 지점 이후의 동작을 재현할 때 적합하다.

## Fork

Fork는 과거 checkpoint의 State를 수정한 뒤, 수정된 값을 기준으로 새로운 실행 경로를 만드는 방식이다. `update_state()`는 기존 checkpoint를 보존하면서 변경된 State를 가진 새 checkpoint를 생성한다.

```text
A → B → C → D
    ↘ U → C' → D'
      State 수정
```

Fork의 흐름은 다음과 같다.

1. 원하는 과거 checkpoint를 선택한다.
2. `update_state()`로 State의 일부를 변경한다.
3. 변경된 값을 가진 새 checkpoint의 설정을 받는다.
4. 새 checkpoint부터 그래프 실행을 재개한다.

`as_node`을 지정하면 변경한 State를 특정 Node가 작성한 결과처럼 처리할 수 있다. 이를 통해 해당 Node 다음의 경로부터 실행을 이어갈 수 있다.

## Replay와 Fork 비교

| 구분 | Replay | Fork |
|---|---|---|
| 과거 State 변경 | 변경하지 않음 | 일부 값을 변경함 |
| 실행 시작점 | 선택한 checkpoint의 `next` | 수정된 checkpoint의 `next` |
| 목적 | 같은 조건의 재실행 | 수정된 조건의 대안 경로 생성 |
| 기존 경로 | 유지 | 유지 |
| 활용 예 | 장애 재현, 동일 결과 재생성 | 수정 반영, 여러 결과 비교 |

## 실행 경로와 부모 관계

Time Travel은 checkpoint를 시간순 목록으로만 저장하지 않는다. 각 checkpoint는 `parent_config`를 통해 자신이 어느 checkpoint에서 파생되었는지 가리킨다. 따라서 Replay와 Fork로 분기된 실행 경로도 추적할 수 있다.

```text
저장 순서: A, B, C, D, U, C', D'

원래 경로: A → B → C → D
분기 경로:       └→ U → C' → D'
```

`thread_id`만 사용해 최신 State를 조회하면 해당 thread에서 가장 최근에 생성된 checkpoint가 반환된다. 특정 과거 경로를 다시 사용하려면 `checkpoint_id`가 포함된 checkpoint의 `config`를 보존해야 한다.

## 활용 사례

### 결과 재생성

서비스에서 사용자가 이전 답변에 대해 “다시 생성”을 요청하면, 선택한 checkpoint에서 Replay를 수행할 수 있다.

### 중간 결과 수정

사용자가 초안이나 검색 조건을 수정한 뒤 “이 내용으로 다시 작성”을 요청하면, 해당 State를 변경한 Fork 경로를 만들 수 있다.

### 장애 재현

오류가 발생한 실행의 직전 checkpoint를 선택하면 전체 작업을 처음부터 수행하지 않고 문제 지점부터 재현할 수 있다.

### 결과 비교

같은 중간 State에서 Prompt, 모델, 검색 조건, 사용자 입력을 다르게 적용해 여러 결과를 비교할 수 있다.

### 비용 절감

앞부분에서 수행한 검색이나 분석 결과를 보존하고 이후 단계만 다시 실행하면, 이미 완료된 비싼 작업을 반복하지 않을 수 있다.

## 외부 작업과 멱등성

Time Travel은 그래프의 State와 실행 경로를 되돌리는 기능이지, 외부 시스템의 상태를 되돌리는 기능이 아니다. 이미 전송된 메일, 처리된 결제, 발행된 게시글, 저장된 데이터는 과거 checkpoint로 이동해도 자동으로 취소되지 않는다.

따라서 외부 작업을 포함하는 그래프에서는 다음을 고려해야 한다.

- 동일 요청을 여러 번 처리해도 결과가 중복되지 않는 멱등성 설계
- 외부 작업 전용 실행 ID와 중복 처리 방지 키
- Replay·Fork 시 외부 작업을 재실행할지 결정하는 정책
- 실제 부작용이 발생하는 Node와 순수 계산 Node의 분리
- 외부 작업 전 승인이나 별도의 보상 트랜잭션

## 운영 시 고려사항

### Checkpointer 저장소

메모리 기반 Checkpointer는 개발과 짧은 테스트에 적합하지만 프로세스가 종료되면 데이터가 사라질 수 있다. 운영 환경에서는 영속 저장소와 보존 기간, 접근 권한, 삭제 정책을 함께 설계해야 한다.

### State 크기

메시지와 검색 문서를 State에 모두 저장하면 checkpoint 크기가 커진다. 재실행에 필요한 최소 정보와 외부 저장소의 참조 ID를 구분하는 것이 좋다.

### 민감 정보

checkpoint에는 사용자 입력, 도구 결과, 모델 응답이 남을 수 있다. 개인정보와 비밀 정보의 마스킹, 접근 제어, 암호화, 보존 기간을 고려해야 한다.

### 경로 식별

사용자에게 내부 `checkpoint_id`를 직접 노출하기보다, 버전·시점·작업 단계처럼 이해하기 쉬운 식별자를 별도로 제공하는 편이 좋다.
