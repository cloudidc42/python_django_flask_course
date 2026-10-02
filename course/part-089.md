# Part 089: FastAPI Dependencies and Security
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- ใช้ Depends() สำหรับ dependency injection
- สร้าง layered dependencies ซ้อนกัน
- ใช้ database session เป็น dependency
- สร้าง OAuth2PasswordBearer authentication
- สร้างและ verify JWT token ด้วย python-jose
- Hash password ด้วย passlib
- สร้าง protected endpoints ด้วย current_user dependency
- ทำ Role-based access control (RBAC)
- สร้าง API key authentication

---

## 1. Depends() สำหรับ Dependency Injection

Dependency Injection (DI) คือ pattern ที่ช่วยให้โค้ดเป็น reusable และ testable โดย inject dependencies เข้าไปใน function แทนที่จะสร้างเอง

```python
# basic_deps.py

from fastapi import FastAPI, Depends, Query, HTTPException
from typing import Optional

app = FastAPI()


# Dependency พื้นฐาน - function ธรรมดา
def get_query_params(
    skip: int = Query(default=0, ge=0, description="จำนวนที่ข้าม"),
    limit: int = Query(default=10, ge=1, le=100, description="จำนวนที่แสดง")
):
    """Dependency สำหรับ pagination parameters"""
    return {"skip": skip, "limit": limit}


# ใช้ Depends() ใน endpoint
@app.get("/items/")
def get_items(pagination: dict = Depends(get_query_params)):
    """ดึงรายการสินค้าด้วย pagination"""
    # pagination จะมี {"skip": ..., "limit": ...} อัตโนมัติ
    mock_items = [{"id": i, "name": f"สินค้า {i}"} for i in range(1, 51)]
    skip = pagination["skip"]
    limit = pagination["limit"]
    return mock_items[skip: skip + limit]


@app.get("/users/")
def get_users(pagination: dict = Depends(get_query_params)):
    """ดึงรายการผู้ใช้ (reuse dependency เดิม)"""
    mock_users = [{"id": i, "name": f"ผู้ใช้ {i}"} for i in range(1, 31)]
    skip = pagination["skip"]
    limit = pagination["limit"]
    return mock_users[skip: skip + limit]


# Dependency แบบ class (callable)
class PaginationParams:
    """Dependency แบบ class สำหรับ pagination"""

    def __init__(
        self,
        skip: int = Query(default=0, ge=0),
        limit: int = Query(default=10, ge=1, le=100),
        sort_by: str = Query(default="id"),
        sort_order: str = Query(default="asc", pattern="^(asc|desc)$")
    ):
        self.skip = skip
        self.limit = limit
        self.sort_by = sort_by
        self.sort_order = sort_order


@app.get("/products/")
def get_products(params: PaginationParams = Depends(PaginationParams)):
    """ดึงสินค้าพร้อม sorting และ pagination"""
    return {
        "skip": params.skip,
        "limit": params.limit,
        "sort_by": params.sort_by,
        "sort_order": params.sort_order
    }


# Dependency แบบมี validation
def validate_api_version(version: str = Query(default="v1")):
    """ตรวจสอบ API version"""
    supported_versions = ["v1", "v2"]
    if version not in supported_versions:
        raise HTTPException(
            status_code=400,
            detail=f"Version {version} ไม่รองรับ กรุณาใช้: {supported_versions}"
        )
    return version


@app.get("/api/data/")
def get_data(
    pagination: dict = Depends(get_query_params),
    version: str = Depends(validate_api_version)
):
    """Endpoint ที่ใช้หลาย dependencies"""
    return {
        "api_version": version,
        "pagination": pagination,
        "data": ["ข้อมูล A", "ข้อมูล B", "ข้อมูล C"]
    }
```

---

## 2. Layered Dependencies (Dependencies ซ้อนกัน)

