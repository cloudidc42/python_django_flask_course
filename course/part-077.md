# Part 077 - Flask Routing and Views

## เป้าหมายการเรียนรู้

- ใช้ `route()` decorator และ HTTP methods
- จัดการ URL variables และ converters
- ใช้ `url_for()` สร้าง URL
- ทำงานกับ Request object
- ส่ง Response, redirect, และ abort
- สร้าง JSON responses ด้วย `jsonify`

---

## 1. Route Decorator พื้นฐาน

```python
from flask import Flask

app = Flask(__name__)

# Route พื้นฐาน - รับเฉพาะ GET
@app.route('/')
def index():
    return 'หน้าแรก'

# หลาย URL ชี้ไปที่ function เดียวกัน
@app.route('/hello')
@app.route('/hi')
def hello():
    return 'สวัสดี!'

# URL ที่ลงท้ายด้วย / (trailing slash)
@app.route('/projects/')  # มี trailing slash - Flask redirect /projects -> /projects/
def projects():
    return 'รายการโปรเจกต์'

@app.route('/about')  # ไม่มี trailing slash - /about/ จะ 404
def about():
    return 'เกี่ยวกับเรา'
```

---

## 2. HTTP Methods

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

# รับเฉพาะ GET (default)
@app.route('/items')
def get_items():
    return jsonify({'items': []})


# รับเฉพาะ POST
@app.route('/items', methods=['POST'])
def create_item():
    data = request.json
    return jsonify({'created': data}), 201


# รับทั้ง GET และ POST ใน function เดียว
@app.route('/contact', methods=['GET', 'POST'])
def contact():
    if request.method == 'POST':
        # จัดการ form submission
        name = request.form.get('name')
        email = request.form.get('email')
        return f'ขอบคุณ {name}!'
    # GET: แสดง form
    return '''
    <form method="POST">
        <input name="name" placeholder="ชื่อ">
        <input name="email" type="email" placeholder="อีเมล">
        <button type="submit">ส่ง</button>
    </form>
    '''


# รับทุก HTTP methods
@app.route('/api/resource', methods=['GET', 'POST', 'PUT', 'DELETE', 'PATCH'])
def resource():
    method = request.method
    
    if method == 'GET':
        return jsonify({'action': 'read'})
    elif method == 'POST':
        return jsonify({'action': 'create'}), 201
    elif method == 'PUT':
        return jsonify({'action': 'replace'})
    elif method == 'PATCH':
        return jsonify({'action': 'update'})
    elif method == 'DELETE':
        return jsonify({'action': 'delete'}), 204
```

### 2.1 RESTful API Routing Pattern

```python
# patterns/restful_routes.py
# Pattern สำหรับ RESTful API

# Collection routes
@app.route('/api/users', methods=['GET'])       # GET all users
@app.route('/api/users', methods=['POST'])      # POST create user

# Item routes
@app.route('/api/users/<int:user_id>', methods=['GET'])     # GET single user
@app.route('/api/users/<int:user_id>', methods=['PUT'])     # PUT replace user
@app.route('/api/users/<int:user_id>', methods=['PATCH'])   # PATCH update user
@app.route('/api/users/<int:user_id>', methods=['DELETE'])  # DELETE user
```

---

## 3. URL Variables และ Converters

```python
# URL variables - ส่วนที่เปลี่ยนแปลงได้ใน URL

# String variable (default)
@app.route('/user/<username>')
def user_profile(username):
    # username เป็น string
    return f'โปรไฟล์ของ {username}'

# Integer variable
@app.route('/post/<int:post_id>')
def get_post(post_id):
    # post_id เป็น int (URL /post/abc จะ 404)
    return f'โพสต์ #{post_id}'

# Float variable
@app.route('/price/<float:amount>')
def show_price(amount):
    # amount เป็น float
    return f'ราคา: {amount:.2f} บาท'

# Path variable (รับ / ด้วย)
@app.route('/files/<path:filepath>')
def download_file(filepath):
    # filepath สามารถมี / ได้ เช่น /files/uploads/2024/image.png
    return f'ไฟล์: {filepath}'

