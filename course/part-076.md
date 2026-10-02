# Part 076 - Flask Introduction

## เป้าหมายการเรียนรู้

- เข้าใจว่า Flask คืออะไรและต่างจาก Django อย่างไร
- ติดตั้ง Flask และสร้าง Hello World application
- เข้าใจ Application Factory pattern
- จัดการ Configuration สำหรับหลายสภาพแวดล้อม
- สร้าง simple web application ด้วย Flask

---

## 1. Flask คืออะไร?

Flask เป็น **micro web framework** สำหรับ Python พัฒนาโดย Armin Ronacher ในปี 2010 คำว่า "micro" ไม่ได้หมายความว่า Flask ทำอะไรได้น้อย แต่หมายความว่า Flask มีแค่ core ที่จำเป็น และให้นักพัฒนาเลือก extension ที่ต้องการเอง

Flask สร้างบน:
- **Werkzeug** - WSGI utility library
- **Jinja2** - Template engine
- **Click** - Command line interface

### 1.1 ปรัชญาของ Flask

```
"Flask is a microframework for Python based on Werkzeug, Jinja 2 and good intentions."
- Armin Ronacher
```

Flask ยึดหลัก:
1. **Simplicity** - เริ่มต้นง่าย ไม่มี boilerplate มาก
2. **Flexibility** - เลือกใช้ tools ที่ต้องการได้
3. **Fine-grained control** - ควบคุมทุกอย่างได้เอง
4. **Convention over configuration (บางส่วน)** - แต่ไม่บังคับ

---

## 2. Flask vs Django

| หัวข้อ | Flask | Django |
|--------|-------|--------|
| ขนาด | Micro framework | Full-stack framework |
| ORM | ไม่มีในตัว (ใช้ SQLAlchemy) | มี Django ORM |
| Admin | ไม่มีในตัว | มี Django Admin |
| Authentication | ต้องติดตั้ง extension | มีในตัว |
| Form handling | ต้องติดตั้ง WTForms | มี Django Forms |
| Learning curve | ต่ำกว่า | สูงกว่า |
| Flexibility | สูงมาก | ต่ำกว่า (มี convention) |
| เหมาะกับ | API, Microservices, Small-Medium apps | Large apps, Rapid development |
| Performance | เร็วกว่าเล็กน้อย | ช้ากว่าเล็กน้อย |
| Community | ใหญ่ | ใหญ่มาก |

### 2.1 เลือกใช้ Flask เมื่อ

```
✅ ต้องการ API สำหรับ microservices
✅ โปรเจกต์ขนาดเล็กถึงกลาง
✅ ต้องการ flexibility สูง
✅ ทีมมีประสบการณ์ Python มาก
✅ ต้องการควบคุม architecture เอง
✅ สร้าง prototype เร็วๆ
```

### 2.2 เลือกใช้ Django เมื่อ

```
✅ โปรเจกต์ขนาดใหญ่
✅ ต้องการ Admin panel
✅ ต้องการ batteries included
✅ ทีมยังไม่มีประสบการณ์มาก
✅ ต้องการ rapid development
```

---

## 3. ติดตั้ง Flask

### 3.1 สร้าง Virtual Environment

```bash
# สร้าง project directory
mkdir my_flask_app
cd my_flask_app

# สร้าง virtual environment
python -m venv venv

# activate (Linux/Mac)
source venv/bin/activate

# activate (Windows)
venv\Scripts\activate

# ตรวจสอบว่า activate แล้ว
which python  # ควรชี้ไปที่ venv
```

### 3.2 ติดตั้ง Flask

```bash
# ติดตั้ง Flask
pip install Flask

# ตรวจสอบ version
python -c "import flask; print(flask.__version__)"
# Output: 3.0.0 (หรือ version ล่าสุด)

# ติดตั้ง packages ที่มักใช้ร่วมกัน
pip install Flask flask-sqlalchemy flask-migrate flask-login python-dotenv

# บันทึก requirements
pip freeze > requirements.txt
```

