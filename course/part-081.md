# Part 081: Flask Authentication
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- ใช้ Flask-Login จัดการ session-based authentication
- ป้องกัน routes ด้วย login_required
- ใช้ current_user เพื่อดึงข้อมูลผู้ใช้ปัจจุบัน
- ใช้ remember_me สำหรับ persistent login
- สร้าง JWT authentication ด้วย Flask-JWT-Extended

---

## 1. Flask-Login

Flask-Login จัดการ user session ทำให้:
- จดจำว่าใครล็อกอินอยู่
- ป้องกัน routes ที่ต้องการ login
- ดึงข้อมูล user ปัจจุบัน

### ติดตั้ง
```bash
pip install flask-login flask-bcrypt
```

### Setup Flask-Login
```python
# app.py

from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager
from flask_bcrypt import Bcrypt

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///auth.db'

db = SQLAlchemy(app)
bcrypt = Bcrypt(app)

# สร้าง LoginManager
login_manager = LoginManager(app)

# หน้าที่จะ redirect ไปเมื่อไม่ได้ login
login_manager.login_view = 'auth.login'

# ข้อความเมื่อต้อง login
login_manager.login_message = 'กรุณาเข้าสู่ระบบก่อน'
login_manager.login_message_category = 'warning'
```

---

## 2. User Model สำหรับ Flask-Login

```python
# models.py

from flask_sqlalchemy import SQLAlchemy
from flask_login import UserMixin
from flask_bcrypt import Bcrypt
from datetime import datetime

db = SQLAlchemy()
bcrypt = Bcrypt()


class User(UserMixin, db.Model):
    """
    UserMixin ให้ method เหล่านี้อัตโนมัติ:
    - is_authenticated: True ถ้า login แล้ว
    - is_active: True ถ้า account active
    - is_anonymous: False สำหรับ user จริง
    - get_id(): คืน user id เป็น string
    """
    
    __tablename__ = 'users'
    
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(256), nullable=False)
    
    is_active = db.Column(db.Boolean, default=True)
    is_admin = db.Column(db.Boolean, default=False)
    
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    last_login = db.Column(db.DateTime, nullable=True)
    
    def set_password(self, password):
        """Hash password ด้วย bcrypt"""
        self.password_hash = bcrypt.generate_password_hash(password).decode('utf-8')
    
    def check_password(self, password):
        """ตรวจสอบ password กับ hash"""
        return bcrypt.check_password_hash(self.password_hash, password)
    
    def update_last_login(self):
        """บันทึกเวลา login ล่าสุด"""
        self.last_login = datetime.utcnow()
        db.session.commit()
    
    def __repr__(self):
        return f'<User {self.username}>'


# Flask-Login ต้องการ user_loader function
# บอกวิธีโหลด User จาก session (จาก user id ที่เก็บใน session)
from app import login_manager

@login_manager.user_loader
def load_user(user_id):
    """โหลด User จาก user_id ที่เก็บใน session"""
    return db.session.get(User, int(user_id))
```

---

## 3. Login และ Logout

