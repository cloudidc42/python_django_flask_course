# Part 079: Flask SQLAlchemy
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- ติดตั้งและ configure Flask-SQLAlchemy
- สร้าง Model classes สำหรับ database
- ใช้ relationships ระหว่าง models
- ทำ CRUD operations กับ database
- ใช้ Flask-Migrate สำหรับ database migrations

---

## 1. Flask-SQLAlchemy คืออะไร?

Flask-SQLAlchemy เป็น extension ที่รวม SQLAlchemy เข้ากับ Flask ทำให้การทำงานกับ database ง่ายขึ้น

### ติดตั้ง
```bash
pip install flask-sqlalchemy flask-migrate
```

### Configuration พื้นฐาน
```python
# app.py

from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

app = Flask(__name__)

# กำหนด database URI
# SQLite (สำหรับ development)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///myapp.db'

# PostgreSQL (สำหรับ production)
# app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://user:pass@localhost/mydb'

# MySQL
# app.config['SQLALCHEMY_DATABASE_URI'] = 'mysql+pymysql://user:pass@localhost/mydb'

# ปิดการ track modifications (ประหยัด memory)
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

# Echo SQL queries (สำหรับ debug)
app.config['SQLALCHEMY_ECHO'] = True  # เปิดเฉพาะ development

# สร้าง db instance
db = SQLAlchemy(app)

# สร้าง migrate instance
migrate = Migrate(app, db)
```

---

## 2. สร้าง Models

### Model พื้นฐาน
```python
# models.py

from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

db = SQLAlchemy()


class User(db.Model):
    """Model สำหรับผู้ใช้"""
    
    # ชื่อ table ใน database (ถ้าไม่ระบุจะใช้ชื่อ class เป็น snake_case)
    __tablename__ = 'users'
    
    # Primary Key
    id = db.Column(db.Integer, primary_key=True)
    
    # String columns
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    
    # Password (เก็บเป็น hash เท่านั้น!)
    password_hash = db.Column(db.String(256), nullable=False)
    
    # Boolean
    is_active = db.Column(db.Boolean, default=True)
    is_admin = db.Column(db.Boolean, default=False)
    
    # DateTime
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Optional text
    bio = db.Column(db.Text, nullable=True)
    
    def __repr__(self):
        """แสดงผลเมื่อ print object"""
        return f'<User {self.username}>'
    
    def to_dict(self):
        """แปลง Model เป็น dictionary สำหรับ JSON response"""
        return {
            'id': self.id,
            'username': self.username,
            'email': self.email,
            'is_active': self.is_active,
            'created_at': self.created_at.isoformat() if self.created_at else None
        }
```

### ประเภท Column ที่ใช้บ่อย
```python
class ColumnTypes(db.Model):
    """ตัวอย่างประเภท Column ต่างๆ"""
    
    id = db.Column(db.Integer, primary_key=True)
    
    # ข้อความ
    name = db.Column(db.String(100))     # VARCHAR(100)
    description = db.Column(db.Text)     # TEXT
    
    # ตัวเลข
    age = db.Column(db.Integer)           # INTEGER
    price = db.Column(db.Float)           # FLOAT
    balance = db.Column(db.Numeric(10, 2))  # DECIMAL(10,2) — แม่นยำกว่า Float
    
    # Boolean
    is_active = db.Column(db.Boolean, default=True)
    
    # วันเวลา
    birthday = db.Column(db.Date)
    created_at = db.Column(db.DateTime)
    last_seen = db.Column(db.DateTime)
    
    # JSON (SQLite 3.38+, PostgreSQL, MySQL 5.7+)
    metadata_ = db.Column(db.JSON)
    
    # Enum
    status = db.Column(db.Enum('active', 'inactive', 'banned'), default='active')
    
    # Auto increment (นอกจาก primary key)
    sequence = db.Column(db.Integer, autoincrement=True)
```

