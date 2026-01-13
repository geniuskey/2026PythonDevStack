# 관측성 & 모니터링

## 2026 표준 스택

| 도구 | 등급 | 역할 |
|------|------|------|
| `structlog` | ⭐ | 구조화 로깅 |
| `opentelemetry` | ⭐ | 분산 추적 (tracing) |
| `prometheus-client` | ⭐ | 메트릭 수집 |

---

## 1. structlog: 구조화 로깅 ⭐

### 핵심 특징

- **JSON 로그**: 파싱 가능한 구조화 로그
- **컨텍스트 바인딩**: 요청별 메타데이터 자동 추가
- **읽기 쉬운 개발 모드**

### 기본 사용

```python
import structlog

# 설정
structlog.configure(
    processors=[
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ]
)

logger = structlog.get_logger()

# 로깅
logger.info("user_login", user_id=123, ip="192.168.1.1")
# → {"event": "user_login", "user_id": 123, "ip": "192.168.1.1", "timestamp": "..."}
```

### FastAPI 통합

```python
from fastapi import FastAPI, Request
import structlog
import uuid

app = FastAPI()

@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    request_id = str(uuid.uuid4())

    # 컨텍스트 바인딩
    logger = structlog.get_logger().bind(
        request_id=request_id,
        path=request.url.path,
        method=request.method,
    )

    logger.info("request_started")
    response = await call_next(request)
    logger.info("request_finished", status_code=response.status_code)

    return response
```

---

## 2. OpenTelemetry: 분산 추적 ⭐

### 핵심 특징

- **표준 프로토콜**: 벤더 중립적
- **자동 계측**: 프레임워크 자동 추적
- **Span과 Trace**: 요청 흐름 시각화

### 기본 설정

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter

# 설정
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Exporter 설정 (Jaeger, Zipkin 등)
span_processor = BatchSpanProcessor(ConsoleSpanExporter())
trace.get_tracer_provider().add_span_processor(span_processor)

# 사용
with tracer.start_as_current_span("operation"):
    # 작업 수행
    result = perform_operation()
```

### FastAPI 자동 계측

```bash
$ uv add opentelemetry-api opentelemetry-sdk \
          opentelemetry-instrumentation-fastapi
```

```python
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

app = FastAPI()
FastAPIInstrumentor.instrument_app(app)

# 자동으로 모든 엔드포인트 추적!
```

---

## 3. prometheus-client: 메트릭 ⭐

### 기본 메트릭

```python
from prometheus_client import Counter, Histogram, Gauge
from prometheus_client import start_http_server

# 카운터
requests_total = Counter('requests_total', 'Total requests', ['method', 'endpoint'])

# 히스토그램
request_duration = Histogram('request_duration_seconds', 'Request duration')

# 게이지
active_users = Gauge('active_users', 'Active users')

# 사용
requests_total.labels(method='GET', endpoint='/users').inc()

with request_duration.time():
    # 작업 수행
    pass

active_users.set(42)

# 메트릭 노출
start_http_server(8000)  # http://localhost:8000/metrics
```

---

**[← 비동기 I/O](async-io.md)** | **[데이터 처리 →](data-processing.md)**
