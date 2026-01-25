# LLM 서비스 구축하기

## 목표

RAG (Retrieval-Augmented Generation) 기반 Q&A 서비스를 처음부터 프로덕션까지 구축하기

실제로 돌아가는 LLM 서비스를 만들려면 생각보다 신경 쓸 게 많습니다. OpenAI API 호출하고 끝이 아니라, 비용 관리, 에러 핸들링, 캐싱, 모니터링까지 다 필요합니다. 여기서는 처음부터 프로덕션 레벨까지 단계별로 만들어보겠습니다.

---

## 기술 스택 선택

| 분야 | 도구 | 이유 | 대안 |
|------|------|------|------|
| LLM 프레임워크 | LangChain | RAG 체인, 에이전트 기능 제공 | LlamaIndex (더 RAG에 특화) |
| 벡터 DB | ChromaDB | 로컬 개발 용이, 간단한 API | Qdrant (프로덕션), Pinecone (관리형) |
| 웹 프레임워크 | FastAPI | 비동기 지원, 자동 문서화 | Flask (간단한 경우) |
| LLM API | OpenAI + Anthropic | GPT-4, Claude 3.5 Sonnet | Gemini, Cohere |
| 임베딩 | OpenAI text-embedding-3-small | 성능/가격 밸런스 좋음 | Voyage AI (더 나은 품질) |
| 캐싱 | Redis | 동일 질문 반복 방지 | Memcached |
| 모니터링 | LangSmith | LangChain 네이티브 통합 | Helicone, Langfuse |

**2026년 기준 추천**:
- 개발 단계: ChromaDB (로컬), OpenAI GPT-4o-mini (저렴)
- 프로덕션: Qdrant (자체 호스팅) or Pinecone (관리형), Claude 3.5 Sonnet + GPT-4 (백업)

ChromaDB vs Qdrant vs Pinecone을 고민한다면:
- 프로토타입: ChromaDB가 설치도 쉽고 빠름
- 자체 서버 운영 가능: Qdrant (무료, 성능 좋음)
- 관리 맡기고 싶음: Pinecone (유료지만 안정적)

---

## 아키텍처

```
사용자 질문
    ↓
[임베딩 변환]
    ↓
[벡터 DB 검색] ← (Redis 캐시 체크)
    ↓
[관련 문서 추출]
    ↓
[프롬프트 + 컨텍스트]
    ↓
[LLM API 호출] ← (Fallback: 다른 모델)
    ↓
[응답 스트리밍]
    ↓
[비용/로그 기록]
    ↓
답변 반환
```

실전에서는 여기에 더해서:
- Rate limiting (사용자당 분당 요청 수 제한)
- Retry logic (API 실패 시 재시도)
- Circuit breaker (API가 계속 실패하면 일시 중단)
- A/B testing (프롬프트 버전별 성능 비교)

이 가이드에서는 핵심 기능 위주로 구현하고, 프로덕션 체크리스트도 제공하겠습니다.

---

## 프로젝트 설정

```bash
# 프로젝트 생성
$ uv init llm-service
$ cd llm-service

# 핵심 의존성
$ uv add fastapi "uvicorn[standard]" \
         langchain langchain-openai langchain-anthropic \
         chromadb qdrant-client \
         tiktoken \
         pydantic-settings python-dotenv

# 캐싱 및 모니터링
$ uv add redis langsmith

# 문서 처리
$ uv add pypdf unstructured

# 개발 도구
$ uv add --dev pytest pytest-asyncio httpx ruff
```

### 디렉토리 구조

```
llm-service/
├── src/
│   └── llmservice/
│       ├── __init__.py
│       ├── main.py              # FastAPI 앱
│       ├── config.py            # 설정 관리
│       ├── models.py            # Pydantic 모델
│       ├── vectorstore.py       # 벡터 DB 관리
│       ├── chain.py             # RAG 체인 로직
│       ├── llm_providers.py     # LLM 제공자 추상화
│       ├── cache.py             # Redis 캐싱
│       ├── monitoring.py        # 로깅/모니터링
│       └── utils.py             # 유틸리티 함수
├── data/
│   └── documents/               # 원본 문서
│       ├── *.pdf
│       ├── *.md
│       └── *.txt
├── chroma_db/                   # 벡터 DB (로컬)
├── tests/
│   ├── test_chain.py
│   └── test_api.py
├── scripts/
│   └── ingest_documents.py      # 문서 인덱싱 스크립트
├── .env
└── pyproject.toml
```

---

## 설정 관리

먼저 환경 변수로 모든 설정을 관리합니다. API 키 같은 민감 정보는 절대 코드에 하드코딩하지 마세요.

```python
# src/llmservice/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from typing import Literal

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file='.env', env_file_encoding='utf-8')

    # API 키
    openai_api_key: str
    anthropic_api_key: str | None = None
    langsmith_api_key: str | None = None

    # LLM 설정
    default_provider: Literal["openai", "anthropic"] = "openai"
    openai_model: str = "gpt-4o-mini"
    anthropic_model: str = "claude-3-5-sonnet-20241022"
    temperature: float = 0.0
    max_tokens: int = 2000

    # 임베딩 설정
    embedding_provider: Literal["openai", "voyage"] = "openai"
    embedding_model: str = "text-embedding-3-small"
    embedding_dimension: int = 1536

    # 벡터 DB
    vectordb_type: Literal["chroma", "qdrant"] = "chroma"
    chroma_persist_directory: str = "./chroma_db"
    qdrant_url: str | None = None
    qdrant_collection: str = "documents"

    # RAG 설정
    chunk_size: int = 1000
    chunk_overlap: int = 200
    retrieval_top_k: int = 3

    # Redis 캐싱
    redis_url: str = "redis://localhost:6379"
    cache_ttl: int = 3600  # 1시간
    enable_cache: bool = True

    # 모니터링
    langsmith_project: str = "llm-service"
    log_level: str = "INFO"

    # Rate limiting
    rate_limit_per_minute: int = 10

    # 비용 추적
    openai_input_price_per_1k: float = 0.00015   # GPT-4o-mini
    openai_output_price_per_1k: float = 0.0006
    anthropic_input_price_per_1k: float = 0.003   # Claude 3.5 Sonnet
    anthropic_output_price_per_1k: float = 0.015

settings = Settings()
```

