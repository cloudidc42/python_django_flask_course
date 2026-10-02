# Part 066: DRF Views

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจความแตกต่างระหว่าง APIView, GenericAPIView, ViewSet
- ใช้ Generic Views เช่น ListCreateAPIView, RetrieveUpdateDestroyAPIView
- สร้าง ViewSet และ ModelViewSet
- ใช้ Mixins เพื่อรวม behaviors
- จัดการ permissions และ authentication ต่อ view

---

## 1. ลำดับชั้นของ DRF Views

```
View (Django)
└── APIView                          # DRF base view
    └── GenericAPIView               # + queryset, serializer_class
        ├── CreateAPIView            # POST
        ├── ListAPIView              # GET list
        ├── RetrieveAPIView          # GET detail
        ├── UpdateAPIView            # PUT/PATCH
        ├── DestroyAPIView           # DELETE
        ├── ListCreateAPIView        # GET list + POST
        └── RetrieveUpdateDestroyAPIView  # GET + PUT/PATCH + DELETE

ViewSetMixin
├── ViewSet                          # ViewSet พื้นฐาน
│   └── GenericViewSet               # + GenericAPIView features
│       ├── ReadOnlyModelViewSet     # GET list + GET detail
│       └── ModelViewSet             # CRUD ครบ
```

---

## 2. APIView

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated
from .models import Article
from .serializers import ArticleSerializer

class ArticleListAPIView(APIView):
    """
    APIView พื้นฐาน - ควบคุมได้ทุกอย่าง แต่เขียนเยอะ
    
    GET  /api/articles/   - รายการบทความ
    POST /api/articles/   - สร้างบทความใหม่
    """
    permission_classes = [IsAuthenticated]
    
    def get(self, request, format=None):
        """GET - list articles"""
        articles = Article.objects.filter(
            status='published'
        ).select_related('author', 'category')
        
        serializer = ArticleSerializer(
            articles,
            many=True,
            context={'request': request}  # ส่ง request ไปด้วย (สำหรับ URL generation)
        )
        return Response(serializer.data)
    
    def post(self, request, format=None):
        """POST - create article"""
        serializer = ArticleSerializer(
            data=request.data,
            context={'request': request}
        )
        
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(
                serializer.data,
                status=status.HTTP_201_CREATED
            )
        
        return Response(
            serializer.errors,
            status=status.HTTP_400_BAD_REQUEST
        )


class ArticleDetailAPIView(APIView):
    """
    GET    /api/articles/<pk>/ - ดูบทความ
    PUT    /api/articles/<pk>/ - แก้ไขบทความ
    DELETE /api/articles/<pk>/ - ลบบทความ
    """
    
    def get_object(self, pk):
        """Helper method ดึง article"""
        from django.shortcuts import get_object_or_404
        return get_object_or_404(Article, pk=pk, status='published')
    
    def get(self, request, pk):
        article = self.get_object(pk)
        serializer = ArticleSerializer(article, context={'request': request})
        return Response(serializer.data)
    
    def put(self, request, pk):
        article = self.get_object(pk)
        
        # ตรวจสอบ permission
        if article.author != request.user and not request.user.is_staff:
            return Response(
                {'error': 'ไม่มีสิทธิ์แก้ไขบทความนี้'},
                status=status.HTTP_403_FORBIDDEN
            )
        
        serializer = ArticleSerializer(
            article,
            data=request.data,
            context={'request': request}
        )
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def patch(self, request, pk):
        """PATCH - partial update"""
        article = self.get_object(pk)
        serializer = ArticleSerializer(
            article,
            data=request.data,
            partial=True,  # อนุญาตให้ส่งแค่บางฟิลด์
            context={'request': request}
        )
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def delete(self, request, pk):
        article = self.get_object(pk)
        article.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

---

## 3. GenericAPIView + Mixins

```python
from rest_framework import generics, mixins
from rest_framework.permissions import IsAuthenticatedOrReadOnly

# GenericAPIView ใช้ร่วมกับ mixins
class ArticleListView(
    mixins.ListModelMixin,
    mixins.CreateModelMixin,
    generics.GenericAPIView
):
    queryset = Article.objects.filter(status='published')
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)
    
    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)


class ArticleDetailView(
    mixins.RetrieveModelMixin,
    mixins.UpdateModelMixin,
    mixins.DestroyModelMixin,
    generics.GenericAPIView
):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    
    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)
    
    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)
    
    def patch(self, request, *args, **kwargs):
        return self.partial_update(request, *args, **kwargs)
    
    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```

---

## 4. Concrete Generic Views

