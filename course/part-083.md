# Part 083: Flask Testing
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- เขียน unit tests ด้วย unittest สำหรับ Flask
- ใช้ Flask Test Client ทดสอบ routes
- ใช้ pytest และ pytest-flask
- Mock dependencies ด้วย unittest.mock
- ทดสอบ database operations

---

## 1. การทดสอบ Flask Application

### ทำไมต้อง Test?
- ค้นหา bugs ก่อน deploy
- มั่นใจว่า feature ทำงานถูกต้องเมื่อแก้โค้ด
- เป็น documentation ว่าแต่ละส่วนทำงานอย่างไร

### ติดตั้ง
```bash
pip install pytest pytest-flask pytest-cov
```

### โครงสร้าง Tests
```
my_app/
├── app/
│   ├── __init__.py
│   ├── models.py
│   └── routes.py
└── tests/
    ├── __init__.py
    ├── conftest.py          # Fixtures
    ├── test_auth.py
    ├── test_blog.py
    └── test_api.py
```

---

## 2. Setup ด้วย unittest

```python
# tests/test_basic.py

import unittest
from app import create_app
from app.models import db, User


class BasicTestCase(unittest.TestCase):
    """Test cases พื้นฐานด้วย unittest"""
    
    def setUp(self):
        """ทำงานก่อนทุก test method"""
        # สร้าง app สำหรับ testing
        self.app = create_app('testing')
        self.client = self.app.test_client()
        self.ctx = self.app.app_context()
        self.ctx.push()
        
        # สร้าง tables ใหม่สำหรับแต่ละ test
        db.create_all()
    
    def tearDown(self):
        """ทำงานหลังทุก test method"""
        db.session.remove()
        db.drop_all()
        self.ctx.pop()
    
    def test_app_exists(self):
        """ทดสอบว่า app สร้างได้"""
        self.assertIsNotNone(self.app)
    
    def test_app_is_testing(self):
        """ทดสอบว่า app อยู่ใน testing mode"""
        self.assertTrue(self.app.config['TESTING'])
    
    def test_home_page(self):
        """ทดสอบหน้าแรก"""
        response = self.client.get('/')
        self.assertEqual(response.status_code, 200)
    
    def test_404_page(self):
        """ทดสอบหน้า 404"""
        response = self.client.get('/page-that-does-not-exist')
        self.assertEqual(response.status_code, 404)


if __name__ == '__main__':
    unittest.main()
```

---

## 3. pytest Fixtures

```python
# tests/conftest.py — pytest Fixtures

import pytest
from app import create_app
from app.models import db as _db, User, Post


@pytest.fixture(scope='session')
def app():
    """สร้าง Flask app สำหรับ test session ทั้งหมด"""
    app = create_app('testing')
    
    with app.app_context():
        yield app


@pytest.fixture(scope='session')
def db(app):
    """สร้าง database สำหรับ test session"""
    _db.create_all()
    yield _db
    _db.drop_all()


@pytest.fixture(scope='function')
def db_session(db):
    """Transaction-based rollback สำหรับแต่ละ test"""
    connection = db.engine.connect()
    transaction = connection.begin()
    
    db.session.bind = connection
    
    yield db.session
    
    db.session.remove()
    transaction.rollback()
    connection.close()


@pytest.fixture
def client(app):
    """Flask test client"""
    return app.test_client()


@pytest.fixture
def runner(app):
    """Flask CLI test runner"""
    return app.test_cli_runner()


@pytest.fixture
def test_user(db_session):
    """สร้าง user สำหรับ testing"""
    user = User(
        username='testuser',
        email='test@example.com'
    )
    user.set_password('testpassword123')
    db_session.add(user)
    db_session.commit()
    return user


@pytest.fixture
def test_admin(db_session):
    """สร้าง admin user สำหรับ testing"""
    admin = User(
        username='admin',
        email='admin@example.com',
        is_admin=True
    )
    admin.set_password('adminpassword123')
    db_session.add(admin)
    db_session.commit()
    return admin


@pytest.fixture
def auth_client(client, test_user):
    """Test client ที่ login แล้ว"""
    client.post('/auth/login', data={
        'username': test_user.username,
        'password': 'testpassword123'
    })
    return client


@pytest.fixture
def test_post(db_session, test_user):
    """สร้าง post สำหรับ testing"""
    post = Post(
        title='Test Post',
        content='This is test content',
        user_id=test_user.id,
        published=True
    )
    db_session.add(post)
    db_session.commit()
    return post
```

