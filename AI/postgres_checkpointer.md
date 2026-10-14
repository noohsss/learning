# PostgreSQL Checkpointer와 Store

LangGraph의 기본 메모리 Checkpointer는 프로세스가 종료되면 데이터가 사라질 수 있다. PostgreSQL 기반 Checkpointer를 사용하면 대화 State와 실행 이력을 영속적으로 저장해 프로세스 재시작 후에도 같은 Thread를 이어갈 수 있다.

## Checkpointer

`PostgresSaver`는 Graph의 checkpoint를 PostgreSQL에 저장한다. Graph 실행 중 생성되는 State와 메타데이터, 노드 실행의 중간 기록을 데이터베이스에 보관하므로 대화 복원과 실패 지점 재개에 활용할 수 있다.

저장되는 주요 테이블은 다음과 같다.

| 테이블 | 역할 |
|---|---|
| `checkpoints` | checkpoint State와 메타데이터 저장 |
| `checkpoint_blobs` | 메시지 등 직렬화된 State 값 저장 |
| `checkpoint_writes` | 노드 실행 중 발생한 중간 결과 저장 |
| `checkpoint_migrations` | Checkpointer 스키마 변경 이력 관리 |

`setup()`은 필요한 테이블과 스키마를 준비한다. 애플리케이션 시작 시 연결 설정과 함께 실행하거나 배포 단계에서 별도로 관리할 수 있다.

## Thread ID

`thread_id`는 하나의 대화 또는 작업 실행 이력을 구분하는 식별자다. 같은 `thread_id`로 Graph를 실행하면 이전 State를 이어서 사용할 수 있고, 다른 ID를 사용하면 별도의 대화로 분리된다.

커널이나 애플리케이션 프로세스가 재시작되어도 PostgreSQL의 checkpoint는 남아 있으므로, Checkpointer를 다시 연결하고 기존 `thread_id`를 사용하면 대화를 복원할 수 있다.

## Store

Checkpointer가 Thread 단위의 실행 State를 저장한다면, `PostgresStore`는 여러 대화에서 공유할 수 있는 사용자·애플리케이션 데이터를 저장한다. 사용자 이름, 관심 분야, 선호 설정처럼 대화가 바뀌어도 유지되어야 하는 정보를 Store에 보관할 수 있다.

Store는 `namespace`와 `key`를 사용해 데이터를 구분한다.

- `namespace`: 데이터의 범주나 소유 영역
- `key`: 해당 범주 안의 개별 데이터 식별자
- `value`: JSON 형태로 저장되는 데이터

같은 namespace와 key로 저장하면 기존 데이터가 갱신되고, 연결을 닫아도 PostgreSQL에 저장된 값은 유지된다.

## Checkpointer와 Store의 구분

| 구분 | Checkpointer | Store |
|---|---|---|
| 저장 단위 | Thread의 실행 이력과 State | 사용자·애플리케이션 장기 데이터 |
| 식별 기준 | `thread_id`, checkpoint | `namespace`, `key` |
| 사용 목적 | 대화 이어가기, 재개, Time Travel | 프로필·선호·공유 데이터 |
| 수명 | 특정 실행 흐름 중심 | 여러 실행과 대화에서 공유 |

두 저장소는 같은 PostgreSQL을 사용할 수 있지만 저장 테이블과 책임은 별도로 관리된다. Graph 실행에서는 `configurable.thread_id`로 Checkpointer를 식별하고, `configurable.user_id` 같은 값으로 Store의 namespace를 결정할 수 있다.

## 연결과 운영

DB 연결 URI에는 사용자, 비밀번호, 호스트, 포트, 데이터베이스 이름이 포함된다. 비밀번호에 특수 문자가 있으면 URI에 맞게 인코딩해야 하며, 인증 정보는 환경 변수나 Secret Manager에 보관한다.

- 연결 풀과 연결 수 제한을 관리한다.
- 애플리케이션 종료 시 연결을 닫는다.
- Checkpointer와 Store의 Migration 상태를 배포 과정에서 관리한다.
- 사용자별 접근 권한과 데이터 보존 정책을 적용한다.
- 메시지와 State에 포함된 개인정보를 보호한다.
