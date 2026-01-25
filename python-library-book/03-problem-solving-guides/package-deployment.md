# 패키지 배포하기

## 목표

Python 패키지를 PyPI에 배포하여 전 세계 누구나 `pip install mypackage`로 설치할 수 있게 만들기

Python 패키지를 만드는 건 생각보다 복잡합니다. 예전에는 `setup.py`로 했는데 2026년에는 `pyproject.toml` 하나로 모든 걸 관리합니다. 이 가이드에서는 최신 방식으로 패키지를 만들고, 테스트하고, PyPI에 올리는 전체 과정을 다룹니다.

---

## 2026년 Python 패키징 현황

### 핵심 도구

| 도구 | 역할 | 2026 추천 | 대안 |
|------|------|-----------|------|
| `pyproject.toml` | 패키지 메타데이터, 설정 통합 | Yes | setup.py (레거시) |
| `hatchling` | 빌드 백엔드 (빠르고 간단) | Yes | setuptools, pdm-backend |
| `build` | 패키지 빌드 도구 | Yes | python -m build |
| `twine` | PyPI 업로드 | Yes | 직접 업로드 (비권장) |
| `uv` | 개발 환경 관리 | Yes | pip-tools, poetry |
| `bump2version` | 버전 관리 자동화 | Yes | 수동 편집 |

### 빌드 백엔드 비교

**Hatchling** (권장):
- 설정 최소화
- 빌드 속도 빠름
- 플러그인 시스템
- 동적 버전 관리 지원

**Setuptools**:
- 가장 오래됨 (호환성 최고)
- 복잡한 빌드 지원 (C 확장)
- 설정이 많음

**PDM-backend**:
- PEP 621 완벽 지원
- PDM 에코시스템과 통합
- 상대적으로 새로움

결론: 일반 Python 패키지는 Hatchling, C 확장이 있으면 Setuptools 사용

---

## 패키지 종류별 구조

### 간단한 라이브러리

```
mylib/
├── src/
│   └── mylib/
│       ├── __init__.py
│       └── core.py
├── tests/
│   └── test_core.py
├── README.md
├── LICENSE
└── pyproject.toml
```

### CLI 도구

```
mycli/
├── src/
│   └── mycli/
│       ├── __init__.py
│       ├── cli.py          # Entry point
│       ├── commands/
│       │   ├── __init__.py
│       │   ├── init.py
│       │   └── deploy.py
│       └── utils.py
├── tests/
├── README.md
└── pyproject.toml
```

### 플러그인 시스템이 있는 프레임워크

```
myframework/
├── src/
│   └── myframework/
│       ├── __init__.py
│       ├── core.py
│       ├── plugin.py       # Plugin interface
│       └── plugins/
│           ├── __init__.py
│           ├── builtin.py
│           └── discover.py
├── tests/
└── pyproject.toml
```

**src/ 레이아웃을 쓰는 이유**:
- Import 혼란 방지 (로컬 소스 vs 설치된 패키지)
- 테스트가 실제 설치된 패키지를 사용하도록 강제
- 빌드물에 테스트 코드 포함 방지

---

## pyproject.toml 완전 가이드

### 기본 설정

