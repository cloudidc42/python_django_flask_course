# Part 087: FastAPI Path Parameters and Query
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- ใช้ Path parameters รับค่าจาก URL
- ใช้ Query parameters กำหนดตัวเลือก
- กำหนด Optional parameters
- Validate ด้วย Path() และ Query()
- สร้าง enum parameters

---

## 1. Path Parameters

Path parameters คือส่วนของ URL ที่เปลี่ยนแปลงได้

```python
# path_params.py

from fastapi import FastAPI, Path, HTTPException

app = FastAPI()


# Path parameter พื้นฐาน
@app.get("/items/{item_id}")
def get_item(item_id: int):
    """
    item_id จาก URL จะถูก convert เป็น int อัตโนมัติ
    GET /items/1  →  item_id = 1 (int)
    GET /items/abc  →  422 Unprocessable Entity (ไม่ใช่ int)
    """
    return {"item_id": item_id}


# หลาย path parameters
@app.get("/users/{user_id}/posts/{post_id}")
def get_user_post(user_id: int, post_id: int):
    """ดึงบทความของ user"""
    return {
        "user_id": user_id,
        "post_id": post_id
    }


# String path parameter
@app.get("/categories/{category_name}")
def get_category(category_name: str):
    """ดึงหมวดหมู่ตามชื่อ"""
    return {"category": category_name}


# Path parameter ด้วย validation
@app.get("/products/{product_id}")
def get_product(
    product_id: int = Path(
        ...,                          # ... = required
        title="Product ID",           # ชื่อใน Swagger
        description="ID ของสินค้า",   # คำอธิบายใน Swagger
        ge=1,                         # >= 1
        le=9999                       # <= 9999
    )
):
    """ดึงสินค้าตาม ID (1-9999)"""
    return {"product_id": product_id}
```

### Path Parameter Types
```python
from fastapi import FastAPI
from uuid import UUID

app = FastAPI()


@app.get("/items/{item_id}")
def get_item_int(item_id: int):
    """Path param เป็น int"""
    return {"id": item_id, "type": "int"}


@app.get("/files/{file_path:path}")
def get_file(file_path: str):
    """
    Path param ที่มี / ได้ (ใช้ :path)
    GET /files/images/photo.jpg  →  file_path = "images/photo.jpg"
    """
    return {"file_path": file_path}


@app.get("/items/{item_uuid}")
def get_item_uuid(item_uuid: UUID):
    """Path param เป็น UUID"""
    return {"uuid": str(item_uuid)}
```

---

## 2. Query Parameters

Query parameters คือ parameters หลัง `?` ใน URL

```python
# query_params.py

from fastapi import FastAPI, Query
from typing import Optional, List

app = FastAPI()


# Query parameters พื้นฐาน
@app.get("/items")
def get_items(skip: int = 0, limit: int = 10):
    """
    GET /items           →  skip=0, limit=10
    GET /items?skip=20   →  skip=20, limit=10
    GET /items?limit=5   →  skip=0, limit=5
    GET /items?skip=5&limit=3  →  skip=5, limit=3
    """
    all_items = [{"id": i, "name": f"Item {i}"} for i in range(1, 101)]
    return all_items[skip:skip + limit]


# Query parameters ที่ required
@app.get("/search")
def search(q: str):
    """
    ต้องระบุ q
    GET /search?q=python  →  q="python"
    GET /search  →  422 error (missing q)
    """
    return {"search": q, "results": []}


# Optional query parameter
@app.get("/posts")
def get_posts(
    published: Optional[bool] = None,
    author: Optional[str] = None,
    page: int = 1
):
    """
    GET /posts                          →  ทั้งหมด
    GET /posts?published=true           →  เฉพาะที่ publish แล้ว
    GET /posts?author=john              →  เฉพาะของ john
    GET /posts?published=true&author=john  →  ของ john ที่ publish แล้ว
    """
    filters = {}
    if published is not None:
        filters['published'] = published
    if author:
        filters['author'] = author
    
    return {"page": page, "filters": filters, "posts": []}


# Query parameter ด้วย validation
@app.get("/products")
def get_products(
    q: Optional[str] = Query(
        None,
        min_length=3,                # ถ้าใส่ต้องมีอย่างน้อย 3 ตัว
        max_length=50,               # ไม่เกิน 50 ตัว
        title="Search Query",
        description="คำค้นหาสินค้า",
        alias="search"              # ชื่อใน URL จะเป็น ?search= แทน ?q=
    ),
    page: int = Query(1, ge=1),
    per_page: int = Query(10, ge=1, le=100)
):
    """
    GET /products?search=laptop&page=2&per_page=20
    """
    return {
        "query": q,
        "page": page,
        "per_page": per_page
    }
```

---

## 3. Optional Parameters