```python
# layered_deps.py

from fastapi import FastAPI, Depends, Header, HTTPException
from typing import Optional

app = FastAPI()


# Layer 1: ตรวจสอบ header พื้นฐาน
def get_request_headers(
    x_request_id: Optional[str] = Header(default=None),
    x_client_version: Optional[str] = Header(default=None)
):
    """ดึง custom headers"""
    return {
        "request_id": x_request_id or "no-id",
        "client_version": x_client_version or "unknown"
    }


# Layer 2: ตรวจสอบ API key (พึ่ง Layer 1)
def verify_api_key(
    x_api_key: str = Header(..., description="API Key"),
    headers: dict = Depends(get_request_headers)
):
    """ตรวจสอบ API key พร้อมดึง headers"""
    valid_keys = {"key-admin": "admin", "key-user": "user", "key-readonly": "readonly"}

    if x_api_key not in valid_keys:
        raise HTTPException(
            status_code=401,
            detail="API key ไม่ถูกต้อง"
        )

    return {
        "role": valid_keys[x_api_key],
        "headers": headers
    }


# Layer 3: ตรวจสอบ permissions (พึ่ง Layer 2)
def require_write_access(auth: dict = Depends(verify_api_key)):
    """ตรวจสอบว่ามีสิทธิ์ write"""
    if auth["role"] == "readonly":
        raise HTTPException(
            status_code=403,
            detail="ไม่มีสิทธิ์ write - กรุณาใช้ API key ที่มีสิทธิ์เพียงพอ"
        )
    return auth


def require_admin_access(auth: dict = Depends(verify_api_key)):
    """ตรวจสอบว่าเป็น admin"""
    if auth["role"] != "admin":
        raise HTTPException(
            status_code=403,
            detail="เฉพาะ admin เท่านั้น"
        )
    return auth


# Endpoints ที่ใช้ layered dependencies
@app.get("/public/data/")
def get_public_data():
    """ไม่ต้องการ authentication"""
    return {"data": "สาธารณะ"}


@app.get("/protected/data/")
def get_protected_data(auth: dict = Depends(verify_api_key)):
    """ต้องการ API key ที่ถูกต้อง"""
    return {
        "data": "ข้อมูลที่ป้องกัน",
        "role": auth["role"],
        "request_id": auth["headers"]["request_id"]
    }


@app.post("/protected/data/")
def create_data(auth: dict = Depends(require_write_access)):
    """ต้องการ write access"""
    return {"message": f"สร้างข้อมูลสำเร็จ โดย role: {auth['role']}"}


@app.delete("/admin/data/{data_id}")
def delete_data(data_id: int, auth: dict = Depends(require_admin_access)):
    """เฉพาะ admin เท่านั้น"""
    return {"message": f"ลบข้อมูล {data_id} สำเร็จ"}
```

---

## 3. Database Session เป็น Dependency

Pattern ที่ใช้กับ SQLAlchemy เพื่อจัดการ database session

```python
# db_dependency.py

from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy import create_engine, Column, Integer, String, Boolean
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from pydantic import BaseModel
from typing import Optional, List, Generator

# === Database Setup ===

DATABASE_URL = "sqlite:///./test.db"  # ใช้ SQLite สำหรับตัวอย่าง

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False}  # เฉพาะ SQLite
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()


# === SQLAlchemy Models ===

class UserModel(Base):
    """Model สำหรับตาราง users ใน database"""
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)
    email = Column(String, unique=True, index=True)
    hashed_password = Column(String)
    is_active = Column(Boolean, default=True)


# สร้างตาราง
Base.metadata.create_all(bind=engine)


# === Pydantic Schemas ===

class UserCreate(BaseModel):
    username: str
    email: str
    password: str


class UserResponse(BaseModel):
    id: int
    username: str
    email: str
    is_active: bool

    class Config:
        from_attributes = True  # สำหรับ Pydantic v2


# === Database Dependency ===

def get_db() -> Generator[Session, None, None]:
    """
    Dependency สำหรับ database session
    
    ใช้ yield เพื่อให้ FastAPI จัดการ lifecycle:
    1. สร้าง session ก่อน request
    2. ส่ง session ให้ endpoint ใช้งาน
    3. ปิด session หลัง request เสร็จ (แม้จะ error)
    """
    db = SessionLocal()
    try:
        yield db  # ส่ง session ให้ endpoint
    finally:
        db.close()  # ปิด session เสมอ


# === FastAPI App ===

app = FastAPI()


@app.post("/users/", response_model=UserResponse, status_code=201)
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    """
    สร้าง user ใหม่
    db session ถูก inject โดยอัตโนมัติ
    """
    # ตรวจสอบว่า username ซ้ำหรือไม่
    existing = db.query(UserModel).filter(UserModel.username == user.username).first()
    if existing:
        raise HTTPException(status_code=400, detail="Username นี้ถูกใช้แล้ว")

    # สร้าง user ใหม่ (hash password ในระบบจริง)
    db_user = UserModel(
        username=user.username,
        email=user.email,
        hashed_password=f"hashed_{user.password}"  # อย่าทำแบบนี้ในระบบจริง!
    )
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user


@app.get("/users/", response_model=List[UserResponse])
def get_users(skip: int = 0, limit: int = 10, db: Session = Depends(get_db)):
    """ดึงรายการ users"""
    users = db.query(UserModel).offset(skip).limit(limit).all()
    return users


@app.get("/users/{user_id}", response_model=UserResponse)
def get_user(user_id: int, db: Session = Depends(get_db)):
    """ดึง user ตาม id"""
    user = db.query(UserModel).filter(UserModel.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="ไม่พบผู้ใช้")
    return user


# Dependency สำหรับตรวจสอบและดึง user
def get_user_or_404(user_id: int, db: Session = Depends(get_db)) -> UserModel:
    """Dependency ที่ดึง user หรือ raise 404"""
    user = db.query(UserModel).filter(UserModel.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="ไม่พบผู้ใช้")
    return user


@app.delete("/users/{user_id}")
def delete_user(user: UserModel = Depends(get_user_or_404), db: Session = Depends(get_db)):
    """ลบ user ด้วย nested dependency"""
    db.delete(user)
    db.commit()
    return {"message": f"ลบผู้ใช้ {user.username} สำเร็จ"}
```

