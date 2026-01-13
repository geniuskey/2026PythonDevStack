# API 서버 만들기

## 목표

RESTful API 서버를 처음부터 프로덕션 준비 상태까지 구축하기

---

## 기술 스택

| 분야 | 도구 | 이유 |
|------|------|------|
| 웹 프레임워크 | FastAPI | 자동 문서화, 타입 안전, 비동기 |
| 데이터 검증 | Pydantic | 타입 안전, 자동 검증 |
| 서버 | Uvicorn | ASGI 표준, 고성능 |
| HTTP 클라이언트 | httpx | 비동기, 테스트용 |
| 테스트 | pytest | 표준 프레임워크 |
| 로깅 | structlog | 구조화 로깅 |
| 환경 설정 | pydantic-settings | 타입 안전 설정 |

---

## 1단계: 프로젝트 설정

```bash
# 프로젝트 생성
$ uv init myapi
$ cd myapi

# 의존성 추가
$ uv add fastapi "uvicorn[standard]" pydantic-settings structlog

# 개발 의존성
$ uv add --dev pytest pytest-asyncio httpx ruff mypy
```

### 디렉토리 구조

```
myapi/
├── src/
│   └── myapi/
│       ├── __init__.py
│       ├── main.py           # 애플리케이션 진입점
│       ├── config.py         # 설정
│       ├── models.py         # Pydantic 모델
│       ├── routers/          # 엔드포인트
│       │   ├── __init__.py
│       │   ├── users.py
│       │   └── items.py
│       ├── services/         # 비즈니스 로직
│       │   └── __init__.py
│       └── dependencies.py   # 의존성 주입
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   └── test_api.py
└── pyproject.toml
```

---

## 2단계: 설정 관리

```python
# src/myapi/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_name: str = "My API"
    app_version: str = "1.0.0"
    debug: bool = False

    # 데이터베이스
    database_url: str = "sqlite:///./test.db"

    # API 키
    api_key_header: str = "X-API-Key"

    # CORS
    allowed_origins: list[str] = ["http://localhost:3000"]

    class Config:
        env_file = ".env"

settings = Settings()
```

---

## 3단계: 데이터 모델

```python
# src/myapi/models.py
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime

class UserBase(BaseModel):
    email: EmailStr
    name: str = Field(..., min_length=1, max_length=100)

class UserCreate(UserBase):
    password: str = Field(..., min_length=8)

class UserResponse(UserBase):
    id: int
    created_at: datetime
    is_active: bool = True

    class Config:
        from_attributes = True

class ItemBase(BaseModel):
    title: str = Field(..., min_length=1)
    description: str | None = None
    price: float = Field(..., gt=0)

class ItemCreate(ItemBase):
    pass

class ItemResponse(ItemBase):
    id: int
    owner_id: int

    class Config:
        from_attributes = True
```

---

## 4단계: 라우터 구현

```python
# src/myapi/routers/users.py
from fastapi import APIRouter, HTTPException, status
from myapi.models import UserCreate, UserResponse

router = APIRouter()

# 임시 데이터 (실제로는 DB 사용)
fake_users_db: dict[int, dict] = {}
user_id_counter = 1

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(user: UserCreate):
    """새 사용자 생성"""
    global user_id_counter

    # 이메일 중복 체크
    for existing_user in fake_users_db.values():
        if existing_user["email"] == user.email:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="Email already registered"
            )

    # 사용자 생성
    user_dict = user.model_dump()
    user_dict.pop("password")  # 비밀번호는 저장하지 않음 (해싱 필요)
    user_dict["id"] = user_id_counter
    user_dict["created_at"] = datetime.now()
    user_dict["is_active"] = True

    fake_users_db[user_id_counter] = user_dict
    user_id_counter += 1

    return UserResponse(**user_dict)

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(user_id: int):
    """사용자 조회"""
    if user_id not in fake_users_db:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )
    return UserResponse(**fake_users_db[user_id])

@router.get("/", response_model=list[UserResponse])
async def list_users(skip: int = 0, limit: int = 10):
    """사용자 목록"""
    users = list(fake_users_db.values())[skip:skip+limit]
    return [UserResponse(**user) for user in users]
```

---

## 5단계: 메인 애플리케이션

```python
# src/myapi/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import structlog

from myapi.config import settings
from myapi.routers import users, items

# 로깅 설정
structlog.configure(
    processors=[
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.dev.ConsoleRenderer() if settings.debug else structlog.processors.JSONRenderer(),
    ]
)

logger = structlog.get_logger()

# 앱 생성
app = FastAPI(
    title=settings.app_name,
    version=settings.app_version,
    docs_url="/docs",
    redoc_url="/redoc",
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 라우터 등록
app.include_router(users.router, prefix="/users", tags=["users"])
app.include_router(items.router, prefix="/items", tags=["items"])

@app.get("/")
async def root():
    """헬스 체크"""
    return {"status": "ok", "app": settings.app_name}

@app.on_event("startup")
async def startup_event():
    logger.info("application_started", version=settings.app_version)

@app.on_event("shutdown")
async def shutdown_event():
    logger.info("application_shutdown")
```

---

## 6단계: 테스트

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from myapi.main import app

@pytest.fixture
def client():
    return TestClient(app)

# tests/test_api.py
def test_root(client):
    response = client.get("/")
    assert response.status_code == 200
    assert response.json()["status"] == "ok"

def test_create_user(client):
    response = client.post(
        "/users/",
        json={
            "email": "test@example.com",
            "name": "Test User",
            "password": "password123"
        }
    )
    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "test@example.com"
    assert data["name"] == "Test User"
    assert "id" in data

def test_get_user(client):
    # 먼저 사용자 생성
    create_response = client.post(
        "/users/",
        json={
            "email": "test2@example.com",
            "name": "Test User 2",
            "password": "password123"
        }
    )
    user_id = create_response.json()["id"]

    # 조회
    response = client.get(f"/users/{user_id}")
    assert response.status_code == 200
    assert response.json()["id"] == user_id

def test_user_not_found(client):
    response = client.get("/users/999")
    assert response.status_code == 404
```

---

## 7단계: 실행

```bash
# 개발 모드 (자동 리로드)
$ uv run uvicorn myapi.main:app --reload

# 프로덕션 모드
$ uv run uvicorn myapi.main:app --host 0.0.0.0 --port 8000

# 자동 문서 확인
# http://localhost:8000/docs
# http://localhost:8000/redoc
```

---

## 추가 기능

### 1. 인증 (JWT)

```bash
$ uv add python-jose[cryptography] passlib[bcrypt]
```

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from passlib.context import CryptContext

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    # JWT 검증 로직
    pass
```

### 2. 데이터베이스 (SQLAlchemy)

```bash
$ uv add sqlalchemy alembic asyncpg
```

### 3. 캐싱 (Redis)

```bash
$ uv add redis aioredis
```

---

## 프로덕션 체크리스트

- [ ] 환경 변수로 설정 관리
- [ ] 구조화 로깅 (structlog)
- [ ] 에러 핸들링 (전역 exception handler)
- [ ] 인증/인가 (JWT)
- [ ] Rate limiting
- [ ] CORS 설정
- [ ] 데이터베이스 마이그레이션
- [ ] 테스트 커버리지 80%+
- [ ] API 문서 완성
- [ ] Healthcheck 엔드포인트
- [ ] 모니터링 (Prometheus, Sentry)

---

**[← LLM 앱](../02-modern-stack-2026/llm-apps.md)** | **[패키지 배포 →](package-deployment.md)**
