# Part 082: Flask REST API
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- สร้าง REST API ด้วย flask_restful
- ใช้ Resource classes จัดการ HTTP methods
- สร้าง Namespaces ด้วย Flask-RESTX
- สร้าง Swagger documentation อัตโนมัติ
- Serialize/Deserialize ข้อมูลด้วย Marshmallow

---

## 1. Flask-RESTful

Flask-RESTful เป็น extension ที่ช่วยสร้าง REST API ด้วย Resource-based approach

### ติดตั้ง
```bash
pip install flask-restful flask-restx marshmallow
```

### Setup พื้นฐาน
```python
# app.py

from flask import Flask
from flask_restful import Api

app = Flask(__name__)
api = Api(app)
```

### Resource Class พื้นฐาน
```python
# resources.py

from flask_restful import Resource, reqparse
from flask import jsonify

# ข้อมูลตัวอย่าง
USERS = {
    1: {'id': 1, 'name': 'สมชาย', 'email': 'somchai@example.com'},
    2: {'id': 2, 'name': 'สมหญิง', 'email': 'somying@example.com'},
}


class UserResource(Resource):
    """Resource สำหรับ User เดียว"""
    
    def get(self, user_id):
        """GET /users/<user_id> — ดึงข้อมูล user"""
        user = USERS.get(user_id)
        if not user:
            return {'message': f'ไม่พบ user ID {user_id}'}, 404
        return user, 200
    
    def put(self, user_id):
        """PUT /users/<user_id> — แก้ไข user"""
        user = USERS.get(user_id)
        if not user:
            return {'message': 'ไม่พบ user'}, 404
        
        # parse request body
        parser = reqparse.RequestParser()
        parser.add_argument('name', type=str)
        parser.add_argument('email', type=str)
        args = parser.parse_args()
        
        if args['name']:
            user['name'] = args['name']
        if args['email']:
            user['email'] = args['email']
        
        USERS[user_id] = user
        return user, 200
    
    def delete(self, user_id):
        """DELETE /users/<user_id> — ลบ user"""
        if user_id not in USERS:
            return {'message': 'ไม่พบ user'}, 404
        
        del USERS[user_id]
        return '', 204


class UserListResource(Resource):
    """Resource สำหรับรายการ Users"""
    
    def get(self):
        """GET /users — ดึงรายการ users ทั้งหมด"""
        return list(USERS.values()), 200
    
    def post(self):
        """POST /users — สร้าง user ใหม่"""
        parser = reqparse.RequestParser()
        parser.add_argument('name', type=str, required=True, 
                           help='ต้องระบุ name')
        parser.add_argument('email', type=str, required=True,
                           help='ต้องระบุ email')
        args = parser.parse_args()
        
        new_id = max(USERS.keys()) + 1 if USERS else 1
        new_user = {
            'id': new_id,
            'name': args['name'],
            'email': args['email']
        }
        USERS[new_id] = new_user
        
        return new_user, 201


# ลงทะเบียน Resources กับ URL
api.add_resource(UserListResource, '/api/users')
api.add_resource(UserResource, '/api/users/<int:user_id>')
```

---

## 2. Request Parsing ด้วย reqparse

```python
# request_parsing.py

from flask_restful import Resource, reqparse
from flask import request


class ArticleResource(Resource):
    
    # สร้าง parser ที่ class level (reuse ได้)
    parser = reqparse.RequestParser()
    parser.add_argument(
        'title',
        type=str,
        required=True,
        help='ต้องระบุหัวเรื่อง',
        location='json'       # รับจาก JSON body
    )
    parser.add_argument(
        'content',
        type=str,
        required=True,
        help='ต้องระบุเนื้อหา',
        location='json'
    )
    parser.add_argument(
        'published',
        type=bool,
        default=False,
        location='json'
    )
    parser.add_argument(
        'tags',
        type=str,
        action='append',      # รับ array
        location='json'
    )
    
    def post(self):
        args = self.parser.parse_args()
        
        article = {
            'title': args['title'],
            'content': args['content'],
            'published': args['published'],
            'tags': args['tags'] or []
        }
        
        return article, 201


class SearchResource(Resource):
    """รับ parameters จาก query string"""
    
    def get(self):
        parser = reqparse.RequestParser()
        parser.add_argument(
            'q',
            type=str,
            required=True,
            help='ต้องระบุคำค้นหา',
            location='args'     # รับจาก query string
        )
        parser.add_argument(
            'page',
            type=int,
            default=1,
            location='args'
        )
        parser.add_argument(
            'per_page',
            type=int,
            default=10,
            location='args'
        )
        
        args = parser.parse_args()
        
        # ค้นหา (ตัวอย่าง)
        results = {
            'query': args['q'],
            'page': args['page'],
            'per_page': args['per_page'],
            'results': []
        }
        
        return results, 200
```