---

## 4. Password Hashing ด้วย passlib

ก่อนอื่นติดตั้ง library:
```bash
pip install passlib[bcrypt]
```

```python
# password_hashing.py

from passlib.context import CryptContext

# สร้าง password context สำหรับ bcrypt
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    """
    Hash password ด้วย bcrypt
    bcrypt สร้าง salt อัตโนมัติ ทำให้ hash ต่างกันทุกครั้ง
    """
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    """
    ตรวจสอบว่า password ตรงกับ hash หรือไม่
    """
    return pwd_context.verify(plain_password, hashed_password)


# ทดสอบ
if __name__ == "__main__":
    password = "my_secret_password"

    # Hash password
    hashed = hash_password(password)
    print(f"Original: {password}")
    print(f"Hashed: {hashed}")

    # Verify
    is_correct = verify_password(password, hashed)
    print(f"Correct password: {is_correct}")

    is_wrong = verify_password("wrong_password", hashed)
    print(f"Wrong password: {is_wrong}")

    # ทดสอบ hash ต่างกันทุกครั้ง
    hash1 = hash_password(password)
    hash2 = hash_password(password)
    print(f"\nHash 1: {hash1[:30]}...")
    print(f"Hash 2: {hash2[:30]}...")
    print(f"Hashes are different: {hash1 != hash2}")
    print(f"But both verify correctly: {verify_password(password, hash1) and verify_password(password, hash2)}")
```

---

## 5. JWT Token ด้วย python-jose

ติดตั้ง:
```bash
pip install python-jose[cryptography]
```

```python
# jwt_utils.py

from jose import JWTError, jwt
from datetime import datetime, timedelta
from typing import Optional

# === Configuration ===
SECRET_KEY = "your-super-secret-key-change-this-in-production-12345"  # เปลี่ยนใน production!
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30
REFRESH_TOKEN_EXPIRE_DAYS = 7


def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """
    สร้าง JWT access token
    
    data: ข้อมูลที่ต้องการ encode (เช่น {"sub": "username"})
    expires_delta: เวลาหมดอายุ (ถ้าไม่กำหนดใช้ค่า default)
    """
    to_encode = data.copy()

    # กำหนดเวลาหมดอายุ
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)

    to_encode.update({"exp": expire})

    # Encode เป็น JWT string
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt


def create_refresh_token(data: dict) -> str:
    """สร้าง JWT refresh token (อายุนานกว่า access token)"""
    expires = timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)
    return create_access_token(data, expires_delta=expires)


def decode_token(token: str) -> dict:
    """
    Decode และ verify JWT token
    
    Returns: payload dict
    Raises: JWTError ถ้า token ไม่ถูกต้อง
    """
    payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
    return payload


def get_username_from_token(token: str) -> Optional[str]:
    """ดึง username จาก token"""
    try:
        payload = decode_token(token)
        username: str = payload.get("sub")
        return username
    except JWTError:
        return None


# ทดสอบ
if __name__ == "__main__":
    # สร้าง token
    token = create_access_token({"sub": "john_doe", "role": "user"})
    print(f"Access Token: {token[:50]}...")

    # Decode
    payload = decode_token(token)
    print(f"Payload: {payload}")

    # ดึง username
    username = get_username_from_token(token)
    print(f"Username: {username}")
```

