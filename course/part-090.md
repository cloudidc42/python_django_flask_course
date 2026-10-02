# Part 090: FastAPI Authentication
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- ใช้ OAuth2PasswordBearer สำหรับ authentication
- สร้าง JWT tokens ด้วย python-jose
- Hash password ด้วย passlib
- สร้าง protected endpoints
- Implement refresh tokens

---

## 1. ติดตั้ง dependencies

```bash
pip install python-jose[cryptography] passlib[bcrypt]
```

---

## 2. JWT Authentication Setup

```python
# auth.py

from datetime import datetime, timedelta, timezone
from typing import Optional
import jwt
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

# ---- Configuration ----
SECRET_KEY = "your-secret-key-change-in-production-use-env-var"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30
REFRESH_TOKEN_EXPIRE_DAYS = 30

# ---- Password Hashing ----
# bcrypt context สำหรับ hash passwords
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# ---- OAuth2 ----
# OAuth2PasswordBearer บอก FastAPI ว่า:
# 1. ต้องการ Bearer token ใน Authorization header
# 2. URL สำหรับรับ token คือ /auth/token
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")


def verify_password(plain_password: str, hashed_password: str) -> bool:
    """ตรวจสอบ password กับ hash"""
    return pwd_context.verify(plain_password, hashed_password)


def get_password_hash(password: str) -> str:
    """Hash password"""
    return pwd_context.hash(password)


def create_access_token(
    data: dict,
    expires_delta: Optional[timedelta] = None
) -> str:
    """สร้าง JWT access token"""
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    
    to_encode.update({"exp": expire, "type": "access"})
    
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def create_refresh_token(data: dict) -> str:
    """สร้าง JWT refresh token"""
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    to_encode.update({"exp": expire, "type": "refresh"})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


def decode_token(token: str) -> dict:
    """Decode และ verify JWT token"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token หมดอายุ",
            headers={"WWW-Authenticate": "Bearer"}
        )
    except jwt.InvalidTokenError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token ไม่ถูกต้อง",
            headers={"WWW-Authenticate": "Bearer"}
        )
```

---

## 3. User Model และ Database

```python
# models.py

from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime, timezone

Base = declarative_base()


class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True, nullable=False)
    email = Column(String, unique=True, index=True, nullable=False)
    hashed_password = Column(String, nullable=False)
    full_name = Column(String, nullable=True)
    is_active = Column(Boolean, default=True)
    is_admin = Column(Boolean, default=False)
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))
```

```python
# schemas.py

from pydantic import BaseModel, EmailStr, Field
from typing import Optional
from datetime import datetime


class UserBase(BaseModel):
    username: str = Field(..., min_length=3, max_length=80)
    email: EmailStr
    full_name: Optional[str] = None


class UserCreate(UserBase):
    password: str = Field(..., min_length=8)


class UserResponse(UserBase):
    id: int
    is_active: bool
    created_at: datetime
    
    class Config:
        from_attributes = True


class Token(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"


class TokenData(BaseModel):
    username: Optional[str] = None
    user_id: Optional[int] = None
```

---

## 4. Auth Routes

```python
# auth_router.py

from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy.orm import Session

from database import get_db
from models import User
from schemas import UserCreate, UserResponse, Token
from auth import (
    verify_password, get_password_hash,
    create_access_token, create_refresh_token,
    decode_token, oauth2_scheme
)

router = APIRouter(prefix="/auth", tags=["Authentication"])


@router.post("/register", response_model=UserResponse, status_code=201)
def register(user_data: UserCreate, db: Session = Depends(get_db)):
    """สมัครสมาชิก"""
    
    # ตรวจสอบ username ซ้ำ
    if db.query(User).filter(User.username == user_data.username).first():
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="username นี้มีอยู่แล้ว"
        )
    
    # ตรวจสอบ email ซ้ำ
    if db.query(User).filter(User.email == user_data.email).first():
        raise HTTPException(
            status_code=status.HTTP_409_CONFLICT,
            detail="email นี้มีอยู่แล้ว"
        )
    
    # สร้าง user
    hashed_password = get_password_hash(user_data.password)
    db_user = User(
        username=user_data.username,
        email=user_data.email,
        hashed_password=hashed_password,
        full_name=user_data.full_name
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    
    return db_user


@router.post("/token", response_model=Token)
def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: Session = Depends(get_db)
):
    """
    Login ด้วย username และ password
    ใช้ OAuth2PasswordRequestForm (form data ไม่ใช่ JSON)
    """
    # ค้นหา user
    user = db.query(User).filter(User.username == form_data.username).first()
    
    if not user or not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง",
            headers={"WWW-Authenticate": "Bearer"}
        )
    
    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="บัญชีนี้ถูกระงับ"
        )
    
    # สร้าง tokens
    token_data = {"sub": str(user.id), "username": user.username}
    access_token = create_access_token(token_data)
    refresh_token = create_refresh_token(token_data)
    
    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer"
    }


@router.post("/refresh", response_model=Token)
def refresh_token(
    refresh_token: str,
    db: Session = Depends(get_db)
):
    """ต่ออายุ access token ด้วย refresh token"""
    payload = decode_token(refresh_token)
    
    if payload.get("type") != "refresh":
        raise HTTPException(401, "ต้องใช้ refresh token")
    
    user_id = int(payload.get("sub"))
    user = db.get(User, user_id)
    
    if not user or not user.is_active:
        raise HTTPException(401, "User ไม่พบหรือถูกระงับ")
    
    token_data = {"sub": str(user.id), "username": user.username}
    new_access_token = create_access_token(token_data)
    new_refresh_token = create_refresh_token(token_data)
    
    return {
        "access_token": new_access_token,
        "refresh_token": new_refresh_token,
        "token_type": "bearer"
    }
```

