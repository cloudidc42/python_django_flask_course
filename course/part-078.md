# Part 078: Flask Blueprints
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- เข้าใจว่า Blueprint คืออะไรและทำไมต้องใช้
- สร้าง Blueprint สำหรับแต่ละส่วนของแอปพลิเคชัน
- ใช้ App Factory Pattern ในการสร้าง Flask app
- ลงทะเบียน Blueprint กับ url_prefix
- แยก template และ static files ต่อ Blueprint

---

## 1. Blueprint คืออะไร?

Blueprint คือวิธีจัดระเบียบโค้ด Flask โดยแบ่งแอปพลิเคชันออกเป็นส่วนๆ ที่เป็นอิสระต่อกัน คล้ายกับ "mini application" ที่รวมกันเป็นแอปใหญ่

### ทำไมต้องใช้ Blueprint?
- **แยกส่วน (Separation of concerns)**: แต่ละ feature มีโค้ดของตัวเอง
- **ง่ายต่อการ maintain**: แก้ไขส่วนหนึ่งโดยไม่กระทบส่วนอื่น
- **นำกลับมาใช้ใหม่ได้**: Blueprint หนึ่งสามารถใช้ในหลาย app
- **ทีมทำงานได้พร้อมกัน**: แต่ละคนดูแล Blueprint ของตัวเอง

### โครงสร้างโปรเจกต์โดยไม่ใช้ Blueprint (ไม่ดี)
```
my_app/
├── app.py          # โค้ดทุกอย่างอยู่ที่นี่ — ยาวมาก!
├── templates/
└── static/
```

### โครงสร้างโปรเจกต์โดยใช้ Blueprint (ดีกว่า)
```
my_app/
├── __init__.py         # App factory
├── auth/
│   ├── __init__.py
│   ├── routes.py
│   ├── templates/auth/
│   └── static/auth/
├── blog/
│   ├── __init__.py
│   ├── routes.py
│   ├── templates/blog/
│   └── static/blog/
├── shop/
│   ├── __init__.py
│   ├── routes.py
│   ├── templates/shop/
│   └── static/shop/
└── config.py
```

---

## 2. สร้าง Blueprint แรก

### ขั้นตอนที่ 1: ติดตั้ง Flask
```bash
pip install flask
```

### ขั้นตอนที่ 2: สร้าง Blueprint สำหรับ auth
```python
# auth/routes.py

from flask import Blueprint, render_template, redirect, url_for, flash, request

# สร้าง Blueprint ชื่อ 'auth'
# __name__ บอก Flask ว่า Blueprint อยู่ที่ไหน
auth_bp = Blueprint(
    'auth',           # ชื่อ Blueprint (ต้อง unique)
    __name__,         # module name
    template_folder='templates',   # โฟลเดอร์ template ของ Blueprint นี้
    static_folder='static',        # โฟลเดอร์ static ของ Blueprint นี้
    url_prefix='/auth'             # prefix สำหรับทุก route ใน Blueprint นี้
)


@auth_bp.route('/login', methods=['GET', 'POST'])
def login():
    """หน้า login — URL จะเป็น /auth/login"""
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        # ตรวจสอบ credential (ตัวอย่าง)
        if username == 'admin' and password == 'secret':
            flash('เข้าสู่ระบบสำเร็จ!', 'success')
            return redirect(url_for('main.index'))
        flash('ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง', 'error')
    return render_template('auth/login.html')


@auth_bp.route('/register', methods=['GET', 'POST'])
def register():
    """หน้าสมัครสมาชิก — URL จะเป็น /auth/register"""
    if request.method == 'POST':
        username = request.form.get('username')
        email = request.form.get('email')
        password = request.form.get('password')
        # บันทึกข้อมูลผู้ใช้ (ตัวอย่าง)
        flash(f'สร้างบัญชีสำเร็จสำหรับ {username}!', 'success')
        return redirect(url_for('auth.login'))
    return render_template('auth/register.html')


@auth_bp.route('/logout')
def logout():
    """ออกจากระบบ — URL จะเป็น /auth/logout"""
    flash('ออกจากระบบแล้ว', 'info')
    return redirect(url_for('main.index'))
```