---

## 3. Flask-RESTX พร้อม Swagger

Flask-RESTX เป็น fork ของ Flask-RESTful ที่เพิ่ม Swagger documentation

```python
# app_restx.py

from flask import Flask
from flask_restx import Api, Resource, fields, Namespace

app = Flask(__name__)

# สร้าง Api instance พร้อม Swagger metadata
api = Api(
    app,
    version='1.0',
    title='Blog API',
    description='REST API สำหรับระบบ Blog',
    doc='/api/docs',        # URL สำหรับ Swagger UI
    prefix='/api/v1'        # URL prefix
)
```

### Namespace

```python
# namespaces.py

from flask_restx import Api, Resource, fields, Namespace

# สร้าง Namespace แทน Blueprint
users_ns = Namespace('users', description='จัดการผู้ใช้')
posts_ns = Namespace('posts', description='จัดการบทความ')
auth_ns = Namespace('auth', description='Authentication')


# Define Models สำหรับ Swagger docs (และ serialization)
user_model = users_ns.model('User', {
    'id': fields.Integer(readonly=True, description='User ID'),
    'username': fields.String(required=True, description='ชื่อผู้ใช้'),
    'email': fields.String(required=True, description='อีเมล'),
    'is_active': fields.Boolean(description='สถานะ active'),
    'created_at': fields.DateTime(description='วันที่สร้าง')
})

user_input_model = users_ns.model('UserInput', {
    'username': fields.String(required=True, description='ชื่อผู้ใช้'),
    'email': fields.String(required=True, description='อีเมล'),
    'password': fields.String(required=True, description='รหัสผ่าน')
})

post_model = posts_ns.model('Post', {
    'id': fields.Integer(readonly=True),
    'title': fields.String(required=True, description='หัวเรื่อง'),
    'content': fields.String(required=True, description='เนื้อหา'),
    'author': fields.String(description='ผู้เขียน'),
    'published': fields.Boolean(default=False),
    'created_at': fields.DateTime(readonly=True)
})

# ข้อมูลตัวอย่าง
USERS_DB = {}
POSTS_DB = {}
next_user_id = 1
next_post_id = 1


@users_ns.route('/')
class UserList(Resource):
    """Endpoint สำหรับรายการ Users"""
    
    @users_ns.doc('list_users')              # ชื่อใน Swagger
    @users_ns.marshal_list_with(user_model)  # serialize ด้วย user_model
    def get(self):
        """ดึงรายการ users ทั้งหมด"""
        return list(USERS_DB.values())
    
    @users_ns.doc('create_user')
    @users_ns.expect(user_input_model)        # รับ request body ตาม model นี้
    @users_ns.marshal_with(user_model, code=201)
    def post(self):
        """สร้าง user ใหม่"""
        global next_user_id
        
        data = users_ns.payload
        
        # ตรวจสอบซ้ำ
        for user in USERS_DB.values():
            if user['username'] == data['username']:
                users_ns.abort(409, 'username นี้มีอยู่แล้ว')
        
        user = {
            'id': next_user_id,
            'username': data['username'],
            'email': data['email'],
            'is_active': True
        }
        USERS_DB[next_user_id] = user
        next_user_id += 1
        
        return user, 201


@users_ns.route('/<int:user_id>')
@users_ns.response(404, 'User ไม่พบ')
@users_ns.param('user_id', 'User ID')
class UserDetail(Resource):
    """Endpoint สำหรับ User เดียว"""
    
    @users_ns.doc('get_user')
    @users_ns.marshal_with(user_model)
    def get(self, user_id):
        """ดึงข้อมูล user ตาม ID"""
        user = USERS_DB.get(user_id)
        if not user:
            users_ns.abort(404, f'ไม่พบ user ID {user_id}')
        return user
    
    @users_ns.doc('update_user')
    @users_ns.expect(user_model)
    @users_ns.marshal_with(user_model)
    def put(self, user_id):
        """แก้ไขข้อมูล user"""
        user = USERS_DB.get(user_id)
        if not user:
            users_ns.abort(404, 'ไม่พบ user')
        
        data = users_ns.payload
        user.update(data)
        USERS_DB[user_id] = user
        return user
    
    @users_ns.doc('delete_user')
    @users_ns.response(204, 'ลบสำเร็จ')
    def delete(self, user_id):
        """ลบ user"""
        if user_id not in USERS_DB:
            users_ns.abort(404, 'ไม่พบ user')
        del USERS_DB[user_id]
        return '', 204
```

