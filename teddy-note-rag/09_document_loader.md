# Chapter 09. 문서 로더

## 문서 로더의 구조 이해하기
- RAG 시스템의 8단계 중 첫 번째 단계로, 문서를 데이터베이스에서 불러오는 역할을 한다.
- LangChain은 다양한 로더들과 통합이 잘 이루어져 있어 PDF, 이미지, 표, JSON, HTML 등 여러 형식의 문서를 간단히 처리한다.

### Document 객체
```python
from langchain_core.documents import Document

doc = Document(
    id="policy-001",
    page_content="청년 월세 지원 대상은 만 19세부터 34세까지입니다.",
    metadata={
        "source": "youth_policy.pdf",
        "page": 3,
        "category": "주거지원",
    },
)

print(doc.page_content)
print(doc.metadata)
print(doc.id)

# 출력
# 청년 월세 지원 대상은 만 19세부터 34세까지입니다.

# {'source': 'youth_policy.pdf', 'page': 3, 'category': '주거지원'}

# policy-001
```
- LangChain의 기본 문서 객체이다.
- 문서 로더로 다양한 형식의 문서 파일을 불러오면 `Document` 객체 형태로 로드된다.
- 자신만의 로더를 구현하려면 반드시 문서를 `Document` 객체로 감싸 줘야 `LangChain`과 호환된다.
- `Document` 객체의 `page_content`에는 문서의 텍스트가 들어 있다.
- `metadata`에는 문서와 관련된 속성 값을 담고 있고, 필드들은 고정되어 있지 않으므로 개발자가 자유롭게 정의할 수 있다.

### 문서 로더 사용해 보기
```python
from langchain_community.document_loaders import PyPDFLoader

# 1. PDF를 페이지 단위로 읽는 로더 생성
loader = PyPDFLoader(
    file_path="./sample.pdf",
    mode="page",
)

# 2. load(): 모든 페이지를 Document 목록으로 로드
docs = loader.load()

print(f"문서 수: {len(docs)}")

for doc in docs:
    print(doc.page_content)
    print(doc.metadata)

from langchain_text_splitters import RecursiveCharacterTextSplitter

# 3. load_and_split(): 문서를 로드하고 청크로 분할
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,  # 청크의 최대 문자 수
    chunk_overlap=50,  # 인접 청크 사이에 겹치는 문자 수
)

docs = loader.load_and_split(
    text_splitter=splitter
)

print(f"청크 수: {len(docs)}")

for doc in docs:
    print(doc.page_content)

# 4. lazy_load(): 반복할 때 페이지를 하나씩 로드
docs = loader.lazy_load()

for doc in docs:
    print(doc.page_content)
    print(doc.metadata)

import asyncio

# 5. aload(): 비동기 방식으로 전체 문서 로드
async def main():
    docs = await loader.aload()

    print(f"문서 수: {len(docs)}")

    for doc in docs:
        print(doc.page_content)

asyncio.run(main())
```
- 문서 로더는 여러 가지 파일에서 불러온 내용을 Document 객체로 변환하는 역할을 한다.
- 종류는 `PyPDFLoader`, `CSVLoader`, `UnstructuredHTMLLoader`, `JSONLoader`, `TextLoader`, `DirectoryLoader` 등 문서의 형태에 따라 다양하다.
- `load()`: 문서 전체 내용을 로드해서 반환한다.
- `load_and_split()`: 문서를 로드한 뒤, 지정된 텍스트 분할기(text splitter)를 사용해 텍스트를 원하는 크기나 기준으로 나누는 작업을 자동으로 수행한다.
- `lazy_load()`:
    - 모든 페이지를 메모리에 즉시 로드하지 않고, 필요할 때 순차적으로 하나씩 로드한다.
    - 반복문이나 직접 호출을 통해 페이지별 데이터를 하나씩 처리할 수 있다.
    - 대량의 문서 데이터를 처리할 때 메모리 사용량을 크게 줄일 수 있다.
- `aload()`: 비동기(async) 방식으로 문서를 로드한다.


## PDF 로더
```python
from langchain_community.document_loaders import PyPDFLoader

# PDF 로더 생성
loader = PyPDFLoader("./sample.pdf")

# PDF 문서 로딩
docs = loader.load()

# 최대 10페이지, 페이지당 300자 출력
for i, doc in enumerate(docs[:10], start=1):
    print(f"\n===== {i}페이지 =====")
    print("문서 내용:")
    print(doc.page_content[:300])

    print("\n메타데이터:")
    print(doc.metadata)
```
- PDF 파일의 내용을 효율적으로 텍스트로 변환하거나 분리하여 다양한 작업에 활용한다.
- PDF 로더로는 PDFMiner, PDFPlumber, PyPDFium2, PyMuPDF 등 여러 가지가 있다. 다만 성능과 메타데이터 처리 방식에는 차이가 있다.