```python
# optional_params.py

from fastapi import FastAPI, Query, Path
from typing import Optional, Union

app = FastAPI()


@app.get("/users/{user_id}/profile")
def get_user_profile(
    user_id: int,
    include_posts: bool = False,
    include_followers: bool = False,
    fields: Optional[str] = None  # comma-separated fields
):
    """
    GET /users/1/profile  →  basic profile
    GET /users/1/profile?include_posts=true  →  + posts
    GET /users/1/profile?fields=id,username,email  →  specific fields
    """
    profile = {
        "id": user_id,
        "username": "john_doe",
        "email": "john@example.com",
        "bio": "Python developer"
    }
    
    if include_posts:
        profile["posts"] = []
    
    if include_followers:
        profile["followers"] = []
    
    if fields:
        field_list = [f.strip() for f in fields.split(',')]
        profile = {k: v for k, v in profile.items() if k in field_list}
    
    return profile


# Union type สำหรับ parameter ที่รับหลาย types
@app.get("/lookup/{identifier}")
def lookup(identifier: Union[int, str]):
    """รับทั้ง int และ str"""
    if isinstance(identifier, int):
        return {"type": "id", "value": identifier}
    return {"type": "username", "value": identifier}
```

---

## 4. List Query Parameters

```python
# list_params.py

from fastapi import FastAPI, Query
from typing import List, Optional

app = FastAPI()


# รับ list จาก query parameters
@app.get("/items/filter")
def filter_items(
    # GET /items/filter?tag=python&tag=flask&tag=fastapi
    tags: Optional[List[str]] = Query(None),
    # GET /items/filter?ids=1&ids=2&ids=3
    ids: Optional[List[int]] = Query(None)
):
    """กรองสินค้าด้วย tags หรือ ids"""
    filters = {}
    if tags:
        filters["tags"] = tags
    if ids:
        filters["ids"] = ids
    return {"filters": filters}


# Default list
@app.get("/default-list")
def get_with_default_list(
    items: List[str] = Query(["a", "b", "c"])
):
    """Query param ที่มี default เป็น list"""
    return {"items": items}
```

---

## 5. Enum Parameters

```python
# enum_params.py

from fastapi import FastAPI
from enum import Enum


# สร้าง Enum
class ItemCategory(str, Enum):
    """หมวดหมู่สินค้า"""
    electronics = "electronics"
    clothing = "clothing"
    food = "food"
    books = "books"


class SortOrder(str, Enum):
    """ลำดับการเรียง"""
    asc = "asc"
    desc = "desc"


class UserRole(str, Enum):
    """บทบาทผู้ใช้"""
    admin = "admin"
    moderator = "moderator"
    user = "user"


app = FastAPI()


@app.get("/items/category/{category}")
def get_by_category(category: ItemCategory):
    """
    GET /items/category/electronics  →  OK
    GET /items/category/invalid  →  422 (ไม่อยู่ใน enum)
    """
    return {
        "category": category,
        "category_value": category.value,
        "items": []
    }


@app.get("/items/sorted")
def get_sorted_items(
    sort_by: str = "name",
    order: SortOrder = SortOrder.asc
):
    """
    GET /items/sorted?order=desc
    GET /items/sorted?sort_by=price&order=asc
    """
    return {
        "sort_by": sort_by,
        "order": order,
        "items": []
    }


@app.get("/admin/users")
def get_users_by_role(role: Optional[UserRole] = None):
    """
    GET /admin/users         →  ทุก role
    GET /admin/users?role=admin  →  เฉพาะ admin
    """
    return {"role": role, "users": []}
```

---

## 6. Parameter Validation

```python
# validation.py

from fastapi import FastAPI, Path, Query
from typing import Optional

app = FastAPI()


@app.get("/validate-path/{item_id}")
def validate_path(
    item_id: int = Path(
        ...,
        title="Item ID",
        description="ID ของ item (1-1000)",
        ge=1,    # greater than or equal to 1
        le=1000  # less than or equal to 1000
    )
):
    return {"item_id": item_id}


@app.get("/validate-query")
def validate_query(
    price_min: float = Query(
        0.0,
        ge=0.0,           # >= 0
        description="ราคาต่ำสุด"
    ),
    price_max: float = Query(
        9999999.0,
        le=9999999.0,     # <= 9,999,999
        description="ราคาสูงสุด"
    ),
    name: Optional[str] = Query(
        None,
        min_length=2,     # ถ้าใส่ต้องมี >= 2 ตัว
        max_length=100,   # ไม่เกิน 100 ตัว
        regex=r"^[a-zA-Z\s]+$"  # ตัวอักษรและ space เท่านั้น
    ),
    page: int = Query(
        1,
        ge=1,
        description="หน้า (เริ่มจาก 1)"
    ),
    per_page: int = Query(
        default=10,
        ge=1,
        le=100,
        description="จำนวนต่อหน้า (1-100)"
    )
):
    """ตัวอย่าง query validation ครบถ้วน"""
    if price_min > price_max:
        from fastapi import HTTPException
        raise HTTPException(400, "price_min ต้องน้อยกว่า price_max")
    
    return {
        "price_range": f"{price_min} - {price_max}",
        "name_filter": name,
        "pagination": {"page": page, "per_page": per_page}
    }
```

