# Part 089: FastAPI Dependencies
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ Dependency Injection ใน FastAPI
- ใช้ Depends() สำหรับ common dependencies
- สร้าง Database session dependency
- ใช้ class-based dependencies
- ทำ dependency chaining

---

## 1. Dependency Injection คืออะไร?

Dependency Injection (DI) คือ pattern ที่ระบบ inject (ส่ง) dependencies ให้ function แทนที่จะสร้างเอง

```python
# ไม่ใช้ DI (ไม่ดี)
def get_current_user():
    # สร้าง database connection เอง ทุกครั้ง
    db = connect_database()
    token = get_token_from_request()  # แต่อยู่ที่ไหน??
    return db.get_user(token)

# ใช้ DI (ดี)
def get_current_user(
    db: Session = Depends(get_db),        # FastAPI inject db ให้
    token: str = Depends(get_token)       # FastAPI inject token ให้
):
    return db.get_user(token)
```

### ประโยชน์ของ DI
- **Reusability**: ใช้ dependency เดียวกันในหลาย endpoints
- **Testability**: ง่ายต่อการ mock ใน tests
- **Maintainability**: แก้ที่เดียวส่งผลทั้ง app

---

## 2. Depends() พื้นฐาน

```python
# basic_depends.py

from fastapi import FastAPI, Depends, HTTPException
from typing import Optional

app = FastAPI()


# ---- Dependency Functions ----

def get_query_params(
    skip: int = 0,
    limit: int = 10,
    q: Optional[str] = None
):
    """Common query parameters"""
    return {"skip": skip, "limit": limit, "q": q}


def verify_api_key(api_key: str = None):
    """ตรวจสอบ API key จาก query parameter"""
    if not api_key:
        raise HTTPException(401, "ต้องระบุ api_key")
    if api_key != "secret-api-key":
        raise HTTPException(403, "API key ไม่ถูกต้อง")
    return api_key


# ---- Routes ที่ใช้ Dependencies ----

@app.get("/items")
def get_items(params: dict = Depends(get_query_params)):
    """ใช้ common pagination"""
    return {
        "skip": params["skip"],
        "limit": params["limit"],
        "query": params["q"],
        "items": []
    }


@app.get("/users")
def get_users(params: dict = Depends(get_query_params)):
    """ใช้ common pagination เหมือนกัน"""
    return {
        "params": params,
        "users": []
    }


@app.get("/protected")
def protected_endpoint(api_key: str = Depends(verify_api_key)):
    """Endpoint ที่ต้องการ API key"""
    return {"message": "Welcome!", "api_key": api_key}
```

---

## 3. Class-based Dependencies

```python
# class_depends.py

from fastapi import FastAPI, Depends, Query
from typing import Optional

app = FastAPI()


class Pagination:
    """Class-based dependency สำหรับ pagination"""
    
    def __init__(
        self,
        page: int = Query(1, ge=1, description="หน้า"),
        per_page: int = Query(10, ge=1, le=100, description="จำนวนต่อหน้า")
    ):
        self.page = page
        self.per_page = per_page
        self.offset = (page - 1) * per_page
    
    def paginate(self, items: list):
        """Paginate list"""
        return items[self.offset:self.offset + self.per_page]


class SearchFilter:
    """Class-based dependency สำหรับ search"""
    
    def __init__(
        self,
        q: Optional[str] = Query(None, min_length=1),
        sort_by: str = Query("created_at"),
        order: str = Query("desc", regex="^(asc|desc)$")
    ):
        self.q = q
        self.sort_by = sort_by
        self.order = order


@app.get("/items")
def get_items(
    pagination: Pagination = Depends(Pagination),
    search: SearchFilter = Depends(SearchFilter)
):
    """ใช้ class-based dependencies"""
    all_items = [{"id": i, "name": f"Item {i}"} for i in range(1, 101)]
    
    # Search
    if search.q:
        all_items = [i for i in all_items if search.q.lower() in i["name"].lower()]
    
    # Sort
    reverse = search.order == "desc"
    all_items.sort(key=lambda x: x.get(search.sort_by, ""), reverse=reverse)
    
    # Paginate
    items = pagination.paginate(all_items)
    
    return {
        "items": items,
        "pagination": {
            "page": pagination.page,
            "per_page": pagination.per_page,
            "total": len(all_items)
        }
    }
```

---

## 4. Database Session Dependency

