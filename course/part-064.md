# Part 064: Django REST Framework (DRF) Setup

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ติดตั้งและตั้งค่า Django REST Framework
- เข้าใจแนวคิด REST API
- สร้าง Serializers เพื่อแปลงข้อมูล
- สร้าง ViewSets และใช้ Routers
- ใช้ Browsable API ของ DRF

---

## 1. REST API คืออะไร?

REST (Representational State Transfer) เป็นรูปแบบการออกแบบ API ที่:
- ใช้ HTTP methods: GET, POST, PUT, PATCH, DELETE
- ใช้ URLs แทน resources
- ส่งข้อมูลในรูปแบบ JSON (หรือ XML)
- Stateless - ทุก request เป็นอิสระ

```
HTTP Method | Action      | URL
GET         | ดูรายการ   | /api/articles/
GET         | ดูชิ้นหนึ่ง | /api/articles/1/
POST        | สร้างใหม่  | /api/articles/
PUT         | แก้ไขทั้งหมด | /api/articles/1/
PATCH       | แก้ไขบางส่วน | /api/articles/1/
DELETE      | ลบ          | /api/articles/1/
```

---

## 2. การติดตั้ง DRF

```bash
pip install djangorestframework
pip install markdown       # สำหรับ Browsable API
pip install django-filter  # สำหรับ filtering
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',           # DRF
    'rest_framework.authtoken', # Token authentication
    'django_filters',           # Filtering
]

# DRF settings
REST_FRAMEWORK = {
    # Authentication classes
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
    ],
    
    # Permission classes
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    
    # Parser classes
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
        'rest_framework.parsers.FormParser',
        'rest_framework.parsers.MultiPartParser',
    ],
    
    # Renderer classes
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',  # Browsable API
    ],
    
    # Pagination
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
    
    # Throttling (rate limiting)
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
    },
    
    # Exception handler
    'EXCEPTION_HANDLER': 'rest_framework.views.exception_handler',
    
    # Date/Time format
    'DATETIME_FORMAT': '%Y-%m-%dT%H:%M:%S',
}
```

---

## 3. Models ตัวอย่าง

```python
# api/models.py
from django.db import models
from django.conf import settings

class Category(models.Model):
    name = models.CharField(max_length=100, verbose_name='ชื่อหมวดหมู่')
    slug = models.SlugField(unique=True)
    description = models.TextField(blank=True, verbose_name='คำอธิบาย')
    
    class Meta:
        verbose_name = 'หมวดหมู่'
        verbose_name_plural = 'หมวดหมู่'
        ordering = ['name']
    
    def __str__(self):
        return self.name


class Article(models.Model):
    STATUS_CHOICES = [
        ('draft', 'ฉบับร่าง'),
        ('published', 'เผยแพร่'),
        ('archived', 'เก็บถาวร'),
    ]
    
    title = models.CharField(max_length=200, verbose_name='หัวข้อ')
    slug = models.SlugField(unique=True)
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='articles',
        verbose_name='ผู้เขียน'
    )
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='articles',
        verbose_name='หมวดหมู่'
    )
    content = models.TextField(verbose_name='เนื้อหา')
    excerpt = models.TextField(max_length=500, blank=True, verbose_name='บทคัดย่อ')
    thumbnail = models.ImageField(upload_to='articles/', null=True, blank=True)
    status = models.CharField(
        max_length=20,
        choices=STATUS_CHOICES,
        default='draft',
        verbose_name='สถานะ'
    )
    views_count = models.PositiveIntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        verbose_name = 'บทความ'
        verbose_name_plural = 'บทความ'
        ordering = ['-created_at']
    
    def __str__(self):
        return self.title


class Comment(models.Model):
    article = models.ForeignKey(
        Article,
        on_delete=models.CASCADE,
        related_name='comments'
    )
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='comments'
    )
    content = models.TextField(verbose_name='ข้อความ')
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return f'Comment by {self.author} on {self.article}'
```

---

## 4. Serializers พื้นฐาน

