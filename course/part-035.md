# Part 035: Database กับ SQLite
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เชื่อมต่อ SQLite database ด้วย sqlite3 module
- สร้าง connection และ cursor
- ทำ CRUD operations: CREATE, READ, UPDATE, DELETE
- Parameterized queries (ป้องกัน SQL Injection)
- fetchall(), fetchone(), fetchmany()
- ใช้ Context Manager กับ database
- Transactions และ rollback
- ตัวอย่างจริง: Blog database

---

## 1. SQLite พื้นฐาน

```python
import sqlite3
import os

# SQLite เป็น file-based database ที่ built-in กับ Python
# ไม่ต้องติดตั้ง server แยก เหมาะกับ development, testing, small apps

# เชื่อมต่อ (สร้างไฟล์ถ้ายังไม่มี)
conn = sqlite3.connect("myapp.db")
print(f"Connected to SQLite version: {sqlite3.version}")

# :memory: = in-memory database (ไม่บันทึกไฟล์)
conn_mem = sqlite3.connect(":memory:")

# cursor สำหรับ execute SQL
cursor = conn.cursor()

# สร้าง table
cursor.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT NOT NULL UNIQUE,
        email TEXT NOT NULL UNIQUE,
        age INTEGER,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
""")

# บันทึกการเปลี่ยนแปลง
conn.commit()
print("Table created!")

# ปิด connection (สำคัญมาก!)
# conn.close()  # เราจะปิดทีหลัง

# ✅ ใช้ Context Manager (แนะนำ)
with sqlite3.connect(":memory:") as conn:
    cursor = conn.cursor()
    cursor.execute("CREATE TABLE test (id INTEGER, name TEXT)")
    cursor.execute("INSERT INTO test VALUES (1, 'Alice')")
    # commit อัตโนมัติเมื่อออกจาก with block (ถ้าไม่มี error)
    # rollback อัตโนมัติถ้ามี exception

# connection info
print(f"Isolation level: {conn.isolation_level}")
print(f"Row factory: {conn.row_factory}")
```

---

## 2. CRUD: Create (INSERT)

```python
import sqlite3
from datetime import datetime

def setup_database(conn: sqlite3.Connection) -> None:
    """สร้าง tables"""
    cursor = conn.cursor()
    
    cursor.executescript("""
        CREATE TABLE IF NOT EXISTS categories (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL UNIQUE,
            description TEXT
        );
        
        CREATE TABLE IF NOT EXISTS products (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            description TEXT,
            price REAL NOT NULL,
            stock INTEGER DEFAULT 0,
            category_id INTEGER,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (category_id) REFERENCES categories(id)
        );
    """)
    conn.commit()

conn = sqlite3.connect(":memory:")
setup_database(conn)
cursor = conn.cursor()

# ❌ อันตราย: String formatting (SQL Injection!)
# name = "'; DROP TABLE products; --"
# cursor.execute(f"INSERT INTO categories (name) VALUES ('{name}')")

# ✅ Parameterized query (แนะนำ)
# ใช้ ? เป็น placeholder

# INSERT ครั้งเดียว
cursor.execute(
    "INSERT INTO categories (name, description) VALUES (?, ?)",
    ("Electronics", "Electronic devices and gadgets")
)
print(f"Inserted category ID: {cursor.lastrowid}")

cursor.execute(
    "INSERT INTO categories (name, description) VALUES (?, ?)",
    ("Books", "Books and educational materials")
)

# INSERT หลายรายการพร้อมกัน
products = [
    ("iPhone 15", "Apple smartphone", 39900.0, 50, 1),
    ("MacBook Pro", "Apple laptop", 79900.0, 20, 1),
    ("Python Book", "Learn Python programming", 590.0, 100, 2),
    ("AirPods Pro", "Wireless earbuds", 9900.0, 75, 1),
    ("Django Book", "Web development with Django", 650.0, 80, 2),
]

cursor.executemany(
    "INSERT INTO products (name, description, price, stock, category_id) VALUES (?, ?, ?, ?, ?)",
    products
)
print(f"Inserted {cursor.rowcount} products")

conn.commit()

# Named placeholders
cursor.execute(
    """INSERT INTO products (name, price, stock, category_id)
       VALUES (:name, :price, :stock, :category_id)""",
    {"name": "Flask Course", "price": 450.0, "stock": 200, "category_id": 2}
)
conn.commit()
print(f"Inserted Flask Course with ID: {cursor.lastrowid}")
```