```python
# auth/routes.py

from flask import Blueprint, render_template, redirect, url_for, flash, request
from flask_login import login_user, logout_user, login_required, current_user
from models import User, db
from forms import LoginForm, RegisterForm

auth_bp = Blueprint('auth', __name__, url_prefix='/auth')


@auth_bp.route('/login', methods=['GET', 'POST'])
def login():
    """หน้า Login"""
    
    # ถ้า login แล้วให้ redirect ไปหน้าหลัก
    if current_user.is_authenticated:
        return redirect(url_for('main.index'))
    
    form = LoginForm()
    
    if form.validate_on_submit():
        # ค้นหา user จาก username หรือ email
        user = User.query.filter_by(username=form.username.data).first()
        
        if user is None:
            # ลองค้นด้วย email
            user = User.query.filter_by(email=form.username.data).first()
        
        # ตรวจสอบ password
        if user is None or not user.check_password(form.password.data):
            flash('ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง', 'danger')
            return redirect(url_for('auth.login'))
        
        if not user.is_active:
            flash('บัญชีนี้ถูกระงับการใช้งาน', 'danger')
            return redirect(url_for('auth.login'))
        
        # Login สำเร็จ!
        # remember=True จะสร้าง long-lived cookie
        login_user(user, remember=form.remember_me.data)
        
        # บันทึกเวลา login
        user.update_last_login()
        
        flash(f'ยินดีต้อนรับกลับมา {user.username}!', 'success')
        
        # Redirect ไปหน้าที่ user พยายามเข้าก่อน (ถ้ามี)
        next_page = request.args.get('next')
        if next_page and next_page.startswith('/'):  # ป้องกัน open redirect
            return redirect(next_page)
        
        return redirect(url_for('main.index'))
    
    return render_template('auth/login.html', form=form)


@auth_bp.route('/logout')
@login_required  # ต้อง login ก่อนจึงจะ logout ได้
def logout():
    """ออกจากระบบ"""
    username = current_user.username
    logout_user()  # ลบข้อมูลจาก session
    flash(f'ออกจากระบบแล้ว ลาก่อน {username}!', 'info')
    return redirect(url_for('main.index'))


@auth_bp.route('/register', methods=['GET', 'POST'])
def register():
    """สมัครสมาชิก"""
    
    if current_user.is_authenticated:
        return redirect(url_for('main.index'))
    
    form = RegisterForm()
    
    if form.validate_on_submit():
        # ตรวจสอบ username ซ้ำ
        if User.query.filter_by(username=form.username.data).first():
            flash('ชื่อผู้ใช้นี้มีคนใช้แล้ว', 'danger')
            return render_template('auth/register.html', form=form)
        
        # ตรวจสอบ email ซ้ำ
        if User.query.filter_by(email=form.email.data).first():
            flash('อีเมลนี้มีคนใช้แล้ว', 'danger')
            return render_template('auth/register.html', form=form)
        
        # สร้าง user ใหม่
        user = User(
            username=form.username.data,
            email=form.email.data.lower()
        )
        user.set_password(form.password.data)
        
        db.session.add(user)
        db.session.commit()
        
        flash('สมัครสมาชิกสำเร็จ! กรุณาเข้าสู่ระบบ', 'success')
        return redirect(url_for('auth.login'))
    
    return render_template('auth/register.html', form=form)
```

---

## 4. login_required และ current_user

```python
# main/routes.py

from flask import Blueprint, render_template, redirect, url_for, flash
from flask_login import login_required, current_user

main_bp = Blueprint('main', __name__)


@main_bp.route('/')
def index():
    """หน้าแรก — ทุกคนเข้าได้"""
    return render_template('main/index.html')


@main_bp.route('/dashboard')
@login_required  # ต้อง login ก่อน
def dashboard():
    """Dashboard — เฉพาะ user ที่ login"""
    # current_user คือ User object ของผู้ที่ login อยู่
    user_posts = current_user.posts.count()
    return render_template('main/dashboard.html',
                          user=current_user,
                          post_count=user_posts)


@main_bp.route('/profile')
@login_required
def profile():
    """หน้าโปรไฟล์ของตัวเอง"""
    return render_template('main/profile.html', user=current_user)


@main_bp.route('/settings')
@login_required
def settings():
    """ตั้งค่าบัญชี"""
    return render_template('main/settings.html', user=current_user)


# Decorator สำหรับ admin only
from functools import wraps

def admin_required(f):
    """Decorator: ต้องเป็น admin"""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if not current_user.is_authenticated:
            return redirect(url_for('auth.login'))
        if not current_user.is_admin:
            flash('ต้องการสิทธิ์ผู้ดูแลระบบ', 'danger')
            return redirect(url_for('main.index'))
        return f(*args, **kwargs)
    return decorated_function


@main_bp.route('/admin')
@admin_required
def admin_dashboard():
    """Admin dashboard — เฉพาะ admin"""
    users = User.query.all()
    return render_template('admin/dashboard.html', users=users)
```