---

## 6. OAuth2 Authentication พร้อม JWT

```python
# auth.py - Complete Authentication System

from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from pydantic import BaseModel
from datetime import datetime, timedelta
from typing import Optional

app = FastAPI(title="Authentication Demo")

# === Configuration ===
SECRET_KEY = "secret-key-please-change-in-production"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# OAuth2 scheme - ระบุ URL สำหรับรับ token
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


# === Models ===

class Token(BaseModel):
    access_token: str
    token_type: str
    expires_in: int


class TokenData(BaseModel):
    username: Optional[str] = None


class User(BaseModel):
    id: int
    username: str
    email: str
    full_name: Optional[str] = None
    is_active: bool = True
    role: str = "user"


class UserInDB(User):
    hashed_password: str


# === Mock Database ===
fake_users_db = {
    "alice": {
        "id": 1,
        "username": "alice",
        "email": "alice@example.com",
        "full_name": "Alice Wonderland",
        "hashed_password": pwd_context.hash("alice123"),
        "is_active": True,
        "role": "admin"
    },
    "bob": {
        "id": 2,
        "username": "bob",
        "email": "bob@example.com",
        "full_name": "Bob Builder",
        "hashed_password": pwd_context.hash("bob123"),
        "is_active": True,
        "role": "user"
    },
    "charlie": {
        "id": 3,
        "username": "charlie",
        "email": "charlie@example.com",
        "full_name": "Charlie Brown",
        "hashed_password": pwd_context.hash("charlie123"),
        "is_active": False,  # บัญชีถูกปิด
        "role": "user"
    }
}


# === Helper Functions ===

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """ตรวจสอบ password"""
    return pwd_context.verify(plain_password, hashed_password)


def get_password_hash(password: str) -> str:
    """Hash password"""
    return pwd_context.hash(password)


def get_user(username: str) -> Optional[UserInDB]:
    """ดึง user จาก database"""
    if username in fake_users_db:
        user_dict = fake_users_db[username]
        return UserInDB(**user_dict)
    return None


def authenticate_user(username: str, password: str) -> Optional[UserInDB]:
    """
    ตรวจสอบ username และ password
    Returns: UserInDB ถ้าถูกต้อง, None ถ้าไม่ถูกต้อง
    """
    user = get_user(username)
    if not user:
        return None
    if not verify_password(password, user.hashed_password):
        return None
    return user


def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """สร้าง JWT token"""
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)


# === Dependencies ===

async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    """
    Dependency สำหรับดึง current user จาก JWT token
    ใช้ใน endpoint ที่ต้องการ authentication
    """
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="ไม่สามารถ verify credentials ได้",
        headers={"WWW-Authenticate": "Bearer"},
    )

    try:
        # Decode JWT token
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
        token_data = TokenData(username=username)
    except JWTError:
        raise credentials_exception

    # ดึง user จาก database
    user = get_user(username=token_data.username)
    if user is None:
        raise credentials_exception

    return user


async def get_current_active_user(current_user: User = Depends(get_current_user)) -> User:
    """
    Dependency สำหรับตรวจสอบว่า user active อยู่
    """
    if not current_user.is_active:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="บัญชีผู้ใช้ถูกปิดการใช้งาน"
        )
    return current_user


# === Endpoints ===

@app.post("/token", response_model=Token)
async def login_for_access_token(
    form_data: OAuth2PasswordRequestForm = Depends()
):
    """
    Login และรับ JWT token
    
    OAuth2PasswordRequestForm จะรับ:
    - username (form field)
    - password (form field)
    """
    # ตรวจสอบ credentials
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง",
            headers={"WWW-Authenticate": "Bearer"},
        )

    # สร้าง access token
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user.username, "role": user.role},
        expires_delta=access_token_expires
    )

    return {
        "access_token": access_token,
        "token_type": "bearer",
        "expires_in": ACCESS_TOKEN_EXPIRE_MINUTES * 60  # วินาที
    }


@app.get("/users/me/", response_model=User)
async def read_users_me(current_user: User = Depends(get_current_active_user)):
    """
    ดึงข้อมูล user ปัจจุบัน
    ต้องส่ง Authorization: Bearer <token> header
    """
    return current_user


@app.get("/users/me/items/")
async def read_own_items(current_user: User = Depends(get_current_active_user)):
    """ดึงสินค้าของ user ปัจจุบัน"""
    return [
        {"id": 1, "name": "สินค้าของ " + current_user.username},
        {"id": 2, "name": "อีกสินค้าของ " + current_user.username}
    ]
```