---

## 3. CRUD: Read (SELECT)

```python
# fetchone(), fetchall(), fetchmany()

# fetchone() - ดึง 1 แถว
cursor.execute("SELECT * FROM categories LIMIT 1")
row = cursor.fetchone()
print(f"\nFirst category: {row}")
# (1, 'Electronics', 'Electronic devices and gadgets')

# fetchall() - ดึงทั้งหมด
cursor.execute("SELECT id, name, price FROM products ORDER BY price DESC")
rows = cursor.fetchall()
print("\nAll products (by price):")
for row in rows:
    print(f"  [{row[0]}] {row[1]}: {row[2]:,.2f} THB")

# fetchmany() - ดึง n แถว
cursor.execute("SELECT name, price FROM products ORDER BY price")
cheapest = cursor.fetchmany(3)
print("\n3 cheapest products:")
for row in cheapest:
    print(f"  {row[0]}: {row[1]:.2f}")

# WHERE clause กับ parameterized query
def find_product_by_id(conn: sqlite3.Connection, product_id: int):
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM products WHERE id = ?", (product_id,))
    return cursor.fetchone()

product = find_product_by_id(conn, 1)
print(f"\nProduct #1: {product}")

# JOIN query
cursor.execute("""
    SELECT p.id, p.name, p.price, c.name AS category
    FROM products p
    JOIN categories c ON p.category_id = c.id
    WHERE p.price < ?
    ORDER BY p.price
""", (1000.0,))

print("\nAffordable products (<1000 THB):")
for row in cursor.fetchall():
    print(f"  [{row[0]}] {row[1]} ({row[3]}): {row[2]:.2f} THB")

# Aggregate functions
cursor.execute("""
    SELECT 
        c.name,
        COUNT(p.id) AS product_count,
        AVG(p.price) AS avg_price,
        SUM(p.price * p.stock) AS total_value
    FROM categories c
    LEFT JOIN products p ON c.id = p.category_id
    GROUP BY c.id, c.name
    ORDER BY total_value DESC
""")

print("\nCategory statistics:")
for row in cursor.fetchall():
    cat, count, avg, total = row
    print(f"  {cat}: {count} products, avg={avg:.0f}, total value={total:,.0f} THB")

# Row factory: เข้าถึง columns ด้วยชื่อ
conn.row_factory = sqlite3.Row  # เปลี่ยน default tuple เป็น Row object

cursor = conn.cursor()
cursor.execute("SELECT id, name, price FROM products WHERE id = 1")
row = cursor.fetchone()

# เข้าถึงด้วยชื่อ column
print(f"\nRow factory demo: {row['name']} costs {row['price']:.2f}")
print(f"Columns: {row.keys()}")

# dict จาก Row
product_dict = dict(row)
print(f"As dict: {product_dict}")
```

---

## 4. CRUD: Update และ Delete

