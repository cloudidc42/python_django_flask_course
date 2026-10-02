# Part 091: FastAPI Database Integration
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- ใช้ async SQLAlchemy กับ FastAPI
- ทำ CRUD operations แบบ async
- ใช้ Alembic สำหรับ migrations
- สร้าง Repository pattern
- จัดการ database transactions

---

## 1. Async SQLAlchemy Setup

```bash
pip install sqlalchemy[asyncio] asyncpg aiosqlite alembic
```

```python
# database.py

from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase
import os

# Database URL
# SQLite (async version ใช้ aiosqlite)
DATABASE_URL = "sqlite+aiosqlite:///./app.db"

# PostgreSQL (async version ใช้ asyncpg)
# DATABASE_URL = "postgresql+asyncpg://user:password@localhost/mydb"

# สร้าง async engine
engine = create_async_engine(
    DATABASE_URL,
    echo=True,           # Log SQL queries (development เท่านั้น)
    future=True
)

# Async session maker
AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False  # ป้องกัน lazy loading หลัง commit
)


class Base(DeclarativeBase):
    """Base class สำหรับ ORM models"""
    pass


# ---- Dependency ----
async def get_db():
    """Async database session dependency"""
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

---

## 2. Async Models

```python
# models.py

from sqlalchemy import Column, Integer, String, Boolean, DateTime, Text, Float, ForeignKey
from sqlalchemy.orm import relationship
from datetime import datetime, timezone
from database import Base


class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String(80), unique=True, index=True, nullable=False)
    email = Column(String(120), unique=True, index=True, nullable=False)
    hashed_password = Column(String(256), nullable=False)
    is_active = Column(Boolean, default=True)
    is_admin = Column(Boolean, default=False)
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))
    
    # Relationship
    posts = relationship("Post", back_populates="author", cascade="all, delete-orphan")
    
    def __repr__(self):
        return f"<User {self.username}>"


class Post(Base):
    __tablename__ = "posts"
    
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False)
    content = Column(Text, nullable=False)
    published = Column(Boolean, default=False)
    view_count = Column(Integer, default=0)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))
    updated_at = Column(DateTime, default=lambda: datetime.now(timezone.utc),
                        onupdate=lambda: datetime.now(timezone.utc))
    
    # Relationships
    author = relationship("User", back_populates="posts")
    comments = relationship("Comment", back_populates="post", cascade="all, delete-orphan")


class Comment(Base):
    __tablename__ = "comments"
    
    id = Column(Integer, primary_key=True, index=True)
    content = Column(Text, nullable=False)
    post_id = Column(Integer, ForeignKey("posts.id"), nullable=False)
    user_id = Column(Integer, ForeignKey("users.id"), nullable=False)
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))
    
    post = relationship("Post", back_populates="comments")
    author = relationship("User")
```

---

## 3. Repository Pattern

```python
# repositories.py

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, update, delete, func
from sqlalchemy.orm import selectinload
from typing import Optional, List, Tuple
from models import User, Post, Comment


class UserRepository:
    """Repository สำหรับจัดการ User data"""
    
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def create(self, username: str, email: str, hashed_password: str) -> User:
        """สร้าง user ใหม่"""
        user = User(username=username, email=email, hashed_password=hashed_password)
        self.db.add(user)
        await self.db.flush()  # ส่งไป db แต่ยังไม่ commit
        await self.db.refresh(user)
        return user
    
    async def get_by_id(self, user_id: int) -> Optional[User]:
        """ดึง user ตาม ID"""
        result = await self.db.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()
    
    async def get_by_username(self, username: str) -> Optional[User]:
        """ดึง user ตาม username"""
        result = await self.db.execute(
            select(User).where(User.username == username)
        )
        return result.scalar_one_or_none()
    
    async def get_by_email(self, email: str) -> Optional[User]:
        """ดึง user ตาม email"""
        result = await self.db.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()
    
    async def get_all(self, skip: int = 0, limit: int = 10) -> Tuple[List[User], int]:
        """ดึง users ทั้งหมดพร้อม total count"""
        # Count
        count_result = await self.db.execute(select(func.count(User.id)))
        total = count_result.scalar()
        
        # Data
        result = await self.db.execute(
            select(User).offset(skip).limit(limit)
        )
        users = result.scalars().all()
        
        return list(users), total
    
    async def update(self, user_id: int, **kwargs) -> Optional[User]:
        """Update user"""
        await self.db.execute(
            update(User).where(User.id == user_id).values(**kwargs)
        )
        return await self.get_by_id(user_id)
    
    async def delete(self, user_id: int) -> bool:
        """ลบ user"""
        result = await self.db.execute(
            delete(User).where(User.id == user_id)
        )
        return result.rowcount > 0
    
    async def get_with_posts(self, user_id: int) -> Optional[User]:
        """ดึง user พร้อม posts (eager loading)"""
        result = await self.db.execute(
            select(User)
            .options(selectinload(User.posts))
            .where(User.id == user_id)
        )
        return result.scalar_one_or_none()


