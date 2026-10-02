# Part 094 - FastAPI Database (SQLAlchemy Async)

## เป้าหมายการเรียนรู้

- ตั้งค่า Async SQLAlchemy
- จัดการ Alembic migrations
- ใช้ Repository pattern
- ทำ Async CRUD operations
- สร้าง Complete CRUD API

---

## 1. ติดตั้ง Dependencies

```bash
pip install sqlalchemy[asyncio] asyncpg aiosqlite alembic

# สำหรับ PostgreSQL: asyncpg
# สำหรับ SQLite: aiosqlite
```

---

## 2. Async Database Setup

```python
# app/database.py - Async database configuration

from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.pool import NullPool
from app.config import get_settings

settings = get_settings()

# สร้าง async engine
# SQLite async:      sqlite+aiosqlite:///./app.db
# PostgreSQL async:  postgresql+asyncpg://user:pass@host/db

engine = create_async_engine(
    settings.database_url,
    echo=settings.db_echo,        # แสดง SQL queries
    pool_size=settings.db_pool_size,
    max_overflow=settings.db_max_overflow,
    pool_pre_ping=True,           # ตรวจสอบ connection ก่อนใช้
)

# Session factory
AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False,  # ไม่ expire objects หลัง commit
    autoflush=False,
    autocommit=False,
)


# Base class สำหรับ models
class Base(DeclarativeBase):
    pass


# ─────────────────────────────────────────
# Database Dependency สำหรับ FastAPI
# ─────────────────────────────────────────

async def get_db() -> AsyncSession:
    """
    Dependency สำหรับ database session
    ใช้ใน router: db: AsyncSession = Depends(get_db)
    """
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()


# ─────────────────────────────────────────
# Startup/Shutdown
# ─────────────────────────────────────────

async def create_tables():
    """สร้างตารางทั้งหมด (สำหรับ development)"""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)


async def drop_tables():
    """ลบตารางทั้งหมด"""
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
```

---

## 3. Models

```python
# app/models/user.py

from sqlalchemy import Integer, String, Boolean, DateTime, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.sql import func
from app.database import Base
from datetime import datetime
from typing import Optional, List


class User(Base):
    __tablename__ = "users"
    
    # Primary key
    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    
    # Required fields
    username: Mapped[str] = mapped_column(String(80), unique=True, nullable=False, index=True)
    email: Mapped[str] = mapped_column(String(120), unique=True, nullable=False, index=True)
    password_hash: Mapped[str] = mapped_column(String(256), nullable=False)
    
    # Optional fields
    full_name: Mapped[Optional[str]] = mapped_column(String(100))
    bio: Mapped[Optional[str]] = mapped_column(Text)
    avatar_url: Mapped[Optional[str]] = mapped_column(String(500))
    
    # Status
    is_active: Mapped[bool] = mapped_column(Boolean, default=True, nullable=False)
    is_admin: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)
    is_verified: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)
    
    # Timestamps
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        nullable=False
    )
    updated_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime(timezone=True),
        onupdate=func.now()
    )
    last_login: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))
    
    # Relationships
    posts: Mapped[List["Post"]] = relationship("Post", back_populates="author", lazy="selectin")
    
    def __repr__(self) -> str:
        return f"<User {self.username}>"


# app/models/post.py

from sqlalchemy import Integer, String, Boolean, DateTime, Text, ForeignKey, Table, Column
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.sql import func
from app.database import Base
from datetime import datetime
from typing import Optional, List


# Association table สำหรับ Post-Tag Many-to-Many
post_tags = Table(
    "post_tags",
    Base.metadata,
    Column("post_id", Integer, ForeignKey("posts.id", ondelete="CASCADE"), primary_key=True),
    Column("tag_id", Integer, ForeignKey("tags.id", ondelete="CASCADE"), primary_key=True),
)


class Tag(Base):
    __tablename__ = "tags"
    
    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    name: Mapped[str] = mapped_column(String(50), unique=True, nullable=False, index=True)
    
    posts: Mapped[List["Post"]] = relationship(
        "Post",
        secondary=post_tags,
        back_populates="tags"
    )


class Post(Base):
    __tablename__ = "posts"
    
    id: Mapped[int] = mapped_column(Integer, primary_key=True, index=True)
    title: Mapped[str] = mapped_column(String(200), nullable=False)
    slug: Mapped[str] = mapped_column(String(220), unique=True, nullable=False, index=True)
    content: Mapped[str] = mapped_column(Text, nullable=False)
    summary: Mapped[Optional[str]] = mapped_column(String(500))
    is_published: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)
    
    # Foreign Keys
    author_id: Mapped[int] = mapped_column(Integer, ForeignKey("users.id", ondelete="CASCADE"))
    
    # Timestamps
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now()
    )
    updated_at: Mapped[Optional[datetime]] = mapped_column(
        DateTime(timezone=True),
        onupdate=func.now()
    )
    published_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))
    
    # Relationships
    author: Mapped["User"] = relationship("User", back_populates="posts", lazy="selectin")
    tags: Mapped[List[Tag]] = relationship(
        "Tag",
        secondary=post_tags,
        back_populates="posts",
        lazy="selectin"
    )
```

