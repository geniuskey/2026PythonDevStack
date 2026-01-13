# LLM 서비스 구축하기

## 목표

RAG (Retrieval-Augmented Generation) 기반 Q&A 서비스를 처음부터 구축하기

---

## 기술 스택

| 분야 | 도구 | 이유 |
|------|------|------|
| LLM 프레임워크 | LangChain | RAG 체인, 에이전트 |
| 벡터 DB | ChromaDB | 로컬 개발 용이 |
| 웹 프레임워크 | FastAPI | 비동기 API |
| LLM API | OpenAI | GPT-4 |
| 임베딩 | OpenAI Embeddings | text-embedding-3-small |
| 문서 로더 | LangChain | PDF, Markdown 등 |

---

## 아키텍처

```
사용자 질문
    ↓
1. 질문 임베딩
    ↓
2. 벡터 DB에서 관련 문서 검색
    ↓
3. 컨텍스트 + 질문 → LLM
    ↓
4. 답변 생성
    ↓
5. 출처와 함께 반환
```

---

## 1단계: 프로젝트 설정

```bash
# 프로젝트 생성
$ uv init llm-service
$ cd llm-service

# 의존성 추가
$ uv add fastapi "uvicorn[standard]" \
         langchain langchain-openai \
         chromadb tiktoken \
         pydantic-settings

# 개발 의존성
$ uv add --dev pytest httpx
```

### 디렉토리 구조

```
llm-service/
├── src/
│   └── llmservice/
│       ├── __init__.py
│       ├── main.py
│       ├── config.py
│       ├── vectorstore.py
│       ├── chain.py
│       └── models.py
├── data/
│   └── documents/
│       └── docs.txt
├── chroma_db/            # 벡터 DB 저장소
├── tests/
└── pyproject.toml
```

---

## 2단계: 설정

```python
# src/llmservice/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    openai_api_key: str

    # LLM 설정
    llm_model: str = "gpt-4-turbo-preview"
    llm_temperature: float = 0.0

    # 임베딩 설정
    embedding_model: str = "text-embedding-3-small"

    # 벡터 DB
    chroma_persist_directory: str = "./chroma_db"

    # RAG 설정
    chunk_size: int = 1000
    chunk_overlap: int = 200
    retrieval_top_k: int = 3

    class Config:
        env_file = ".env"

settings = Settings()
```

---

## 3단계: 벡터 저장소 초기화

```python
# src/llmservice/vectorstore.py
from langchain.document_loaders import DirectoryLoader, TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain.vectorstores import Chroma

from llmservice.config import settings

def load_documents(directory: str = "./data/documents"):
    """문서 로드"""
    loader = DirectoryLoader(
        directory,
        glob="**/*.txt",
        loader_cls=TextLoader
    )
    documents = loader.load()
    return documents

def split_documents(documents):
    """문서 청크 분할"""
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=settings.chunk_size,
        chunk_overlap=settings.chunk_overlap,
        length_function=len,
    )
    splits = text_splitter.split_documents(documents)
    return splits

def create_vectorstore():
    """벡터 저장소 생성"""
    # 문서 로드 및 분할
    documents = load_documents()
    splits = split_documents(documents)

    # 임베딩 생성
    embeddings = OpenAIEmbeddings(
        model=settings.embedding_model,
        openai_api_key=settings.openai_api_key
    )

    # 벡터 저장소 생성
    vectorstore = Chroma.from_documents(
        documents=splits,
        embedding=embeddings,
        persist_directory=settings.chroma_persist_directory
    )

    return vectorstore

def get_vectorstore():
    """기존 벡터 저장소 로드"""
    embeddings = OpenAIEmbeddings(
        model=settings.embedding_model,
        openai_api_key=settings.openai_api_key
    )

    vectorstore = Chroma(
        persist_directory=settings.chroma_persist_directory,
        embedding_function=embeddings
    )

    return vectorstore
```

---

## 4단계: RAG 체인 구성

