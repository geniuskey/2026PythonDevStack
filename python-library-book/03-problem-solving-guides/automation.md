# 자동화 시스템 구축

## 목표

반복 작업을 자동화하여 생산성을 향상시키고, 안정적이고 확장 가능한 자동화 시스템 구축하기

---

## 자동화 시스템 아키텍처

```mermaid
graph TD
    A[트리거] --> B{실행 방식}
    B -->|스케줄| C[시간 기반]
    B -->|이벤트| D[파일/웹훅]
    B -->|수동| E[CLI 명령]

    C --> F[작업 실행]
    D --> F
    E --> F

    F --> G[로깅]
    F --> H[알림]
    F --> I[결과 저장]

    style F fill:#4CAF50
    style G fill:#FF9800
    style H fill:#2196F3
```

---

## 기술 스택

| 시나리오 | 도구 | 사용 예시 | 2026 권장 |
|----------|------|----------|-----------|
| **스케줄링** | APScheduler | 매일 아침 리포트 생성 | - 2026 권장: |
| **CLI 도구** | click / typer | 명령줄 인터페이스 | - 2026 권장: |
| **웹 스크래핑** | httpx + bs4 | 데이터 수집 | - 2026 권장: |
| **브라우저 자동화** | playwright | 웹 테스트, 크롤링 | - 2026 권장: |
| **파일 감시** | watchdog | 파일 변경 시 자동 처리 | - 2026 권장: |
| **작업 큐** | dramatiq | 비동기 작업 처리 | - 2026 권장: |
| **원격 실행** | fabric | 원격 서버 작업 | - |
| **워크플로우** | Dagster | 복잡한 자동화 체인 | - 2026 권장: |

---

## 프로젝트 구조

```
automation-system/
├── pyproject.toml
├── .env
├── config/
│   └── settings.py
├── tasks/
│   ├── __init__.py
│   ├── scheduler.py      # 스케줄 작업
│   ├── scraper.py        # 웹 스크래핑
│   ├── file_watcher.py   # 파일 감시
│   └── notifications.py  # 알림
├── cli/
│   └── main.py           # CLI 진입점
├── utils/
│   ├── logger.py
│   └── helpers.py
├── data/
│   ├── input/
│   └── output/
└── logs/
```

### 의존성 설치

```bash
$ uv init automation-system
$ cd automation-system

# 핵심 라이브러리
$ uv add apscheduler click httpx beautifulsoup4

# 브라우저 자동화
$ uv add playwright
$ uv run playwright install

# 파일 감시
$ uv add watchdog

# 작업 큐
$ uv add dramatiq redis

# 알림
$ uv add slack-sdk

# 로깅
$ uv add structlog

# 개발 도구
$ uv add --dev pytest pytest-asyncio
```

---

## 스케줄링 자동화

```mermaid
graph LR
    A[APScheduler] --> B[Cron-like]
    A --> C[Interval]
    A --> D[Date-based]

    B --> E[작업 실행]
    C --> E
    D --> E

    E --> F[로그]
    E --> G[알림]

    style A fill:#9C27B0
    style E fill:#4CAF50
```

### 패턴 1: 기본 스케줄링