### ขั้นตอนที่ 3: สร้าง Blueprint สำหรับ blog
```python
# blog/routes.py

from flask import Blueprint, render_template, request, abort

# สร้าง Blueprint สำหรับ blog
blog_bp = Blueprint(
    'blog',
    __name__,
    template_folder='templates',
    static_folder='static',
    url_prefix='/blog'
)

# ข้อมูลตัวอย่าง (จริงๆ ควรดึงจาก database)
POSTS = [
    {'id': 1, 'title': 'บทความแรก', 'content': 'เนื้อหาบทความแรก', 'author': 'admin'},
    {'id': 2, 'title': 'เรียน Python', 'content': 'Python เป็นภาษาที่ยอดเยี่ยม', 'author': 'teacher'},
    {'id': 3, 'title': 'Flask คืออะไร', 'content': 'Flask เป็น web framework', 'author': 'admin'},
]


@blog_bp.route('/')
def index():
    """หน้าแสดงรายการบทความ — URL: /blog/"""
    page = request.args.get('page', 1, type=int)
    per_page = 10
    # ในตัวอย่างนี้แสดงทุกบทความ
    posts = POSTS
    return render_template('blog/index.html', posts=posts, page=page)


@blog_bp.route('/<int:post_id>')
def detail(post_id):
    """หน้าแสดงบทความ — URL: /blog/1"""
    # หาบทความตาม id
    post = next((p for p in POSTS if p['id'] == post_id), None)
    if post is None:
        abort(404)  # ถ้าไม่พบให้แสดง 404
    return render_template('blog/detail.html', post=post)


@blog_bp.route('/create', methods=['GET', 'POST'])
def create():
    """สร้างบทความใหม่ — URL: /blog/create"""
    if request.method == 'POST':
        title = request.form.get('title')
        content = request.form.get('content')
        # บันทึกบทความ (ตัวอย่าง)
        new_post = {
            'id': len(POSTS) + 1,
            'title': title,
            'content': content,
            'author': 'current_user'
        }
        POSTS.append(new_post)
        return redirect(url_for('blog.index'))
    return render_template('blog/create.html')
```

### ขั้นตอนที่ 4: สร้าง Blueprint สำหรับ main
```python
# main/routes.py

from flask import Blueprint, render_template

# Blueprint สำหรับหน้าหลัก (ไม่มี prefix)
main_bp = Blueprint(
    'main',
    __name__,
    template_folder='templates'
)


@main_bp.route('/')
def index():
    """หน้าแรก — URL: /"""
    return render_template('main/index.html')


@main_bp.route('/about')
def about():
    """หน้าเกี่ยวกับ — URL: /about"""
    return render_template('main/about.html')


@main_bp.route('/contact')
def contact():
    """หน้าติดต่อ — URL: /contact"""
    return render_template('main/contact.html')
```

---

## 3. App Factory Pattern

App Factory Pattern คือการสร้าง Flask app ภายใน function แทนที่จะสร้างตรงๆ ที่ระดับ module

### ทำไมต้องใช้ App Factory?
- **Testing**: สร้าง app หลายตัวด้วย config ต่างกันได้
- **Multiple instances**: รัน dev และ production พร้อมกันได้
- **Avoid circular imports**: โหลด extension หลังจาก app ถูกสร้าง

