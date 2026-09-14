# RAG Agent

## RAG Agent란

Retriever를 Agent의 Tool로 등록하면 대화형 RAG를 만들 수 있다. 기본 RAG는 질문마다 검색하지만, RAG Agent는 LLM이 질문을 보고 검색 필요 여부를 판단한다.

| 구분 | 기본 RAG | RAG Agent |
|---|---|---|
| 검색 판단 | 항상 검색 | LLM이 필요 여부 판단 |
| 검색 호출 | 애플리케이션이 직접 호출 | Retriever Tool 호출 |
| 대화 기억 | 별도 구현 | Checkpointer 연결 가능 |

## Retriever를 Tool로 변환

create_retriever_tool()은 Retriever의 invoke(query)를 Agent가 호출할 수 있는 Tool로 바꾼다. Tool description은 검색 대상과 호출 조건을 설명하므로 구체적으로 작성해야 한다.

~~~python
retriever_tool = create_retriever_tool(
    retriever,
    name="search_company_rules",
    description=(
        "회사 내부 규정에서 근무 시간, 휴가, 복리후생, 출장비, "
        "보안 및 인사 정책을 검색한다. "
        "회사 규정이나 사내 정책에 관한 질문을 받았을 때 사용한다."
    ),
    document_prompt=document_prompt,
    document_separator="\n\n---\n\n",
)
~~~

Retriever는 Document 목록을 반환하지만 Tool은 검색 문서를 문자열로 합쳐 Agent에 전달한다. metadata를 답변에 사용하려면 document_prompt에 명시한다.

~~~python
document_prompt = PromptTemplate.from_template(
    "[규정명: {policy_name}]\n"
    "[출처: {source}]\n"
    "{page_content}"
)
~~~

document_prompt에 사용한 metadata 키는 모든 검색 Document에 존재해야 한다.

## Agent 구성

~~~python
llm = ChatGoogleGenerativeAI(model=MODEL_NAME)
graph = create_agent(model=llm, tools=[retriever_tool])
~~~

회사 규정 질문은 검색 Tool을 호출하고, 인사말처럼 검색이 필요하지 않은 질문은 Tool 호출 없이 처리할 수 있다.

## 문서 기반 답변 제한

검색 결과가 부족할 때 일반 지식으로 답하지 않게 하려면 System Prompt에 문서 기반 답변 정책을 명시한다.

~~~python
strict_rag_agent = create_agent(
    model=llm,
    tools=[retriever_tool],
    system_prompt="""
사내 규정과 관련된 질문에는 search_company_rules 도구를 반드시 사용한다.
답변은 검색된 사내 규정만 근거로 작성하고, 참고한 규정명과 출처를 포함한다.
검색 결과에 질문에 답할 충분한 근거가 없으면
'해당 정보를 사내 규정에서 찾을 수 없습니다.'라고 답한다.
문서에 없는 내용을 추측하거나 일반 지식으로 보완하지 않는다.
""",
)
~~~

개방형 RAG는 문서와 일반 지식을 함께 활용할 수 있다. 사내 규정이나 고객 응대처럼 승인된 문서만 근거로 해야 하는 경우에는 제한형 RAG가 적합하다.

## 대화형 RAG

~~~python
rag_agent = create_agent(
    model=llm,
    tools=[retriever_tool],
    checkpointer=MemorySaver(),
)
~~~

같은 thread_id를 사용하면 첫 검색 결과를 바탕으로 후속 질문을 처리할 수 있다.

~~~text
재택근무 가능 횟수 → 검색 Tool 호출
가장 중요한 것 하나 → 이전 답변에서 선별
영어로 번역해줘     → 대화 맥락으로 처리
~~~

## PDF 기반 RAG Agent

PDF를 PyPDFLoader로 읽고 페이지 metadata에 source, year, month를 저장하면 월호나 출처를 이용한 후속 질문에 대응할 수 있다. 청크마다 chunk_id를 부여하면 검색 결과를 추적하기 쉽다.

~~~text
PDF 로드 → 페이지 metadata 추가 → 청크 분할 → chunk_id 부여
        → Chroma 저장 → Retriever Tool → RAG Agent
~~~

## 주의할 점

- Retriever Tool description이 모호하면 Agent가 검색을 건너뛸 수 있다.
- metadata를 답변에 사용하려면 document_prompt에 포함해야 한다.
- 검색 결과가 부족할 때 추측하지 않도록 답변 정책을 정해야 한다.
- thread_id가 달라지면 대화 State가 분리된다.
- 검색 결과와 답변에 출처를 포함하면 검증이 쉬워진다.
