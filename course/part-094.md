# Part 094: FastAPI Testing
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- ใช้ TestClient ทดสอบ FastAPI endpoints
- สร้าง pytest fixtures สำหรับ FastAPI
- Mock database ใน tests
- ทดสอบ authentication
- เขียน integration tests

---

## 1. Setup Testing

```bash
pip install pytest pytest-asyncio httpx
```

### โครงสร้าง tests
```
my_app/
├── app/
│   ├── main.py
│   ├── models.py
│   ├── database.py
│   └── routers/
└── tests/
    ├── __init__.py
    ├── conftest.py
    ├── test_users.py
    ├── test_posts.py
    └── test_auth.py
```

---

## 2. TestClient พื้นฐาน

```python
# tests/test_basic.py

from fastapi import FastAPI
from fastapi.testclient import TestClient

# สร้าง app สำหรับ test
app = FastAPI()


@app.get("/")
def root():
    return {"message": "Hello World"}


@app.get("/items/{item_id}")
def get_item(item_id: int):
    if item_id <= 0:
        from fastapi import HTTPException
        raise HTTPException(404, "ไม่พบ")
    return {"id": item_id, "name": f"Item {item_id}"}


@app.post("/items")
def create_item(name: str, price: float):
    return {"id": 1, "name": name, "price": price}


# สร้าง TestClient
client = TestClient(app)


# Tests
def test_root():
    """ทดสอบ GET /"""
    response = client.get("/")
    
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}


def test_get_item_success():
    """ทดสอบ GET /items/1"""
    response = client.get("/items/1")
    
    assert response.status_code == 200
    data = response.json()
    assert data["id"] == 1
    assert "name" in data


def test_get_item_not_found():
    """ทดสอบ 404"""
    response = client.get("/items/0")
    
    assert response.status_code == 404
    assert "ไม่พบ" in response.json()["detail"]


def test_create_item():
    """ทดสอบ POST /items"""
    response = client.post("/items?name=Laptop&price=45000")
    
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "Laptop"
    assert data["price"] == 45000.0
```

---

## 3. pytest Fixtures

```python
# tests/conftest.py

import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import StaticPool

from app.main import app
from app.database import Base, get_db
from app.models import User


# ---- Database Fixtures ----

@pytest.fixture(scope="session")
def engine():
    """สร้าง in-memory SQLite database สำหรับ tests"""
    engine = create_engine(
        "sqlite:///:memory:",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,  # ใช้ single connection ป้องกัน issues
    )
    Base.metadata.create_all(bind=engine)
    yield engine
    Base.metadata.drop_all(bind=engine)


@pytest.fixture(scope="function")
def db_session(engine):
    """Database session สำหรับแต่ละ test"""
    TestingSessionLocal = sessionmaker(
        autocommit=False,
        autoflush=False,
        bind=engine
    )
    session = TestingSessionLocal()
    
    try:
        yield session
    finally:
        session.rollback()  # ยกเลิกทุกการเปลี่ยนแปลง
        session.close()


# ---- Client Fixtures ----

@pytest.fixture(scope="function")
def client(db_session):
    """TestClient ที่ใช้ test database"""
    
    def override_get_db():
        yield db_session
    
    # Override dependency
    app.dependency_overrides[get_db] = override_get_db
    
    with TestClient(app) as c:
        yield c
    
    # ลบ override
    app.dependency_overrides.clear()


# ---- Data Fixtures ----

@pytest.fixture
def test_user(db_session):
    """สร้าง test user"""
    from app.auth import get_password_hash
    
    user = User(
        username="testuser",
        email="test@example.com",
        hashed_password=get_password_hash("testpassword123"),
        is_active=True
    )
    db_session.add(user)
    db_session.commit()
    db_session.refresh(user)
    return user


@pytest.fixture
def test_admin(db_session):
    """สร้าง admin user"""
    from app.auth import get_password_hash
    
    admin = User(
        username="admin",
        email="admin@example.com",
        hashed_password=get_password_hash("adminpassword123"),
        is_active=True,
        is_admin=True
    )
    db_session.add(admin)
    db_session.commit()
    db_session.refresh(admin)
    return admin


@pytest.fixture
def auth_headers(client, test_user):
    """Headers พร้อม JWT token สำหรับ test_user"""
    response = client.post("/auth/token", data={
        "username": test_user.username,
        "password": "testpassword123"
    })
    token = response.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}


@pytest.fixture
def admin_headers(client, test_admin):
    """Headers พร้อม JWT token สำหรับ admin"""
    response = client.post("/auth/token", data={
        "username": test_admin.username,
        "password": "adminpassword123"
    })
    token = response.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}
```

