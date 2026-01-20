# uv 완벽 가이드

> **차세대 파이썬 패키지 관리자**

⭐ **2026 추천** | 📦 패키지 관리 | ⚡ 초고속 | 🦀 Rust 기반

---

## 개요

| 항목 | 내용 |
|------|------|
| **공식 사이트** | https://github.com/astral-sh/uv |
| **PyPI** | https://pypi.org/project/uv/ |
| **첫 릴리즈** | 2024년 2월 |
| **창시자** | Astral (ruff 개발팀) |
| **라이선스** | Apache 2.0 / MIT |

### 한 줄 요약

**pip, pipenv, poetry를 10-100배 빠르게 대체하는 Rust 기반 패키지 관리자**

---

## 왜 uv인가

### 속도 비교

```
패키지 설치 시간 (Django + 의존성):
pip:     45초
pipenv:  90초
poetry:  60초
uv:      0.5초  ← 100배 빠름!
```

### 주요 특징

- ✅ **초고속**: Rust로 작성, 병렬 처리
- ✅ **통합 도구**: 가상환경 + 패키지 관리 + 프로젝트 관리
- ✅ **pip 호환**: `pip install` → `uv pip install`
- ✅ **pyproject.toml 완벽 지원**
- ✅ **캐싱**: 글로벌 캐시로 중복 다운로드 제거

---

## 설치

```bash
# macOS / Linux
$ curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
$ powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# 또는 pip로
$ pip install uv

# 검증
$ uv --version
```

---

## 기본 사용법

### 프로젝트 생성

```bash
# 새 프로젝트
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

### 패키지 설치

```bash
# 패키지 추가
$ uv add fastapi httpx

# 개발 의존성
$ uv add --dev pytest ruff

# 버전 지정
$ uv add "fastapi>=0.109.0,<1.0.0"

# 그룹별 의존성
$ uv add --group docs sphinx
```

### 가상환경 관리

```bash
# 자동으로 .venv 생성 및 동기화
$ uv sync

# 가상환경 없이 실행
$ uv run python main.py
$ uv run pytest

# 명시적 활성화
$ source .venv/bin/activate  # Unix
$ .venv\Scripts\activate     # Windows
```

---

## 실전 패턴

### 패턴 1: 기존 프로젝트 마이그레이션

```bash
# requirements.txt에서
$ uv pip install -r requirements.txt
$ uv init  # pyproject.toml 생성

# poetry에서
$ uv add $(poetry export -f requirements.txt | cut -d'=' -f1)
```

### 패턴 2: 잠금 파일

```bash
# uv.lock 생성 (의존성 버전 고정)
$ uv lock

# 잠긴 버전으로 설치
$ uv sync --locked

# CI/CD에서 사용
$ uv sync --frozen  # lock 파일 업데이트 금지
```

### 패턴 3: pip 호환 모드

```bash
# 기존 pip 명령어 그대로 사용
$ uv pip install requests
$ uv pip list
$ uv pip freeze
$ uv pip uninstall requests

# requirements.txt 생성
$ uv pip freeze > requirements.txt
```

### 패턴 4: 빠른 도커 빌드

```dockerfile
FROM python:3.12-slim

# uv 설치
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app

# 의존성만 먼저 설치 (캐싱)
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# 소스 복사
COPY . .

CMD ["uv", "run", "python", "main.py"]
```

---

## pip vs uv 마이그레이션

| pip 명령어 | uv 명령어 |
|------------|-----------|
| `pip install package` | `uv add package` |
| `pip install -r requirements.txt` | `uv pip install -r requirements.txt` |
| `pip freeze` | `uv pip freeze` |
| `python -m venv .venv` | `uv venv` (또는 자동) |
| `pip install -e .` | `uv pip install -e .` |

---

## 베스트 프랙티스

### 1. uv.lock 커밋

```bash
# .gitignore
.venv/
__pycache__/

# uv.lock은 커밋 (재현성)
# git add uv.lock
```

### 2. CI/CD 설정

```yaml
# .github/workflows/test.yml
- name: Install uv
  run: curl -LsSf https://astral.sh/uv/install.sh | sh

- name: Install dependencies
  run: uv sync --frozen

- name: Run tests
  run: uv run pytest
```

### 3. 스크립트 정의

```toml
# pyproject.toml
[project.scripts]
myapp = "myapp.cli:main"

# 실행
$ uv run myapp
```

---

## poetry vs uv

| 특징 | uv | poetry |
|------|----|----|
| 속도 | 🚀 | 느림 |
| 파일 | pyproject.toml | pyproject.toml |
| 설치 | 바이너리 | pip |
| 성숙도 | 신생 | 성숙 |

**uv 선택 시기:** 속도가 중요, 새 프로젝트
**poetry 유지 시기:** 기존 프로젝트, 팀이 익숙함

---

**[← 패키지 관리](../../04-library-catalog/package-management/README.md)** | **[다음: Poetry →](poetry.md)**