```python
# tasks/scheduler.py
from apscheduler.schedulers.blocking import BlockingScheduler
from apscheduler.triggers.cron import CronTrigger
from apscheduler.triggers.interval import IntervalTrigger
import structlog
from datetime import datetime

logger = structlog.get_logger()

class TaskScheduler:
    def __init__(self):
        self.scheduler = BlockingScheduler()
        self._setup_jobs()

    def _setup_jobs(self):
        """작업 등록"""

        # 매일 오전 9시
        self.scheduler.add_job(
            self.daily_report,
            trigger=CronTrigger(hour=9, minute=0),
            id='daily_report',
            name='일일 리포트 생성'
        )

        # 1시간마다
        self.scheduler.add_job(
            self.hourly_check,
            trigger=IntervalTrigger(hours=1),
            id='hourly_check',
            name='시간별 체크'
        )

        # 매주 월요일 오전 10시
        self.scheduler.add_job(
            self.weekly_summary,
            trigger=CronTrigger(day_of_week='mon', hour=10, minute=0),
            id='weekly_summary',
            name='주간 요약'
        )

        # 특정 날짜/시간 (일회성)
        self.scheduler.add_job(
            self.one_time_task,
            trigger='date',
            run_date=datetime(2026, 12, 31, 23, 59, 59),
            id='year_end_task'
        )

    def daily_report(self):
        """일일 리포트 생성"""
        logger.info("daily_report_started")

        try:
            # 데이터 수집
            data = self._collect_daily_data()

            # 리포트 생성
            report = self._generate_report(data)

            # 저장
            self._save_report(report, f"daily_report_{datetime.now():%Y%m%d}.pdf")

            # 알림 전송
            self._send_notification("일일 리포트 생성 완료")

            logger.info("daily_report_completed")

        except Exception as e:
            logger.error("daily_report_failed", error=str(e))
            self._send_alert(f"일일 리포트 실패: {e}")

    def hourly_check(self):
        """시간별 시스템 체크"""
        logger.info("hourly_check_started")

        # 시스템 상태 확인
        # 데이터베이스 연결 확인
        # 디스크 공간 확인
        # ...

        logger.info("hourly_check_completed")

    def weekly_summary(self):
        """주간 요약"""
        logger.info("weekly_summary_started")

        # 지난 주 데이터 집계
        # 주간 리포트 생성
        # 이메일 발송

        logger.info("weekly_summary_completed")

    def one_time_task(self):
        """일회성 작업"""
        logger.info("one_time_task_executed")

    def start(self):
        """스케줄러 시작"""
        logger.info("scheduler_started", jobs=len(self.scheduler.get_jobs()))

        try:
            self.scheduler.start()
        except (KeyboardInterrupt, SystemExit):
            logger.info("scheduler_stopped")

# 실행
if __name__ == '__main__':
    scheduler = TaskScheduler()
    scheduler.start()
```

### 패턴 2: 비동기 스케줄링

```python
from apscheduler.schedulers.asyncio import AsyncIOScheduler
import asyncio
import httpx

class AsyncTaskScheduler:
    def __init__(self):
        self.scheduler = AsyncIOScheduler()
        self.http_client = httpx.AsyncClient()

    async def fetch_api_data(self):
        """API 데이터 수집 (비동기)"""
        logger.info("fetching_api_data")

        endpoints = [
            '/api/users',
            '/api/orders',
            '/api/products'
        ]

        # 병렬 요청
        tasks = [self.http_client.get(f'https://api.example.com{ep}') for ep in endpoints]
        responses = await asyncio.gather(*tasks)

        for endpoint, response in zip(endpoints, responses):
            data = response.json()
            logger.info("api_data_fetched", endpoint=endpoint, records=len(data))

    async def process_queue(self):
        """큐 처리"""
        logger.info("processing_queue")

        # Redis 큐에서 작업 가져오기
        # 비동기 처리
        # ...

    def start(self):
        """스케줄러 시작"""
        # 작업 등록
        self.scheduler.add_job(
            self.fetch_api_data,
            trigger=IntervalTrigger(minutes=5),
            id='api_fetch'
        )

        self.scheduler.add_job(
            self.process_queue,
            trigger=IntervalTrigger(seconds=30),
            id='queue_process'
        )

        self.scheduler.start()

        # 이벤트 루프 유지
        asyncio.get_event_loop().run_forever()
```

### 패턴 3: 조건부 스케줄링