---

## 4. ทดสอบ CRUD Endpoints

```python
# tests/test_users.py

import pytest


class TestCreateUser:
    """ทดสอบ POST /users"""
    
    def test_create_user_success(self, client):
        """สร้าง user สำเร็จ"""
        response = client.post("/users", json={
            "username": "newuser",
            "email": "new@example.com",
            "password": "securepassword123"
        })
        
        assert response.status_code == 201
        data = response.json()
        assert data["username"] == "newuser"
        assert data["email"] == "new@example.com"
        assert "password" not in data
        assert "id" in data
    
    def test_create_user_duplicate_username(self, client, test_user):
        """สร้าง user ที่ชื่อซ้ำ"""
        response = client.post("/users", json={
            "username": test_user.username,  # ชื่อซ้ำ
            "email": "other@example.com",
            "password": "password123"
        })
        
        assert response.status_code == 409
        assert "มีอยู่แล้ว" in response.json()["detail"]
    
    def test_create_user_invalid_email(self, client):
        """email ไม่ถูกต้อง"""
        response = client.post("/users", json={
            "username": "user123",
            "email": "not-an-email",
            "password": "password123"
        })
        
        assert response.status_code == 422  # Validation error
    
    def test_create_user_short_password(self, client):
        """password สั้นเกินไป"""
        response = client.post("/users", json={
            "username": "user123",
            "email": "user@example.com",
            "password": "short"  # น้อยกว่า 8 ตัว
        })
        
        assert response.status_code == 422
    
    def test_create_user_missing_fields(self, client):
        """ข้อมูลไม่ครบ"""
        response = client.post("/users", json={"username": "user"})
        assert response.status_code == 422


class TestGetUser:
    """ทดสอบ GET /users"""
    
    def test_get_user_success(self, client, test_user):
        """ดึง user สำเร็จ"""
        response = client.get(f"/users/{test_user.id}")
        
        assert response.status_code == 200
        data = response.json()
        assert data["id"] == test_user.id
        assert data["username"] == test_user.username
    
    def test_get_user_not_found(self, client):
        """ดึง user ที่ไม่มี"""
        response = client.get("/users/99999")
        assert response.status_code == 404
    
    def test_get_users_list(self, client, test_user, test_admin):
        """ดึงรายการ users"""
        response = client.get("/users")
        
        assert response.status_code == 200
        data = response.json()
        assert isinstance(data, list)
        assert len(data) >= 2
```

---

## 5. ทดสอบ Authentication

```python
# tests/test_auth.py

import pytest


class TestLogin:
    """ทดสอบ Login"""
    
    def test_login_success(self, client, test_user):
        """Login สำเร็จ"""
        response = client.post("/auth/token", data={
            "username": test_user.username,
            "password": "testpassword123"
        })
        
        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert "refresh_token" in data
        assert data["token_type"] == "bearer"
    
    def test_login_wrong_password(self, client, test_user):
        """รหัสผ่านผิด"""
        response = client.post("/auth/token", data={
            "username": test_user.username,
            "password": "wrongpassword"
        })
        
        assert response.status_code == 401
    
    def test_login_wrong_username(self, client):
        """ชื่อผู้ใช้ไม่มีในระบบ"""
        response = client.post("/auth/token", data={
            "username": "nonexistent",
            "password": "anypassword"
        })
        
        assert response.status_code == 401
    
    def test_login_missing_fields(self, client):
        """ข้อมูลไม่ครบ"""
        response = client.post("/auth/token", data={"username": "user"})
        assert response.status_code == 422


class TestProtectedEndpoints:
    """ทดสอบ endpoints ที่ต้องการ authentication"""
    
    def test_get_me_with_token(self, client, auth_headers):
        """GET /users/me สำเร็จ"""
        response = client.get("/users/me", headers=auth_headers)
        assert response.status_code == 200
    
    def test_get_me_without_token(self, client):
        """GET /users/me ไม่มี token"""
        response = client.get("/users/me")
        assert response.status_code == 401
    
    def test_get_me_invalid_token(self, client):
        """GET /users/me token ไม่ถูกต้อง"""
        response = client.get("/users/me", headers={
            "Authorization": "Bearer invalid-token-here"
        })
        assert response.status_code == 401
    
    def test_admin_endpoint_as_user(self, client, auth_headers):
        """Admin endpoint ด้วย user token ปกติ"""
        response = client.get("/admin/users", headers=auth_headers)
        assert response.status_code == 403
    
    def test_admin_endpoint_as_admin(self, client, admin_headers):
        """Admin endpoint ด้วย admin token"""
        response = client.get("/admin/users", headers=admin_headers)
        assert response.status_code == 200
```

