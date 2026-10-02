# Part 60: Django REST Framework (DRF) Introduction

## เป้าหมายของบทเรียน

- ติดตั้งและตั้งค่า Django REST Framework
- สร้าง Serializers (ModelSerializer, Nested)
- สร้าง APIView
- ใช้ ViewSets และ Routers
- ตั้งค่า JWT Authentication
- กำหนด Permissions

---

## 1. Django REST Framework คืออะไร?

DRF (Django REST Framework) เป็น toolkit สำหรับสร้าง Web APIs ด้วย Django

**คุณสมบัติ:**
- Serialization สำหรับแปลง model เป็น JSON
- Authentication ระดับสูง
- Permissions system
- Browsable API (ดูและทดสอบ API ผ่าน browser)
- Throttling (rate limiting)
- Filtering, searching, ordering
- Pagination

---

## 2. ติดตั้ง DRF

```bash
# ติดตั้ง DRF
pip install djangorestframework

# ติดตั้ง packages เพิ่มเติม
pip install djangorestframework-simplejwt  # JWT authentication
pip install django-filter                   # Filtering
pip install Pillow                          # Image handling

# บันทึก requirements
pip freeze > requirements.txt
```

```python
# settings.py
INSTALLED_APPS = [
    # Django apps
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    # Third-party apps
    'rest_framework',           # DRF
    'rest_framework_simplejwt', # JWT
    'django_filters',           # Filtering
    
    # Local apps
    'blog',
    'accounts',
]

# DRF Configuration
REST_FRAMEWORK = {
    # Authentication
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
        'rest_framework.authentication.SessionAuthentication',
    ],
    
    # Permissions
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticatedOrReadOnly',
    ],
    
    # Pagination
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,
    
    # Filtering
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
    
    # Renderer
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',  # ลบออกใน production
    ],
    
    # Parser
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
        'rest_framework.parsers.MultiPartParser',  # สำหรับ file upload
        'rest_framework.parsers.FormParser',
    ],
    
    # Throttling (rate limiting)
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/day',
        'user': '1000/day',
    },
}

# JWT Configuration
from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,
    'AUTH_HEADER_TYPES': ('Bearer',),
}
```

---

## 3. Serializers

Serializer แปลง model instances เป็น JSON (serialization) และ JSON เป็น model instances (deserialization)

### Serializer พื้นฐาน

```python
# blog/serializers.py
from rest_framework import serializers
from .models import Category, Tag, Post, Comment
from django.contrib.auth import get_user_model

User = get_user_model()


class CategorySerializer(serializers.ModelSerializer):
    """Serializer สำหรับ Category"""
    
    # เพิ่ม field ที่ไม่มีใน model (computed field)
    post_count = serializers.SerializerMethodField()
    
    class Meta:
        model = Category
        fields = ['id', 'name', 'slug', 'description', 'color', 'post_count']
        # หรือ fields = '__all__'  ทุก fields
        # หรือ exclude = ['created_at']  ยกเว้น
        
        # fields ที่ read-only
        read_only_fields = ['slug']
    
    def get_post_count(self, obj):
        """คำนวณจำนวน posts"""
        return obj.posts.filter(status='published').count()


class TagSerializer(serializers.ModelSerializer):
    """Serializer สำหรับ Tag"""
    
    class Meta:
        model = Tag
        fields = ['id', 'name', 'slug']
        read_only_fields = ['slug']


class UserBasicSerializer(serializers.ModelSerializer):
    """Serializer สำหรับ User (ข้อมูลพื้นฐาน)"""
    full_name = serializers.SerializerMethodField()
    
    class Meta:
        model = User
        fields = ['id', 'username', 'full_name', 'email']
    
    def get_full_name(self, obj):
        return obj.get_full_name() or obj.username
```

### Nested Serializers