# UUID variable
@app.route('/order/<uuid:order_id>')
def get_order(order_id):
    # order_id เป็น UUID object
    return f'คำสั่งซื้อ: {order_id}'
```

### 3.1 ตาราง URL Converters

| Converter | ตัวอย่าง URL | Type | หมายเหตุ |
|-----------|-------------|------|---------|
| `string` | `/user/john` | str | ค่า default ไม่รับ `/` |
| `int` | `/post/42` | int | ตัวเลขบวก |
| `float` | `/price/9.99` | float | ตัวเลขทศนิยม |
| `path` | `/files/a/b/c` | str | รับ `/` ได้ |
| `uuid` | `/item/550e8400-...` | UUID | UUID format |

### 3.2 Custom URL Converters

```python
# custom_converter.py - สร้าง converter เอง

from werkzeug.routing import BaseConverter

class ListConverter(BaseConverter):
    """Converter ที่แปลง a+b+c เป็น ['a', 'b', 'c']"""
    
    def to_python(self, value):
        """แปลง URL string เป็น Python value"""
        return value.split('+')
    
    def to_url(self, value):
        """แปลง Python value เป็น URL string"""
        return '+'.join(value)


class BoolConverter(BaseConverter):
    """Converter สำหรับ boolean"""
    regex = r'(?:true|false)'
    
    def to_python(self, value):
        return value.lower() == 'true'
    
    def to_url(self, value):
        return 'true' if value else 'false'


# ลงทะเบียน converters
app.url_map.converters['list'] = ListConverter
app.url_map.converters['bool'] = BoolConverter


# การใช้งาน
@app.route('/tags/<list:tag_list>')
def filter_by_tags(tag_list):
    # /tags/python+flask+web -> ['python', 'flask', 'web']
    return f'Tags: {", ".join(tag_list)}'

@app.route('/items/active/<bool:is_active>')
def items_by_status(is_active):
    # /items/active/true -> True
    return f'Active: {is_active}'
```

---

## 4. url_for() - สร้าง URL จาก Function Name

```python
from flask import Flask, url_for, redirect

app = Flask(__name__)

@app.route('/')
def index():
    return 'หน้าแรก'

@app.route('/user/<username>')
def user_profile(username):
    return f'โปรไฟล์: {username}'

@app.route('/post/<int:post_id>')
def get_post(post_id):
    return f'โพสต์: {post_id}'

@app.route('/redirect-test')
def redirect_test():
    # สร้าง URL จาก function name
    home_url = url_for('index')                     # '/'
    user_url = url_for('user_profile', username='john')  # '/user/john'
    post_url = url_for('get_post', post_id=42)      # '/post/42'
    
    # เพิ่ม query string
    search_url = url_for('index', q='flask', page=2)  # '/?q=flask&page=2'
    
    # สร้าง absolute URL (external=True)
    abs_url = url_for('index', _external=True)  # 'http://localhost:5000/'
    
    # เพิ่ม anchor
    anchor_url = url_for('index', _anchor='section1')  # '/#section1'
    
    return f'''
    home: {home_url}<br>
    user: {user_url}<br>
    post: {post_url}<br>
    search: {search_url}<br>
    absolute: {abs_url}<br>
    anchor: {anchor_url}
    '''


@app.route('/go-home')
def go_home():
    # redirect ไปที่ function อื่น
    return redirect(url_for('index'))
```

### 4.1 url_for() ใน Templates

```html
<!-- templates/navigation.html -->

<!-- สร้าง URL ด้วย url_for ใน Jinja2 -->
<nav>
    <a href="{{ url_for('index') }}">หน้าแรก</a>
    <a href="{{ url_for('user_profile', username=current_user.username) }}">โปรไฟล์</a>
    <a href="{{ url_for('get_post', post_id=1) }}">โพสต์แรก</a>
    
    <!-- Static files -->
    <link href="{{ url_for('static', filename='css/style.css') }}" rel="stylesheet">
    <img src="{{ url_for('static', filename='img/logo.png') }}" alt="Logo">
