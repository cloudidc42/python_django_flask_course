# Part 069: DRF Filtering and Pagination

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ django-filter สำหรับ filtering
- ใช้ SearchFilter และ OrderingFilter
- สร้าง Custom Filter
- ตั้งค่า PageNumberPagination
- ใช้ CursorPagination สำหรับ large datasets

---

## 1. Filtering ใน DRF

### Manual Filtering

```python
class ArticleViewSet(viewsets.ModelViewSet):
    
    def get_queryset(self):
        queryset = Article.objects.all()
        
        # กรองด้วย query params
        status = self.request.query_params.get('status')
        if status:
            queryset = queryset.filter(status=status)
        
        category = self.request.query_params.get('category')
        if category:
            queryset = queryset.filter(category__slug=category)
        
        author = self.request.query_params.get('author')
        if author:
            queryset = queryset.filter(author__username=author)
        
        return queryset
```

---

## 2. DRF Built-in Filter Backends

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
}
```

### SearchFilter

```python
from rest_framework.filters import SearchFilter

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    filter_backends = [SearchFilter]
    
    # กำหนด fields ที่ search ได้
    search_fields = [
        'title',           # ค้นหาใน title
        'content',         # ค้นหาใน content
        'author__username', # ค้นหาใน related field
        '=slug',           # exact match
        '^title',          # startswith
        '@content',        # full-text search (รองรับ MySQL, PostgreSQL)
    ]
    
    # ใช้งาน: GET /api/articles/?search=django
```

### OrderingFilter

```python
from rest_framework.filters import OrderingFilter

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    filter_backends = [OrderingFilter]
    
    # fields ที่ ordering ได้
    ordering_fields = ['title', 'created_at', 'views_count', 'author__username']
    
    # default ordering
    ordering = ['-created_at']  # เรียงจากใหม่ไปเก่า
    
    # ใช้งาน:
    # GET /api/articles/?ordering=title          (A-Z)
    # GET /api/articles/?ordering=-title         (Z-A)
    # GET /api/articles/?ordering=-created_at    (ใหม่ก่อน)
    # GET /api/articles/?ordering=views_count,-created_at  (หลาย fields)
```

---

## 3. django-filter

```bash
pip install django-filter
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'django_filters',
]

REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
    ],
}
```

### Simple FilterSet

```python
# filters.py
import django_filters
from django_filters import rest_framework as filters
from .models import Article, Category

class ArticleFilter(filters.FilterSet):
    """FilterSet สำหรับ Article"""
    
    # Exact match
    status = filters.CharFilter(field_name='status', lookup_expr='exact')
    
    # Case-insensitive contains
    title = filters.CharFilter(field_name='title', lookup_expr='icontains')
    
    # Range filters
    created_after = filters.DateTimeFilter(
        field_name='created_at',
        lookup_expr='gte'  # greater than or equal
    )
    created_before = filters.DateTimeFilter(
        field_name='created_at',
        lookup_expr='lte'  # less than or equal
    )
    
    # Foreign key
    category = filters.ModelChoiceFilter(queryset=Category.objects.all())
    category_slug = filters.CharFilter(
        field_name='category__slug',
        lookup_expr='exact'
    )
    
    # Multiple values
    status_in = filters.MultipleChoiceFilter(
        field_name='status',
        choices=Article.STATUS_CHOICES
    )
    
    # Boolean
    is_published = filters.BooleanFilter(
        method='filter_published'  # custom filter method
    )
    
    # Number range
    min_views = filters.NumberFilter(field_name='views_count', lookup_expr='gte')
    max_views = filters.NumberFilter(field_name='views_count', lookup_expr='lte')
    
    class Meta:
        model = Article
        fields = {
            'status': ['exact', 'in'],
            'title': ['exact', 'icontains', 'startswith'],
            'created_at': ['exact', 'gte', 'lte', 'year', 'month'],
            'views_count': ['exact', 'gte', 'lte'],
        }
    
    def filter_published(self, queryset, name, value):
        """Custom filter method"""
        if value:
            return queryset.filter(status='published')
        return queryset.exclude(status='published')