```python
from rest_framework import generics
from rest_framework.permissions import IsAuthenticated, IsAuthenticatedOrReadOnly, AllowAny

# ListCreateAPIView = ListModelMixin + CreateModelMixin + GenericAPIView
class ArticleListCreateView(generics.ListCreateAPIView):
    """GET list + POST create"""
    queryset = Article.objects.filter(status='published').order_by('-created_at')
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def get_queryset(self):
        """Custom queryset"""
        queryset = super().get_queryset()
        
        # Filter by category
        category = self.request.query_params.get('category')
        if category:
            queryset = queryset.filter(category__slug=category)
        
        # Search
        search = self.request.query_params.get('search')
        if search:
            from django.db.models import Q
            queryset = queryset.filter(
                Q(title__icontains=search) | Q(content__icontains=search)
            )
        
        return queryset
    
    def perform_create(self, serializer):
        """กำหนดค่าเพิ่มเติมเมื่อ create"""
        serializer.save(author=self.request.user)
    
    def get_serializer_class(self):
        """ใช้ serializer ต่างกันตาม action"""
        if self.request.method == 'POST':
            return ArticleCreateSerializer
        return ArticleListSerializer


# RetrieveAPIView = RetrieveModelMixin + GenericAPIView
class ArticleDetailView(generics.RetrieveAPIView):
    """GET detail only"""
    queryset = Article.objects.all()
    serializer_class = ArticleDetailSerializer
    permission_classes = [AllowAny]
    lookup_field = 'slug'  # ใช้ slug แทน pk
    
    def retrieve(self, request, *args, **kwargs):
        """Override retrieve เพิ่ม views count"""
        instance = self.get_object()
        instance.views_count += 1
        instance.save(update_fields=['views_count'])  # update เฉพาะ field นี้
        
        serializer = self.get_serializer(instance)
        return Response(serializer.data)


# RetrieveUpdateDestroyAPIView = GET + PUT/PATCH + DELETE
class ArticleManageView(generics.RetrieveUpdateDestroyAPIView):
    """CRUD สำหรับ article เดียว"""
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticated]
    
    def get_object(self):
        """Override เพื่อตรวจสอบว่าเป็น author"""
        obj = super().get_object()
        if obj.author != self.request.user and not self.request.user.is_staff:
            from rest_framework.exceptions import PermissionDenied
            raise PermissionDenied('ไม่มีสิทธิ์จัดการบทความนี้')
        return obj
    
    def perform_destroy(self, instance):
        """Custom behavior เมื่อลบ"""
        # ส่งอีเมลแจ้งเตือน หรือ log
        print(f'Deleting article: {instance.title}')
        instance.delete()


# UpdateAPIView = PUT/PATCH only
class ArticleUpdateView(generics.UpdateAPIView):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticated]


# DestroyAPIView = DELETE only
class ArticleDeleteView(generics.DestroyAPIView):
    queryset = Article.objects.all()
    permission_classes = [IsAuthenticated]
```

---

## 5. ViewSet

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework import status