</nav>
```

---

## 5. Request Object

Request object ให้ข้อมูลทั้งหมดเกี่ยวกับ HTTP request ปัจจุบัน

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/request-info', methods=['GET', 'POST'])
def request_info():
    # ข้อมูลพื้นฐาน
    method = request.method          # 'GET', 'POST', etc.
    url = request.url                # URL เต็ม: http://localhost:5000/request-info
    base_url = request.base_url      # URL ไม่มี query string
    host = request.host              # 'localhost:5000'
    path = request.path              # '/request-info'
    full_path = request.full_path    # '/request-info?name=john'
    
    # Remote address
    ip = request.remote_addr         # IP ของ client
    
    print(f'Method: {method}')
    print(f'URL: {url}')
    print(f'IP: {ip}')
    
    return 'OK'
```

### 5.1 Query Parameters (GET parameters)

```python
@app.route('/search')
def search():
    """
    GET /search?q=flask&page=2&per_page=10&sort=date&order=desc
    """
    # รับ query parameters
    q = request.args.get('q')                        # 'flask' หรือ None
    page = request.args.get('page', 1, type=int)     # 2 (แปลงเป็น int, default=1)
    per_page = request.args.get('per_page', 10, type=int)  # 10
    sort = request.args.get('sort', 'date')          # 'date'
    order = request.args.get('order', 'desc')        # 'desc'
    
    # รับทุก values ของ key เดียวกัน (multi-select)
    # /search?tag=python&tag=flask&tag=web
    tags = request.args.getlist('tag')               # ['python', 'flask', 'web']
    
    # ดู args ทั้งหมด
    all_args = request.args.to_dict()
    
    # validation
    if not q:
        return jsonify({'error': 'กรุณากรอก search term'}), 400
    
    return jsonify({
        'query': q,
        'page': page,
        'per_page': per_page,
        'sort': sort,
        'order': order,
        'tags': tags
    })
```

### 5.2 Form Data (POST parameters)

```python
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        # รับค่าจาก HTML form
        username = request.form.get('username', '').strip()
        password = request.form.get('password', '')
        remember = request.form.get('remember') == 'on'  # checkbox
        
        # รับ multiple values (checkboxes, multi-select)
        # <input type="checkbox" name="roles" value="admin">
        # <input type="checkbox" name="roles" value="user">
        roles = request.form.getlist('roles')  # ['admin', 'user']
        
        # ดู form data ทั้งหมด
        all_form = request.form.to_dict()
        
        # Validation
        if not username or not password:
            return 'กรุณากรอก username และ password', 400
        
        # Process login...
        return f'เข้าสู่ระบบ: {username}'
    
    # GET: แสดง form
    return '''
    <form method="POST">
        <input name="username" placeholder="Username" required><br>
        <input name="password" type="password" placeholder="Password" required><br>
        <input type="checkbox" name="remember"> จำรหัสผ่าน<br>
        <button type="submit">เข้าสู่ระบบ</button>
    </form>
    '''
```

### 5.3 JSON Body

```python
@app.route('/api/users', methods=['POST'])
def create_user():
    """รับ JSON body"""
    # ตรวจสอบ Content-Type
    if not request.is_json:
        return jsonify({'error': 'Content-Type must be application/json'}), 415
    
    # รับ JSON data
    data = request.json  # dict หรือ None
    # หรือใช้
    data = request.get_json()           # เหมือนกัน
    data = request.get_json(silent=True)  # return None แทน error ถ้า parse ไม่ได้
    data = request.get_json(force=True)   # parse แม้ Content-Type ไม่ใช่ JSON
    
    if data is None:
        return jsonify({'error': 'Invalid JSON'}), 400
    
    # ดึงข้อมูล
    name = data.get('name')
    email = data.get('email')
    age = data.get('age', 0)
    
    # Nested data
    address = data.get('address', {})
    city = address.get('city')
    
    # Array
    tags = data.get('tags', [])
    
    if not name or not email:
        return jsonify({'error': 'name และ email จำเป็น'}), 422
    
    return jsonify({
        'id': 1,
        'name': name,
        'email': email,
        'created': True
    }), 201
```

### 5.4 File Upload