**실전 팁**:
- `pydantic-settings`를 쓰면 환경 변수가 자동으로 타입 변환됩니다
- `.env` 파일은 git에 커밋하지 마세요 (`.gitignore`에 추가)
- 프로덕션에서는 AWS Secrets Manager나 HashiCorp Vault 사용 권장

```bash
# .env 예시
OPENAI_API_KEY=sk-proj-...
ANTHROPIC_API_KEY=sk-ant-...
LANGSMITH_API_KEY=lsv2_...

DEFAULT_PROVIDER=anthropic
OPENAI_MODEL=gpt-4o-mini
TEMPERATURE=0.0

VECTORDB_TYPE=chroma
ENABLE_CACHE=true
REDIS_URL=redis://localhost:6379
```

---

## 문서 처리 및 벡터 저장소

RAG의 핵심은 좋은 문서 인덱싱입니다. 텍스트를 어떻게 쪼개느냐에 따라 검색 품질이 크게 달라집니다.

```python
# src/llmservice/vectorstore.py
import logging
from pathlib import Path
from typing import List

from langchain_community.document_loaders import (
    DirectoryLoader,
    TextLoader,
    PyPDFLoader,
    UnstructuredMarkdownLoader
)
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain.schema import Document

from llmservice.config import settings

logger = logging.getLogger(__name__)


def load_documents(directory: str = "./data/documents") -> List[Document]:
    """다양한 형식의 문서 로드"""
    documents = []
    doc_path = Path(directory)

    if not doc_path.exists():
        logger.warning(f"문서 디렉토리가 없습니다: {directory}")
        return documents

    # PDF 파일
    pdf_files = list(doc_path.glob("**/*.pdf"))
    for pdf_file in pdf_files:
        logger.info(f"PDF 로딩: {pdf_file}")
        loader = PyPDFLoader(str(pdf_file))
        documents.extend(loader.load())

    # Markdown 파일
    md_files = list(doc_path.glob("**/*.md"))
    for md_file in md_files:
        logger.info(f"Markdown 로딩: {md_file}")
        loader = UnstructuredMarkdownLoader(str(md_file))
        documents.extend(loader.load())

    # 텍스트 파일
    txt_files = list(doc_path.glob("**/*.txt"))
    for txt_file in txt_files:
        logger.info(f"텍스트 로딩: {txt_file}")
        loader = TextLoader(str(txt_file))
        documents.extend(loader.load())

    logger.info(f"총 {len(documents)}개 문서 로드 완료")
    return documents


def split_documents(documents: List[Document]) -> List[Document]:
    """문서를 청크로 분할

    RecursiveCharacterTextSplitter는 다음 순서로 분할을 시도:
    1. 두 개의 개행 문자 (단락 구분)
    2. 한 개의 개행 문자
    3. 공백
    4. 문자 단위

    이렇게 하면 의미 단위로 최대한 보존됩니다.
    """
    text_splitter = RecursiveCharacterTextSplitter(
        chunk_size=settings.chunk_size,
        chunk_overlap=settings.chunk_overlap,
        length_function=len,
        separators=["\n\n", "\n", " ", ""]
    )

    splits = text_splitter.split_documents(documents)
    logger.info(f"{len(documents)}개 문서 → {len(splits)}개 청크로 분할")

    return splits


def create_vectorstore() -> Chroma:
    """벡터 저장소 생성 (초기 인덱싱)"""
    # 문서 로드 및 분할
    documents = load_documents()
    if not documents:
        raise ValueError("인덱싱할 문서가 없습니다")

    splits = split_documents(documents)

    # 임베딩 생성
    embeddings = OpenAIEmbeddings(
        model=settings.embedding_model,
        openai_api_key=settings.openai_api_key
    )

    # 벡터 저장소 생성
    logger.info("벡터 저장소 생성 중...")
    vectorstore = Chroma.from_documents(
        documents=splits,
        embedding=embeddings,
        persist_directory=settings.chroma_persist_directory
    )

    logger.info(f"벡터 저장소 생성 완료: {settings.chroma_persist_directory}")
    return vectorstore


def get_vectorstore() -> Chroma:
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


def add_documents_to_vectorstore(documents: List[Document]) -> None:
    """기존 벡터 저장소에 문서 추가"""
    vectorstore = get_vectorstore()
    splits = split_documents(documents)
    vectorstore.add_documents(splits)
    logger.info(f"{len(splits)}개 청크 추가 완료")
```

**청크 크기 선택 가이드**:
- **작은 청크 (200-500자)**: 검색 정확도 높음, 하지만 컨텍스트 부족
- **큰 청크 (1500-2000자)**: 풍부한 컨텍스트, 하지만 노이즈 많음
- **권장 (800-1200자)**: 대부분의 경우 밸런스가 좋음

Overlap도 중요합니다. 예를 들어 chunk_size=1000, overlap=200이면:
```
청크1: [0-1000자]
청크2: [800-1800자]  # 200자 중복
청크3: [1600-2600자]
```

중요한 정보가 청크 경계에 걸쳐있을 때를 대비한 안전장치입니다.

---

## LLM 제공자 추상화

OpenAI와 Anthropic을 쉽게 전환할 수 있도록 추상화 레이어를 만듭니다.