### ใช้ current_user ใน template
```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <title>{% block title %}My App{% endblock %}</title>
</head>
<body>
    <nav>
        <a href="{{ url_for('main.index') }}">หน้าแรก</a>
        
        {% if current_user.is_authenticated %}
            <!-- แสดงเมื่อ login แล้ว -->
            <span>ยินดีต้อนรับ, {{ current_user.username }}!</span>
            <a href="{{ url_for('main.dashboard') }}">Dashboard</a>
            <a href="{{ url_for('main.profile') }}">โปรไฟล์</a>
            
            {% if current_user.is_admin %}
                <!-- แสดงเฉพาะ admin -->
                <a href="{{ url_for('main.admin_dashboard') }}">Admin</a>
            {% endif %}
            
            <a href="{{ url_for('auth.logout') }}">ออกจากระบบ</a>
        {% else %}
            <!-- แสดงเมื่อยังไม่ได้ login -->
            <a href="{{ url_for('auth.login') }}">เข้าสู่ระบบ</a>
            <a href="{{ url_for('auth.register') }}">สมัครสมาชิก</a>
        {% endif %}
    </nav>
    
    {% block content %}{% endblock %}
</body>
</html>
```

---

## 5. Remember Me

```python
# forms.py

class LoginForm(FlaskForm):
    username = StringField('ชื่อผู้ใช้', validators=[DataRequired()])
    password = PasswordField('รหัสผ่าน', validators=[DataRequired()])
    remember_me = BooleanField('จดจำฉัน (30 วัน)')
    submit = SubmitField('เข้าสู่ระบบ')
```

```python
# routes.py — การใช้ remember_me

@auth_bp.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    
    if form.validate_on_submit():
        user = User.query.filter_by(username=form.username.data).first()
        
        if user and user.check_password(form.password.data):
            # remember=True สร้าง cookie ที่อยู่ได้นาน
            # Flask-Login จะสร้าง "remember me" cookie
            login_user(user, remember=form.remember_me.data)
            
            # กำหนดเวลา expire ของ remember_me cookie
            # ใน app config:
            # app.config['REMEMBER_COOKIE_DURATION'] = timedelta(days=30)
            # app.config['REMEMBER_COOKIE_SECURE'] = True  # HTTPS only
            # app.config['REMEMBER_COOKIE_HTTPONLY'] = True
            
            return redirect(url_for('main.index'))
    
    return render_template('auth/login.html', form=form)
```

```python
# app.py — Configuration สำหรับ Remember Me

from datetime import timedelta

app.config['REMEMBER_COOKIE_DURATION'] = timedelta(days=30)
app.config['REMEMBER_COOKIE_SECURE'] = False   # True ใน production (HTTPS only)
app.config['REMEMBER_COOKIE_HTTPONLY'] = True  # ป้องกัน JavaScript เข้าถึง
app.config['REMEMBER_COOKIE_SAMESITE'] = 'Lax' # ป้องกัน CSRF
```

---

## 6. Password Reset

```python
# models.py — เพิ่ม reset password functionality

import jwt
from datetime import datetime, timedelta
from flask import current_app


class User(UserMixin, db.Model):
    # ... fields ...
    
    def generate_reset_token(self, expires_in=3600):
        """สร้าง token สำหรับ reset password"""
        payload = {
            'reset_password': self.id,
            'exp': datetime.utcnow() + timedelta(seconds=expires_in)
        }
        token = jwt.encode(
            payload,
            current_app.config['SECRET_KEY'],
            algorithm='HS256'
        )
        return token
    
    @staticmethod
    def verify_reset_token(token):
        """ตรวจสอบ token และคืน User"""
        try:
            payload = jwt.decode(
                token,
                current_app.config['SECRET_KEY'],
                algorithms=['HS256']
            )
            user_id = payload['reset_password']
        except Exception:
            return None
        return db.session.get(User, user_id)
```