```python
class PostListSerializer(serializers.ModelSerializer):
    """
    Serializer สำหรับรายการ posts (ข้อมูลบางส่วน)
    """
    author = UserBasicSerializer(read_only=True)  # nested serializer
    category = CategorySerializer(read_only=True)
    tags = TagSerializer(many=True, read_only=True)  # many=True สำหรับ list
    reading_time = serializers.SerializerMethodField()
    
    class Meta:
        model = Post
        fields = [
            'id', 'title', 'slug', 'excerpt',
            'author', 'category', 'tags',
            'featured_image', 'status', 'is_featured',
            'created_at', 'views_count', 'reading_time',
        ]
    
    def get_reading_time(self, obj):
        return obj.reading_time


class PostDetailSerializer(serializers.ModelSerializer):
    """
    Serializer สำหรับรายละเอียด post (ข้อมูลครบ)
    """
    author = UserBasicSerializer(read_only=True)
    category = CategorySerializer(read_only=True)
    tags = TagSerializer(many=True, read_only=True)
    comment_count = serializers.SerializerMethodField()
    
    # field สำหรับ write (รับ ID แทน nested object)
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(),
        source='category',
        write_only=True,
        required=False,
        allow_null=True,
    )
    tag_ids = serializers.PrimaryKeyRelatedField(
        queryset=Tag.objects.all(),
        source='tags',
        many=True,
        write_only=True,
        required=False,
    )
    
    class Meta:
        model = Post
        fields = [
            'id', 'title', 'slug', 'excerpt', 'content',
            'author', 'category', 'category_id',
            'tags', 'tag_ids',
            'featured_image', 'status', 'is_featured',
            'published_at', 'created_at', 'updated_at',
            'views_count', 'comment_count',
        ]
        read_only_fields = ['slug', 'author', 'views_count', 'created_at', 'updated_at']
    
    def get_comment_count(self, obj):
        return obj.comments.filter(is_approved=True).count()
    
    def create(self, validated_data):
        """Custom create"""
        tags = validated_data.pop('tags', [])
        post = Post.objects.create(**validated_data)
        post.tags.set(tags)
        return post
    
    def update(self, instance, validated_data):
        """Custom update"""
        tags = validated_data.pop('tags', None)
        
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        
        if tags is not None:
            instance.tags.set(tags)
        
        return instance


class CommentSerializer(serializers.ModelSerializer):
    """Serializer สำหรับ Comment"""
    author = UserBasicSerializer(read_only=True)
    replies = serializers.SerializerMethodField()
    
    class Meta:
        model = Comment
        fields = ['id', 'content', 'author', 'post', 'parent', 'replies', 
                  'is_approved', 'created_at']
        read_only_fields = ['author', 'is_approved', 'created_at']
    
    def get_replies(self, obj):
        if obj.replies.exists():
            return CommentSerializer(
                obj.replies.filter(is_approved=True),
                many=True,
                context=self.context
            ).data
        return []
```

### Custom Validation ใน Serializer

```python
class PostCreateSerializer(serializers.ModelSerializer):
    """Serializer สำหรับสร้าง Post"""
    
    class Meta:
        model = Post
        fields = ['title', 'excerpt', 'content', 'category', 'tags', 
                  'featured_image', 'status']
    
    def validate_title(self, value):
        """Validate title"""
        if len(value) < 10:
            raise serializers.ValidationError(
                'หัวข้อต้องมีอย่างน้อย 10 ตัวอักษร'
            )
        return value
    
    def validate_content(self, value):
        """Validate content"""
        word_count = len(value.split())
        if word_count < 50:
            raise serializers.ValidationError(
                f'เนื้อหาน้อยเกินไป ({word_count} คำ, ต้องมีอย่างน้อย 50 คำ)'
            )
        return value
    
    def validate(self, data):
        """Cross-field validation"""
        status = data.get('status')
        content = data.get('content', '')
        
        if status == 'published' and len(content.split()) < 100:
            raise serializers.ValidationError({
                'content': 'บทความที่เผยแพร่ต้องมีอย่างน้อย 100 คำ'
            })
        
        return data
    
    def create(self, validated_data):
        tags = validated_data.pop('tags', [])
        # ดึง author จาก context (request.user)
        author = self.context['request'].user
        post = Post.objects.create(author=author, **validated_data)
        post.tags.set(tags)
        return post
```

