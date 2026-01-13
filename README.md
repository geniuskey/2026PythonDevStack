# 파이썬 라이브러리 완벽 가이드 2026

> **2026년 기준으로 다시 정리한 파이썬 생태계의 완전한 지도**

[![Deploy Docs](https://github.com/geniuskey/2026PythonDevStack/actions/workflows/deploy-docs.yml/badge.svg)](https://github.com/geniuskey/2026PythonDevStack/actions/workflows/deploy-docs.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## 📚 온라인 문서

**👉 [https://geniuskey.github.io/2026PythonDevStack/](https://geniuskey.github.io/2026PythonDevStack/)**

## 🎯 이 책에 대하여

파이썬 생태계의 방대한 라이브러리들을 체계적으로 정리한 실용 가이드입니다. 단순한 라이브러리 나열이 아닌, **2026년 현재 실무에서 무엇을 선택해야 하는지** 명확한 방향을 제시합니다.

### 특징

- ✅ **압도적인 커버리지**: 웹, 데이터, AI, 자동화 등 모든 분야 포괄
- ✅ **명확한 우선순위**: 2026년 기준 추천 스택 제시
- ✅ **문제 중심 접근**: 실무 시나리오별 라이브러리 조합 가이드
- ✅ **등급 시스템**: 각 라이브러리의 현재 상태를 명확히 표시 (⭐⚠️🔄🧩)

## 📖 구성

### PART 1: 시작하기
- 프롤로그 - 2026년의 파이썬 생태계
- 이 책의 사용법
- 등급 시스템 안내

### PART 2: 2026 현대 파이썬 표준 스택
- 프로젝트 관리 & 패키징 (uv, pyproject.toml)
- 코드 품질 & 린팅 (ruff, mypy)
- 테스트 & 품질 보증 (pytest)
- 웹 API 개발 (FastAPI, pydantic)
- 비동기 I/O & 네트워킹 (httpx, asyncio)
- 관측성 & 모니터링 (structlog, OpenTelemetry)
- 현대적 데이터 처리 (polars, duckdb)
- LLM & AI 애플리케이션 (langchain, vector DB)

### PART 3: 문제 해결 가이드
- API 서버 만들기
- 패키지 배포하기
- LLM 서비스 구축하기
- 데이터 파이프라인 구축
- 자동화 시스템 구축

### PART 4: 라이브러리 카탈로그
- 웹 개발
- 데이터 과학
- 머신러닝 & AI
- 시스템 & 자동화
- 문서 처리
- 멀티미디어
- 네트워킹
- 보안 & 암호화
- 유틸리티

## 🏷️ 등급 시스템

| 등급 | 의미 |
|------|------|
| ⭐ | **2026 추천** - 지금 새 프로젝트를 시작한다면 이것을 선택하세요 |
| ⚠️ | **레거시/유지보수용** - 기존 프로젝트에서는 유효하나, 신규 채택은 재고 필요 |
| 🔄 | **대체재 존재** - 더 나은 대안이 있으니 해당 섹션 참고 |
| 🧩 | **특정 상황 특화** - 특수한 상황이나 니치한 용도에 최적화됨 |

## 🚀 로컬에서 문서 보기

### 요구사항
- Python 3.10+

### 설치 및 실행

```bash
# 저장소 클론
git clone https://github.com/geniuskey/2026PythonDevStack.git
cd 2026PythonDevStack

# 의존성 설치
pip install -r requirements-docs.txt

# 로컬 서버 실행
mkdocs serve

# 브라우저에서 http://127.0.0.1:8000 접속
```

### 빌드

```bash
# 정적 사이트 생성 (site/ 폴더에 생성됨)
mkdocs build
```

## 🤝 기여하기

이 책은 지속적으로 업데이트됩니다. 기여는 환영합니다!

1. 이 저장소를 Fork
2. 새 브랜치 생성 (`git checkout -b feature/amazing-feature`)
3. 변경사항 커밋 (`git commit -m 'Add some amazing feature'`)
4. 브랜치에 푸시 (`git push origin feature/amazing-feature`)
5. Pull Request 생성

### 기여 가이드라인
- 새로운 라이브러리 추가 시 등급과 이유를 명확히 기재
- 코드 예제는 Python 3.10+ 기준
- 마크다운 문법 준수
- 한글 맞춤법 확인

## 📜 라이선스

MIT License - 자유롭게 사용, 수정, 배포 가능합니다.

## 💡 피드백

- **버그 리포트**: [GitHub Issues](https://github.com/geniuskey/2026PythonDevStack/issues)
- **기능 제안**: [GitHub Discussions](https://github.com/geniuskey/2026PythonDevStack/discussions)
- **문의사항**: Issue를 통해 질문해주세요

## 🌟 스타 히스토리

이 프로젝트가 도움이 되었다면 ⭐️ 를 눌러주세요!

---

**Let's build something amazing with Python in 2026! 🚀**