### 3.3 โครงสร้างไฟล์เบื้องต้น

```
my_flask_app/
├── venv/
├── app.py          # หรือ run.py
├── requirements.txt
└── .env
```

---

## 4. Hello World - Flask แบบเรียบง่าย

```python
# app.py - Flask ขั้นต้นที่สุด

from flask import Flask

# สร้าง Flask instance
# __name__ คือชื่อของ module นี้
# Flask ใช้ __name__ เพื่อหา root path ของ application
app = Flask(__name__)

# สร้าง route สำหรับ URL "/"
@app.route('/')
def hello_world():
    """หน้าแรกของ application"""
    return 'Hello, World!'

# รัน application เมื่อ execute ไฟล์นี้โดยตรง
if __name__ == '__main__':
    # debug=True: auto-reload เมื่อแก้ไขโค้ด + แสดง error ละเอียด
    app.run(debug=True)
```

```bash
# รัน application
python app.py

# Output:
# * Serving Flask app 'app'
# * Debug mode: on
# * Running on http://127.0.0.1:5000
# Press CTRL+C to quit
```

เปิด browser ไปที่ `http://127.0.0.1:5000` จะเห็น "Hello, World!"

---

## 5. Flask Application Object

```python
# ทำความเข้าใจ Flask(__name__)

from flask import Flask

# __name__ คือ '__main__' เมื่อรันตรง
# หรือชื่อ module เมื่อ import
app = Flask(__name__)

# สามารถกำหนด static_folder และ template_folder เองได้
app = Flask(
    __name__,
    static_folder='assets',      # default: 'static'
    template_folder='templates',  # default: 'templates'
    static_url_path='/files'      # default: '/static'
)

# ดู configuration ของ app
print(app.config)
print(app.root_path)  # absolute path ของ app
```

---

## 6. Configuration Management

### 6.1 วิธีตั้งค่า Configuration

```python
# วิธีที่ 1: ตั้งค่าโดยตรง
app = Flask(__name__)
app.config['SECRET_KEY'] = 'my-secret-key'
app.config['DEBUG'] = True
app.config['DATABASE_URI'] = 'sqlite:///myapp.db'

# วิธีที่ 2: ใช้ dict
app.config.update({
    'SECRET_KEY': 'my-secret-key',
    'DEBUG': True,
})

# วิธีที่ 3: จาก environment variable
import os
app.config['SECRET_KEY'] = os.environ.get('SECRET_KEY', 'fallback-key')
```

### 6.2 Config Classes Pattern

