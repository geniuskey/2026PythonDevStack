# pytest 완벽 가이드

> **파이썬 테스팅의 사실상 표준**


---

## 목차

- [개요](#개요)
- [왜 pytest인가](#왜-pytest인가)
- [핵심 개념](#핵심-개념)
- [설치 및 환경 설정](#설치-및-환경-설정)
- [기본 사용법](#기본-사용법)
- [실전 패턴 10가지](#실전-패턴-10가지)
- [함정 및 주의사항](#함정-및-주의사항)
- [플러그인 생태계](#플러그인-생태계)
- [다른 테스팅 프레임워크와 비교](#다른-테스팅-프레임워크와-비교)

---

## 개요

### 기본 정보

| 항목 | 내용 |
|------|------|
| **공식 사이트** | https://pytest.org |
| **GitHub** | https://github.com/pytest-dev/pytest |
| **PyPI** | https://pypi.org/project/pytest/ |
| **첫 릴리즈** | 2004년 |
| **라이선스** | MIT |
| **Python 버전** | 3.8+ |

### 한 줄 요약

**간결한 문법과 강력한 기능을 가진 파이썬 테스트 프레임워크**

---

## 왜 pytest인가

### unittest vs pytest

```python
# unittest (표준 라이브러리)
import unittest

class TestCalculator(unittest.TestCase):
    def test_add(self):
        self.assertEqual(2 + 2, 4)

# pytest (훨씬 간결)
def test_add():
    assert 2 + 2 == 4
```

**pytest의 장점:**
- - `assert` 문만으로 충분 (self.assertEqual 불필요)
- - 클래스 없이 함수만으로 테스트
- - 자동 테스트 발견
- - 상세한 실패 메시지
- - Fixture 시스템
- - 풍부한 플러그인 생태계

---

## 핵심 개념

### 1. 간결한 Assert

```python
# pytest는 assert를 "똑똑하게" 만듦
def test_string():
    result = "hello world"
    assert result == "hello python"
    # 실패 시:
    # AssertionError: assert 'hello world' == 'hello python'
    #   - hello python
    #   + hello world
```

### 2. Fixture (테스트 전제조건)

```python
import pytest

@pytest.fixture
def sample_data():
    return {"name": "Alice", "age": 30}

def test_with_fixture(sample_data):
    # sample_data가 자동으로 주입됨
    assert sample_data["name"] == "Alice"
```

### 3. 테스트 파라미터화

```python
@pytest.mark.parametrize("input,expected", [
    (1, 2),
    (2, 4),
    (3, 6),
])
def test_double(input, expected):
    assert input * 2 == expected
    # 3개의 테스트가 자동 생성됨
```

---

## 설치 및 환경 설정

### 기본 설치

```bash
$ uv add --dev pytest pytest-cov pytest-asyncio
```

### pyproject.toml 설정

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "-v",                    # verbose
    "--strict-markers",      # 등록된 마커만 허용
    "--tb=short",            # 짧은 traceback
    "--cov=src",             # 커버리지
    "--cov-report=html",     # HTML 리포트
    "--cov-report=term",     # 터미널 리포트
]
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]
```

### 프로젝트 구조

```
myproject/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── calculator.py
│       └── user.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py        # 전역 fixture
│   ├── unit/
│   │   ├── test_calculator.py
│   │   └── test_user.py
│   └── integration/
│       └── test_api.py
└── pyproject.toml
```

---

## 기본 사용법

### 1단계: 첫 테스트

```python
# tests/test_calculator.py
def test_add():
    result = 2 + 2
    assert result == 4

def test_subtract():
    result = 5 - 3
    assert result == 2
```

```bash
# 실행
$ pytest

# verbose 모드
$ pytest -v

# 특정 파일
$ pytest tests/test_calculator.py

# 특정 테스트
$ pytest tests/test_calculator.py::test_add
```

### 2단계: Fixture 사용

```python
# tests/conftest.py
import pytest

@pytest.fixture
def calculator():
    return Calculator()

@pytest.fixture
def sample_users():
    return [
        {"name": "Alice", "age": 30},
        {"name": "Bob", "age": 25},
    ]

# tests/test_calculator.py
def test_calculator_add(calculator):
    result = calculator.add(2, 3)
    assert result == 5
```

### 3단계: 파라미터화

```python
@pytest.mark.parametrize("a,b,expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
    (100, 200, 300),
])
def test_add_parametrized(calculator, a, b, expected):
    assert calculator.add(a, b) == expected
```

### 4단계: 예외 테스트

```python
def test_divide_by_zero(calculator):
    with pytest.raises(ZeroDivisionError):
        calculator.divide(10, 0)

# 예외 메시지도 검증
def test_invalid_input(calculator):
    with pytest.raises(ValueError, match="must be a number"):
        calculator.add("a", "b")
```

### 5단계: 비동기 테스트

```python
import pytest

@pytest.mark.asyncio
async def test_async_function():
    result = await async_add(2, 3)
    assert result == 5
```

---

## 실전 패턴 10가지

### 패턴 1: Setup/Teardown (Fixture)

```python
@pytest.fixture
def database():
    # Setup
    db = create_database()
    db.connect()

    yield db  # 테스트에 전달

    # Teardown
    db.disconnect()
    db.delete()

def test_query(database):
    result = database.query("SELECT * FROM users")
    assert len(result) > 0
```

### 패턴 2: Fixture 스코프

```python
# 함수마다 새로 생성 (기본)
@pytest.fixture(scope="function")
def temp_file():
    f = open("temp.txt", "w")
    yield f
    f.close()

# 모듈당 한 번
@pytest.fixture(scope="module")
def db_connection():
    conn = create_connection()
    yield conn
    conn.close()

# 세션당 한 번 (전체 테스트 실행에서 한 번)
@pytest.fixture(scope="session")
def docker_container():
    container = start_docker()
    yield container
    stop_docker(container)
```

### 패턴 3: Fixture 체인

```python
@pytest.fixture
def database():
    return Database()

@pytest.fixture
def user_repository(database):
    return UserRepository(database)

@pytest.fixture
def user_service(user_repository):
    return UserService(user_repository)

def test_user_service(user_service):
    user = user_service.create_user("Alice")
    assert user.name == "Alice"
```

### 패턴 4: Parametrize 고급

```python
# 여러 매개변수
@pytest.mark.parametrize("username,email", [
    ("alice", "alice@example.com"),
    ("bob", "bob@example.com"),
])
def test_user_creation(username, email):
    user = User(username, email)
    assert user.email == email

# ID 추가 (테스트 식별 용이)
@pytest.mark.parametrize("value,expected", [
    pytest.param(0, False, id="zero"),
    pytest.param(1, True, id="positive"),
    pytest.param(-1, False, id="negative"),
])
def test_is_positive(value, expected):
    assert is_positive(value) == expected
```

### 패턴 5: 조건부 Skip

```python
import sys

@pytest.mark.skipif(sys.platform == "win32", reason="Unix only")
def test_unix_feature():
    assert os.fork() > 0

@pytest.mark.skip(reason="Not implemented yet")
def test_future_feature():
    pass

# 특정 조건에서만 실행
@pytest.mark.skipif(not has_gpu(), reason="Requires GPU")
def test_gpu_computation():
    pass
```

### 패턴 6: 커스텀 마커

```python
# pyproject.toml에 등록
# markers = ["slow", "integration", "unit"]

@pytest.mark.slow
def test_heavy_computation():
    # 오래 걸리는 테스트
    pass

@pytest.mark.integration
def test_api_integration():
    # 외부 API 호출
    pass

# 실행 시 필터링
# pytest -m "not slow"  # slow 제외
# pytest -m integration  # integration만
```

### 패턴 7: Mocking

```python
from unittest.mock import Mock, patch

def test_with_mock():
    # Mock 객체 생성
    mock_db = Mock()
    mock_db.query.return_value = [{"name": "Alice"}]

    service = UserService(mock_db)
    users = service.get_all_users()

    assert len(users) == 1
    mock_db.query.assert_called_once()

# patch로 함수 교체
@patch('mymodule.external_api_call')
def test_with_patch(mock_api):
    mock_api.return_value = {"status": "success"}

    result = my_function()
    assert result == "success"
```

### 패턴 8: Tmp 디렉토리

```python
def test_file_operation(tmp_path):
    # tmp_path는 임시 디렉토리 (자동 정리됨)
    test_file = tmp_path / "test.txt"
    test_file.write_text("hello")

    assert test_file.read_text() == "hello"
    # 테스트 종료 후 자동 삭제
```

### 패턴 9: Monkeypatch

```python
def test_with_monkeypatch(monkeypatch):
    # 환경 변수 임시 변경
    monkeypatch.setenv("API_KEY", "test-key")

    # 함수 임시 교체
    monkeypatch.setattr("module.function", lambda: "mocked")

    # 테스트 종료 후 자동 복원
```

### 패턴 10: FastAPI 테스트

```python
from fastapi.testclient import TestClient
from myapi.main import app

@pytest.fixture
def client():
    return TestClient(app)

def test_create_user(client):
    response = client.post(
        "/users/",
        json={"name": "Alice", "email": "alice@example.com"}
    )
    assert response.status_code == 201
    assert response.json()["name"] == "Alice"
```

---

## 함정 및 주의사항

### - 미지원: 함정 1: Fixture Scope 실수

```python
# 나쁨: 데이터가 테스트 간 공유됨
@pytest.fixture(scope="module")
def mutable_data():
    return []  # 모든 테스트가 같은 리스트 공유!

def test_1(mutable_data):
    mutable_data.append(1)
    assert len(mutable_data) == 1  # OK

def test_2(mutable_data):
    mutable_data.append(2)
    assert len(mutable_data) == 1  # FAIL! (길이가 2)

# 좋음: 함수 스코프 (기본값)
@pytest.fixture  # scope="function"이 기본
def mutable_data():
    return []  # 매 테스트마다 새로 생성
```

### - 미지원: 함정 2: Assert 메시지 오용

```python
# 나쁨: assert 메시지는 거의 필요없음
def test_add():
    result = add(2, 3)
    assert result == 5, "Addition failed"  # 불필요

# pytest는 이미 상세한 정보 제공:
# AssertionError: assert 6 == 5
#   where 6 = add(2, 3)

# 좋음: 복잡한 조건에서만 사용
def test_complex():
    data = complex_operation()
    assert is_valid(data), f"Invalid data: {data}"
```

### - 미지원: 함정 3: 테스트 순서 의존

```python
# 나쁨: 테스트 순서에 의존
class TestUser:
    def test_1_create_user(self):
        self.user = User("Alice")

    def test_2_update_user(self):
        self.user.update("Bob")  # test_1이 먼저 실행되어야 함!

# 좋음: 각 테스트가 독립적
@pytest.fixture
def user():
    return User("Alice")

def test_create_user(user):
    assert user.name == "Alice"

def test_update_user(user):
    user.update("Bob")
    assert user.name == "Bob"
```

---

## 플러그인 생태계

### 필수 플러그인

```bash
# 커버리지
$ uv add --dev pytest-cov

# 비동기 테스트
$ uv add --dev pytest-asyncio

# 병렬 실행
$ uv add --dev pytest-xdist

# FastAPI 테스트
$ uv add --dev httpx

# Mocking 개선
$ uv add --dev pytest-mock
```

### 사용 예시

```bash
# 커버리지 측정
$ pytest --cov=src --cov-report=html

# 병렬 실행 (4개 워커)
$ pytest -n 4

# 실패한 테스트만 재실행
$ pytest --lf

# 가장 느린 10개 테스트 표시
$ pytest --durations=10
```

---

## 다른 테스팅 프레임워크와 비교

### pytest vs unittest

| 특징 | pytest | unittest |
|------|--------|----------|
| 문법 | 간결 | 장황 |
| 클래스 필요 | - 미지원: | - |
| Assert | assert | self.assertEqual |
| Fixture | 강력 | setUp/tearDown |
| 생태계 | 풍부 | 제한적 |

### pytest vs nose2

nose2는 사실상 deprecated. pytest 사용 권장.

---

## 실전 CI/CD 통합

### GitHub Actions

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: |
          pip install -e .[dev]

      - name: Run tests
        run: |
          pytest --cov=src --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

---

## 베스트 프랙티스

### 1. 테스트 이름

```python
# 좋음: 설명적
def test_user_creation_with_valid_email():
    pass

def test_user_creation_fails_with_invalid_email():
    pass

# 나쁨: 모호함
def test_1():
    pass
```

### 2. AAA 패턴

```python
def test_user_update():
    # Arrange (준비)
    user = User("Alice")

    # Act (실행)
    user.update("Bob")

    # Assert (검증)
    assert user.name == "Bob"
```

### 3. 한 테스트, 한 검증

```python
# 나쁨: 여러 검증
def test_user():
    user = User("Alice")
    assert user.name == "Alice"
    user.update("Bob")
    assert user.name == "Bob"
    assert user.is_updated == True

# 좋음: 분리
def test_user_creation():
    user = User("Alice")
    assert user.name == "Alice"

def test_user_update():
    user = User("Alice")
    user.update("Bob")
    assert user.name == "Bob"
```

---

**[← 테스팅 카탈로그](../../04-library-catalog/testing/README.md)** | **[다음: hypothesis →](hypothesis.md)**