### ลงทะเบียน Namespaces
```python
# app_restx.py (ต่อ)

from namespaces import users_ns, posts_ns, auth_ns

# เพิ่ม namespaces เข้า api
api.add_namespace(users_ns, path='/users')
api.add_namespace(posts_ns, path='/posts')
api.add_namespace(auth_ns, path='/auth')

if __name__ == '__main__':
    app.run(debug=True)
```

---

## 4. Authentication ใน Flask-RESTX

```python
# auth_namespace.py

from flask_restx import Namespace, Resource, fields
from flask_jwt_extended import create_access_token, jwt_required, get_jwt_identity

auth_ns = Namespace('auth', description='Authentication')

# Models
login_model = auth_ns.model('Login', {
    'username': fields.String(required=True),
    'password': fields.String(required=True)
})

token_model = auth_ns.model('Token', {
    'access_token': fields.String(description='JWT Access Token'),
    'token_type': fields.String(default='bearer')
})


@auth_ns.route('/login')
class Login(Resource):
    
    @auth_ns.expect(login_model)
    @auth_ns.marshal_with(token_model)
    @auth_ns.response(401, 'ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง')
    def post(self):
        """Login และรับ JWT token"""
        data = auth_ns.payload
        
        # ตรวจสอบ credential (ตัวอย่าง)
        if data['username'] == 'admin' and data['password'] == 'admin123':
            token = create_access_token(identity=1)
            return {'access_token': token, 'token_type': 'bearer'}
        
        auth_ns.abort(401, 'ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง')


# Decorator สำหรับ protect endpoints ด้วย JWT
from functools import wraps
from flask_jwt_extended import verify_jwt_in_request


def jwt_required_ns(fn):
    """Decorator สำหรับ Flask-RESTX ที่ต้องการ JWT"""
    @wraps(fn)
    def wrapper(*args, **kwargs):
        verify_jwt_in_request()
        return fn(*args, **kwargs)
    return wrapper


@auth_ns.route('/me')
class CurrentUser(Resource):
    
    @auth_ns.doc(security='Bearer')  # แสดงใน Swagger ว่าต้องการ token
    @jwt_required_ns
    def get(self):
        """ดึงข้อมูล user ปัจจุบัน"""
        user_id = get_jwt_identity()
        return {'id': user_id, 'username': 'admin'}
```

---

## 5. Marshmallow สำหรับ Serialization

```python
# schemas.py

from marshmallow import Schema, fields, validate, validates, ValidationError, post_load


class UserSchema(Schema):
    """Schema สำหรับ serialize/deserialize User"""
    
    # Fields
    id = fields.Int(dump_only=True)              # read-only (serialize เท่านั้น)
    username = fields.Str(
        required=True,
        validate=validate.Length(min=3, max=80)
    )
    email = fields.Email(required=True)          # validate email format
    password = fields.Str(
        required=True,
        load_only=True,                          # write-only (deserialize เท่านั้น)
        validate=validate.Length(min=8)
    )
    is_active = fields.Bool(dump_default=True)
    created_at = fields.DateTime(dump_only=True)
    
    # Custom validator
    @validates('username')
    def validate_username(self, value):
        import re
        if not re.match(r'^[a-zA-Z0-9_]+$', value):
            raise ValidationError('ชื่อผู้ใช้ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _')
    
    # post_load: ทำงานหลัง deserialize
    @post_load
    def make_user(self, data, **kwargs):
        """แปลง data เป็น User object"""
        # from models import User
        # user = User(**data)
        # return user
        return data


class PostSchema(Schema):
    id = fields.Int(dump_only=True)
    title = fields.Str(required=True, validate=validate.Length(min=1, max=200))
    content = fields.Str(required=True)
    published = fields.Bool(dump_default=False)
    author = fields.Nested('UserSchema', only=['id', 'username'], dump_only=True)
    created_at = fields.DateTime(dump_only=True)
    tags = fields.List(fields.Str())


# สร้าง instances
user_schema = UserSchema()
users_schema = UserSchema(many=True)         # สำหรับ list
post_schema = PostSchema()
posts_schema = PostSchema(many=True)
```