```python
# routes.py — Password Reset Flow

@auth_bp.route('/forgot-password', methods=['GET', 'POST'])
def forgot_password():
    """ขอ reset password"""
    if current_user.is_authenticated:
        return redirect(url_for('main.index'))
    
    form = ForgotPasswordForm()
    
    if form.validate_on_submit():
        user = User.query.filter_by(email=form.email.data).first()
        
        if user:
            token = user.generate_reset_token()
            reset_url = url_for('auth.reset_password', token=token, _external=True)
            
            # ส่งอีเมล (ตัวอย่าง)
            print(f'Reset URL: {reset_url}')
            # send_email(user.email, 'Reset Password', reset_url)
        
        # บอก user เสมอ ไม่ว่าจะหาเจอ email หรือไม่ (ป้องกัน enumeration)
        flash('ถ้ามีบัญชีอีเมลนี้ เราจะส่ง link reset password ไปให้', 'info')
        return redirect(url_for('auth.login'))
    
    return render_template('auth/forgot_password.html', form=form)


@auth_bp.route('/reset-password/<token>', methods=['GET', 'POST'])
def reset_password(token):
    """Reset password ด้วย token"""
    if current_user.is_authenticated:
        return redirect(url_for('main.index'))
    
    user = User.verify_reset_token(token)
    if not user:
        flash('Link reset password ไม่ถูกต้องหรือหมดอายุ', 'danger')
        return redirect(url_for('auth.forgot_password'))
    
    form = ResetPasswordForm()
    
    if form.validate_on_submit():
        user.set_password(form.password.data)
        db.session.commit()
        flash('รหัสผ่านถูกเปลี่ยนแล้ว กรุณาเข้าสู่ระบบ', 'success')
        return redirect(url_for('auth.login'))
    
    return render_template('auth/reset_password.html', form=form, token=token)
```

---

## 7. Flask-JWT-Extended

JWT (JSON Web Token) เหมาะสำหรับ REST API ที่ไม่ใช้ session

### ติดตั้ง
```bash
pip install flask-jwt-extended
```

### Setup
```python
# app.py

from flask import Flask
from flask_jwt_extended import JWTManager

app = Flask(__name__)
app.config['JWT_SECRET_KEY'] = 'jwt-secret-key-change-in-production'
app.config['JWT_ACCESS_TOKEN_EXPIRES'] = timedelta(hours=1)   # Access token อยู่ 1 ชั่วโมง
app.config['JWT_REFRESH_TOKEN_EXPIRES'] = timedelta(days=30)  # Refresh token อยู่ 30 วัน

jwt = JWTManager(app)


# Callback เมื่อ token หมดอายุ
@jwt.expired_token_loader
def expired_token_callback(jwt_header, jwt_payload):
    return {'message': 'Token หมดอายุ', 'error': 'token_expired'}, 401


# Callback เมื่อ token ไม่ถูกต้อง
@jwt.invalid_token_loader
def invalid_token_callback(error):
    return {'message': 'Token ไม่ถูกต้อง', 'error': 'invalid_token'}, 401


# Callback เมื่อไม่มี token
@jwt.unauthorized_loader
def missing_token_callback(error):
    return {'message': 'ต้องการ Authorization token', 'error': 'authorization_required'}, 401
```