## HWP 로더
- 한글 파일은 국내 공공기관과 기업에서 널리 사용된다.
- HWP 파일은 파일 형식과 지원 범위에 맞는 전용 로더를 선택해야 한다.
- 이 예제에서는 자체 라이브러리(langchain_teddynote.document_loaders)에서 제공하는 로더를 사용하며, 코드는 생략한다.

## CSV 로더와 데이터프레임 로더
- CSV 파일은 CSVLoader로 쉽게 로드할 수 있다.
- 로드한 문서는 그대로 임베딩하지 않고 XML 형식으로 데이터를 구조화하면 검색 및 처리가 쉬워진다.
- XML 작업이 번거롭다면 UnstructuredCSVLoader를 대안으로 사용할 수 있다.
- Pandas를 사용한 데이터프레임은 DataFrameLoader를 통해 문서로 변환하면 LLM 기반의 검색 및 처리가 가능해진다.

### CSVLoader
```python
from langchain_community.document_loaders import CSVLoader

# sample.csv
# 이안;27;개발자
# 철수;30;디자이너
# 영희;25;기획자

loader = CSVLoader(
    file_path="./sample.csv",
    encoding="utf-8-sig",
    csv_args={
        "delimiter": ";",
        "quotechar": '"',
        "fieldnames": ["name", "age", "job"],
    },
)

docs = loader.load()

# 두 번째 행의 문서 내용 출력 (인덱스 1)
print(docs[1].page_content)

# 두 번째 행의 메타데이터 출력
print(docs[1].metadata)

# 출력
# name: 철수
# age: 30
# job: 디자이너

# {'source': './sample.csv', 'row': 1}
```

- CSV의 각 데이터 행을 하나의 `Document`로 변환하며, `page_content`에는 `열 이름: 값` 형식으로 내용을 저장한다.
- `metadata`에는 기본적으로 파일 경로인 `source`와 0부터 시작하는 행 번호인 `row`가 포함된다.
- `csv_args`로 CSV를 읽는 방식을 지정한다.
    - `delimiter`: 열을 구분하는 문자이다. 예제에서는 세미콜론(`;`)을 사용한다.
    - `quotechar`: 구분자가 포함된 값을 감싸는 인용 문자이다.
    - `fieldnames`: 헤더가 없는 파일의 열 이름을 지정한다. 헤더가 있는 파일에서는 생략해야 첫 행을 열 이름으로 읽는다.
- 예제는 세 행을 세 개의 문서로 변환하며, `docs[1]`은 두 번째 행인 철수의 정보를 담는다.

### UnstructuredCSVLoader
```python
from langchain_community.document_loaders.csv_loader import UnstructuredCSVLoader

# 1. CSV 로더 생성
loader = UnstructuredCSVLoader(
    file_path="./sample.csv",
    mode="elements",
)

# 2. 문서 로딩
docs = loader.load()

# 3. 문서 내용 출력 (최대 300자)
print("=== 문서 내용 ===")
print(docs[0].page_content[:300])

# 4. 메타데이터 출력
print("\n=== 메타데이터 ===")
print(docs[0].metadata)

# 5. HTML 테이블 출력
print("\n=== HTML 테이블 ===")
print(docs[0].metadata["text_as_html"])
```
- UnstructuredCSVLoader를 사용해 테이블 데이터를 로드할 수 있다.
- 의존성이 높은 라이브러리를 사용해야 하지만 XML 변환 작업을 줄일 수 있는 장점이 있다.
- `mode="elements"`에서는 CSV 테이블을 Table 요소로 로드한다. CSVLoader처럼 각 행을 개별 Document로 반환하는 방식과는 다르다.
- docs의 첫 번째 문서(docs[0])에서 메타데이터 키 "text_as_html"를 지정해 데이터를 HTML 테이블 형식으로 가져온다.