```python
# src/llmservice/llm_providers.py
from abc import ABC, abstractmethod
from typing import AsyncIterator
import tiktoken

from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain.schema import BaseMessage, HumanMessage, AIMessage

from llmservice.config import settings


class LLMProvider(ABC):
    """LLM 제공자 인터페이스"""

    @abstractmethod
    async def generate(self, messages: list[BaseMessage]) -> str:
        """텍스트 생성"""
        pass

    @abstractmethod
    async def stream(self, messages: list[BaseMessage]) -> AsyncIterator[str]:
        """스트리밍 생성"""
        pass

    @abstractmethod
    def count_tokens(self, text: str) -> int:
        """토큰 수 계산"""
        pass

    @abstractmethod
    def calculate_cost(self, input_tokens: int, output_tokens: int) -> float:
        """비용 계산"""
        pass


class OpenAIProvider(LLMProvider):
    def __init__(self):
        self.llm = ChatOpenAI(
            model=settings.openai_model,
            temperature=settings.temperature,
            max_tokens=settings.max_tokens,
            openai_api_key=settings.openai_api_key
        )
        self.encoding = tiktoken.encoding_for_model(settings.openai_model)

    async def generate(self, messages: list[BaseMessage]) -> str:
        response = await self.llm.ainvoke(messages)
        return response.content

    async def stream(self, messages: list[BaseMessage]) -> AsyncIterator[str]:
        async for chunk in self.llm.astream(messages):
            if chunk.content:
                yield chunk.content

    def count_tokens(self, text: str) -> int:
        return len(self.encoding.encode(text))

    def calculate_cost(self, input_tokens: int, output_tokens: int) -> float:
        input_cost = (input_tokens / 1000) * settings.openai_input_price_per_1k
        output_cost = (output_tokens / 1000) * settings.openai_output_price_per_1k
        return input_cost + output_cost


class AnthropicProvider(LLMProvider):
    def __init__(self):
        self.llm = ChatAnthropic(
            model=settings.anthropic_model,
            temperature=settings.temperature,
            max_tokens=settings.max_tokens,
            anthropic_api_key=settings.anthropic_api_key
        )
        # Anthropic은 대략적으로 문자 수 / 4
        # 정확한 토큰 계산은 Anthropic API 응답의 usage 필드 사용

    async def generate(self, messages: list[BaseMessage]) -> str:
        response = await self.llm.ainvoke(messages)
        return response.content

    async def stream(self, messages: list[BaseMessage]) -> AsyncIterator[str]:
        async for chunk in self.llm.astream(messages):
            if chunk.content:
                yield chunk.content

    def count_tokens(self, text: str) -> int:
        # 대략적인 추정 (실제로는 API 응답에서 가져와야 함)
        return len(text) // 4

    def calculate_cost(self, input_tokens: int, output_tokens: int) -> float:
        input_cost = (input_tokens / 1000) * settings.anthropic_input_price_per_1k
        output_cost = (output_tokens / 1000) * settings.anthropic_output_price_per_1k
        return input_cost + output_cost


def get_llm_provider(provider_name: str | None = None) -> LLMProvider:
    """LLM 제공자 팩토리"""
    provider_name = provider_name or settings.default_provider

    if provider_name == "openai":
        return OpenAIProvider()
    elif provider_name == "anthropic":
        return AnthropicProvider()
    else:
        raise ValueError(f"알 수 없는 제공자: {provider_name}")
```

이렇게 하면 나중에 설정만 바꿔서 OpenAI ↔ Anthropic 전환이 쉽습니다. 실전에서는:
- 평소: Claude 3.5 Sonnet (품질 좋음)
- Claude API 장애 시: GPT-4o로 자동 전환
- 간단한 쿼리: GPT-4o-mini (저렴)
- 복잡한 추론: Claude Opus or GPT-4

---

## Redis 캐싱

동일한 질문이 반복되면 LLM API를 다시 호출하는 건 비용 낭비입니다. Redis로 캐싱하세요.

```python
# src/llmservice/cache.py
import hashlib
import json
import logging
from typing import Any

import redis.asyncio as redis

from llmservice.config import settings

logger = logging.getLogger(__name__)


class Cache:
    def __init__(self):
        self.redis_client: redis.Redis | None = None
        self.enabled = settings.enable_cache

    async def connect(self):
        """Redis 연결"""
        if not self.enabled:
            logger.info("캐시 비활성화됨")
            return

        try:
            self.redis_client = await redis.from_url(
                settings.redis_url,
                encoding="utf-8",
                decode_responses=True
            )
            await self.redis_client.ping()
            logger.info("Redis 연결 성공")
        except Exception as e:
            logger.warning(f"Redis 연결 실패: {e}. 캐시 없이 계속 진행")
            self.enabled = False

    async def disconnect(self):
        """Redis 연결 종료"""
        if self.redis_client:
            await self.redis_client.close()

    def _make_key(self, question: str, model: str) -> str:
        """질문으로 캐시 키 생성"""
        # 질문 + 모델명을 해시하여 키 생성
        content = f"{question}:{model}"
        hash_key = hashlib.sha256(content.encode()).hexdigest()
        return f"llm:answer:{hash_key}"

    async def get(self, question: str, model: str) -> dict | None:
        """캐시에서 답변 조회"""
        if not self.enabled or not self.redis_client:
            return None

        key = self._make_key(question, model)
        try:
            cached = await self.redis_client.get(key)
            if cached:
                logger.info(f"캐시 히트: {question[:50]}...")
                return json.loads(cached)
        except Exception as e:
            logger.error(f"캐시 조회 실패: {e}")

        return None

    async def set(self, question: str, model: str, answer: dict) -> None:
        """답변을 캐시에 저장"""
        if not self.enabled or not self.redis_client:
            return

        key = self._make_key(question, model)
        try:
            await self.redis_client.setex(
                key,
                settings.cache_ttl,
                json.dumps(answer, ensure_ascii=False)
            )
            logger.debug(f"캐시 저장: {question[:50]}...")
        except Exception as e:
            logger.error(f"캐시 저장 실패: {e}")

    async def clear(self) -> None:
        """모든 캐시 삭제"""
        if not self.enabled or not self.redis_client:
            return

        try:
            keys = await self.redis_client.keys("llm:answer:*")
            if keys:
                await self.redis_client.delete(*keys)
                logger.info(f"{len(keys)}개 캐시 삭제")
        except Exception as e:
            logger.error(f"캐시 삭제 실패: {e}")


# 싱글톤 인스턴스
cache = Cache()
```