---

## 4. Repository Pattern

```python
# app/repositories/base.py - Generic async repository

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, func, delete, update
from typing import TypeVar, Generic, Type, Optional, List, Any, Dict
from app.database import Base

ModelType = TypeVar("ModelType", bound=Base)


class BaseRepository(Generic[ModelType]):
    """Generic repository สำหรับ async CRUD"""
    
    def __init__(self, model: Type[ModelType], db: AsyncSession):
        self.model = model
        self.db = db
    
    async def get(self, id: int) -> Optional[ModelType]:
        """ดึง record ด้วย primary key"""
        result = await self.db.execute(
            select(self.model).where(self.model.id == id)
        )
        return result.scalar_one_or_none()
    
    async def get_or_404(self, id: int) -> ModelType:
        """ดึง record หรือ raise 404"""
        from fastapi import HTTPException
        obj = await self.get(id)
        if obj is None:
            raise HTTPException(
                status_code=404,
                detail=f"{self.model.__name__} with id {id} not found"
            )
        return obj
    
    async def get_all(
        self,
        skip: int = 0,
        limit: int = 100,
        **filters
    ) -> List[ModelType]:
        """ดึงทั้งหมดพร้อม filtering"""
        query = select(self.model)
        
        # Apply filters
        for key, value in filters.items():
            if hasattr(self.model, key) and value is not None:
                query = query.where(getattr(self.model, key) == value)
        
        query = query.offset(skip).limit(limit)
        result = await self.db.execute(query)
        return result.scalars().all()
    
    async def count(self, **filters) -> int:
        """นับจำนวน records"""
        query = select(func.count(self.model.id))
        
        for key, value in filters.items():
            if hasattr(self.model, key) and value is not None:
                query = query.where(getattr(self.model, key) == value)
        
        result = await self.db.execute(query)
        return result.scalar_one()
    
    async def create(self, **data) -> ModelType:
        """สร้าง record ใหม่"""
        obj = self.model(**data)
        self.db.add(obj)
        await self.db.flush()  # ส่งไป DB แต่ยัง rollback ได้
        await self.db.refresh(obj)
        return obj
    
    async def update(self, id: int, **data) -> Optional[ModelType]:
        """อัปเดต record"""
        obj = await self.get_or_404(id)
        
        for key, value in data.items():
            if hasattr(obj, key):
                setattr(obj, key, value)
        
        await self.db.flush()
        await self.db.refresh(obj)
        return obj
    
    async def delete(self, id: int) -> bool:
        """ลบ record"""
        obj = await self.get(id)
        if obj is None:
            return False
        await self.db.delete(obj)
        await self.db.flush()
        return True
    
    async def exists(self, **filters) -> bool:
        """ตรวจสอบว่า record มีอยู่หรือไม่"""
        query = select(func.count(self.model.id))
        for key, value in filters.items():
            if hasattr(self.model, key):
                query = query.where(getattr(self.model, key) == value)
        result = await self.db.execute(query)
        return result.scalar_one() > 0
```

