# FastAPI 완벽 가이드

> **2026년 파이썬 웹 API 개발의 표준**

⭐ **2026 추천** | 🌐 웹 프레임워크 | 🚀 비동기 | 📚 자동 문서화

---

## 목차

- [개요](#개요)
- [왜 FastAPI인가](#왜-fastapi인가)
- [핵심 개념](#핵심-개념)
- [설치 및 환경 설정](#설치-및-환경-설정)
- [기본 사용법](#기본-사용법)
- [실전 패턴 10가지](#실전-패턴-10가지)
- [함정 및 주의사항](#함정-및-주의사항)
- [성능 최적화](#성능-최적화)
- [다른 프레임워크와 비교](#다른-프레임워크와-비교)
- [프로덕션 체크리스트](#프로덕션-체크리스트)

---

## 개요

### 기본 정보

| 항목 | 내용 |
|------|------|
| **공식 사이트** | https://fastapi.tiangolo.com |
| **GitHub** | https://github.com/tiangolo/fastapi |
| **PyPI** | https://pypi.org/project/fastapi/ |
| **첫 릴리즈** | 2018년 12월 |
| **창시자** | Sebastián Ramírez (tiangolo) |
| **라이선스** | MIT |
| **Python 버전** | 3.8+ (3.12+ 권장) |

### 한 줄 요약

**현대적 타입 힌트를 기반으로 한 고성능 비동기 웹 프레임워크로, 자동 문서화와 데이터 검증을 제공**

---

## 왜 FastAPI인가

### 역사적 맥락

2018년 이전 파이썬 웹 프레임워크 상황:
- **Flask**: 동기 기반, 수동 문서화, 타입 검증 부족
- **Django**: 무겁고, API 전용으로는 과함
- **Tornado/Sanic**: 비동기지만 개발자 경험(DX) 부족

**FastAPI는 이 모든 문제를 해결하기 위해 탄생:**
- Starlette (비동기 프레임워크) 기반
- Pydantic (데이터 검증) 통합
- 현대 Python 타입 힌트 전면 활용

### 2026년 현재 위치

- **GitHub Stars**: 75k+ (Python 웹 프레임워크 중 가장 빠른 성장)
- **커뮤니티**: 매우 활발, 한국 커뮤니티도 활성화
- **채택**: Netflix, Uber, Microsoft 등 대기업 사용
- **트렌드**: 2026년 신규 API 프로젝트의 사실상 표준

---

## 핵심 개념

### 1. 타입 힌트 기반 개발

FastAPI의 핵심 철학:

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

# 타입 힌트만으로 모든 것이 해결됨
class Item(BaseModel):
    name: str
    price: float
    is_offer: bool = False

@app.post("/items/")
async def create_item(item: Item):
    # item은 자동으로 검증됨
    # 문서도 자동 생성됨
    # IDE 자동완성도 됨
    return {"name": item.name, "price": item.price}
```

**타입 힌트로 얻는 것:**
- ✅ 자동 데이터 검증
- ✅ 자동 직렬화/역직렬화
- ✅ 자동 API 문서 생성
- ✅ IDE 자동완성 및 타입 체크

### 2. 의존성 주입 (Dependency Injection)

FastAPI의 킬러 기능:

```python
from fastapi import Depends, HTTPException

# 의존성 정의
async def get_current_user(token: str):
    user = verify_token(token)
    if not user:
        raise HTTPException(status_code=401)
    return user

# 의존성 사용
@app.get("/users/me")
async def read_users_me(current_user: User = Depends(get_current_user)):
    return current_user
```

**의존성 주입의 장점:**
- 코드 재사용성 극대화
- 테스트 용이성 (mocking 쉬움)
- 관심사 분리

### 3. 비동기 우선 (Async First)

```python
import httpx

@app.get("/external")
async def call_external():
    async with httpx.AsyncClient() as client:
        # 비동기 호출 - 다른 요청 처리 가능
        response = await client.get("https://api.example.com")
    return response.json()
```

**비동기의 이점:**
- I/O 대기 시간 동안 다른 요청 처리
- 동시 처리 성능 10배+ 향상 가능
- 웹소켓, SSE 등 실시간 통신 지원

### 4. 자동 문서화

코드 작성 → 문서 자동 생성:
- **Swagger UI**: `/docs`
- **ReDoc**: `/redoc`
- **OpenAPI 스키마**: `/openapi.json`

---

## 설치 및 환경 설정

### 기본 설치

```bash
# 권장: uv 사용
$ uv add fastapi "uvicorn[standard]"

# 또는 pip
$ pip install fastapi "uvicorn[standard]"
```

### 개발 환경 전체 설정

```bash
# 프로젝트 생성
$ uv init myapi
$ cd myapi

# 의존성 추가
$ uv add fastapi "uvicorn[standard]" pydantic-settings

# 개발 의존성
$ uv add --dev pytest pytest-asyncio httpx ruff mypy
```

### 프로젝트 구조

```
myapi/
├── src/
│   └── myapi/
│       ├── __init__.py
│       ├── main.py           # 앱 진입점
│       ├── config.py         # 설정
│       ├── models/           # Pydantic 모델
│       │   ├── __init__.py
│       │   ├── user.py
│       │   └── item.py
│       ├── routers/          # API 라우터
│       │   ├── __init__.py
│       │   ├── users.py
│       │   └── items.py
│       ├── services/         # 비즈니스 로직
│       │   └── __init__.py
│       ├── dependencies.py   # 의존성
│       └── database.py       # DB 연결
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   └── test_api.py
├── .env
└── pyproject.toml
```

---

## 기본 사용법

### 1단계: 최소 앱

```python
# src/myapi/main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}

# 실행: uvicorn myapi.main:app --reload
# 접속: http://localhost:8000
# 문서: http://localhost:8000/docs
```

### 2단계: 경로 매개변수

```python
@app.get("/items/{item_id}")
async def read_item(item_id: int):
    # item_id는 자동으로 int로 변환됨
    # 숫자가 아니면 422 에러 자동 반환
    return {"item_id": item_id}

@app.get("/users/{user_id}/items/{item_id}")
async def read_user_item(user_id: int, item_id: str):
    return {"user_id": user_id, "item_id": item_id}
```

### 3단계: 쿼리 매개변수

```python
@app.get("/items/")
async def read_items(skip: int = 0, limit: int = 10):
    # GET /items/?skip=20&limit=10
    return {"skip": skip, "limit": limit}

# 필수 쿼리 매개변수
@app.get("/search/")
async def search(q: str):
    # q가 없으면 422 에러
    return {"query": q}
```

### 4단계: Request Body (Pydantic)

```python
from pydantic import BaseModel, Field, EmailStr

class User(BaseModel):
    username: str = Field(..., min_length=3, max_length=50)
    email: EmailStr
    full_name: str | None = None
    age: int = Field(..., ge=0, le=150)

@app.post("/users/")
async def create_user(user: User):
    # user는 자동 검증됨
    # 검증 실패 시 422 에러 (어떤 필드가 왜 실패했는지 포함)
    return user
```

### 5단계: Response Model

```python
class UserInDB(BaseModel):
    username: str
    email: str
    hashed_password: str

class UserOut(BaseModel):
    username: str
    email: str
    # password는 반환하지 않음

@app.post("/users/", response_model=UserOut)
async def create_user(user: User):
    # DB에 저장
    user_in_db = UserInDB(
        username=user.username,
        email=user.email,
        hashed_password=hash_password(user.password)
    )
    # UserOut만 반환 (password 제외)
    return user_in_db
```

### 6단계: 에러 처리

```python
from fastapi import HTTPException, status

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    if item_id not in items_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Item not found",
            headers={"X-Error": "Custom header"}
        )
    return items_db[item_id]
```

---

## 실전 패턴 10가지

### 패턴 1: 라우터 분리

```python
# routers/users.py
from fastapi import APIRouter

router = APIRouter(
    prefix="/users",
    tags=["users"],
    responses={404: {"description": "Not found"}}
)

@router.get("/")
async def list_users():
    return [{"username": "user1"}]

@router.get("/{user_id}")
async def get_user(user_id: int):
    return {"user_id": user_id}

# main.py
from routers import users

app = FastAPI()
app.include_router(users.router)
```

### 패턴 2: 의존성 주입으로 DB 세션 관리

```python
from sqlalchemy.orm import Session
from fastapi import Depends

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users/")
async def read_users(db: Session = Depends(get_db)):
    users = db.query(User).all()
    return users
```

### 패턴 3: 인증 (JWT)

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = get_user(username)
    if user is None:
        raise credentials_exception
    return user

@app.get("/users/me")
async def read_users_me(current_user: User = Depends(get_current_user)):
    return current_user
```

### 패턴 4: 백그라운드 작업

```python
from fastapi import BackgroundTasks

def send_email(email: str, message: str):
    # 시간이 걸리는 작업
    time.sleep(5)
    print(f"Sending email to {email}: {message}")

@app.post("/send-notification/")
async def send_notification(
    email: str,
    background_tasks: BackgroundTasks
):
    # 응답 즉시 반환, 이메일은 백그라운드에서 전송
    background_tasks.add_task(send_email, email, "Hello!")
    return {"message": "Notification sent in background"}
```

### 패턴 5: 미들웨어

```python
import time
from fastapi import Request

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response
```

### 패턴 6: CORS 설정

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],  # 프론트엔드
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 패턴 7: 파일 업로드

```python
from fastapi import File, UploadFile

@app.post("/uploadfile/")
async def create_upload_file(file: UploadFile = File(...)):
    contents = await file.read()
    # 파일 처리
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(contents)
    }
```

### 패턴 8: 웹소켓

```python
from fastapi import WebSocket

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        await websocket.send_text(f"Echo: {data}")
```

### 패턴 9: 페이지네이션

```python
from typing import List

class PaginatedResponse(BaseModel):
    items: List[Item]
    total: int
    page: int
    size: int

@app.get("/items/", response_model=PaginatedResponse)
async def list_items(page: int = 1, size: int = 10):
    skip = (page - 1) * size
    items = db.query(Item).offset(skip).limit(size).all()
    total = db.query(Item).count()

    return PaginatedResponse(
        items=items,
        total=total,
        page=page,
        size=size
    )
```

### 패턴 10: 의존성 캐싱

```python
from functools import lru_cache

@lru_cache()
def get_settings():
    return Settings()

@app.get("/info")
async def info(settings: Settings = Depends(get_settings)):
    # settings는 한 번만 로드됨
    return {"app_name": settings.app_name}
```

---

## 함정 및 주의사항

### ❌ 함정 1: 동기 함수를 async로 잘못 사용

```python
# 나쁨: blocking 작업을 async로 선언
@app.get("/bad")
async def bad_endpoint():
    time.sleep(10)  # 전체 서버 블로킹!
    return {"done": True}

# 좋음: blocking 작업은 동기 함수로
@app.get("/good")
def good_endpoint():
    time.sleep(10)  # 스레드풀에서 실행됨
    return {"done": True}

# 또는 진짜 비동기로
@app.get("/best")
async def best_endpoint():
    await asyncio.sleep(10)  # 다른 요청 처리 가능
    return {"done": True}
```

### ❌ 함정 2: Pydantic 모델 재사용 실수

```python
# 나쁨: DB 모델과 API 모델 혼용
class User(BaseModel):
    username: str
    email: str
    password: str  # 절대 반환하면 안 됨!

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    user = db.get_user(user_id)
    return user  # password 노출!

# 좋음: 분리
class UserCreate(BaseModel):
    username: str
    email: str
    password: str

class UserOut(BaseModel):
    username: str
    email: str

@app.post("/users/", response_model=UserOut)
async def create_user(user: UserCreate):
    # ...
    return user_db
```

### ❌ 함정 3: 의존성에서 예외 처리 누락

```python
# 나쁨: 에러가 500으로 반환됨
async def get_current_user(token: str = Depends(oauth2_scheme)):
    user = decode_token(token)  # 실패 시 예외
    return user

# 좋음: 명시적 HTTPException
async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        user = decode_token(token)
    except InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")
    return user
```

### ❌ 함정 4: 대용량 파일 메모리 로드

```python
# 나쁨: 전체 파일을 메모리에
@app.post("/upload/")
async def upload(file: UploadFile):
    contents = await file.read()  # 메모리 부족 가능
    # ...

# 좋음: 스트리밍 처리
@app.post("/upload/")
async def upload(file: UploadFile):
    async with aiofiles.open(f"uploads/{file.filename}", "wb") as f:
        while chunk := await file.read(1024 * 1024):  # 1MB씩
            await f.write(chunk)
```

---

## 성능 최적화

### 1. 비동기 DB 드라이버 사용

```python
# asyncpg (PostgreSQL)
import asyncpg

@app.on_event("startup")
async def startup():
    app.state.pool = await asyncpg.create_pool(DATABASE_URL)

@app.get("/users/")
async def get_users():
    async with app.state.pool.acquire() as conn:
        rows = await conn.fetch("SELECT * FROM users")
    return rows
```

### 2. 응답 캐싱

```python
from fastapi_cache import FastAPICache
from fastapi_cache.decorator import cache

@app.get("/expensive-computation")
@cache(expire=3600)  # 1시간 캐싱
async def expensive():
    result = heavy_computation()
    return result
```

### 3. 연결 풀링

```python
# httpx 연결 재사용
from httpx import AsyncClient

@app.on_event("startup")
async def startup():
    app.state.http_client = AsyncClient()

@app.on_event("shutdown")
async def shutdown():
    await app.state.http_client.aclose()

@app.get("/external")
async def call_external():
    response = await app.state.http_client.get("https://api.example.com")
    return response.json()
```

### 4. Gunicorn + Uvicorn 워커

```bash
# 프로덕션 배포
gunicorn myapi.main:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000
```

---

## 다른 프레임워크와 비교

### FastAPI vs Flask

| 특징 | FastAPI | Flask |
|------|---------|-------|
| 비동기 | ✅ 네이티브 | ⚠️ 확장 필요 |
| 타입 안전 | ✅ | ❌ |
| 자동 문서 | ✅ | ❌ (수동) |
| 성능 | 🚀 매우 빠름 | 보통 |
| 학습 곡선 | 중간 | 낮음 |
| 생태계 | 빠르게 성장 | 성숙 |

**Flask를 선택할 때:**
- 간단한 도구나 내부 서비스
- 기존 Flask 생태계에 의존
- 동기 코드로 충분

### FastAPI vs Django REST Framework

| 특징 | FastAPI | DRF |
|------|---------|-----|
| 속도 | 🚀 | 느림 |
| ORM | 선택적 | Django ORM 필수 |
| Admin | ❌ | ✅ |
| 전체 스택 | ❌ | ✅ |
| 적합 용도 | API 전용 | 전체 웹 앱 |

**Django를 선택할 때:**
- Admin 패널 필요
- ORM, Auth, 전체 솔루션 원함
- 큰 팀, 표준화된 구조 선호

---

## 프로덕션 체크리스트

### 필수 사항

- [ ] 환경 변수로 설정 관리 (pydantic-settings)
- [ ] 구조화 로깅 (structlog)
- [ ] 에러 핸들링 (전역 exception handler)
- [ ] 인증/인가 (JWT or OAuth2)
- [ ] Rate limiting
- [ ] CORS 설정
- [ ] 데이터베이스 마이그레이션
- [ ] 테스트 커버리지 80%+
- [ ] API 문서 완성 (description, tags)
- [ ] Healthcheck 엔드포인트
- [ ] 모니터링 (Prometheus + Grafana)
- [ ] Sentry 통합 (에러 추적)

### 배포

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["gunicorn", "myapi.main:app", "--workers", "4", "--worker-class", "uvicorn.workers.UvicornWorker", "--bind", "0.0.0.0:8000"]
```

---

## 학습 리소스

### 공식 문서
- **튜토리얼**: https://fastapi.tiangolo.com/tutorial/
- **고급 가이드**: https://fastapi.tiangolo.com/advanced/

### 추천 강의
- FastAPI 공식 문서 (무료, 최고)
- Real Python FastAPI 튜토리얼

### 커뮤니티
- Discord: https://discord.gg/fastapi
- GitHub Discussions

---

**[← 웹 개발 카탈로그](../../04-library-catalog/web-development/README.md)** | **[다음: Pydantic →](pydantic.md)**