```python
# config.py - ไฟล์ configuration หลัก

import os
from dotenv import load_dotenv

# โหลด .env ไฟล์
load_dotenv()

class Config:
    """Base configuration - ตั้งค่าที่ใช้ร่วมกันทุกสภาพแวดล้อม"""
    # จำเป็นสำหรับ session และ CSRF protection
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'dev-secret-key-change-in-production'
    
    # SQLAlchemy
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    
    # Mail settings
    MAIL_SERVER = os.environ.get('MAIL_SERVER')
    MAIL_PORT = int(os.environ.get('MAIL_PORT') or 587)
    MAIL_USE_TLS = os.environ.get('MAIL_USE_TLS', 'true').lower() in ['true', '1', 'yes']
    MAIL_USERNAME = os.environ.get('MAIL_USERNAME')
    MAIL_PASSWORD = os.environ.get('MAIL_PASSWORD')
    
    @staticmethod
    def init_app(app):
        """เรียกเมื่อ init app ด้วย config นี้"""
        pass


class DevelopmentConfig(Config):
    """Configuration สำหรับ development"""
    DEBUG = True
    SQLALCHEMY_DATABASE_URI = os.environ.get('DEV_DATABASE_URL') or \
        'sqlite:///dev.db'
    
    # แสดง SQL queries ใน console (debug)
    SQLALCHEMY_ECHO = True


class TestingConfig(Config):
    """Configuration สำหรับ testing"""
    TESTING = True
    # ใช้ in-memory database สำหรับ test
    SQLALCHEMY_DATABASE_URI = 'sqlite:///:memory:'
    
    # ปิด CSRF protection ใน test
    WTF_CSRF_ENABLED = False
    
    # ปิด error catching เพื่อให้ test จับ error ได้
    PROPAGATE_EXCEPTIONS = True


class ProductionConfig(Config):
    """Configuration สำหรับ production"""
    DEBUG = False
    TESTING = False
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or \
        'postgresql://user:pass@localhost/myapp'
    
    # ตรวจสอบว่า SECRET_KEY ตั้งค่าแล้ว
    @classmethod
    def init_app(cls, app):
        Config.init_app(app)
        
        # ตรวจสอบ required environment variables
        import logging
        from logging.handlers import RotatingFileHandler
        
        if not os.environ.get('SECRET_KEY'):
            raise ValueError("SECRET_KEY must be set in production!")
        
        # ตั้งค่า production logging
        file_handler = RotatingFileHandler(
            'logs/myapp.log',
            maxBytes=10240,
            backupCount=10
        )
        file_handler.setFormatter(logging.Formatter(
            '%(asctime)s %(levelname)s: %(message)s [in %(pathname)s:%(lineno)d]'
        ))
        file_handler.setLevel(logging.INFO)
        app.logger.addHandler(file_handler)
        app.logger.setLevel(logging.INFO)
        app.logger.info('MyApp startup')


# Dictionary สำหรับ lookup config ตาม environment name
config = {
    'development': DevelopmentConfig,
    'testing': TestingConfig,
    'production': ProductionConfig,
    'default': DevelopmentConfig,  # ค่า default
}
```

### 6.3 .env file

```bash
# .env - ไม่ควร commit ใน git!

FLASK_ENV=development
SECRET_KEY=your-very-secret-key-here
DEV_DATABASE_URL=sqlite:///dev.db
DATABASE_URL=postgresql://user:pass@localhost/myapp
MAIL_SERVER=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-app-password
```

```bash
# .gitignore - เพิ่ม .env
.env
*.pyc
__pycache__/
venv/
.flask_session/
instance/
```

---

## 7. Application Factory Pattern

Application Factory คือ pattern ที่สร้าง Flask app ใน function แทนที่จะสร้างเป็น global variable

### 7.1 ทำไมต้องใช้ Application Factory?

```
ปัญหาของ global app:
1. ทดสอบยาก - ไม่สามารถสร้าง app หลายตัวด้วย config ต่างกัน
2. Circular imports - เมื่อ project ใหญ่ขึ้น
3. ไม่ flexible - เปลี่ยน config ไม่ได้ตอน runtime

ข้อดีของ Application Factory:
1. ทดสอบง่าย - สร้าง app ใหม่สำหรับแต่ละ test
2. ป้องกัน circular imports
3. Deploy หลาย instances ด้วย config ต่างกันได้
```

### 7.2 โครงสร้างโปรเจกต์แบบ Factory

```
myapp/
├── app/
│   ├── __init__.py      # Application Factory อยู่ที่นี่
│   ├── models.py        # Database models
│   ├── routes.py        # Routes/Views
│   ├── templates/
│   │   ├── base.html
│   │   └── index.html
│   └── static/
│       ├── css/
│       └── js/
├── config.py            # Configuration classes
├── run.py               # Entry point
├── requirements.txt
└── .env
```

### 7.3 สร้าง Application Factory