### สร้าง App Factory
```python
# __init__.py (root ของ package)

from flask import Flask
from .config import config


def create_app(config_name='default'):
    """
    App Factory Function
    สร้างและ configure Flask application
    
    Args:
        config_name: ชื่อ configuration ('development', 'production', 'testing')
    
    Returns:
        Flask application instance
    """
    # สร้าง Flask app
    app = Flask(__name__)
    
    # โหลด configuration
    app.config.from_object(config[config_name])
    
    # Initialize extensions
    _init_extensions(app)
    
    # ลงทะเบียน Blueprints
    _register_blueprints(app)
    
    # ลงทะเบียน error handlers
    _register_error_handlers(app)
    
    return app


def _init_extensions(app):
    """Initialize Flask extensions"""
    # เพิ่ม extensions ที่นี่ (database, login manager, etc.)
    # from .extensions import db, login_manager
    # db.init_app(app)
    # login_manager.init_app(app)
    pass


def _register_blueprints(app):
    """ลงทะเบียน Blueprints ทั้งหมด"""
    # Import Blueprints
    from .main.routes import main_bp
    from .auth.routes import auth_bp
    from .blog.routes import blog_bp
    
    # ลงทะเบียน Blueprints กับ app
    app.register_blueprint(main_bp)                    # ไม่มี prefix
    app.register_blueprint(auth_bp, url_prefix='/auth') # override prefix
    app.register_blueprint(blog_bp, url_prefix='/blog') # override prefix


def _register_error_handlers(app):
    """ลงทะเบียน error handlers"""
    
    @app.errorhandler(404)
    def not_found(error):
        from flask import render_template
        return render_template('errors/404.html'), 404
    
    @app.errorhandler(500)
    def server_error(error):
        from flask import render_template
        return render_template('errors/500.html'), 500
    
    @app.errorhandler(403)
    def forbidden(error):
        from flask import render_template
        return render_template('errors/403.html'), 403
```

---

## 4. ลงทะเบียน Blueprint กับ url_prefix

### วิธีการลงทะเบียน Blueprint
```python
# วิธีที่ 1: กำหนด url_prefix ใน Blueprint
blog_bp = Blueprint('blog', __name__, url_prefix='/blog')

# วิธีที่ 2: กำหนด url_prefix ตอนลงทะเบียน (override)
app.register_blueprint(blog_bp, url_prefix='/articles')

# วิธีที่ 3: ไม่มี url_prefix (หน้าหลัก)
app.register_blueprint(main_bp)
```

### ตัวอย่าง URL ที่ได้จาก Blueprint
```python
# Blueprint: auth_bp (url_prefix='/auth')
# Route: /login  →  URL จริง: /auth/login
# Route: /register  →  URL จริง: /auth/register
# Route: /logout  →  URL จริง: /auth/logout

# Blueprint: blog_bp (url_prefix='/blog')
# Route: /  →  URL จริง: /blog/
# Route: /<int:id>  →  URL จริง: /blog/1
# Route: /create  →  URL จริง: /blog/create

# Blueprint: main_bp (ไม่มี prefix)
# Route: /  →  URL จริง: /
# Route: /about  →  URL จริง: /about
```

### url_for ใน Blueprint
```python
# ใน Blueprint ต้องระบุ blueprint_name.function_name
url_for('auth.login')      # → /auth/login
url_for('auth.register')   # → /auth/register
url_for('blog.index')      # → /blog/
url_for('blog.detail', post_id=1)  # → /blog/1
url_for('main.index')      # → /

# ใน template เหมือนกัน
# {{ url_for('auth.login') }}
```

---

## 5. Template และ Static Per Blueprint

### โครงสร้างไฟล์
```
my_app/
├── __init__.py
├── templates/              # Templates ทั่วไป
│   ├── base.html
│   └── errors/
│       ├── 404.html
│       └── 500.html
├── static/                 # Static files ทั่วไป
│   ├── css/
│   │   └── main.css
│   └── js/
│       └── app.js
├── auth/
│   ├── __init__.py
│   ├── routes.py
│   ├── templates/          # Templates ของ auth Blueprint
│   │   └── auth/           # ซ้ำชื่อ Blueprint เพื่อป้องกัน conflict
│   │       ├── login.html
│   │       └── register.html
│   └── static/             # Static files ของ auth Blueprint
│       └── auth/
│           └── auth.css
└── blog/
    ├── __init__.py
    ├── routes.py
    ├── templates/           # Templates ของ blog Blueprint
    │   └── blog/
    │       ├── index.html
    │       └── detail.html
    └── static/              # Static files ของ blog Blueprint
        └── blog/
            └── blog.css
```