```python
import os
from werkzeug.utils import secure_filename

UPLOAD_FOLDER = 'uploads'
ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'gif', 'pdf'}
MAX_CONTENT_LENGTH = 16 * 1024 * 1024  # 16 MB

app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
app.config['MAX_CONTENT_LENGTH'] = MAX_CONTENT_LENGTH


def allowed_file(filename):
    """ตรวจสอบว่า file extension อนุญาตไหม"""
    return '.' in filename and \
           filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS


@app.route('/upload', methods=['GET', 'POST'])
def upload_file():
    if request.method == 'POST':
        # ตรวจสอบว่ามีไฟล์
        if 'file' not in request.files:
            return jsonify({'error': 'ไม่พบไฟล์'}), 400
        
        file = request.files['file']
        
        # ตรวจสอบว่าเลือกไฟล์แล้ว
        if file.filename == '':
            return jsonify({'error': 'ไม่ได้เลือกไฟล์'}), 400
        
        if not allowed_file(file.filename):
            return jsonify({'error': 'ประเภทไฟล์ไม่ได้รับอนุญาต'}), 400
        
        # ทำให้ชื่อไฟล์ปลอดภัย (ป้องกัน path traversal)
        filename = secure_filename(file.filename)
        
        # เพิ่ม timestamp เพื่อป้องกันชื่อซ้ำ
        import uuid
        unique_filename = f"{uuid.uuid4().hex}_{filename}"
        
        # สร้าง folder ถ้าไม่มี
        os.makedirs(app.config['UPLOAD_FOLDER'], exist_ok=True)
        
        # บันทึกไฟล์
        filepath = os.path.join(app.config['UPLOAD_FOLDER'], unique_filename)
        file.save(filepath)
        
        # รับข้อมูลไฟล์
        file_size = os.path.getsize(filepath)
        content_type = file.content_type
        
        return jsonify({
            'filename': unique_filename,
            'original_name': filename,
            'size': file_size,
            'content_type': content_type,
            'url': url_for('serve_file', filename=unique_filename, _external=True)
        })
    
    # GET: แสดง upload form
    return '''
    <form method="POST" enctype="multipart/form-data">
        <input type="file" name="file" accept="image/*,.pdf">
        <button type="submit">อัปโหลด</button>
    </form>
    '''


# Upload หลายไฟล์พร้อมกัน
@app.route('/upload-multiple', methods=['POST'])
def upload_multiple():
    files = request.files.getlist('files')  # รับหลายไฟล์
    uploaded = []
    
    for file in files:
        if file and allowed_file(file.filename):
            filename = secure_filename(file.filename)
            file.save(os.path.join(UPLOAD_FOLDER, filename))
            uploaded.append(filename)
    
    return jsonify({'uploaded': uploaded, 'count': len(uploaded)})
```

### 5.5 Headers และ Cookies

```python
@app.route('/headers-example')
def headers_example():
    # อ่าน request headers
    user_agent = request.headers.get('User-Agent')
    content_type = request.headers.get('Content-Type')
    auth_header = request.headers.get('Authorization')
    
    # Custom header
    api_key = request.headers.get('X-API-Key')
    
    # อ่าน cookies
    session_id = request.cookies.get('session_id')
    user_pref = request.cookies.get('user_preference')
    
    # ดู headers ทั้งหมด
    all_headers = dict(request.headers)
    
    return jsonify({
        'user_agent': user_agent,
        'api_key': api_key,
        'session_id': session_id
    })
```

---

## 6. Response Object

```python
from flask import Flask, Response, make_response, jsonify, redirect, abort
import json

app = Flask(__name__)


# 1. Return string (Flask สร้าง Response ให้)
@app.route('/simple')
def simple():
    return 'Hello!'  # status 200, text/html


# 2. Return tuple (body, status)
@app.route('/created')
def created():
    return 'Created!', 201


# 3. Return tuple (body, status, headers)
@app.route('/custom-header')
def custom_header():
    return 'Hello!', 200, {'X-Custom': 'value'}


# 4. Return Response object โดยตรง
@app.route('/response-object')
def response_object():
    response = Response(
        response='Custom response',
        status=200,
        headers={'Content-Type': 'text/plain; charset=utf-8'}
    )
    return response


# 5. make_response() - สร้าง response แล้วแก้ไข
@app.route('/make-response')
def make_resp():
    resp = make_response('Hello with cookie!')
    
    # ตั้งค่า headers
    resp.headers['X-Custom-Header'] = 'my-value'
    resp.headers['Cache-Control'] = 'no-cache'
    
    # ตั้งค่า cookies
    resp.set_cookie(
        'user_id',
        value='123',
        max_age=3600,          # หมดอายุใน 1 ชั่วโมง (วินาที)
        secure=True,           # ส่งเฉพาะ HTTPS
        httponly=True,         # JavaScript เข้าถึงไม่ได้
        samesite='Lax'         # CSRF protection
    )
    
    # ลบ cookie
    resp.delete_cookie('old_cookie')
    
    resp.status_code = 200
    return resp
```

