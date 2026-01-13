# 2026 파이썬 표준 스택: 개요

## 핵심 질문

**"지금 파이썬 프로젝트를 시작한다면, 무엇을 써야 할까?"**

이 질문에 대한 2026년의 답이 여기 있습니다.

---

## 표준 스택이란?

"표준 스택"은 **공식 표준이 아닙니다**. 하지만:

- ✅ **커뮤니티 대다수가 선택**하는 도구들
- ✅ **신규 프로젝트의 기본값**으로 자리잡음
- ✅ **취업/이직 시 요구되는** 기술 스택
- ✅ **상호 호환성이 검증**된 조합

즉, **"사실상의 표준(De facto standard)"** 입니다.

---

## 2026 표준 스택 한눈에 보기

| 분야 | 2026 표준 | 핵심 이유 |
|------|-----------|----------|
| **프로젝트 관리** | `uv` + `pyproject.toml` | 10-100배 속도, 표준 포맷 |
| **코드 품질** | `ruff` + `mypy` | 50배 빠른 린팅, 타입 체크 |
| **테스트** | `pytest` | 사실상 표준, 플러그인 생태계 |
| **웹 API** | `FastAPI` + `pydantic` | 비동기, 자동 문서화, 타입 안전 |
| **HTTP 클라이언트** | `httpx` | async 지원, requests 호환 |
| **비동기 I/O** | `asyncio` (내장) | 표준 라이브러리, 생태계 지원 |
| **관측성** | `structlog` + `OpenTelemetry` | 구조화 로깅, 분산 추적 |
| **데이터 처리** | `polars` + `duckdb` | 10배 빠름, SQL 통합 |
| **LLM 앱** | `langchain` / `llamaindex` | RAG, 체인, 에이전트 프레임워크 |

---

## 왜 이것들인가?

### 1. 성능 혁명

**과거 (2023)**
```python
# pipenv으로 패키지 설치: 30-60초
$ pipenv install requests

# flake8 + black + isort 실행: 5-10초
$ flake8 . && black . && isort .

# pandas로 대용량 데이터: 느림
```

**현재 (2026)**
```python
# uv로 패키지 설치: 1-2초
$ uv add requests

# ruff로 통합 린팅: 0.1초
$ ruff check . && ruff format .

# polars로 대용량 데이터: 10배 빠름
```

**→ 개발 속도가 실질적으로 향상**

### 2. 현대적 기능

**비동기가 표준이 됨**

```python
# 과거: 동기 코드가 기본
import requests
response = requests.get('https://api.example.com')

# 현재: async가 기본
import httpx
async with httpx.AsyncClient() as client:
    response = await client.get('https://api.example.com')
```

**타입 안전성이 필수가 됨**

```python
# 과거: 런타임 에러 가능
def process(data):
    return data['name'].upper()

# 현재: 컴파일 타임 체크
from pydantic import BaseModel

class Data(BaseModel):
    name: str

def process(data: Data) -> str:
    return data.name.upper()
```

### 3. 개발자 경험 (DX) 개선

**통합과 자동화**

```toml
# 과거: 여러 설정 파일
# setup.py, setup.cfg, requirements.txt,
# .flake8, pyproject.toml, ...

# 현재: pyproject.toml 하나로 통일
[project]
name = "myapp"
version = "0.1.0"
dependencies = ["fastapi", "httpx"]

[tool.ruff]
line-length = 88

[tool.mypy]
strict = true
```

**자동 문서화**

```python
# FastAPI: 코드 작성하면 문서 자동 생성
@app.get("/users/{user_id}")
async def get_user(user_id: int) -> User:
    """사용자 정보 조회"""
    ...

# → http://localhost:8000/docs 에 Swagger UI 자동 제공
```

### 4. 생태계 통합

이 도구들은 **서로 잘 어울리도록** 설계됨:

```python
# FastAPI + pydantic + httpx + pytest 조합
from fastapi import FastAPI
from pydantic import BaseModel
import httpx
import pytest

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

@app.post("/items/")
async def create_item(item: Item):
    # httpx로 외부 API 호출
    async with httpx.AsyncClient() as client:
        response = await client.post(...)
    return item

# pytest로 테스트
@pytest.mark.asyncio
async def test_create_item():
    # FastAPI 테스트 클라이언트
    ...
```

---

## 이전 스택과의 비교

### 2023년 일반적 스택

```
패키징: pipenv / poetry
린팅: flake8 + black + isort + pylint
테스트: pytest (동일)
웹: Flask / Django
HTTP: requests
데이터: pandas
```

### 2026년 표준 스택

```
패키징: uv + pyproject.toml
린팅: ruff + mypy
테스트: pytest + hypothesis
웹: FastAPI (또는 Django)
HTTP: httpx
데이터: polars + duckdb
```

### 주요 차이점

| 측면 | 2023 | 2026 | 변화 |
|------|------|------|------|
| **속도** | 느림 | 10-100배 빠름 | Rust 기반 도구들 |
| **비동기** | 선택적 | 기본값 | async/await 표준화 |
| **타입** | 선택적 | 필수 트렌드 | pydantic, mypy 보편화 |
| **통합** | 분산된 도구 | 통합 도구 | ruff, pyproject.toml |
| **LLM** | 실험적 | 프로덕션 | langchain, vector DB |

---

## 분야별 깊이 보기

각 분야의 상세 내용은 개별 챕터에서 다룹니다:

### 🔧 [프로젝트 관리 & 패키징](project-packaging.md)
- `uv`: 초고속 패키지 관리자
- `pyproject.toml`: 통합 설정 포맷
- `build` + `twine`: 배포 도구

**핵심**: 개발 환경 설정이 10배 빨라짐