```python
# app/repositories/user.py - User-specific repository

from sqlalchemy import select, or_
from sqlalchemy.ext.asyncio import AsyncSession
from app.repositories.base import BaseRepository
from app.models.user import User
from typing import Optional, List
from datetime import datetime
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


class UserRepository(BaseRepository[User]):
    
    def __init__(self, db: AsyncSession):
        super().__init__(User, db)
    
    async def get_by_email(self, email: str) -> Optional[User]:
        """ดึง user ด้วย email"""
        result = await self.db.execute(
            select(User).where(User.email == email.lower())
        )
        return result.scalar_one_or_none()
    
    async def get_by_username(self, username: str) -> Optional[User]:
        """ดึง user ด้วย username"""
        result = await self.db.execute(
            select(User).where(User.username == username)
        )
        return result.scalar_one_or_none()
    
    async def get_active_users(self, skip: int = 0, limit: int = 20) -> List[User]:
        """ดึง active users"""
        result = await self.db.execute(
            select(User)
            .where(User.is_active == True)
            .offset(skip)
            .limit(limit)
            .order_by(User.created_at.desc())
        )
        return result.scalars().all()
    
    async def search(self, query: str, skip: int = 0, limit: int = 20) -> List[User]:
        """ค้นหา users"""
        search_term = f"%{query}%"
        result = await self.db.execute(
            select(User)
            .where(
                or_(
                    User.username.ilike(search_term),
                    User.email.ilike(search_term),
                    User.full_name.ilike(search_term)
                )
            )
            .offset(skip)
            .limit(limit)
        )
        return result.scalars().all()
    
    async def create_user(self, username: str, email: str, password: str, **kwargs) -> User:
        """สร้าง user ใหม่พร้อม hash password"""
        # ตรวจสอบ duplicate
        if await self.exists(email=email.lower()):
            raise ValueError(f"Email {email} มีอยู่แล้ว")
        if await self.exists(username=username):
            raise ValueError(f"Username {username} มีอยู่แล้ว")
        
        password_hash = pwd_context.hash(password)
        
        return await self.create(
            username=username,
            email=email.lower(),
            password_hash=password_hash,
            **kwargs
        )
    
    def verify_password(self, plain_password: str, password_hash: str) -> bool:
        """ตรวจสอบ password"""
        return pwd_context.verify(plain_password, password_hash)
    
    async def update_last_login(self, user_id: int):
        """อัปเดต last_login"""
        await self.update(user_id, last_login=datetime.utcnow())
```

---

## 5. Schemas (Pydantic)

```python
# app/schemas/user.py

from pydantic import BaseModel, Field, EmailStr
from typing import Optional
from datetime import datetime


class UserBase(BaseModel):
    username: str = Field(..., min_length=3, max_length=80)
    email: str


class UserCreate(UserBase):
    password: str = Field(..., min_length=8)
    full_name: Optional[str] = None


class UserUpdate(BaseModel):
    full_name: Optional[str] = None
    bio: Optional[str] = Field(None, max_length=500)
    avatar_url: Optional[str] = None


class UserPasswordUpdate(BaseModel):
    current_password: str
    new_password: str = Field(..., min_length=8)


class UserResponse(UserBase):
    id: int
    full_name: Optional[str]
    bio: Optional[str]
    avatar_url: Optional[str]
    is_active: bool
    is_admin: bool
    created_at: datetime
    
    class Config:
        from_attributes = True  # อ่านจาก ORM object


class UserListResponse(BaseModel):
    items: list[UserResponse]
    total: int
    page: int
    per_page: int
    pages: int
```

---

## 6. Router และ Complete CRUD