### 6.1 JSON Response

```python
from flask import jsonify

@app.route('/api/data')
def api_data():
    """ส่ง JSON response"""
    data = {
        'users': [
            {'id': 1, 'name': 'Alice', 'email': 'alice@example.com'},
            {'id': 2, 'name': 'Bob', 'email': 'bob@example.com'},
        ],
        'total': 2,
        'page': 1
    }
    
    # jsonify แปลง dict/list เป็น JSON response
    # Content-Type: application/json
    return jsonify(data)


@app.route('/api/user/<int:user_id>')
def get_user(user_id):
    """ส่ง JSON หรือ error"""
    user = {'id': user_id, 'name': 'Alice'}  # mock data
    
    if not user:
        # ส่ง error response
        return jsonify({'error': 'ไม่พบผู้ใช้', 'code': 404}), 404
    
    return jsonify(user)


# Custom JSON encoder สำหรับ types พิเศษ
from datetime import datetime, date
from decimal import Decimal

@app.route('/api/special-types')
def special_types():
    data = {
        'created_at': datetime.now().isoformat(),  # แปลง datetime เป็น string
        'date': date.today().isoformat(),
        'price': float(Decimal('19.99')),  # แปลง Decimal เป็น float
        'id': str(1),  # แปลงเป็น string
    }
    return jsonify(data)
```

### 6.2 Redirect

```python
from flask import redirect, url_for

@app.route('/old-path')
def old_path():
    """301 Permanent redirect"""
    return redirect(url_for('new_path'), 301)


@app.route('/new-path')
def new_path():
    return 'หน้าใหม่'


@app.route('/external')
def external():
    """Redirect ไปที่ URL ภายนอก"""
    return redirect('https://www.google.com')


@app.route('/login-required')
def login_required_example():
    """Redirect ถ้าไม่ได้ login"""
    user_logged_in = False  # ตรวจสอบจริงๆ
    
    if not user_logged_in:
        # redirect พร้อม next parameter
        return redirect(url_for('login', next=request.url))
    
    return 'Welcome!'


@app.route('/login')
def login():
    next_url = request.args.get('next', url_for('index'))
    # หลัง login สำเร็จ:
    # return redirect(next_url)
    return f'Login page, will redirect to: {next_url}'
```

### 6.3 Abort - ยกเลิก Request

```python
from flask import abort

@app.route('/admin')
def admin():
    """ตรวจสอบสิทธิ์"""
    user_is_admin = False  # ตรวจสอบจริงๆ
    
    if not user_is_admin:
        abort(403)  # Forbidden
    
    return 'Admin Panel'


@app.route('/item/<int:item_id>')
def get_item(item_id):
    """หาข้อมูลหรือ 404"""
    item = None  # ค้นหาจาก database
    
    if item is None:
        abort(404)  # Not Found
    
    return jsonify(item)


# abort พร้อม response
@app.route('/validate')
def validate():
    data = request.json
    
    if not data:
        # abort พร้อม custom response
        abort(make_response(
            jsonify({'error': 'ต้องการ JSON body'}),
            400
        ))
    
    return jsonify({'valid': True})
```

---

## 7. Error Handlers