ทดสอบ:
```bash
# Login
curl -X POST "http://localhost:8000/token" \
     -d "username=alice&password=alice123"

# ใช้ token
curl -H "Authorization: Bearer <your-token>" \
     "http://localhost:8000/users/me/"
```

---

## 7. Role-Based Access Control (RBAC)

```python
# rbac.py

from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from typing import List
from enum import Enum

app = FastAPI()

SECRET_KEY = "secret-key"
ALGORITHM = "HS256"
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")


# กำหนด roles
class Role(str, Enum):
    ADMIN = "admin"
    EDITOR = "editor"
    USER = "user"
    READONLY = "readonly"


# กำหนด permissions ของแต่ละ role
ROLE_PERMISSIONS = {
    Role.ADMIN: ["read", "write", "delete", "manage_users"],
    Role.EDITOR: ["read", "write"],
    Role.USER: ["read", "write_own"],
    Role.READONLY: ["read"]
}


# Mock user data
USERS = {
    "admin_user": {"id": 1, "username": "admin_user", "role": Role.ADMIN, "is_active": True},
    "editor_user": {"id": 2, "username": "editor_user", "role": Role.EDITOR, "is_active": True},
    "regular_user": {"id": 3, "username": "regular_user", "role": Role.USER, "is_active": True},
    "reader_user": {"id": 4, "username": "reader_user", "role": Role.READONLY, "is_active": True},
}


# === Permission Checker ===

def has_permission(role: Role, permission: str) -> bool:
    """ตรวจสอบว่า role มี permission หรือไม่"""
    return permission in ROLE_PERMISSIONS.get(role, [])


# === Dependencies ===

async def get_current_user_from_token(token: str = Depends(oauth2_scheme)) -> dict:
    """ดึง user จาก token"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username = payload.get("sub")
        if not username or username not in USERS:
            raise HTTPException(status_code=401, detail="Invalid token")
        return USERS[username]
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")


def require_permission(permission: str):
    """
    Factory function สร้าง dependency สำหรับ permission ที่ระบุ
    ใช้แบบนี้: Depends(require_permission("write"))
    """
    async def check_permission(user: dict = Depends(get_current_user_from_token)):
        if not has_permission(user["role"], permission):
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"ไม่มีสิทธิ์ {permission} - role ของคุณคือ {user['role']}"
            )
        return user
    return check_permission


def require_role(roles: List[Role]):
    """
    Factory function สร้าง dependency สำหรับ roles ที่ระบุ
    """
    async def check_role(user: dict = Depends(get_current_user_from_token)):
        if user["role"] not in roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Role {user['role']} ไม่ได้รับอนุญาต"
            )
        return user
    return check_role


# === Endpoints ===

@app.get("/articles/")
async def list_articles(user: dict = Depends(require_permission("read"))):
    """ทุก role อ่านได้"""
    return {"articles": ["บทความ 1", "บทความ 2"], "reader": user["username"]}


@app.post("/articles/")
async def create_article(user: dict = Depends(require_permission("write"))):
    """admin และ editor เขียนได้"""
    return {"message": "สร้างบทความสำเร็จ", "author": user["username"]}


@app.delete("/articles/{article_id}")
async def delete_article(
    article_id: int,
    user: dict = Depends(require_permission("delete"))
):
    """เฉพาะ admin ลบได้"""
    return {"message": f"ลบบทความ {article_id} สำเร็จ", "deleted_by": user["username"]}


@app.get("/admin/users/")
async def manage_users(user: dict = Depends(require_role([Role.ADMIN]))):
    """เฉพาะ admin จัดการ users ได้"""
    return {
        "message": "รายการ users",
        "managed_by": user["username"],
        "users": list(USERS.keys())
    }


@app.get("/dashboard/")
async def get_dashboard(
    user: dict = Depends(require_role([Role.ADMIN, Role.EDITOR]))
):
    """admin และ editor ดู dashboard ได้"""
    return {
        "dashboard": "ข้อมูล dashboard",
        "viewer": user["username"],
        "role": user["role"]
    }
```

