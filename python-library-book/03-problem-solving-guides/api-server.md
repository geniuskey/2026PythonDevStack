# API 서버 만들기

## 목표

RESTful API 서버를 처음부터 프로덕션 준비 상태까지 구축하기 - 인증, 데이터베이스, 캐싱, 모니터링을 포함한 완전한 백엔드 시스템

---

## API 서버 아키텍처

```mermaid
graph TD
    A[클라이언트] --> B[NGINX/Load Balancer]
    B --> C[FastAPI 서버]

    C --> D[인증 미들웨어]
    D --> E[Rate Limiting]
    E --> F[라우터]

    F --> G[비즈니스 로직]
    G --> H[(PostgreSQL)]
    G --> I[(Redis 캐시)]
    G --> J[백그라운드 작업]

    C --> K[로깅]
    C --> L[모니터링]

    style C fill:#4CAF50
    style H fill:#2196F3
    style I fill:#FF9800
```

---

## 기술 스택

| 분야 | 도구 | 이유 | 대안 |
|------|------|------|------|
| **웹 프레임워크** | FastAPI | 자동 문서화, 타입 안전, 비동기 | Django, Flask |
| **데이터 검증** | Pydantic | 타입 안전, 자동 검증 | marshmallow |
| **서버** | Uvicorn | ASGI 표준, 고성능 | Gunicorn + Uvicorn |
| **데이터베이스** | PostgreSQL | 강력한 RDBMS, JSON 지원 | MySQL, SQLite |
| **ORM** | SQLAlchemy 2.0 | 비동기 지원, 타입 힌트 | Tortoise ORM |
| **캐시** | Redis | 인메모리 캐시, 세션 저장 | Memcached |
| **인증** | JWT | Stateless, 확장 가능 | Session |
| **테스트** | pytest | 표준 프레임워크, 플러그인 풍부 | unittest |
| **로깅** | structlog | 구조화 로깅, JSON 출력 | logging |
| **모니터링** | Prometheus + Grafana | 메트릭 수집, 시각화 | Datadog |

---

## PART 1: 프로젝트 설정

### 1. 프로젝트 초기화

```bash
# 프로젝트 생성
$ uv init myapi
$ cd myapi

# 핵심 의존성
$ uv add fastapi "uvicorn[standard]" pydantic-settings

# 데이터베이스
$ uv add "sqlalchemy[asyncio]" asyncpg alembic

# 캐싱 & 작업 큐
$ uv add redis aioredis

# 인증
$ uv add "python-jose[cryptography]" "passlib[bcrypt]" python-multipart

# HTTP 클라이언트
$ uv add httpx

# 로깅
$ uv add structlog

# 모니터링
$ uv add prometheus-client

# 개발 도구
$ uv add --dev pytest pytest-asyncio httpx ruff mypy
```

### 2. 디렉토리 구조

```
myapi/
├── src/
│   └── myapi/
│       ├── __init__.py
│       ├── main.py              # 애플리케이션 진입점
│       ├── config.py            # 설정
│       ├── database.py          # DB 연결
│       ├── models/              # SQLAlchemy 모델
│       │   ├── __init__.py
│       │   ├── user.py
│       │   └── item.py
│       ├── schemas/             # Pydantic 스키마
│       │   ├── __init__.py
│       │   ├── user.py
│       │   └── item.py
│       ├── routers/             # API 엔드포인트
│       │   ├── __init__.py
│       │   ├── auth.py
│       │   ├── users.py
│       │   └── items.py
│       ├── services/            # 비즈니스 로직
│       │   ├── __init__.py
│       │   ├── auth_service.py
│       │   └── user_service.py
│       ├── dependencies.py      # 의존성 주입
│       ├── middleware.py        # 커스텀 미들웨어
│       └── utils/
│           ├── security.py      # 보안 유틸리티
│           └── cache.py         # 캐시 유틸리티
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_auth.py
│   └── test_users.py
├── alembic/                     # DB 마이그레이션
│   ├── versions/
│   └── env.py
├── .env                         # 환경 변수
├── .env.example
├── alembic.ini
├── Dockerfile
├── docker-compose.yml
└── pyproject.toml
```

---

## PART 2: 설정 관리

