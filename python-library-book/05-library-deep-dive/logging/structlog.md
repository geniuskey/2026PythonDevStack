# structlog 완벽 가이드

> **구조화 로깅 라이브러리**


---

## 개요

| 항목 | 내용 |
|------|------|
| **공식 사이트** | https://www.structlog.org |
| **GitHub** | https://github.com/hynek/structlog |

### 한 줄 요약

**JSON 형식의 파싱 가능한 구조화 로그를 생성하는 라이브러리**

---

## 왜 structlog인가

### 일반 로깅의 문제

```python
# 표준 logging
import logging

logging.info("User login successful")
# 2024-01-13 10:30:00 INFO User login successful

# 문제: 파싱 어려움, 검색 어려움, 컨텍스트 부족
```

### structlog 방식

```python
import structlog

logger = structlog.get_logger()

logger.info("user_login", user_id=123, ip="192.168.1.1", success=True)
# {"event": "user_login", "user_id": 123, "ip": "192.168.1.1", "success": true, "timestamp": "..."}

# 장점: JSON 파싱, Elasticsearch/Kibana 검색, 컨텍스트 풍부
```

---

## 설치 및 설정

```bash
$ uv add structlog
```

### 기본 설정

```python
import structlog

structlog.configure(
    processors=[
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ]
)

logger = structlog.get_logger()
```

---

## 실전 패턴

### 패턴 1: 컨텍스트 바인딩

```python
# 요청별 로거
logger = structlog.get_logger()
logger = logger.bind(request_id="abc-123", user_id=456)

logger.info("processing_start")
# {"event": "processing_start", "request_id": "abc-123", "user_id": 456}

logger.info("processing_complete", duration_ms=150)
# {"event": "processing_complete", "request_id": "abc-123", "user_id": 456, "duration_ms": 150}
```

### 패턴 2: FastAPI 통합

```python
from fastapi import FastAPI, Request
import structlog
import uuid

app = FastAPI()

@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    request_id = str(uuid.uuid4())

    logger = structlog.get_logger().bind(
        request_id=request_id,
        path=request.url.path,
        method=request.method
    )

    logger.info("request_started")
    response = await call_next(request)
    logger.info("request_completed", status_code=response.status_code)

    return response
```

### 패턴 3: 개발 vs 프로덕션

```python
import structlog
import sys

def configure_logging(dev_mode=False):
    processors = [
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
    ]

    if dev_mode:
        # 개발: 읽기 쉬운 형식
        processors.append(structlog.dev.ConsoleRenderer())
    else:
        # 프로덕션: JSON
        processors.append(structlog.processors.JSONRenderer())

    structlog.configure(processors=processors)

# 사용
configure_logging(dev_mode=('--dev' in sys.argv))
```

---

## 프로덕션 설정

```python
import structlog

structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)
```

---

**[← 로깅](../../04-library-catalog/logging/README.md)** | **[다음: SQLAlchemy →](../data-processing/sqlalchemy.md)**