---

## 8. API Key Authentication

```python
# api_key_auth.py

from fastapi import FastAPI, Security, HTTPException, status, Depends
from fastapi.security import APIKeyHeader, APIKeyQuery, APIKeyCookie
from typing import Optional
import secrets
import hashlib

app = FastAPI()


# === API Key via Header ===
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

# === API Key via Query Parameter ===
api_key_query = APIKeyQuery(name="api_key", auto_error=False)

# === API Key via Cookie ===
api_key_cookie = APIKeyCookie(name="session_key", auto_error=False)


# Mock API key database (ในระบบจริงเก็บใน database)
API_KEYS = {
    "ak_prod_abc123": {
        "client_name": "Production App",
        "rate_limit": 1000,
        "permissions": ["read", "write"],
        "is_active": True
    },
    "ak_test_xyz789": {
        "client_name": "Test App",
        "rate_limit": 100,
        "permissions": ["read"],
        "is_active": True
    },
    "ak_admin_secret": {
        "client_name": "Admin Dashboard",
        "rate_limit": 10000,
        "permissions": ["read", "write", "admin"],
        "is_active": True
    }
}


def validate_api_key(api_key: str) -> Optional[dict]:
    """ตรวจสอบ API key"""
    if api_key not in API_KEYS:
        return None
    key_info = API_KEYS[api_key]
    if not key_info["is_active"]:
        return None
    return key_info


async def get_api_key(
    header_key: Optional[str] = Security(api_key_header),
    query_key: Optional[str] = Security(api_key_query),
    cookie_key: Optional[str] = Security(api_key_cookie)
) -> dict:
    """
    รับ API key จาก header, query param, หรือ cookie
    ลองทีละอย่างตามลำดับ
    """
    # ลองจาก header ก่อน
    if header_key:
        key_info = validate_api_key(header_key)
        if key_info:
            return {**key_info, "key": header_key, "source": "header"}

    # ลองจาก query param
    if query_key:
        key_info = validate_api_key(query_key)
        if key_info:
            return {**key_info, "key": query_key, "source": "query"}

    # ลองจาก cookie
    if cookie_key:
        key_info = validate_api_key(cookie_key)
        if key_info:
            return {**key_info, "key": cookie_key, "source": "cookie"}

    raise HTTPException(
        status_code=status.HTTP_403_FORBIDDEN,
        detail="API key ไม่ถูกต้อง หรือไม่ได้ระบุ"
    )


def require_api_permission(permission: str):
    """Factory สำหรับตรวจสอบ permission ของ API key"""
    async def check_permission(key_info: dict = Depends(get_api_key)):
        if permission not in key_info["permissions"]:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"API key ไม่มีสิทธิ์ {permission}"
            )
        return key_info
    return check_permission


# === Endpoints ===

@app.get("/api/data/")
async def get_data(key_info: dict = Depends(get_api_key)):
    """ต้องการ API key ที่ถูกต้อง"""
    return {
        "data": "ข้อมูลที่ป้องกัน",
        "client": key_info["client_name"],
        "key_source": key_info["source"]
    }


@app.post("/api/data/")
async def create_data(key_info: dict = Depends(require_api_permission("write"))):
    """ต้องการ write permission"""
    return {
        "message": "สร้างข้อมูลสำเร็จ",
        "client": key_info["client_name"]
    }


@app.get("/api/admin/")
async def admin_endpoint(key_info: dict = Depends(require_api_permission("admin"))):
    """เฉพาะ admin API key"""
    return {
        "message": "Admin endpoint",
        "client": key_info["client_name"],
        "all_keys": list(API_KEYS.keys())  # admin เห็นได้
    }


# สร้าง API key ใหม่ (ในระบบจริงต้อง authenticate ก่อน)
@app.post("/api/keys/generate/")
def generate_api_key(client_name: str, permissions: list = ["read"]):
    """สร้าง API key ใหม่"""
    # สร้าง random key
    new_key = f"ak_{secrets.token_urlsafe(16)}"

    # เก็บใน database
    API_KEYS[new_key] = {
        "client_name": client_name,
        "rate_limit": 100,
        "permissions": permissions,
        "is_active": True
    }

    return {
        "api_key": new_key,
        "client_name": client_name,
        "permissions": permissions,
        "warning": "เก็บ API key นี้ไว้ให้ดี จะไม่แสดงอีก!"
    }
```

