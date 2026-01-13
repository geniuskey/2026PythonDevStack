# 코드 품질 & 린팅

## 2026 표준 스택

| 도구 | 등급 | 역할 |
|------|------|------|
| `ruff` | ⭐ | 초고속 통합 린터/포매터 |
| `mypy` | ⭐ | 정적 타입 체커 |
| `pre-commit` | ⭐ | Git 훅 자동화 |

---

## 1. Ruff: 통합 린터 ⭐

### 핵심 특징

- **50배 빠름**: flake8보다 10-100배, black보다 50배
- **통합 도구**: flake8 + black + isort + pylint 기능 포함
- **Rust 기반**: 네이티브 속도

### 설치 및 사용

```bash
# 설치
$ uv add --dev ruff

# 린팅
$ ruff check .

# 포매팅
$ ruff format .

# 자동 수정
$ ruff check --fix .
```

### pyproject.toml 설정

```toml
[tool.ruff]
line-length = 88
target-version = "py312"
extend-exclude = ["migrations"]

[tool.ruff.lint]
select = [
    "E",  # pycodestyle errors
    "F",  # pyflakes
    "I",  # isort
    "N",  # pep8-naming
    "W",  # pycodestyle warnings
    "UP", # pyupgrade
]
ignore = ["E203", "E501"]

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
```

### VS Code 통합

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

---

## 2. mypy: 타입 체커 ⭐

### 핵심 특징

- **정적 타입 검사**: 런타임 전에 타입 에러 발견
- **점진적 도입**: 기존 코드에 서서히 적용 가능

### 설치 및 사용

```bash
$ uv add --dev mypy

# 타입 체크
$ mypy src/

# 특정 파일
$ mypy main.py
```

### 기본 타입 힌트

```python
# 함수 타입 힌트
def greet(name: str) -> str:
    return f"Hello, {name}"

# 변수 타입 힌트
age: int = 30
names: list[str] = ["Alice", "Bob"]

# Optional 타입
from typing import Optional

def find_user(user_id: int) -> Optional[str]:
    if user_id == 1:
        return "Alice"
    return None
```

### pyproject.toml 설정

```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

# 점진적 도입
[[tool.mypy.overrides]]
module = "legacy.*"
ignore_errors = true
```

---

## 3. pre-commit: 자동화 ⭐

### 핵심 특징

- **Git 커밋 전 자동 검사**
- **팀 전체 일관성** 보장

### 설치 및 사용

```bash
$ uv add --dev pre-commit

# 훅 설치
$ pre-commit install

# 수동 실행
$ pre-commit run --all-files
```

### .pre-commit-config.yaml

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.14
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
```

---

## 대체 도구들

### flake8 ⚠️

- **이유**: ruff가 50배 빠르고 더 많은 기능
- **언제 OK**: 기존 프로젝트에 복잡한 플러그인 사용 중

### black ⚠️

- **이유**: ruff format이 동일 결과를 50배 빠르게
- **언제 OK**: black과 100% 호환 필요

### pylint ⚠️

- **이유**: ruff가 대부분 규칙 커버, 훨씬 빠름
- **언제 OK**: pylint 전용 규칙 필요

---

## 실전 워크플로우

```bash
# 1. 프로젝트에 도구 추가
$ uv add --dev ruff mypy pre-commit

# 2. 설정 파일 작성
$ cat > pyproject.toml
$ cat > .pre-commit-config.yaml

# 3. pre-commit 활성화
$ pre-commit install

# 4. 기존 코드 정리
$ ruff check --fix .
$ ruff format .

# 5. 이후 커밋마다 자동 검사!
$ git commit -m "feat: add feature"
# → 자동으로 ruff, mypy 실행
```

---

**[← 프로젝트 관리](project-packaging.md)** | **[테스트 →](testing.md)**
