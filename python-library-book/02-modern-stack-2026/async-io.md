# 비동기 I/O & 네트워킹

## 2026 표준 스택

| 도구 | 등급 | 역할 |
|------|------|------|
| `asyncio` | Yes | 표준 비동기 라이브러리 |
| `httpx` | Yes | 비동기 HTTP 클라이언트 |
| `aiofiles` | Yes | 비동기 파일 I/O |

---

## 1. asyncio: 표준 비동기 ⭐

### 기본 개념

```python
import asyncio

# 비동기 함수 정의
async def say_hello():
    print("Hello")
    await asyncio.sleep(1)
    print("World")

# 실행
asyncio.run(say_hello())
```

### 동시 실행

```python
async def fetch_data(n):
    await asyncio.sleep(1)
    return f"Data {n}"

async def main():
    # 순차 실행 (느림)
    data1 = await fetch_data(1)
    data2 = await fetch_data(2)
    # 총 2초

    # 병렬 실행 (빠름)
    results = await asyncio.gather(
        fetch_data(1),
        fetch_data(2),
        fetch_data(3),
    )
    # 총 1초
    print(results)

asyncio.run(main())
```

---

## 2. httpx: 현대적 HTTP 클라이언트 ⭐

### 핵심 특징

- **requests와 호환되는 API**
- **async/await 지원**
- **HTTP/2 지원**

### 동기 사용 (requests 대체)

```python
import httpx

# requests와 동일한 API
response = httpx.get("https://api.example.com/users")
print(response.json())

# Context manager
with httpx.Client() as client:
    response = client.get("https://api.example.com")
```

### 비동기 사용 (권장)

```python
import httpx
import asyncio

async def fetch_user(user_id):
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://api.example.com/users/{user_id}")
        return response.json()

async def main():
    # 여러 요청 병렬 처리
    users = await asyncio.gather(
        fetch_user(1),
        fetch_user(2),
        fetch_user(3),
    )
    print(users)

asyncio.run(main())
```

### 실전 예제: API 래퍼

```python
import httpx
from typing import Optional

class APIClient:
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        self.headers = {"Authorization": f"Bearer {api_key}"}

    async def get(self, endpoint: str, params: Optional[dict] = None):
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{self.base_url}/{endpoint}",
                params=params,
                headers=self.headers,
                timeout=10.0
            )
            response.raise_for_status()
            return response.json()

    async def post(self, endpoint: str, data: dict):
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/{endpoint}",
                json=data,
                headers=self.headers
            )
            response.raise_for_status()
            return response.json()
```

---

## 3. aiofiles: 비동기 파일 I/O ⭐

```python
import aiofiles
import asyncio

async def read_file(filename):
    async with aiofiles.open(filename, mode='r') as f:
        contents = await f.read()
    return contents

async def write_file(filename, content):
    async with aiofiles.open(filename, mode='w') as f:
        await f.write(content)

async def main():
    # 병렬 파일 읽기
    files = await asyncio.gather(
        read_file('file1.txt'),
        read_file('file2.txt'),
        read_file('file3.txt'),
    )
    print(files)

asyncio.run(main())
```

---

## 실전 패턴

### 재시도 로직

```python
import httpx
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
async def fetch_with_retry(url: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()
```

### 속도 제한

```python
import asyncio
from asyncio import Semaphore

async def rate_limited_fetch(url: str, semaphore: Semaphore):
    async with semaphore:
        async with httpx.AsyncClient() as client:
            response = await client.get(url)
            return response.json()

async def main():
    # 동시에 최대 5개 요청
    semaphore = Semaphore(5)

    urls = [f"https://api.example.com/item/{i}" for i in range(100)]
    tasks = [rate_limited_fetch(url, semaphore) for url in urls]
    results = await asyncio.gather(*tasks)
```

---

## 대체 도구들

### requests ⚠️

- **이유**: httpx가 async 지원하면서 동일 API
- **언제 OK**: 완전 동기 코드, 단순 스크립트

### aiohttp ⚠️

- **이유**: httpx가 더 직관적 (requests 호환 API)
- **언제 OK**: 서버와 클라이언트 모두 필요

---

**[← 웹 API](web-api.md)** | **[관측성 →](observability.md)**