```python
from flask import jsonify, render_template

# Error handler สำหรับ HTML pages
@app.errorhandler(404)
def not_found(error):
    """จัดการ 404 Not Found"""
    # ตรวจสอบว่า client ต้องการ JSON หรือ HTML
    if request.accept_mimetypes.best == 'application/json':
        return jsonify({'error': 'ไม่พบทรัพยากร', 'code': 404}), 404
    return render_template('errors/404.html'), 404


@app.errorhandler(403)
def forbidden(error):
    """จัดการ 403 Forbidden"""
    return jsonify({'error': 'ไม่มีสิทธิ์เข้าถึง', 'code': 403}), 403


@app.errorhandler(500)
def internal_error(error):
    """จัดการ 500 Internal Server Error"""
    # log error
    app.logger.error(f'Server Error: {error}')
    return jsonify({'error': 'เซิร์ฟเวอร์เกิดข้อผิดพลาด', 'code': 500}), 500


# Handler สำหรับ custom exception
class APIError(Exception):
    """Custom exception สำหรับ API errors"""
    def __init__(self, message, status_code=400, payload=None):
        super().__init__()
        self.message = message
        self.status_code = status_code
        self.payload = payload


@app.errorhandler(APIError)
def handle_api_error(error):
    """จัดการ APIError"""
    response = {
        'error': error.message,
        'code': error.status_code
    }
    if error.payload:
        response['details'] = error.payload
    return jsonify(response), error.status_code


@app.route('/api/risky')
def risky_operation():
    """ตัวอย่างการ raise APIError"""
    value = request.args.get('value', type=int)
    
    if value is None:
        raise APIError('ต้องการ parameter "value"', 422)
    
    if value < 0:
        raise APIError('value ต้องเป็นบวก', 422, {'field': 'value', 'min': 0})
    
    return jsonify({'result': value * 2})
```

---

## 8. Before/After Request Hooks

```python
from flask import g, request
import time

@app.before_request
def before_each_request():
    """รันก่อนทุก request"""
    g.start_time = time.time()
    
    # ตรวจสอบ API key สำหรับ /api/* routes
    if request.path.startswith('/api/'):
        api_key = request.headers.get('X-API-Key')
        if not api_key or api_key != app.config.get('API_KEY'):
            return jsonify({'error': 'API key ไม่ถูกต้อง'}), 401


@app.after_request
def after_each_request(response):
    """รันหลังทุก request (มีหรือไม่มี error)"""
    # เพิ่ม CORS headers
    response.headers['Access-Control-Allow-Origin'] = '*'
    response.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, OPTIONS'
    response.headers['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
    
    # Log response time
    if hasattr(g, 'start_time'):
        duration = time.time() - g.start_time
        response.headers['X-Response-Time'] = f'{duration:.3f}s'
    
    return response


@app.teardown_request
def teardown(exception=None):
    """รันหลัง request เสมอ แม้มี exception"""
    if exception:
        app.logger.error(f'Request failed: {exception}')
    
    # ปิด database connection
    db_conn = g.pop('db', None)
    if db_conn is not None:
        db_conn.close()
```

---

## 9. ตัวอย่าง Complete REST API