```python
# src/myapi/config.py
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    # 앱 정보
    app_name: str = "My API"
    app_version: str = "1.0.0"
    debug: bool = False
    environment: str = "development"  # development, staging, production

    # 서버
    host: str = "0.0.0.0"
    port: int = 8000

    # 데이터베이스
    database_url: str
    db_echo: bool = False
    db_pool_size: int = 5
    db_max_overflow: int = 10

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # JWT
    secret_key: str
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 7

    # CORS
    allowed_origins: list[str] = ["http://localhost:3000"]

    # Rate Limiting
    rate_limit_per_minute: int = 60

    # 로깅
    log_level: str = "INFO"

    class Config:
        env_file = ".env"
        case_sensitive = False

@lru_cache()
def get_settings() -> Settings:
    return Settings()

settings = get_settings()
```

### 환경 변수 예시

```.env
# .env.example
APP_NAME=My API
DEBUG=true
ENVIRONMENT=development

DATABASE_URL=postgresql+asyncpg://user:password@localhost/mydb
DB_ECHO=false

REDIS_URL=redis://localhost:6379/0

SECRET_KEY=your-secret-key-change-this-in-production
ACCESS_TOKEN_EXPIRE_MINUTES=30

ALLOWED_ORIGINS=["http://localhost:3000","http://localhost:5173"]
```

---

## PART 3: 데이터베이스 설정

```python
# src/myapi/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import declarative_base
from myapi.config import settings

# 비동기 엔진
engine = create_async_engine(
    settings.database_url,
    echo=settings.db_echo,
    pool_size=settings.db_pool_size,
    max_overflow=settings.db_max_overflow,
)

# 세션 팩토리
AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)

# Base 클래스
Base = declarative_base()

# 의존성 주입용 세션
async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
        finally:
            await session.close()
```

### SQLAlchemy 모델

```python
# src/myapi/models/user.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.orm import relationship
from datetime import datetime
from myapi.database import Base

class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    username = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
    is_active = Column(Boolean, default=True)
    is_superuser = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # 관계
    items = relationship("Item", back_populates="owner")

# src/myapi/models/item.py
from sqlalchemy import Column, Integer, String, Float, ForeignKey, Text
from sqlalchemy.orm import relationship
from myapi.database import Base

class Item(Base):
    __tablename__ = "items"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, index=True, nullable=False)
    description = Column(Text, nullable=True)
    price = Column(Float, nullable=False)
    owner_id = Column(Integer, ForeignKey("users.id"))

    # 관계
    owner = relationship("User", back_populates="items")
```

### Alembic 마이그레이션

```bash
# Alembic 초기화
$ uv run alembic init alembic

# 마이그레이션 생성
$ uv run alembic revision --autogenerate -m "Create users and items tables"

# 마이그레이션 실행
$ uv run alembic upgrade head

# 롤백
$ uv run alembic downgrade -1
```

```python
# alembic/env.py
from myapi.database import Base
from myapi.models.user import User
from myapi.models.item import Item

target_metadata = Base.metadata
```

---

## PART 4: Pydantic 스키마

```python
# src/myapi/schemas/user.py
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime

class UserBase(BaseModel):
    email: EmailStr
    username: str = Field(..., min_length=3, max_length=50)

class UserCreate(UserBase):
    password: str = Field(..., min_length=8, max_length=100)

class UserUpdate(BaseModel):
    email: EmailStr | None = None
    username: str | None = None

class UserResponse(UserBase):
    id: int
    is_active: bool
    is_superuser: bool
    created_at: datetime

    class Config:
        from_attributes = True

class UserLogin(BaseModel):
    username: str
    password: str

class Token(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"

class TokenData(BaseModel):
    username: str | None = None

# src/myapi/schemas/item.py
from pydantic import BaseModel, Field

class ItemBase(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    description: str | None = None
    price: float = Field(..., gt=0)

class ItemCreate(ItemBase):
    pass

class ItemUpdate(BaseModel):
    title: str | None = None
    description: str | None = None
    price: float | None = Field(None, gt=0)

class ItemResponse(ItemBase):
    id: int
    owner_id: int

    class Config:
        from_attributes = True
```

---

## PART 5: 인증 시스템