```python
# app/routers/users.py - Complete async CRUD router

from fastapi import APIRouter, Depends, HTTPException, Query, status
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db
from app.repositories.user import UserRepository
from app.schemas.user import UserCreate, UserUpdate, UserResponse, UserListResponse
from typing import Optional

router = APIRouter(prefix="/users", tags=["users"])


# ─────────────────────────────────────────
# Dependency helpers
# ─────────────────────────────────────────

def get_user_repo(db: AsyncSession = Depends(get_db)) -> UserRepository:
    """Dependency สำหรับ UserRepository"""
    return UserRepository(db)


# ─────────────────────────────────────────
# GET /users - รายการ users
# ─────────────────────────────────────────

@router.get("", response_model=UserListResponse)
async def list_users(
    page: int = Query(1, ge=1),
    per_page: int = Query(20, ge=1, le=100),
    search: Optional[str] = Query(None, min_length=2),
    active_only: bool = Query(True),
    repo: UserRepository = Depends(get_user_repo)
):
    """ดูรายการ users"""
    skip = (page - 1) * per_page
    
    if search:
        users = await repo.search(search, skip=skip, limit=per_page)
        total = len(users)  # approximate
    elif active_only:
        users = await repo.get_active_users(skip=skip, limit=per_page)
        total = await repo.count(is_active=True)
    else:
        users = await repo.get_all(skip=skip, limit=per_page)
        total = await repo.count()
    
    pages = (total + per_page - 1) // per_page
    
    return UserListResponse(
        items=users,
        total=total,
        page=page,
        per_page=per_page,
        pages=pages
    )


# ─────────────────────────────────────────
# GET /users/{user_id}
# ─────────────────────────────────────────

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    repo: UserRepository = Depends(get_user_repo)
):
    """ดู user เดียว"""
    return await repo.get_or_404(user_id)


# ─────────────────────────────────────────
# POST /users - สร้าง user
# ─────────────────────────────────────────

@router.post("", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    user_data: UserCreate,
    repo: UserRepository = Depends(get_user_repo)
):
    """สร้าง user ใหม่"""
    try:
        user = await repo.create_user(
            username=user_data.username,
            email=user_data.email,
            password=user_data.password,
            full_name=user_data.full_name
        )
        return user
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail=str(e)
        )


# ─────────────────────────────────────────
# PATCH /users/{user_id} - อัปเดต
# ─────────────────────────────────────────

@router.patch("/{user_id}", response_model=UserResponse)
async def update_user(
    user_id: int,
    user_data: UserUpdate,
    repo: UserRepository = Depends(get_user_repo)
):
    """อัปเดต user"""
    # exclude_unset ดึงเฉพาะ fields ที่ส่งมา
    update_data = user_data.model_dump(exclude_unset=True)
    
    if not update_data:
        raise HTTPException(status_code=400, detail="ไม่มีข้อมูลที่ต้องการอัปเดต")
    
    return await repo.update(user_id, **update_data)


# ─────────────────────────────────────────
# DELETE /users/{user_id}
# ─────────────────────────────────────────

@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_user(
    user_id: int,
    repo: UserRepository = Depends(get_user_repo)
):
    """ลบ user"""
    deleted = await repo.delete(user_id)
    if not deleted:
        raise HTTPException(status_code=404, detail="ไม่พบ user")
```

---

## 7. Alembic Migrations

```bash
# ติดตั้ง alembic
pip install alembic

# Initialize
alembic init alembic

# สร้าง migration
alembic revision --autogenerate -m "Create users and posts tables"

# Apply migrations
alembic upgrade head

# Rollback
alembic downgrade -1

# ดู history
alembic history
```

```python
# alembic/env.py - ตั้งค่า alembic

from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from sqlalchemy.ext.asyncio import AsyncEngine
from alembic import context
import asyncio

# Import models เพื่อให้ alembic รู้จัก
from app.database import Base
from app.models import user, post  # noqa: F401
from app.config import get_settings

config = context.config
settings = get_settings()

# Set DB URL จาก settings
config.set_main_option("sqlalchemy.url", settings.database_url)

target_metadata = Base.metadata


def run_migrations_offline():
    """Offline mode"""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


async def run_migrations_online():
    """Online mode"""
    from sqlalchemy.ext.asyncio import create_async_engine
    
    connectable = create_async_engine(settings.database_url)
    
    async with connectable.connect() as connection:
        await connection.run_sync(
            lambda sync_conn: context.configure(
                connection=sync_conn,
                target_metadata=target_metadata
            )
        )
        async with connection.begin():
            await connection.run_sync(lambda _: context.run_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    asyncio.run(run_migrations_online())
```

---

## 8. Main Application

```python
# app/main.py

from fastapi import FastAPI
from contextlib import asynccontextmanager
from app.database import create_tables
from app.routers import users, posts


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: สร้างตาราง (development)
    await create_tables()
    yield
    # Shutdown


app = FastAPI(title="Async FastAPI", lifespan=lifespan)

app.include_router(users.router, prefix="/api/v1")
app.include_router(posts.router, prefix="/api/v1")
```

---

## Exercises

### Exercise 1: Post CRUD
สร้าง async CRUD สำหรับ Post:
- PostRepository extends BaseRepository
- Create post พร้อม tags
- Search posts ด้วย title/content
- Filter published/draft

### Exercise 2: N+1 Problem
แก้ N+1 query problem:
- ใช้ joinedload/selectinload
- วัดเวลา before/after
- Explain query plan

---

## สรุป

สิ่งที่เรียนรู้:
- **Async SQLAlchemy** สำหรับ non-blocking database operations
- **Repository Pattern** abstraction layer สำหรับ data access
- **Alembic** migrations management
- **Async CRUD** operations ที่ efficient

---

## ลิงก์ Part ถัดไป

➡️ [Part 095 - FastAPI Authentication](./part-095.md)