```toml
[project]
name = "mypackage"
version = "0.1.0"
description = "A brief but clear description of what your package does"
readme = "README.md"
requires-python = ">=3.10"
license = {text = "MIT"}
keywords = ["data", "processing", "utility"]  # PyPI 검색 키워드
authors = [
    {name = "Your Name", email = "you@example.com"}
]
maintainers = [
    {name = "Maintainer Name", email = "maintainer@example.com"}
]

# PyPI 분류 (https://pypi.org/classifiers/)
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Topic :: Software Development :: Libraries :: Python Modules",
    "Typing :: Typed",  # Type hints 포함
]

dependencies = [
    "requests>=2.31.0,<3.0.0",
    "click>=8.1.0",
]

[project.optional-dependencies]
# 개발 도구
dev = [
    "pytest>=7.4.0",
    "pytest-cov>=4.1.0",
    "ruff>=0.1.0",
    "mypy>=1.7.0",
    "build>=1.0.0",
    "twine>=4.0.0",
    "bump2version>=1.0.0",
]

# 문서 생성
docs = [
    "mkdocs>=1.5.0",
    "mkdocs-material>=9.5.0",
]

# 성능 최적화 (선택적)
perf = [
    "numpy>=1.26.0",
    "polars>=0.20.0",
]

# 전체 설치
all = [
    "mypackage[dev,docs,perf]",
]

[project.urls]
Homepage = "https://github.com/yourusername/mypackage"
Documentation = "https://mypackage.readthedocs.io"
Repository = "https://github.com/yourusername/mypackage.git"
Issues = "https://github.com/yourusername/mypackage/issues"
Changelog = "https://github.com/yourusername/mypackage/blob/main/CHANGELOG.md"

# CLI 도구 등록
[project.scripts]
mypackage = "mypackage.cli:main"
mypackage-admin = "mypackage.cli:admin_main"

# GUI 애플리케이션 (Windows에서 콘솔 창 안 뜸)
[project.gui-scripts]
mypackage-gui = "mypackage.gui:main"

# 플러그인 진입점
[project.entry-points."myframework.plugins"]
builtin = "mypackage.plugins:BuiltinPlugin"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatchling.build.targets.wheel]
packages = ["src/mypackage"]

[tool.hatchling.build.targets.sdist]
include = [
    "/src",
    "/tests",
    "/README.md",
    "/LICENSE",
]
exclude = [
    "/.github",
    "/docs",
]
```

**의존성 버전 지정 팁**:
- `requests>=2.31.0,<3.0.0`: 안전한 범위 (권장)
- `requests~=2.31.0`: 2.31.x만 허용
- `requests>=2.31.0`: 위험 (3.0에서 깨질 수 있음)
- `requests==2.31.0`: 너무 엄격 (비권장)

---

## 동적 버전 관리

매번 버전을 여러 곳에 수동으로 업데이트하는 건 귀찮습니다. 자동화하세요.

### 방법 1: hatchling-vcs (Git 태그 기반)

```toml
[build-system]
requires = ["hatchling", "hatchling-vcs"]
build-backend = "hatchling.build"

[tool.hatchling.version]
source = "vcs"

[tool.hatchling.build.hooks.vcs]
version-file = "src/mypackage/_version.py"
```

```bash
# Git 태그로 버전 자동 결정
$ git tag v0.1.0
$ uv run python -m build
# → 0.1.0 빌드

$ git commit -m "Add feature"
$ uv run python -m build
# → 0.1.0.post1+g1234567 (개발 버전)
```

### 방법 2: __init__.py에서 읽기

```toml
[tool.hatchling.version]
path = "src/mypackage/__init__.py"
```

```python
# src/mypackage/__init__.py
__version__ = "0.1.0"
```

### 방법 3: bump2version으로 자동 업데이트

```ini
# .bumpversion.cfg
[bumpversion]
current_version = 0.1.0
commit = True
tag = True

[bumpversion:file:pyproject.toml]
search = version = "{current_version}"
replace = version = "{new_version}"

[bumpversion:file:src/mypackage/__init__.py]
search = __version__ = "{current_version}"
replace = __version__ = "{new_version}"
```

```bash
$ uv run bump2version patch  # 0.1.0 → 0.1.1
$ uv run bump2version minor  # 0.1.1 → 0.2.0
$ uv run bump2version major  # 0.2.0 → 1.0.0
```

**추천**: Git 태그 기반 방식이 가장 간단하고 실수가 적습니다.

---

## 패키지 코드 작성

### 기본 구조

```python
# src/mypackage/__init__.py
"""MyPackage - A utility library for data processing.

This package provides efficient tools for:
- Data validation
- Format conversion
- File I/O operations
"""

__version__ = "0.1.0"
__author__ = "Your Name"
__all__ = ["process_data", "validate", "DataProcessor"]

from .core import process_data, validate
from .processor import DataProcessor
```

### CLI 도구 (Click 사용)

```python
# src/mypackage/cli.py
import click
from mypackage import __version__

@click.group()
@click.version_option(version=__version__)
def main():
    """MyPackage CLI tool."""
    pass

@main.command()
@click.argument('input_file')
@click.option('--output', '-o', help='Output file path')
@click.option('--format', type=click.Choice(['json', 'csv']), default='json')
def process(input_file, output, format):
    """Process input file and save result."""
    click.echo(f"Processing {input_file}...")
    # 실제 처리 로직
    click.secho("✓ Done!", fg='green')

@main.command()
def init():
    """Initialize configuration."""
    click.echo("Creating config file...")

if __name__ == '__main__':
    main()
```