---

## 4. ทดสอบ Routes

```python
# tests/test_routes.py

import pytest
from flask import url_for


class TestMainRoutes:
    """ทดสอบ main routes"""
    
    def test_index_page(self, client):
        """ทดสอบหน้าแรก"""
        response = client.get('/')
        assert response.status_code == 200
    
    def test_about_page(self, client):
        """ทดสอบหน้า about"""
        response = client.get('/about')
        assert response.status_code == 200
    
    def test_contact_page_get(self, client):
        """ทดสอบ GET /contact"""
        response = client.get('/contact')
        assert response.status_code == 200
        assert b'ติดต่อ' in response.data  # ตรวจสอบ content
    
    def test_404(self, client):
        """ทดสอบ 404"""
        response = client.get('/nonexistent-page')
        assert response.status_code == 404


class TestAuthRoutes:
    """ทดสอบ auth routes"""
    
    def test_login_page_get(self, client):
        """ทดสอบหน้า login"""
        response = client.get('/auth/login')
        assert response.status_code == 200
    
    def test_login_success(self, client, test_user):
        """ทดสอบ login สำเร็จ"""
        response = client.post('/auth/login', data={
            'username': test_user.username,
            'password': 'testpassword123',
        }, follow_redirects=True)
        
        assert response.status_code == 200
        # ตรวจสอบว่า redirect ไปหน้าหลัก
        # assert b'Dashboard' in response.data
    
    def test_login_wrong_password(self, client, test_user):
        """ทดสอบ login รหัสผ่านผิด"""
        response = client.post('/auth/login', data={
            'username': test_user.username,
            'password': 'wrongpassword',
        })
        
        assert response.status_code == 200  # ไม่ redirect
        assert b'ไม่ถูกต้อง' in response.data
    
    def test_login_wrong_username(self, client):
        """ทดสอบ login ชื่อผู้ใช้ไม่มีในระบบ"""
        response = client.post('/auth/login', data={
            'username': 'nonexistent',
            'password': 'password',
        })
        assert response.status_code == 200
        assert b'ไม่ถูกต้อง' in response.data
    
    def test_logout(self, auth_client):
        """ทดสอบ logout"""
        response = auth_client.get('/auth/logout', follow_redirects=True)
        assert response.status_code == 200
    
    def test_register_success(self, client, db_session):
        """ทดสอบสมัครสมาชิกสำเร็จ"""
        response = client.post('/auth/register', data={
            'username': 'newuser',
            'email': 'new@example.com',
            'password': 'Password123!',
            'confirm_password': 'Password123!'
        }, follow_redirects=True)
        
        assert response.status_code == 200
        
        # ตรวจสอบว่า user ถูกสร้างใน database
        from app.models import User
        user = User.query.filter_by(username='newuser').first()
        assert user is not None
        assert user.email == 'new@example.com'
    
    def test_protected_route_redirect(self, client):
        """ทดสอบว่า route ที่ต้อง login redirect ไหน"""
        response = client.get('/dashboard')
        assert response.status_code == 302
        assert '/auth/login' in response.location
    
    def test_protected_route_logged_in(self, auth_client):
        """ทดสอบ route ที่ต้อง login เมื่อ login แล้ว"""
        response = auth_client.get('/dashboard')
        assert response.status_code == 200
```

---

## 5. ทดสอบ API Endpoints