```python
class ConditionalScheduler:
    """조건에 따라 작업 실행"""

    def __init__(self):
        self.scheduler = BlockingScheduler()

    def should_run_backup(self) -> bool:
        """백업 실행 조건 확인"""
        # 주말에만 실행
        now = datetime.now()
        if now.weekday() >= 5:  # 토, 일
            return True

        # 디스크 공간 확인
        import shutil
        stat = shutil.disk_usage("/")
        free_gb = stat.free / (1024**3)

        return free_gb > 10  # 10GB 이상 여유 있을 때만

    def conditional_backup(self):
        """조건부 백업"""
        if not self.should_run_backup():
            logger.info("backup_skipped", reason="condition_not_met")
            return

        logger.info("backup_started")
        # 백업 로직
        logger.info("backup_completed")

    def start(self):
        self.scheduler.add_job(
            self.conditional_backup,
            trigger=CronTrigger(hour=2, minute=0),  # 매일 새벽 2시 체크
            id='conditional_backup'
        )
        self.scheduler.start()
```

---

## CLI 자동화 도구

```mermaid
graph TD
    A[CLI 명령] --> B[click/typer]
    B --> C[인자 파싱]
    C --> D[유효성 검증]
    D --> E[작업 실행]
    E --> F[결과 출력]
    E --> G[로그 기록]

    style B fill:#FF6B6B
    style E fill:#4ECDC4
```

### 패턴 1: Click 기반 CLI

```python
# cli/main.py
import click
from pathlib import Path
import structlog

logger = structlog.get_logger()

@click.group()
@click.option('--config', type=click.Path(exists=True), help='설정 파일 경로')
@click.pass_context
def cli(ctx, config):
    """자동화 CLI 도구"""
    ctx.ensure_object(dict)
    ctx.obj['config'] = config

    if config:
        click.echo(f"설정 파일 로드: {config}")

@cli.command()
@click.option('--source', '-s', required=True, help='소스 디렉토리')
@click.option('--dest', '-d', required=True, help='대상 디렉토리')
@click.option('--pattern', default='*.csv', help='파일 패턴')
@click.option('--dry-run', is_flag=True, help='실제 실행 없이 미리보기')
def sync(source, dest, pattern, dry_run):
    """파일 동기화"""

    source_path = Path(source)
    dest_path = Path(dest)

    if not source_path.exists():
        click.secho(f"- 미지원: 소스 디렉토리가 없습니다: {source}", fg='red')
        return

    files = list(source_path.glob(pattern))

    if dry_run:
        click.secho(f"🔍 미리보기 모드 ({len(files)}개 파일)", fg='yellow')
        for f in files:
            click.echo(f"  - {f.name}")
        return

    with click.progressbar(files, label='동기화 중') as bar:
        for file in bar:
            import shutil
            shutil.copy2(file, dest_path / file.name)

    click.secho(f"- {len(files)}개 파일 동기화 완료", fg='green')

@cli.command()
@click.argument('url')
@click.option('--output', '-o', type=click.File('w'), default='-')
@click.option('--format', type=click.Choice(['json', 'csv', 'text']), default='json')
def fetch(url, output, format):
    """URL에서 데이터 가져오기"""

    import httpx

    with click.spinner('데이터 가져오는 중...'):
        response = httpx.get(url, timeout=30)

    if response.status_code != 200:
        click.secho(f"- 미지원: HTTP {response.status_code}", fg='red')
        return

    if format == 'json':
        import json
        data = response.json()
        output.write(json.dumps(data, indent=2, ensure_ascii=False))
    elif format == 'text':
        output.write(response.text)

    click.secho(f"- 데이터 저장 완료", fg='green')

@cli.command()
@click.option('--days', default=7, help='처리할 일수')
@click.option('--verbose', '-v', is_flag=True, help='자세한 출력')
def process(days, verbose):
    """데이터 처리"""

    if verbose:
        click.echo(f"처리 기간: 최근 {days}일")

    # 데이터 처리 로직
    import time

    with click.progressbar(range(days), label='처리 중') as bar:
        for day in bar:
            time.sleep(0.1)  # 시뮬레이션

    click.secho("- 처리 완료", fg='green')

@cli.command()
def status():
    """시스템 상태 확인"""

    click.echo(click.style("시스템 상태", bold=True, fg='blue'))
    click.echo("─" * 40)

    # CPU
    import psutil
    cpu_percent = psutil.cpu_percent(interval=1)
    click.echo(f"CPU: {cpu_percent}%")

    # 메모리
    mem = psutil.virtual_memory()
    click.echo(f"메모리: {mem.percent}% (사용 중: {mem.used / 1024**3:.1f}GB)")

    # 디스크
    disk = psutil.disk_usage('/')
    click.echo(f"디스크: {disk.percent}% (여유: {disk.free / 1024**3:.1f}GB)")

if __name__ == '__main__':
    cli()
```