### Login และ Token Generation
```python
# auth/routes.py — JWT version

from flask import Blueprint, jsonify, request
from flask_jwt_extended import (
    create_access_token,
    create_refresh_token,
    jwt_required,
    get_jwt_identity,
    get_jwt
)
from models import User

api_auth_bp = Blueprint('api_auth', __name__, url_prefix='/api/auth')


@api_auth_bp.route('/login', methods=['POST'])
def login():
    """Login และรับ JWT token"""
    data = request.get_json()
    
    if not data:
        return jsonify({'message': 'ต้องส่งข้อมูลเป็น JSON'}), 400
    
    username = data.get('username')
    password = data.get('password')
    
    if not username or not password:
        return jsonify({'message': 'ต้องระบุ username และ password'}), 400
    
    # ค้นหา user
    user = User.query.filter_by(username=username).first()
    
    if not user or not user.check_password(password):
        return jsonify({'message': 'ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง'}), 401
    
    if not user.is_active:
        return jsonify({'message': 'บัญชีนี้ถูกระงับ'}), 401
    
    # สร้าง tokens
    # identity คือข้อมูลที่เก็บใน token (มักใช้ user id)
    access_token = create_access_token(
        identity=user.id,
        additional_claims={
            'username': user.username,
            'is_admin': user.is_admin
        }
    )
    
    refresh_token = create_refresh_token(identity=user.id)
    
    return jsonify({
        'access_token': access_token,
        'refresh_token': refresh_token,
        'user': {
            'id': user.id,
            'username': user.username,
            'email': user.email
        }
    }), 200


@api_auth_bp.route('/refresh', methods=['POST'])
@jwt_required(refresh=True)  # ต้องใช้ refresh token
def refresh():
    """ต่ออายุ access token"""
    current_user_id = get_jwt_identity()
    
    # สร้าง access token ใหม่
    new_access_token = create_access_token(identity=current_user_id)
    
    return jsonify({'access_token': new_access_token}), 200


@api_auth_bp.route('/logout', methods=['DELETE'])
@jwt_required()
def logout():
    """Logout — เพิ่ม token ลง blocklist"""
    jti = get_jwt()['jti']  # JWT ID
    
    # เพิ่ม token ลง blocklist (ต้องเก็บใน Redis หรือ database)
    # jwt_redis_blocklist.set(jti, "", ex=ACCESS_EXPIRES)
    
    return jsonify({'message': 'Logout สำเร็จ'}), 200
```

### Protected Endpoints
```python
# routes.py — Endpoints ที่ต้องการ JWT

from flask_jwt_extended import jwt_required, get_jwt_identity, get_jwt


@app.route('/api/me', methods=['GET'])
@jwt_required()  # ต้องมี valid JWT token
def get_current_user():
    """ดึงข้อมูล user ปัจจุบัน"""
    current_user_id = get_jwt_identity()
    user = db.session.get(User, current_user_id)
    
    if not user:
        return jsonify({'message': 'User ไม่พบ'}), 404
    
    return jsonify({
        'id': user.id,
        'username': user.username,
        'email': user.email
    })


@app.route('/api/admin/users', methods=['GET'])
@jwt_required()
def admin_get_users():
    """Admin: ดึงรายการ users ทั้งหมด"""
    claims = get_jwt()  # ดึง claims ทั้งหมดจาก token
    
    if not claims.get('is_admin'):
        return jsonify({'message': 'ต้องการสิทธิ์ admin'}), 403
    
    users = User.query.all()
    return jsonify([{
        'id': u.id,
        'username': u.username,
        'email': u.email,
        'is_active': u.is_active
    } for u in users])


@app.route('/api/posts', methods=['POST'])
@jwt_required()
def create_post():
    """สร้างบทความ — ต้อง login"""
    current_user_id = get_jwt_identity()
    data = request.get_json()
    
    post = Post(
        title=data['title'],
        content=data['content'],
        user_id=current_user_id
    )
    db.session.add(post)
    db.session.commit()
    
    return jsonify({'id': post.id, 'title': post.title}), 201
```

---

## 8. JWT Token Blocklist

```python
# extensions.py — Token Blocklist ด้วย Set (ควรใช้ Redis ใน production)

from flask_jwt_extended import JWTManager

jwt = JWTManager()

# ใน memory blocklist (ไม่เหมาะกับ production)
jwt_blocklist = set()


def init_jwt(app):
    jwt.init_app(app)
    
    @jwt.token_in_blocklist_loader
    def check_if_token_revoked(jwt_header, jwt_payload):
        """ตรวจสอบว่า token ถูก revoke แล้วหรือไม่"""
        jti = jwt_payload['jti']
        return jti in jwt_blocklist
    
    return jwt


# routes.py — Logout ด้วย blocklist
@api_auth_bp.route('/logout', methods=['DELETE'])
@jwt_required()
def logout():
    jti = get_jwt()['jti']
    jwt_blocklist.add(jti)
    return jsonify({'message': 'Logout สำเร็จ'}), 200
```