**캐싱 전략**:
- TTL은 1시간 정도가 적당 (질문/답변의 신선도 vs 비용 절감)
- 사용자별로 다른 캐시 키를 쓸 수도 있음 (개인화)
- 프롬프트나 모델이 바뀌면 캐시 무효화 필요

---

## RAG 체인 구성

이제 핵심 로직입니다. 질문을 받아서 문서를 검색하고 LLM으로 답변을 생성합니다.

```python
# src/llmservice/chain.py
import logging
from typing import List

from langchain.prompts import ChatPromptTemplate
from langchain.schema import Document, HumanMessage, SystemMessage

from llmservice.config import settings
from llmservice.vectorstore import get_vectorstore
from llmservice.llm_providers import get_llm_provider
from llmservice.cache import cache

logger = logging.getLogger(__name__)


# 프롬프트 템플릿
SYSTEM_PROMPT = """당신은 제공된 문서를 바탕으로 정확하게 답변하는 AI 어시스턴트입니다.

답변 시 규칙:
1. 제공된 컨텍스트만을 기반으로 답변하세요
2. 컨텍스트에 정보가 없으면 "제공된 문서에서 해당 정보를 찾을 수 없습니다"라고 솔직히 답하세요
3. 추측하거나 외부 지식을 사용하지 마세요
4. 간결하고 명확하게 답변하세요
5. 가능하면 컨텍스트의 원문을 인용하세요"""

USER_PROMPT_TEMPLATE = """다음 문서 컨텍스트를 참고하여 질문에 답변하세요:

<context>
{context}
</context>

질문: {question}"""


def format_docs(docs: List[Document]) -> str:
    """문서를 컨텍스트 문자열로 변환"""
    formatted = []
    for i, doc in enumerate(docs, 1):
        source = doc.metadata.get("source", "알 수 없음")
        formatted.append(f"[문서 {i} - {source}]\n{doc.page_content}\n")

    return "\n".join(formatted)


async def ask_question(
    question: str,
    provider_name: str | None = None,
    use_cache: bool = True
) -> dict:
    """질문에 답변

    Args:
        question: 사용자 질문
        provider_name: LLM 제공자 ("openai" or "anthropic")
        use_cache: 캐시 사용 여부

    Returns:
        {
            "question": str,
            "answer": str,
            "sources": List[dict],
            "cost": float,
            "cached": bool
        }
    """
    # LLM 제공자 선택
    llm_provider = get_llm_provider(provider_name)
    model_name = provider_name or settings.default_provider

    # 캐시 확인
    if use_cache:
        cached_answer = await cache.get(question, model_name)
        if cached_answer:
            cached_answer["cached"] = True
            return cached_answer

    # 벡터 저장소에서 관련 문서 검색
    logger.info(f"질문: {question}")
    vectorstore = get_vectorstore()
    retriever = vectorstore.as_retriever(
        search_kwargs={"k": settings.retrieval_top_k}
    )

    docs = retriever.get_relevant_documents(question)
    logger.info(f"{len(docs)}개 문서 검색됨")

    if not docs:
        return {
            "question": question,
            "answer": "관련된 문서를 찾을 수 없습니다.",
            "sources": [],
            "cost": 0.0,
            "cached": False
        }

    # 프롬프트 구성
    context = format_docs(docs)
    user_prompt = USER_PROMPT_TEMPLATE.format(context=context, question=question)

    messages = [
        SystemMessage(content=SYSTEM_PROMPT),
        HumanMessage(content=user_prompt)
    ]

    # LLM 호출
    try:
        answer = await llm_provider.generate(messages)
    except Exception as e:
        logger.error(f"LLM 호출 실패: {e}")
        raise

    # 토큰 및 비용 계산
    input_text = SYSTEM_PROMPT + user_prompt
    input_tokens = llm_provider.count_tokens(input_text)
    output_tokens = llm_provider.count_tokens(answer)
    cost = llm_provider.calculate_cost(input_tokens, output_tokens)

    logger.info(f"입력: {input_tokens} 토큰, 출력: {output_tokens} 토큰, 비용: ${cost:.4f}")

    # 출처 정보
    sources = [
        {
            "content": doc.page_content[:200] + "...",
            "metadata": doc.metadata
        }
        for doc in docs
    ]

    result = {
        "question": question,
        "answer": answer,
        "sources": sources,
        "cost": cost,
        "input_tokens": input_tokens,
        "output_tokens": output_tokens,
        "model": model_name,
        "cached": False
    }

    # 캐시 저장
    if use_cache:
        await cache.set(question, model_name, result)

    return result


async def ask_question_stream(
    question: str,
    provider_name: str | None = None
):
    """스트리밍 답변 생성"""
    llm_provider = get_llm_provider(provider_name)

    # 문서 검색
    vectorstore = get_vectorstore()
    retriever = vectorstore.as_retriever(
        search_kwargs={"k": settings.retrieval_top_k}
    )
    docs = retriever.get_relevant_documents(question)

    if not docs:
        yield "관련된 문서를 찾을 수 없습니다."
        return

    # 프롬프트 구성
    context = format_docs(docs)
    user_prompt = USER_PROMPT_TEMPLATE.format(context=context, question=question)

    messages = [
        SystemMessage(content=SYSTEM_PROMPT),
        HumanMessage(content=user_prompt)
    ]

    # 스트리밍 생성
    async for chunk in llm_provider.stream(messages):
        yield chunk
```

**프롬프트 엔지니어링 팁**:
- System prompt에서 역할과 제약사항을 명확히
- "외부 지식 사용 금지"를 강조하면 환각(hallucination) 감소
- `<context>` 태그로 구조화하면 모델이 더 잘 이해함
- 출처 인용을 요구하면 신뢰도 상승

---

## API 모델