# ใช้ใน ViewSet
from django_filters.rest_framework import DjangoFilterBackend

class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    filter_backends = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class = ArticleFilter
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'views_count', 'title']
    ordering = ['-created_at']
    
    # ใช้งาน:
    # GET /api/articles/?status=published
    # GET /api/articles/?title=django
    # GET /api/articles/?created_after=2024-01-01&created_before=2024-12-31
    # GET /api/articles/?min_views=100
    # GET /api/articles/?search=python&ordering=-views_count
```

---

## 4. Custom Filter Backend

```python
from rest_framework import filters

class IsOwnerFilterBackend(filters.BaseFilterBackend):
    """Filter แสดงเฉพาะ objects ของ user ที่ login"""
    
    def filter_queryset(self, request, queryset, view):
        # ถ้าเป็น staff แสดงทั้งหมด
        if request.user.is_staff:
            return queryset
        
        # ถ้าเป็น user ปกติแสดงเฉพาะของตัวเอง
        if request.user.is_authenticated:
            return queryset.filter(author=request.user)
        
        # ถ้าไม่ login แสดงเฉพาะ published
        return queryset.filter(status='published')
    
    def get_schema_fields(self, view):
        """สำหรับ schema generation (optional)"""
        return []


class DateRangeFilterBackend(filters.BaseFilterBackend):
    """Filter ตาม date range"""
    
    def filter_queryset(self, request, queryset, view):
        start_date = request.query_params.get('start_date')
        end_date = request.query_params.get('end_date')
        date_field = getattr(view, 'date_filter_field', 'created_at')
        
        if start_date:
            try:
                from datetime import datetime
                start = datetime.strptime(start_date, '%Y-%m-%d')
                queryset = queryset.filter(**{f'{date_field}__gte': start})
            except ValueError:
                pass
        
        if end_date:
            try:
                from datetime import datetime
                end = datetime.strptime(end_date, '%Y-%m-%d')
                queryset = queryset.filter(**{f'{date_field}__lte': end})
            except ValueError:
                pass
        
        return queryset
```

---

## 5. Pagination

### PageNumberPagination

```python
# pagination.py
from rest_framework.pagination import PageNumberPagination
from rest_framework.response import Response

class StandardPagination(PageNumberPagination):
    """Pagination มาตรฐาน"""
    page_size = 20                    # items ต่อหน้า
    page_size_query_param = 'page_size'  # client กำหนด page_size ได้
    max_page_size = 100               # สูงสุดที่ client กำหนดได้
    page_query_param = 'page'         # ?page=2
    
    def get_paginated_response(self, data):
        """Custom response format"""
        return Response({
            'pagination': {
                'count': self.page.paginator.count,
                'total_pages': self.page.paginator.num_pages,
                'current_page': self.page.number,
                'page_size': self.get_page_size(self.request),
                'next': self.get_next_link(),
                'previous': self.get_previous_link(),
            },
            'results': data
        })


class SmallPagination(PageNumberPagination):
    page_size = 5
    max_page_size = 20


class LargePagination(PageNumberPagination):
    page_size = 50
    max_page_size = 200
