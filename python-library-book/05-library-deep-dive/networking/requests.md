# requests 완벽 가이드

> **가장 인기있는 HTTP 라이브러리**


---

## 개요

| 항목 | 내용 |
|------|------|
| **공식 사이트** | https://requests.readthedocs.io |
| **GitHub** | https://github.com/psf/requests |
| **PyPI** | https://pypi.org/project/requests/ |
| **첫 릴리즈** | 2011년 |

### 한 줄 요약

**"인간을 위한 HTTP" - 가장 직관적인 Python HTTP 클라이언트**

---

## 왜 requests인가

### HTTP를 쉽게 만든 라이브러리

```python
# urllib (표준 라이브러리) - 복잡함
import urllib.request
import urllib.parse

url = 'https://api.example.com/data'
params = urllib.parse.urlencode({'key': 'value'})
req = urllib.request.Request(f'{url}?{params}')
req.add_header('User-Agent', 'Mozilla/5.0')
response = urllib.request.urlopen(req)
data = response.read().decode('utf-8')

# requests - 간단함!
import requests

response = requests.get('https://api.example.com/data', params={'key': 'value'})
data = response.json()
```

---

## - 레거시: 2026년 주의사항

### 왜 레거시인가?

| 항목 | requests | httpx |
|------|----------|-------|
| 비동기 지원 | - 미지원: | - |
| HTTP/2 | - 미지원: | - |
| 마지막 업데이트 | 2023 | 활발 |
| 유지보수 | 최소한 | 적극적 |

**권장 사항:**
- - 새 프로젝트: httpx 사용
- - 기존 프로젝트: requests 유지 가능
- - 비동기 필요: httpx로 마이그레이션

---

## 설치

```bash
$ uv add requests
```

---

## 기본 사용법

### GET 요청

```python
import requests

# 기본 GET
response = requests.get('https://api.example.com/users')
print(response.status_code)  # 200
print(response.json())        # JSON 파싱

# 쿼리 파라미터
response = requests.get('https://api.example.com/search', params={
    'q': 'python',
    'page': 1,
    'limit': 10
})

# 헤더
response = requests.get('https://api.example.com/protected', headers={
    'Authorization': 'Bearer token123',
    'User-Agent': 'MyApp/1.0'
})
```

### POST 요청

```python
# JSON 전송
response = requests.post('https://api.example.com/users', json={
    'name': 'Alice',
    'email': 'alice@example.com'
})

# 폼 데이터
response = requests.post('https://api.example.com/login', data={
    'username': 'alice',
    'password': 'secret'
})

# 파일 업로드
files = {'file': open('document.pdf', 'rb')}
response = requests.post('https://api.example.com/upload', files=files)
```

### 응답 처리

```python
response = requests.get('https://api.example.com/data')

# 상태 코드
print(response.status_code)        # 200
response.raise_for_status()         # 에러 발생시 예외

# 응답 본문
print(response.text)                # 문자열
print(response.content)             # 바이트
print(response.json())              # JSON 파싱

# 헤더
print(response.headers['Content-Type'])
print(response.encoding)            # 인코딩
```

---

## 실전 패턴

### 패턴 1: 세션 재사용

```python
# 나쁜 예 - 매번 연결
for i in range(100):
    response = requests.get(f'https://api.example.com/users/{i}')

# 좋은 예 - 세션 재사용 (연결 풀링)
session = requests.Session()
session.headers.update({'Authorization': 'Bearer token123'})

for i in range(100):
    response = session.get(f'https://api.example.com/users/{i}')
```

### 패턴 2: 타임아웃

```python
# 항상 타임아웃 설정!
try:
    response = requests.get('https://api.example.com/data', timeout=5)
except requests.Timeout:
    print("요청 시간 초과")

# 연결/읽기 타임아웃 분리
response = requests.get('https://api.example.com/data',
                       timeout=(3.05, 10))  # (연결, 읽기)
```

### 패턴 3: 재시도