### 패턴 2: Typer 기반 CLI (타입 안전)

```python
import typer
from typing import Optional, List
from pathlib import Path
from enum import Enum

app = typer.Typer()

class OutputFormat(str, Enum):
    JSON = "json"
    CSV = "csv"
    TEXT = "text"

@app.command()
def convert(
    input_file: Path = typer.Argument(..., help="입력 파일"),
    output_file: Optional[Path] = typer.Option(None, "--output", "-o", help="출력 파일"),
    format: OutputFormat = typer.Option(OutputFormat.JSON, help="출력 형식"),
    overwrite: bool = typer.Option(False, "--overwrite", help="기존 파일 덮어쓰기"),
):
    """파일 형식 변환"""

    if not input_file.exists():
        typer.secho(f"- 미지원: 파일이 없습니다: {input_file}", fg=typer.colors.RED)
        raise typer.Exit(1)

    if output_file and output_file.exists() and not overwrite:
        typer.secho(f"- 레거시:  파일이 이미 존재합니다: {output_file}", fg=typer.colors.YELLOW)
        if not typer.confirm("덮어쓰시겠습니까?"):
            raise typer.Exit()

    typer.echo(f"변환 중: {input_file} → {format.value}")

    # 변환 로직
    # ...

    typer.secho("- 변환 완료", fg=typer.colors.GREEN)

@app.command()
def batch(
    files: List[Path] = typer.Argument(..., help="처리할 파일들"),
    workers: int = typer.Option(4, "--workers", "-w", help="병렬 워커 수"),
):
    """일괄 처리"""

    from concurrent.futures import ThreadPoolExecutor

    typer.echo(f"처리할 파일: {len(files)}개, 워커: {workers}개")

    with ThreadPoolExecutor(max_workers=workers) as executor:
        results = list(executor.map(process_file, files))

    typer.secho(f"- {len(results)}개 파일 처리 완료", fg=typer.colors.GREEN)

def process_file(file: Path):
    """파일 처리"""
    # 처리 로직
    pass

if __name__ == "__main__":
    app()
```

---

## 웹 스크래핑 자동화

### 패턴 1: httpx + BeautifulSoup