---

## 9. ตัวอย่างสมบูรณ์: Auth System

```python
# complete_auth_system.py - ระบบ Authentication สมบูรณ์

from fastapi import FastAPI, Depends, HTTPException, status, BackgroundTasks
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from pydantic import BaseModel, EmailStr
from datetime import datetime, timedelta
from typing import Optional, List
import secrets

app = FastAPI(title="Complete Auth System")

# Config
SECRET_KEY = secrets.token_urlsafe(32)  # สุ่ม key ทุกครั้ง restart
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30
REFRESH_TOKEN_EXPIRE_DAYS = 7

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/login")

# Mock DB
users_db = {}
refresh_tokens_db = {}  # เก็บ refresh tokens ที่ยังใช้ได้


# === Schemas ===

class UserRegister(BaseModel):
    username: str
    email: str
    password: str
    full_name: str


class UserLogin(BaseModel):
    username: str
    password: str


class UserOut(BaseModel):
    id: str
    username: str
    email: str
    full_name: str
    role: str
    is_active: bool
    created_at: datetime


class TokenPair(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int


class RefreshTokenRequest(BaseModel):
    refresh_token: str


class PasswordChange(BaseModel):
    current_password: str
    new_password: str
    confirm_password: str


# === Auth Functions ===

def create_tokens(user_id: str, username: str, role: str) -> TokenPair:
    """สร้าง access + refresh token"""
    access_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    refresh_expires = timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS)

    access_payload = {
        "sub": username,
        "user_id": user_id,
        "role": role,
        "type": "access",
        "exp": datetime.utcnow() + access_expires
    }

    refresh_payload = {
        "sub": username,
        "user_id": user_id,
        "type": "refresh",
        "exp": datetime.utcnow() + refresh_expires
    }

    access_token = jwt.encode(access_payload, SECRET_KEY, ALGORITHM)
    refresh_token = jwt.encode(refresh_payload, SECRET_KEY, ALGORITHM)

    # เก็บ refresh token
    refresh_tokens_db[refresh_token] = {
        "user_id": user_id,
        "expires_at": datetime.utcnow() + refresh_expires
    }

    return TokenPair(
        access_token=access_token,
        refresh_token=refresh_token,
        expires_in=ACCESS_TOKEN_EXPIRE_MINUTES * 60
    )


async def get_current_user(token: str = Depends(oauth2_scheme)) -> dict:
    """ดึง current user จาก JWT"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        if payload.get("type") != "access":
            raise HTTPException(status_code=401, detail="ประเภท token ไม่ถูกต้อง")
        username = payload.get("sub")
        user_id = payload.get("user_id")
        if not username or not user_id:
            raise HTTPException(status_code=401, detail="Token ไม่ถูกต้อง")
    except JWTError:
        raise HTTPException(status_code=401, detail="Token ไม่ถูกต้องหรือหมดอายุ")

    if user_id not in users_db:
        raise HTTPException(status_code=401, detail="ไม่พบผู้ใช้")

    user = users_db[user_id]
    if not user["is_active"]:
        raise HTTPException(status_code=400, detail="บัญชีถูกปิดการใช้งาน")

    return user


# === Auth Endpoints ===

@app.post("/auth/register", response_model=UserOut, status_code=201)
def register(user_data: UserRegister):
    """ลงทะเบียนผู้ใช้ใหม่"""
    # ตรวจสอบ username ซ้ำ
    for user in users_db.values():
        if user["username"] == user_data.username:
            raise HTTPException(status_code=400, detail="Username นี้ถูกใช้แล้ว")
        if user["email"] == user_data.email:
            raise HTTPException(status_code=400, detail="Email นี้ถูกใช้แล้ว")

    user_id = secrets.token_urlsafe(8)
    new_user = {
        "id": user_id,
        "username": user_data.username,
        "email": user_data.email,
        "full_name": user_data.full_name,
        "hashed_password": pwd_context.hash(user_data.password),
        "role": "user",
        "is_active": True,
        "created_at": datetime.utcnow()
    }
    users_db[user_id] = new_user
    return new_user


@app.post("/auth/login", response_model=TokenPair)
def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """Login และรับ token"""
    # หา user
    user = None
    for u in users_db.values():
        if u["username"] == form_data.username:
            user = u
            break

    if not user or not pwd_context.verify(form_data.password, user["hashed_password"]):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง"
        )

    if not user["is_active"]:
        raise HTTPException(status_code=400, detail="บัญชีถูกปิดการใช้งาน")

    return create_tokens(user["id"], user["username"], user["role"])


@app.post("/auth/refresh", response_model=TokenPair)
def refresh_token(request: RefreshTokenRequest):
    """รับ access token ใหม่ด้วย refresh token"""
    try:
        payload = jwt.decode(request.refresh_token, SECRET_KEY, algorithms=[ALGORITHM])
        if payload.get("type") != "refresh":
            raise HTTPException(status_code=400, detail="ต้องเป็น refresh token")

        # ตรวจสอบว่า refresh token ยังใช้ได้
        if request.refresh_token not in refresh_tokens_db:
            raise HTTPException(status_code=400, detail="Refresh token ไม่ถูกต้องหรือถูก revoke แล้ว")

    except JWTError:
        raise HTTPException(status_code=400, detail="Refresh token หมดอายุหรือไม่ถูกต้อง")

    user_id = payload.get("user_id")
    if user_id not in users_db:
        raise HTTPException(status_code=400, detail="ไม่พบผู้ใช้")

    user = users_db[user_id]

    # Revoke refresh token เก่า
    del refresh_tokens_db[request.refresh_token]

    return create_tokens(user["id"], user["username"], user["role"])


@app.post("/auth/logout")
def logout(
    request: RefreshTokenRequest,
    current_user: dict = Depends(get_current_user)
):
    """Logout - revoke refresh token"""
    if request.refresh_token in refresh_tokens_db:
        del refresh_tokens_db[request.refresh_token]
    return {"message": "Logout สำเร็จ"}


@app.get("/users/me", response_model=UserOut)
def get_me(current_user: dict = Depends(get_current_user)):
    """ดูข้อมูลตัวเอง"""
    return current_user


@app.put("/users/me/password")
def change_password(
    password_data: PasswordChange,
    current_user: dict = Depends(get_current_user)
):
    """เปลี่ยนรหัสผ่าน"""
    if password_data.new_password != password_data.confirm_password:
        raise HTTPException(status_code=400, detail="รหัสผ่านใหม่ไม่ตรงกัน")

    if not pwd_context.verify(password_data.current_password, current_user["hashed_password"]):
        raise HTTPException(status_code=400, detail="รหัสผ่านปัจจุบันไม่ถูกต้อง")

    # อัปเดต password
    current_user["hashed_password"] = pwd_context.hash(password_data.new_password)
    return {"message": "เปลี่ยนรหัสผ่านสำเร็จ"}
```