```python
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

session = requests.Session()

# 재시도 전략
retry_strategy = Retry(
    total=3,                      # 총 3회 시도
    backoff_factor=1,             # 1, 2, 4초 대기
    status_forcelist=[429, 500, 502, 503, 504],
    allowed_methods=["GET", "POST"]
)

adapter = HTTPAdapter(max_retries=retry_strategy)
session.mount("https://", adapter)
session.mount("http://", adapter)

# 자동 재시도
response = session.get('https://api.example.com/data')
```

### 패턴 4: 인증

```python
from requests.auth import HTTPBasicAuth, HTTPDigestAuth

# Basic Auth
response = requests.get('https://api.example.com/data',
                       auth=HTTPBasicAuth('user', 'pass'))

# 또는 튜플로
response = requests.get('https://api.example.com/data',
                       auth=('user', 'pass'))

# Bearer Token (가장 흔함)
headers = {'Authorization': 'Bearer YOUR_TOKEN'}
response = requests.get('https://api.example.com/data', headers=headers)

# OAuth 2.0 (requests-oauthlib)
from requests_oauthlib import OAuth2Session

oauth = OAuth2Session(client_id, token=token)
response = oauth.get('https://api.example.com/data')
```

### 패턴 5: 에러 처리

```python
import requests
from requests.exceptions import (
    RequestException,
    HTTPError,
    ConnectionError,
    Timeout
)

try:
    response = requests.get('https://api.example.com/data', timeout=5)
    response.raise_for_status()
    data = response.json()

except HTTPError as e:
    # 4xx, 5xx 에러
    print(f"HTTP 에러: {e}")
    print(f"상태 코드: {e.response.status_code}")

except ConnectionError:
    # 네트워크 연결 실패
    print("연결 실패")

except Timeout:
    # 타임아웃
    print("요청 시간 초과")

except RequestException as e:
    # 모든 requests 에러의 베이스
    print(f"요청 실패: {e}")
```

### 패턴 6: 대용량 파일 다운로드

```python
# 스트리밍 다운로드 (메모리 효율적)
url = 'https://example.com/large-file.zip'

with requests.get(url, stream=True) as response:
    response.raise_for_status()
    with open('large-file.zip', 'wb') as f:
        for chunk in response.iter_content(chunk_size=8192):
            f.write(chunk)

# 진행률 표시
from tqdm import tqdm

response = requests.get(url, stream=True)
total_size = int(response.headers.get('content-length', 0))

with open('file.zip', 'wb') as f:
    with tqdm(total=total_size, unit='B', unit_scale=True) as pbar:
        for chunk in response.iter_content(chunk_size=8192):
            f.write(chunk)
            pbar.update(len(chunk))
```

### 패턴 7: JSON API 클라이언트

```python
class APIClient:
    def __init__(self, base_url, api_key):
        self.session = requests.Session()
        self.session.headers.update({
            'Authorization': f'Bearer {api_key}',
            'Content-Type': 'application/json'
        })
        self.base_url = base_url

    def _request(self, method, endpoint, **kwargs):
        url = f'{self.base_url}{endpoint}'
        response = self.session.request(method, url, timeout=10, **kwargs)
        response.raise_for_status()
        return response.json()

    def get(self, endpoint, **kwargs):
        return self._request('GET', endpoint, **kwargs)

    def post(self, endpoint, data=None, **kwargs):
        return self._request('POST', endpoint, json=data, **kwargs)

    def put(self, endpoint, data=None, **kwargs):
        return self._request('PUT', endpoint, json=data, **kwargs)

    def delete(self, endpoint, **kwargs):
        return self._request('DELETE', endpoint, **kwargs)

# 사용
client = APIClient('https://api.example.com', 'token123')

users = client.get('/users')
user = client.post('/users', data={'name': 'Alice'})
client.delete(f'/users/{user["id"]}')
```

### 패턴 8: 동시 요청 (threading)

