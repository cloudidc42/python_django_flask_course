# Part 031: Unit Testing ด้วย pytest
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจหลักการ Unit Testing
- เขียน tests ด้วย pytest
- ใช้ fixtures, parametrize, mocking
- วัด test coverage
- TDD (Test-Driven Development) พื้นฐาน

---

## 1. ทำไมต้อง Test?

```
ประโยชน์ของ Testing:
✅ ค้นพบ bugs ก่อน production
✅ โค้ดกล้า refactor ได้
✅ เป็น documentation ที่อัพเดทอัตโนมัติ
✅ ทำงานเป็นทีมได้ดีขึ้น
✅ CI/CD pipeline
```

---

## 2. pytest เริ่มต้น

```bash
# ติดตั้ง
pip install pytest pytest-cov

# รัน tests
pytest                  # รันทุก test ใน directory ปัจจุบัน
pytest test_file.py     # รัน file เฉพาะ
pytest -v               # verbose mode
pytest -v -s            # แสดง print statements ด้วย
pytest -k "test_add"    # รันเฉพาะ test ที่ชื่อตรงกัน
pytest --tb=short       # traceback แบบสั้น
```

```python
# test_basic.py
def add(a, b):
    return a + b

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

# Test functions ต้องขึ้นต้นด้วย test_
def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0
    assert add(0, 0) == 0

def test_add_floats():
    result = add(0.1, 0.2)
    assert abs(result - 0.3) < 1e-9  # float comparison

def test_divide():
    assert divide(10, 2) == 5
    assert divide(7, 2) == 3.5

def test_divide_by_zero():
    import pytest
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(10, 0)
```

---

## 3. Fixtures

```python
# conftest.py - shared fixtures
import pytest

@pytest.fixture
def sample_user():
    return {
        "name": "Alice",
        "email": "alice@example.com",
        "age": 25,
        "is_active": True
    }

@pytest.fixture
def sample_users():
    return [
        {"name": "Alice", "score": 85},
        {"name": "Bob", "score": 72},
        {"name": "Charlie", "score": 91},
    ]

@pytest.fixture
def temp_file(tmp_path):
    """สร้างไฟล์ชั่วคราว"""
    file = tmp_path / "test.txt"
    file.write_text("Hello, World!")
    return file

# test_fixtures.py
def test_user_properties(sample_user):
    assert sample_user["name"] == "Alice"
    assert sample_user["age"] >= 18
    assert "@" in sample_user["email"]

def test_user_modification(sample_user):
    sample_user["age"] = 26
    assert sample_user["age"] == 26

def test_file_content(temp_file):
    content = temp_file.read_text()
    assert content == "Hello, World!"

def test_file_exists(temp_file):
    assert temp_file.exists()

# Fixture scope
@pytest.fixture(scope="session")
def database():
    """สร้าง database ครั้งเดียวสำหรับทั้ง session"""
    db = {"connected": True, "data": []}
    yield db
    db["connected"] = False  # cleanup

@pytest.fixture(scope="function")  # default
def fresh_list():
    return []

@pytest.fixture(scope="class")
def class_data():
    return {"value": 0}
```

---

## 4. Parametrize - ทดสอบหลายกรณีพร้อมกัน

```python
import pytest

def is_prime(n):
    if n < 2:
        return False
    for i in range(2, int(n**0.5) + 1):
        if n % i == 0:
            return False
    return True

@pytest.mark.parametrize("n, expected", [
    (2, True),
    (3, True),
    (4, False),
    (5, True),
    (6, False),
    (7, True),
    (8, False),
    (9, False),
    (10, False),
    (11, True),
    (1, False),
    (0, False),
    (-1, False),
])
def test_is_prime(n, expected):
    assert is_prime(n) == expected

# Parametrize กับหลาย arguments
@pytest.mark.parametrize("a, b, expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
    (100, 200, 300),
])
def test_add_multiple(a, b, expected):
    assert add(a, b) == expected
```

---

## 5. Mocking

```python
# มี 3 วิธีหลัก:
# 1. unittest.mock.patch
# 2. pytest-mock (mocker fixture)
# 3. Manual mocking

from unittest.mock import MagicMock, patch

# ฟังก์ชันที่ต้อง mock
import requests

def get_user_data(user_id):
    response = requests.get(f"https://api.example.com/users/{user_id}")
    return response.json()

# Test ด้วย mock
def test_get_user_data():
    mock_response = MagicMock()
    mock_response.json.return_value = {"id": 1, "name": "Alice"}
    
    with patch("requests.get", return_value=mock_response):
        result = get_user_data(1)
        assert result == {"id": 1, "name": "Alice"}

# Mock ด้วย decorator
@patch("requests.get")
def test_get_user_data_decorator(mock_get):
    mock_get.return_value.json.return_value = {"id": 1, "name": "Bob"}
    result = get_user_data(1)
    assert result["name"] == "Bob"
    mock_get.assert_called_once_with("https://api.example.com/users/1")
```

---

## 6. Test Coverage

```bash
# วัด coverage
pytest --cov=mymodule --cov-report=html

# แสดงใน terminal
pytest --cov=. --cov-report=term-missing

# HTML report ดูได้ที่ htmlcov/index.html
```

```python
# setup.cfg หรือ pytest.ini
[tool:pytest]
testpaths = tests
addopts = --cov=src --cov-report=html --cov-fail-under=80
```

---

## 7. TDD Example

```python
# วิธี TDD: เขียน test ก่อน แล้วค่อยเขียนโค้ด

# Step 1: เขียน failing test
def test_shopping_cart_add():
    cart = ShoppingCart()
    cart.add_item("apple", 2, 10.0)
    assert cart.total() == 20.0

# Step 2: เขียนโค้ดให้ test ผ่าน
class ShoppingCart:
    def __init__(self):
        self.items = []
    
    def add_item(self, name, quantity, price):
        self.items.append({"name": name, "qty": quantity, "price": price})
    
    def total(self):
        return sum(item["qty"] * item["price"] for item in self.items)
    
    def remove_item(self, name):
        self.items = [i for i in self.items if i["name"] != name]
    
    def apply_discount(self, percent):
        return self.total() * (1 - percent/100)

# Step 3: เพิ่ม tests
def test_shopping_cart_multiple_items():
    cart = ShoppingCart()
    cart.add_item("apple", 2, 10.0)
    cart.add_item("banana", 3, 5.0)
    assert cart.total() == 35.0

def test_shopping_cart_remove():
    cart = ShoppingCart()
    cart.add_item("apple", 2, 10.0)
    cart.remove_item("apple")
    assert cart.total() == 0.0

def test_shopping_cart_discount():
    cart = ShoppingCart()
    cart.add_item("item", 1, 100.0)
    assert cart.apply_discount(10) == 90.0
```

---

## 8. สรุป Part 031

✅ **pytest** - framework สำหรับ testing  
✅ **assert** - ตรวจสอบผลลัพธ์  
✅ **fixtures** - shared setup/teardown  
✅ **parametrize** - ทดสอบหลาย input พร้อมกัน  
✅ **mocking** - mock external dependencies  
✅ **coverage** - วัดความครอบคลุม  
✅ **TDD** - เขียน test ก่อนโค้ด  

---

## ➡️ ถัดไป: Part 032 - Concurrency - Threading

*Part 031/100+ | Python Course - Beginner to World-Class*