```python
# src/llmservice/models.py
from pydantic import BaseModel, Field
from typing import List, Literal


class QuestionRequest(BaseModel):
    """질문 요청"""
    text: str = Field(..., description="질문 내용", min_length=1, max_length=1000)
    provider: Literal["openai", "anthropic"] | None = Field(None, description="LLM 제공자")
    use_cache: bool = Field(True, description="캐시 사용 여부")


class Source(BaseModel):
    """출처 정보"""
    content: str
    metadata: dict


class AnswerResponse(BaseModel):
    """답변 응답"""
    question: str
    answer: str
    sources: List[Source]
    cost: float
    input_tokens: int | None = None
    output_tokens: int | None = None
    model: str
    cached: bool


class IndexRequest(BaseModel):
    """문서 인덱싱 요청"""
    directory: str | None = Field(None, description="문서 디렉토리 경로")


class HealthResponse(BaseModel):
    """헬스 체크 응답"""
    status: str
    vectorstore_ready: bool
    cache_ready: bool
```

---

## FastAPI 서버

이제 모든 것을 연결합니다.

```python
# src/llmservice/main.py
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException, status
from fastapi.responses import StreamingResponse
from fastapi.middleware.cors import CORSMiddleware

from llmservice.models import (
    QuestionRequest,
    AnswerResponse,
    IndexRequest,
    HealthResponse
)
from llmservice.chain import ask_question, ask_question_stream
from llmservice.vectorstore import create_vectorstore, get_vectorstore
from llmservice.cache import cache
from llmservice.config import settings

# 로깅 설정
logging.basicConfig(
    level=getattr(logging, settings.log_level),
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """앱 시작/종료 시 실행"""
    # 시작
    logger.info("LLM 서비스 시작")
    await cache.connect()

    yield

    # 종료
    logger.info("LLM 서비스 종료")
    await cache.disconnect()


app = FastAPI(
    title="LLM Q&A Service",
    description="RAG 기반 질의응답 서비스",
    version="1.0.0",
    lifespan=lifespan
)

# CORS 설정 (프론트엔드에서 접근할 경우)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # 프로덕션에서는 특정 도메인만 허용
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


@app.post("/ask", response_model=AnswerResponse)
async def ask(request: QuestionRequest):
    """질문에 답변

    캐시가 활성화되어 있으면 동일한 질문에 대해 캐시된 답변을 반환합니다.
    """
    try:
        result = await ask_question(
            question=request.text,
            provider_name=request.provider,
            use_cache=request.use_cache
        )
        return AnswerResponse(**result)
    except Exception as e:
        logger.error(f"질문 처리 실패: {e}", exc_info=True)
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"질문 처리 중 오류 발생: {str(e)}"
        )


@app.post("/ask/stream")
async def ask_stream(request: QuestionRequest):
    """스트리밍 답변

    답변을 점진적으로 반환합니다. 긴 답변의 경우 사용자 경험이 개선됩니다.
    """
    async def generate():
        try:
            async for chunk in ask_question_stream(
                question=request.text,
                provider_name=request.provider
            ):
                yield chunk
        except Exception as e:
            logger.error(f"스트리밍 실패: {e}", exc_info=True)
            yield f"\n\n오류 발생: {str(e)}"

    return StreamingResponse(generate(), media_type="text/plain")


@app.post("/index")
async def index_documents(request: IndexRequest | None = None):
    """문서 인덱싱

    새로운 문서를 벡터 DB에 추가합니다. 시간이 오래 걸릴 수 있습니다.
    """
    try:
        directory = request.directory if request else None
        if directory:
            # 특정 디렉토리 인덱싱 (향후 구현)
            raise HTTPException(
                status_code=status.HTTP_501_NOT_IMPLEMENTED,
                detail="디렉토리 지정 인덱싱은 아직 미구현"
            )

        create_vectorstore()
        return {
            "status": "success",
            "message": "문서 인덱싱 완료"
        }
    except Exception as e:
        logger.error(f"인덱싱 실패: {e}", exc_info=True)
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"인덱싱 중 오류 발생: {str(e)}"
        )


@app.delete("/cache")
async def clear_cache():
    """캐시 삭제

    프롬프트나 모델을 변경한 후 기존 캐시를 무효화할 때 사용합니다.
    """
    await cache.clear()
    return {"status": "success", "message": "캐시 삭제 완료"}


@app.get("/health", response_model=HealthResponse)
async def health():
    """헬스 체크"""
    vectorstore_ready = True
    cache_ready = cache.enabled and cache.redis_client is not None

    try:
        # 벡터 저장소 확인
        vectorstore = get_vectorstore()
        # 간단한 검색 테스트
        vectorstore.similarity_search("test", k=1)
    except Exception as e:
        logger.warning(f"벡터 저장소 헬스 체크 실패: {e}")
        vectorstore_ready = False

    return HealthResponse(
        status="ok" if vectorstore_ready else "degraded",
        vectorstore_ready=vectorstore_ready,
        cache_ready=cache_ready
    )


@app.get("/")
async def root():
    """루트 엔드포인트"""
    return {
        "service": "LLM Q&A Service",
        "version": "1.0.0",
        "docs": "/docs"
    }
```

**API 설계 포인트**:
- `/ask`: 일반 질문 (캐시 지원)
- `/ask/stream`: 스트리밍 (긴 답변에 유용)
- `/index`: 문서 재인덱싱 (관리자용)
- `/cache DELETE`: 캐시 무효화
- `/health`: 모니터링용 헬스 체크

---

## 문서 인덱싱 스크립트

서버와 별도로 문서를 인덱싱하는 스크립트를 만들면 편합니다.

```python
# scripts/ingest_documents.py
import asyncio
import sys
from pathlib import Path

# 프로젝트 루트를 Python 경로에 추가
sys.path.insert(0, str(Path(__file__).parent.parent / "src"))

from llmservice.vectorstore import create_vectorstore
from llmservice.config import settings

async def main():
    print(f"문서 디렉토리: {settings.chroma_persist_directory}")
    print(f"청크 크기: {settings.chunk_size}")
    print(f"청크 오버랩: {settings.chunk_overlap}")
    print("\n인덱싱 시작...")

    try:
        vectorstore = create_vectorstore()
        print("\n인덱싱 완료!")
    except Exception as e:
        print(f"\n인덱싱 실패: {e}")
        sys.exit(1)

if __name__ == "__main__":
    asyncio.run(main())
```