```mermaid
graph LR
    A[로그인 요청] --> B[비밀번호 검증]
    B --> C{유효?}
    C -->|Yes| D[JWT 토큰 생성]
    C -->|No| E[401 Unauthorized]

    D --> F[Access Token]
    D --> G[Refresh Token]

    H[API 요청] --> I[토큰 검증]
    I --> J{유효?}
    J -->|Yes| K[요청 처리]
    J -->|No| E

    style D fill:#4CAF50
    style K fill:#4CAF50
    style E fill:#F44336
```

### 보안 유틸리티

```python
# src/myapi/utils/security.py
from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta
from myapi.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """비밀번호 검증"""
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    """비밀번호 해싱"""
    return pwd_context.hash(password)

def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    """Access Token 생성"""
    to_encode = data.copy()

    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=settings.access_token_expire_minutes)

    to_encode.update({"exp": expire, "type": "access"})
    encoded_jwt = jwt.encode(to_encode, settings.secret_key, algorithm=settings.algorithm)
    return encoded_jwt

def create_refresh_token(data: dict) -> str:
    """Refresh Token 생성"""
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(days=settings.refresh_token_expire_days)

    to_encode.update({"exp": expire, "type": "refresh"})
    encoded_jwt = jwt.encode(to_encode, settings.secret_key, algorithm=settings.algorithm)
    return encoded_jwt

def decode_token(token: str) -> dict:
    """토큰 디코딩"""
    try:
        payload = jwt.decode(token, settings.secret_key, algorithms=[settings.algorithm])
        return payload
    except JWTError:
        return None
```

### 인증 서비스

```python
# src/myapi/services/auth_service.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from fastapi import HTTPException, status

from myapi.models.user import User
from myapi.schemas.user import UserCreate, Token
from myapi.utils.security import (
    verify_password,
    get_password_hash,
    create_access_token,
    create_refresh_token,
)

class AuthService:
    @staticmethod
    async def authenticate_user(
        db: AsyncSession,
        username: str,
        password: str
    ) -> User | None:
        """사용자 인증"""
        result = await db.execute(
            select(User).where(User.username == username)
        )
        user = result.scalar_one_or_none()

        if not user:
            return None
        if not verify_password(password, user.hashed_password):
            return None

        return user

    @staticmethod
    async def create_user(db: AsyncSession, user_create: UserCreate) -> User:
        """사용자 생성"""
        # 이메일 중복 체크
        result = await db.execute(
            select(User).where(User.email == user_create.email)
        )
        if result.scalar_one_or_none():
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="Email already registered"
            )

        # 사용자명 중복 체크
        result = await db.execute(
            select(User).where(User.username == user_create.username)
        )
        if result.scalar_one_or_none():
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="Username already taken"
            )

        # 사용자 생성
        hashed_password = get_password_hash(user_create.password)
        db_user = User(
            email=user_create.email,
            username=user_create.username,
            hashed_password=hashed_password,
        )
        db.add(db_user)
        await db.commit()
        await db.refresh(db_user)

        return db_user

    @staticmethod
    def create_tokens(user: User) -> Token:
        """토큰 생성"""
        access_token = create_access_token(data={"sub": user.username})
        refresh_token = create_refresh_token(data={"sub": user.username})

        return Token(
            access_token=access_token,
            refresh_token=refresh_token,
        )
```

### 의존성: 현재 사용자

```python
# src/myapi/dependencies.py
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from myapi.database import get_db
from myapi.models.user import User
from myapi.utils.security import decode_token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    """현재 인증된 사용자 조회"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    payload = decode_token(token)
    if payload is None:
        raise credentials_exception

    username: str = payload.get("sub")
    token_type: str = payload.get("type")

    if username is None or token_type != "access":
        raise credentials_exception

    result = await db.execute(
        select(User).where(User.username == username)
    )
    user = result.scalar_one_or_none()

    if user is None:
        raise credentials_exception

    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Inactive user"
        )

    return user

async def get_current_active_superuser(
    current_user: User = Depends(get_current_user),
) -> User:
    """슈퍼유저 권한 체크"""
    if not current_user.is_superuser:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not enough permissions"
        )
    return current_user
```

---

## PART 6: 라우터 구현

### 인증 라우터