class PostRepository:
    """Repository สำหรับจัดการ Post data"""
    
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def create(self, title: str, content: str, user_id: int,
                     published: bool = False) -> Post:
        """สร้าง post ใหม่"""
        post = Post(title=title, content=content, user_id=user_id, published=published)
        self.db.add(post)
        await self.db.flush()
        await self.db.refresh(post)
        return post
    
    async def get_by_id(self, post_id: int) -> Optional[Post]:
        """ดึง post ตาม ID"""
        result = await self.db.execute(
            select(Post)
            .options(selectinload(Post.author))
            .where(Post.id == post_id)
        )
        return result.scalar_one_or_none()
    
    async def get_published(
        self,
        skip: int = 0,
        limit: int = 10,
        search: Optional[str] = None
    ) -> Tuple[List[Post], int]:
        """ดึง posts ที่ published"""
        query = select(Post).where(Post.published == True)
        
        if search:
            query = query.where(
                Post.title.ilike(f"%{search}%") |
                Post.content.ilike(f"%{search}%")
            )
        
        # Count
        count_query = select(func.count()).select_from(query.subquery())
        count_result = await self.db.execute(count_query)
        total = count_result.scalar()
        
        # Data with ordering
        query = query.order_by(Post.created_at.desc()).offset(skip).limit(limit)
        query = query.options(selectinload(Post.author))
        result = await self.db.execute(query)
        posts = result.scalars().all()
        
        return list(posts), total
    
    async def increment_view_count(self, post_id: int) -> None:
        """เพิ่ม view count"""
        await self.db.execute(
            update(Post)
            .where(Post.id == post_id)
            .values(view_count=Post.view_count + 1)
        )
    
    async def update(self, post_id: int, **kwargs) -> Optional[Post]:
        """Update post"""
        await self.db.execute(
            update(Post).where(Post.id == post_id).values(**kwargs)
        )
        return await self.get_by_id(post_id)
    
    async def delete(self, post_id: int) -> bool:
        """ลบ post"""
        result = await self.db.execute(
            delete(Post).where(Post.id == post_id)
        )
        return result.rowcount > 0
```

---

## 4. Async CRUD Routes

```python
# routes.py

from fastapi import APIRouter, Depends, HTTPException, status, Query
from sqlalchemy.ext.asyncio import AsyncSession
from typing import Optional, List
from pydantic import BaseModel, Field
from datetime import datetime

from database import get_db
from repositories import UserRepository, PostRepository

router = APIRouter()