사용:
```bash
$ mypackage process data.csv -o result.json --format json
$ mypackage init
```

### 플러그인 시스템

```python
# src/mypackage/plugin.py
from abc import ABC, abstractmethod
from typing import Any

class Plugin(ABC):
    """Plugin interface."""

    @property
    @abstractmethod
    def name(self) -> str:
        """Plugin name."""
        pass

    @abstractmethod
    def process(self, data: Any) -> Any:
        """Process data."""
        pass

# src/mypackage/discover.py
from importlib.metadata import entry_points

def discover_plugins() -> dict[str, Plugin]:
    """Discover and load plugins."""
    plugins = {}

    for ep in entry_points(group='mypackage.plugins'):
        try:
            plugin_class = ep.load()
            plugin = plugin_class()
            plugins[plugin.name] = plugin
        except Exception as e:
            print(f"Failed to load plugin {ep.name}: {e}")

    return plugins
```

플러그인 제작자는:
```toml
# their-plugin/pyproject.toml
[project.entry-points."mypackage.plugins"]
custom = "their_plugin:CustomPlugin"
```

---

## 타입 힌트 및 Docstring

### 타입 힌트 (필수)

```python
from typing import TypeVar, Generic, Protocol
from pathlib import Path

T = TypeVar('T')

class Processor(Protocol):
    def process(self, data: str) -> str: ...

def process_file(
    input_path: Path,
    output_path: Path | None = None,
    encoding: str = "utf-8"
) -> dict[str, int]:
    """Process a file and return statistics.

    Args:
        input_path: Path to input file
        output_path: Optional output path. If None, returns result only
        encoding: File encoding (default: utf-8)

    Returns:
        Dictionary with processing statistics:
        - lines: Number of lines processed
        - bytes: Total bytes read
        - errors: Number of errors encountered

    Raises:
        FileNotFoundError: If input_path doesn't exist
        ValueError: If encoding is invalid

    Example:
        >>> stats = process_file(Path("data.txt"))
        >>> print(f"Processed {stats['lines']} lines")
    """
    # 구현...
```

### py.typed 마커

타입 체커가 패키지의 타입 힌트를 인식하도록:

```python
# src/mypackage/py.typed
# (빈 파일)
```

```toml
# pyproject.toml
[tool.hatchling.build.targets.wheel]
packages = ["src/mypackage"]
include = ["src/mypackage/py.typed"]
```

이제 사용자가 mypy로 체크할 때 타입 정보가 활용됩니다.

---

## 테스트 작성

### pytest 기본

```python
# tests/test_core.py
import pytest
from mypackage import process_data, validate

def test_process_data():
    """Test basic data processing."""
    result = process_data({"key": "value"})
    assert result["key"] == "VALUE"

def test_validate_success():
    """Test validation with valid data."""
    assert validate({"name": "test", "age": 25}) is True

def test_validate_failure():
    """Test validation with invalid data."""
    with pytest.raises(ValueError, match="Invalid age"):
        validate({"name": "test", "age": -1})

@pytest.fixture
def sample_data():
    """Provide sample data for tests."""
    return {"name": "test", "age": 25}

def test_with_fixture(sample_data):
    """Test using fixture."""
    assert validate(sample_data) is True
```

### 커버리지 측정

```bash
$ uv run pytest --cov=mypackage --cov-report=html

# 결과 확인
$ open htmlcov/index.html
```

### tox로 여러 Python 버전 테스트

```toml
# pyproject.toml
[tool.tox]
legacy_tox_ini = """
[tox]
env_list = py310,py311,py312

[testenv]
deps = pytest
       pytest-cov
commands = pytest {posargs}
"""
```

```bash
$ uv run tox  # 모든 버전에서 테스트
```

---

## 문서 작성

### README.md (필수)