```python
# api/serializers.py
from rest_framework import serializers
from .models import Category, Article, Comment
from django.contrib.auth import get_user_model

User = get_user_model()

class CategorySerializer(serializers.ModelSerializer):
    """Serializer สำหรับ Category"""
    
    # เพิ่ม computed field
    articles_count = serializers.SerializerMethodField()
    
    class Meta:
        model = Category
        fields = ['id', 'name', 'slug', 'description', 'articles_count']
        read_only_fields = ['id']
    
    def get_articles_count(self, obj):
        """จำนวนบทความใน category นี้"""
        return obj.articles.filter(status='published').count()


class ArticleListSerializer(serializers.ModelSerializer):
    """Serializer สำหรับ list articles (แสดงข้อมูลน้อย)"""
    author_name = serializers.CharField(source='author.get_full_name', read_only=True)
    category_name = serializers.CharField(source='category.name', read_only=True)
    
    class Meta:
        model = Article
        fields = [
            'id', 'title', 'slug', 'author_name', 'category_name',
            'excerpt', 'status', 'views_count', 'created_at'
        ]
        read_only_fields = ['id', 'views_count', 'created_at']


class ArticleDetailSerializer(serializers.ModelSerializer):
    """Serializer สำหรับ article detail (แสดงข้อมูลเยอะ)"""
    author = serializers.SerializerMethodField()
    category = CategorySerializer(read_only=True)
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(),
        source='category',
        write_only=True,
        required=False
    )
    comments_count = serializers.SerializerMethodField()
    
    class Meta:
        model = Article
        fields = [
            'id', 'title', 'slug', 'author', 'category', 'category_id',
            'content', 'excerpt', 'thumbnail', 'status', 'views_count',
            'comments_count', 'created_at', 'updated_at'
        ]
        read_only_fields = ['id', 'views_count', 'created_at', 'updated_at']
    
    def get_author(self, obj):
        return {
            'id': obj.author.id,
            'username': obj.author.username,
            'full_name': obj.author.get_full_name(),
        }
    
    def get_comments_count(self, obj):
        return obj.comments.count()
    
    def validate_title(self, value):
        """Validate title"""
        if len(value) < 5:
            raise serializers.ValidationError('หัวข้อต้องมีอย่างน้อย 5 ตัวอักษร')
        return value
    
    def create(self, validated_data):
        """Override create เพื่อกำหนด author"""
        validated_data['author'] = self.context['request'].user
        return super().create(validated_data)
```

---

## 5. ViewSets

```python
# api/views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated, IsAuthenticatedOrReadOnly
from .models import Category, Article, Comment
from .serializers import (
    CategorySerializer, ArticleListSerializer, ArticleDetailSerializer
)

class CategoryViewSet(viewsets.ModelViewSet):
    """ViewSet สำหรับ Category CRUD"""
    queryset = Category.objects.all()
    serializer_class = CategorySerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    lookup_field = 'slug'  # ใช้ slug แทน pk ใน URL


class ArticleViewSet(viewsets.ModelViewSet):
    """ViewSet สำหรับ Article CRUD"""
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def get_queryset(self):
        """กรอง queryset ตาม request"""
        queryset = Article.objects.select_related('author', 'category')
        
        # กรองตาม status
        status_param = self.request.query_params.get('status')
        if status_param:
            queryset = queryset.filter(status=status_param)
        else:
            # default แสดงเฉพาะ published
            queryset = queryset.filter(status='published')
        
        # กรองตาม category
        category = self.request.query_params.get('category')
        if category:
            queryset = queryset.filter(category__slug=category)
        
        return queryset
    
    def get_serializer_class(self):
        """ใช้ serializer ต่างกันตาม action"""
        if self.action == 'list':
            return ArticleListSerializer
        return ArticleDetailSerializer
    
    def perform_create(self, serializer):
        """กำหนด author เมื่อสร้าง article"""
        serializer.save(author=self.request.user)
    
    # Custom actions
    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def publish(self, request, pk=None):
        """เผยแพร่บทความ"""
        article = self.get_object()
        
        # ตรวจสอบว่าเป็น author หรือไม่
        if article.author != request.user and not request.user.is_staff:
            return Response(
                {'error': 'ไม่มีสิทธิ์เผยแพร่บทความนี้'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        article.status = 'published'
        article.save()
        
        serializer = self.get_serializer(article)
        return Response(serializer.data)
    
    @action(detail=False, methods=['get'], permission_classes=[IsAuthenticated])
    def my_articles(self, request):
        """ดูบทความของตัวเอง"""
        articles = Article.objects.filter(author=request.user)
        page = self.paginate_queryset(articles)
        if page is not None:
            serializer = ArticleListSerializer(page, many=True)
            return self.get_paginated_response(serializer.data)
        serializer = ArticleListSerializer(articles, many=True)
        return Response(serializer.data)
    
    @action(detail=True, methods=['get'])
    def comments(self, request, pk=None):
        """ดู comments ของ article"""
        article = self.get_object()
        comments = article.comments.all()
        # ส่งข้อมูลแบบง่าย
        data = [
            {
                'id': c.id,
                'author': c.author.username,
                'content': c.content,
                'created_at': c.created_at,
            }
            for c in comments
        ]
        return Response(data)
```

---

## 6. Routers

```python
# api/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

# สร้าง Router
router = DefaultRouter()

# Register ViewSets กับ Router
router.register(r'categories', views.CategoryViewSet, basename='category')
router.register(r'articles', views.ArticleViewSet, basename='article')

# Router จะสร้าง URLs เหล่านี้อัตโนมัติ:
# GET    /api/categories/          - list
# POST   /api/categories/          - create
# GET    /api/categories/{slug}/   - retrieve
# PUT    /api/categories/{slug}/   - update
# PATCH  /api/categories/{slug}/   - partial_update
# DELETE /api/categories/{slug}/   - destroy
# 
# GET    /api/articles/            - list
# POST   /api/articles/            - create
# GET    /api/articles/{pk}/       - retrieve
# PUT    /api/articles/{pk}/       - update
# PATCH  /api/articles/{pk}/       - partial_update
# DELETE /api/articles/{pk}/       - destroy
# POST   /api/articles/{pk}/publish/   - custom action
# GET    /api/articles/my_articles/    - custom action
# GET    /api/articles/{pk}/comments/  - custom action

urlpatterns = [
    path('', include(router.urls)),
]
```