---

## 4. APIView

```python
# blog/views_api.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated, IsAuthenticatedOrReadOnly
from django.shortcuts import get_object_or_404

from .models import Post, Category, Comment
from .serializers import (
    PostListSerializer, PostDetailSerializer, PostCreateSerializer,
    CategorySerializer, CommentSerializer
)


class PostListAPIView(APIView):
    """
    GET /api/posts/ -> รายการ posts
    POST /api/posts/ -> สร้าง post ใหม่
    """
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def get(self, request):
        """GET: รายการ posts"""
        posts = Post.objects.filter(
            status='published'
        ).select_related('author', 'category').prefetch_related('tags')
        
        # Search
        search = request.query_params.get('q', '')
        if search:
            posts = posts.filter(title__icontains=search)
        
        # Filter by category
        category = request.query_params.get('category', '')
        if category:
            posts = posts.filter(category__slug=category)
        
        # Pagination
        page_size = int(request.query_params.get('page_size', 10))
        page = int(request.query_params.get('page', 1))
        start = (page - 1) * page_size
        end = start + page_size
        
        total = posts.count()
        posts_page = posts[start:end]
        
        serializer = PostListSerializer(posts_page, many=True, context={'request': request})
        
        return Response({
            'count': total,
            'page': page,
            'page_size': page_size,
            'total_pages': (total + page_size - 1) // page_size,
            'results': serializer.data,
        })
    
    def post(self, request):
        """POST: สร้าง post ใหม่"""
        serializer = PostCreateSerializer(
            data=request.data,
            context={'request': request}
        )
        
        if serializer.is_valid():
            post = serializer.save()
            # return detail serializer
            detail_serializer = PostDetailSerializer(post, context={'request': request})
            return Response(detail_serializer.data, status=status.HTTP_201_CREATED)
        
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class PostDetailAPIView(APIView):
    """
    GET /api/posts/{slug}/ -> รายละเอียด post
    PUT/PATCH /api/posts/{slug}/ -> อัปเดต post
    DELETE /api/posts/{slug}/ -> ลบ post
    """
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def get_object(self, slug):
        """ดึง post หรือ 404"""
        return get_object_or_404(
            Post.objects.select_related('author', 'category').prefetch_related('tags'),
            slug=slug,
            status='published'
        )
    
    def get(self, request, slug):
        """GET: รายละเอียด post"""
        post = self.get_object(slug)
        post.increment_views()
        serializer = PostDetailSerializer(post, context={'request': request})
        return Response(serializer.data)
    
    def put(self, request, slug):
        """PUT: อัปเดต post ทั้งหมด"""
        post = self.get_object(slug)
        
        # ตรวจสอบว่าเป็นเจ้าของ
        if post.author != request.user and not request.user.is_staff:
            return Response(
                {'error': 'คุณไม่มีสิทธิ์แก้ไขบทความนี้'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        serializer = PostDetailSerializer(
            post,
            data=request.data,
            context={'request': request}
        )
        
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def patch(self, request, slug):
        """PATCH: อัปเดต post บางส่วน"""
        post = self.get_object(slug)
        
        if post.author != request.user and not request.user.is_staff:
            return Response(
                {'error': 'คุณไม่มีสิทธิ์'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        serializer = PostDetailSerializer(
            post,
            data=request.data,
            partial=True,  # partial=True สำหรับ PATCH
            context={'request': request}
        )
        
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def delete(self, request, slug):
        """DELETE: ลบ post"""
        post = self.get_object(slug)
        
        if post.author != request.user and not request.user.is_staff:
            return Response(
                {'error': 'คุณไม่มีสิทธิ์'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        post.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

---

## 5. ViewSets และ Routers

ViewSets รวม logic ของหลาย views ไว้ในที่เดียว

```python
# blog/viewsets.py
from rest_framework import viewsets, filters, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated, IsAuthenticatedOrReadOnly
from django_filters.rest_framework import DjangoFilterBackend
from django.db.models import Count