```markdown
# MyPackage

[![PyPI version](https://badge.fury.io/py/mypackage.svg)](https://pypi.org/project/mypackage/)
[![Tests](https://github.com/user/mypackage/actions/workflows/test.yml/badge.svg)](https://github.com/user/mypackage/actions)
[![Coverage](https://codecov.io/gh/user/mypackage/branch/main/graph/badge.svg)](https://codecov.io/gh/user/mypackage)

Brief one-liner description.

## Features

- Fast data processing
- Type-safe API
- Extensible plugin system
- Comprehensive documentation

## Installation

\`\`\`bash
pip install mypackage

# With optional dependencies
pip install mypackage[perf]
\`\`\`

## Quick Start

\`\`\`python
from mypackage import process_data

result = process_data({"key": "value"})
print(result)
\`\`\`

## Documentation

Full documentation: https://mypackage.readthedocs.io

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)

## License

MIT License - see [LICENSE](LICENSE)
```

### MkDocs 문서 (선택적이지만 권장)

```yaml
# mkdocs.yml
site_name: MyPackage Documentation
theme:
  name: material
  palette:
    primary: indigo

nav:
  - Home: index.md
  - Getting Started: getting-started.md
  - API Reference: api.md
  - Examples: examples.md

plugins:
  - search
  - mkdocstrings:
      handlers:
        python:
          options:
            show_source: true
```

```bash
$ uv add --dev mkdocs mkdocs-material mkdocstrings[python]
$ uv run mkdocs serve  # http://localhost:8000
$ uv run mkdocs build  # site/ 디렉토리에 생성
```

### Sphinx (대안)

```bash
$ uv add --dev sphinx sphinx-rtd-theme
$ cd docs && uv run sphinx-quickstart
$ uv run make html
```

Read the Docs 연동:
- https://readthedocs.org 에 프로젝트 등록
- GitHub 연동하면 자동 빌드/배포

---

## 빌드 및 배포

### 로컬 빌드

```bash
# 개발 의존성 설치
$ uv add --dev build twine

# 이전 빌드 삭제
$ rm -rf dist/

# 빌드 (wheel + source distribution)
$ uv run python -m build

# 결과 확인
$ ls -lh dist/
-rw-r--r-- mypackage-0.1.0-py3-none-any.whl  # Wheel (빠른 설치)
-rw-r--r-- mypackage-0.1.0.tar.gz            # Source dist (소스 포함)
```

**Wheel vs Source Distribution**:
- **Wheel**: 미리 빌드된 바이너리, 설치 빠름, 권장
- **Source Dist**: 소스 코드 포함, 모든 플랫폼 지원, 빌드 필요

### 빌드물 검증

```bash
# twine으로 검증
$ uv run twine check dist/*
Checking dist/mypackage-0.1.0-py3-none-any.whl: PASSED
Checking dist/mypackage-0.1.0.tar.gz: PASSED

# 로컬 설치 테스트
$ uv pip install dist/mypackage-0.1.0-py3-none-any.whl

# Python에서 import 테스트
$ python -c "import mypackage; print(mypackage.__version__)"
0.1.0

# CLI 도구 테스트 (entry point 확인)
$ mypackage --version
mypackage, version 0.1.0
```

### TestPyPI 업로드 (테스트 환경)

먼저 실제 PyPI 말고 테스트 환경에서 업로드 연습하세요.

```bash
# 1. TestPyPI 계정 생성
# https://test.pypi.org/account/register/

# 2. API 토큰 생성
# https://test.pypi.org/manage/account/#api-tokens

# 3. ~/.pypirc 파일 생성
$ cat > ~/.pypirc << EOF
[distutils]
index-servers =
    pypi
    testpypi

[testpypi]
repository = https://test.pypi.org/legacy/
username = __token__
password = pypi-AgE...your-test-token...
EOF

$ chmod 600 ~/.pypirc  # 권한 제한 (중요!)

# 4. 업로드
$ uv run twine upload --repository testpypi dist/*

Uploading distributions to https://test.pypi.org/legacy/
Uploading mypackage-0.1.0-py3-none-any.whl
Uploading mypackage-0.1.0.tar.gz
...
View at: https://test.pypi.org/project/mypackage/0.1.0/

# 5. 설치 테스트
$ pip install --index-url https://test.pypi.org/simple/ mypackage

# 6. 동작 확인
$ python -c "import mypackage; print(mypackage.__version__)"
```

**주의사항**:
- TestPyPI는 주기적으로 삭제됨 (영구 저장 X)
- 의존성이 TestPyPI에 없으면 설치 실패할 수 있음
- 실제 배포 전 반드시 여기서 먼저 테스트