```python
# project/urls.py
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
    path('api-auth/', include('rest_framework.urls')),  # Login/Logout สำหรับ Browsable API
]
```

---

## 7. Browsable API

DRF มี Browsable API ที่ทำให้ทดสอบ API ได้จาก browser โดยตรง

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',  # เปิด Browsable API
    ],
}
```

เมื่อเปิด http://localhost:8000/api/ จะเห็น interface สวยงามที่:
- แสดง URL ทั้งหมด
- ทดสอบ GET/POST/PUT/DELETE ได้
- แสดง documentation อัตโนมัติ
- Login/Logout ได้

---

## 8. Response Formats

```python
# views.py
from rest_framework.response import Response
from rest_framework import status

# Response พื้นฐาน
return Response({'message': 'สำเร็จ'})

# Response พร้อม status code
return Response(data, status=status.HTTP_201_CREATED)
return Response({'error': 'ไม่พบข้อมูล'}, status=status.HTTP_404_NOT_FOUND)
return Response({'error': 'ไม่มีสิทธิ์'}, status=status.HTTP_403_FORBIDDEN)
return Response(None, status=status.HTTP_204_NO_CONTENT)

# Status codes ที่ใช้บ่อย:
# 200 OK
# 201 Created
# 204 No Content
# 400 Bad Request
# 401 Unauthorized
# 403 Forbidden
# 404 Not Found
# 500 Internal Server Error
```

---

## 9. API Documentation ด้วย drf-spectacular

```bash
pip install drf-spectacular
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'drf_spectacular',
]

REST_FRAMEWORK = {
    # ...
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'My API',
    'DESCRIPTION': 'API Documentation',
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,
}
```

```python
# urls.py
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
    SpectacularRedocView,
)

urlpatterns = [
    # Schema YAML/JSON
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    
    # Swagger UI
    path('api/docs/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    
    # ReDoc
    path('api/redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),
]
```

### เพิ่ม Documentation ใน ViewSet

```python
from drf_spectacular.utils import extend_schema, OpenApiParameter, OpenApiExample

class ArticleViewSet(viewsets.ModelViewSet):
    
    @extend_schema(
        summary='รายการบทความ',
        description='ดึงรายการบทความทั้งหมด รองรับ filtering และ pagination',
        parameters=[
            OpenApiParameter(
                name='status',
                description='กรองตาม status',
                required=False,
                type=str,
                enum=['draft', 'published', 'archived']
            ),
            OpenApiParameter(
                name='search',
                description='ค้นหาใน title และ content',
                required=False,
                type=str
            ),
        ],
        responses={200: ArticleListSerializer(many=True)},
        examples=[
            OpenApiExample(
                'Example Response',
                value={
                    'count': 10,
                    'results': [{'id': 1, 'title': 'Test Article'}]
                }
            )
        ]
    )
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```

---

## 10. ทดสอบ API ด้วย curl

```bash
# ดู API root
curl http://localhost:8000/api/

# Login รับ token
curl -X POST http://localhost:8000/api/auth/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "admin123"}'

# GET articles
curl -H "Authorization: Token <your-token>" \
  "http://localhost:8000/api/articles/"

# GET articles with filter
curl "http://localhost:8000/api/articles/?status=published&search=django"

# POST - สร้าง article
curl -X POST "http://localhost:8000/api/articles/" \
  -H "Authorization: Token <your-token>" \
  -H "Content-Type: application/json" \
  -d '{"title": "New Article", "slug": "new-article", "content": "Content here", "status": "draft"}'

# PATCH - แก้ไขบทความ
curl -X PATCH "http://localhost:8000/api/articles/1/" \
  -H "Authorization: Token <your-token>" \
  -H "Content-Type: application/json" \
  -d '{"title": "Updated Title"}'

# DELETE - ลบบทความ
curl -X DELETE "http://localhost:8000/api/articles/1/" \
  -H "Authorization: Token <your-token>"
```

---

## 11. สรุป Part 064

✅ **DRF** เป็น framework ยอดนิยมสำหรับสร้าง REST API ด้วย Django
✅ **Serializers** แปลง Python objects เป็น JSON และ validate ข้อมูล
✅ **ViewSets** รวม CRUD operations ใน class เดียว
✅ **Routers** สร้าง URL patterns อัตโนมัติจาก ViewSets
✅ **Custom actions** ใช้ `@action` decorator สร้าง endpoints พิเศษ
✅ **Browsable API** ช่วย test และ document API ได้จาก browser
✅ การตั้งค่า `REST_FRAMEWORK` ใน settings.py ควบคุมพฤติกรรม default
✅ **drf-spectacular** สร้าง OpenAPI documentation อัตโนมัติ

## ➡️ ถัดไป: Part 065 - DRF Serializers

*Part 064/100+ | Python Course - Beginner to World-Class*