### Base Template
```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}My App{% endblock %}</title>
    <!-- Static file จาก main app -->
    <link rel="stylesheet" href="{{ url_for('static', filename='css/main.css') }}">
    {% block extra_css %}{% endblock %}
</head>
<body>
    <nav>
        <a href="{{ url_for('main.index') }}">หน้าแรก</a>
        <a href="{{ url_for('blog.index') }}">บทความ</a>
        <a href="{{ url_for('auth.login') }}">เข้าสู่ระบบ</a>
    </nav>
    
    <!-- Flash messages -->
    {% with messages = get_flashed_messages(with_categories=true) %}
        {% if messages %}
            {% for category, message in messages %}
                <div class="alert alert-{{ category }}">{{ message }}</div>
            {% endfor %}
        {% endif %}
    {% endwith %}
    
    <main>
        {% block content %}{% endblock %}
    </main>
    
    <script src="{{ url_for('static', filename='js/app.js') }}"></script>
    {% block extra_js %}{% endblock %}
</body>
</html>
```

### Auth Template
```html
<!-- auth/templates/auth/login.html -->
{% extends "base.html" %}

{% block title %}เข้าสู่ระบบ{% endblock %}

{% block extra_css %}
    <!-- Static file จาก auth Blueprint -->
    <link rel="stylesheet" href="{{ url_for('auth.static', filename='auth/auth.css') }}">
{% endblock %}

{% block content %}
<div class="login-container">
    <h1>เข้าสู่ระบบ</h1>
    
    <form method="POST" action="{{ url_for('auth.login') }}">
        <div class="form-group">
            <label for="username">ชื่อผู้ใช้:</label>
            <input type="text" id="username" name="username" required>
        </div>
        
        <div class="form-group">
            <label for="password">รหัสผ่าน:</label>
            <input type="password" id="password" name="password" required>
        </div>
        
        <button type="submit">เข้าสู่ระบบ</button>
    </form>
    
    <p>ยังไม่มีบัญชี? <a href="{{ url_for('auth.register') }}">สมัครสมาชิก</a></p>
</div>
{% endblock %}
```

### Blog Template
```html
<!-- blog/templates/blog/index.html -->
{% extends "base.html" %}

{% block title %}บทความทั้งหมด{% endblock %}

{% block extra_css %}
    <link rel="stylesheet" href="{{ url_for('blog.static', filename='blog/blog.css') }}">
{% endblock %}

{% block content %}
<div class="blog-container">
    <h1>บทความทั้งหมด</h1>
    
    {% for post in posts %}
    <article class="post-card">
        <h2><a href="{{ url_for('blog.detail', post_id=post.id) }}">{{ post.title }}</a></h2>
        <p class="author">โดย {{ post.author }}</p>
        <p>{{ post.content[:100] }}...</p>
    </article>
    {% else %}
    <p>ยังไม่มีบทความ</p>
    {% endfor %}
    
    <a href="{{ url_for('blog.create') }}" class="btn">สร้างบทความใหม่</a>
</div>
{% endblock %}
```

---

## 6. โปรเจกต์ตัวอย่างสมบูรณ์

### โครงสร้างโปรเจกต์
```
flask_blueprints_demo/
├── app/
│   ├── __init__.py          # App Factory
│   ├── config.py            # Configuration
│   ├── extensions.py        # Flask extensions
│   ├── templates/           # Global templates
│   │   └── base.html
│   ├── static/              # Global static files
│   ├── main/
│   │   ├── __init__.py
│   │   └── routes.py
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── routes.py
│   │   └── templates/auth/
│   ├── blog/
│   │   ├── __init__.py
│   │   ├── routes.py
│   │   └── templates/blog/
│   └── api/
│       ├── __init__.py
│       └── routes.py
├── run.py                   # Entry point
└── requirements.txt
```