```python
# Reset row_factory
conn.row_factory = sqlite3.Row
cursor = conn.cursor()

# UPDATE
cursor.execute(
    "UPDATE products SET price = ?, stock = ? WHERE id = ?",
    (37900.0, 45, 1)
)
print(f"Updated {cursor.rowcount} row(s)")
conn.commit()

# UPDATE หลายแถว
cursor.execute(
    "UPDATE products SET stock = stock + ? WHERE category_id = ?",
    (10, 2)  # เพิ่ม stock 10 ให้ทุก products ใน category 2
)
print(f"Updated {cursor.rowcount} rows (added stock)")
conn.commit()

# Verify update
cursor.execute("SELECT name, price, stock FROM products WHERE id = 1")
row = cursor.fetchone()
print(f"Updated iPhone: price={row['price']}, stock={row['stock']}")

# DELETE
cursor.execute("DELETE FROM products WHERE id = ?", (3,))
print(f"Deleted {cursor.rowcount} row(s)")
conn.commit()

# Soft delete pattern
cursor.execute("""
    ALTER TABLE products ADD COLUMN IF NOT EXISTS is_deleted INTEGER DEFAULT 0
""")

def soft_delete(conn: sqlite3.Connection, product_id: int) -> bool:
    """Soft delete - ไม่ลบจริง แค่ mark"""
    cursor = conn.cursor()
    cursor.execute(
        "UPDATE products SET is_deleted = 1 WHERE id = ?",
        (product_id,)
    )
    conn.commit()
    return cursor.rowcount > 0

# Query products ที่ไม่ถูก delete
cursor.execute("SELECT id, name FROM products WHERE is_deleted = 0 OR is_deleted IS NULL")
active_products = cursor.fetchall()
print(f"\nActive products: {len(active_products)}")

# Bulk delete
ids_to_delete = [4, 5]
placeholders = ", ".join("?" for _ in ids_to_delete)
cursor.execute(
    f"DELETE FROM products WHERE id IN ({placeholders})",
    ids_to_delete
)
print(f"Bulk deleted {cursor.rowcount} rows")
conn.commit()

# DELETE all (with confirmation via RETURNING, SQLite 3.35+)
cursor.execute("SELECT COUNT(*) FROM products")
count_before = cursor.fetchone()[0]
print(f"Products before: {count_before}")
```

---

## 5. Transactions

```python
import sqlite3

conn = sqlite3.connect(":memory:")
cursor = conn.cursor()

# Setup
cursor.executescript("""
    CREATE TABLE accounts (
        id INTEGER PRIMARY KEY,
        owner TEXT NOT NULL,
        balance REAL NOT NULL DEFAULT 0
    );
    INSERT INTO accounts VALUES (1, 'Alice', 10000.0);
    INSERT INTO accounts VALUES (2, 'Bob', 5000.0);
""")
conn.commit()

def transfer_money(
    conn: sqlite3.Connection,
    from_id: int,
    to_id: int,
    amount: float
) -> bool:
    """โอนเงิน - ต้องเป็น atomic operation"""
    cursor = conn.cursor()
    
    try:
        # เริ่ม transaction
        conn.execute("BEGIN")
        
        # ตรวจสอบยอดเงิน
        cursor.execute("SELECT balance FROM accounts WHERE id = ?", (from_id,))
        row = cursor.fetchone()
        if not row:
            raise ValueError(f"Account {from_id} not found")
        
        current_balance = row[0]
        if current_balance < amount:
            raise ValueError(f"Insufficient funds: {current_balance} < {amount}")
        
        # หักเงินจากบัญชี from
        cursor.execute(
            "UPDATE accounts SET balance = balance - ? WHERE id = ?",
            (amount, from_id)
        )
        
        # ✅ Simulate error (uncomment เพื่อทดสอบ rollback)
        # raise RuntimeError("Simulated network error!")
        
        # เพิ่มเงินในบัญชี to
        cursor.execute(
            "UPDATE accounts SET balance = balance + ? WHERE id = ?",
            (amount, to_id)
        )
        
        # Commit ถ้าทุกอย่างสำเร็จ
        conn.commit()
        print(f"  ✅ Transferred {amount:.2f} from Account {from_id} to {to_id}")
        return True
    
    except Exception as e:
        # Rollback ถ้าเกิด error ใดๆ
        conn.rollback()
        print(f"  ❌ Transfer failed: {e} - rolled back!")
        return False

def show_balances(conn: sqlite3.Connection) -> None:
    cursor = conn.cursor()
    cursor.execute("SELECT id, owner, balance FROM accounts")
    for row in cursor.fetchall():
        print(f"  Account {row[0]} ({row[1]}): {row[2]:,.2f} THB")

print("Initial balances:")
show_balances(conn)

print("\nTransfer 3000 from Alice to Bob:")
transfer_money(conn, 1, 2, 3000.0)
show_balances(conn)

print("\nTransfer 20000 from Bob (insufficient):")
transfer_money(conn, 2, 1, 20000.0)
show_balances(conn)

# Context manager สำหรับ transactions
print("\nUsing connection as context manager:")
try:
    with conn:  # auto commit/rollback
        conn.execute("UPDATE accounts SET balance = balance + 1000 WHERE id = 1")
        conn.execute("UPDATE accounts SET balance = balance - 1000 WHERE id = 2")
        print("  Bonus applied!")
except Exception as e:
    print(f"  Error: {e}")

show_balances(conn)
```