---

## 7. ตัวอย่างสมบูรณ์: Product Search API

```python
# product_search.py

from fastapi import FastAPI, Path, Query, HTTPException
from pydantic import BaseModel
from typing import Optional, List
from enum import Enum
from datetime import datetime

app = FastAPI(title="Product Search API")


class ProductCategory(str, Enum):
    all = "all"
    electronics = "electronics"
    clothing = "clothing"
    food = "food"
    books = "books"


class SortField(str, Enum):
    name = "name"
    price = "price"
    created_at = "created_at"


class SortOrder(str, Enum):
    asc = "asc"
    desc = "desc"


class ProductResponse(BaseModel):
    id: int
    name: str
    category: str
    price: float
    in_stock: bool


class SearchResponse(BaseModel):
    query: Optional[str]
    category: str
    price_range: dict
    sort: dict
    pagination: dict
    total: int
    products: List[ProductResponse]


# ข้อมูลตัวอย่าง
PRODUCTS = [
    {"id": 1, "name": "MacBook Pro", "category": "electronics", "price": 59900.0, "in_stock": True},
    {"id": 2, "name": "iPhone 15", "category": "electronics", "price": 35900.0, "in_stock": True},
    {"id": 3, "name": "เสื้อยืด", "category": "clothing", "price": 299.0, "in_stock": True},
    {"id": 4, "name": "กางเกงยีนส์", "category": "clothing", "price": 790.0, "in_stock": False},
    {"id": 5, "name": "Python Book", "category": "books", "price": 590.0, "in_stock": True},
]


@app.get("/products/search", response_model=SearchResponse)
def search_products(
    # Search
    q: Optional[str] = Query(None, min_length=1, description="คำค้นหา"),
    
    # Filter
    category: ProductCategory = Query(ProductCategory.all, description="หมวดหมู่"),
    price_min: float = Query(0.0, ge=0.0, description="ราคาต่ำสุด"),
    price_max: float = Query(999999.0, le=999999.0, description="ราคาสูงสุด"),
    in_stock: Optional[bool] = Query(None, description="มีสินค้าในสต็อก"),
    
    # Sort
    sort_by: SortField = Query(SortField.name, description="เรียงตาม"),
    order: SortOrder = Query(SortOrder.asc, description="ลำดับ"),
    
    # Pagination
    page: int = Query(1, ge=1),
    per_page: int = Query(10, ge=1, le=50)
):
    """ค้นหาสินค้าพร้อม filter, sort, pagination"""
    
    # Validate price range
    if price_min > price_max:
        raise HTTPException(400, "price_min ต้องน้อยกว่าหรือเท่ากับ price_max")
    
    # Filter
    results = PRODUCTS.copy()
    
    if q:
        results = [p for p in results if q.lower() in p["name"].lower()]
    
    if category != ProductCategory.all:
        results = [p for p in results if p["category"] == category.value]
    
    results = [p for p in results if price_min <= p["price"] <= price_max]
    
    if in_stock is not None:
        results = [p for p in results if p["in_stock"] == in_stock]
    
    # Sort
    reverse = (order == SortOrder.desc)
    results.sort(key=lambda x: x[sort_by.value], reverse=reverse)
    
    # Paginate
    total = len(results)
    start = (page - 1) * per_page
    items = results[start:start + per_page]
    
    return {
        "query": q,
        "category": category.value,
        "price_range": {"min": price_min, "max": price_max},
        "sort": {"by": sort_by.value, "order": order.value},
        "pagination": {
            "page": page,
            "per_page": per_page,
            "total": total,
            "pages": (total + per_page - 1) // per_page
        },
        "total": total,
        "products": items
    }


@app.get("/products/{product_id}", response_model=ProductResponse)
def get_product(
    product_id: int = Path(..., ge=1, description="Product ID")
):
    """ดึงสินค้าตาม ID"""
    product = next((p for p in PRODUCTS if p["id"] == product_id), None)
    if not product:
        raise HTTPException(404, f"ไม่พบสินค้า ID {product_id}")
    return product
```

---

## 8. สรุป Part 087

✅ **Path parameters** รับค่าจาก URL เช่น `/items/{item_id}`  
✅ **Type hints** บอก FastAPI ว่าจะ convert และ validate อย่างไร  
✅ **Query parameters** รับค่าหลัง `?` เช่น `/items?page=1&limit=10`  
✅ **Path() / Query()** ใช้กำหนด validation rules เพิ่มเติม  
✅ **Optional parameters** ใช้ `Optional[type]` หรือ `= None`  
✅ **Enum parameters** จำกัดค่าที่รับได้  
✅ **List parameters** รับหลายค่าด้วย `List[type]`  

---

## ➡️ ถัดไป: Part 088 - FastAPI Request Body

*Part 087/100+ | Python Course - Beginner to World-Class*