사용법:
```bash
$ uv run python scripts/ingest_documents.py
```

---

## 테스트

```python
# tests/test_chain.py
import pytest
from llmservice.chain import ask_question
from llmservice.vectorstore import create_vectorstore

@pytest.mark.asyncio
async def test_ask_question():
    """질문 응답 테스트"""
    # 테스트용 벡터 저장소 생성 (fixture로 분리 권장)
    # create_vectorstore()

    result = await ask_question(
        question="Python은 누가 만들었나요?",
        use_cache=False
    )

    assert result["question"] == "Python은 누가 만들었나요?"
    assert len(result["answer"]) > 0
    assert "cost" in result
    assert result["cost"] >= 0


# tests/test_api.py
import pytest
from fastapi.testclient import TestClient
from llmservice.main import app

client = TestClient(app)

def test_health():
    """헬스 체크 테스트"""
    response = client.get("/health")
    assert response.status_code == 200
    data = response.json()
    assert data["status"] in ["ok", "degraded"]

def test_ask():
    """질문 API 테스트"""
    response = client.post(
        "/ask",
        json={"text": "Python이란?", "use_cache": False}
    )
    assert response.status_code == 200
    data = response.json()
    assert "answer" in data
    assert "cost" in data
```

실전 팁:
- 실제 API를 호출하지 않고 모킹(mocking)하는 게 더 빠름
- 벡터 저장소는 pytest fixture로 한 번만 생성
- 비용 테스트는 로컬 모델이나 모킹 사용

---

## 사용 예시

### 서버 시작

```bash
# 1. 환경 변수 설정
$ cat > .env << EOF
OPENAI_API_KEY=sk-proj-...
ANTHROPIC_API_KEY=sk-ant-...
DEFAULT_PROVIDER=anthropic
ENABLE_CACHE=true
EOF

# 2. Redis 실행 (Docker 사용 시)
$ docker run -d -p 6379:6379 redis:7-alpine

# 3. 문서 인덱싱
$ uv run python scripts/ingest_documents.py

# 4. 서버 시작
$ uv run uvicorn llmservice.main:app --reload --host 0.0.0.0 --port 8000
```

### API 호출

```bash
# 일반 질문
$ curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Python의 특징은 무엇인가요?",
    "provider": "openai",
    "use_cache": true
  }'

# 스트리밍
$ curl -X POST http://localhost:8000/ask/stream \
  -H "Content-Type: application/json" \
  -d '{"text": "FastAPI 사용법을 자세히 알려주세요"}' \
  --no-buffer

# 캐시 삭제
$ curl -X DELETE http://localhost:8000/cache

# 헬스 체크
$ curl http://localhost:8000/health
```

### Python 클라이언트

```python
import httpx
import asyncio

async def ask_question(question: str):
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "http://localhost:8000/ask",
            json={"text": question, "use_cache": True}
        )
        return response.json()

# 사용
result = asyncio.run(ask_question("RAG가 뭔가요?"))
print(f"답변: {result['answer']}")
print(f"비용: ${result['cost']:.4f}")
print(f"캐시 히트: {result['cached']}")
```

---

## 고급 기능

### 하이브리드 검색 (키워드 + 벡터)

벡터 검색만으로는 정확한 키워드 매칭이 약할 수 있습니다. BM25 (키워드 검색)와 결합하면 더 좋습니다.

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

def create_hybrid_retriever():
    """하이브리드 검색기 생성"""
    # 벡터 검색
    vectorstore = get_vectorstore()
    vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

    # BM25 키워드 검색
    documents = load_documents()
    splits = split_documents(documents)
    bm25_retriever = BM25Retriever.from_documents(splits)
    bm25_retriever.k = 5

    # 앙상블 (벡터 70%, BM25 30%)
    ensemble_retriever = EnsembleRetriever(
        retrievers=[vector_retriever, bm25_retriever],
        weights=[0.7, 0.3]
    )

    return ensemble_retriever
```

### Reranking (재정렬)

검색된 문서를 LLM으로 다시 정렬하면 정확도가 올라갑니다.

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

def create_reranking_retriever():
    """Reranking 검색기"""
    base_retriever = get_vectorstore().as_retriever(search_kwargs={"k": 10})

    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    compressor = LLMChainExtractor.from_llm(llm)

    compression_retriever = ContextualCompressionRetriever(
        base_compressor=compressor,
        base_retriever=base_retriever
    )

    return compression_retriever
```

이렇게 하면:
1. 벡터 검색으로 10개 문서 가져오기
2. LLM으로 질문과 가장 관련 있는 3-5개만 선택
3. 노이즈 감소, 정확도 상승

단점: LLM 호출이 한 번 더 들어가서 비용 증가 (하지만 최종 답변 품질은 좋아짐)

### Query Expansion (질문 확장)

사용자 질문을 여러 버전으로 확장하면 검색 품질이 좋아집니다.

```python
async def expand_query(question: str) -> List[str]:
    """질문을 여러 버전으로 확장"""
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

    prompt = f"""다음 질문을 3가지 다른 방식으로 다시 표현하세요:

원본 질문: {question}

1.
2.
3.
"""

    response = await llm.ainvoke([HumanMessage(content=prompt)])
    # 파싱해서 리스트로 반환
    expanded = [question] + response.content.split("\n")
    return expanded

# 사용
queries = await expand_query("Python 속도 개선 방법은?")
# ["Python 속도 개선 방법은?", "Python 성능 최적화 기법", "파이썬 실행 속도 높이는 법", ...]

# 각 쿼리로 검색 후 결과 합치기
all_docs = []
for query in queries:
    docs = retriever.get_relevant_documents(query)
    all_docs.extend(docs)

# 중복 제거 및 스코어링
unique_docs = deduplicate_documents(all_docs)
```

### Agent 패턴 (도구 사용)

RAG만으로는 한계가 있을 때 외부 도구를 사용하는 에이전트를 만들 수 있습니다.