---

## 6. Row Factory และ Custom Types

```python
import sqlite3
import json
from datetime import datetime
from typing import Optional

# Custom Row Factory: คืนค่าเป็น dict
def dict_factory(cursor, row):
    """แปลง rows เป็น dictionaries"""
    fields = [description[0] for description in cursor.description]
    return {k: v for k, v in zip(fields, row)}

conn = sqlite3.connect(":memory:")
conn.row_factory = dict_factory

cursor = conn.cursor()
cursor.executescript("""
    CREATE TABLE employees (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        department TEXT,
        salary REAL,
        skills TEXT,
        hire_date TEXT
    );
    
    INSERT INTO employees VALUES (1, 'Alice', 'Engineering', 85000, '["python","django","aws"]', '2022-01-15');
    INSERT INTO employees VALUES (2, 'Bob', 'Marketing', 65000, '["excel","powerpoint","seo"]', '2021-03-20');
    INSERT INTO employees VALUES (3, 'Charlie', 'Engineering', 95000, '["python","fastapi","k8s"]', '2020-06-01');
""")

# ตอนนี้ rows เป็น dict
cursor.execute("SELECT * FROM employees")
employees = cursor.fetchall()
for emp in employees:
    print(f"  {emp['name']} ({emp['department']}): {emp['salary']:,.0f} THB")

# Adapter: แปลง Python types เป็น SQLite
# Converter: แปลง SQLite values กลับเป็น Python types

# Register JSON adapter/converter
def adapt_json(data) -> str:
    return json.dumps(data)

def convert_json(data: bytes):
    return json.loads(data)

sqlite3.register_adapter(list, adapt_json)
sqlite3.register_adapter(dict, adapt_json)
sqlite3.register_converter("JSON", convert_json)

conn2 = sqlite3.connect(":memory:", detect_types=sqlite3.PARSE_DECLTYPES)
cursor2 = conn2.cursor()
cursor2.execute("CREATE TABLE items (id INTEGER, data JSON)")

# ใส่ list/dict โดยตรง
cursor2.execute("INSERT INTO items VALUES (?, ?)", (1, {"name": "test", "value": 42}))
cursor2.execute("INSERT INTO items VALUES (?, ?)", (2, [1, 2, 3, 4, 5]))
conn2.commit()

cursor2.execute("SELECT id, data FROM items")
for row in cursor2.fetchall():
    print(f"  [{row[0]}] {row[1]} (type: {type(row[1]).__name__})")
# [1] {'name': 'test', 'value': 42} (type: dict)
# [2] [1, 2, 3, 4, 5] (type: list)
```

---

## 7. Database Wrapper Class