### PyPI 업로드 (프로덕션)

```bash
# 1. PyPI 계정 생성
# https://pypi.org/account/register/

# 2. API 토큰 생성
# https://pypi.org/manage/account/#api-tokens

# 3. ~/.pypirc 업데이트
[pypi]
username = __token__
password = pypi-AgE...your-production-token...

# 4. 최종 확인
$ uv run pytest  # 모든 테스트 통과 확인
$ uv run twine check dist/*  # 빌드물 검증

# 5. 업로드
$ uv run twine upload dist/*

Uploading distributions to https://upload.pypi.org/legacy/
...
View at: https://pypi.org/project/mypackage/0.1.0/

# 6. 전 세계에서 설치 가능!
$ pip install mypackage
```

**중요**: PyPI에 올린 버전은 삭제 불가능합니다. 같은 버전 번호로 재업로드도 안 됩니다. 신중하게!

---

## 버전 업데이트 워크플로우

### 수동 방식

```bash
# 1. 코드 수정 및 테스트
$ uv run pytest
$ uv run ruff check src/

# 2. CHANGELOG.md 업데이트
$ vim CHANGELOG.md
## [0.1.1] - 2026-01-25
### Fixed
- Fixed bug in data validation

# 3. 버전 번호 업데이트
$ vim pyproject.toml  # version = "0.1.1"
$ vim src/mypackage/__init__.py  # __version__ = "0.1.1"

# 4. Git 커밋 및 태그
$ git add .
$ git commit -m "Bump version to 0.1.1"
$ git tag v0.1.1
$ git push && git push --tags

# 5. 빌드 및 업로드
$ rm -rf dist/
$ uv run python -m build
$ uv run twine check dist/*
$ uv run twine upload dist/*
```

### 자동화 방식 (bump2version)

```bash
# 1. .bumpversion.cfg 설정 (위에서 작성)

# 2. 버전 업
$ uv run bump2version patch  # 0.1.0 → 0.1.1 (버그 수정)
$ uv run bump2version minor  # 0.1.0 → 0.2.0 (새 기능)
$ uv run bump2version major  # 0.1.0 → 1.0.0 (breaking changes)

# 자동으로:
# - pyproject.toml 업데이트
# - __init__.py 업데이트
# - Git 커밋 생성
# - Git 태그 생성

# 3. 푸시
$ git push && git push --tags

# 4. 빌드 및 업로드
$ rm -rf dist/ && uv run python -m build
$ uv run twine upload dist/*
```

---

## GitHub Actions 자동화

### 기본 CI/CD

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.10', '3.11', '3.12']

    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        run: curl -LsSf https://astral.sh/uv/install.sh | sh

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: uv sync --all-extras

      - name: Run tests
        run: uv run pytest --cov=mypackage --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
```

### PyPI 자동 배포

```yaml
# .github/workflows/publish.yml
name: Publish to PyPI

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # OIDC for trusted publishing

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
        uses: pypa/gh-action-pypi-publish@release/v1
        with:
          # Trusted publishing (API 토큰 불필요!)
          # PyPI에서 설정: https://pypi.org/manage/account/publishing/
```

**Trusted Publishing 설정**:
1. PyPI → Account settings → Publishing
2. Add a new publisher
3. GitHub 레포지토리, workflow 파일 등록
4. API 토큰 없이 안전하게 배포 가능

사용법:
```bash
# GitHub에서 Release 만들기
$ git tag v0.1.1
$ git push --tags

# GitHub UI에서 Release 생성
# → 자동으로 테스트 → 빌드 → PyPI 배포
```

---

## 고급 기능

### C 확장 포함 (선택적)

```toml
[build-system]
requires = ["setuptools>=61", "wheel", "Cython"]
build-backend = "setuptools.build_meta"

[tool.setuptools.ext-modules]
fast_module = {sources = ["src/mypackage/_fast.pyx"]}
```

빌드 시 Cython 코드가 C로 컴파일되어 성능 향상.

### Namespace 패키지

여러 패키지가 같은 네임스페이스를 공유:

```
mycompany-core/
└── src/
    └── mycompany/
        └── core/

mycompany-plugins/
└── src/
    └── mycompany/
        └── plugins/

