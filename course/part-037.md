# Part 037: SQLAlchemy ORM
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ SQLAlchemy ORM กับ Python ได้
- สร้าง Models, Relationships
- CRUD operations ด้วย Session
- Query ขั้นสูง
- Alembic Migrations
- ใช้กับ Flask และ FastAPI

---

## 1. SQLAlchemy Setup

```bash
pip install sqlalchemy psycopg2-binary alembic
# สำหรับ async:
pip install sqlalchemy[asyncio] asyncpg
```

```python
# database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker

# SQLite (development)
DATABASE_URL = "sqlite:///./myapp.db"

# PostgreSQL (production)
# DATABASE_URL = "postgresql://user:password@localhost/mydb"

engine = create_engine(DATABASE_URL, echo=True)  # echo=True แสดง SQL
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

class Base(DeclarativeBase):
    pass
```

---

## 2. Defining Models

```python
# models.py
from sqlalchemy import Column, Integer, String, Float, Boolean, DateTime, ForeignKey, Text, Enum
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from database import Base
import enum

class UserRole(enum.Enum):
    ADMIN = "admin"
    USER = "user"
    STAFF = "staff"

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String(50), unique=True, nullable=False, index=True)
    email = Column(String(255), unique=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)
    full_name = Column(String(100))
    role = Column(Enum(UserRole), default=UserRole.USER)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, onupdate=func.now())
    
    # Relationships
    posts = relationship("Post", back_populates="author", cascade="all, delete-orphan")
    comments = relationship("Comment", back_populates="user")
    
    def __repr__(self):
        return f"<User(id={self.id}, username={self.username!r})>"

class Post(Base):
    __tablename__ = "posts"
    
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False)
    content = Column(Text, nullable=False)
    slug = Column(String(200), unique=True, index=True)
    is_published = Column(Boolean, default=False)
    views = Column(Integer, default=0)
    created_at = Column(DateTime, server_default=func.now())
    
    # Foreign Key
    author_id = Column(Integer, ForeignKey("users.id", ondelete="CASCADE"), nullable=False)
    category_id = Column(Integer, ForeignKey("categories.id"), nullable=True)
    
    # Relationships
    author = relationship("User", back_populates="posts")
    category = relationship("Category", back_populates="posts")
    comments = relationship("Comment", back_populates="post", cascade="all, delete-orphan")
    tags = relationship("Tag", secondary="post_tags", back_populates="posts")
    
    def __repr__(self):
        return f"<Post(id={self.id}, title={self.title!r})>"

class Category(Base):
    __tablename__ = "categories"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(100), unique=True, nullable=False)
    description = Column(String(500))
    
    posts = relationship("Post", back_populates="category")

class Tag(Base):
    __tablename__ = "tags"
    
    id = Column(Integer, primary_key=True)
    name = Column(String(50), unique=True, nullable=False)
    
    posts = relationship("Post", secondary="post_tags", back_populates="tags")

# Many-to-Many association table
from sqlalchemy import Table
post_tags = Table(
    "post_tags",
    Base.metadata,
    Column("post_id", Integer, ForeignKey("posts.id"), primary_key=True),
    Column("tag_id", Integer, ForeignKey("tags.id"), primary_key=True),
)

class Comment(Base):
    __tablename__ = "comments"
    
    id = Column(Integer, primary_key=True)
    content = Column(Text, nullable=False)
    created_at = Column(DateTime, server_default=func.now())
    
    post_id = Column(Integer, ForeignKey("posts.id", ondelete="CASCADE"))
    user_id = Column(Integer, ForeignKey("users.id", ondelete="SET NULL"), nullable=True)
    
    post = relationship("Post", back_populates="comments")
    user = relationship("User", back_populates="comments")

# สร้างตาราง
from database import engine
Base.metadata.create_all(bind=engine)
```

---

## 3. CRUD Operations

```python
# crud.py
from sqlalchemy.orm import Session
from sqlalchemy import select, update, delete
from models import User, Post, Tag
from database import SessionLocal

# Dependency สำหรับ FastAPI
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# CREATE
def create_user(db: Session, username: str, email: str, password_hash: str):
    user = User(username=username, email=email, hashed_password=password_hash)
    db.add(user)
    db.commit()
    db.refresh(user)  # โหลดข้อมูลที่ database สร้าง (id, created_at)
    return user

def create_post(db: Session, title: str, content: str, author_id: int, tag_names: list = None):
    slug = title.lower().replace(" ", "-")
    post = Post(title=title, content=content, slug=slug, author_id=author_id)
    
    if tag_names:
        for tag_name in tag_names:
            tag = db.query(Tag).filter(Tag.name == tag_name).first()
            if not tag:
                tag = Tag(name=tag_name)
                db.add(tag)
            post.tags.append(tag)
    
    db.add(post)
    db.commit()
    db.refresh(post)
    return post

# READ
def get_user(db: Session, user_id: int):
    return db.query(User).filter(User.id == user_id).first()

def get_user_by_email(db: Session, email: str):
    return db.query(User).filter(User.email == email).first()

def get_users(db: Session, skip: int = 0, limit: int = 100, active_only: bool = True):
    query = db.query(User)
    if active_only:
        query = query.filter(User.is_active == True)
    return query.offset(skip).limit(limit).all()

def get_posts_by_author(db: Session, author_id: int, published_only: bool = True):
    query = db.query(Post).filter(Post.author_id == author_id)
    if published_only:
        query = query.filter(Post.is_published == True)
    return query.order_by(Post.created_at.desc()).all()

# UPDATE
def update_user(db: Session, user_id: int, **kwargs):
    user = get_user(db, user_id)
    if not user:
        return None
    for key, value in kwargs.items():
        setattr(user, key, value)
    db.commit()
    db.refresh(user)
    return user

def publish_post(db: Session, post_id: int):
    db.query(Post).filter(Post.id == post_id).update({"is_published": True})
    db.commit()

# DELETE
def delete_user(db: Session, user_id: int):
    user = get_user(db, user_id)
    if user:
        db.delete(user)
        db.commit()
        return True
    return False
```