```python
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain.tools import Tool

def create_agent():
    """RAG + 도구 사용 에이전트"""
    llm = ChatOpenAI(model="gpt-4o", temperature=0)

    # 도구 정의
    def search_documents(query: str) -> str:
        """문서 검색 도구"""
        retriever = get_vectorstore().as_retriever()
        docs = retriever.get_relevant_documents(query)
        return format_docs(docs)

    def calculate(expression: str) -> str:
        """계산기 도구"""
        try:
            result = eval(expression)  # 실전에서는 ast.literal_eval 사용
            return str(result)
        except:
            return "계산 실패"

    tools = [
        Tool(
            name="SearchDocuments",
            func=search_documents,
            description="문서에서 정보를 검색합니다"
        ),
        Tool(
            name="Calculator",
            func=calculate,
            description="수학 계산을 수행합니다 (예: 2+2)"
        )
    ]

    agent = create_openai_tools_agent(llm, tools, prompt)
    agent_executor = AgentExecutor(agent=agent, tools=tools)

    return agent_executor

# 사용
agent = create_agent()
result = agent.invoke({"input": "문서에서 Python 버전을 찾고, 2024에서 그 버전을 뺀 값은?"})
```

---

## 비용 최적화

LLM API는 비쌉니다. 실전에서 비용을 낮추는 방법:

### 모델 선택 전략

```python
def select_model_by_complexity(question: str) -> str:
    """질문 복잡도에 따라 모델 선택"""
    # 간단한 휴리스틱: 질문 길이, 키워드 체크
    if len(question) < 50 and "간단" in question:
        return "gpt-4o-mini"  # 저렴
    elif "자세히" in question or "분석" in question:
        return "claude-3-5-sonnet-20241022"  # 비싸지만 품질 좋음
    else:
        return "gpt-4o"  # 중간

# 사용
model = select_model_by_complexity(question)
result = await ask_question(question, provider_name=model)
```

### 캐시 적극 활용

```python
# 설정에서 TTL 늘리기
CACHE_TTL = 86400  # 24시간

# 유사 질문 감지 후 캐시 히트
from difflib import SequenceMatcher

async def find_similar_cached_question(question: str, threshold: float = 0.9):
    """유사한 질문이 캐시에 있는지 확인"""
    # Redis에서 모든 질문 키 가져오기
    keys = await cache.redis_client.keys("llm:answer:*")

    for key in keys:
        cached = await cache.redis_client.get(key)
        cached_data = json.loads(cached)
        cached_question = cached_data["question"]

        similarity = SequenceMatcher(None, question, cached_question).ratio()
        if similarity >= threshold:
            return cached_data

    return None
```

### 프롬프트 압축

토큰을 줄이면 비용이 줄어듭니다.

```python
def compress_context(docs: List[Document], max_tokens: int = 2000) -> str:
    """컨텍스트를 토큰 제한 내로 압축"""
    encoding = tiktoken.encoding_for_model("gpt-4")

    compressed_parts = []
    total_tokens = 0

    for doc in docs:
        tokens = encoding.encode(doc.page_content)
        if total_tokens + len(tokens) > max_tokens:
            # 남은 토큰만큼만 추가
            remaining = max_tokens - total_tokens
            compressed_parts.append(encoding.decode(tokens[:remaining]))
            break
        else:
            compressed_parts.append(doc.page_content)
            total_tokens += len(tokens)

    return "\n\n".join(compressed_parts)
```

### 비용 모니터링

```python
# src/llmservice/monitoring.py
import logging
from datetime import datetime
from typing import Dict

logger = logging.getLogger(__name__)

class CostTracker:
    def __init__(self):
        self.daily_cost: Dict[str, float] = {}
        self.request_count: Dict[str, int] = {}

    def log_request(self, model: str, cost: float):
        """요청 비용 기록"""
        today = datetime.now().strftime("%Y-%m-%d")

        if today not in self.daily_cost:
            self.daily_cost[today] = 0.0
            self.request_count[today] = 0

        self.daily_cost[today] += cost
        self.request_count[today] += 1

        logger.info(
            f"[비용] {model}: ${cost:.4f} | "
            f"일일 누적: ${self.daily_cost[today]:.2f} | "
            f"요청 수: {self.request_count[today]}"
        )

        # 일일 예산 초과 경고
        if self.daily_cost[today] > 10.0:  # $10
            logger.warning(f"일일 비용 초과: ${self.daily_cost[today]:.2f}")

    def get_stats(self, date: str | None = None) -> dict:
        """비용 통계 조회"""
        date = date or datetime.now().strftime("%Y-%m-%d")
        return {
            "date": date,
            "total_cost": self.daily_cost.get(date, 0.0),
            "request_count": self.request_count.get(date, 0)
        }

cost_tracker = CostTracker()
```

FastAPI에 추가:
```python
@app.get("/stats")
async def get_stats():
    """비용 통계 조회"""
    return cost_tracker.get_stats()
```

---

## 모니터링 및 평가

### LangSmith 통합

LangChain 공식 모니터링 도구입니다.

```bash
# .env에 추가
LANGSMITH_API_KEY=lsv2_...
LANGSMITH_PROJECT=llm-service
LANGCHAIN_TRACING_V2=true
```

이것만 설정하면 자동으로:
- 모든 LLM 호출 로그
- 체인 실행 추적
- 토큰 사용량 기록
- 레이턴시 측정

LangSmith 대시보드에서 실시간 확인 가능합니다.

### RAG 평가 메트릭

