# SQLAlchemy 완벽 가이드

> **파이썬 ORM의 표준**


---

## 개요

| 항목 | 내용 |
|------|------|
| **공식 사이트** | https://www.sqlalchemy.org |
| **GitHub** | https://github.com/sqlalchemy/sqlalchemy |
| **현재 버전** | 2.0+ |

### 한 줄 요약

**파이썬 객체와 데이터베이스를 연결하는 강력한 ORM 및 SQL 툴킷**

---

## 설치

```bash
$ uv add sqlalchemy

# PostgreSQL
$ uv add 'sqlalchemy[postgresql]' asyncpg

# MySQL
$ uv add 'sqlalchemy[mysql]' aiomysql
```

---

## 기본 사용법

### 모델 정의

```python
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey
from sqlalchemy.orm import declarative_base, relationship, Session

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    username = Column(String(50), unique=True, nullable=False)
    email = Column(String(100), unique=True)

    posts = relationship('Post', back_populates='author')

class Post(Base):
    __tablename__ = 'posts'

    id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)
    content = Column(String)
    author_id = Column(Integer, ForeignKey('users.id'))

    author = relationship('User', back_populates='posts')

# 데이터베이스 생성
engine = create_engine('sqlite:///example.db')
Base.metadata.create_all(engine)
```

### CRUD 작업

```python
from sqlalchemy.orm import Session

# 생성
with Session(engine) as session:
    user = User(username='alice', email='alice@example.com')
    session.add(user)
    session.commit()

# 조회
with Session(engine) as session:
    user = session.query(User).filter_by(username='alice').first()
    all_users = session.query(User).all()

# 업데이트
with Session(engine) as session:
    user = session.query(User).filter_by(username='alice').first()
    user.email = 'newemail@example.com'
    session.commit()

# 삭제
with Session(engine) as session:
    user = session.query(User).filter_by(username='alice').first()
    session.delete(user)
    session.commit()
```

---

## SQLAlchemy 2.0 스타일

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

with Session(engine) as session:
    # 2.0 스타일
    stmt = select(User).where(User.username == 'alice')
    user = session.execute(stmt).scalar_one()

    # JOIN
    stmt = (
        select(User, Post)
        .join(Post.author)
        .where(User.username == 'alice')
    )
    results = session.execute(stmt).all()
```

---

## 비동기 SQLAlchemy

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy import select

# 비동기 엔진
engine = create_async_engine('postgresql+asyncpg://user:pass@localhost/db')

async def get_users():
    async with AsyncSession(engine) as session:
        stmt = select(User)
        result = await session.execute(stmt)
        return result.scalars().all()
```

---

## FastAPI 통합

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

engine = create_async_engine('postgresql+asyncpg://...')
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

app = FastAPI()

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session

@app.get("/users/")
async def list_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    return result.scalars().all()
```

---

**[← 데이터베이스](../../04-library-catalog/database/README.md)**
