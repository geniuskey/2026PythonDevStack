# 프로젝트 관리 & 패키징

## 이 분야에서 해결하는 문제

- 📦 **패키지 설치 및 관리**: 의존성 어떻게 관리?
- 🔒 **재현 가능한 환경**: 다른 머신에서도 동일하게?
- 📤 **패키지 배포**: PyPI에 어떻게 올리지?
- ⚙️ **프로젝트 설정**: 메타데이터는 어디에?

---

## 2026 표준 스택

| 도구 | 등급 | 역할 | 대체 |
|------|------|------|------|
| `uv` | ⭐ | 초고속 패키지 관리자 | pip, pipenv, poetry |
| `pyproject.toml` | ⭐ | 통합 설정 파일 | setup.py, requirements.txt |
| `build` | ⭐ | 패키지 빌드 도구 | setuptools 직접 사용 |
| `twine` | ⭐ | PyPI 업로드 | - |

---

## 1. uv: 게임 체인저 ⭐

### 무엇인가?

[`uv`](https://github.com/astral-sh/uv)는 **Rust로 작성된 초고속 파이썬 패키지 관리자**입니다.

### 왜 ⭐ 인가?

**속도가 압도적입니다**

```
pip: 30-60초
pipenv: 60-120초
poetry: 20-40초
uv: 1-2초  ← 10-100배 빠름
```

**기능이 완전합니다**

- ✅ 패키지 설치 (pip 대체)
- ✅ 가상환경 관리 (venv 대체)
- ✅ 의존성 해결 (poetry 대체)
- ✅ 프로젝트 초기화 (cookiecutter 대체)
- ✅ `pyproject.toml` 완벽 지원

### 설치

```bash
# macOS / Linux
$ curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
$ powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# 검증
$ uv --version
```

### 기본 사용법

#### 새 프로젝트 시작

```bash
# 프로젝트 생성
$ uv init myproject
$ cd myproject

# 생성되는 구조
myproject/
├── pyproject.toml
├── README.md
└── src/
    └── myproject/
        └── __init__.py
```

#### 패키지 설치

```bash
# 패키지 추가
$ uv add fastapi httpx

# 개발 의존성 추가
$ uv add --dev pytest ruff

# 그룹별 의존성
$ uv add --group docs sphinx
```

#### 가상환경 관리

```bash
# 자동으로 .venv 생성 및 활성화
$ uv sync

# 패키지 설치 (가상환경 없이도 작동)
$ uv run python main.py
$ uv run pytest

# 명시적 가상환경 활성화
$ source .venv/bin/activate  # Unix
$ .venv\Scripts\activate     # Windows
```

#### 의존성 잠금

```bash
# uv.lock 파일 생성 (자동)
$ uv lock

# 잠긴 버전으로 설치
$ uv sync --locked
```

### pyproject.toml 예시

```toml
[project]
name = "myproject"
version = "0.1.0"
description = "My awesome project"
authors = [
    {name = "Your Name", email = "you@example.com"}
]
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.109.0",
    "httpx>=0.26.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "ruff>=0.1.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.uv]
dev-dependencies = [
    "mypy>=1.8.0",
]
```

### 마이그레이션

#### pip + requirements.txt에서

```bash
# 기존 프로젝트에서
$ uv init

# requirements.txt를 pyproject.toml로 변환
$ uv add $(cat requirements.txt)

# 또는 pip install 대신
$ uv pip install -r requirements.txt
```

#### pipenv에서

```bash
# Pipfile을 읽어서 변환
$ uv init
$ uv add $(pipenv requirements)
```

#### poetry에서

```bash
# pyproject.toml은 유지 가능
# poetry → uv로 점진적 전환
$ uv sync  # poetry.lock과 호환
```

### 실전 워크플로우

```bash
# 1. 프로젝트 생성
$ uv init myapi
$ cd myapi

# 2. 의존성 추가
$ uv add fastapi "uvicorn[standard]"
$ uv add --dev pytest httpx

# 3. 코드 작성
$ cat > src/myapi/main.py << 'EOF'
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}
EOF

# 4. 실행
$ uv run uvicorn myapi.main:app --reload

# 5. 테스트
$ uv run pytest

# 6. 린팅
$ uv run ruff check .
```

---

## 2. pyproject.toml: 통합 설정 ⭐

### 무엇인가?

**모든 프로젝트 설정을 하나의 파일로 통일**하는 표준 포맷 (PEP 518, 621)

### 과거의 문제

```
프로젝트 루트/
├── setup.py          # 패키지 메타데이터
├── setup.cfg         # 추가 설정
├── requirements.txt  # 의존성
├── dev-requirements.txt
├── .flake8          # 린터 설정
├── .pylintrc
├── pytest.ini       # 테스트 설정
├── mypy.ini         # 타입 체커
└── ...              # 끝없는 설정 파일
```

### 현재의 해결책

```toml
# pyproject.toml 하나로 통일
[project]
# 패키지 메타데이터

[build-system]
# 빌드 설정

[tool.ruff]
# 린터 설정

[tool.pytest.ini_options]
# 테스트 설정

[tool.mypy]
# 타입 체커 설정
```

### 완전한 예시

```toml
# ==========================================
# 프로젝트 메타데이터 (PEP 621)
# ==========================================
[project]
name = "myapp"
version = "1.0.0"
description = "Production-ready Python application"
authors = [
    {name = "John Doe", email = "john@example.com"}
]
readme = "README.md"
license = {text = "MIT"}
requires-python = ">=3.12"
keywords = ["api", "fastapi", "async"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "Programming Language :: Python :: 3.12",
]

# 런타임 의존성
dependencies = [
    "fastapi>=0.109.0",
    "httpx>=0.26.0",
    "pydantic>=2.5.0",
    "structlog>=24.1.0",
]

# 선택적 의존성
[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-asyncio>=0.23.0",
    "httpx",
]
docs = [
    "mkdocs>=1.5.0",
    "mkdocs-material>=9.5.0",
]

# CLI 진입점
[project.scripts]
myapp = "myapp.cli:main"

[project.urls]
Homepage = "https://github.com/user/myapp"
Documentation = "https://myapp.readthedocs.io"
Repository = "https://github.com/user/myapp.git"
Issues = "https://github.com/user/myapp/issues"

# ==========================================
# 빌드 시스템
# ==========================================
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

# ==========================================
# 도구별 설정
# ==========================================

# Ruff (린터 + 포매터)
[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W"]
ignore = ["E203", "E501"]

# pytest
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "--strict-markers",
    "--strict-config",
    "-ra",
]

# mypy
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true

# coverage
[tool.coverage.run]
source = ["src"]
omit = ["tests/*"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
]
```

### 마이그레이션 체크리스트

#### setup.py → pyproject.toml

```python
# setup.py (과거)
from setuptools import setup, find_packages

setup(
    name="myapp",
    version="1.0.0",
    packages=find_packages(),
    install_requires=[
        "fastapi>=0.109.0",
    ],
)
```

↓

```toml
# pyproject.toml (현재)
[project]
name = "myapp"
version = "1.0.0"
dependencies = [
    "fastapi>=0.109.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

---

## 3. build: 패키지 빌드 ⭐

### 무엇인가?

PEP 517 표준에 따라 파이썬 패키지를 빌드하는 도구

### 사용법

```bash
# 설치
$ uv add --dev build

# 빌드 (wheel + sdist 생성)
$ uv run python -m build

# 결과
dist/
├── myapp-1.0.0-py3-none-any.whl
└── myapp-1.0.0.tar.gz
```

### pyproject.toml 설정

```toml
[build-system]
requires = ["hatchling"]  # 또는 setuptools, flit, pdm
build-backend = "hatchling.build"

[tool.hatchling.build.targets.wheel]
packages = ["src/myapp"]
```

---

## 4. twine: PyPI 업로드 ⭐

### 무엇인가?

안전하게 PyPI에 패키지를 업로드하는 도구

### 사용법

```bash
# 설치
$ uv add --dev twine

# 빌드물 검증
$ uv run twine check dist/*

# TestPyPI에 먼저 업로드 (테스트)
$ uv run twine upload --repository testpypi dist/*

# PyPI에 업로드 (프로덕션)
$ uv run twine upload dist/*
```

### PyPI 토큰 설정

```bash
# ~/.pypirc
[pypi]
username = __token__
password = pypi-AgE...your-token-here...

[testpypi]
username = __token__
password = pypi-AgE...your-test-token...
```

---

## 대안 도구들

### poetry ⚠️

**여전히 훌륭하지만, uv가 더 빠름**

```toml
# pyproject.toml (poetry 스타일)
[tool.poetry]
name = "myapp"
version = "1.0.0"

[tool.poetry.dependencies]
python = "^3.12"
fastapi = "^0.109.0"
```

**언제 poetry를 선택?**
- 기존 프로젝트가 poetry
- 팀이 이미 익숙함
- poetry 전용 플러그인 사용

### pipenv ⚠️

**느리고, 유지보수가 느려짐**

```bash
# Pipfile (pipenv)
[packages]
fastapi = ">=0.109.0"

[dev-packages]
pytest = "*"
```

**언제 pipenv를 선택?**
- 레거시 프로젝트 유지보수
- 특별한 이유 없으면 마이그레이션 권장

### pip + venv ⚠️

**기본이지만, 현대 기능 부족**

```bash
# 여전히 작동하지만...
$ python -m venv .venv
$ source .venv/bin/activate
$ pip install -r requirements.txt
```

**언제 pip를 선택?**
- 단순 스크립트
- 가벼운 환경 (CI 등)
- 표준 라이브러리만 써야 할 때

---

## 실전 시나리오

### 시나리오 1: 새 프로젝트 시작

```bash
# 1. 프로젝트 생성
$ uv init myapp
$ cd myapp

# 2. 의존성 추가
$ uv add fastapi "uvicorn[standard]"

# 3. 개발 의존성 추가
$ uv add --dev pytest ruff mypy

# 4. 코드 작성
# ...

# 5. 테스트
$ uv run pytest

# 완료! 의존성은 pyproject.toml에 자동 저장
```

### 시나리오 2: 팀 협업

```bash
# 1. 저장소 클론
$ git clone https://github.com/team/project.git
$ cd project

# 2. 의존성 설치 (uv.lock 기준)
$ uv sync --locked

# 3. 개발 시작
$ uv run python main.py

# 모든 팀원이 동일한 환경!
```

### 시나리오 3: CI/CD

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
        run: uv sync --locked

      - name: Run tests
        run: uv run pytest
```

### 시나리오 4: 패키지 배포

```bash
# 1. 버전 업데이트
$ vim pyproject.toml  # version = "1.0.1"

# 2. 빌드
$ uv run python -m build

# 3. 검증
$ uv run twine check dist/*

# 4. 업로드
$ uv run twine upload dist/*

# 완료! pip install myapp 가능
```

---

## 베스트 프랙티스

### 1. 항상 pyproject.toml 사용

```toml
# ✅ 좋음: 모든 설정 한 곳에
[project]
dependencies = [...]

[tool.ruff]
line-length = 88

# ❌ 나쁨: 파편화된 설정
# requirements.txt
# .flake8
# setup.py
```

### 2. 의존성 버전 고정

```toml
# 라이브러리는 범위 지정
[project]
dependencies = [
    "fastapi>=0.109.0,<1.0.0",
]

# 애플리케이션은 엄격하게
dependencies = [
    "fastapi==0.109.2",
]
```

### 3. uv.lock 파일 커밋

```bash
# .gitignore
.venv/
__pycache__/
*.pyc

# uv.lock은 커밋 (재현성)
```

### 4. 개발 의존성 분리

```toml
[project.optional-dependencies]
dev = ["pytest", "ruff"]
docs = ["mkdocs"]
```

---

## 다음 단계

프로젝트 관리를 마스터했다면, 코드 품질을 높일 차례입니다.

**[코드 품질 & 린팅 →](code-quality.md)**

---

**[← 2026 표준 스택 개요](overview.md)** | **[코드 품질 →](code-quality.md)**