from .models import Post, Category, Tag, Comment
from .serializers import (
    PostListSerializer, PostDetailSerializer, PostCreateSerializer,
    CategorySerializer, TagSerializer, CommentSerializer
)
from .permissions import IsOwnerOrReadOnly


class CategoryViewSet(viewsets.ModelViewSet):
    """
    ViewSet สำหรับ Category CRUD
    GET /api/categories/ -> list
    POST /api/categories/ -> create
    GET /api/categories/{slug}/ -> retrieve
    PUT /api/categories/{slug}/ -> update
    PATCH /api/categories/{slug}/ -> partial_update
    DELETE /api/categories/{slug}/ -> destroy
    """
    queryset = Category.objects.annotate(post_count=Count('posts'))
    serializer_class = CategorySerializer
    lookup_field = 'slug'  # ใช้ slug แทน id
    
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    filter_backends = [filters.SearchFilter, filters.OrderingFilter]
    search_fields = ['name', 'description']
    ordering_fields = ['name', 'post_count']
    ordering = ['name']


class PostViewSet(viewsets.ModelViewSet):
    """
    ViewSet สำหรับ Post CRUD
    """
    lookup_field = 'slug'
    
    # Filtering
    filter_backends = [
        DjangoFilterBackend,
        filters.SearchFilter,
        filters.OrderingFilter,
    ]
    filterset_fields = ['status', 'is_featured', 'category']
    search_fields = ['title', 'content', 'author__username', 'tags__name']
    ordering_fields = ['created_at', 'views_count', 'title']
    ordering = ['-created_at']
    
    def get_queryset(self):
        """กรอง queryset ตาม action"""
        queryset = Post.objects.select_related(
            'author', 'category'
        ).prefetch_related('tags')
        
        # ถ้าเป็น authenticated user แสดง posts ของตัวเอง (ทุก status)
        if self.request.user.is_authenticated and self.action in ['list', 'retrieve']:
            # public posts + own posts
            from django.db.models import Q
            queryset = queryset.filter(
                Q(status='published') | Q(author=self.request.user)
            )
        else:
            queryset = queryset.filter(status='published')
        
        return queryset
    
    def get_serializer_class(self):
        """ใช้ serializer ต่างกันตาม action"""
        if self.action == 'list':
            return PostListSerializer
        elif self.action == 'create':
            return PostCreateSerializer
        return PostDetailSerializer
    
    def get_permissions(self):
        """กำหนด permissions ตาม action"""
        if self.action in ['create']:
            permissions = [IsAuthenticated()]
        elif self.action in ['update', 'partial_update', 'destroy']:
            permissions = [IsAuthenticated(), IsOwnerOrReadOnly()]
        else:
            permissions = [IsAuthenticatedOrReadOnly()]
        return permissions
    
    def perform_create(self, serializer):
        """กำหนด author เป็น user ที่ login"""
        serializer.save(author=self.request.user)
    
    # Custom actions
    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def publish(self, request, slug=None):
        """
        POST /api/posts/{slug}/publish/
        เผยแพร่บทความ
        """
        post = self.get_object()
        
        if post.author != request.user and not request.user.is_staff:
            return Response(
                {'error': 'คุณไม่มีสิทธิ์เผยแพร่บทความนี้'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        if post.status == 'published':
            return Response({'message': 'บทความนี้เผยแพร่แล้ว'})
        
        post.publish()
        serializer = self.get_serializer(post)
        return Response(serializer.data)
    
    @action(detail=True, methods=['get'])
    def comments(self, request, slug=None):
        """
        GET /api/posts/{slug}/comments/
        ดึง comments ของ post
        """
        post = self.get_object()
        comments = post.comments.filter(
            is_approved=True,
            parent__isnull=True
        ).select_related('author').prefetch_related('replies__author')
        
        serializer = CommentSerializer(
            comments, many=True, context={'request': request}
        )
        return Response(serializer.data)
    
    @action(detail=True, methods=['post'], permission_classes=[IsAuthenticated])
    def add_comment(self, request, slug=None):
        """
        POST /api/posts/{slug}/add_comment/
        เพิ่ม comment
        """
        post = self.get_object()
        
        serializer = CommentSerializer(
            data=request.data,
            context={'request': request}
        )
        
        if serializer.is_valid():
            serializer.save(
                post=post,
                author=request.user
            )
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    @action(detail=False, methods=['get'])
    def featured(self, request):
        """
        GET /api/posts/featured/
        ดึง featured posts
        """
        posts = self.get_queryset().filter(is_featured=True)[:5]
        serializer = PostListSerializer(posts, many=True, context={'request': request})
        return Response(serializer.data)
    
    @action(detail=False, methods=['get'], permission_classes=[IsAuthenticated])
    def my_posts(self, request):
        """
        GET /api/posts/my_posts/
        ดึง posts ของ user ที่ login
        """
        posts = Post.objects.filter(
            author=request.user
        ).order_by('-created_at')
        
        serializer = PostListSerializer(posts, many=True, context={'request': request})
        return Response(serializer.data)
```

### Routers

```python
# blog/urls.py (API URLs)
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import viewsets

# สร้าง router
router = DefaultRouter()
router.register('posts', viewsets.PostViewSet, basename='post')
router.register('categories', viewsets.CategoryViewSet, basename='category')

urlpatterns = [
    # API endpoints จาก router
    path('api/', include(router.urls)),
    
    # Manual API views
    path('api/posts/<slug:slug>/comments/', viewsets.PostComments.as_view()),
]
```

```
# URLs ที่ Router สร้างให้:
GET    /api/posts/              -> list
POST   /api/posts/              -> create
GET    /api/posts/{slug}/       -> retrieve
PUT    /api/posts/{slug}/       -> update
PATCH  /api/posts/{slug}/       -> partial_update
DELETE /api/posts/{slug}/       -> destroy
POST   /api/posts/{slug}/publish/  -> custom action
GET    /api/posts/{slug}/comments/ -> custom action
POST   /api/posts/{slug}/add_comment/ -> custom action
GET    /api/posts/featured/     -> custom list action
GET    /api/posts/my_posts/     -> custom list action
```

---

## 6. JWT Authentication

```python
# accounts/urls.py (JWT endpoints)
from django.urls import path
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
    TokenBlacklistView,
)
from . import views

urlpatterns = [
    # JWT endpoints
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
    path('api/token/blacklist/', TokenBlacklistView.as_view(), name='token_blacklist'),  # logout
    
    # User endpoints
    path('api/register/', views.RegisterAPIView.as_view(), name='api_register'),
    path('api/me/', views.UserProfileAPIView.as_view(), name='api_me'),
]
```

### Custom JWT Claims

```python
# accounts/serializers.py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer


class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    """เพิ่ม custom claims ใน JWT token"""
    
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        
        # เพิ่ม claims พิเศษ
        token['username'] = user.username
        token['email'] = user.email
        token['is_staff'] = user.is_staff
        token['full_name'] = user.get_full_name()
        
        return token
```

### Register API

```python
# accounts/views_api.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import AllowAny, IsAuthenticated
from rest_framework_simplejwt.tokens import RefreshToken
from django.contrib.auth import get_user_model
from .serializers import UserSerializer, RegisterSerializer

User = get_user_model()


class RegisterAPIView(APIView):
    """
    POST /api/accounts/register/
    สมัครสมาชิก
    """
    permission_classes = [AllowAny]
    
    def post(self, request):
        serializer = RegisterSerializer(data=request.data)
        
        if serializer.is_valid():
            user = serializer.save()
            
            # สร้าง JWT tokens
            refresh = RefreshToken.for_user(user)
            
            return Response({
                'user': UserSerializer(user).data,
                'refresh': str(refresh),
                'access': str(refresh.access_token),
                'message': 'สมัครสมาชิกสำเร็จ!'
            }, status=status.HTTP_201_CREATED)
        
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)


