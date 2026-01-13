# 웹 API 개발

## 2026 표준 스택

| 도구 | 등급 | 역할 |
|------|------|------|
| `FastAPI` | ⭐ | 현대적 웹 프레임워크 |
| `pydantic` | ⭐ | 데이터 검증 & 직렬화 |
| `uvicorn` | ⭐ | ASGI 서버 |
| `httpx` | ⭐ | HTTP 클라이언트 (테스트용) |

---

## 1. FastAPI ⭐

### 핵심 특징

- **자동 문서화**: Swagger UI 자동 생성
- **타입 안전**: Pydantic 통합
- **비동기 기본**: async/await 네이티브
- **빠름**: Starlette 기반, Node.js 수준 성능

### 최소 예제

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}

@app.get("/items/{item_id}")
async def read_item(item_id: int, q: str = None):
    return {"item_id": item_id, "q": q}
```

### 실행

```bash
$ uv add fastapi "uvicorn[standard]"
$ uvicorn main:app --reload

# 자동 문서: http://localhost:8000/docs
```

---

## 2. pydantic: 데이터 검증 ⭐

### 기본 모델

```python
from pydantic import BaseModel, EmailStr, Field

class User(BaseModel):
    id: int
    name: str = Field(..., min_length=1, max_length=100)
    email: EmailStr
    age: int = Field(..., ge=0, le=150)
    is_active: bool = True

# 자동 검증
user = User(
    id=1,
    name="Alice",
    email="alice@example.com",
    age=30
)

# JSON 변환
user.model_dump()
user.model_dump_json()
```

### FastAPI 통합

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class CreateUserRequest(BaseModel):
    name: str
    email: str

class UserResponse(BaseModel):
    id: int
    name: str
    email: str

@app.post("/users/", response_model=UserResponse)
async def create_user(user: CreateUserRequest):
    # 자동 검증, 자동 문서화, 자동 타입 체크
    new_user = UserResponse(
        id=1,
        name=user.name,
        email=user.email
    )
    return new_user
```

---

## 3. 실전 구조

```python
# app/main.py
from fastapi import FastAPI
from app.routers import users, items

app = FastAPI(
    title="My API",
    version="1.0.0",
    docs_url="/docs",
)

app.include_router(users.router, prefix="/users", tags=["users"])
app.include_router(items.router, prefix="/items", tags=["items"])

# app/routers/users.py
from fastapi import APIRouter, HTTPException
from app.models import User
from app.schemas import UserCreate, UserResponse

router = APIRouter()

@router.post("/", response_model=UserResponse)
async def create_user(user: UserCreate):
    # 비즈니스 로직
    return UserResponse(...)

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(user_id: int):
    # 비즈니스 로직
    return UserResponse(...)
```

---

## 4. 의존성 주입

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    # 토큰 검증 로직
    user = verify_token(token)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials"
        )
    return user

@app.get("/users/me")
async def read_users_me(current_user: User = Depends(get_current_user)):
    return current_user
```

---

## 5. 테스트

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_create_user():
    response = client.post(
        "/users/",
        json={"name": "Alice", "email": "alice@example.com"}
    )
    assert response.status_code == 200
    assert response.json()["name"] == "Alice"

@pytest.mark.asyncio
async def test_async_endpoint():
    async with httpx.AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/users/1")
    assert response.status_code == 200
```

---

## 대체 도구들

### Flask ⚠️

- **이유**: FastAPI가 더 현대적 (async, 타입, 문서)
- **언제 OK**: 간단한 도구, 레거시 프로젝트

### Django (REST Framework) ⚠️

- **이유**: API 전용이면 FastAPI가 더 가벼움
- **언제 OK**: 전체 웹 앱, admin, ORM 필요

---

**[← 테스트](testing.md)** | **[비동기 I/O →](async-io.md)**