---

## 6. Mock Dependencies

```python
# tests/test_with_mocks.py

import pytest
from unittest.mock import patch, AsyncMock, MagicMock
from fastapi.testclient import TestClient
from app.main import app


class TestEmailMocking:
    """ทดสอบโดย mock email service"""
    
    @patch("app.services.email.send_email")
    def test_register_sends_welcome_email(self, mock_send, client):
        """ตรวจสอบว่าส่งอีเมลต้อนรับ"""
        mock_send.return_value = True
        
        response = client.post("/auth/register", json={
            "username": "emailtest",
            "email": "emailtest@example.com",
            "password": "password123456"
        })
        
        assert response.status_code == 201
        
        # ตรวจสอบว่า send_email ถูกเรียก
        mock_send.assert_called_once()
        call_args = mock_send.call_args
        assert "emailtest@example.com" in str(call_args)
    
    @patch("app.services.email.send_email")
    def test_register_handles_email_failure(self, mock_send, client):
        """ต้องสำเร็จแม้อีเมลจะส่งไม่ได้"""
        mock_send.side_effect = Exception("Email service down")
        
        response = client.post("/auth/register", json={
            "username": "emailfail",
            "email": "fail@example.com",
            "password": "password123456"
        })
        
        # User ควรถูกสร้างแม้ email จะล้มเหลว
        assert response.status_code == 201


def test_override_dependency(db_session):
    """ทดสอบโดย override dependency"""
    from app.database import get_db
    from app.dependencies import get_current_user
    from app.models import User
    
    # Mock current user
    mock_user = User(id=1, username="mockuser", email="mock@example.com")
    
    def mock_get_current_user():
        return mock_user
    
    app.dependency_overrides[get_current_user] = mock_get_current_user
    
    with TestClient(app) as client:
        response = client.get("/users/me")
        assert response.status_code == 200
        assert response.json()["username"] == "mockuser"
    
    # ล้าง override
    del app.dependency_overrides[get_current_user]
```

---

## 7. Async Tests

```python
# tests/test_async.py

import pytest
import pytest_asyncio
from httpx import AsyncClient
from app.main import app


@pytest.mark.asyncio
async def test_async_endpoint():
    """ทดสอบ async endpoint ด้วย AsyncClient"""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/")
    
    assert response.status_code == 200


@pytest.mark.asyncio
async def test_create_and_get_user():
    """ทดสอบ create แล้ว get"""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        # สร้าง user
        create_response = await ac.post("/users", json={
            "username": "asyncuser",
            "email": "async@example.com",
            "password": "password123456"
        })
        assert create_response.status_code == 201
        user_id = create_response.json()["id"]
        
        # ดึง user
        get_response = await ac.get(f"/users/{user_id}")
        assert get_response.status_code == 200
        assert get_response.json()["username"] == "asyncuser"
```

---

## 8. pytest.ini Configuration

```ini
# pytest.ini

[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
asyncio_mode = auto

# Markers
markers =
    slow: marks tests as slow
    integration: marks integration tests
    unit: marks unit tests
```

### รัน tests
```bash
# รัน tests ทั้งหมด
pytest

# รัน พร้อม coverage
pytest --cov=app --cov-report=html

# รัน เฉพาะ file
pytest tests/test_users.py

# รัน เฉพาะ test
pytest tests/test_users.py::TestCreateUser::test_create_user_success

# รัน verbose
pytest -v

# รัน เฉพาะ marker
pytest -m "not slow"
```

---

## 9. สรุป Part 094

✅ **TestClient** จาก `fastapi.testclient` สำหรับ synchronous tests  
✅ **AsyncClient** จาก `httpx` สำหรับ async tests  
✅ **conftest.py** กำหนด fixtures ที่ใช้ร่วมกัน  
✅ **Database override** ใช้ in-memory SQLite ใน tests  
✅ **Dependency override** แทนที่ dependencies ด้วย mock  
✅ **Mock** ใช้ `patch()` สำหรับ external services  
✅ **pytest markers** จัดกลุ่ม tests  

---

## ➡️ ถัดไป: Part 095 - FastAPI WebSockets

*Part 094/100+ | Python Course - Beginner to World-Class*