class ArticleViewSet(viewsets.ViewSet):
    """
    ViewSet พื้นฐาน - ต้อง implement methods เอง
    
    list()          -> GET /articles/
    create()        -> POST /articles/
    retrieve()      -> GET /articles/{pk}/
    update()        -> PUT /articles/{pk}/
    partial_update()-> PATCH /articles/{pk}/
    destroy()       -> DELETE /articles/{pk}/
    """
    
    def list(self, request):
        queryset = Article.objects.all()
        serializer = ArticleSerializer(queryset, many=True)
        return Response(serializer.data)
    
    def create(self, request):
        serializer = ArticleSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save(author=request.user)
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def retrieve(self, request, pk=None):
        article = get_object_or_404(Article, pk=pk)
        serializer = ArticleSerializer(article)
        return Response(serializer.data)
    
    def update(self, request, pk=None):
        article = get_object_or_404(Article, pk=pk)
        serializer = ArticleSerializer(article, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def partial_update(self, request, pk=None):
        article = get_object_or_404(Article, pk=pk)
        serializer = ArticleSerializer(article, data=request.data, partial=True)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def destroy(self, request, pk=None):
        article = get_object_or_404(Article, pk=pk)
        article.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

---

## 6. ModelViewSet

```python
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.permissions import IsAuthenticated, IsAuthenticatedOrReadOnly

class ArticleModelViewSet(viewsets.ModelViewSet):
    """
    ModelViewSet - CRUD ครบพร้อมใช้งาน
    
    - list, create, retrieve, update, partial_update, destroy
    - Custom actions ด้วย @action
    """
    serializer_class = ArticleSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
    
    def get_queryset(self):
        user = self.request.user
        queryset = Article.objects.select_related('author', 'category')
        
        # ถ้าไม่ได้ login แสดงเฉพาะ published
        if not user.is_authenticated:
            return queryset.filter(status='published')
        
        # ถ้าเป็น staff เห็นทั้งหมด
        if user.is_staff:
            return queryset.all()
        
        # user ปกติเห็น published + ของตัวเอง
        from django.db.models import Q
        return queryset.filter(
            Q(status='published') | Q(author=user)
        )
    
    def get_serializer_class(self):
        if self.action in ['list']:
            return ArticleListSerializer
        elif self.action in ['create', 'update', 'partial_update']:
            return ArticleWriteSerializer
        return ArticleDetailSerializer
    
    def get_permissions(self):
        """กำหนด permissions ตาม action"""
        if self.action in ['list', 'retrieve']:
            return [AllowAny()]
        elif self.action in ['create']:
            return [IsAuthenticated()]
        elif self.action in ['update', 'partial_update', 'destroy']:
            return [IsAuthenticated(), IsArticleAuthor()]
        return super().get_permissions()
    
    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
    
    # Custom actions
    @action(
        detail=True,
        methods=['post'],
        permission_classes=[IsAuthenticated],
        url_path='publish'
    )
    def publish(self, request, pk=None):
        """POST /api/articles/{pk}/publish/"""
        article = self.get_object()
        
        if article.status == 'published':
            return Response(
                {'message': 'บทความนี้เผยแพร่แล้ว'},
                status=status.HTTP_400_BAD_REQUEST
            )
        
        article.status = 'published'
        article.save()
        
        serializer = self.get_serializer(article)
        return Response(serializer.data)
    
    @action(
        detail=True,
        methods=['post'],
        permission_classes=[IsAuthenticated],
        url_path='unpublish'
    )
    def unpublish(self, request, pk=None):
        """POST /api/articles/{pk}/unpublish/"""
        article = self.get_object()
        article.status = 'draft'
        article.save()
        return Response({'message': 'นำบทความออกจากการเผยแพร่แล้ว'})
    
    @action(
        detail=False,
        methods=['get'],
        permission_classes=[IsAuthenticated],
        url_path='my-articles'
    )
    def my_articles(self, request):
        """GET /api/articles/my-articles/"""
        queryset = Article.objects.filter(author=request.user).order_by('-created_at')
        
        # Pagination
        page = self.paginate_queryset(queryset)
        if page is not None:
            serializer = ArticleListSerializer(page, many=True, context={'request': request})
            return self.get_paginated_response(serializer.data)
        
        serializer = ArticleListSerializer(queryset, many=True, context={'request': request})
        return Response(serializer.data)
    
    @action(
        detail=True,
        methods=['get', 'post'],
        permission_classes=[IsAuthenticatedOrReadOnly],
        url_path='comments'
    )
    def comments(self, request, pk=None):
        """
        GET  /api/articles/{pk}/comments/ - ดู comments
        POST /api/articles/{pk}/comments/ - เพิ่ม comment
        """
        article = self.get_object()
        
        if request.method == 'GET':
            comments = article.comments.all().order_by('created_at')
            serializer = CommentSerializer(comments, many=True, context={'request': request})
            return Response(serializer.data)
        
        elif request.method == 'POST':
            serializer = CommentSerializer(data=request.data, context={'request': request})
            if serializer.is_valid():
                serializer.save(article=article, author=request.user)
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    @action(
        detail=False,
        methods=['get'],
        url_path='trending'
    )
    def trending(self, request):
        """GET /api/articles/trending/ - บทความยอดนิยม"""
        articles = Article.objects.filter(
            status='published'
        ).order_by('-views_count')[:10]
        
        serializer = ArticleListSerializer(articles, many=True, context={'request': request})
        return Response(serializer.data)
```

---

## 7. ReadOnlyModelViewSet

```python
class CategoryReadOnlyViewSet(viewsets.ReadOnlyModelViewSet):
    """
    ViewSet สำหรับอ่านอย่างเดียว
    list() และ retrieve() เท่านั้น
    """
    queryset = Category.objects.annotate(
        articles_count=Count('articles', filter=Q(articles__status='published'))
    ).order_by('name')
    serializer_class = CategorySerializer
    permission_classes = [AllowAny]
    lookup_field = 'slug'
```

---

## 8. URLs สำหรับ ViewSets

```python
# api/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter, SimpleRouter
from . import views

# DefaultRouter - สร้าง URL ครบและมี API root view
router = DefaultRouter()
router.register(r'articles', views.ArticleModelViewSet, basename='article')
router.register(r'categories', views.CategoryReadOnlyViewSet, basename='category')

# SimpleRouter - ไม่มี API root view
simple_router = SimpleRouter()
simple_router.register(r'comments', views.CommentViewSet, basename='comment')

urlpatterns = [
    path('', include(router.urls)),
    path('', include(simple_router.urls)),
    
    # Manual URL สำหรับ APIView/Generic Views
    path('auth/login/', views.LoginAPIView.as_view(), name='api-login'),
    path('auth/logout/', views.LogoutAPIView.as_view(), name='api-logout'),
]
```

---

## 9. สรุป Part 066

✅ **APIView** ควบคุมได้ทุกอย่าง แต่เขียนโค้ดเยอะ
✅ **GenericAPIView** + Mixins เพิ่มความสะดวก
✅ **Concrete Generic Views** เช่น `ListCreateAPIView`, `RetrieveUpdateDestroyAPIView` ใช้งานได้เลย
✅ **ViewSet** รวม actions ต่างๆ ไว้ใน class เดียว
✅ **ModelViewSet** มี CRUD ครบพร้อมใช้งาน
✅ **@action decorator** เพิ่ม custom endpoints เข้า ViewSet
✅ **Router** สร้าง URL patterns อัตโนมัติจาก ViewSets
✅ **get_queryset()**, **get_serializer_class()**, **get_permissions()** override ได้ทุกอย่าง

## ➡️ ถัดไป: Part 067 - DRF Authentication

*Part 066/100+ | Python Course - Beginner to World-Class*