```python
# src/llmservice/chain.py
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.schema.output_parser import StrOutputParser
from langchain.schema.runnable import RunnablePassthrough

from llmservice.config import settings
from llmservice.vectorstore import get_vectorstore

# 프롬프트 템플릿
PROMPT_TEMPLATE = """
당신은 문서를 바탕으로 정확하게 답변하는 AI 어시스턴트입니다.

다음 문서 컨텍스트를 바탕으로 질문에 답변하세요:

{context}

질문: {question}

답변 시 주의사항:
1. 제공된 컨텍스트를 기반으로만 답변하세요
2. 컨텍스트에 정보가 없으면 "제공된 문서에서 해당 정보를 찾을 수 없습니다"라고 답하세요
3. 간결하고 명확하게 답변하세요
"""

def format_docs(docs):
    """문서를 컨텍스트 문자열로 변환"""
    return "\n\n".join([doc.page_content for doc in docs])

def create_rag_chain():
    """RAG 체인 생성"""
    # LLM 초기화
    llm = ChatOpenAI(
        model=settings.llm_model,
        temperature=settings.llm_temperature,
        openai_api_key=settings.openai_api_key
    )

    # 벡터 저장소
    vectorstore = get_vectorstore()
    retriever = vectorstore.as_retriever(
        search_kwargs={"k": settings.retrieval_top_k}
    )

    # 프롬프트
    prompt = ChatPromptTemplate.from_template(PROMPT_TEMPLATE)

    # 체인 구성
    chain = (
        {"context": retriever | format_docs, "question": RunnablePassthrough()}
        | prompt
        | llm
        | StrOutputParser()
    )

    return chain, retriever

async def ask_question(question: str):
    """질문에 답변"""
    chain, retriever = create_rag_chain()

    # 관련 문서 검색
    docs = retriever.get_relevant_documents(question)

    # LLM 호출
    answer = await chain.ainvoke(question)

    # 출처 정보 추가
    sources = [
        {
            "content": doc.page_content[:200],
            "metadata": doc.metadata
        }
        for doc in docs
    ]

    return {
        "question": question,
        "answer": answer,
        "sources": sources
    }
```

---

## 5단계: FastAPI 서버

```python
# src/llmservice/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

from llmservice.chain import ask_question
from llmservice.vectorstore import create_vectorstore

app = FastAPI(title="LLM Q&A Service")

class Question(BaseModel):
    text: str

class Answer(BaseModel):
    question: str
    answer: str
    sources: list[dict]

@app.post("/ask", response_model=Answer)
async def ask(question: Question):
    """질문에 답변"""
    try:
        result = await ask_question(question.text)
        return Answer(**result)
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/index")
async def index_documents():
    """문서 인덱싱"""
    try:
        create_vectorstore()
        return {"status": "success", "message": "Documents indexed"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    return {"status": "ok"}
```

---

## 6단계: 사용

### 문서 준비

```bash
# data/documents/guide.txt
Python은 1991년 귀도 반 로섬이 만든 프로그래밍 언어입니다.
간결하고 읽기 쉬운 문법이 특징입니다.
데이터 과학, 웹 개발, 자동화 등 다양한 분야에서 사용됩니다.
```

### 환경 변수

```bash
# .env
OPENAI_API_KEY=sk-...your-api-key...
```

### 서버 실행

```bash
# 1. 문서 인덱싱
$ uv run python -c "from llmservice.vectorstore import create_vectorstore; create_vectorstore()"

# 2. 서버 시작
$ uv run uvicorn llmservice.main:app --reload

# 3. 질문하기
$ curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"text": "Python은 누가 만들었나요?"}'
```

---

## 7단계: 고급 기능

### 1. 스트리밍 응답

```python
from fastapi.responses import StreamingResponse

@app.post("/ask/stream")
async def ask_stream(question: Question):
    async def generate():
        chain, _ = create_rag_chain()
        async for chunk in chain.astream(question.text):
            yield chunk

    return StreamingResponse(generate(), media_type="text/plain")
```

### 2. 대화 메모리

```python
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory()

# 체인에 메모리 추가
chain_with_memory = ConversationalRetrievalChain.from_llm(
    llm=llm,
    retriever=retriever,
    memory=memory
)
```

### 3. 비용 추적

```python
import tiktoken

def count_tokens(text: str, model: str = "gpt-4") -> int:
    encoding = tiktoken.encoding_for_model(model)
    return len(encoding.encode(text))

# 사용
input_tokens = count_tokens(question)
output_tokens = count_tokens(answer)
total_cost = (input_tokens * 0.03 + output_tokens * 0.06) / 1000
```

---

## 프로덕션 체크리스트

- [ ] API 키 보안 (환경 변수, Secrets Manager)
- [ ] Rate limiting
- [ ] 캐싱 (동일 질문 반복 방지)
- [ ] 로깅 (질문, 답변, 토큰 수)
- [ ] 에러 핸들링
- [ ] 모니터링 (Langsmith, Helicone)
- [ ] 프롬프트 버전 관리
- [ ] A/B 테스팅
- [ ] 평가 (정확도, 관련성)
- [ ] 벡터 DB 백업

---

**[← 패키지 배포](package-deployment.md)** | **[데이터 파이프라인 →](data-pipeline.md)**