### Column Options
```python
class ColumnOptions(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    
    # unique — ห้ามซ้ำ
    email = db.Column(db.String(120), unique=True)
    
    # nullable — ห้ามเป็น NULL
    username = db.Column(db.String(80), nullable=False)
    
    # default — ค่าเริ่มต้น
    is_active = db.Column(db.Boolean, default=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    # index — สร้าง index เพื่อค้นหาเร็วขึ้น
    last_name = db.Column(db.String(80), index=True)
    
    # server_default — ค่าเริ่มต้นจาก database server
    created_at2 = db.Column(db.DateTime, server_default=db.func.now())
```

---

## 3. Relationships

### One-to-Many (หนึ่งต่อหลาย)

```python
# models.py

class User(db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    
    # Relationship: User มีหลาย Post
    # backref สร้าง .user attribute ใน Post โดยอัตโนมัติ
    posts = db.relationship('Post', backref='author', lazy='dynamic')
    
    def __repr__(self):
        return f'<User {self.username}>'


class Post(db.Model):
    __tablename__ = 'posts'
    
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    content = db.Column(db.Text, nullable=False)
    
    # Foreign Key — เชื่อมกับ User
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    published = db.Column(db.Boolean, default=False)
    
    # Relationship: Post มีหลาย Comment
    comments = db.relationship('Comment', backref='post', lazy='dynamic',
                               cascade='all, delete-orphan')
    
    def __repr__(self):
        return f'<Post {self.title}>'


class Comment(db.Model):
    __tablename__ = 'comments'
    
    id = db.Column(db.Integer, primary_key=True)
    content = db.Column(db.Text, nullable=False)
    
    # Foreign Keys
    post_id = db.Column(db.Integer, db.ForeignKey('posts.id'), nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    # Relationship กับ User
    user = db.relationship('User', backref='comments')
    
    def __repr__(self):
        return f'<Comment {self.id}>'
```

### Many-to-Many (หลายต่อหลาย)

```python
# Association table สำหรับ Many-to-Many
post_tags = db.Table(
    'post_tags',
    db.Column('post_id', db.Integer, db.ForeignKey('posts.id'), primary_key=True),
    db.Column('tag_id', db.Integer, db.ForeignKey('tags.id'), primary_key=True)
)

# หรือใช้ user_followers สำหรับ self-referential relationship
followers = db.Table(
    'followers',
    db.Column('follower_id', db.Integer, db.ForeignKey('users.id'), primary_key=True),
    db.Column('followed_id', db.Integer, db.ForeignKey('users.id'), primary_key=True)
)


class Tag(db.Model):
    __tablename__ = 'tags'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), unique=True, nullable=False)
    slug = db.Column(db.String(50), unique=True, nullable=False)
    
    def __repr__(self):
        return f'<Tag {self.name}>'


class Post(db.Model):
    __tablename__ = 'posts'
    
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    content = db.Column(db.Text, nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    # Many-to-Many กับ Tag ผ่าน association table
    tags = db.relationship('Tag', secondary=post_tags,
                           backref=db.backref('posts', lazy='dynamic'))


class User(db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    
    # Self-referential Many-to-Many: User ติดตาม User
    followed = db.relationship(
        'User',
        secondary=followers,
        primaryjoin=(followers.c.follower_id == id),
        secondaryjoin=(followers.c.followed_id == id),
        backref=db.backref('followers', lazy='dynamic'),
        lazy='dynamic'
    )
    
    def follow(self, user):
        """ติดตาม user"""
        if not self.is_following(user):
            self.followed.append(user)
    
    def unfollow(self, user):
        """เลิกติดตาม user"""
        if self.is_following(user):
            self.followed.remove(user)
    
    def is_following(self, user):
        """ตรวจสอบว่าติดตาม user อยู่หรือไม่"""
        return self.followed.filter(
            followers.c.followed_id == user.id
        ).count() > 0
```

### One-to-One (หนึ่งต่อหนึ่ง)

```python
class UserProfile(db.Model):
    """ข้อมูลเพิ่มเติมของ User (One-to-One)"""
    
    __tablename__ = 'user_profiles'
    
    id = db.Column(db.Integer, primary_key=True)
    
    # Foreign Key + unique = One-to-One
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), unique=True, nullable=False)
    
    first_name = db.Column(db.String(80))
    last_name = db.Column(db.String(80))
    phone = db.Column(db.String(20))
    avatar_url = db.Column(db.String(255))
    website = db.Column(db.String(255))
    
    # uselist=False บอกว่าเป็น One-to-One ไม่ใช่ One-to-Many
    user = db.relationship('User', backref=db.backref('profile', uselist=False))
```