### config.py
```python
# app/config.py

import os


class Config:
    """Base configuration"""
    SECRET_KEY = os.environ.get('SECRET_KEY', 'dev-secret-key-change-in-production')
    DEBUG = False
    TESTING = False


class DevelopmentConfig(Config):
    """Development configuration"""
    DEBUG = True
    DATABASE_URI = 'sqlite:///dev.db'


class ProductionConfig(Config):
    """Production configuration"""
    SECRET_KEY = os.environ.get('SECRET_KEY')  # ต้องตั้งจาก environment
    DATABASE_URI = os.environ.get('DATABASE_URL')


class TestingConfig(Config):
    """Testing configuration"""
    TESTING = True
    DATABASE_URI = 'sqlite:///:memory:'  # ใช้ in-memory database


# Dictionary สำหรับ map ชื่อ config
config = {
    'development': DevelopmentConfig,
    'production': ProductionConfig,
    'testing': TestingConfig,
    'default': DevelopmentConfig
}
```

### extensions.py
```python
# app/extensions.py
# เก็บ Flask extension instances ไว้ที่นี่

# from flask_sqlalchemy import SQLAlchemy
# from flask_login import LoginManager
# from flask_migrate import Migrate

# db = SQLAlchemy()
# login_manager = LoginManager()
# migrate = Migrate()
```

### API Blueprint
```python
# app/api/routes.py
# Blueprint สำหรับ REST API

from flask import Blueprint, jsonify, request

# API Blueprint ไม่มี template เพราะ return JSON
api_bp = Blueprint('api', __name__, url_prefix='/api/v1')


@api_bp.route('/posts', methods=['GET'])
def get_posts():
    """API: ดึงรายการบทความ"""
    # ตัวอย่างข้อมูล
    posts = [
        {'id': 1, 'title': 'บทความแรก', 'author': 'admin'},
        {'id': 2, 'title': 'เรียน Python', 'author': 'teacher'},
    ]
    return jsonify({
        'status': 'success',
        'data': posts,
        'count': len(posts)
    })


@api_bp.route('/posts/<int:post_id>', methods=['GET'])
def get_post(post_id):
    """API: ดึงบทความตาม ID"""
    # ตัวอย่าง
    post = {'id': post_id, 'title': 'บทความ', 'content': 'เนื้อหา'}
    return jsonify({'status': 'success', 'data': post})


@api_bp.route('/posts', methods=['POST'])
def create_post():
    """API: สร้างบทความใหม่"""
    data = request.get_json()
    if not data or 'title' not in data:
        return jsonify({'status': 'error', 'message': 'ต้องระบุ title'}), 400
    
    new_post = {
        'id': 100,  # จริงๆ จะได้จาก database
        'title': data['title'],
        'content': data.get('content', '')
    }
    return jsonify({'status': 'success', 'data': new_post}), 201
```

### run.py
```python
# run.py — Entry point ของแอปพลิเคชัน

import os
from app import create_app

# อ่าน environment จาก environment variable
# ถ้าไม่ตั้ง จะใช้ 'default' (development)
env = os.environ.get('FLASK_ENV', 'default')

# สร้าง app ด้วย factory function
app = create_app(env)

if __name__ == '__main__':
    # รันใน development mode
    app.run(
        host='0.0.0.0',
        port=5000,
        debug=app.config.get('DEBUG', False)
    )
```

---

## 7. Blueprint Hooks

Blueprint มี hooks ที่ทำงานก่อน/หลัง request เฉพาะของ Blueprint นั้น