### ใช้ Marshmallow ใน routes
```python
# routes.py

from flask import Flask, jsonify, request
from marshmallow import ValidationError
from schemas import user_schema, users_schema

app = Flask(__name__)


@app.route('/api/users', methods=['GET'])
def get_users():
    users = [
        {'id': 1, 'username': 'john', 'email': 'john@example.com'},
        {'id': 2, 'username': 'jane', 'email': 'jane@example.com'},
    ]
    # Serialize (Python → JSON)
    result = users_schema.dump(users)
    return jsonify(result)


@app.route('/api/users', methods=['POST'])
def create_user():
    json_data = request.get_json()
    
    if not json_data:
        return jsonify({'error': 'ต้องส่งข้อมูลเป็น JSON'}), 400
    
    try:
        # Deserialize และ validate (JSON → Python)
        data = user_schema.load(json_data)
    except ValidationError as err:
        # err.messages คือ dict ของ errors
        return jsonify({'errors': err.messages}), 422
    
    # สร้าง user (ตัวอย่าง)
    new_user = {'id': 3, **data}
    
    # Serialize result
    return jsonify(user_schema.dump(new_user)), 201
```

---

## 6. Error Handling

```python
# error_handlers.py

from flask import Flask, jsonify
from flask_restful import Api

app = Flask(__name__)
api = Api(app)


# Custom error handler สำหรับ 404
@app.errorhandler(404)
def not_found(error):
    return jsonify({
        'error': 'Not Found',
        'message': 'ไม่พบสิ่งที่คุณค้นหา',
        'status': 404
    }), 404


@app.errorhandler(405)
def method_not_allowed(error):
    return jsonify({
        'error': 'Method Not Allowed',
        'message': 'HTTP method นี้ไม่รองรับสำหรับ endpoint นี้',
        'status': 405
    }), 405


@app.errorhandler(422)
def unprocessable_entity(error):
    return jsonify({
        'error': 'Unprocessable Entity',
        'message': 'ข้อมูลไม่ถูกต้อง',
        'status': 422
    }), 422


@app.errorhandler(500)
def internal_server_error(error):
    return jsonify({
        'error': 'Internal Server Error',
        'message': 'เกิดข้อผิดพลาดภายในเซิร์ฟเวอร์',
        'status': 500
    }), 500


# Custom Exception
class APIError(Exception):
    """Base class สำหรับ API errors"""
    
    def __init__(self, message, status_code=400, payload=None):
        super().__init__()
        self.message = message
        self.status_code = status_code
        self.payload = payload
    
    def to_dict(self):
        rv = dict(self.payload or ())
        rv['message'] = self.message
        rv['status'] = self.status_code
        return rv


class NotFoundError(APIError):
    def __init__(self, message='ไม่พบข้อมูล'):
        super().__init__(message, status_code=404)


class ValidationError(APIError):
    def __init__(self, message='ข้อมูลไม่ถูกต้อง', errors=None):
        payload = {'errors': errors} if errors else None
        super().__init__(message, status_code=422, payload=payload)


@app.errorhandler(APIError)
def handle_api_error(error):
    response = jsonify(error.to_dict())
    response.status_code = error.status_code
    return response


# ใช้ custom exception
# raise NotFoundError('ไม่พบสินค้า')
# raise ValidationError('ข้อมูลไม่ถูกต้อง', errors={'name': ['ต้องระบุชื่อ']})
```

---

## 7. Pagination

```python
# pagination.py

from flask import request, url_for


class Paginator:
    """Helper class สำหรับ pagination"""
    
    def __init__(self, query, page=1, per_page=10):
        self.query = query
        self.page = page
        self.per_page = per_page
        self._pagination = None
    
    @property
    def pagination(self):
        if not self._pagination:
            self._pagination = self.query.paginate(
                page=self.page,
                per_page=self.per_page,
                error_out=False
            )
        return self._pagination
    
    def to_dict(self, schema, endpoint=None):
        """แปลง pagination เป็น dict"""
        items = schema.dump(self.pagination.items)
        
        result = {
            'items': items,
            'meta': {
                'page': self.pagination.page,
                'per_page': self.pagination.per_page,
                'total': self.pagination.total,
                'pages': self.pagination.pages,
                'has_next': self.pagination.has_next,
                'has_prev': self.pagination.has_prev
            }
        }
        
        if endpoint:
            result['links'] = {
                'self': url_for(endpoint, page=self.pagination.page, per_page=self.per_page, _external=True),
                'next': url_for(endpoint, page=self.pagination.next_num, per_page=self.per_page, _external=True) if self.pagination.has_next else None,
                'prev': url_for(endpoint, page=self.pagination.prev_num, per_page=self.per_page, _external=True) if self.pagination.has_prev else None,
            }
        
        return result


# ใช้ใน route
@app.route('/api/posts')
def get_posts():
    page = request.args.get('page', 1, type=int)
    per_page = min(request.args.get('per_page', 10, type=int), 100)
    
    paginator = Paginator(Post.query.filter_by(published=True), page, per_page)
    return jsonify(paginator.to_dict(posts_schema, endpoint='get_posts'))
```