```python
# 평가 데이터셋 준비
evaluation_data = [
    {
        "question": "Python은 누가 만들었나요?",
        "expected_answer": "귀도 반 로섬",
        "ground_truth_docs": ["python_history.txt"]
    },
    # ... 더 많은 테스트 케이스
]

# 평가 함수
from difflib import SequenceMatcher

async def evaluate_rag():
    """RAG 시스템 평가"""
    results = []

    for item in evaluation_data:
        result = await ask_question(item["question"], use_cache=False)

        # 답변 정확도 (간단한 문자열 유사도)
        similarity = SequenceMatcher(
            None,
            result["answer"].lower(),
            item["expected_answer"].lower()
        ).ratio()

        # 올바른 문서를 검색했는지 확인
        retrieved_sources = [s["metadata"].get("source") for s in result["sources"]]
        source_precision = len(
            set(retrieved_sources) & set(item["ground_truth_docs"])
        ) / len(retrieved_sources) if retrieved_sources else 0

        results.append({
            "question": item["question"],
            "answer_similarity": similarity,
            "source_precision": source_precision,
            "cost": result["cost"]
        })

    # 평균 점수
    avg_similarity = sum(r["answer_similarity"] for r in results) / len(results)
    avg_precision = sum(r["source_precision"] for r in results) / len(results)
    total_cost = sum(r["cost"] for r in results)

    print(f"답변 정확도: {avg_similarity:.2%}")
    print(f"검색 정확도: {avg_precision:.2%}")
    print(f"총 비용: ${total_cost:.4f}")

    return results
```

실전에서는:
- RAGAS 라이브러리 사용 (faithfulness, answer relevancy 등)
- 사람이 직접 평가 (1-5점 척도)
- A/B 테스트로 프롬프트 버전 비교

---

## 프로덕션 배포

### Docker

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app

# uv 설치
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

# 의존성 복사
COPY pyproject.toml ./
COPY uv.lock ./

# 의존성 설치
RUN uv sync --frozen

# 소스 복사
COPY src/ ./src/
COPY scripts/ ./scripts/

# 문서 및 벡터 DB는 볼륨으로 마운트
VOLUME ["/app/data", "/app/chroma_db"]

# 환경 변수
ENV PYTHONPATH=/app/src

# 서버 실행
CMD ["uv", "run", "uvicorn", "llmservice.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  llm-service:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - REDIS_URL=redis://redis:6379
    volumes:
      - ./data:/app/data
      - ./chroma_db:/app/chroma_db
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

실행:
```bash
$ docker-compose up -d
```

### 환경별 설정

```python
# config.py에 추가
import os

class Settings(BaseSettings):
    environment: Literal["development", "production"] = "development"

    # 프로덕션에서는 더 엄격한 설정
    @property
    def rate_limit_per_minute(self) -> int:
        return 50 if self.environment == "production" else 10

    @property
    def log_level(self) -> str:
        return "WARNING" if self.environment == "production" else "INFO"
```

### 보안 체크리스트

프로덕션 배포 전 확인사항:

- [ ] API 키를 환경 변수나 Secrets Manager로 관리
- [ ] CORS 설정을 특정 도메인만 허용
- [ ] Rate limiting 설정 (slowapi 사용)
- [ ] HTTPS 적용 (nginx + Let's Encrypt)
- [ ] 입력 검증 (질문 길이 제한, SQL injection 방지)
- [ ] 에러 메시지에 민감 정보 노출 방지
- [ ] 로그에 API 키나 개인정보 제외
- [ ] 의존성 보안 업데이트 (uv lock)

### Rate Limiting

```python
# main.py에 추가
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.post("/ask")
@limiter.limit("10/minute")  # 분당 10회 제한
async def ask(request: Request, question: QuestionRequest):
    # ... 기존 코드
```

---

## 트러블슈팅

### 문제: 벡터 검색이 관련 없는 문서를 반환

**원인**:
- 임베딩 모델이 도메인에 맞지 않음
- 청크 크기가 너무 크거나 작음
- 질문이 애매모호함

**해결**:
1. 하이브리드 검색 (벡터 + BM25) 사용
2. 청크 크기 실험 (800, 1000, 1200 등)
3. Query expansion으로 질문 명확화
4. Fine-tuned 임베딩 모델 사용 (도메인별 학습)

### 문제: LLM이 환각(hallucination)을 일으킴

**원인**:
- 프롬프트에서 "외부 지식 사용 금지"가 약함
- Temperature가 너무 높음
- 검색된 문서에 정보가 부족

**해결**:
1. System prompt 강화: "반드시 제공된 컨텍스트만 사용"
2. Temperature = 0으로 설정
3. 답변 검증: 출처 확인 로직 추가
4. "정보 없음" 답변을 허용하도록 프롬프트 수정

### 문제: 비용이 너무 많이 나옴

**원인**:
- 캐시 미사용
- 비싼 모델 남용 (GPT-4, Claude Opus)
- 컨텍스트가 너무 큼

**해결**:
1. Redis 캐싱 활성화
2. 간단한 질문은 GPT-4o-mini 사용
3. 컨텍스트 압축 (토큰 제한)
4. 일일 예산 모니터링 및 알림

### 문제: 응답 속도가 느림

**원인**:
- 동기 처리
- 벡터 검색이 느림 (ChromaDB 로컬)
- LLM API 레이턴시

**해결**:
1. 비동기 처리 확인 (async/await)
2. Qdrant나 Pinecone으로 전환 (더 빠름)
3. 스트리밍 사용 (사용자 체감 속도 개선)
4. 캐싱으로 반복 질문 최적화

---

## 마무리

여기까지 하면 프로덕션에서 쓸 수 있는 RAG 서비스가 완성됩니다.

**핵심 요약**:
- RAG = 문서 검색 + LLM 생성
- 좋은 청크 분할이 검색 품질의 80%
- 캐싱으로 비용 절감 (중요!)
- 여러 LLM 제공자 지원으로 안정성 확보
- 모니터링 없이는 프로덕션 불가능

**다음 단계**:
- 멀티모달 RAG (이미지, 표 포함 문서)
- Fine-tuning (도메인 특화 임베딩)
- Graph RAG (지식 그래프 기반)
- Multi-agent RAG (여러 에이전트 협업)

실전에서 가장 중요한 건 평가입니다. 정량적 메트릭(RAGAS)과 사용자 피드백을 지속적으로 수집해서 프롬프트와 검색 로직을 개선해야 합니다. 처음부터 완벽할 순 없고, 반복적으로 개선하는 게 정답입니다.

**[← 자동화 스크립트](automation.md)** | **[패키지 배포 →](package-deployment.md)**
