# 패키지 배포하기

## 목표

Python 패키지를 PyPI에 배포하여 `pip install mypackage`로 설치 가능하게 만들기

---

## 기술 스택

| 도구 | 역할 |
|------|------|
| `pyproject.toml` | 패키지 메타데이터 |
| `build` | 패키지 빌드 |
| `twine` | PyPI 업로드 |
| `uv` | 개발 환경 관리 |

---

## 1단계: 패키지 구조

```
mypackage/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── core.py
│       └── utils.py
├── tests/
│   └── test_core.py
├── README.md
├── LICENSE
├── .gitignore
└── pyproject.toml
```

---

## 2단계: pyproject.toml 작성

```toml
[project]
name = "mypackage"
version = "0.1.0"
description = "A brief description of your package"
readme = "README.md"
requires-python = ">=3.10"
license = {text = "MIT"}
keywords = ["example", "package"]
authors = [
    {name = "Your Name", email = "you@example.com"}
]
classifiers = [
    "Development Status :: 3 - Alpha",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]

dependencies = [
    "requests>=2.31.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "ruff>=0.1.0",
    "build>=1.0.0",
    "twine>=4.0.0",
]

[project.urls]
Homepage = "https://github.com/yourusername/mypackage"
Documentation = "https://mypackage.readthedocs.io"
Repository = "https://github.com/yourusername/mypackage.git"
Issues = "https://github.com/yourusername/mypackage/issues"

[project.scripts]
mypackage = "mypackage.cli:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatchling.build.targets.wheel]
packages = ["src/mypackage"]
```

---

## 3단계: 패키지 코드 작성

```python
# src/mypackage/__init__.py
"""MyPackage - A simple example package."""

__version__ = "0.1.0"

from .core import hello, add

__all__ = ["hello", "add", "__version__"]
```

```python
# src/mypackage/core.py
def hello(name: str) -> str:
    """Say hello to someone."""
    return f"Hello, {name}!"

def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b
```

---

## 4단계: 테스트 작성

```python
# tests/test_core.py
from mypackage import hello, add

def test_hello():
    assert hello("World") == "Hello, World!"

def test_add():
    assert add(2, 3) == 5
```

---

## 5단계: README.md

```markdown
# MyPackage

A brief description of what your package does.

## Installation

\`\`\`bash
pip install mypackage
\`\`\`

## Usage

\`\`\`python
from mypackage import hello

print(hello("World"))
# Output: Hello, World!
\`\`\`

## Development

\`\`\`bash
# Clone repository
git clone https://github.com/yourusername/mypackage.git
cd mypackage

# Install dependencies
uv sync

# Run tests
uv run pytest
\`\`\`

## License

MIT
```

---

## 6단계: LICENSE 파일

```
MIT License

Copyright (c) 2026 Your Name

Permission is hereby granted, free of charge, to any person obtaining a copy...
```

---

## 7단계: 빌드

```bash
# 개발 의존성 설치
$ uv add --dev build twine

# 빌드 (wheel + source distribution 생성)
$ uv run python -m build

# 결과 확인
$ ls dist/
mypackage-0.1.0-py3-none-any.whl
mypackage-0.1.0.tar.gz
```

---

## 8단계: 검증

```bash
# 빌드물 검증
$ uv run twine check dist/*

# 로컬 설치 테스트
$ uv pip install dist/mypackage-0.1.0-py3-none-any.whl

# Python에서 테스트
$ python -c "from mypackage import hello; print(hello('Test'))"
```

---

## 9단계: TestPyPI에 업로드 (테스트)

```bash
# TestPyPI 계정 생성 (https://test.pypi.org)
# API 토큰 생성

# ~/.pypirc 파일 생성
[testpypi]
username = __token__
password = pypi-AgE...your-test-token...

# 업로드
$ uv run twine upload --repository testpypi dist/*

# 설치 테스트
$ pip install --index-url https://test.pypi.org/simple/ mypackage
```

---

## 10단계: PyPI에 업로드 (프로덕션)

```bash
# PyPI 계정 생성 (https://pypi.org)
# API 토큰 생성

# ~/.pypirc 업데이트
[pypi]
username = __token__
password = pypi-AgE...your-production-token...

# 업로드
$ uv run twine upload dist/*

# 완료! 이제 누구나 설치 가능
$ pip install mypackage
```

---

## 버전 업데이트 워크플로우

```bash
# 1. 코드 수정
# 2. 테스트
$ uv run pytest

# 3. 버전 업데이트
$ vim pyproject.toml  # version = "0.1.1"
$ vim src/mypackage/__init__.py  # __version__ = "0.1.1"

# 4. Git 태그
$ git add .
$ git commit -m "Bump version to 0.1.1"
$ git tag v0.1.1
$ git push && git push --tags

# 5. 기존 빌드 삭제
$ rm -rf dist/

# 6. 새로 빌드
$ uv run python -m build

# 7. 업로드
$ uv run twine upload dist/*
```

---

## GitHub Actions 자동화

```yaml
# .github/workflows/publish.yml
name: Publish to PyPI

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        run: curl -LsSf https://astral.sh/uv/install.sh | sh

      - name: Install dependencies
        run: uv sync

      - name: Run tests
        run: uv run pytest

      - name: Build package
        run: uv run python -m build

      - name: Publish to PyPI
        env:
          TWINE_USERNAME: __token__
          TWINE_PASSWORD: ${{ secrets.PYPI_API_TOKEN }}
        run: uv run twine upload dist/*
```

---

## 베스트 프랙티스

### 1. 시멘틱 버저닝

- `0.1.0` → `0.1.1`: 버그 수정
- `0.1.0` → `0.2.0`: 새 기능 추가
- `0.9.0` → `1.0.0`: 주요 변경 (Breaking changes)

### 2. CHANGELOG.md 작성

```markdown
# Changelog

## [0.1.1] - 2026-01-13
### Fixed
- Bug fix in hello function

## [0.1.0] - 2026-01-01
### Added
- Initial release
- hello() function
- add() function
```

### 3. 타입 힌트 포함

```python
def hello(name: str) -> str:
    return f"Hello, {name}!"
```

### 4. Docstring 작성

```python
def hello(name: str) -> str:
    """
    Say hello to someone.

    Args:
        name: The name of the person to greet

    Returns:
        A greeting message

    Example:
        >>> hello("World")
        'Hello, World!'
    """
    return f"Hello, {name}!"
```

---

## 체크리스트

배포 전 확인 사항:

- [ ] 테스트 모두 통과
- [ ] README.md 작성 완료
- [ ] LICENSE 파일 포함
- [ ] pyproject.toml 메타데이터 완성
- [ ] 버전 번호 업데이트
- [ ] CHANGELOG.md 업데이트
- [ ] Git 태그 생성
- [ ] TestPyPI에서 테스트
- [ ] 실제 설치 테스트

---

**[← API 서버](api-server.md)** | **[LLM 서비스 →](llm-service.md)**