```python
# tests/test_api.py

import pytest
import json
from app.models import User, Post


class TestUserAPI:
    """ทดสอบ User API"""
    
    def test_get_users(self, client):
        """GET /api/users"""
        response = client.get('/api/users')
        assert response.status_code == 200
        
        data = json.loads(response.data)
        assert isinstance(data, list)
    
    def test_get_user_success(self, client, test_user):
        """GET /api/users/<id> สำเร็จ"""
        response = client.get(f'/api/users/{test_user.id}')
        assert response.status_code == 200
        
        data = json.loads(response.data)
        assert data['id'] == test_user.id
        assert data['username'] == test_user.username
        assert 'password' not in data  # ไม่ควรส่ง password
    
    def test_get_user_not_found(self, client):
        """GET /api/users/<id> ไม่พบ"""
        response = client.get('/api/users/9999')
        assert response.status_code == 404
    
    def test_create_user(self, client):
        """POST /api/users สร้าง user"""
        response = client.post(
            '/api/users',
            data=json.dumps({
                'username': 'apiuser',
                'email': 'apiuser@example.com',
                'password': 'Password123!'
            }),
            content_type='application/json'
        )
        
        assert response.status_code == 201
        data = json.loads(response.data)
        assert data['username'] == 'apiuser'
    
    def test_create_user_duplicate_username(self, client, test_user):
        """POST /api/users ชื่อซ้ำ"""
        response = client.post(
            '/api/users',
            data=json.dumps({
                'username': test_user.username,  # ชื่อซ้ำ
                'email': 'different@example.com',
                'password': 'Password123!'
            }),
            content_type='application/json'
        )
        
        assert response.status_code == 409  # Conflict
    
    def test_create_user_missing_fields(self, client):
        """POST /api/users ข้อมูลไม่ครบ"""
        response = client.post(
            '/api/users',
            data=json.dumps({'username': 'test'}),  # ขาด email, password
            content_type='application/json'
        )
        
        assert response.status_code in [400, 422]
    
    def test_update_user(self, client, test_user, auth_headers):
        """PUT /api/users/<id>"""
        response = client.put(
            f'/api/users/{test_user.id}',
            data=json.dumps({'email': 'updated@example.com'}),
            content_type='application/json',
            headers=auth_headers
        )
        
        assert response.status_code == 200
        data = json.loads(response.data)
        assert data['email'] == 'updated@example.com'
    
    def test_delete_user(self, client, test_user, auth_headers):
        """DELETE /api/users/<id>"""
        response = client.delete(
            f'/api/users/{test_user.id}',
            headers=auth_headers
        )
        
        assert response.status_code == 204
        
        # ตรวจสอบว่าถูกลบจริง
        deleted_response = client.get(f'/api/users/{test_user.id}')
        assert deleted_response.status_code == 404


@pytest.fixture
def auth_headers(client, test_user):
    """Headers พร้อม JWT token"""
    response = client.post(
        '/api/auth/login',
        data=json.dumps({
            'username': test_user.username,
            'password': 'testpassword123'
        }),
        content_type='application/json'
    )
    data = json.loads(response.data)
    token = data.get('access_token', '')
    return {'Authorization': f'Bearer {token}'}
```

---

## 6. Mock ด้วย unittest.mock

```python
# tests/test_with_mocks.py

import pytest
from unittest.mock import patch, MagicMock, Mock


class TestEmailService:
    """ทดสอบการส่งอีเมลโดย mock"""
    
    @patch('app.email.send_email')  # mock function send_email
    def test_register_sends_email(self, mock_send_email, client):
        """ทดสอบว่าสมัครสมาชิกแล้วส่งอีเมลยืนยัน"""
        # กำหนดให้ mock return ค่า
        mock_send_email.return_value = True
        
        response = client.post('/auth/register', data={
            'username': 'emailuser',
            'email': 'emailuser@example.com',
            'password': 'Password123!',
            'confirm_password': 'Password123!'
        })
        
        # ตรวจสอบว่า send_email ถูกเรียก
        assert mock_send_email.called
        
        # ตรวจสอบ arguments ที่ส่งไป
        call_args = mock_send_email.call_args
        assert 'emailuser@example.com' in str(call_args)
    
    @patch('app.services.payment.charge_card')
    def test_checkout_payment_failure(self, mock_charge, client, auth_client):
        """ทดสอบเมื่อ payment ล้มเหลว"""
        # กำหนดให้ mock raise exception
        mock_charge.side_effect = Exception('Payment failed')
        
        response = auth_client.post('/shop/checkout', data={
            'card_number': '4111111111111111',
            'amount': '100.00'
        })
        
        assert response.status_code == 400
    
    @patch('app.services.weather.get_weather')
    def test_dashboard_with_weather(self, mock_weather, auth_client):
        """ทดสอบ dashboard ที่แสดงอากาศ"""
        # Mock return value
        mock_weather.return_value = {
            'temperature': 32,
            'condition': 'Sunny',
            'city': 'Bangkok'
        }
        
        response = auth_client.get('/dashboard')
        assert response.status_code == 200
        assert b'32' in response.data  # ตรวจว่าแสดงอุณหภูมิ


class TestExternalAPI:
    """ทดสอบโค้ดที่เรียก external API"""
    
    @patch('requests.get')
    def test_fetch_exchange_rate(self, mock_get):
        """Mock requests.get"""
        # กำหนด mock response
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
            'rates': {'USD': 0.028, 'EUR': 0.026}
        }
        mock_get.return_value = mock_response
        
        from app.services.currency import get_exchange_rates
        rates = get_exchange_rates('THB')
        
        assert rates['USD'] == 0.028
        mock_get.assert_called_once()
    
    @patch('requests.get')
    def test_fetch_exchange_rate_error(self, mock_get):
        """Mock requests.get เมื่อเกิด error"""
        mock_get.side_effect = Exception('Connection error')
        
        from app.services.currency import get_exchange_rates
        rates = get_exchange_rates('THB')
        
        assert rates is None  # หรือ default value
```