```python
# src/myapi/routers/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.ext.asyncio import AsyncSession

from myapi.database import get_db
from myapi.schemas.user import UserCreate, UserResponse, Token
from myapi.services.auth_service import AuthService

router = APIRouter()

@router.post("/register", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def register(
    user_create: UserCreate,
    db: AsyncSession = Depends(get_db),
):
    """사용자 등록"""
    user = await AuthService.create_user(db, user_create)
    return user

@router.post("/login", response_model=Token)
async def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db),
):
    """로그인"""
    user = await AuthService.authenticate_user(db, form_data.username, form_data.password)

    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )

    tokens = AuthService.create_tokens(user)
    return tokens

@router.post("/refresh", response_model=Token)
async def refresh_token(
    refresh_token: str,
    db: AsyncSession = Depends(get_db),
):
    """토큰 갱신"""
    from myapi.utils.security import decode_token

    payload = decode_token(refresh_token)

    if payload is None or payload.get("type") != "refresh":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid refresh token",
        )

    username = payload.get("sub")

    from sqlalchemy import select
    from myapi.models.user import User

    result = await db.execute(select(User).where(User.username == username))
    user = result.scalar_one_or_none()

    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found",
        )

    tokens = AuthService.create_tokens(user)
    return tokens
```

### 사용자 라우터

```python
# src/myapi/routers/users.py
from fastapi import APIRouter, Depends, HTTPException, status, Query
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from myapi.database import get_db
from myapi.models.user import User
from myapi.schemas.user import UserResponse, UserUpdate
from myapi.dependencies import get_current_user, get_current_active_superuser

router = APIRouter()

@router.get("/me", response_model=UserResponse)
async def get_current_user_info(
    current_user: User = Depends(get_current_user),
):
    """현재 사용자 정보"""
    return current_user

@router.put("/me", response_model=UserResponse)
async def update_current_user(
    user_update: UserUpdate,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    """현재 사용자 정보 수정"""
    update_data = user_update.model_dump(exclude_unset=True)

    for field, value in update_data.items():
        setattr(current_user, field, value)

    await db.commit()
    await db.refresh(current_user)

    return current_user

@router.get("/", response_model=list[UserResponse])
async def list_users(
    skip: int = Query(0, ge=0),
    limit: int = Query(10, ge=1, le=100),
    current_user: User = Depends(get_current_active_superuser),
    db: AsyncSession = Depends(get_db),
):
    """사용자 목록 (관리자 전용)"""
    result = await db.execute(
        select(User).offset(skip).limit(limit)
    )
    users = result.scalars().all()
    return users

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    """특정 사용자 조회"""
    result = await db.execute(
        select(User).where(User.id == user_id)
    )
    user = result.scalar_one_or_none()

    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="User not found"
        )

    return user
```

### Items 라우터

```python
# src/myapi/routers/items.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from myapi.database import get_db
from myapi.models.user import User
from myapi.models.item import Item
from myapi.schemas.item import ItemCreate, ItemUpdate, ItemResponse
from myapi.dependencies import get_current_user

router = APIRouter()

@router.post("/", response_model=ItemResponse, status_code=status.HTTP_201_CREATED)
async def create_item(
    item_create: ItemCreate,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    """아이템 생성"""
    db_item = Item(**item_create.model_dump(), owner_id=current_user.id)
    db.add(db_item)
    await db.commit()
    await db.refresh(db_item)
    return db_item

@router.get("/", response_model=list[ItemResponse])
async def list_items(
    skip: int = 0,
    limit: int = 10,
    db: AsyncSession = Depends(get_db),
):
    """아이템 목록"""
    result = await db.execute(
        select(Item).offset(skip).limit(limit)
    )
    items = result.scalars().all()
    return items

@router.get("/{item_id}", response_model=ItemResponse)
async def get_item(
    item_id: int,
    db: AsyncSession = Depends(get_db),
):
    """아이템 조회"""
    result = await db.execute(
        select(Item).where(Item.id == item_id)
    )
    item = result.scalar_one_or_none()

    if not item:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Item not found"
        )

    return item

@router.put("/{item_id}", response_model=ItemResponse)
async def update_item(
    item_id: int,
    item_update: ItemUpdate,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    """아이템 수정"""
    result = await db.execute(
        select(Item).where(Item.id == item_id)
    )
    item = result.scalar_one_or_none()

    if not item:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Item not found"
        )

    # 소유자 확인
    if item.owner_id != current_user.id and not current_user.is_superuser:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not enough permissions"
        )

    update_data = item_update.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(item, field, value)

    await db.commit()
    await db.refresh(item)

    return item

@router.delete("/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_item(
    item_id: int,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    """아이템 삭제"""
    result = await db.execute(
        select(Item).where(Item.id == item_id)
    )
    item = result.scalar_one_or_none()

    if not item:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Item not found"
        )

    # 소유자 확인
    if item.owner_id != current_user.id and not current_user.is_superuser:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Not enough permissions"
        )

    await db.delete(item)
    await db.commit()
```