class UserProfileAPIView(APIView):
    """
    GET /api/accounts/me/ -> ดูข้อมูล user
    PUT/PATCH /api/accounts/me/ -> อัปเดตข้อมูล
    """
    permission_classes = [IsAuthenticated]
    
    def get(self, request):
        serializer = UserSerializer(request.user)
        return Response(serializer.data)
    
    def patch(self, request):
        serializer = UserSerializer(
            request.user,
            data=request.data,
            partial=True
        )
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

### ใช้ JWT ใน Client

```bash
# 1. ขอ access token
curl -X POST http://localhost:8000/api/token/ \
    -H "Content-Type: application/json" \
    -d '{"username": "somchai", "password": "password123"}'

# Response:
# {
#   "access": "eyJ0eXAiOiJKV1Qi...",
#   "refresh": "eyJ0eXAiOiJKV1Qi..."
# }

# 2. ใช้ access token
curl http://localhost:8000/api/posts/ \
    -H "Authorization: Bearer eyJ0eXAiOiJKV1Qi..."

# 3. Refresh token
curl -X POST http://localhost:8000/api/token/refresh/ \
    -H "Content-Type: application/json" \
    -d '{"refresh": "eyJ0eXAiOiJKV1Qi..."}'
```

---

## 7. Permissions