```

```python
# settings.py - กำหนด default pagination
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'myapp.pagination.StandardPagination',
    'PAGE_SIZE': 20,
}
```

```python
# ใช้ใน ViewSet
class ArticleViewSet(viewsets.ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer
    pagination_class = StandardPagination
    
    # ปิด pagination สำหรับ ViewSet นี้
    # pagination_class = None
```

### Response Format

```json
{
    "pagination": {
        "count": 150,
        "total_pages": 8,
        "current_page": 1,
        "page_size": 20,
        "next": "http://api.example.com/articles/?page=2",
        "previous": null
    },
    "results": [
        {"id": 1, "title": "..."},
        ...
    ]
}
```

```bash
# ใช้งาน
GET /api/articles/                    # หน้าที่ 1
GET /api/articles/?page=2             # หน้าที่ 2
GET /api/articles/?page=2&page_size=5 # หน้าที่ 2 แสดง 5 items
```

---

## 6. LimitOffsetPagination

```python
from rest_framework.pagination import LimitOffsetPagination

class ArticleLimitOffsetPagination(LimitOffsetPagination):
    default_limit = 20
    limit_query_param = 'limit'   # จำนวน items
    offset_query_param = 'offset' # เริ่มจากตำแหน่งไหน
    max_limit = 100

# ใช้งาน:
# GET /api/articles/?limit=10&offset=0   - 10 items แรก
# GET /api/articles/?limit=10&offset=10  - items 11-20
# GET /api/articles/?limit=10&offset=20  - items 21-30
```

---

## 7. CursorPagination

เหมาะสำหรับ large datasets หรือ real-time data

```python
from rest_framework.pagination import CursorPagination

class ArticleCursorPagination(CursorPagination):
    """Cursor-based pagination - เหมาะกับ real-time feeds"""
    page_size = 20
    cursor_query_param = 'cursor'
    ordering = '-created_at'  # ต้องระบุ ordering
    
    # ข้อดีของ cursor pagination:
    # - Consistent results แม้มีข้อมูลใหม่เพิ่ม
    # - ไม่มีปัญหา duplicate/missing items
    # - เหมาะกับ infinite scroll
    
    # ข้อเสีย:
    # - ไม่รู้จำนวนหน้าทั้งหมด
    # - ไม่สามารถ jump ไปหน้าที่ต้องการได้

# Response:
# {
#   "next": "http://api.example.com/articles/?cursor=cD0yMDIz...",
#   "previous": null,
#   "results": [...]
# }
```

---

## 8. ตัวอย่างครบ

```python
# views.py
from rest_framework import viewsets, filters
from django_filters.rest_framework import DjangoFilterBackend
from .filters import ArticleFilter
from .pagination import StandardPagination

class ArticleViewSet(viewsets.ModelViewSet):
    """
    API endpoint สำหรับบทความ
    
    Filtering:
    - ?status=published
    - ?category_slug=python
    - ?created_after=2024-01-01
    - ?min_views=100
    
    Search:
    - ?search=django  (ค้นใน title, content)
    
    Ordering:
    - ?ordering=-created_at  (ใหม่ก่อน)
    - ?ordering=views_count  (น้อยไปมาก)
    
    Pagination:
    - ?page=2
    - ?page_size=10
    """
    queryset = Article.objects.select_related(
        'author', 'category'
    ).prefetch_related('tags').all()
    serializer_class = ArticleSerializer
    pagination_class = StandardPagination
    
    filter_backends = [
        DjangoFilterBackend,
        filters.SearchFilter,
        filters.OrderingFilter,
    ]
    
    filterset_class = ArticleFilter
    search_fields = ['title', 'content', 'author__username']
    ordering_fields = ['title', 'created_at', 'views_count']
    ordering = ['-created_at']
    
    def get_queryset(self):
        queryset = super().get_queryset()
        
        # กรองตาม tag
        tag = self.request.query_params.get('tag')
        if tag:
            queryset = queryset.filter(tags__slug=tag)
        
        return queryset
```

---

## 9. สรุป Part 069

✅ **SearchFilter** ค้นหาข้อมูลใน fields ที่กำหนด ด้วย `?search=keyword`
✅ **OrderingFilter** เรียงข้อมูล ด้วย `?ordering=field` หรือ `?ordering=-field`
✅ **django-filter** สร้าง FilterSet ที่ยืดหยุ่นสูง
✅ **Custom Filter Backend** สร้าง filter พิเศษตาม business logic
✅ **PageNumberPagination** แบ่งหน้าด้วย ?page=N
✅ **LimitOffsetPagination** ใช้ ?limit=N&offset=M
✅ **CursorPagination** เหมาะกับ real-time feeds ขนาดใหญ่

## ➡️ ถัดไป: Part 070 - Django Signals

*Part 069/100+ | Python Course - Beginner to World-Class*