# 사용자는:
from mycompany.core import something
from mycompany.plugins import another
```

```toml
[tool.setuptools.packages.find]
where = ["src"]
namespaces = true
```

### 플랫폼별 의존성

```toml
[project]
dependencies = [
    "requests>=2.31.0",
    "pywin32>=306; platform_system=='Windows'",
    "uvloop>=0.19.0; platform_system!='Windows'",
]
```

### 데이터 파일 포함

```toml
[tool.hatchling.build.targets.wheel]
packages = ["src/mypackage"]

[tool.hatchling.build.targets.wheel.shared-data]
"data/config.json" = "share/mypackage/config.json"
```

```python
# 런타임에 접근
from importlib.resources import files

config_path = files('mypackage').joinpath('data/config.json')
```

---

## 보안 및 베스트 프랙티스

### 보안 체크리스트

- [ ] `.pypirc` 파일 권한 600으로 제한
- [ ] API 토큰을 Git에 커밋하지 않기
- [ ] Trusted Publishing 사용 (API 토큰 노출 위험 제거)
- [ ] 의존성 버전 범위 제한 (`>=x,<y`)
- [ ] `pip-audit`로 취약점 스캔
- [ ] SBOM (Software Bill of Materials) 생성

```bash
# 취약점 스캔
$ uv run pip-audit

# SBOM 생성
$ uv run cyclonedx-py -i pyproject.toml -o sbom.json
```

### Deprecated 기능 경고

```python
import warnings

def old_function():
    warnings.warn(
        "old_function is deprecated, use new_function instead",
        DeprecationWarning,
        stacklevel=2
    )
    return new_function()
```

### Semantic Versioning (필수)

**MAJOR.MINOR.PATCH** (예: 1.2.3)

- **MAJOR**: Breaking changes (API 변경)
- **MINOR**: 새 기능 추가 (하위 호환)
- **PATCH**: 버그 수정

예시:
- `1.0.0` → `1.0.1`: 버그 수정
- `1.0.1` → `1.1.0`: 새 함수 추가
- `1.1.0` → `2.0.0`: 함수 시그니처 변경 (breaking)

**0.x 버전**:
- 0.x는 불안정 버전 (언제든 API 변경 가능)
- 안정화되면 1.0.0 출시

---

## 패키지 유지보수

### CHANGELOG.md (필수)

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [Unreleased]

### Added
- New feature X

### Changed
- Improved performance of Y

### Deprecated
- Function Z will be removed in 2.0.0

### Fixed
- Bug in edge case

## [0.2.0] - 2026-01-20

### Added
- Plugin system
- CLI tool

### Changed
- Refactored core module

## [0.1.0] - 2026-01-01

### Added
- Initial release
- Basic data processing
```

### 사용자 피드백 처리

**Issue 템플릿**:
```markdown
<!-- .github/ISSUE_TEMPLATE/bug_report.md -->
## Bug Description

**Expected behavior:**

**Actual behavior:**

**Minimal reproduction:**
\`\`\`python
# Your code here
\`\`\`

**Environment:**
- OS:
- Python version:
- Package version:
```

### 마이그레이션 가이드

Breaking changes가 있을 때:

```markdown
# Migration Guide: v1.x to v2.0

## Breaking Changes

### Function signatures

**Old (v1.x):**
\`\`\`python
process_data(data, format)
\`\`\`

**New (v2.0):**
\`\`\`python
process_data(data, output_format: str = "json")
\`\`\`

### Removed features

- `old_function()` → Use `new_function()` instead
```

---

## 트러블슈팅

### 문제: "Module not found" after installation

**원인**: `packages` 설정이 잘못됨

**해결**:
```toml
# pyproject.toml
[tool.hatchling.build.targets.wheel]
packages = ["src/mypackage"]  # src/ 레이아웃 사용 시
```

### 문제: 의존성 버전 충돌

**원인**: 너무 엄격하거나 너무 느슨한 버전 지정

**해결**:
```toml
# 나쁜 예
dependencies = ["requests==2.31.0"]  # 너무 엄격

# 좋은 예
dependencies = ["requests>=2.31.0,<3.0.0"]  # 안전한 범위
```

### 문제: Wheel이 플랫폼별로 다름

**원인**: C 확장이나 바이너리 포함