---

## PART 7: 메인 애플리케이션

```python
# src/myapi/main.py
from fastapi import FastAPI, Request, status
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from contextlib import asynccontextmanager
import structlog
import time

from myapi.config import settings
from myapi.database import engine
from myapi.routers import auth, users, items

# 로깅 설정
structlog.configure(
    processors=[
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.dev.ConsoleRenderer() if settings.debug else structlog.processors.JSONRenderer(),
    ]
)

logger = structlog.get_logger()

# 라이프사이클
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    logger.info("application_starting", version=settings.app_version)
    yield
    # Shutdown
    await engine.dispose()
    logger.info("application_shutdown")

# 앱 생성
app = FastAPI(
    title=settings.app_name,
    version=settings.app_version,
    docs_url="/docs" if settings.debug else None,
    redoc_url="/redoc" if settings.debug else None,
    lifespan=lifespan,
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 요청 로깅 미들웨어
@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()

    logger.info(
        "request_started",
        method=request.method,
        path=request.url.path,
    )

    response = await call_next(request)

    process_time = time.time() - start_time
    logger.info(
        "request_completed",
        method=request.method,
        path=request.url.path,
        status_code=response.status_code,
        process_time=round(process_time, 3),
    )

    response.headers["X-Process-Time"] = str(process_time)
    return response

# 전역 예외 핸들러
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.error(
        "unhandled_exception",
        exc_type=type(exc).__name__,
        exc_msg=str(exc),
        path=request.url.path,
    )

    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={"detail": "Internal server error"},
    )

# 라우터 등록
app.include_router(auth.router, prefix="/auth", tags=["authentication"])
app.include_router(users.router, prefix="/users", tags=["users"])
app.include_router(items.router, prefix="/items", tags=["items"])

# 헬스체크
@app.get("/health")
async def health_check():
    """헬스 체크"""
    return {
        "status": "ok",
        "app": settings.app_name,
        "version": settings.app_version,
        "environment": settings.environment,
    }

@app.get("/")
async def root():
    """루트 엔드포인트"""
    return {
        "message": "Welcome to My API",
        "docs": "/docs",
        "health": "/health",
    }
```

---

## PART 8: 캐싱

```python
# src/myapi/utils/cache.py
import redis.asyncio as redis
from myapi.config import settings
import json
from typing import Any

class CacheService:
    def __init__(self):
        self.redis = redis.from_url(settings.redis_url, decode_responses=True)

    async def get(self, key: str) -> Any | None:
        """캐시 조회"""
        value = await self.redis.get(key)
        if value:
            return json.loads(value)
        return None

    async def set(self, key: str, value: Any, expire: int = 300):
        """캐시 저장 (기본 5분)"""
        await self.redis.set(key, json.dumps(value), ex=expire)

    async def delete(self, key: str):
        """캐시 삭제"""
        await self.redis.delete(key)

    async def close(self):
        """연결 종료"""
        await self.redis.close()

cache_service = CacheService()

# 캐싱 데코레이터
from functools import wraps

def cached(expire: int = 300):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # 캐시 키 생성
            cache_key = f"{func.__name__}:{str(args)}:{str(kwargs)}"

            # 캐시 조회
            cached_result = await cache_service.get(cache_key)
            if cached_result is not None:
                logger.info("cache_hit", key=cache_key)
                return cached_result

            # 함수 실행
            result = await func(*args, **kwargs)

            # 캐시 저장
            await cache_service.set(cache_key, result, expire)
            logger.info("cache_miss", key=cache_key)

            return result
        return wrapper
    return decorator

# 사용 예시
@router.get("/items/cached", response_model=list[ItemResponse])
@cached(expire=60)
async def list_items_cached(
    skip: int = 0,
    limit: int = 10,
    db: AsyncSession = Depends(get_db),
):
    """캐시된 아이템 목록"""
    result = await db.execute(
        select(Item).offset(skip).limit(limit)
    )
    items = result.scalars().all()
    return items
```

