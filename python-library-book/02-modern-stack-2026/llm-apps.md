# LLM & AI 애플리케이션

## 2026 표준 스택

| 도구 | 등급 | 역할 |
|------|------|------|
| `langchain` / `llamaindex` | - 2026 권장: | LLM 애플리케이션 프레임워크 |
| `chromadb` / `pinecone` | - 2026 권장: | 벡터 데이터베이스 |
| `openai` / `anthropic` | - 2026 권장: | LLM API 클라이언트 |
| `tiktoken` | - 2026 권장: | 토큰 카운팅 |

---

## 1. LangChain: LLM 프레임워크 ⭐

### 핵심 기능

- **체인**: 여러 LLM 호출 연결
- **RAG**: 문서 검색 + 생성
- **에이전트**: 자율 행동 LLM
- **메모리**: 대화 컨텍스트 관리

### 기본 사용

```python
from langchain.chat_models import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.schema.output_parser import StrOutputParser

# LLM 초기화
llm = ChatOpenAI(model="gpt-4", temperature=0)

# 프롬프트 템플릿
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    ("user", "{input}")
])

# 체인 구성
chain = prompt | llm | StrOutputParser()

# 실행
result = chain.invoke({"input": "What is Python?"})
print(result)
```

### RAG (Retrieval-Augmented Generation)

```python
from langchain.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Chroma
from langchain.chains import RetrievalQA

# 1. 문서 로드
loader = TextLoader("docs.txt")
documents = loader.load()

# 2. 청크 분할
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200
)
splits = text_splitter.split_documents(documents)

# 3. 벡터 저장소 생성
vectorstore = Chroma.from_documents(
    documents=splits,
    embedding=OpenAIEmbeddings()
)

# 4. RAG 체인
qa_chain = RetrievalQA.from_chain_type(
    llm=ChatOpenAI(model="gpt-4"),
    retriever=vectorstore.as_retriever()
)

# 질문
result = qa_chain.invoke("What is the main topic?")
```

---

## 2. 벡터 데이터베이스

### ChromaDB (로컬) ⭐

```python
import chromadb
from chromadb.utils import embedding_functions

# 클라이언트 생성
client = chromadb.Client()

# 컬렉션 생성
collection = client.create_collection(
    name="my_docs",
    embedding_function=embedding_functions.OpenAIEmbeddingFunction()
)

# 문서 추가
collection.add(
    documents=["Python is a programming language", "FastAPI is a web framework"],
    ids=["1", "2"]
)

# 검색
results = collection.query(
    query_texts=["web development"],
    n_results=2
)
```

### Pinecone (클라우드) ⭐

```python
import pinecone
from openai import OpenAI

# 초기화
pinecone.init(api_key="your-api-key")
index = pinecone.Index("my-index")

# 임베딩 생성
client = OpenAI()
embedding = client.embeddings.create(
    input="Python programming",
    model="text-embedding-3-small"
).data[0].embedding

# 저장
index.upsert(vectors=[("id1", embedding, {"text": "Python programming"})])

# 검색
results = index.query(vector=embedding, top_k=5, include_metadata=True)
```

---

## 3. 실전 예시: Q&A 시스템

```python
from fastapi import FastAPI
from pydantic import BaseModel
from langchain.chat_models import ChatOpenAI
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings

app = FastAPI()

# 벡터 저장소 로드
vectorstore = Chroma(
    persist_directory="./chroma_db",
    embedding_function=OpenAIEmbeddings()
)

class Question(BaseModel):
    text: str

@app.post("/ask")
async def ask_question(question: Question):
    # 관련 문서 검색
    docs = vectorstore.similarity_search(question.text, k=3)

    # 컨텍스트 구성
    context = "\n".join([doc.page_content for doc in docs])

    # LLM 호출
    llm = ChatOpenAI(model="gpt-4")
    prompt = f"""
    Based on the following context, answer the question.

    Context:
    {context}

    Question: {question.text}

    Answer:
    """

    answer = llm.invoke(prompt).content

    return {
        "question": question.text,
        "answer": answer,
        "sources": [doc.metadata for doc in docs]
    }
```

---

## 4. 토큰 관리

```python
import tiktoken

# 토큰 카운팅
encoding = tiktoken.encoding_for_model("gpt-4")

text = "Hello, how are you?"
tokens = encoding.encode(text)
print(f"Token count: {len(tokens)}")

# 토큰 제한
def truncate_to_tokens(text: str, max_tokens: int) -> str:
    tokens = encoding.encode(text)
    if len(tokens) > max_tokens:
        tokens = tokens[:max_tokens]
    return encoding.decode(tokens)
```

---

## 5. 비용 최적화

```python
from functools import lru_cache

# 캐싱
@lru_cache(maxsize=1000)
def get_embedding(text: str):
    return openai.Embedding.create(input=text, model="text-embedding-3-small")

# 배치 처리
def batch_embed(texts: list[str], batch_size: int = 100):
    results = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i+batch_size]
        embeddings = openai.Embedding.create(input=batch, model="text-embedding-3-small")
        results.extend(embeddings.data)
    return results
```

---

## 6. 평가 (Evaluation)

```python
from langchain.evaluation import load_evaluator

# 답변 품질 평가
evaluator = load_evaluator("qa")

result = evaluator.evaluate_strings(
    prediction="Paris is the capital of France.",
    reference="The capital of France is Paris.",
    input="What is the capital of France?"
)

print(result["score"])
```

---

**[← 데이터 처리](data-processing.md)** | **[문제 해결 가이드 →](../03-problem-solving-guides/api-server.md)**