```python
import requests
from concurrent.futures import ThreadPoolExecutor, as_completed

def fetch_url(url):
    try:
        response = requests.get(url, timeout=5)
        return url, response.status_code
    except requests.RequestException as e:
        return url, str(e)

urls = [f'https://api.example.com/users/{i}' for i in range(100)]

# 10개 스레드로 동시 처리
with ThreadPoolExecutor(max_workers=10) as executor:
    futures = [executor.submit(fetch_url, url) for url in urls]

    for future in as_completed(futures):
        url, result = future.result()
        print(f"{url}: {result}")
```

### 패턴 9: 프록시

```python
# 프록시 설정
proxies = {
    'http': 'http://proxy.example.com:8080',
    'https': 'http://proxy.example.com:8080'
}

response = requests.get('https://api.example.com/data', proxies=proxies)

# 환경 변수 사용 (HTTP_PROXY, HTTPS_PROXY)
# 프록시 무시
response = requests.get('https://api.example.com/data', proxies={})
```

### 패턴 10: SSL 인증서 검증

```python
# 기본값: 검증 활성화
response = requests.get('https://api.example.com/data')

# 검증 비활성화 (개발 환경만!)
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

response = requests.get('https://api.example.com/data', verify=False)

# 커스텀 CA 인증서
response = requests.get('https://api.example.com/data',
                       verify='/path/to/certfile.pem')
```

---

## 고급 기능

### 쿠키 처리

```python
# 쿠키 전송
cookies = {'session_id': 'abc123'}
response = requests.get('https://example.com', cookies=cookies)

# 응답에서 쿠키 읽기
print(response.cookies['session_id'])

# 세션으로 쿠키 유지
session = requests.Session()
session.get('https://example.com/login')  # 쿠키 설정
session.get('https://example.com/dashboard')  # 쿠키 자동 전송
```

### Prepared Requests

```python
from requests import Request, Session

# 요청 사전 준비
req = Request('POST', 'https://api.example.com/data',
              data={'key': 'value'},
              headers={'User-Agent': 'MyApp'})

prepared = req.prepare()

# 요청 수정 가능
prepared.headers['Custom-Header'] = 'value'

# 실행
session = Session()
response = session.send(prepared, timeout=5)
```

### 이벤트 훅

```python
def print_url(response, *args, **kwargs):
    print(f"요청 URL: {response.url}")

# 응답 후 훅 실행
response = requests.get('https://api.example.com/data',
                       hooks={'response': print_url})
```

---

## httpx로 마이그레이션

### 기본 API는 동일

```python
# requests
import requests
response = requests.get('https://api.example.com/data')
data = response.json()

# httpx (동기 모드)
import httpx
response = httpx.get('https://api.example.com/data')
data = response.json()
```

### 비동기로 업그레이드

```python
# httpx (비동기 모드)
import httpx
import asyncio

async def fetch_data():
    async with httpx.AsyncClient() as client:
        response = await client.get('https://api.example.com/data')
        return response.json()

data = asyncio.run(fetch_data())
```

### 세션 마이그레이션

```python
# requests
session = requests.Session()
session.headers.update({'Authorization': 'Bearer token'})
response = session.get('https://api.example.com/data')

# httpx
client = httpx.Client(headers={'Authorization': 'Bearer token'})
response = client.get('https://api.example.com/data')
client.close()

# 또는 context manager
with httpx.Client(headers={'Authorization': 'Bearer token'}) as client:
    response = client.get('https://api.example.com/data')
```

---

## 자주 하는 실수

### 1. 타임아웃 설정 안함

```python
# - 미지원: 나쁜 예 - 무한 대기 가능
response = requests.get('https://slow-api.example.com/data')

# - 좋은 예
response = requests.get('https://slow-api.example.com/data', timeout=5)
```

### 2. 세션 미사용