```python
# tasks/scraper.py
import httpx
from bs4 import BeautifulSoup
import asyncio
from typing import List, Dict
import structlog

logger = structlog.get_logger()

class WebScraper:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.client = httpx.AsyncClient(
            timeout=30.0,
            headers={'User-Agent': 'Mozilla/5.0 (compatible; Bot/1.0)'}
        )

    async def scrape_page(self, url: str) -> Dict:
        """단일 페이지 스크래핑"""
        logger.info("scraping_page", url=url)

        response = await self.client.get(url)
        response.raise_for_status()

        soup = BeautifulSoup(response.text, 'html.parser')

        # 데이터 추출
        data = {
            'title': soup.find('h1').text.strip() if soup.find('h1') else None,
            'content': soup.find('article').text.strip() if soup.find('article') else None,
            'links': [a['href'] for a in soup.find_all('a', href=True)],
            'images': [img['src'] for img in soup.find_all('img', src=True)],
        }

        return data

    async def scrape_multiple(self, urls: List[str]) -> List[Dict]:
        """여러 페이지 병렬 스크래핑"""
        logger.info("scraping_multiple", count=len(urls))

        tasks = [self.scrape_page(url) for url in urls]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        # 에러 처리
        success_count = sum(1 for r in results if not isinstance(r, Exception))
        logger.info("scraping_completed", success=success_count, total=len(urls))

        return [r for r in results if not isinstance(r, Exception)]

    async def scrape_with_pagination(self, start_url: str, max_pages: int = 10) -> List[Dict]:
        """페이지네이션 처리"""
        all_data = []
        current_url = start_url

        for page in range(1, max_pages + 1):
            logger.info("scraping_page", page=page, url=current_url)

            response = await self.client.get(current_url)
            soup = BeautifulSoup(response.text, 'html.parser')

            # 데이터 추출
            items = soup.find_all('div', class_='item')
            for item in items:
                all_data.append({
                    'title': item.find('h2').text.strip(),
                    'price': item.find('span', class_='price').text.strip(),
                    'url': item.find('a')['href'],
                })

            # 다음 페이지 찾기
            next_link = soup.find('a', class_='next')
            if not next_link:
                break

            current_url = next_link['href']

            # Rate limiting
            await asyncio.sleep(1)

        logger.info("pagination_complete", total_items=len(all_data))
        return all_data

    async def close(self):
        await self.client.aclose()

# 사용 예시
async def main():
    scraper = WebScraper("https://example.com")

    # 단일 페이지
    data = await scraper.scrape_page("https://example.com/article/123")

    # 여러 페이지
    urls = [f"https://example.com/article/{i}" for i in range(1, 11)]
    results = await scraper.scrape_multiple(urls)

    # 페이지네이션
    all_items = await scraper.scrape_with_pagination("https://example.com/products")

    await scraper.close()

if __name__ == '__main__':
    asyncio.run(main())
```

### 패턴 2: Playwright 브라우저 자동화

```python
from playwright.async_api import async_playwright, Browser, Page
import asyncio

class BrowserAutomation:
    """Playwright 브라우저 자동화"""

    async def __aenter__(self):
        self.playwright = await async_playwright().start()
        self.browser = await self.playwright.chromium.launch(headless=True)
        return self

    async def __aexit__(self, *args):
        await self.browser.close()
        await self.playwright.stop()

    async def scrape_spa(self, url: str) -> Dict:
        """SPA (Single Page Application) 스크래핑"""
        page = await self.browser.new_page()

        try:
            logger.info("loading_spa", url=url)

            # 페이지 로드
            await page.goto(url, wait_until='networkidle')

            # JavaScript 렌더링 대기
            await page.wait_for_selector('.content', timeout=10000)

            # 데이터 추출
            data = await page.evaluate('''() => {
                return {
                    title: document.querySelector('h1')?.textContent,
                    items: Array.from(document.querySelectorAll('.item')).map(el => ({
                        name: el.querySelector('.name')?.textContent,
                        price: el.querySelector('.price')?.textContent
                    }))
                };
            }''')

            return data

        finally:
            await page.close()

    async def fill_form(self, url: str, form_data: Dict):
        """폼 자동 입력"""
        page = await self.browser.new_page()

        try:
            await page.goto(url)

            # 폼 입력
            await page.fill('input[name="username"]', form_data['username'])
            await page.fill('input[name="password"]', form_data['password'])

            # 제출
            await page.click('button[type="submit"]')

            # 결과 대기
            await page.wait_for_url('**/dashboard')

            logger.info("form_submitted", url=page.url)

        finally:
            await page.close()

    async def take_screenshot(self, url: str, output_path: str):
        """스크린샷 캡처"""
        page = await self.browser.new_page()

        try:
            await page.goto(url, wait_until='networkidle')
            await page.screenshot(path=output_path, full_page=True)

            logger.info("screenshot_saved", path=output_path)

        finally:
            await page.close()

# 사용
async def main():
    async with BrowserAutomation() as bot:
        # SPA 스크래핑
        data = await bot.scrape_spa("https://app.example.com")

        # 폼 제출
        await bot.fill_form("https://example.com/login", {
            'username': 'user@example.com',
            'password': 'secret'
        })

        # 스크린샷
        await bot.take_screenshot("https://example.com", "screenshot.png")

if __name__ == '__main__':
    asyncio.run(main())
```

---

## 파일 감시 자동화