---

## 4. Advanced Queries

```python
from sqlalchemy import func, and_, or_, not_, desc, asc
from sqlalchemy.orm import joinedload, selectinload

def search_posts(db: Session, keyword: str, category_id: int = None):
    query = db.query(Post).filter(
        and_(
            Post.is_published == True,
            or_(
                Post.title.ilike(f"%{keyword}%"),
                Post.content.ilike(f"%{keyword}%"),
            )
        )
    )
    
    if category_id:
        query = query.filter(Post.category_id == category_id)
    
    return query.order_by(desc(Post.created_at)).all()

def get_posts_with_authors(db: Session):
    """Eager loading - ป้องกัน N+1 query"""
    return db.query(Post).options(
        joinedload(Post.author),        # JOIN query
        selectinload(Post.tags),        # separate SELECT
        selectinload(Post.comments),
    ).filter(Post.is_published == True).all()

def get_post_statistics(db: Session):
    """Aggregate queries"""
    from sqlalchemy import case
    
    stats = db.query(
        func.count(Post.id).label("total_posts"),
        func.count(case((Post.is_published == True, 1))).label("published"),
        func.avg(Post.views).label("avg_views"),
        func.max(Post.views).label("max_views"),
    ).first()
    
    return {
        "total": stats.total_posts,
        "published": stats.published,
        "avg_views": round(float(stats.avg_views or 0), 2),
        "max_views": stats.max_views or 0,
    }

def get_top_authors(db: Session, limit: int = 5):
    """Group by + Having"""
    return (
        db.query(
            User.id,
            User.username,
            func.count(Post.id).label("post_count"),
        )
        .join(Post, Post.author_id == User.id)
        .filter(Post.is_published == True)
        .group_by(User.id, User.username)
        .having(func.count(Post.id) > 0)
        .order_by(desc("post_count"))
        .limit(limit)
        .all()
    )

def paginate_posts(db: Session, page: int = 1, per_page: int = 10):
    """Pagination"""
    offset = (page - 1) * per_page
    
    total = db.query(func.count(Post.id)).filter(Post.is_published == True).scalar()
    posts = (
        db.query(Post)
        .filter(Post.is_published == True)
        .order_by(desc(Post.created_at))
        .offset(offset)
        .limit(per_page)
        .all()
    )
    
    return {
        "items": posts,
        "total": total,
        "page": page,
        "per_page": per_page,
        "pages": (total + per_page - 1) // per_page,
    }
```

---

## 5. Alembic Migrations

```bash
# ติดตั้ง
pip install alembic

# เริ่มต้น
alembic init alembic

# แก้ไข alembic.ini
# sqlalchemy.url = sqlite:///./myapp.db

# แก้ไข alembic/env.py เพิ่ม:
# from models import Base
# target_metadata = Base.metadata

# สร้าง migration
alembic revision --autogenerate -m "create users and posts tables"

# รัน migration
alembic upgrade head

# ดู migration history
alembic history

# ย้อนกลับ
alembic downgrade -1    # ย้อน 1 step
alembic downgrade base  # ย้อนทั้งหมด
```

---

## 6. SQLAlchemy กับ FastAPI

```python
# main.py (FastAPI)
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from database import engine, Base, get_db
import crud, schemas

Base.metadata.create_all(bind=engine)
app = FastAPI()

@app.post("/users/", response_model=schemas.UserResponse)
def create_user(user: schemas.UserCreate, db: Session = Depends(get_db)):
    existing = crud.get_user_by_email(db, email=user.email)
    if existing:
        raise HTTPException(status_code=400, detail="Email already registered")
    return crud.create_user(db, **user.dict())

@app.get("/users/{user_id}", response_model=schemas.UserResponse)
def read_user(user_id: int, db: Session = Depends(get_db)):
    user = crud.get_user(db, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.get("/posts/")
def read_posts(
    skip: int = 0,
    limit: int = 10,
    keyword: str = None,
    db: Session = Depends(get_db)
):
    if keyword:
        posts = crud.search_posts(db, keyword)
    else:
        result = crud.paginate_posts(db, page=skip//limit+1, per_page=limit)
        return result
    return posts
```

---

## 7. สรุป Part 037

✅ **SQLAlchemy ORM** - Python ORM ที่ทรงพลัง  
✅ **Models** - กำหนด schema ด้วย Python class  
✅ **Relationships** - one-to-many, many-to-many  
✅ **Session** - จัดการ database connection  
✅ **CRUD** - create, read, update, delete  
✅ **Advanced Queries** - joins, aggregates, pagination  
✅ **Alembic** - database migrations  

---

## ➡️ ถัดไป: Part 038 - Design Patterns ใน Python

*Part 037/100+ | Python Course - Beginner to World-Class*