```python
import sqlite3
from typing import Optional, List, Dict, Any, Tuple
from contextlib import contextmanager

class Database:
    """Database wrapper สำหรับ SQLite"""
    
    def __init__(self, db_path: str) -> None:
        self.db_path = db_path
        self._conn: Optional[sqlite3.Connection] = None
    
    def connect(self) -> None:
        self._conn = sqlite3.connect(self.db_path)
        self._conn.row_factory = sqlite3.Row
        # เปิด foreign key support
        self._conn.execute("PRAGMA foreign_keys = ON")
    
    def disconnect(self) -> None:
        if self._conn:
            self._conn.close()
            self._conn = None
    
    def __enter__(self):
        self.connect()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.disconnect()
    
    @contextmanager
    def transaction(self):
        """Context manager สำหรับ transactions"""
        try:
            yield self._conn
            self._conn.commit()
        except Exception:
            self._conn.rollback()
            raise
    
    def execute(self, sql: str, params: tuple = ()) -> sqlite3.Cursor:
        cursor = self._conn.cursor()
        cursor.execute(sql, params)
        return cursor
    
    def executemany(self, sql: str, params_list: list) -> sqlite3.Cursor:
        cursor = self._conn.cursor()
        cursor.executemany(sql, params_list)
        return cursor
    
    def fetchone(self, sql: str, params: tuple = ()) -> Optional[sqlite3.Row]:
        return self.execute(sql, params).fetchone()
    
    def fetchall(self, sql: str, params: tuple = ()) -> List[sqlite3.Row]:
        return self.execute(sql, params).fetchall()
    
    def insert(self, table: str, data: Dict[str, Any]) -> int:
        columns = ", ".join(data.keys())
        placeholders = ", ".join("?" * len(data))
        sql = f"INSERT INTO {table} ({columns}) VALUES ({placeholders})"
        cursor = self.execute(sql, tuple(data.values()))
        self._conn.commit()
        return cursor.lastrowid
    
    def update(self, table: str, data: Dict[str, Any], where: str, params: tuple = ()) -> int:
        set_clause = ", ".join(f"{k} = ?" for k in data.keys())
        sql = f"UPDATE {table} SET {set_clause} WHERE {where}"
        cursor = self.execute(sql, tuple(data.values()) + params)
        self._conn.commit()
        return cursor.rowcount
    
    def delete(self, table: str, where: str, params: tuple = ()) -> int:
        sql = f"DELETE FROM {table} WHERE {where}"
        cursor = self.execute(sql, params)
        self._conn.commit()
        return cursor.rowcount

# ใช้งาน
with Database(":memory:") as db:
    # Setup
    db.execute("""
        CREATE TABLE tasks (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            done INTEGER DEFAULT 0,
            priority INTEGER DEFAULT 3
        )
    """)
    db._conn.commit()
    
    # Insert
    task_id = db.insert("tasks", {
        "title": "Learn Python",
        "priority": 1
    })
    print(f"Inserted task ID: {task_id}")
    
    db.insert("tasks", {"title": "Build API", "priority": 2})
    db.insert("tasks", {"title": "Write tests", "priority": 1})
    
    # Select
    tasks = db.fetchall("SELECT * FROM tasks ORDER BY priority")
    print("\nAll tasks:")
    for t in tasks:
        print(f"  [{t['id']}] P{t['priority']}: {t['title']}")
    
    # Update
    count = db.update("tasks", {"done": 1}, "id = ?", (task_id,))
    print(f"\nUpdated {count} task(s)")
    
    # Delete
    count = db.delete("tasks", "done = ?", (0,))
    # (ไม่ลบอะไรเพราะ priority 2,3 ยังไม่ done)
    
    # Transaction
    with db.transaction():
        db.execute("UPDATE tasks SET priority = 2 WHERE id = 2")
        db.execute("UPDATE tasks SET priority = 3 WHERE id = 3")
    
    print("\nAfter transaction:")
    for t in db.fetchall("SELECT * FROM tasks"):
        print(f"  [{t['id']}] P{t['priority']}: {t['title']} (done={t['done']})")
```

---

## 8. ตัวอย่างจริง: Blog Database