### ✨ [코드 품질 & 린팅](code-quality.md)
- `ruff`: 통합 린터/포매터 (flake8+black+isort 대체)
- `mypy`: 정적 타입 체커
- `pre-commit`: 자동화 훅

**핵심**: 코드 품질 검사가 50배 빨라짐

### 🧪 [테스트 & 품질 보증](testing.md)
- `pytest`: 테스팅 프레임워크
- `hypothesis`: 속성 기반 테스트
- `coverage`: 커버리지 측정

**핵심**: 더 강력한 테스트, 더 적은 코드

### 🌐 [웹 API 개발](web-api.md)
- `FastAPI`: 현대적 웹 프레임워크
- `pydantic`: 데이터 검증
- `uvicorn`: ASGI 서버

**핵심**: 자동 문서화 + 타입 안전 + 고성능

### ⚡ [비동기 I/O & 네트워킹](async-io.md)
- `asyncio`: 표준 비동기 라이브러리
- `httpx`: async HTTP 클라이언트
- `aiofiles`, `aiohttp`: 비동기 유틸

**핵심**: 동시성 처리가 표준이 됨

### 📊 [관측성 & 모니터링](observability.md)
- `structlog`: 구조화 로깅
- `OpenTelemetry`: 분산 추적
- `prometheus_client`: 메트릭 수집

**핵심**: 프로덕션 운영 필수 도구

### 📈 [현대적 데이터 처리](data-processing.md)
- `polars`: 고성능 데이터프레임
- `duckdb`: 인메모리 분석 DB
- `pyarrow`: 컬럼형 데이터 포맷

**핵심**: pandas보다 10배 빠른 데이터 처리

### 🤖 [LLM & AI 애플리케이션](llm-apps.md)
- `langchain` / `llamaindex`: LLM 프레임워크
- `chromadb` / `pinecone`: 벡터 DB
- `openai` / `anthropic`: LLM API 클라이언트

**핵심**: AI 애플리케이션이 표준 워크로드가 됨

---

## 예외와 대안

### 이 스택이 항상 정답은 아닙니다

#### Django를 쓰고 있다면?

**Django는 여전히 훌륭합니다**

```
Django 생태계 안에서는:
- ORM, admin, auth 등 통합 솔루션
- FastAPI보다 더 적합할 수 있음
```

**FastAPI와의 비교**:
- Django: 전체 솔루션 (batteries included)
- FastAPI: API 전용, 마이크로서비스

#### 데이터 과학 중심이라면?

**pandas는 여전히 중심입니다**

```
pandas가 나은 경우:
- Jupyter 노트북 작업
- 복잡한 데이터 변환
- 거대한 생태계 (sklearn, scipy 등)
```

**polars와의 비교**:
- pandas: 더 넓은 생태계, 성숙도
- polars: 성능, 병렬 처리

#### 간단한 스크립트라면?

**표준 라이브러리만으로 충분합니다**

```python
# CLI 도구, 자동화 스크립트 등
import os
import sys
import argparse
import subprocess
```

---

## 학습 우선순위

### Level 1: 필수 (모든 개발자)

```
1. pytest - 테스팅 기본
2. ruff - 코드 품질
3. uv + pyproject.toml - 프로젝트 관리
```

**이것만으로도 생산성 2배**

### Level 2: 웹 개발자

```
+ FastAPI - API 개발
+ pydantic - 데이터 검증
+ httpx - HTTP 클라이언트
+ asyncio 기초
```

**현대적 웹 서비스 개발 가능**

### Level 3: 데이터/AI 개발자

```
+ polars - 고성능 데이터 처리
+ duckdb - SQL 분석
+ langchain - LLM 앱
+ vector DB - 의미 검색
```

**AI 애플리케이션 구축 가능**

### Level 4: 프로덕션 엔지니어

```
+ structlog - 구조화 로깅
+ OpenTelemetry - 분산 추적
+ prometheus - 메트릭
+ Docker + k8s (Python 밖)
```

**프로덕션 운영 가능**

---

## 마이그레이션 가이드

### 기존 프로젝트를 2026 스택으로?

**점진적 접근을 추천합니다**

#### 1단계: 개발 도구 (즉시 가능)

```bash
# ruff 도입 (기존 린터 대체)
$ uv add --dev ruff
$ ruff check .

# uv로 패키지 관리 전환
$ uv init
$ uv sync
```

**영향**: 개발 속도만 빨라짐, 코드 변경 없음

#### 2단계: 테스트 개선 (점진적)

```bash
# pytest로 전환 (unittest에서)
$ uv add --dev pytest
# 기존 테스트를 pytest 스타일로 하나씩 전환
```

**영향**: 테스트만 변경, 프로덕션 코드 그대로

#### 3단계: 코어 로직 (신중히)

```python
# 새 모듈은 FastAPI로
# 기존 Flask 엔드포인트는 유지
# 점진적으로 마이그레이션
```

**영향**: 새 기능부터 현대 스택 적용

---

## 다음 단계

이제 각 분야를 깊이 있게 탐색할 차례입니다.

### 추천 읽기 순서

**웹 개발자라면**:
1. [프로젝트 관리](project-packaging.md)
2. [웹 API 개발](web-api.md)
3. [비동기 I/O](async-io.md)

**데이터/AI 개발자라면**:
1. [프로젝트 관리](project-packaging.md)
2. [데이터 처리](data-processing.md)
3. [LLM 애플리케이션](llm-apps.md)

**모든 개발자에게**:
1. [프로젝트 관리](project-packaging.md) (필수)
2. [코드 품질](code-quality.md) (필수)
3. [테스트](testing.md) (필수)

---

**[← 등급 시스템](../01-introduction/rating-system.md)** | **[프로젝트 관리 →](project-packaging.md)**