---

## 4. CRUD Operations

### Create (เพิ่มข้อมูล)
```python
from flask import Flask, jsonify, request
from models import db, User, Post, Tag

# เพิ่ม user ใหม่
def create_user(username, email, password_hash):
    """สร้าง User ใหม่"""
    user = User(
        username=username,
        email=email,
        password_hash=password_hash
    )
    db.session.add(user)
    db.session.commit()
    return user


# เพิ่มหลาย records พร้อมกัน
def create_bulk_users(users_data):
    """สร้าง Users หลายคนพร้อมกัน"""
    users = [User(**data) for data in users_data]
    db.session.add_all(users)
    db.session.commit()
    return users


# เพิ่ม Post พร้อม Tags
def create_post_with_tags(title, content, user_id, tag_names):
    """สร้าง Post พร้อม Tags"""
    # สร้างหรือดึง Tags
    tags = []
    for tag_name in tag_names:
        tag = Tag.query.filter_by(name=tag_name).first()
        if not tag:
            tag = Tag(name=tag_name, slug=tag_name.lower().replace(' ', '-'))
            db.session.add(tag)
        tags.append(tag)
    
    # สร้าง Post
    post = Post(
        title=title,
        content=content,
        user_id=user_id
    )
    post.tags = tags
    
    db.session.add(post)
    db.session.commit()
    return post
```

### Read (อ่านข้อมูล)
```python
def read_examples():
    """ตัวอย่างการ query ข้อมูล"""
    
    # ดึงทุก record
    all_users = User.query.all()
    
    # ดึงด้วย primary key
    user = User.query.get(1)           # deprecated ใน SQLAlchemy 2.0
    user = db.session.get(User, 1)     # วิธีใหม่
    
    # filter_by (ใช้สำหรับ exact match)
    user = User.query.filter_by(username='john').first()
    active_users = User.query.filter_by(is_active=True).all()
    
    # filter (ใช้สำหรับ complex queries)
    from sqlalchemy import and_, or_, not_
    
    # WHERE username = 'john' AND is_active = True
    user = User.query.filter(
        and_(User.username == 'john', User.is_active == True)
    ).first()
    
    # WHERE email LIKE '%@gmail.com'
    gmail_users = User.query.filter(
        User.email.like('%@gmail.com')
    ).all()
    
    # WHERE id IN (1, 2, 3)
    some_users = User.query.filter(
        User.id.in_([1, 2, 3])
    ).all()
    
    # ORDER BY
    users_by_date = User.query.order_by(User.created_at.desc()).all()
    
    # LIMIT และ OFFSET
    page = 1
    per_page = 10
    users_page = User.query.offset((page-1)*per_page).limit(per_page).all()
    
    # COUNT
    user_count = User.query.count()
    admin_count = User.query.filter_by(is_admin=True).count()
    
    # EXISTS
    has_admin = User.query.filter_by(is_admin=True).first() is not None
    
    # Pagination (built-in)
    pagination = User.query.paginate(page=1, per_page=10, error_out=False)
    users = pagination.items
    total = pagination.total
    pages = pagination.pages
    
    return all_users


def search_posts(keyword, page=1, per_page=10):
    """ค้นหา Posts"""
    query = Post.query.filter(
        or_(
            Post.title.ilike(f'%{keyword}%'),   # case-insensitive
            Post.content.ilike(f'%{keyword}%')
        )
    ).filter_by(published=True)
    
    # Join กับ User เพื่อดึงข้อมูล author
    query = query.join(User).filter(User.is_active == True)
    
    # เรียงตาม date ล่าสุด
    query = query.order_by(Post.created_at.desc())
    
    # Paginate
    return query.paginate(page=page, per_page=per_page, error_out=False)
```