```python
# blog/permissions.py
from rest_framework.permissions import BasePermission, SAFE_METHODS


class IsOwnerOrReadOnly(BasePermission):
    """
    Custom permission:
    - Read (GET, HEAD, OPTIONS): ทุกคน
    - Write (POST, PUT, PATCH, DELETE): เฉพาะเจ้าของ
    """
    
    def has_object_permission(self, request, view, obj):
        # Read permissions
        if request.method in SAFE_METHODS:
            return True
        
        # Write permissions: เจ้าของหรือ staff
        return obj.author == request.user or request.user.is_staff


class IsStaffOrReadOnly(BasePermission):
    """
    Staff เท่านั้นที่ write ได้
    """
    
    def has_permission(self, request, view):
        if request.method in SAFE_METHODS:
            return True
        return request.user.is_authenticated and request.user.is_staff


class IsOwner(BasePermission):
    """
    เฉพาะเจ้าของเท่านั้น
    """
    
    def has_object_permission(self, request, view, obj):
        return obj.author == request.user


class ReadOnly(BasePermission):
    """
    Read-only เท่านั้น
    """
    
    def has_permission(self, request, view):
        return request.method in SAFE_METHODS
```

### ใช้ Permission ใน View

```python
from rest_framework.permissions import IsAuthenticated, AllowAny
from .permissions import IsOwnerOrReadOnly, IsStaffOrReadOnly


class PostViewSet(viewsets.ModelViewSet):
    # ...
    
    def get_permissions(self):
        if self.action == 'list' or self.action == 'retrieve':
            # ทุกคนอ่านได้
            return [AllowAny()]
        elif self.action == 'create':
            # ต้อง login
            return [IsAuthenticated()]
        else:
            # update/delete: เจ้าของหรือ staff
            return [IsAuthenticated(), IsOwnerOrReadOnly()]
```

---

## 8. Pagination

```python
# blog/pagination.py
from rest_framework.pagination import PageNumberPagination, CursorPagination


class StandardResultsPagination(PageNumberPagination):
    """
    Pagination มาตรฐาน
    """
    page_size = 10
    page_size_query_param = 'page_size'  # ?page_size=20
    max_page_size = 100
    
    def get_paginated_response(self, data):
        """Custom response format"""
        return Response({
            'pagination': {
                'count': self.page.paginator.count,
                'next': self.get_next_link(),
                'previous': self.get_previous_link(),
                'page': self.page.number,
                'total_pages': self.page.paginator.num_pages,
            },
            'results': data,
        })


class PostCursorPagination(CursorPagination):
    """
    Cursor pagination สำหรับ real-time feeds
    """
    page_size = 10
    ordering = '-created_at'


# ใช้กับ ViewSet
class PostViewSet(viewsets.ModelViewSet):
    pagination_class = StandardResultsPagination
    # ...
```

---

## 9. ตัวอย่างเต็ม: Blog API