**해결**: 여러 플랫폼에서 빌드 필요
```yaml
# GitHub Actions에서 multi-platform 빌드
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
```

### 문제: PyPI 업로드 실패 (403 Forbidden)

**원인**:
- API 토큰 만료
- 패키지 이름 중복
- 권한 부족

**해결**:
```bash
# 1. 새 토큰 생성
# 2. ~/.pypirc 업데이트
# 3. 패키지 이름 변경 (이미 존재하는 경우)
$ vim pyproject.toml  # name = "mypackage-username"
```

### 문제: 테스트가 로컬에서만 통과

**원인**: 환경 변수, 파일 경로 의존

**해결**:
```python
# 절대 경로 대신 상대 경로
from pathlib import Path

def get_data_file():
    package_dir = Path(__file__).parent
    return package_dir / "data" / "config.json"
```

---

## 실전 예시: CLI 도구 완전 패키지

전체 프로젝트 구조:

```
mycli/
├── src/
│   └── mycli/
│       ├── __init__.py
│       ├── cli.py
│       ├── commands/
│       │   ├── __init__.py
│       │   ├── init.py
│       │   └── deploy.py
│       └── utils.py
├── tests/
│   ├── conftest.py
│   ├── test_cli.py
│   └── test_commands/
│       ├── test_init.py
│       └── test_deploy.py
├── docs/
│   ├── index.md
│   ├── commands.md
│   └── api.md
├── .github/
│   └── workflows/
│       ├── test.yml
│       └── publish.yml
├── README.md
├── LICENSE
├── CHANGELOG.md
├── pyproject.toml
└── mkdocs.yml
```

```toml
# pyproject.toml
[project]
name = "mycli"
version = "1.0.0"
description = "A powerful CLI tool for X"
readme = "README.md"
requires-python = ">=3.10"
license = {text = "MIT"}
authors = [{name = "Your Name"}]

dependencies = [
    "click>=8.1.0",
    "rich>=13.0.0",  # 예쁜 터미널 출력
    "httpx>=0.26.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-cov>=4.1.0",
    "ruff>=0.1.0",
]

[project.scripts]
mycli = "mycli.cli:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

```python
# src/mycli/cli.py
import click
from rich.console import Console

console = Console()

@click.group()
@click.version_option()
def main():
    """MyCLI - Powerful command-line tool."""
    pass

@main.command()
@click.argument('name')
def init(name):
    """Initialize a new project."""
    console.print(f"[green]✓[/green] Created project: {name}")

@main.command()
@click.option('--env', type=click.Choice(['dev', 'prod']), default='dev')
def deploy(env):
    """Deploy to environment."""
    with console.status(f"Deploying to {env}..."):
        # 실제 배포 로직
        pass
    console.print("[green]✓[/green] Deployment successful!")
```

배포:
```bash
$ uv run python -m build
$ uv run twine upload dist/*

# 사용자는:
$ pip install mycli
$ mycli init my-project
$ mycli deploy --env prod
```

---

## 마무리

여기까지 하면 프로페셔널한 Python 패키지를 만들고 배포할 수 있습니다.

**핵심 요약**:
- `pyproject.toml` 하나로 모든 설정 관리
- src/ 레이아웃 사용 (테스트 안정성)
- TestPyPI에서 먼저 테스트
- Semantic versioning 준수
- CHANGELOG.md 작성 (사용자 친화적)
- GitHub Actions로 자동화 (CI/CD)
- Trusted Publishing 사용 (보안)

**처음 만들 때**:
1. 간단하게 시작 (최소 기능만)
2. TestPyPI에서 배포 연습
3. 실제 사용자 피드백 수집
4. 점진적으로 개선

**유지보수 팁**:
- 의존성 버전 정기 업데이트
- Issue에 빠르게 응답
- Breaking changes는 메이저 버전에서만
- Deprecation warning 먼저, 다음 버전에서 제거

Python 패키징은 처음엔 복잡해 보이지만, pyproject.toml + uv + GitHub Actions 조합이면 웬만한 건 다 커버됩니다. 완벽하게 시작하려고 하지 말고, 작게 시작해서 필요한 만큼만 추가하세요.

**[← LLM 서비스](llm-service.md)** | **[자동화 스크립트 →](automation.md)**