```python
# app/__init__.py - Application Factory

from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager
from config import config

# สร้าง extensions เป็น global แต่ยังไม่ผูกกับ app
db = SQLAlchemy()
migrate = Migrate()
login_manager = LoginManager()

# ตั้งค่า Login Manager
login_manager.login_view = 'auth.login'  # redirect ไปที่นี่เมื่อไม่ได้ login
login_manager.login_message = 'กรุณาเข้าสู่ระบบก่อน'
login_manager.login_message_category = 'info'


def create_app(config_name='default'):
    """
    Application Factory function
    
    Args:
        config_name: ชื่อ config ('development', 'testing', 'production')
    
    Returns:
        Flask application instance
    """
    # สร้าง Flask instance
    app = Flask(__name__)
    
    # โหลด configuration
    app.config.from_object(config[config_name])
    config[config_name].init_app(app)
    
    # ผูก extensions กับ app
    db.init_app(app)
    migrate.init_app(app, db)
    login_manager.init_app(app)
    
    # ลงทะเบียน Blueprints
    from app.main import main as main_blueprint
    app.register_blueprint(main_blueprint)
    
    from app.auth import auth as auth_blueprint
    app.register_blueprint(auth_blueprint, url_prefix='/auth')
    
    from app.api import api as api_blueprint
    app.register_blueprint(api_blueprint, url_prefix='/api/v1')
    
    return app
```

### 7.4 Main Blueprint

```python
# app/main/__init__.py
from flask import Blueprint

# สร้าง Blueprint ชื่อ 'main'
main = Blueprint('main', __name__)

# import routes หลังจากสร้าง blueprint (ป้องกัน circular imports)
from app.main import routes, errors
```

```python
# app/main/routes.py
from flask import render_template, redirect, url_for, flash, current_app
from app.main import main

@main.route('/')
def index():
    """หน้าแรก"""
    return render_template('index.html', title='หน้าแรก')

@main.route('/about')
def about():
    """หน้า About"""
    return render_template('about.html', title='เกี่ยวกับเรา')
```

```python
# app/main/errors.py - Error handlers
from flask import render_template
from app.main import main

@main.app_errorhandler(404)
def page_not_found(e):
    """จัดการ 404 error"""
    return render_template('errors/404.html'), 404

@main.app_errorhandler(500)
def internal_server_error(e):
    """จัดการ 500 error"""
    return render_template('errors/500.html'), 500
```

### 7.5 Run File

```python
# run.py - Entry point ของ application

import os
from app import create_app

# อ่าน environment จาก .env
config_name = os.environ.get('FLASK_ENV', 'default')

# สร้าง app
app = create_app(config_name)

if __name__ == '__main__':
    app.run(
        host='0.0.0.0',  # รับ connection จากทุก IP
        port=int(os.environ.get('PORT', 5000)),
        debug=app.config['DEBUG']
    )
```

---

## 8. สร้าง Simple Web Application

มาสร้าง Todo app อย่างง่ายๆ เพื่อรวบรวมสิ่งที่เรียนมา

### 8.1 โครงสร้าง Todo App

```
todo_app/
├── app.py
├── templates/
│   ├── base.html
│   └── index.html
├── static/
│   └── style.css
└── requirements.txt
```

### 8.2 app.py