```python
# complete_api.py - REST API สำหรับจัดการรายการสินค้า

from flask import Flask, jsonify, request, abort, make_response
from datetime import datetime

app = Flask(__name__)
app.config['SECRET_KEY'] = 'api-secret-key'

# Mock database (ใน memory)
products = {
    1: {'id': 1, 'name': 'สมุดโน้ต', 'price': 59.0, 'stock': 100, 'created_at': '2024-01-01'},
    2: {'id': 2, 'name': 'ปากกา', 'price': 25.0, 'stock': 200, 'created_at': '2024-01-01'},
    3: {'id': 3, 'name': 'ดินสอ', 'price': 15.0, 'stock': 150, 'created_at': '2024-01-01'},
}
next_id = 4


def validate_product_data(data, required=True):
    """Validate product data"""
    errors = {}
    
    if required or 'name' in data:
        name = data.get('name', '').strip()
        if not name:
            errors['name'] = 'ชื่อสินค้าจำเป็น'
        elif len(name) > 100:
            errors['name'] = 'ชื่อสินค้าต้องไม่เกิน 100 ตัวอักษร'
    
    if required or 'price' in data:
        price = data.get('price')
        if price is None:
            if required:
                errors['price'] = 'ราคาจำเป็น'
        elif not isinstance(price, (int, float)) or price < 0:
            errors['price'] = 'ราคาต้องเป็นตัวเลขที่มากกว่าหรือเท่ากับ 0'
    
    if 'stock' in data:
        stock = data.get('stock')
        if not isinstance(stock, int) or stock < 0:
            errors['stock'] = 'จำนวนสต็อกต้องเป็นจำนวนเต็มที่มากกว่าหรือเท่ากับ 0'
    
    return errors


# ─────────────────────────────────────────
# GET /api/products - ดูสินค้าทั้งหมด
# ─────────────────────────────────────────
@app.route('/api/products', methods=['GET'])
def list_products():
    """รายการสินค้าทั้งหมด พร้อม filtering และ pagination"""
    # Query parameters
    page = request.args.get('page', 1, type=int)
    per_page = request.args.get('per_page', 10, type=int)
    search = request.args.get('search', '').strip()
    min_price = request.args.get('min_price', type=float)
    max_price = request.args.get('max_price', type=float)
    sort_by = request.args.get('sort_by', 'id')
    order = request.args.get('order', 'asc')
    
    # Validation
    if per_page > 100:
        per_page = 100
    
    # Filter
    result = list(products.values())
    
    if search:
        result = [p for p in result if search.lower() in p['name'].lower()]
    
    if min_price is not None:
        result = [p for p in result if p['price'] >= min_price]
    
    if max_price is not None:
        result = [p for p in result if p['price'] <= max_price]
    
    # Sort
    valid_sort_fields = {'id', 'name', 'price', 'stock'}
    if sort_by in valid_sort_fields:
        reverse = (order == 'desc')
        result = sorted(result, key=lambda x: x[sort_by], reverse=reverse)
    
    # Pagination
    total = len(result)
    start = (page - 1) * per_page
    end = start + per_page
    paginated = result[start:end]
    
    return jsonify({
        'data': paginated,
        'pagination': {
            'page': page,
            'per_page': per_page,
            'total': total,
            'pages': (total + per_page - 1) // per_page,
        }
    })


# ─────────────────────────────────────────
# GET /api/products/<id> - ดูสินค้าเดียว
# ─────────────────────────────────────────
@app.route('/api/products/<int:product_id>', methods=['GET'])
def get_product(product_id):
    """ดูสินค้าตาม ID"""
    product = products.get(product_id)
    if not product:
        return jsonify({'error': 'ไม่พบสินค้า'}), 404
    return jsonify(product)


# ─────────────────────────────────────────
# POST /api/products - สร้างสินค้าใหม่
# ─────────────────────────────────────────
@app.route('/api/products', methods=['POST'])
def create_product():
    """สร้างสินค้าใหม่"""
    global next_id
    
    if not request.is_json:
        return jsonify({'error': 'Content-Type must be application/json'}), 415
    
    data = request.get_json()
    if not data:
        return jsonify({'error': 'Invalid JSON'}), 400
    
    # Validate
    errors = validate_product_data(data, required=True)
    if errors:
        return jsonify({'errors': errors}), 422
    
    # สร้างสินค้า
    product = {
        'id': next_id,
        'name': data['name'].strip(),
        'price': float(data['price']),
        'stock': int(data.get('stock', 0)),
        'created_at': datetime.now().isoformat(),
    }
    products[next_id] = product
    next_id += 1
    
    return jsonify(product), 201


# ─────────────────────────────────────────
# PUT /api/products/<id> - แทนที่สินค้า
# ─────────────────────────────────────────
@app.route('/api/products/<int:product_id>', methods=['PUT'])
def update_product(product_id):
    """แทนที่ข้อมูลสินค้าทั้งหมด"""
    if product_id not in products:
        return jsonify({'error': 'ไม่พบสินค้า'}), 404
    
    if not request.is_json:
        return jsonify({'error': 'Content-Type must be application/json'}), 415
    
    data = request.get_json()
    errors = validate_product_data(data, required=True)
    if errors:
        return jsonify({'errors': errors}), 422
    
    # แทนที่ข้อมูลทั้งหมด
    products[product_id].update({
        'name': data['name'].strip(),
        'price': float(data['price']),
        'stock': int(data.get('stock', 0)),
    })
    
    return jsonify(products[product_id])


# ─────────────────────────────────────────
# PATCH /api/products/<id> - อัปเดตบางส่วน
# ─────────────────────────────────────────
@app.route('/api/products/<int:product_id>', methods=['PATCH'])
def patch_product(product_id):
    """อัปเดตข้อมูลสินค้าบางส่วน"""
    if product_id not in products:
        return jsonify({'error': 'ไม่พบสินค้า'}), 404
    
    data = request.get_json()
    if not data:
        return jsonify({'error': 'ต้องการ JSON body'}), 400
    
    errors = validate_product_data(data, required=False)
    if errors:
        return jsonify({'errors': errors}), 422
    
    # อัปเดตเฉพาะ fields ที่ส่งมา
    product = products[product_id]
    if 'name' in data:
        product['name'] = data['name'].strip()
    if 'price' in data:
        product['price'] = float(data['price'])
    if 'stock' in data:
        product['stock'] = int(data['stock'])
    
    return jsonify(product)


# ─────────────────────────────────────────
# DELETE /api/products/<id> - ลบสินค้า
# ─────────────────────────────────────────
@app.route('/api/products/<int:product_id>', methods=['DELETE'])
def delete_product(product_id):
    """ลบสินค้า"""
    if product_id not in products:
        return jsonify({'error': 'ไม่พบสินค้า'}), 404
    
    del products[product_id]
    return '', 204  # 204 No Content


if __name__ == '__main__':
    app.run(debug=True)
```