---

## 9. ตัวอย่างสมบูรณ์: Auth System

```python
# complete_auth.py

from flask import Flask, jsonify, request
from flask_sqlalchemy import SQLAlchemy
from flask_bcrypt import Bcrypt
from flask_jwt_extended import (
    JWTManager, create_access_token, create_refresh_token,
    jwt_required, get_jwt_identity
)
from datetime import timedelta

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///auth_demo.db'
app.config['JWT_SECRET_KEY'] = 'super-secret-jwt-key'
app.config['JWT_ACCESS_TOKEN_EXPIRES'] = timedelta(hours=1)

db = SQLAlchemy(app)
bcrypt = Bcrypt(app)
jwt = JWTManager(app)


class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(256))
    
    def set_password(self, password):
        self.password_hash = bcrypt.generate_password_hash(password).decode('utf-8')
    
    def check_password(self, password):
        return bcrypt.check_password_hash(self.password_hash, password)


@app.route('/api/register', methods=['POST'])
def register():
    data = request.get_json()
    
    # Validate
    if not data or not all(k in data for k in ['username', 'email', 'password']):
        return jsonify({'error': 'ต้องระบุ username, email, password'}), 400
    
    if User.query.filter_by(username=data['username']).first():
        return jsonify({'error': 'username นี้มีอยู่แล้ว'}), 409
    
    if User.query.filter_by(email=data['email']).first():
        return jsonify({'error': 'email นี้มีอยู่แล้ว'}), 409
    
    # สร้าง user
    user = User(username=data['username'], email=data['email'])
    user.set_password(data['password'])
    db.session.add(user)
    db.session.commit()
    
    return jsonify({'message': 'สมัครสมาชิกสำเร็จ', 'id': user.id}), 201


@app.route('/api/login', methods=['POST'])
def login():
    data = request.get_json()
    
    user = User.query.filter_by(username=data.get('username')).first()
    
    if not user or not user.check_password(data.get('password', '')):
        return jsonify({'error': 'ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง'}), 401
    
    access_token = create_access_token(identity=user.id)
    refresh_token = create_refresh_token(identity=user.id)
    
    return jsonify({
        'access_token': access_token,
        'refresh_token': refresh_token
    })


@app.route('/api/me', methods=['GET'])
@jwt_required()
def me():
    user_id = get_jwt_identity()
    user = db.session.get(User, user_id)
    return jsonify({'id': user.id, 'username': user.username, 'email': user.email})


if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(debug=True)
```

### ทดสอบด้วย curl
```bash
# สมัครสมาชิก
curl -X POST http://localhost:5000/api/register \
  -H "Content-Type: application/json" \
  -d '{"username":"john","email":"john@example.com","password":"password123"}'

# Login
curl -X POST http://localhost:5000/api/login \
  -H "Content-Type: application/json" \
  -d '{"username":"john","password":"password123"}'

# ดึงข้อมูล user (ต้องมี token)
curl http://localhost:5000/api/me \
  -H "Authorization: Bearer <your-access-token>"
```

---

## 10. สรุป Part 081

✅ **Flask-Login** จัดการ session-based authentication  
✅ **UserMixin** ให้ method จำเป็นสำหรับ Flask-Login  
✅ **login_user() / logout_user()** สำหรับ login/logout  
✅ **@login_required** ป้องกัน route ที่ต้องการ authentication  
✅ **current_user** ดึงข้อมูลผู้ใช้ที่ login อยู่  
✅ **remember_me** สร้าง persistent cookie  
✅ **Flask-JWT-Extended** สำหรับ JWT token authentication ใน API  
✅ **create_access_token() / create_refresh_token()** สร้าง JWT tokens  
✅ **@jwt_required()** ป้องกัน API endpoints  

---

## ➡️ ถัดไป: Part 082 - Flask REST API

*Part 081/100+ | Python Course - Beginner to World-Class*