```python
# database.py

from sqlalchemy import create_engine, Column, Integer, String, Boolean
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session

DATABASE_URL = "sqlite:///./test.db"

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False}  # SQLite เท่านั้น
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()


# Models
class UserDB(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)
    email = Column(String, unique=True)
    is_active = Column(Boolean, default=True)


# สร้าง tables
Base.metadata.create_all(bind=engine)


# ---- Database Dependency ----
def get_db():
    """
    Database session dependency
    
    ทำงานอย่างไร:
    1. สร้าง session ใหม่สำหรับแต่ละ request
    2. yield session ให้ endpoint ใช้
    3. ปิด session หลัง request เสร็จ (ไม่ว่าจะ success หรือ error)
    """
    db = SessionLocal()
    try:
        yield db          # ส่ง db ให้ endpoint
    finally:
        db.close()        # ปิดทุกครั้ง ไม่ว่าจะเกิดอะไรขึ้น
```

### ใช้ Database Dependency ใน Endpoints
```python
# user_routes.py

from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from database import get_db, UserDB
from pydantic import BaseModel
from typing import List, Optional

app = FastAPI()


class UserCreate(BaseModel):
    username: str
    email: str


class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    is_active: bool
    
    class Config:
        from_attributes = True


@app.post("/users", response_model=UserResponse)
def create_user(
    user: UserCreate,
    db: Session = Depends(get_db)  # inject db session
):
    """สร้าง user ใหม่"""
    # ตรวจสอบซ้ำ
    existing = db.query(UserDB).filter(
        UserDB.username == user.username
    ).first()
    if existing:
        raise HTTPException(409, "username นี้มีอยู่แล้ว")
    
    # สร้าง user
    db_user = UserDB(username=user.username, email=user.email)
    db.add(db_user)
    db.commit()
    db.refresh(db_user)  # โหลดข้อมูล id ที่ database สร้างให้
    return db_user


@app.get("/users", response_model=List[UserResponse])
def get_users(
    skip: int = 0,
    limit: int = 10,
    db: Session = Depends(get_db)
):
    """ดึงรายการ users"""
    users = db.query(UserDB).offset(skip).limit(limit).all()
    return users


@app.get("/users/{user_id}", response_model=UserResponse)
def get_user(
    user_id: int,
    db: Session = Depends(get_db)
):
    """ดึงข้อมูล user"""
    user = db.get(UserDB, user_id)
    if not user:
        raise HTTPException(404, f"ไม่พบ user ID {user_id}")
    return user


@app.delete("/users/{user_id}", status_code=204)
def delete_user(
    user_id: int,
    db: Session = Depends(get_db)
):
    """ลบ user"""
    user = db.get(UserDB, user_id)
    if not user:
        raise HTTPException(404, "ไม่พบ user")
    db.delete(user)
    db.commit()
    return None
```

---

## 5. Dependency Chaining

```python
# dependency_chain.py

from fastapi import FastAPI, Depends, HTTPException, Header
from typing import Optional

app = FastAPI()


# Level 1: ดึง token
async def get_token(authorization: Optional[str] = Header(None)):
    """ดึง token จาก Authorization header"""
    if not authorization:
        raise HTTPException(401, "ต้องมี Authorization header")
    
    if not authorization.startswith("Bearer "):
        raise HTTPException(401, "รูปแบบ token ไม่ถูกต้อง (ต้องเป็น Bearer token)")
    
    token = authorization.split(" ")[1]
    return token


# Level 2: ตรวจสอบ token และดึง user
async def get_current_user(token: str = Depends(get_token)):
    """ตรวจสอบ token และคืน user"""
    # จำลองการตรวจสอบ token
    if token == "valid-token":
        return {"id": 1, "username": "john", "role": "user"}
    elif token == "admin-token":
        return {"id": 2, "username": "admin", "role": "admin"}
    
    raise HTTPException(401, "Token ไม่ถูกต้องหรือหมดอายุ")


# Level 3: ตรวจสอบว่าเป็น active user
async def get_active_user(
    current_user: dict = Depends(get_current_user)
):
    """ตรวจสอบว่า user active"""
    if not current_user.get("is_active", True):
        raise HTTPException(403, "บัญชีนี้ถูกระงับ")
    return current_user


# Level 4: ตรวจสอบว่าเป็น admin
async def get_admin_user(
    current_user: dict = Depends(get_active_user)
):
    """ตรวจสอบว่าเป็น admin"""
    if current_user.get("role") != "admin":
        raise HTTPException(403, "ต้องการสิทธิ์ admin")
    return current_user


# ---- Endpoints ----

@app.get("/me")
def get_my_profile(current_user: dict = Depends(get_active_user)):
    """Profile ของตัวเอง"""
    return current_user


@app.get("/admin/dashboard")
def admin_dashboard(admin: dict = Depends(get_admin_user)):
    """Admin dashboard"""
    return {"message": "Welcome Admin!", "admin": admin}
```