```python
# blog/routes.py — เพิ่ม hooks

from flask import Blueprint, g, request, current_app

blog_bp = Blueprint('blog', __name__, url_prefix='/blog')


@blog_bp.before_request
def before_blog_request():
    """ทำงานก่อน request ทุกอันใน blog Blueprint"""
    current_app.logger.info(f'Blog request: {request.method} {request.path}')
    # สามารถตรวจสอบ authentication ที่นี่ได้


@blog_bp.after_request
def after_blog_request(response):
    """ทำงานหลัง request ทุกอันใน blog Blueprint"""
    # เพิ่ม header ให้ทุก response ของ blog
    response.headers['X-Blog-Version'] = '1.0'
    return response


@blog_bp.teardown_request
def teardown_blog_request(exception):
    """ทำงานหลังสิ้นสุด request (แม้จะมี exception)"""
    # ปิด database connection เป็นต้น
    pass


@blog_bp.context_processor
def inject_blog_globals():
    """เพิ่มตัวแปรให้ทุก template ใน blog Blueprint"""
    return {
        'blog_title': 'My Awesome Blog',
        'blog_version': '1.0'
    }
```

---

## 8. Blueprint สำหรับ Admin

```python
# admin/routes.py

from flask import Blueprint, render_template, redirect, url_for, flash, request
from functools import wraps

# สร้าง Admin Blueprint
admin_bp = Blueprint(
    'admin',
    __name__,
    url_prefix='/admin',
    template_folder='templates',
    static_folder='static'
)


def admin_required(f):
    """Decorator ตรวจสอบว่าเป็น admin"""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        # ตรวจสอบว่าเป็น admin (ตัวอย่าง)
        is_admin = request.cookies.get('is_admin') == 'true'
        if not is_admin:
            flash('ต้องการสิทธิ์ผู้ดูแลระบบ', 'error')
            return redirect(url_for('main.index'))
        return f(*args, **kwargs)
    return decorated_function


@admin_bp.route('/')
@admin_required
def index():
    """Dashboard ของ admin — URL: /admin/"""
    stats = {
        'total_users': 150,
        'total_posts': 45,
        'total_comments': 320
    }
    return render_template('admin/index.html', stats=stats)


@admin_bp.route('/users')
@admin_required
def users():
    """จัดการผู้ใช้ — URL: /admin/users"""
    # ดึงรายการผู้ใช้ (ตัวอย่าง)
    users_list = [
        {'id': 1, 'username': 'admin', 'email': 'admin@example.com'},
        {'id': 2, 'username': 'user1', 'email': 'user1@example.com'},
    ]
    return render_template('admin/users.html', users=users_list)


@admin_bp.route('/users/<int:user_id>/delete', methods=['POST'])
@admin_required
def delete_user(user_id):
    """ลบผู้ใช้ — URL: /admin/users/1/delete"""
    # ลบผู้ใช้จาก database (ตัวอย่าง)
    flash(f'ลบผู้ใช้ ID {user_id} แล้ว', 'success')
    return redirect(url_for('admin.users'))
```

---

## 9. Nested Blueprint (Flask 2.x+)

```python
# Flask 2.x รองรับ nested blueprints

from flask import Blueprint

# Parent Blueprint
api_v1 = Blueprint('api_v1', __name__, url_prefix='/api/v1')

# Child Blueprints
users_bp = Blueprint('users', __name__)
posts_bp = Blueprint('posts', __name__)

# เพิ่ม routes ใน child blueprints
@users_bp.route('/')
def list_users():
    return {'users': []}

@users_bp.route('/<int:user_id>')
def get_user(user_id):
    return {'user_id': user_id}

@posts_bp.route('/')
def list_posts():
    return {'posts': []}

# ลงทะเบียน child เข้า parent
api_v1.register_blueprint(users_bp, url_prefix='/users')
api_v1.register_blueprint(posts_bp, url_prefix='/posts')

# ลงทะเบียน parent เข้า app
# app.register_blueprint(api_v1)

# URL ที่ได้:
# /api/v1/users/           → api_v1.users.list_users
# /api/v1/users/1          → api_v1.users.get_user
# /api/v1/posts/           → api_v1.posts.list_posts
```

---

## 10. การ Test Blueprint