```python
# app.py - Simple Todo App (ใช้ in-memory storage)

from flask import Flask, render_template, request, redirect, url_for, flash

app = Flask(__name__)
# SECRET_KEY จำเป็นสำหรับ flash messages
app.config['SECRET_KEY'] = 'your-secret-key'

# ใช้ list เก็บ todos (ใน memory - รีสตาร์ทแล้วหาย)
todos = []
next_id = 1  # counter สำหรับ ID


@app.route('/')
def index():
    """หน้าแสดงรายการ todos"""
    return render_template('index.html', todos=todos)


@app.route('/add', methods=['POST'])
def add_todo():
    """เพิ่ม todo ใหม่"""
    global next_id
    
    # รับค่าจาก form
    title = request.form.get('title', '').strip()
    
    # validation
    if not title:
        flash('กรุณากรอกชื่อ Todo', 'error')
        return redirect(url_for('index'))
    
    if len(title) > 200:
        flash('ชื่อ Todo ต้องไม่เกิน 200 ตัวอักษร', 'error')
        return redirect(url_for('index'))
    
    # เพิ่ม todo
    todos.append({
        'id': next_id,
        'title': title,
        'done': False
    })
    next_id += 1
    
    flash(f'เพิ่ม "{title}" สำเร็จ', 'success')
    return redirect(url_for('index'))


@app.route('/toggle/<int:todo_id>')
def toggle_todo(todo_id):
    """สลับสถานะ done/not done"""
    # หา todo ที่ต้องการ
    todo = next((t for t in todos if t['id'] == todo_id), None)
    
    if todo is None:
        flash('ไม่พบ Todo นี้', 'error')
        return redirect(url_for('index'))
    
    # สลับสถานะ
    todo['done'] = not todo['done']
    status = 'เสร็จแล้ว' if todo['done'] else 'ยังไม่เสร็จ'
    flash(f'อัปเดต "{todo["title"]}" เป็น {status}', 'info')
    
    return redirect(url_for('index'))


@app.route('/delete/<int:todo_id>')
def delete_todo(todo_id):
    """ลบ todo"""
    global todos
    
    # หา todo
    todo = next((t for t in todos if t['id'] == todo_id), None)
    
    if todo is None:
        flash('ไม่พบ Todo นี้', 'error')
        return redirect(url_for('index'))
    
    # ลบออกจาก list
    todos = [t for t in todos if t['id'] != todo_id]
    flash(f'ลบ "{todo["title"]}" สำเร็จ', 'success')
    
    return redirect(url_for('index'))


@app.route('/clear-done')
def clear_done():
    """ลบ todos ที่เสร็จแล้วทั้งหมด"""
    global todos
    count = sum(1 for t in todos if t['done'])
    todos = [t for t in todos if not t['done']]
    flash(f'ลบ {count} รายการที่เสร็จแล้ว', 'info')
    return redirect(url_for('index'))


if __name__ == '__main__':
    app.run(debug=True)
```

### 8.3 Templates

```html
<!-- templates/base.html - Base template -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Todo App{% endblock %}</title>
    <!-- Bootstrap 5 CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" 
          rel="stylesheet">
    <!-- Custom CSS -->
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
<body>
    <!-- Navbar -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
        <div class="container">
            <a class="navbar-brand" href="{{ url_for('index') }}">
                📝 Todo App
            </a>
        </div>
    </nav>
    
    <div class="container mt-4">
        <!-- Flash messages -->
        {% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                {% for category, message in messages %}
                    <div class="alert alert-{{ 'danger' if category == 'error' else category }} 
                                alert-dismissible fade show" role="alert">
                        {{ message }}
                        <button type="button" class="btn-close" 
                                data-bs-dismiss="alert"></button>
                    </div>
                {% endfor %}
            {% endif %}
        {% endwith %}
        
        <!-- Page content -->
        {% block content %}{% endblock %}
    </div>
    
    <!-- Bootstrap 5 JS -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js">
    </script>
</body>
</html>
```