```mermaid
graph LR
    A[파일 시스템] --> B[watchdog]
    B --> C{이벤트 감지}
    C -->|생성| D[파일 생성]
    C -->|수정| E[파일 수정]
    C -->|삭제| F[파일 삭제]

    D --> G[처리 로직]
    E --> G
    F --> G

    style B fill:#FF6B6B
    style G fill:#4ECDC4
```

### 패턴 1: 파일 변경 감지

```python
# tasks/file_watcher.py
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler, FileSystemEvent
from pathlib import Path
import time
import structlog

logger = structlog.get_logger()

class FileWatcher(FileSystemEventHandler):
    """파일 변경 감지 핸들러"""

    def __init__(self, processor_func=None):
        super().__init__()
        self.processor = processor_func or self.default_processor

    def on_created(self, event: FileSystemEvent):
        """파일 생성 시"""
        if event.is_directory:
            return

        logger.info("file_created", path=event.src_path)
        self.processor(event.src_path, action='created')

    def on_modified(self, event: FileSystemEvent):
        """파일 수정 시"""
        if event.is_directory:
            return

        logger.info("file_modified", path=event.src_path)
        self.processor(event.src_path, action='modified')

    def on_deleted(self, event: FileSystemEvent):
        """파일 삭제 시"""
        if event.is_directory:
            return

        logger.info("file_deleted", path=event.src_path)

    def default_processor(self, filepath: str, action: str):
        """기본 처리 로직"""
        logger.info("processing_file", filepath=filepath, action=action)

class AutoProcessor:
    """파일 자동 처리"""

    def __init__(self, watch_dir: str, output_dir: str):
        self.watch_dir = Path(watch_dir)
        self.output_dir = Path(output_dir)
        self.output_dir.mkdir(parents=True, exist_ok=True)

    def process_csv(self, filepath: str, action: str):
        """CSV 파일 처리"""
        if not filepath.endswith('.csv'):
            return

        logger.info("processing_csv", filepath=filepath)

        import polars as pl

        # CSV 읽기
        df = pl.read_csv(filepath)

        # 처리 (예: 필터링, 변환)
        df_processed = df.filter(pl.col("value") > 0)

        # 결과 저장
        output_path = self.output_dir / f"processed_{Path(filepath).name}"
        df_processed.write_csv(output_path)

        logger.info("csv_processed", output=str(output_path), rows=len(df_processed))

    def process_image(self, filepath: str, action: str):
        """이미지 처리"""
        if not filepath.lower().endswith(('.png', '.jpg', '.jpeg')):
            return

        logger.info("processing_image", filepath=filepath)

        from PIL import Image

        # 이미지 열기
        img = Image.open(filepath)

        # 리사이즈
        img_resized = img.resize((800, 600))

        # 저장
        output_path = self.output_dir / f"resized_{Path(filepath).name}"
        img_resized.save(output_path)

        logger.info("image_processed", output=str(output_path))

    def start_watching(self):
        """파일 감시 시작"""
        handler = FileWatcher(processor_func=self.process_csv)
        observer = Observer()
        observer.schedule(handler, str(self.watch_dir), recursive=False)
        observer.start()

        logger.info("file_watcher_started", directory=str(self.watch_dir))

        try:
            while True:
                time.sleep(1)
        except KeyboardInterrupt:
            observer.stop()
            logger.info("file_watcher_stopped")

        observer.join()

# 사용
if __name__ == '__main__':
    processor = AutoProcessor(
        watch_dir='data/input',
        output_dir='data/output'
    )
    processor.start_watching()
```

### 패턴 2: 패턴 기반 처리