```python
# - 미지원: 나쁜 예 - 매번 TCP 연결
for i in range(100):
    requests.get(f'https://api.example.com/users/{i}')

# - 좋은 예 - 연결 재사용
session = requests.Session()
for i in range(100):
    session.get(f'https://api.example.com/users/{i}')
```

### 3. 에러 처리 없음

```python
# - 미지원: 나쁜 예
response = requests.get('https://api.example.com/data')
data = response.json()

# - 좋은 예
try:
    response = requests.get('https://api.example.com/data', timeout=5)
    response.raise_for_status()
    data = response.json()
except requests.RequestException as e:
    print(f"요청 실패: {e}")
```

### 4. 스트리밍 미사용

```python
# - 미지원: 나쁜 예 - 대용량 파일은 메모리 부족
response = requests.get('https://example.com/huge-file.zip')
with open('file.zip', 'wb') as f:
    f.write(response.content)

# - 좋은 예 - 청크 단위 처리
with requests.get('https://example.com/huge-file.zip', stream=True) as r:
    with open('file.zip', 'wb') as f:
        for chunk in r.iter_content(chunk_size=8192):
            f.write(chunk)
```

---

## 언제 requests, 언제 httpx?

### requests 사용 적합:

- - 완전 동기 코드
- - 기존 프로젝트 (이미 requests 사용 중)
- - 간단한 스크립트
- - 레거시 시스템 호환성

### httpx로 전환 고려:

- - 비동기 I/O 필요 (FastAPI, asyncio)
- - 여러 API 동시 호출
- - HTTP/2 필요
- - 새 프로젝트
- - 장기 유지보수 예정

---

## 추가 라이브러리

### requests-toolbelt

```bash
$ uv add requests-toolbelt
```

```python
from requests_toolbelt import MultipartEncoder

# 멀티파트 업로드
encoder = MultipartEncoder(
    fields={
        'field1': 'value1',
        'file': ('filename.txt', open('file.txt', 'rb'), 'text/plain')
    }
)

response = requests.post('https://api.example.com/upload',
                        data=encoder,
                        headers={'Content-Type': encoder.content_type})
```

### requests-cache

```bash
$ uv add requests-cache
```

```python
import requests_cache

# HTTP 캐싱
requests_cache.install_cache('demo_cache', expire_after=300)

# 첫 요청 - 실제 네트워크 호출
response = requests.get('https://api.example.com/data')

# 두 번째 요청 - 캐시에서 반환 (훨씬 빠름)
response = requests.get('https://api.example.com/data')
print(response.from_cache)  # True
```

---

## 베스트 프랙티스

### 1. 항상 타임아웃 설정

```python
TIMEOUT = 5
response = requests.get(url, timeout=TIMEOUT)
```

### 2. 세션 재사용

```python
session = requests.Session()
# 여러 요청...
session.close()
```

### 3. 에러 처리

```python
try:
    response = requests.get(url, timeout=5)
    response.raise_for_status()
except requests.RequestException as e:
    logger.error(f"Request failed: {e}")
```

### 4. 재시도 로직

```python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(total=3, backoff_factor=1)
adapter = HTTPAdapter(max_retries=retry)
session.mount('https://', adapter)
```

### 5. 로깅

```python
import logging
import http.client

# HTTP 요청/응답 로깅
http.client.HTTPConnection.debuglevel = 1
logging.basicConfig()
logging.getLogger().setLevel(logging.DEBUG)
```

---

## 요약

### requests의 강점:
- - 매우 직관적인 API
- - 풍부한 생태계 (캐싱, OAuth 등)
- - 검증된 안정성
- - 광범위한 커뮤니티 지원

### requests의 한계:
- - 미지원: 비동기 미지원
- - 미지원: HTTP/2 미지원
- - 미지원: 최소한의 유지보수

### 2026년 권장사항:
- **새 프로젝트**: httpx 사용
- **기존 프로젝트**: requests 유지 가능
- **비동기 필요시**: httpx로 전환

---

**[← 네트워킹](../../04-library-catalog/networking/README.md)**