```python
# blog/api_urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import viewsets
from accounts.views_api import RegisterAPIView, UserProfileAPIView
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

router = DefaultRouter()
router.register('posts', viewsets.PostViewSet, basename='post')
router.register('categories', viewsets.CategoryViewSet, basename='category')
router.register('tags', viewsets.TagViewSet, basename='tag')

urlpatterns = [
    # Blog API
    path('', include(router.urls)),
    
    # Auth
    path('auth/register/', RegisterAPIView.as_view(), name='api_register'),
    path('auth/login/', TokenObtainPairView.as_view(), name='api_login'),
    path('auth/refresh/', TokenRefreshView.as_view(), name='api_token_refresh'),
    path('auth/me/', UserProfileAPIView.as_view(), name='api_me'),
]
```

```python
# mysite/urls.py
urlpatterns = [
    path('admin/', admin.site.urls),
    path('blog/', include('blog.urls')),          # Web URLs
    path('api/v1/', include('blog.api_urls')),    # API URLs
    path('', include('pages.urls')),
]
```

---

## 10. API Documentation

```bash
# ติดตั้ง drf-spectacular
pip install drf-spectacular
```

```python
# settings.py
INSTALLED_APPS += ['drf_spectacular']

REST_FRAMEWORK['DEFAULT_SCHEMA_CLASS'] = 'drf_spectacular.openapi.AutoSchema'

SPECTACULAR_SETTINGS = {
    'TITLE': 'MyBlog API',
    'DESCRIPTION': 'API สำหรับ Blog application',
    'VERSION': '1.0.0',
}
```

```python
# urls.py
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
    SpectacularRedocView,
)

urlpatterns += [
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path('api/docs/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    path('api/redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),
]
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: E-commerce API
สร้าง REST API สำหรับ e-commerce:
- ProductViewSet (CRUD, custom action: add_to_cart)
- CategoryViewSet (nested subcategories)
- OrderViewSet (create order, list my orders)
- JWT authentication

### แบบฝึกหัดที่ 2: Nested Serializer
สร้าง serializer ที่มี nested relations:
- `OrderSerializer` ที่มี `OrderItemSerializer` แบบ nested
- write operation: สร้าง order พร้อม items ในครั้งเดียว
- Validation: ตรวจสอบ stock ก่อน create

### แบบฝึกหัดที่ 3: Custom Permission
สร้าง permission:
- `IsPremiumUser` - เฉพาะ premium users เท่านั้น
- `IsVerifiedUser` - เฉพาะ users ที่ verify email แล้ว
- `RateLimitedView` - จำกัดจำนวน requests ต่อ user

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- ติดตั้งและตั้งค่า DRF
- Serializers (ModelSerializer, Nested, Custom Validation)
- APIView สำหรับ manual API views
- ViewSets และ Routers สำหรับ CRUD API
- JWT Authentication ด้วย simplejwt
- Custom Permissions
- Pagination
- API Documentation ด้วย drf-spectacular

---

## สรุปหลักสูตร Django (Part 51-60)

เราได้เรียนรู้ Django ตั้งแต่พื้นฐานจนถึงขั้นสูง:

| Part | หัวข้อ | สาระสำคัญ |
|------|--------|-----------|
| 51 | Introduction & Setup | MVT Pattern, Project Setup, Hello World |
| 52 | Models | Field Types, ORM, QuerySet API |
| 53 | Migrations | makemigrations, migrate, Data Migrations |
| 54 | Views | FBV, CBV, Generic Views |
| 55 | Templates | DTL, Template Tags, Bootstrap |
| 56 | URLs | urlpatterns, Namespace, reverse() |
| 57 | Admin | ModelAdmin, Inlines, Custom Actions |
| 58 | Forms | Form, ModelForm, Validation |
| 59 | Authentication | Login, Permissions, Custom User Model |
| 60 | DRF | Serializers, ViewSets, JWT Auth |

---

## บทถัดไป

➡️ **[Part 61: Flask Introduction](part-061.md)** - เริ่มต้นเรียน Flask micro-framework
