# 자동화 시스템 구축

## 목표

반복 작업을 자동화하여 생산성 향상

---

## 시나리오별 도구

| 시나리오 | 도구 | 사용 예시 |
|----------|------|----------|
| 스케줄링 | schedule | 매일 아침 리포트 생성 |
| CLI 도구 | click / typer | 명령줄 인터페이스 |
| 웹 스크래핑 | beautifulsoup4 / scrapy | 데이터 수집 |
| 브라우저 자동화 | playwright | 웹 테스트, 크롤링 |
| 파일 감시 | watchdog | 파일 변경 시 자동 처리 |
| 시스템 관리 | fabric | 원격 서버 작업 |

---

## 예시 1: 스케줄링

```python
import schedule
import time

def job():
    print("작업 실행!")

# 매일 오전 9시
schedule.every().day.at("09:00").do(job)

# 1시간마다
schedule.every().hour.do(job)

while True:
    schedule.run_pending()
    time.sleep(60)
```

---

## 예시 2: CLI 도구 (Typer)

```python
import typer
from pathlib import Path

app = typer.Typer()

@app.command()
def process(
    input_file: Path,
    output_file: Path,
    verbose: bool = False
):
    """파일 처리"""
    if verbose:
        typer.echo(f"Processing {input_file}...")

    # 처리 로직
    content = input_file.read_text()
    result = content.upper()
    output_file.write_text(result)

    typer.echo(f"Done! Output: {output_file}")

if __name__ == "__main__":
    app()
```

---

## 예시 3: 웹 스크래핑

```python
import httpx
from bs4 import BeautifulSoup

async def scrape_page(url: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(url)

    soup = BeautifulSoup(response.text, 'html.parser')

    # 데이터 추출
    titles = [h2.text for h2 in soup.find_all('h2')]

    return titles
```

---

**[← 데이터 파이프라인](data-pipeline.md)** | **[라이브러리 카탈로그 →](../04-library-catalog/README.md)**