```html
<!-- templates/index.html - Todo List page -->
{% extends "base.html" %}

{% block title %}รายการ Todo{% endblock %}

{% block content %}
<div class="row justify-content-center">
    <div class="col-md-8">
        <div class="card shadow-sm">
            <div class="card-header bg-primary text-white">
                <h4 class="mb-0">📝 รายการสิ่งที่ต้องทำ</h4>
            </div>
            <div class="card-body">
                <!-- Form เพิ่ม Todo -->
                <form action="{{ url_for('add_todo') }}" method="POST" class="mb-4">
                    <div class="input-group">
                        <input type="text" 
                               name="title" 
                               class="form-control form-control-lg" 
                               placeholder="เพิ่มสิ่งที่ต้องทำ..."
                               maxlength="200"
                               required
                               autofocus>
                        <button type="submit" class="btn btn-primary btn-lg">
                            + เพิ่ม
                        </button>
                    </div>
                </form>
                
                <!-- สรุป -->
                <div class="d-flex justify-content-between align-items-center mb-3">
                    <span class="text-muted">
                        ทั้งหมด: {{ todos|length }} รายการ
                        | เสร็จแล้ว: {{ todos|selectattr('done')|list|length }} รายการ
                    </span>
                    {% if todos|selectattr('done')|list|length > 0 %}
                        <a href="{{ url_for('clear_done') }}" 
                           class="btn btn-sm btn-outline-danger"
                           onclick="return confirm('ต้องการลบรายการที่เสร็จแล้วทั้งหมด?')">
                            🗑️ ลบที่เสร็จแล้ว
                        </a>
                    {% endif %}
                </div>
                
                <!-- รายการ Todos -->
                {% if todos %}
                    <ul class="list-group">
                        {% for todo in todos %}
                            <li class="list-group-item d-flex justify-content-between 
                                       align-items-center
                                       {{ 'list-group-item-success' if todo.done else '' }}">
                                <div class="d-flex align-items-center gap-2">
                                    <!-- Checkbox toggle -->
                                    <a href="{{ url_for('toggle_todo', todo_id=todo.id) }}"
                                       class="btn btn-sm 
                                              {{ 'btn-success' if todo.done else 'btn-outline-secondary' }}">
                                        {{ '✓' if todo.done else '○' }}
                                    </a>
                                    <!-- Todo title -->
                                    <span class="{{ 'text-decoration-line-through text-muted' 
                                                    if todo.done else '' }}">
                                        {{ todo.title }}
                                    </span>
                                </div>
                                <!-- Delete button -->
                                <a href="{{ url_for('delete_todo', todo_id=todo.id) }}"
                                   class="btn btn-sm btn-outline-danger"
                                   onclick="return confirm('ต้องการลบรายการนี้?')">
                                    🗑️
                                </a>
                            </li>
                        {% endfor %}
                    </ul>
                {% else %}
                    <!-- Empty state -->
                    <div class="text-center py-5 text-muted">
                        <div style="font-size: 3rem;">📋</div>
                        <p>ยังไม่มีรายการ<br>เพิ่มสิ่งที่ต้องทำด้านบน!</p>
                    </div>
                {% endif %}
            </div>
        </div>
    </div>
</div>
{% endblock %}
```

### 8.4 Static CSS

```css
/* static/style.css */

/* ปรับแต่ง base styles */
body {
    background-color: #f8f9fa;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

/* Navbar brand */
.navbar-brand {
    font-size: 1.4rem;
    font-weight: bold;
}

/* Card สวยขึ้น */
.card {
    border-radius: 12px;
    border: none;
}

.card-header {
    border-radius: 12px 12px 0 0 !important;
    padding: 1rem 1.5rem;
}

/* List group items */
.list-group-item {
    transition: background-color 0.2s ease;
    padding: 0.75rem 1rem;
}

.list-group-item:hover {
    background-color: #e9ecef;
}

/* Input focus effect */
.form-control:focus {
    border-color: #0d6efd;
    box-shadow: 0 0 0 0.25rem rgba(13, 110, 253, 0.1);
}
```

---

## 9. Flask Debug Mode และ Development Tools

```python
# ตั้งค่า Debug Mode

# วิธีที่ 1: ใน code
app.run(debug=True)

# วิธีที่ 2: environment variable
# export FLASK_DEBUG=1
# flask run

# วิธีที่ 3: FLASK_ENV
# export FLASK_ENV=development
# flask run
```

### 9.1 Flask Shell

```bash
# เปิด interactive shell พร้อม app context
flask shell

# ใน shell:
>>> from app import db
>>> db.create_all()
>>> from app.models import User
>>> User.query.all()
```