### Update (แก้ไขข้อมูล)
```python
def update_examples():
    """ตัวอย่างการ update ข้อมูล"""
    
    # วิธีที่ 1: แก้ไข attribute โดยตรง
    user = db.session.get(User, 1)
    if user:
        user.email = 'newemail@example.com'
        user.is_active = False
        db.session.commit()
    
    # วิธีที่ 2: ใช้ query().update()
    User.query.filter_by(is_active=False).update({'is_active': True})
    db.session.commit()
    
    # วิธีที่ 3: Bulk update
    db.session.query(User).filter(
        User.created_at < datetime(2020, 1, 1)
    ).update({'is_active': False})
    db.session.commit()


def update_user(user_id, **kwargs):
    """Update User โดยระบุ fields ที่ต้องการเปลี่ยน"""
    user = db.session.get(User, user_id)
    if not user:
        return None
    
    # อัปเดตเฉพาะ fields ที่ส่งมา
    allowed_fields = {'username', 'email', 'bio', 'is_active'}
    for key, value in kwargs.items():
        if key in allowed_fields:
            setattr(user, key, value)
    
    user.updated_at = datetime.utcnow()
    db.session.commit()
    return user
```

### Delete (ลบข้อมูล)
```python
def delete_examples():
    """ตัวอย่างการ delete ข้อมูล"""
    
    # ลบ record เดียว
    user = db.session.get(User, 1)
    if user:
        db.session.delete(user)
        db.session.commit()
    
    # ลบตาม condition
    User.query.filter_by(is_active=False).delete()
    db.session.commit()
    
    # Soft delete (ไม่ลบจริง แค่ mark ว่าลบแล้ว)
    user = db.session.get(User, 1)
    if user:
        user.deleted_at = datetime.utcnow()
        user.is_deleted = True
        db.session.commit()
```

---

## 5. Flask-Migrate สำหรับ Database Migrations

### ติดตั้งและ setup
```bash
pip install flask-migrate
```

```python
# app.py

from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///myapp.db'

db = SQLAlchemy(app)
migrate = Migrate(app, db)

# Import models เพื่อให้ Migrate รู้จัก
from models import User, Post, Comment, Tag  # noqa
```

### คำสั่ง Flask-Migrate
```bash
# 1. Initialize migrations directory (ทำครั้งแรกครั้งเดียว)
flask db init

# 2. สร้าง migration file จาก model changes
flask db migrate -m "Initial migration"

# 3. Apply migration ลง database
flask db upgrade

# 4. Rollback migration (ย้อนกลับ 1 step)
flask db downgrade

# 5. ดูประวัติ migrations
flask db history

# 6. ดู current migration
flask db current

# 7. Downgrade ไป revision เฉพาะ
flask db downgrade <revision_id>

# 8. Upgrade ไป revision เฉพาะ
flask db upgrade <revision_id>
```

### ตัวอย่าง Migration File
```python
# migrations/versions/abc123_add_user_avatar.py
# Auto-generated แต่สามารถแก้ไขได้

"""add user avatar

Revision ID: abc123
Revises: prev_revision
Create Date: 2024-01-15 10:00:00.000000
"""

from alembic import op
import sqlalchemy as sa

# revision identifiers
revision = 'abc123'
down_revision = 'prev_revision'
branch_labels = None
depends_on = None


def upgrade():
    """เพิ่ม column avatar_url ใน table users"""
    op.add_column('users',
        sa.Column('avatar_url', sa.String(255), nullable=True)
    )


def downgrade():
    """ลบ column avatar_url ออกจาก table users"""
    op.drop_column('users', 'avatar_url')
```

### Custom Migration (Data Migration)
```python
# migrations/versions/def456_populate_slugs.py

"""populate post slugs

Revision ID: def456
Revises: abc123
Create Date: 2024-01-16 10:00:00.000000
"""

from alembic import op
import sqlalchemy as sa
from sqlalchemy.sql import text


def upgrade():
    """เพิ่ม slug column และเติมข้อมูล"""
    # เพิ่ม column (nullable=True ก่อน)
    op.add_column('posts',
        sa.Column('slug', sa.String(200), nullable=True)
    )
    
    # เติมข้อมูล slug จาก title
    connection = op.get_bind()
    posts = connection.execute(text("SELECT id, title FROM posts"))
    
    for post in posts:
        slug = post.title.lower().replace(' ', '-')
        connection.execute(
            text("UPDATE posts SET slug = :slug WHERE id = :id"),
            {'slug': slug, 'id': post.id}
        )
    
    # เปลี่ยนเป็น NOT NULL หลังเติมข้อมูลแล้ว
    op.alter_column('posts', 'slug', nullable=False)
    
    # สร้าง unique index
    op.create_unique_constraint('uq_posts_slug', 'posts', ['slug'])


def downgrade():
    """ลบ slug column"""
    op.drop_constraint('uq_posts_slug', 'posts', type_='unique')
    op.drop_column('posts', 'slug')
```