---

## 10. สรุป Part 089

✅ **Depends()** - inject dependencies เข้า endpoint โดยอัตโนมัติ, reusable และ testable

✅ **Layered Dependencies** - dependencies ที่ซ้อนกัน เช่น check_header → verify_key → check_permission

✅ **Database Session** - ใช้ `yield` ใน dependency เพื่อจัดการ session lifecycle อัตโนมัติ

✅ **passlib** - hash password ด้วย bcrypt อย่างปลอดภัย

✅ **python-jose** - สร้างและ verify JWT token สำหรับ stateless authentication

✅ **OAuth2PasswordBearer** - รับ Bearer token จาก Authorization header

✅ **get_current_user** - dependency pattern สำหรับ protected endpoints

✅ **RBAC** - Role-Based Access Control ด้วย `require_permission()` factory function

✅ **API Key Auth** - รับ API key จาก header, query param, หรือ cookie

✅ **Refresh Token** - pattern สำหรับ token rotation และ long-lived sessions

---

## ➡️ ถัดไป: Part 090 - FastAPI Advanced Features

ใน Part ถัดไปเราจะเรียนรู้:
- Custom exception handlers
- Middleware สำหรับ logging และ timing
- Background tasks
- Lifespan events (startup/shutdown)
- Custom response classes
- Server-Sent Events (SSE)

*Part 089/100+ | Python Course - Beginner to World-Class*