### 9.2 Flask CLI

```python
# app/__init__.py - เพิ่ม custom CLI commands

import click
from flask.cli import with_appcontext

def create_app(config_name='default'):
    app = Flask(__name__)
    # ... setup ...
    
    # ลงทะเบียน custom commands
    register_commands(app)
    
    return app


def register_commands(app):
    """ลงทะเบียน custom CLI commands"""
    
    @app.cli.command('create-db')
    @with_appcontext
    def create_db():
        """สร้างฐานข้อมูล"""
        db.create_all()
        click.echo('สร้างฐานข้อมูลสำเร็จ!')
    
    @app.cli.command('seed-db')
    @click.argument('count', default=10, type=int)
    @with_appcontext
    def seed_db(count):
        """เพิ่มข้อมูลตัวอย่าง"""
        from app.models import User
        for i in range(count):
            user = User(username=f'user{i}', email=f'user{i}@example.com')
            db.session.add(user)
        db.session.commit()
        click.echo(f'เพิ่มข้อมูลตัวอย่าง {count} รายการสำเร็จ!')
```

```bash
# รัน custom commands
flask create-db
flask seed-db 20
```

---

## 10. Application Context และ Request Context

```python
# Flask มี 2 contexts:
# 1. Application Context - ข้อมูลเกี่ยวกับ app
# 2. Request Context - ข้อมูลเกี่ยวกับ HTTP request ปัจจุบัน

from flask import Flask, g, current_app, request

app = Flask(__name__)

@app.before_request
def before_request():
    """รันก่อนทุก request"""
    # g คือ global object สำหรับเก็บข้อมูลใน request
    g.user = None
    g.start_time = __import__('time').time()
    
    # log request
    current_app.logger.info(f'{request.method} {request.path}')


@app.after_request
def after_request(response):
    """รันหลังทุก request"""
    import time
    duration = time.time() - g.start_time
    current_app.logger.info(f'Request took {duration:.3f}s')
    return response


@app.teardown_request
def teardown_request(exception):
    """รันหลัง request เสมอ (แม้มี error)"""
    if exception:
        current_app.logger.error(f'Request error: {exception}')


# การใช้ Application Context นอก request
def do_background_work():
    """ทำงานนอก request context"""
    with app.app_context():
        # ตอนนี้ current_app, g ใช้ได้แล้ว
        current_app.logger.info('Background work')
        # ทำ database operations ได้
```

---

## 11. Logging

```python
# app/__init__.py - ตั้งค่า logging

import logging
from logging.handlers import RotatingFileHandler
import os

def configure_logging(app):
    """ตั้งค่า logging สำหรับ production"""
    if not app.debug and not app.testing:
        # สร้าง logs directory
        if not os.path.exists('logs'):
            os.mkdir('logs')
        
        # File handler
        file_handler = RotatingFileHandler(
            'logs/myapp.log',
            maxBytes=10240,  # 10 KB per file
            backupCount=10   # เก็บ 10 ไฟล์
        )
        
        # Format
        formatter = logging.Formatter(
            '[%(asctime)s] %(levelname)s in %(module)s: %(message)s'
        )
        file_handler.setFormatter(formatter)
        file_handler.setLevel(logging.INFO)
        
        app.logger.addHandler(file_handler)
        app.logger.setLevel(logging.INFO)
        app.logger.info('Application startup')


# การใช้งาน logging ใน views
@app.route('/test-log')
def test_log():
    app.logger.debug('Debug message')
    app.logger.info('Info message')
    app.logger.warning('Warning message')
    app.logger.error('Error message')
    return 'Logged!'
```

---

## 12. ตัวอย่าง Complete Simple App