---

## 6. Model Methods และ Class Methods

```python
# models.py — เพิ่ม methods ใน Model

import hashlib
from datetime import datetime
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()


class User(db.Model):
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(256))
    is_active = db.Column(db.Boolean, default=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    posts = db.relationship('Post', backref='author', lazy='dynamic',
                            cascade='all, delete-orphan')
    
    # ---- Instance Methods ----
    
    def set_password(self, password):
        """Hash และบันทึก password"""
        # ในการใช้จริงควรใช้ bcrypt หรือ werkzeug
        self.password_hash = hashlib.sha256(password.encode()).hexdigest()
    
    def check_password(self, password):
        """ตรวจสอบ password"""
        return self.password_hash == hashlib.sha256(password.encode()).hexdigest()
    
    def get_posts(self, published_only=True):
        """ดึง posts ของ user"""
        query = self.posts
        if published_only:
            query = query.filter_by(published=True)
        return query.order_by(Post.created_at.desc()).all()
    
    def to_dict(self, include_posts=False):
        """แปลงเป็น dictionary"""
        data = {
            'id': self.id,
            'username': self.username,
            'email': self.email,
            'is_active': self.is_active,
            'created_at': self.created_at.isoformat() if self.created_at else None,
            'post_count': self.posts.count()
        }
        if include_posts:
            data['posts'] = [p.to_dict() for p in self.get_posts()]
        return data
    
    # ---- Class Methods ----
    
    @classmethod
    def get_by_username(cls, username):
        """ดึง User จาก username"""
        return cls.query.filter_by(username=username).first()
    
    @classmethod
    def get_by_email(cls, email):
        """ดึง User จาก email"""
        return cls.query.filter_by(email=email).first()
    
    @classmethod
    def get_active_users(cls):
        """ดึง Users ที่ active ทั้งหมด"""
        return cls.query.filter_by(is_active=True).all()
    
    @classmethod
    def create(cls, username, email, password):
        """สร้าง User ใหม่"""
        user = cls(username=username, email=email)
        user.set_password(password)
        db.session.add(user)
        db.session.commit()
        return user
    
    # ---- Static Methods ----
    
    @staticmethod
    def validate_email(email):
        """ตรวจสอบรูปแบบ email"""
        import re
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return bool(re.match(pattern, email))
    
    def __repr__(self):
        return f'<User {self.username}>'
```

---

## 7. Advanced Queries

```python
# advanced_queries.py

from models import db, User, Post, Comment, Tag
from sqlalchemy import func, and_, or_, desc


def advanced_query_examples():
    """ตัวอย่าง queries ขั้นสูง"""
    
    # JOIN queries
    # ดึง Posts พร้อมข้อมูล User
    results = db.session.query(Post, User).join(User).filter(
        Post.published == True
    ).all()
    
    for post, user in results:
        print(f'{post.title} by {user.username}')
    
    # COUNT ด้วย GROUP BY
    # นับจำนวน posts ต่อ user
    from sqlalchemy import func
    post_counts = db.session.query(
        User.username,
        func.count(Post.id).label('post_count')
    ).join(Post).group_by(User.id).all()
    
    for username, count in post_counts:
        print(f'{username}: {count} posts')
    
    # Subquery
    # ดึง Users ที่มี posts มากกว่า 5 บทความ
    active_writers = db.session.query(User).filter(
        User.id.in_(
            db.session.query(Post.user_id).group_by(
                Post.user_id
            ).having(func.count(Post.id) > 5)
        )
    ).all()
    
    # Aggregate functions
    stats = db.session.query(
        func.count(User.id).label('total_users'),
        func.max(User.created_at).label('latest_user'),
        func.min(User.created_at).label('oldest_user')
    ).first()
    
    print(f'Total users: {stats.total_users}')
    
    # HAVING clause
    prolific_users = db.session.query(
        User.username,
        func.count(Post.id).label('post_count')
    ).join(Post, Post.user_id == User.id, isouter=True).group_by(
        User.id
    ).having(
        func.count(Post.id) >= 3
    ).order_by(
        desc('post_count')
    ).all()
    
    return prolific_users
```