### DataFrameLoader
```python
import pandas as pd
from langchain_community.document_loaders import DataFrameLoader

# sample.csv
# name,age,job
# 이안,27,개발자
# 철수,30,디자이너
# 영희,25,기획자

# 1. CSV 파일을 DataFrame으로 로딩
df = pd.read_csv("./sample.csv", encoding="utf-8-sig")

# 2. DataFrameLoader 생성
loader = DataFrameLoader(
    data_frame=df,
    page_content_column="name",
)

# 3. Document 객체로 변환
docs = loader.load()

# 4. 첫 번째 문서 출력
print(docs[0].page_content)
print(docs[0].metadata)

# 출력
# 이안
# {'age': 27, 'job': '개발자'}
```
- CSV 데이터를 데이터프레임 형식으로 저장한다.
- Pandas 데이터프레임을 문서 형식으로 변환하려면 DataFrameLoader를 사용한다.
- 데이터프레임의 특정 컬럼을 페이지 내용으로 지정하고, 나머지 컬럼을 메타데이터로 변환한다.

## WebBaseLoader
```python
import bs4
from langchain_community.document_loaders import WebBaseLoader

loader = WebBaseLoader(
    web_paths=(
        "https://n.news.naver.com/article/437/0000378416",
    ),
    header_template={
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
                      "AppleWebKit/537.36 (KHTML, like Gecko) "
                      "Chrome/131.0.0.0 Safari/537.36",
        "Accept-Language": "ko-KR,ko;q=0.9,en-US;q=0.8",
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    },
    bs_kwargs={
        "parse_only": bs4.SoupStrainer(
            "div",
            attrs={
                "class": [
                    "newsct_article_article_body",
                    "media_end_head_title",
                ]
            },
        )
    },
)

# 문서 로딩
docs = loader.load()

# 헤드라인 및 본문 출력
print(docs[0].page_content)
```
- 웹페이지의 내용을 로드하는 데 사용되며, bs4(BeautifulSoup) 라이브러리를 활용해 웹 문서를 크롤링하여 가져올 수 있다.
- SSL 인증서 검증 활성화 설정: `loader.requests_kwargs = {"verify": True}`

## DirectoryLoader
- 디렉터리에 저장된 다양한 형식의 파일(PDF, Markdown, HTML 등)을 읽어와 document 객체로 변환한다.
- 와일드카드 패턴을 사용하면 특정 확장자를 가진 파일들을 한꺼번에 로드하거나 이름이 비슷한 파일들을 일괄적으로 가져올 수 있다.
- 파일 입출력(I/O)에 멀티스레딩을 활용해 로딩 속도를 향상시킬 수 있다.
- 코드와 같은 특정 파일 유형을 파싱하기 위해 사용자 정의 로더 클래스를 사용할 수 있다.

### DirectoryLoader
```python
from pathlib import Path
from langchain_community.document_loaders import DirectoryLoader, TextLoader

# 상위 폴더 및 모든 하위 폴더의 .md 파일 로드
loader = DirectoryLoader(
    path="../",
    glob="**/*.md",
    loader_cls=TextLoader,
    loader_kwargs={"encoding": "utf-8"},
)

docs = loader.load()

print(f"로드된 문서 수: {len(docs)}")

# 상위 폴더의 READ.md 파일 찾기
target = Path("../READ.md").resolve()

for doc in docs:
    if Path(doc.metadata["source"]).resolve() == target:
        print("\n=== READ.md 내용 ===")
        print(doc.page_content[:300])
        print("\n메타데이터:", doc.metadata)
        break
else:
    print("상위 폴더에서 READ.md 파일을 찾지 못했습니다.")
```
- `show_progress=True`를 지정하면 문서를 로드할 때 진행 상태 표시줄(progress bar)이 표시된다.
- `use_multithreading=True`를 추가하면 멀티스레딩 방식으로 여러 개의 스레드를 통해 더 빨리 로드할 수 있다.

### loader_cls 변경하기
```python
# TextLoader로 변경
from langchain_community.document_loaders import DirectoryLoader, TextLoader

loader = DirectoryLoader(
    path="../",
    glob="**/*.txt",
    loader_cls=TextLoader,
    loader_kwargs={"encoding": "utf-8"},
)

docs = loader.load()

print(docs[0].page_content[:300])
print(docs[0].metadata)

# PythonLoader로 변경
from langchain_community.document_loaders import DirectoryLoader, PythonLoader

loader = DirectoryLoader(
    path="../",
    glob="**/*.py",
    loader_cls=PythonLoader,
)

docs = loader.load()

print(docs[0].page_content[:300])
print(docs[0].metadata)
```