```python
import re
from typing import Callable, Dict

class PatternBasedWatcher:
    """파일 패턴별 처리"""

    def __init__(self, watch_dir: str):
        self.watch_dir = Path(watch_dir)
        self.handlers: Dict[str, Callable] = {}

    def register_handler(self, pattern: str, handler: Callable):
        """패턴 핸들러 등록"""
        self.handlers[pattern] = handler
        logger.info("handler_registered", pattern=pattern)

    def dispatch(self, filepath: str, action: str):
        """패턴에 맞는 핸들러 실행"""
        filename = Path(filepath).name

        for pattern, handler in self.handlers.items():
            if re.match(pattern, filename):
                logger.info("handler_matched", pattern=pattern, file=filename)
                handler(filepath, action)
                return

        logger.info("no_handler_matched", file=filename)

# 사용
watcher = PatternBasedWatcher('data/input')

# 핸들러 등록
watcher.register_handler(r'sales_\d{8}\.csv', process_sales_report)
watcher.register_handler(r'invoice_.*\.pdf', process_invoice)
watcher.register_handler(r'log_.*\.txt', process_log_file)

# 감시 시작
handler = FileWatcher(processor_func=watcher.dispatch)
observer = Observer()
observer.schedule(handler, 'data/input', recursive=False)
observer.start()
```

---

## 작업 큐 자동화

### Dramatiq으로 비동기 작업 처리

```python
import dramatiq
from dramatiq.brokers.redis import RedisBroker
import structlog

logger = structlog.get_logger()

# Redis 브로커 설정
redis_broker = RedisBroker(host="localhost", port=6379)
dramatiq.set_broker(redis_broker)

@dramatiq.actor(max_retries=3, min_backoff=1000, max_backoff=60000)
def send_email(to: str, subject: str, body: str):
    """이메일 전송 (비동기)"""
    logger.info("sending_email", to=to, subject=subject)

    import smtplib
    from email.message import EmailMessage

    msg = EmailMessage()
    msg['Subject'] = subject
    msg['From'] = 'noreply@example.com'
    msg['To'] = to
    msg.set_content(body)

    # SMTP 전송
    # with smtplib.SMTP('localhost') as smtp:
    #     smtp.send_message(msg)

    logger.info("email_sent", to=to)

@dramatiq.actor
def process_image_async(filepath: str):
    """이미지 처리 (비동기)"""
    logger.info("processing_image_async", filepath=filepath)

    from PIL import Image

    img = Image.open(filepath)
    # 처리 로직

    logger.info("image_processed_async", filepath=filepath)

@dramatiq.actor
def generate_report(data: dict):
    """리포트 생성 (비동기)"""
    logger.info("generating_report", data_keys=list(data.keys()))

    # 리포트 생성 로직
    report_path = f"reports/report_{data['id']}.pdf"

    # 완료 후 이메일 전송 (작업 체인)
    send_email.send(
        to=data['email'],
        subject="리포트 생성 완료",
        body=f"리포트가 생성되었습니다: {report_path}"
    )

# 작업 큐에 추가
send_email.send("user@example.com", "테스트", "본문")
process_image_async.send("/path/to/image.jpg")
generate_report.send({'id': 123, 'email': 'user@example.com'})
```

---

## 알림 자동화

### 패턴 1: Slack 알림

```python
# tasks/notifications.py
from slack_sdk import WebClient
from slack_sdk.errors import SlackApiError
import structlog

logger = structlog.get_logger()

class SlackNotifier:
    def __init__(self, token: str, channel: str):
        self.client = WebClient(token=token)
        self.channel = channel

    def send_message(self, text: str, blocks=None):
        """메시지 전송"""
        try:
            response = self.client.chat_postMessage(
                channel=self.channel,
                text=text,
                blocks=blocks
            )
            logger.info("slack_message_sent", channel=self.channel)
            return response

        except SlackApiError as e:
            logger.error("slack_error", error=str(e))

    def send_alert(self, title: str, message: str, severity: str = "info"):
        """알림 전송"""
        color = {
            'info': '#36a64f',
            'warning': '#ff9800',
            'error': '#f44336'
        }[severity]

        blocks = [
            {
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"*{title}*\n{message}"
                }
            }
        ]

        self.send_message(text=title, blocks=blocks)

    def send_report(self, title: str, metrics: dict):
        """리포트 전송"""
        fields = [
            {
                "type": "mrkdwn",
                "text": f"*{key}*\n{value}"
            }
            for key, value in metrics.items()
        ]

        blocks = [
            {
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"*{title}*"
                }
            },
            {
                "type": "section",
                "fields": fields
            }
        ]

        self.send_message(text=title, blocks=blocks)

# 사용
notifier = SlackNotifier(
    token="xoxb-your-token",
    channel="#alerts"
)

# 알림 전송
notifier.send_alert(
    title="데이터 처리 완료",
    message="1000개 레코드 처리 완료",
    severity="info"
)

# 리포트 전송
notifier.send_report(
    title="일일 통계",
    metrics={
        "처리된 파일": "150개",
        "총 레코드": "50,000개",
        "에러": "0개"
    }
)
```