---

## 8. ตัวอย่างใช้งานจริง: Blog API

```python
# app.py — Complete Blog API

from flask import Flask, jsonify, request, abort
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from datetime import datetime

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///blog.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)
migrate = Migrate(app, db)


# Models
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    posts = db.relationship('Post', backref='author', lazy='dynamic')


class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    content = db.Column(db.Text, nullable=False)
    published = db.Column(db.Boolean, default=False)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)


# Routes
@app.route('/api/users', methods=['GET'])
def get_users():
    users = User.query.all()
    return jsonify([{
        'id': u.id,
        'username': u.username,
        'email': u.email,
        'post_count': u.posts.count()
    } for u in users])


@app.route('/api/users', methods=['POST'])
def create_user():
    data = request.get_json()
    
    if not data or not data.get('username') or not data.get('email'):
        abort(400, description='username และ email จำเป็นต้องระบุ')
    
    # ตรวจสอบซ้ำ
    if User.query.filter_by(username=data['username']).first():
        abort(409, description='username นี้มีอยู่แล้ว')
    
    user = User(
        username=data['username'],
        email=data['email']
    )
    db.session.add(user)
    db.session.commit()
    
    return jsonify({'id': user.id, 'username': user.username}), 201


@app.route('/api/posts', methods=['GET'])
def get_posts():
    page = request.args.get('page', 1, type=int)
    per_page = request.args.get('per_page', 10, type=int)
    
    pagination = Post.query.filter_by(
        published=True
    ).order_by(
        Post.created_at.desc()
    ).paginate(page=page, per_page=per_page, error_out=False)
    
    posts = [{
        'id': p.id,
        'title': p.title,
        'author': p.author.username,
        'created_at': p.created_at.isoformat()
    } for p in pagination.items]
    
    return jsonify({
        'posts': posts,
        'total': pagination.total,
        'pages': pagination.pages,
        'current_page': page
    })


@app.route('/api/posts/<int:post_id>', methods=['PUT'])
def update_post(post_id):
    post = db.session.get(Post, post_id)
    if not post:
        abort(404, description='ไม่พบบทความ')
    
    data = request.get_json()
    
    if 'title' in data:
        post.title = data['title']
    if 'content' in data:
        post.content = data['content']
    if 'published' in data:
        post.published = data['published']
    
    db.session.commit()
    
    return jsonify({'id': post.id, 'title': post.title, 'published': post.published})


@app.route('/api/posts/<int:post_id>', methods=['DELETE'])
def delete_post(post_id):
    post = db.session.get(Post, post_id)
    if not post:
        abort(404, description='ไม่พบบทความ')
    
    db.session.delete(post)
    db.session.commit()
    
    return '', 204


if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(debug=True)
```

---

## 9. สรุป Part 079

✅ **Flask-SQLAlchemy** ทำให้การใช้ SQLAlchemy กับ Flask ง่ายขึ้น  
✅ **Model** คือ Python class ที่ map กับ database table  
✅ **Column types** มีหลายประเภท: Integer, String, Text, Boolean, DateTime, etc.  
✅ **Relationships**: One-to-Many, Many-to-Many, One-to-One  
✅ **CRUD**: Create ด้วย db.session.add(), Read ด้วย query(), Update ด้วยแก้ attribute, Delete ด้วย db.session.delete()  
✅ **Flask-Migrate** จัดการ database schema changes ผ่าน `flask db migrate` และ `flask db upgrade`  

---

## ➡️ ถัดไป: Part 080 - Flask Forms

*Part 079/100+ | Python Course - Beginner to World-Class*