## UpstageDocumentParseLoader
```python
from langchain_upstage import UpstageDocumentParseLoader

# 1. 문서 로더 생성
loader = UpstageDocumentParseLoader(
    file_path="./sample.pdf",
    split="page",
    output_format="text",
    ocr="auto",
)

# 2. 문서 로딩
docs = loader.load()

# 3. 최대 10페이지, 페이지당 300자 출력
for i, doc in enumerate(docs[:10], start=1):
    print(f"\n===== {i}페이지 =====")
    print("문서 내용:")
    print(doc.page_content[:300])

    print("\n메타데이터:")
    print(doc.metadata)
```
- 문서의 구조를 분석하여 제목, 단락, 표, 이미지 등 다양한 요소를 자동으로 인식하고 추출한다.
- OCR 기능을 지원하여 이미지로 저장된 PDF 문서에서도 텍스트와 레이아웃 정보를 정밀하게 추출할 수 있다.
- Upstage 콘솔 웹사이트에서 API 키를 발급받아야 한다.

## LlamaParse
```python
from llama_parse import LlamaParse
from llama_index.core import SimpleDirectoryReader

# 1. LlamaParse 생성
parser = LlamaParse(
    result_type="markdown",
    language="ko",
    num_workers=8,
    verbose=True,
)

# 2. PDF 파일에 사용할 파서 지정
file_extractor = {
    ".pdf": parser
}

# 3. SimpleDirectoryReader로 PDF 로딩
reader = SimpleDirectoryReader(
    input_files=["./sample.pdf"],
    file_extractor=file_extractor,
)

docs = reader.load_data()

# 4. 최대 10개 문서, 문서당 300자 출력
for i, doc in enumerate(docs[:10], start=1):
    print(f"\n===== {i}번째 문서 =====")
    print("문서 내용:")
    print(doc.text[:300])

    print("\n메타데이터:")
    print(doc.metadata)
```
- PDF, 워드, 파워포인트, 엑셀 등 다양한 문서를 손쉽게 파싱하는 라이브러리이다.
- API 이용 비용과 무료 처리 한도는 사용 중인 요금제와 파싱 방식에 따라 확인한다.

### 멀티모달 모델로 파싱하기
```python
import os
from llama_parse import LlamaParse

# 1. 파싱 지침
parsing_instruction = """
이 문서는 한국어 PDF 문서입니다.

다음 지침에 따라 문서를 파싱하세요.
1. 모든 텍스트를 누락 없이 추출하세요.
2. 제목, 소제목, 문단 구조를 유지하세요.
3. 표는 행과 열의 관계를 유지하여 Markdown 표로 변환하세요.
4. 이미지, 차트, 그래프는 시각적 내용을 분석하고 설명하세요.
5. 숫자, 날짜, 단위 등은 원본 그대로 유지하세요.
6. 추출 결과는 Markdown 형식으로 작성하세요.
"""

# 2. GPT-4o 멀티모달 파서 생성
parser = LlamaParse(
    result_type="markdown",
    language="ko",
    num_workers=8,
    verbose=True,
    use_vendor_multimodal_model=True,
    vendor_multimodal_model_name="openai-gpt4o",
    vendor_multimodal_api_key=os.environ["OPENAI_API_KEY"],
    parsing_instruction=parsing_instruction,
)

# 3. PDF 파싱
docs = parser.load_data("./sample.pdf")

# 4. 파싱 결과 출력
for i, doc in enumerate(docs[:10], start=1):
    print(f"\n===== {i}번째 문서 =====")
    print(doc.text[:300])
    print(doc.metadata)
```
- LlamaParse는 마크다운 형식을 파싱하는 것뿐만 아니라 멀티모달 기능도 뛰어나다.
- 멀티모달에 활용하는 주요 매개변수:
    - use_vendor_multimodal_model: 멀티모달 모델 사용 여부를 지정한다. True로 설정하면 외부 벤더의 멀티모달 모델을 사용한다.
    - vendor_multimodal_model_name: 사용할 멀티모달 모델 이름을 지정한다.
    - vendor_multimodal_api_key: 멀티모달 모델 API 키를 지정한다.
    - result_type: 파싱 결과의 형식을 지정한다.
    - language: 문서의 언어를 지정한다.
    - skip_diagonal_text: 대각선 텍스트를 건너뛸지 여부를 결정한다.
    - page_separator: 페이지 구분자를 지정한다.
- parsing_instruction을 추가해 무엇을 분석하고 무엇을 추출해야 할지 명시할 수 있다.
