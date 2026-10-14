# Parent-Child RAG

Parent-Child RAG는 검색 단위와 답변에 전달할 문맥 단위를 분리하는 검색 구조다. 작은 Child Chunk로 관련 내용을 정밀하게 찾고, 해당 Child가 속한 Parent Section 전체를 가져와 답변에 사용한다.

```text
질문 → Child 벡터 검색 → parent_id 수집 → Parent 조회 → 답변 생성
```

## 검색과 문맥의 균형

작은 청크는 질문과 관련된 부분을 정밀하게 찾는 데 유리하지만, 너무 작으면 문맥이 부족하다. 큰 청크는 문맥을 보존하지만 관련 없는 내용이 많이 포함되어 검색 정밀도가 낮아질 수 있다.

Parent-Child 구조는 검색에는 작은 단위를 사용하고 답변에는 의미 단위 전체를 사용해 이 균형을 조정한다.

## 저장 구조

크롤링 원문과 검색용 청크를 서로 다른 저장소에 보관할 수 있다.

| 저장소 | 내용 | 연결 키 |
|---|---|---|
| PostgreSQL `crawled_documents` | URL, 제목, 원문 HTML, 수집 시각 | `id` |
| PostgreSQL `rag_parent_chunks` | Section 제목과 본문 | `parent_id`, `document_id` |
| Chroma | Parent 제목을 포함한 Child와 벡터 | `parent_id` |

원문 HTML을 보존하면 나중에 청크 크기나 Section 기준을 바꿔 다시 생성할 수 있다. Parent는 문서의 Section처럼 의미가 있는 범위로 정하고, Child는 Parent 본문을 문자 수나 토큰 수 기준으로 나눈다.

## Section 기반 Parent

HTML 문서에서는 최상위 Section을 Parent로 사용하고 하위 헤딩은 같은 Parent에 포함할 수 있다. 첫 헤딩 앞의 본문은 도입부로 별도 구분하며, 각주·참고문헌·외부 링크처럼 검색에 불필요한 영역은 제외할 수 있다.

Parent 제목에는 문서 제목과 Section 제목을 함께 기록하면 검색 문맥을 보강할 수 있다. Child의 임베딩 입력에도 이 제목을 붙이면 제목의 의미가 검색에 반영된다.

## Child 메타데이터

각 Child에는 연결된 Parent를 찾을 수 있도록 `parent_id`를 메타데이터로 저장한다. 메타데이터는 검색 결과와 함께 반환되지만 임베딩 입력에 자동으로 포함되는 것은 아니므로, 검색 의미에 반영할 제목은 본문에 명시적으로 포함해야 한다.

## 검색 후 Parent 조회

Child 검색 결과에서 `parent_id`를 추출하고 중복을 제거한다. 이후 PostgreSQL에서 Parent를 한 번에 조회하고, 검색 결과의 순서를 기준으로 다시 정렬해 답변 Context를 구성한다.

`max_parents`는 답변에 전달할 Parent 수의 상한이다. 여러 Child가 같은 Parent에 속하면 실제 Parent 수는 검색된 Child 수보다 적을 수 있다.

## 컬렉션과 재생성

청크 크기, overlap, 제목 포함 방식, 임베딩 모델이 바뀌면 기존 벡터와 의미가 달라질 수 있다. 변경된 설정은 새로운 collection 이름으로 구분하거나 기존 데이터를 재생성해야 한다.

원문 저장과 Parent·Child 생성 단계를 분리하면 원문을 다시 크롤링하지 않고도 다양한 청킹 설정을 비교할 수 있다. 같은 Child ID를 다시 저장하지 않도록 중복을 관리하는 것도 필요하다.

## RDB와 벡터 DB의 역할

- 벡터 DB: 자연어 질문과 의미적으로 가까운 Child 검색
- RDB: Parent 본문, URL, 제목, 수집 시각 등 구조화된 원문 조회
- LLM: 검색된 Parent 문맥을 바탕으로 답변 생성

벡터 검색 결과만으로 답변하지 않고 Parent를 원문 DB에서 다시 조회하면 문맥 범위와 출처 정보를 관리하기 쉽다.

## 설계 기준

- Parent는 문서의 의미 단위, Child는 검색에 적합한 크기로 정한다.
- Parent와 Child 사이의 연결 키를 안정적으로 관리한다.
- 검색 순서를 보존하면서 중복 Parent를 제거한다.
- 답변 Context의 Parent 수와 전체 토큰 양을 제한한다.
- 원문, 청킹 설정, 임베딩 모델의 변경 이력을 관리한다.
- URL과 수집 시각을 보존해 출처와 최신성을 확인한다.
