# PostgreSQL을 활용하는 Agent

PostgreSQL을 Agent의 조회 도구와 연결하면 자연어 질문으로 관계형 데이터를 조회할 수 있다. 다만 LLM의 Tool 선택과 SQL 생성을 그대로 신뢰하지 않고, DB 권한·읽기 전용 연결·SQL 검사 계층을 함께 적용해야 한다.

## 최소 권한 DB 계정

Agent용 DB 계정은 필요한 데이터베이스와 테이블에 필요한 권한만 가져야 한다. 조회 Agent라면 `CONNECT`, 스키마 `USAGE`, 대상 테이블의 `SELECT`만 부여하고 `INSERT`, `UPDATE`, `DELETE` 권한은 부여하지 않는다.

```sql
GRANT CONNECT ON DATABASE world TO agent_reader;
GRANT USAGE ON SCHEMA public TO agent_reader;
GRANT SELECT ON country, city, countrylanguage TO agent_reader;
```

DB 권한은 첫 번째 안전장치다. 코드의 검사에 오류가 있어도 Agent 계정이 변경 권한을 갖지 않으면 데이터 변조 범위를 줄일 수 있다.

## 고정 SQL Tool

고정 SQL Tool은 개발자가 SQL 구조를 미리 작성하고 LLM은 Tool과 인자만 선택하는 방식이다. SQL의 구조와 접근 테이블이 제한되므로 보안과 예측 가능성이 높다.

값은 SQL 문자열에 직접 연결하지 않고 파라미터 바인딩으로 전달한다.

```python
query = "SELECT code, name FROM country WHERE name ILIKE %s"
params = ("%Korea%",)
```

파라미터 바인딩은 SQL Injection을 줄이고 값과 SQL 구조를 분리한다. 검색 기능이 제한적이거나 업무 규칙이 명확한 경우 고정 SQL Tool이 적합하다.

## 읽기 전용 실행

DB 계정 권한과 별도로 연결을 읽기 전용 트랜잭션으로 설정하면 이중 방어가 된다. 계정에 실수로 변경 권한이 추가되더라도 해당 연결에서 데이터 변경을 차단할 수 있다.

조회 결과는 튜플이나 컬럼명을 Key로 가진 딕셔너리 형태로 받을 수 있다. Agent에 전달할 결과의 크기, 정렬, 민감 컬럼 노출 범위도 제한해야 한다.

## Text-to-SQL

Text-to-SQL은 자연어 질문과 DB 스키마를 바탕으로 LLM이 SQL을 생성하는 방식이다.

```text
스키마 조회 → SQL 생성 → SQL 검사 → 읽기 전용 실행 → 결과 기반 답변
```

고정 SQL Tool보다 다양한 집계·조인 질문에 대응할 수 있지만, 잘못된 컬럼명이나 의도하지 않은 변경 SQL을 생성할 수 있다. 따라서 SQL 생성과 실행을 분리하고 실행 전 검사를 둬야 한다.

## SQL 검사

SQL 검사기는 SQL 문자열의 명령과 구조를 분석해 허용 범위를 확인한다. 단순히 문자열이 `SELECT`로 시작하는지만 확인하면 여러 문장, `WITH` 내부의 변경 명령, `SELECT INTO` 같은 우회가 남을 수 있다.

조회 Agent에서는 다음을 제한할 수 있다.

- SELECT 문 한 개만 허용
- INSERT·UPDATE·DELETE 차단
- CREATE·ALTER·DROP 차단
- `SELECT INTO` 차단
- 허용된 테이블만 접근
- 실행 시간과 반환 행 수 제한

SQLGlot 같은 파서를 사용하면 SQL의 AST를 분석해 명령 종류와 내부 구문을 확인할 수 있다.

## 오류 재생성

SQL 실행 오류가 발생하면 오류 메시지와 스키마, 이전 SQL을 LLM에 전달해 SQL을 수정할 수 있다. 재시도 횟수는 제한하고, 계속 실패하면 결과를 추측하지 않고 오류를 안내해야 한다.

## 고정 SQL과 Text-to-SQL 비교

| 구분 | 고정 SQL Tool | Text-to-SQL |
|---|---|---|
| SQL 작성 | 개발자가 미리 작성 | LLM이 질문마다 생성 |
| LLM이 결정하는 값 | Tool과 인자 | 컬럼, 조건, JOIN, 집계 |
| 유연성 | 제한적 | 높음 |
| 통제 난이도 | 낮음 | 높음 |
| 적합한 경우 | 정해진 업무 조회 | 다양한 분석·집계 질문 |

## 보안 설계 기준

- Agent 전용 최소 권한 계정을 사용한다.
- 파라미터 바인딩으로 값을 전달한다.
- 실행 전 SQL을 파싱하고 허용 명령만 통과시킨다.
- 읽기 전용 트랜잭션을 사용한다.
- 허용 테이블과 컬럼을 제한한다.
- 반환 결과의 행 수와 민감 정보 노출을 제한한다.
- 생성된 SQL, 실행 오류, 최종 답변을 함께 기록한다.