---

## 7. ทดสอบ Models

```python
# tests/test_models.py

import pytest
from app.models import User, Post, db


class TestUserModel:
    """ทดสอบ User model"""
    
    def test_create_user(self, db_session):
        """ทดสอบสร้าง user"""
        user = User(username='newuser', email='new@example.com')
        user.set_password('password123')
        db_session.add(user)
        db_session.commit()
        
        assert user.id is not None
        assert user.username == 'newuser'
        assert user.is_active == True
    
    def test_password_hashing(self, db_session):
        """ทดสอบ password hashing"""
        user = User(username='hashtest', email='hash@example.com')
        user.set_password('mypassword')
        
        # password ควรถูก hash
        assert user.password_hash != 'mypassword'
        
        # ตรวจสอบ password ถูกต้อง
        assert user.check_password('mypassword') == True
        
        # ตรวจสอบ password ผิด
        assert user.check_password('wrongpassword') == False
    
    def test_user_repr(self, test_user):
        """ทดสอบ __repr__"""
        assert 'testuser' in repr(test_user)
    
    def test_user_to_dict(self, test_user):
        """ทดสอบ to_dict method"""
        data = test_user.to_dict()
        
        assert 'id' in data
        assert 'username' in data
        assert 'email' in data
        assert 'password' not in data      # ไม่ควรมี password
        assert 'password_hash' not in data  # ไม่ควรมี hash
    
    def test_user_unique_username(self, db_session, test_user):
        """ทดสอบว่า username ต้อง unique"""
        from sqlalchemy.exc import IntegrityError
        
        duplicate_user = User(
            username=test_user.username,  # username ซ้ำ
            email='different@example.com'
        )
        db_session.add(duplicate_user)
        
        with pytest.raises(IntegrityError):
            db_session.commit()


class TestPostModel:
    """ทดสอบ Post model"""
    
    def test_create_post(self, db_session, test_user):
        """ทดสอบสร้าง post"""
        post = Post(
            title='Test Title',
            content='Test content',
            user_id=test_user.id
        )
        db_session.add(post)
        db_session.commit()
        
        assert post.id is not None
        assert post.published == False  # default
    
    def test_post_author_relationship(self, test_post, test_user):
        """ทดสอบ relationship กับ User"""
        assert test_post.author.id == test_user.id
        assert test_post.author.username == test_user.username
    
    def test_user_posts_relationship(self, test_user, test_post):
        """ทดสอบ user.posts"""
        assert test_post in test_user.posts.all()
```

---

## 8. Coverage Report

```bash
# รัน tests พร้อม coverage
pytest tests/ --cov=app --cov-report=html

# ดู report ใน terminal
pytest tests/ --cov=app --cov-report=term-missing

# กำหนด minimum coverage
pytest tests/ --cov=app --cov-fail-under=80
```

```ini
# pytest.ini หรือ setup.cfg
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*

[coverage:run]
source = app
omit = 
    */tests/*
    */migrations/*
    */config.py
```

---

## 9. สรุป Part 083

✅ **unittest** ใช้ setUp/tearDown สำหรับ test isolation  
✅ **Flask Test Client** ทดสอบ HTTP requests โดยไม่ต้องรัน server จริง  
✅ **pytest fixtures** ใช้ conftest.py สร้าง reusable test data  
✅ **Mock** ใช้ patch() แทนที่ external dependencies  
✅ **Database testing** ใช้ transaction rollback ให้แต่ละ test เป็น isolated  
✅ **Coverage** ตรวจสอบว่าโค้ดถูก test ครบแค่ไหน  

---

## ➡️ ถัดไป: Part 084 - Flask Configuration

*Part 083/100+ | Python Course - Beginner to World-Class*