```python
# tests/test_auth.py

import pytest
from app import create_app


@pytest.fixture
def app():
    """สร้าง app สำหรับ testing"""
    app = create_app('testing')
    yield app


@pytest.fixture
def client(app):
    """สร้าง test client"""
    return app.test_client()


def test_login_page(client):
    """ทดสอบว่าหน้า login โหลดได้"""
    response = client.get('/auth/login')
    assert response.status_code == 200
    assert b'เข้าสู่ระบบ' in response.data


def test_login_success(client):
    """ทดสอบ login สำเร็จ"""
    response = client.post('/auth/login', data={
        'username': 'admin',
        'password': 'secret'
    }, follow_redirects=True)
    assert response.status_code == 200
    assert b'เข้าสู่ระบบสำเร็จ' in response.data


def test_login_failure(client):
    """ทดสอบ login ล้มเหลว"""
    response = client.post('/auth/login', data={
        'username': 'wrong',
        'password': 'wrong'
    })
    assert b'ไม่ถูกต้อง' in response.data


def test_blog_index(client):
    """ทดสอบหน้าบทความ"""
    response = client.get('/blog/')
    assert response.status_code == 200


def test_blog_post_not_found(client):
    """ทดสอบเมื่อไม่พบบทความ"""
    response = client.get('/blog/999')
    assert response.status_code == 404
```

---

## 11. รัน Application

```bash
# ติดตั้ง dependencies
pip install flask

# ตั้ง environment variables
export FLASK_ENV=development
export FLASK_APP=run.py

# รัน app
flask run

# หรือรัน run.py โดยตรง
python run.py
```

### ทดสอบ URL
```
http://localhost:5000/           → main.index
http://localhost:5000/about      → main.about
http://localhost:5000/auth/login → auth.login
http://localhost:5000/blog/      → blog.index
http://localhost:5000/api/v1/posts → api.get_posts
```

---

## 12. Best Practices สำหรับ Blueprint

### ✅ ทำ
```python
# 1. ตั้งชื่อ Blueprint ให้ชัดเจน
auth_bp = Blueprint('auth', __name__)

# 2. ใช้ url_for ด้วย blueprint_name เสมอ
url_for('auth.login')  # ถูก
url_for('login')       # ผิด! จะไม่พบ function

# 3. ซ้ำชื่อโฟลเดอร์ใน templates เพื่อป้องกัน conflict
# auth/templates/auth/login.html  ← ดี
# auth/templates/login.html       ← อาจ conflict กับ Blueprint อื่น

# 4. แยก Blueprint ตาม feature ไม่ใช่ตาม type
# ดี: auth/, blog/, shop/, admin/
# ไม่ดี: models/, views/, controllers/
```

### ❌ ไม่ทำ
```python
# 1. อย่าสร้าง Blueprint ที่มีชื่อซ้ำ
bp1 = Blueprint('users', __name__)  # ชื่อซ้ำ!
bp2 = Blueprint('users', __name__)  # จะเกิด error

# 2. อย่า import app ใน Blueprint (circular import)
from app import app  # ผิด!
# ใช้ current_app แทน
from flask import current_app  # ถูก!
current_app.config['SECRET_KEY']  # ถูก!
```

---

## 13. สรุป Part 078

✅ **Blueprint** คือวิธีแบ่งแอปพลิเคชัน Flask ออกเป็นส่วนๆ ที่จัดการได้ง่าย  
✅ **App Factory Pattern** ช่วยให้สร้าง app หลายตัวด้วย config ต่างกันได้  
✅ **url_prefix** กำหนด URL prefix สำหรับทุก route ใน Blueprint  
✅ **Template folder** แต่ละ Blueprint มี template ของตัวเองได้  
✅ **Static folder** แต่ละ Blueprint มี static files ของตัวเองได้  
✅ **url_for** ต้องใช้รูปแบบ `blueprint_name.function_name`  
✅ **Blueprint hooks** (before_request, after_request) ทำงานเฉพาะ Blueprint นั้น  

---

## ➡️ ถัดไป: Part 079 - Flask SQLAlchemy

*Part 078/100+ | Python Course - Beginner to World-Class*