```python
# complete_app.py - App ครบถ้วนสำหรับ Hello World
# รวม routing, templates, config, logging

import os
import logging
from datetime import datetime
from flask import Flask, render_template, request, jsonify, redirect, url_for

def create_app():
    app = Flask(__name__)
    
    # Configuration
    app.config.update(
        SECRET_KEY=os.environ.get('SECRET_KEY', 'dev-key'),
        DEBUG=os.environ.get('FLASK_DEBUG', 'false').lower() == 'true',
    )
    
    # Logging
    if not app.debug:
        logging.basicConfig(level=logging.INFO)
    
    # Routes
    @app.route('/')
    def index():
        """หน้าแรก"""
        return f"""
        <!DOCTYPE html>
        <html>
        <body>
            <h1>🌟 Flask App</h1>
            <p>เวลาปัจจุบัน: {datetime.now().strftime('%H:%M:%S')}</p>
            <ul>
                <li><a href="/hello">Hello World</a></li>
                <li><a href="/hello/Flask">Hello Flask</a></li>
                <li><a href="/api/info">API Info</a></li>
            </ul>
        </body>
        </html>
        """
    
    @app.route('/hello')
    @app.route('/hello/<name>')
    def hello(name='World'):
        """Hello page พร้อม URL parameter"""
        return f'<h1>Hello, {name}!</h1>'
    
    @app.route('/api/info')
    def api_info():
        """API endpoint ส่งข้อมูลเป็น JSON"""
        return jsonify({
            'app': 'Flask Demo',
            'version': '1.0.0',
            'time': datetime.now().isoformat(),
            'debug': app.debug
        })
    
    @app.errorhandler(404)
    def not_found(e):
        return f'<h1>404 - ไม่พบหน้านี้</h1><a href="/">กลับหน้าแรก</a>', 404
    
    return app


app = create_app()

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port, debug=True)
```

---

## 13. การจัดการ Static Files และ Templates Folder

```python
# app.py - กำหนด folder ด้วยตนเอง

from flask import Flask, send_from_directory
import os

app = Flask(
    __name__,
    static_folder='public',       # ค้นหา static files ใน 'public/'
    template_folder='views',       # ค้นหา templates ใน 'views/'
    static_url_path='/static'      # URL path สำหรับ static files
)

# Custom static file serving
@app.route('/files/<path:filename>')
def serve_file(filename):
    """Serve ไฟล์จาก custom directory"""
    return send_from_directory('uploads', filename)
```

---

## Exercises

### Exercise 1: Hello World ขั้นสูง
สร้าง Flask app ที่มี routes ต่อไปนี้:
- `/` - แสดงหน้าแรก
- `/greet/<name>` - แสดง "สวัสดี, {name}!"
- `/time` - แสดงเวลาปัจจุบัน
- `/random` - แสดงตัวเลขสุ่ม 1-100

### Exercise 2: Config Management
สร้าง config classes สำหรับ:
- Development (SQLite, Debug=True)
- Testing (In-memory SQLite)
- Production (PostgreSQL)

และสร้าง factory function ที่ใช้ config เหล่านี้

### Exercise 3: Application Factory
แปลง Todo app ให้ใช้ Application Factory pattern พร้อม:
- แยก config ออกเป็นไฟล์ `config.py`
- สร้าง `__init__.py` พร้อม `create_app()`
- สร้าง Blueprint สำหรับ todos

---

## สรุป

สิ่งที่เรียนรู้ใน Part นี้:
- **Flask** เป็น micro framework ที่ flexible
- **ต่างจาก Django** ตรงที่ต้องเลือก components เอง
- **Application Factory** ช่วยให้ test ง่ายและ flexible
- **Config classes** ช่วยจัดการ settings ตาม environment
- **Flask extensions** เพิ่มความสามารถได้ตามต้องการ

---

## ลิงก์ Part ถัดไป

➡️ [Part 077 - Flask Routing and Views](./part-077.md)

---

*หากมีคำถาม สามารถถามใน Discussion ของหลักสูตรได้เลย!*