```python
import sqlite3
from datetime import datetime
from typing import Optional, List
from contextlib import contextmanager

class BlogDatabase:
    """Blog system database"""
    
    SCHEMA = """
        CREATE TABLE IF NOT EXISTS authors (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT NOT NULL UNIQUE,
            email TEXT NOT NULL UNIQUE,
            bio TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );
        
        CREATE TABLE IF NOT EXISTS posts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            slug TEXT NOT NULL UNIQUE,
            content TEXT NOT NULL,
            excerpt TEXT,
            author_id INTEGER NOT NULL,
            published INTEGER DEFAULT 0,
            views INTEGER DEFAULT 0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (author_id) REFERENCES authors(id)
        );
        
        CREATE TABLE IF NOT EXISTS tags (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL UNIQUE,
            slug TEXT NOT NULL UNIQUE
        );
        
        CREATE TABLE IF NOT EXISTS post_tags (
            post_id INTEGER NOT NULL,
            tag_id INTEGER NOT NULL,
            PRIMARY KEY (post_id, tag_id),
            FOREIGN KEY (post_id) REFERENCES posts(id),
            FOREIGN KEY (tag_id) REFERENCES tags(id)
        );
        
        CREATE TABLE IF NOT EXISTS comments (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            post_id INTEGER NOT NULL,
            author_name TEXT NOT NULL,
            content TEXT NOT NULL,
            approved INTEGER DEFAULT 0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (post_id) REFERENCES posts(id)
        );
        
        CREATE INDEX IF NOT EXISTS idx_posts_slug ON posts(slug);
        CREATE INDEX IF NOT EXISTS idx_posts_published ON posts(published);
    """
    
    def __init__(self, db_path: str = ":memory:") -> None:
        self.db_path = db_path
        self.conn = sqlite3.connect(db_path)
        self.conn.row_factory = sqlite3.Row
        self.conn.execute("PRAGMA foreign_keys = ON")
        self.conn.executescript(self.SCHEMA)
        self.conn.commit()
    
    # --- Authors ---
    def create_author(self, username: str, email: str, bio: str = "") -> int:
        cursor = self.conn.cursor()
        cursor.execute(
            "INSERT INTO authors (username, email, bio) VALUES (?, ?, ?)",
            (username, email, bio)
        )
        self.conn.commit()
        return cursor.lastrowid
    
    # --- Posts ---
    def create_post(
        self,
        title: str,
        content: str,
        author_id: int,
        tags: List[str] = None,
        published: bool = False
    ) -> int:
        slug = title.lower().replace(" ", "-").replace("'", "")
        excerpt = content[:200] + "..." if len(content) > 200 else content
        
        cursor = self.conn.cursor()
        try:
            self.conn.execute("BEGIN")
            
            cursor.execute(
                """INSERT INTO posts (title, slug, content, excerpt, author_id, published)
                   VALUES (?, ?, ?, ?, ?, ?)""",
                (title, slug, content, excerpt, author_id, int(published))
            )
            post_id = cursor.lastrowid
            
            # เพิ่ม tags
            if tags:
                for tag_name in tags:
                    tag_slug = tag_name.lower().replace(" ", "-")
                    cursor.execute(
                        "INSERT OR IGNORE INTO tags (name, slug) VALUES (?, ?)",
                        (tag_name, tag_slug)
                    )
                    cursor.execute("SELECT id FROM tags WHERE slug = ?", (tag_slug,))
                    tag_id = cursor.fetchone()["id"]
                    cursor.execute(
                        "INSERT INTO post_tags (post_id, tag_id) VALUES (?, ?)",
                        (post_id, tag_id)
                    )
            
            self.conn.commit()
            return post_id
        
        except Exception as e:
            self.conn.rollback()
            raise
    
    def get_post(self, slug: str) -> Optional[dict]:
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT p.*, a.username AS author_name
            FROM posts p
            JOIN authors a ON a.id = p.author_id
            WHERE p.slug = ? AND p.published = 1
        """, (slug,))
        row = cursor.fetchone()
        if not row:
            return None
        
        post = dict(row)
        
        # ดึง tags
        cursor.execute("""
            SELECT t.name, t.slug FROM tags t
            JOIN post_tags pt ON pt.tag_id = t.id
            WHERE pt.post_id = ?
        """, (post["id"],))
        post["tags"] = [dict(r) for r in cursor.fetchall()]
        
        # เพิ่ม view count
        self.conn.execute(
            "UPDATE posts SET views = views + 1 WHERE id = ?",
            (post["id"],)
        )
        self.conn.commit()
        
        return post
    
    def get_published_posts(self, page: int = 1, per_page: int = 5) -> dict:
        offset = (page - 1) * per_page
        cursor = self.conn.cursor()
        
        cursor.execute(
            "SELECT COUNT(*) as total FROM posts WHERE published = 1"
        )
        total = cursor.fetchone()["total"]
        
        cursor.execute("""
            SELECT p.id, p.title, p.slug, p.excerpt, p.views,
                   p.created_at, a.username AS author_name
            FROM posts p
            JOIN authors a ON a.id = p.author_id
            WHERE p.published = 1
            ORDER BY p.created_at DESC
            LIMIT ? OFFSET ?
        """, (per_page, offset))
        
        posts = [dict(r) for r in cursor.fetchall()]
        
        return {
            "posts": posts,
            "total": total,
            "page": page,
            "per_page": per_page,
            "total_pages": (total + per_page - 1) // per_page
        }
    
    def add_comment(self, post_id: int, author: str, content: str) -> int:
        cursor = self.conn.cursor()
        cursor.execute(
            "INSERT INTO comments (post_id, author_name, content) VALUES (?, ?, ?)",
            (post_id, author, content)
        )
        self.conn.commit()
        return cursor.lastrowid
    
    def get_stats(self) -> dict:
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT
                (SELECT COUNT(*) FROM authors) as authors,
                (SELECT COUNT(*) FROM posts WHERE published = 1) as published_posts,
                (SELECT COUNT(*) FROM posts WHERE published = 0) as draft_posts,
                (SELECT COUNT(*) FROM comments WHERE approved = 1) as approved_comments,
                (SELECT SUM(views) FROM posts) as total_views
        """)
        return dict(cursor.fetchone())

# ใช้งาน Blog Database
print("=== Blog Database Demo ===\n")
blog = BlogDatabase()

# สร้าง authors
alice_id = blog.create_author("alice", "alice@blog.com", "Python developer")
bob_id = blog.create_author("bob", "bob@blog.com", "Tech writer")
print(f"Authors: alice={alice_id}, bob={bob_id}")

# สร้าง posts
post1_id = blog.create_post(
    title="Getting Started with Python",
    content="Python is an amazing language. " * 20,
    author_id=alice_id,
    tags=["python", "beginner", "programming"],
    published=True
)

post2_id = blog.create_post(
    title="Advanced Django Tips",
    content="These Django tips will make you a better developer. " * 15,
    author_id=alice_id,
    tags=["django", "python", "web"],
    published=True
)

post3_id = blog.create_post(
    title="Draft Post",
    content="This is still a draft...",
    author_id=bob_id,
    published=False  # draft
)

print(f"Posts created: {post1_id}, {post2_id}, {post3_id}")

# ดึง post
post = blog.get_post("getting-started-with-python")
if post:
    print(f"\nPost: {post['title']}")
    print(f"Author: {post['author_name']}")
    print(f"Tags: {[t['name'] for t in post['tags']]}")
    print(f"Views: {post['views']}")

# List posts
result = blog.get_published_posts()
print(f"\nPublished posts ({result['total']} total):")
for p in result["posts"]:
    print(f"  [{p['id']}] {p['title']} by {p['author_name']} ({p['views']} views)")

# Comments
blog.add_comment(post1_id, "Reader A", "Great article!")
blog.add_comment(post1_id, "Reader B", "Very helpful, thanks!")

# Stats
stats = blog.get_stats()
print(f"\nBlog Stats:")
for k, v in stats.items():
    print(f"  {k}: {v}")

blog.conn.close()
```

---

## 9. สรุป Part 035

✅ **sqlite3.connect()** เชื่อมต่อ database (file หรือ :memory:)  
✅ **cursor.execute()** รัน SQL queries  
✅ **Parameterized queries** ด้วย ? ป้องกัน SQL Injection  
✅ **fetchone(), fetchall(), fetchmany()** ดึงผลลัพธ์  
✅ **executemany()** INSERT หลายแถวพร้อมกัน  
✅ **commit() / rollback()** ควบคุม transactions  
✅ **row_factory = sqlite3.Row** เข้าถึง columns ด้วยชื่อ  
✅ **Context Manager** สำหรับ automatic cleanup  
✅ **PRAGMA foreign_keys = ON** เปิด FK constraint  

---

## ➡️ ถัดไป: Part 036 - Logging

*Part 035/100+ | Python Course - Beginner to World-Class*