# Schemas
class PostCreate(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    content: str = Field(..., min_length=1)
    published: bool = False


class PostUpdate(BaseModel):
    title: Optional[str] = Field(None, max_length=200)
    content: Optional[str] = None
    published: Optional[bool] = None


class AuthorInfo(BaseModel):
    id: int
    username: str
    
    class Config:
        from_attributes = True


class PostResponse(BaseModel):
    id: int
    title: str
    content: str
    published: bool
    view_count: int
    author: Optional[AuthorInfo] = None
    created_at: datetime
    
    class Config:
        from_attributes = True


class PaginatedPosts(BaseModel):
    posts: List[PostResponse]
    total: int
    page: int
    per_page: int
    pages: int


# Routes
@router.get("/posts", response_model=PaginatedPosts)
async def list_posts(
    page: int = Query(1, ge=1),
    per_page: int = Query(10, ge=1, le=50),
    search: Optional[str] = Query(None),
    db: AsyncSession = Depends(get_db)
):
    """ดึงรายการ posts ที่ published"""
    skip = (page - 1) * per_page
    repo = PostRepository(db)
    
    posts, total = await repo.get_published(skip=skip, limit=per_page, search=search)
    pages = (total + per_page - 1) // per_page
    
    return {
        "posts": posts,
        "total": total,
        "page": page,
        "per_page": per_page,
        "pages": pages
    }


@router.post("/posts", response_model=PostResponse, status_code=201)
async def create_post(
    post_data: PostCreate,
    db: AsyncSession = Depends(get_db),
    # current_user: User = Depends(get_current_user)  # เพิ่ม auth
):
    """สร้าง post ใหม่"""
    repo = PostRepository(db)
    post = await repo.create(
        title=post_data.title,
        content=post_data.content,
        user_id=1,  # ควรใช้ current_user.id
        published=post_data.published
    )
    return post


@router.get("/posts/{post_id}", response_model=PostResponse)
async def get_post(
    post_id: int,
    db: AsyncSession = Depends(get_db)
):
    """ดึง post ตาม ID"""
    repo = PostRepository(db)
    post = await repo.get_by_id(post_id)
    
    if not post:
        raise HTTPException(404, f"ไม่พบ post ID {post_id}")
    
    # เพิ่ม view count
    await repo.increment_view_count(post_id)
    
    return post


@router.put("/posts/{post_id}", response_model=PostResponse)
async def update_post(
    post_id: int,
    post_data: PostUpdate,
    db: AsyncSession = Depends(get_db)
):
    """แก้ไข post"""
    repo = PostRepository(db)
    
    existing = await repo.get_by_id(post_id)
    if not existing:
        raise HTTPException(404, "ไม่พบ post")
    
    update_data = post_data.model_dump(exclude_none=True)
    if not update_data:
        return existing
    
    post = await repo.update(post_id, **update_data)
    return post


@router.delete("/posts/{post_id}", status_code=204)
async def delete_post(
    post_id: int,
    db: AsyncSession = Depends(get_db)
):
    """ลบ post"""
    repo = PostRepository(db)
    
    if not await repo.delete(post_id):
        raise HTTPException(404, "ไม่พบ post")
    
    return None
```

---

## 5. Alembic Migrations

```bash
# ติดตั้ง alembic
pip install alembic

# Initialize alembic
alembic init alembic

# สร้าง migration
alembic revision --autogenerate -m "create users and posts tables"

# Apply migration
alembic upgrade head

# Rollback
alembic downgrade -1

# ดู history
alembic history
```

### alembic.ini (แก้ sqlalchemy.url)
```ini
# alembic.ini
[alembic]
script_location = alembic
sqlalchemy.url = sqlite+aiosqlite:///./app.db
```

### alembic/env.py
```python
# alembic/env.py — แก้ให้ import models

from database import Base
from models import User, Post, Comment  # import ทุก model

target_metadata = Base.metadata
```

### ตัวอย่าง Migration File
```python
# alembic/versions/001_create_tables.py

"""create users and posts tables

Revision ID: 001
Revises: 
Create Date: 2024-01-01
"""

from alembic import op
import sqlalchemy as sa

revision = '001'
down_revision = None


def upgrade() -> None:
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('username', sa.String(80), nullable=False),
        sa.Column('email', sa.String(120), nullable=False),
        sa.Column('hashed_password', sa.String(256), nullable=False),
        sa.Column('is_active', sa.Boolean(), default=True),
        sa.Column('created_at', sa.DateTime(), nullable=True),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('username'),
        sa.UniqueConstraint('email')
    )
    
    op.create_table(
        'posts',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('title', sa.String(200), nullable=False),
        sa.Column('content', sa.Text(), nullable=False),
        sa.Column('published', sa.Boolean(), default=False),
        sa.Column('view_count', sa.Integer(), default=0),
        sa.Column('user_id', sa.Integer(), nullable=False),
        sa.Column('created_at', sa.DateTime(), nullable=True),
        sa.ForeignKeyConstraint(['user_id'], ['users.id'], ondelete='CASCADE'),
        sa.PrimaryKeyConstraint('id')
    )


def downgrade() -> None:
    op.drop_table('posts')
    op.drop_table('users')
```

---

## 6. สรุป Part 091

✅ **Async SQLAlchemy** ใช้ `create_async_engine` และ `AsyncSession`  
✅ **Async dependency** ใช้ `async def get_db()` กับ `yield`  
✅ **Repository pattern** แยก database logic ออกจาก routes  
✅ **selectinload** สำหรับ eager loading relationships  
✅ **Alembic** จัดการ database migrations  
✅ **model_dump(exclude_none=True)** สำหรับ partial updates  

---

## ➡️ ถัดไป: Part 092 - FastAPI Background Tasks

*Part 091/100+ | Python Course - Beginner to World-Class*