---

## 8. ตัวอย่าง Blog API สมบูรณ์

```python
# complete_blog_api.py

from flask import Flask, jsonify, request
from flask_sqlalchemy import SQLAlchemy
from flask_restx import Api, Resource, fields, Namespace
from datetime import datetime

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///blog_api.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)

api = Api(
    app,
    version='1.0',
    title='Blog API',
    description='REST API สำหรับระบบ Blog',
    doc='/docs'
)

# Namespaces
posts_ns = Namespace('posts', description='จัดการบทความ', path='/api/posts')
api.add_namespace(posts_ns)


# Models
class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    content = db.Column(db.Text, nullable=False)
    author = db.Column(db.String(80), default='Anonymous')
    published = db.Column(db.Boolean, default=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)


# API Models (สำหรับ Swagger)
post_model = posts_ns.model('Post', {
    'id': fields.Integer(readonly=True),
    'title': fields.String(required=True),
    'content': fields.String(required=True),
    'author': fields.String(default='Anonymous'),
    'published': fields.Boolean(default=False),
    'created_at': fields.DateTime(readonly=True)
})

post_input = posts_ns.model('PostInput', {
    'title': fields.String(required=True, min_length=1),
    'content': fields.String(required=True),
    'author': fields.String(),
    'published': fields.Boolean(default=False)
})


@posts_ns.route('/')
class PostList(Resource):
    
    @posts_ns.marshal_list_with(post_model)
    def get(self):
        """ดึงรายการบทความ"""
        page = request.args.get('page', 1, type=int)
        per_page = request.args.get('per_page', 10, type=int)
        
        pagination = Post.query.order_by(
            Post.created_at.desc()
        ).paginate(page=page, per_page=per_page, error_out=False)
        
        return pagination.items
    
    @posts_ns.expect(post_input)
    @posts_ns.marshal_with(post_model, code=201)
    def post(self):
        """สร้างบทความใหม่"""
        data = posts_ns.payload
        
        post = Post(
            title=data['title'],
            content=data['content'],
            author=data.get('author', 'Anonymous'),
            published=data.get('published', False)
        )
        db.session.add(post)
        db.session.commit()
        
        return post, 201


@posts_ns.route('/<int:post_id>')
@posts_ns.response(404, 'บทความไม่พบ')
class PostDetail(Resource):
    
    @posts_ns.marshal_with(post_model)
    def get(self, post_id):
        """ดึงบทความตาม ID"""
        post = db.session.get(Post, post_id)
        if not post:
            posts_ns.abort(404, f'ไม่พบบทความ ID {post_id}')
        return post
    
    @posts_ns.expect(post_input)
    @posts_ns.marshal_with(post_model)
    def put(self, post_id):
        """แก้ไขบทความ"""
        post = db.session.get(Post, post_id)
        if not post:
            posts_ns.abort(404, 'ไม่พบบทความ')
        
        data = posts_ns.payload
        post.title = data.get('title', post.title)
        post.content = data.get('content', post.content)
        post.published = data.get('published', post.published)
        
        db.session.commit()
        return post
    
    @posts_ns.response(204, 'ลบสำเร็จ')
    def delete(self, post_id):
        """ลบบทความ"""
        post = db.session.get(Post, post_id)
        if not post:
            posts_ns.abort(404, 'ไม่พบบทความ')
        
        db.session.delete(post)
        db.session.commit()
        return '', 204


if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(debug=True)

# เปิด http://localhost:5000/docs เพื่อดู Swagger UI
```

---

## 9. สรุป Part 082

✅ **Flask-RESTful** สร้าง REST API ด้วย Resource classes  
✅ **Resource class** กำหนด HTTP methods: get(), post(), put(), delete()  
✅ **reqparse** parse และ validate request parameters  
✅ **Flask-RESTX** เพิ่ม Swagger documentation อัตโนมัติ  
✅ **Namespace** จัดกลุ่ม endpoints คล้าย Blueprint  
✅ **fields.model** กำหนด schema สำหรับ Swagger และ serialization  
✅ **Marshmallow** ช่วย serialize/deserialize ข้อมูลอย่างละเอียด  

---

## ➡️ ถัดไป: Part 083 - Flask Testing

*Part 082/100+ | Python Course - Beginner to World-Class*
