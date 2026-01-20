# Ruff 완벽 가이드

> **올인원 파이썬 린터 & 포매터**

⭐ **2026 추천** | 🔍 린팅 | ✨ 포매팅 | ⚡ 50배 빠름

---

## 개요

| 항목 | 내용 |
|------|------|
| **공식 사이트** | https://docs.astral.sh/ruff/ |
| **GitHub** | https://github.com/astral-sh/ruff |
| **첫 릴리즈** | 2022년 8월 |
| **창시자** | Charlie Marsh (Astral) |
| **라이선스** | MIT |

### 한 줄 요약

**flake8 + black + isort + pylint를 하나로 통합, 50배 빠른 Rust 기반 린터**

---

## 왜 Ruff인가

### 속도 비교

```
대형 코드베이스 린팅 시간:
flake8:     15초
pylint:     45초
ruff:       0.3초  ← 50배 빠름!
```

### 대체하는 도구들

**Ruff 하나로 대체:**
- flake8 (린팅)
- black (포매팅)
- isort (import 정렬)
- pyupgrade (문법 현대화)
- pydocstyle (docstring)
- 그 외 20+ 도구

---

## 설치

```bash
$ uv add --dev ruff

# 또는
$ pip install ruff
```

---

## 기본 사용법

### 린팅

```bash
# 모든 파일 검사
$ ruff check .

# 자동 수정
$ ruff check --fix .

# 특정 파일만
$ ruff check src/main.py

# 무시할 규칙 지정
$ ruff check --ignore E501 .
```

### 포매팅

```bash
# 모든 파일 포맷
$ ruff format .

# 검사만 (변경 안 함)
$ ruff format --check .

# 린팅 + 포매팅 한번에
$ ruff check --fix . && ruff format .
```

---

## 설정

### pyproject.toml

```toml
[tool.ruff]
line-length = 88
target-version = "py312"
extend-exclude = ["migrations", "venv"]

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "N",   # pep8-naming
    "UP",  # pyupgrade
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
]
ignore = [
    "E501",  # line too long (formatter handles)
]

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]  # allow assert in tests

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

---

## 실전 패턴

### 패턴 1: pre-commit 통합

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.14
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

### 패턴 2: VS Code 통합

```json
// .vscode/settings.json
{
  "[python]": {
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.organizeImports": true
    },
    "editor.defaultFormatter": "charliermarsh.ruff"
  }
}
```

### 패턴 3: CI/CD

```yaml
# .github/workflows/lint.yml
- name: Run Ruff
  run: |
    ruff check .
    ruff format --check .
```

### 패턴 4: 점진적 도입

```bash
# 기존 코드는 무시, 새 코드만 검사
$ ruff check --diff

# 특정 규칙만 활성화
$ ruff check --select E,F .

# 점차 규칙 추가
$ ruff check --select E,F,I .
```

---

## flake8/black에서 마이그레이션

### 설정 변환

```toml
# 기존 setup.cfg
[flake8]
max-line-length = 88
ignore = E203,W503

[isort]
profile = black

# Ruff로 (pyproject.toml)
[tool.ruff]
line-length = 88

[tool.ruff.lint]
ignore = ["E203", "W503"]

[tool.ruff.lint.isort]
# isort 호환 설정
```

### 명령어 대체

| 기존 | Ruff |
|------|------|
| `flake8 .` | `ruff check .` |
| `black .` | `ruff format .` |
| `isort .` | `ruff check --select I --fix .` |
| `모두 실행` | `ruff check --fix . && ruff format .` |

---

## 규칙 선택 가이드

### 추천 세트

**최소 (엄격):**
```toml
select = ["E", "F"]  # 문법 에러만
```

**표준:**
```toml
select = ["E", "F", "I", "N", "UP"]
```

**엄격:**
```toml
select = ["E", "F", "I", "N", "UP", "B", "C4", "SIM", "TCH"]
```

### 주요 규칙

- `E`, `W`: PEP 8 스타일
- `F`: 문법 에러 (undefined name 등)
- `I`: import 정렬
- `N`: 네이밍 컨벤션
- `UP`: 현대 파이썬 문법 사용
- `B`: 버그 가능성 있는 코드
- `SIM`: 코드 단순화
- `TCH`: 타입 체킹 최적화

---

## 베스트 프랙티스

### 1. 모든 프로젝트에 필수

```toml
# 최소한의 설정
[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I"]
```

### 2. 포매팅과 린팅 분리

```bash
# 먼저 포맷 (코드 변경)
$ ruff format .

# 그 다음 린트 (수정 가능한 것만)
$ ruff check --fix .

# 남은 문제 확인
$ ruff check .
```

### 3. 점진적 규칙 추가

```bash
# 1주차: 기본만
select = ["E", "F"]

# 2주차: import 정렬 추가
select = ["E", "F", "I"]

# 3주차: 네이밍
select = ["E", "F", "I", "N"]

# ...
```

---

**[← 코드 품질](../../04-library-catalog/code-quality/README.md)** | **[다음: mypy →](mypy.md)**