---

## 6. Global Dependencies

```python
# global_depends.py

from fastapi import FastAPI, Depends, Header, HTTPException
from typing import Optional


async def verify_request_id(x_request_id: Optional[str] = Header(None)):
    """ตรวจสอบ X-Request-ID header"""
    if not x_request_id:
        # ไม่บังคับ แต่ log warning
        print("Warning: X-Request-ID ไม่มี")
    return x_request_id


async def rate_limiter():
    """Rate limiting (ตัวอย่างอย่างง่าย)"""
    # จริงๆ ควรใช้ Redis หรือ in-memory store
    pass


# Apply dependency ให้ทั้ง app
app = FastAPI(dependencies=[Depends(verify_request_id)])

# หรือ apply ให้เฉพาะ router
from fastapi import APIRouter

router = APIRouter(
    prefix="/api",
    dependencies=[Depends(rate_limiter)]
)


@router.get("/items")
def get_items():
    return []


@router.get("/users")
def get_users():
    return []


app.include_router(router)
```

---

## 7. ตัวอย่างสมบูรณ์

```python
# complete_depends.py

from fastapi import FastAPI, Depends, HTTPException, Header, Query
from sqlalchemy.orm import Session
from typing import Optional, List
from pydantic import BaseModel
from datetime import datetime, timezone

# Database setup (simplified)
from database import get_db, UserDB


app = FastAPI(title="Complete Dependencies Demo")


# ---- Pydantic Models ----

class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    
    class Config:
        from_attributes = True


# ---- Dependencies ----

async def get_current_user_from_token(
    authorization: Optional[str] = Header(None),
    db: Session = Depends(get_db)
):
    """ดึง current user จาก token + database"""
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(401, "ต้องมี Bearer token")
    
    token = authorization.split(" ")[1]
    
    # จำลองการ decode token เพื่อหา user_id
    if token == "user-token-1":
        user_id = 1
    elif token == "user-token-2":
        user_id = 2
    else:
        raise HTTPException(401, "Token ไม่ถูกต้อง")
    
    user = db.get(UserDB, user_id)
    if not user:
        raise HTTPException(401, "User ไม่พบ")
    
    return user


class PaginationDep:
    """Pagination dependency"""
    def __init__(
        self,
        page: int = Query(1, ge=1),
        per_page: int = Query(10, ge=1, le=50)
    ):
        self.page = page
        self.per_page = per_page
        self.offset = (page - 1) * per_page


# ---- Routes ----

@app.get("/my/profile", response_model=UserResponse)
async def my_profile(
    current_user: UserDB = Depends(get_current_user_from_token)
):
    """ดู profile ของตัวเอง"""
    return current_user


@app.get("/users", response_model=List[UserResponse])
def list_users(
    pagination: PaginationDep = Depends(PaginationDep),
    db: Session = Depends(get_db),
    current_user: UserDB = Depends(get_current_user_from_token)
):
    """ดูรายการ users (ต้อง login)"""
    users = db.query(UserDB).offset(pagination.offset).limit(pagination.per_page).all()
    return users


if __name__ == "__main__":
    import uvicorn
    uvicorn.run("complete_depends:app", host="0.0.0.0", port=8000, reload=True)
```

---

## 8. สรุป Part 089

✅ **Depends()** inject dependency ให้ endpoint อัตโนมัติ  
✅ **Function dependencies** สำหรับ simple reusable logic  
✅ **Class dependencies** สำหรับ complex dependencies ที่มี state  
✅ **Database dependency** ใช้ `yield` เพื่อจัดการ cleanup  
✅ **Dependency chaining** สร้าง dependencies ที่ depend on กันได้  
✅ **Global dependencies** apply ให้ทั้ง app หรือ router  

---

## ➡️ ถัดไป: Part 090 - FastAPI Authentication

*Part 089/100+ | Python Course - Beginner to World-Class*