---

## PART 9: 테스트

```python
# tests/conftest.py
import pytest
import pytest_asyncio
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker

from myapi.main import app
from myapi.database import get_db, Base
from myapi.config import settings

# 테스트 DB
TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"

@pytest_asyncio.fixture
async def test_db():
    """테스트 데이터베이스"""
    engine = create_async_engine(TEST_DATABASE_URL, echo=False)

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    TestSessionLocal = async_sessionmaker(
        engine,
        class_=AsyncSession,
        expire_on_commit=False,
    )

    async with TestSessionLocal() as session:
        yield session

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

    await engine.dispose()

@pytest_asyncio.fixture
async def client(test_db):
    """테스트 클라이언트"""
    async def override_get_db():
        yield test_db

    app.dependency_overrides[get_db] = override_get_db

    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

    app.dependency_overrides.clear()

# tests/test_auth.py
@pytest.mark.asyncio
async def test_register(client):
    """회원가입 테스트"""
    response = await client.post(
        "/auth/register",
        json={
            "email": "test@example.com",
            "username": "testuser",
            "password": "password123"
        }
    )
    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "test@example.com"
    assert data["username"] == "testuser"
    assert "id" in data

@pytest.mark.asyncio
async def test_login(client):
    """로그인 테스트"""
    # 회원가입
    await client.post(
        "/auth/register",
        json={
            "email": "test@example.com",
            "username": "testuser",
            "password": "password123"
        }
    )

    # 로그인
    response = await client.post(
        "/auth/login",
        data={
            "username": "testuser",
            "password": "password123"
        }
    )
    assert response.status_code == 200
    data = response.json()
    assert "access_token" in data
    assert "refresh_token" in data
    assert data["token_type"] == "bearer"
```

---

## PART 10: 프로덕션 배포

### Docker

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app

# uv 설치
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# 의존성 복사 및 설치
COPY pyproject.toml .
RUN uv sync --frozen --no-dev

# 소스 복사
COPY src/ ./src/
COPY alembic/ ./alembic/
COPY alembic.ini .

# 포트 노출
EXPOSE 8000

# 실행
CMD ["uv", "run", "uvicorn", "myapi.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://user:password@postgres/mydb
      - REDIS_URL=redis://redis:6379/0
      - SECRET_KEY=${SECRET_KEY}
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api

volumes:
  postgres_data:
```

### NGINX 설정

```nginx
# nginx.conf
upstream api {
    server api:8000;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 프로덕션 체크리스트

### 보안
- [x] JWT 토큰 인증
- [x] 비밀번호 해싱 (bcrypt)
- [x] HTTPS (NGINX + Let's Encrypt)
- [x] CORS 설정
- [ ] Rate limiting (slowapi)
- [ ] SQL Injection 방어 (SQLAlchemy ORM 사용)
- [ ] XSS 방어 (Pydantic 검증)

### 성능
- [x] 비동기 데이터베이스 (asyncpg)
- [x] Redis 캐싱
- [x] 연결 풀링
- [ ] CDN (정적 파일)
- [ ] 데이터베이스 인덱싱

### 모니터링
- [x] 구조화 로깅 (structlog)
- [ ] 메트릭 (Prometheus)
- [ ] 에러 트래킹 (Sentry)
- [x] 헬스체크 엔드포인트

### 인프라
- [x] Docker 컨테이너화
- [x] Docker Compose
- [x] 환경 변수 관리
- [ ] CI/CD (GitHub Actions)
- [ ] 자동 백업

---

## 요약

이 가이드로 구축한 API 서버:

✅ **완전한 인증 시스템**: JWT + Refresh Token
✅ **비동기 데이터베이스**: SQLAlchemy 2.0 + PostgreSQL
✅ **캐싱**: Redis로 성능 최적화
✅ **타입 안전**: Pydantic + mypy
✅ **테스트**: pytest로 자동화된 테스트
✅ **프로덕션 준비**: Docker + NGINX + 모니터링

---

**[← LLM 앱](../02-modern-stack-2026/llm-apps.md)** | **[패키지 배포 →](package-deployment.md)**
