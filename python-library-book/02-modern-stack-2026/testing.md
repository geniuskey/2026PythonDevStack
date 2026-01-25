# 테스트 & 품질 보증

## 2026 표준 스택

| 도구 | 등급 | 역할 |
|------|------|------|
| `pytest` | - 2026 권장: | 테스팅 프레임워크 |
| `pytest-asyncio` | - 2026 권장: | 비동기 테스트 지원 |
| `hypothesis` | - 2026 권장: | 속성 기반 테스트 |
| `coverage` | - 2026 권장: | 코드 커버리지 측정 |

---

## 1. pytest: 표준 테스트 프레임워크 ⭐

### 핵심 특징

- **간결한 문법**: assert 문만으로 테스트
- **강력한 픽스처**: 재사용 가능한 테스트 설정
- **풍부한 플러그인**: 생태계 최대

### 설치 및 사용

```bash
$ uv add --dev pytest pytest-asyncio

# 테스트 실행
$ pytest

# 자세한 출력
$ pytest -v

# 특정 테스트
$ pytest tests/test_api.py::test_create_user
```

### 기본 테스트 작성

```python
# tests/test_calculator.py

def test_add():
    result = 2 + 2
    assert result == 4

def test_divide():
    result = 10 / 2
    assert result == 5.0

def test_divide_by_zero():
    with pytest.raises(ZeroDivisionError):
        1 / 0
```

### Fixture 사용

```python
import pytest

@pytest.fixture
def sample_user():
    return {"id": 1, "name": "Alice"}

def test_user_name(sample_user):
    assert sample_user["name"] == "Alice"

@pytest.fixture(scope="session")
def db_connection():
    # 세션 전체에서 한 번만 실행
    conn = create_connection()
    yield conn
    conn.close()
```

### 비동기 테스트

```python
import pytest
import httpx

@pytest.mark.asyncio
async def test_api_call():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com")
        assert response.status_code == 200
```

---

## 2. hypothesis: 속성 기반 테스트 ⭐

### 핵심 특징

- **자동으로 테스트 케이스 생성**
- **엣지 케이스 발견**: 예상 못한 버그 찾기

### 설치 및 사용

```bash
$ uv add --dev hypothesis
```

### 예시

```python
from hypothesis import given
from hypothesis import strategies as st

# 일반 테스트: 특정 케이스만
def test_reverse():
    assert list(reversed([1, 2, 3])) == [3, 2, 1]

# hypothesis: 수많은 케이스 자동 생성
@given(st.lists(st.integers()))
def test_reverse_property(lst):
    # 어떤 리스트든: reverse를 두 번 하면 원본
    assert list(reversed(list(reversed(lst)))) == lst
```

---

## 3. coverage: 커버리지 측정 ⭐

### 설치 및 사용

```bash
$ uv add --dev coverage[toml]

# 커버리지 측정하며 테스트
$ coverage run -m pytest

# 리포트 보기
$ coverage report

# HTML 리포트 생성
$ coverage html
$ open htmlcov/index.html
```

### pyproject.toml 설정

```toml
[tool.coverage.run]
source = ["src"]
omit = ["tests/*", "**/__init__.py"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
]
```

---

## 실전 구조

```
myproject/
├── src/
│   └── myproject/
│       ├── __init__.py
│       ├── api.py
│       └── models.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py          # 공통 fixture
│   ├── test_api.py
│   └── test_models.py
└── pyproject.toml
```

### conftest.py 예시

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from myproject.api import app

@pytest.fixture
def client():
    return TestClient(app)

@pytest.fixture
def sample_data():
    return {
        "users": [
            {"id": 1, "name": "Alice"},
            {"id": 2, "name": "Bob"},
        ]
    }
```

---

## CI/CD 통합

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        run: curl -LsSf https://astral.sh/uv/install.sh | sh

      - name: Install dependencies
        run: uv sync

      - name: Run tests with coverage
        run: |
          uv run coverage run -m pytest
          uv run coverage report
          uv run coverage html

      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

---

**[← 코드 품질](code-quality.md)** | **[웹 API →](web-api.md)**
