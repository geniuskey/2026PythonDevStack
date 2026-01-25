# 파이썬 라이브러리 완벽 가이드 2026

> **2026년 기준으로 다시 정리한 파이썬 생태계의 완전한 지도**

## 📖 이 책에 대하여

이 책은 파이썬 생태계의 방대한 라이브러리들을 체계적으로 정리한 실용 가이드입니다. 단순한 라이브러리 나열이 아닌, **2026년 현재 실무에서 무엇을 선택해야 하는지** 명확한 방향을 제시합니다.

### 이 책의 특징

- - **압도적인 커버리지**: 웹, 데이터, AI, 자동화 등 모든 분야 포괄
- - **명확한 우선순위**: 2026년 기준 추천 스택 제시
- - **문제 중심 접근**: 실무 시나리오별 라이브러리 조합 가이드
- - **등급 시스템**: 각 라이브러리의 현재 상태를 명확히 표시

---

## 📚 목차

### PART 1: 시작하기

- [프롤로그](01-introduction/prologue.md) - 2026년의 파이썬 생태계
- [이 책의 사용법](01-introduction/how-to-use.md)
- [등급 시스템 안내](01-introduction/rating-system.md)

### PART 2: 2026 현대 파이썬 표준 스택

- [개요: 2026년 파이썬 개발의 중심축](02-modern-stack-2026/overview.md)
- [프로젝트 관리 & 패키징](02-modern-stack-2026/project-packaging.md)
- [코드 품질 & 린팅](02-modern-stack-2026/code-quality.md)
- [테스트 & 품질 보증](02-modern-stack-2026/testing.md)
- [웹 API 개발](02-modern-stack-2026/web-api.md)
- [비동기 I/O & 네트워킹](02-modern-stack-2026/async-io.md)
- [관측성 & 모니터링](02-modern-stack-2026/observability.md)
- [현대적 데이터 처리](02-modern-stack-2026/data-processing.md)
- [LLM & AI 애플리케이션](02-modern-stack-2026/llm-apps.md)

### PART 3: 문제 해결 가이드

실무에서 자주 마주하는 과제별 라이브러리 조합과 구현 가이드

- [API 서버 만들기](03-problem-solving-guides/api-server.md)
- [패키지 배포하기](03-problem-solving-guides/package-deployment.md)
- [LLM 서비스 구축하기](03-problem-solving-guides/llm-service.md)
- [데이터 파이프라인 구축](03-problem-solving-guides/data-pipeline.md)
- [자동화 시스템 구축](03-problem-solving-guides/automation.md)

### PART 4: 라이브러리 카탈로그

분야별 라이브러리 상세 레퍼런스

- [웹 개발](04-library-catalog/web-development/README.md)
- [데이터 과학](04-library-catalog/data-science/README.md)
- [머신러닝 & AI](04-library-catalog/machine-learning/README.md)
- [시스템 & 자동화](04-library-catalog/system-automation/README.md)
- [문서 처리](04-library-catalog/document-processing/README.md)
- [멀티미디어](04-library-catalog/multimedia/README.md)
- [네트워킹](04-library-catalog/networking/README.md)
- [보안 & 암호화](04-library-catalog/security/README.md)
- [유틸리티](04-library-catalog/utilities/README.md)

---

## 🏷️ 등급 시스템

이 책의 모든 라이브러리는 다음 등급으로 분류됩니다:

| 등급 | 의미 | 설명 |
|------|------|------|
| - 2026 권장: | **2026 추천** | 지금 새 프로젝트를 시작한다면 이것을 선택하세요 |
| - 레거시: | **레거시/유지보수용** | 기존 프로젝트에서는 유효하나, 신규 채택은 재고 필요 |
| - 대체: | **대체재 존재** | 더 나은 대안이 있으니 해당 섹션 참고 |
| - 니치: | **특정 상황 특화** | 특수한 상황이나 니치한 용도에 최적화됨 |

---

## 🎯 이 책의 사용법

### 초보자라면
1. [프롤로그](01-introduction/prologue.md)부터 읽기
2. [2026 표준 스택](02-modern-stack-2026/overview.md)으로 전체 그림 파악
3. [문제 해결 가이드](03-problem-solving-guides/)에서 관심 주제 선택

### 경험자라면
1. [2026 표준 스택](02-modern-stack-2026/overview.md)에서 최신 트렌드 확인
2. [라이브러리 카탈로그](04-library-catalog/)에서 필요한 도구 탐색
3. - 2026 권장: 마크가 있는 항목 위주로 업데이트 검토

### 특정 문제를 해결하고 싶다면
- [문제 해결 가이드](03-problem-solving-guides/)에서 시나리오별 가이드 참고

---

## 💡 왜 2026년 개정판인가?

파이썬 생태계는 빠르게 진화합니다. 3년 전만 해도:

- `pipenv`가 표준이었지만, 지금은 `uv`가 대세
- `flake8`로 린팅했지만, 지금은 `ruff`가 50배 빠름
- `requests`가 기본이었지만, 지금은 `httpx`로 async 지원
- API는 `Flask`였지만, 지금은 `FastAPI`가 표준
- LLM은 실험이었지만, 지금은 프로덕션 워크로드

이 책은 **파이썬으로 할 수 있는 모든 것의 지도**이면서 동시에,
**2026년 지금, 무엇을 선택해야 하는지 알려주는 나침반**입니다.

---

## 📅 업데이트 정보

- **2026-01**: 초판 발행
- **기준 버전**: Python 3.12+
- **검증 환경**: Linux, macOS, Windows

---

## 🤝 기여 및 피드백

이 책은 지속적으로 업데이트됩니다. 오류 발견, 새로운 라이브러리 제안, 개선 의견은 언제든 환영합니다.

---

## 📜 라이선스

이 책의 모든 내용은 교육 및 참고 목적으로 자유롭게 사용할 수 있습니다.

---

**Let's build something amazing with Python in 2026! 🚀**