---

## 통합 자동화 시스템

### 모든 기능 통합

```python
# main.py
from tasks.scheduler import TaskScheduler
from tasks.file_watcher import AutoProcessor
from tasks.notifications import SlackNotifier
import structlog
from concurrent.futures import ThreadPoolExecutor

logger = structlog.get_logger()

class AutomationSystem:
    """통합 자동화 시스템"""

    def __init__(self, config: dict):
        self.config = config
        self.notifier = SlackNotifier(
            token=config['slack_token'],
            channel=config['slack_channel']
        )

    def start_scheduler(self):
        """스케줄러 시작"""
        scheduler = TaskScheduler()
        scheduler.start()

    def start_file_watcher(self):
        """파일 감시 시작"""
        processor = AutoProcessor(
            watch_dir=self.config['watch_dir'],
            output_dir=self.config['output_dir']
        )
        processor.start_watching()

    def run(self):
        """시스템 실행"""
        logger.info("automation_system_starting")

        # 시작 알림
        self.notifier.send_alert(
            title="자동화 시스템 시작",
            message="모든 서비스 가동 중",
            severity="info"
        )

        # 병렬 실행
        with ThreadPoolExecutor(max_workers=2) as executor:
            executor.submit(self.start_scheduler)
            executor.submit(self.start_file_watcher)

# 실행
if __name__ == '__main__':
    config = {
        'slack_token': 'xoxb-...',
        'slack_channel': '#automation',
        'watch_dir': 'data/input',
        'output_dir': 'data/output'
    }

    system = AutomationSystem(config)
    system.run()
```

---

## 모니터링 및 로깅

### structlog 설정

```python
# utils/logger.py
import structlog
import logging

def setup_logging():
    structlog.configure(
        processors=[
            structlog.stdlib.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.JSONRenderer()
        ],
        logger_factory=structlog.stdlib.LoggerFactory(),
    )

    logging.basicConfig(level=logging.INFO)
```

---

## 프로덕션 배포

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  automation:
    build: .
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
    environment:
      - REDIS_URL=redis://redis:6379
      - SLACK_TOKEN=${SLACK_TOKEN}
    depends_on:
      - redis

  dramatiq-worker:
    build: .
    command: dramatiq tasks.workers
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

---

## 마무리

여기 나온 패턴들은 실제로 자주 쓰는 것들입니다. APScheduler로 스케줄링하고, watchdog로 파일 감시하고, Playwright로 웹 스크래핑하는 건 대부분의 자동화 프로젝트에서 필요한 기능들이에요.

처음엔 간단하게 시작하세요:
- 스케줄링만 필요하면 APScheduler만 써도 충분
- CLI 도구는 click이 typer보다 더 널리 쓰임 (typer는 더 현대적이긴 함)
- 웹 스크래핑은 일단 httpx + BeautifulSoup로 시도하고, JavaScript 렌더링이 필요하면 Playwright 도입

작업 큐(dramatiq)는 규모가 커지면 필요한데, 처음부터 넣으면 오히려 복잡해질 수 있습니다. 단순한 스케줄링으로 버틸 수 있으면 그게 더 나아요.

---

**[← 데이터 파이프라인](data-pipeline.md)** | **[라이브러리 카탈로그 →](../04-library-catalog/README.md)**