---

## 10. OPTIONS และ CORS

```python
# cors_example.py - จัดการ CORS

@app.route('/api/data', methods=['GET', 'OPTIONS'])
def api_with_cors():
    """ตอบ CORS preflight request"""
    if request.method == 'OPTIONS':
        # ตอบ preflight
        response = make_response()
        response.headers['Access-Control-Allow-Origin'] = '*'
        response.headers['Access-Control-Allow-Methods'] = 'GET, POST, OPTIONS'
        response.headers['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
        response.headers['Access-Control-Max-Age'] = '86400'
        return response
    
    return jsonify({'data': 'Hello from CORS-enabled endpoint'})


# ใช้ flask-cors extension (แนะนำ)
# pip install flask-cors
from flask_cors import CORS

app = Flask(__name__)
CORS(app)  # อนุญาตทุก origin

# หรือกำหนดเฉพาะเจาะจง
CORS(app, resources={
    r'/api/*': {
        'origins': ['https://example.com', 'https://app.example.com'],
        'methods': ['GET', 'POST', 'PUT', 'DELETE'],
        'allow_headers': ['Content-Type', 'Authorization']
    }
})
```

---

## Exercises

### Exercise 1: Calculator API
สร้าง REST API สำหรับ calculator ที่มี endpoints:
- `GET /api/calc/add?a=5&b=3` → `{"result": 8}`
- `POST /api/calc/calculate` รับ JSON `{"op": "multiply", "a": 4, "b": 6}` → `{"result": 24}`
- รองรับ operations: add, subtract, multiply, divide
- Handle division by zero

### Exercise 2: URL Converter
สร้าง custom URL converter สำหรับ date format:
- `/events/2024-01-15` → รับ date object
- `/events/2024-01` → รับ month (ทุกวันในเดือน)

### Exercise 3: Complete CRUD
สร้าง REST API สำหรับ "Notes" (บันทึกย่อ) ที่มีทั้ง:
- Create, Read, Update, Delete
- Search by title
- Filter by date
- Pagination

---

## สรุป

สิ่งที่เรียนรู้ใน Part นี้:
- **route()** decorator พร้อม HTTP methods ต่างๆ
- **URL Variables** และ **Converters** สำหรับ dynamic URLs
- **url_for()** สร้าง URL จาก function name
- **Request object** เข้าถึง args, form, JSON, files, headers
- **Response** ส่งข้อมูลกลับในรูปแบบต่างๆ
- **Error handlers** จัดการ HTTP errors
- **Before/After hooks** ทำงานก่อน/หลัง request

---

## ลิงก์ Part ถัดไป

➡️ [Part 078 - Flask Templates (Jinja2)](./part-078.md)