---

## 5. Current User Dependency

```python
# dependencies.py

from fastapi import Depends, HTTPException, status
from sqlalchemy.orm import Session
from database import get_db
from models import User
from auth import oauth2_scheme, decode_token


async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
) -> User:
    """ดึง current user จาก JWT token"""
    
    # Decode token
    payload = decode_token(token)
    
    if payload.get("type") != "access":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token ประเภทไม่ถูกต้อง"
        )
    
    user_id_str = payload.get("sub")
    if not user_id_str:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token ไม่มีข้อมูล user"
        )
    
    # ดึง user จาก database
    user = db.get(User, int(user_id_str))
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User ไม่พบ"
        )
    
    return user


async def get_current_active_user(
    current_user: User = Depends(get_current_user)
) -> User:
    """ตรวจสอบว่า user active"""
    if not current_user.is_active:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="บัญชีนี้ถูกระงับ"
        )
    return current_user


async def get_admin_user(
    current_user: User = Depends(get_current_active_user)
) -> User:
    """ตรวจสอบว่าเป็น admin"""
    if not current_user.is_admin:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="ต้องการสิทธิ์ admin"
        )
    return current_user
```

---

## 6. Protected Endpoints

```python
# user_routes.py

from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

from database import get_db
from models import User
from schemas import UserResponse
from dependencies import get_current_active_user, get_admin_user

router = APIRouter(prefix="/users", tags=["Users"])


@router.get("/me", response_model=UserResponse)
def get_my_profile(
    current_user: User = Depends(get_current_active_user)
):
    """ดูโปรไฟล์ของตัวเอง"""
    return current_user


@router.put("/me", response_model=UserResponse)
def update_my_profile(
    full_name: str,
    current_user: User = Depends(get_current_active_user),
    db: Session = Depends(get_db)
):
    """แก้ไขโปรไฟล์ของตัวเอง"""
    current_user.full_name = full_name
    db.commit()
    db.refresh(current_user)
    return current_user


@router.get("/", response_model=List[UserResponse])
def list_users(
    skip: int = 0,
    limit: int = 10,
    admin: User = Depends(get_admin_user),  # เฉพาะ admin
    db: Session = Depends(get_db)
):
    """Admin: ดูรายการ users ทั้งหมด"""
    return db.query(User).offset(skip).limit(limit).all()


@router.delete("/{user_id}", status_code=204)
def delete_user(
    user_id: int,
    admin: User = Depends(get_admin_user),
    db: Session = Depends(get_db)
):
    """Admin: ลบ user"""
    user = db.get(User, user_id)
    if not user:
        raise HTTPException(404, "ไม่พบ user")
    db.delete(user)
    db.commit()
    return None
```

---

## 7. Complete Application

```python
# main.py

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from database import engine, Base
from auth_router import router as auth_router
from user_routes import router as user_router

# สร้าง tables
Base.metadata.create_all(bind=engine)

app = FastAPI(
    title="Auth Demo API",
    description="FastAPI Authentication Demo",
    version="1.0.0"
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"]
)

# Include routers
app.include_router(auth_router)
app.include_router(user_router)


@app.get("/")
def root():
    return {
        "message": "Auth Demo API",
        "docs": "/docs"
    }


if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

### ทดสอบด้วย curl
```bash
# สมัครสมาชิก
curl -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"john","email":"john@example.com","password":"password123"}'

# Login (form data)
curl -X POST http://localhost:8000/auth/token \
  -d "username=john&password=password123"

# ดูโปรไฟล์ (ต้องมี token)
curl http://localhost:8000/users/me \
  -H "Authorization: Bearer <your-token>"
```

---

## 8. สรุป Part 090

✅ **OAuth2PasswordBearer** รับ Bearer token จาก Authorization header  
✅ **JWT tokens** สร้างด้วย `python-jose`, decode และ verify  
✅ **passlib bcrypt** hash passwords อย่างปลอดภัย  
✅ **Access + Refresh tokens** pattern สำหรับ secure auth  
✅ **get_current_user** dependency chain ดึง user จาก token  
✅ **Admin protection** dependency เพิ่มชั้นสิทธิ์  

---

## ➡️ ถัดไป: Part 091 - FastAPI Database Integration

*Part 090/100+ | Python Course - Beginner to World-Class*
