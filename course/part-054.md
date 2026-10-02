# Part 54: Django Views (Function-Based และ Class-Based)

## เป้าหมายของบทเรียน

- เขียน Function-Based Views (FBV)
- เขียน Class-Based Views (CBV)
- ใช้ Generic Views (ListView, DetailView, CreateView, UpdateView, DeleteView)
- เข้าใจ HttpRequest และ HttpResponse
- ส่ง context ไปยัง templates
- ใช้ decorators และ mixins

---

## 1. View คืออะไร?

View เป็น Python function (หรือ class) ที่รับ `HttpRequest` object และส่งกลับ `HttpResponse` object

```python
# รูปแบบพื้นฐาน
from django.http import HttpResponse

def my_view(request):
    # request = HttpRequest object
    # ทำการประมวลผล...
    return HttpResponse("Hello!")
    # คืน HttpResponse object
```

---

## 2. HttpRequest Object

```python
from django.http import HttpResponse


def request_info(request):
    """
    แสดงข้อมูล HttpRequest
    """
    info = {
        # Method
        'method': request.method,           # 'GET', 'POST', 'PUT', etc.
        
        # URL
        'path': request.path,               # '/blog/posts/'
        'full_path': request.get_full_path(), # '/blog/posts/?page=2'
        
        # GET parameters
        'get_params': dict(request.GET),    # {'page': ['2'], 'q': ['django']}
        
        # POST data
        'post_data': dict(request.POST),    # form data
        
        # Files
        'files': dict(request.FILES),       # uploaded files
        
        # Headers
        'content_type': request.content_type,
        'user_agent': request.META.get('HTTP_USER_AGENT', ''),
        
        # User
        'user': str(request.user),          # 'AnonymousUser' หรือ username
        'is_authenticated': request.user.is_authenticated,
        
        # Session
        # 'session': dict(request.session), # session data
        
        # Cookies
        'cookies': dict(request.COOKIES),
    }
    
    # ตรวจสอบ method
    if request.method == 'GET':
        search_query = request.GET.get('q', '')
        page = request.GET.get('page', 1)
    
    return HttpResponse(str(info))
```

---

## 3. HttpResponse ประเภทต่างๆ

```python
from django.http import (
    HttpResponse,
    HttpResponseRedirect,
    HttpResponseNotFound,
    HttpResponseForbidden,
    HttpResponseBadRequest,
    HttpResponseServerError,
    JsonResponse,
    FileResponse,
    StreamingHttpResponse,
)
from django.shortcuts import render, redirect, get_object_or_404


def response_examples(request):
    # HttpResponse ธรรมดา
    response = HttpResponse("Hello World")
    response = HttpResponse("Hello", status=200)
    response = HttpResponse("Not Found", status=404)
    
    # กำหนด Content-Type
    response = HttpResponse(
        content='<html><body>Hello</body></html>',
        content_type='text/html; charset=utf-8',
        status=200,
    )
    
    # Set headers
    response['X-Custom-Header'] = 'value'
    response['Cache-Control'] = 'no-cache'
    
    # HttpResponseRedirect
    return HttpResponseRedirect('/new-url/')
    # หรือใช้ shortcut
    return redirect('/new-url/')
    return redirect('blog:post_list')  # ใช้ URL name
    
    # Response ด้วย status code
    return HttpResponseNotFound('<h1>หน้านี้ไม่มี</h1>')
    return HttpResponseForbidden('คุณไม่มีสิทธิ์')
    return HttpResponseBadRequest('ข้อมูลไม่ถูกต้อง')
    
    # JsonResponse
    data = {'status': 'success', 'message': 'ทำงานสำเร็จ'}
    return JsonResponse(data)
    return JsonResponse(data, status=201)
    return JsonResponse({'error': 'not found'}, status=404)
    
    # JsonResponse สำหรับ list
    return JsonResponse(data, safe=False)  # safe=False สำหรับ non-dict
    
    # FileResponse
    import os
    file_path = '/path/to/file.pdf'
    return FileResponse(open(file_path, 'rb'), content_type='application/pdf')
```

---

## 4. Function-Based Views (FBV)

### FBV พื้นฐาน

```python
# blog/views.py
from django.shortcuts import render, get_object_or_404, redirect
from django.http import HttpResponse, JsonResponse
from django.contrib.auth.decorators import login_required
from django.views.decorators.http import require_http_methods, require_GET, require_POST
from django.contrib import messages
from django.core.paginator import Paginator

from .models import Post, Category, Comment
from .forms import PostForm, CommentForm


def post_list(request):
    """
    แสดงรายการบทความ
    รองรับ search และ pagination
    """
    # ดึงทุก posts ที่ published
    posts = Post.objects.filter(
        status='published'
    ).select_related('author', 'category').prefetch_related('tags')
    
    # Search
    search_query = request.GET.get('q', '').strip()
    if search_query:
        posts = posts.filter(title__icontains=search_query)
    
    # Filter by category
    category_slug = request.GET.get('category', '')
    if category_slug:
        posts = posts.filter(category__slug=category_slug)
    
    # Pagination
    paginator = Paginator(posts, 10)  # 10 posts ต่อหน้า
    page_number = request.GET.get('page', 1)
    page_obj = paginator.get_page(page_number)
    
    # ดึง categories สำหรับ filter
    categories = Category.objects.all()
    
    context = {
        'page_obj': page_obj,
        'posts': page_obj,  # alias
        'categories': categories,
        'search_query': search_query,
        'selected_category': category_slug,
        'total_count': posts.count(),
    }
    
    return render(request, 'blog/post_list.html', context)


def post_detail(request, slug):
    """
    แสดงรายละเอียดบทความ
    """
    # ดึง post หรือ 404
    post = get_object_or_404(
        Post.objects.select_related('author', 'category').prefetch_related('tags'),
        slug=slug,
        status='published'
    )
    
    # เพิ่ม views count
    post.increment_views()
    
    # ดึง comments
    comments = post.comments.filter(
        is_approved=True,
        parent__isnull=True  # แสดงเฉพาะ top-level comments
    ).select_related('author').prefetch_related('replies__author')
    
    # Comment form
    comment_form = CommentForm()
    
    # บทความที่เกี่ยวข้อง
    related_posts = Post.objects.filter(
        category=post.category,
        status='published'
    ).exclude(pk=post.pk)[:3]
    
    context = {
        'post': post,
        'comments': comments,
        'comment_count': comments.count(),
        'comment_form': comment_form,
        'related_posts': related_posts,
    }
    
    return render(request, 'blog/post_detail.html', context)


@login_required
def post_create(request):
    """
    สร้างบทความใหม่ (ต้อง login)
    จัดการทั้ง GET (แสดงฟอร์ม) และ POST (บันทึกข้อมูล)
    """
    if request.method == 'POST':
        form = PostForm(request.POST, request.FILES)
        if form.is_valid():
            # สร้าง instance แต่ยังไม่ save
            post = form.save(commit=False)
            # กำหนด author
            post.author = request.user
            post.save()
            # save ManyToMany fields
            form.save_m2m()
            
            messages.success(request, f'สร้างบทความ "{post.title}" สำเร็จ!')
            return redirect('blog:post_detail', slug=post.slug)
        else:
            messages.error(request, 'กรุณาตรวจสอบข้อมูลที่กรอก')
    else:
        form = PostForm()
    
    return render(request, 'blog/post_form.html', {
        'form': form,
        'action': 'สร้าง',
        'title': 'สร้างบทความใหม่',
    })


@login_required
def post_update(request, slug):
    """
    แก้ไขบทความ (เฉพาะเจ้าของ)
    """
    post = get_object_or_404(Post, slug=slug, author=request.user)
    
    if request.method == 'POST':
        form = PostForm(request.POST, request.FILES, instance=post)
        if form.is_valid():
            form.save()
            messages.success(request, 'อัปเดตบทความสำเร็จ!')
            return redirect('blog:post_detail', slug=post.slug)
    else:
        form = PostForm(instance=post)
    
    return render(request, 'blog/post_form.html', {
        'form': form,
        'post': post,
        'action': 'แก้ไข',
        'title': f'แก้ไข: {post.title}',
    })


@login_required
def post_delete(request, slug):
    """
    ลบบทความ (เฉพาะเจ้าของ)
    ต้อง confirm ก่อน
    """
    post = get_object_or_404(Post, slug=slug, author=request.user)
    
    if request.method == 'POST':
        title = post.title
        post.delete()
        messages.success(request, f'ลบบทความ "{title}" สำเร็จ!')
        return redirect('blog:post_list')
    
    return render(request, 'blog/post_confirm_delete.html', {'post': post})


@login_required
@require_POST  # รับเฉพาะ POST method
def add_comment(request, post_slug):
    """
    เพิ่มความคิดเห็น
    """
    post = get_object_or_404(Post, slug=post_slug, status='published')
    form = CommentForm(request.POST)
    
    if form.is_valid():
        comment = form.save(commit=False)
        comment.post = post
        comment.author = request.user
        
        # ถ้าเป็นการ reply
        parent_id = request.POST.get('parent_id')
        if parent_id:
            comment.parent = get_object_or_404(Comment, pk=parent_id)
        
        comment.save()
        messages.success(request, 'เพิ่มความคิดเห็นสำเร็จ!')
    else:
        messages.error(request, 'กรุณาตรวจสอบข้อมูล')
    
    return redirect('blog:post_detail', slug=post_slug)


@require_GET  # รับเฉพาะ GET method
def post_list_api(request):
    """
    API endpoint สำหรับดึงรายการ posts (JSON)
    """
    posts = Post.objects.filter(
        status='published'
    ).values('id', 'title', 'slug', 'created_at', 'views_count')[:20]
    
    return JsonResponse({
        'posts': list(posts),
        'count': posts.count(),
    })
```

### Decorators

```python
from django.contrib.auth.decorators import login_required, permission_required
from django.views.decorators.http import require_http_methods
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie
from functools import wraps


# login_required - redirect ถ้าไม่ได้ login
@login_required
def my_view(request):
    pass

# กำหนด login URL
@login_required(login_url='/accounts/login/')
def my_view(request):
    pass

# permission_required - ต้องมี permission
@permission_required('blog.add_post')
def create_post(request):
    pass

# require_http_methods - กำหนด methods ที่ยอมรับ
@require_http_methods(['GET', 'POST'])
def my_view(request):
    pass

# cache_page - cache response
@cache_page(60 * 15)  # cache 15 นาที
def my_view(request):
    pass

# Custom decorator
def owner_required(view_func):
    """Decorator ที่ตรวจสอบว่าเป็นเจ้าของ post หรือไม่"""
    @wraps(view_func)
    def _wrapped_view(request, *args, **kwargs):
        post = get_object_or_404(Post, slug=kwargs.get('slug'))
        if post.author != request.user and not request.user.is_staff:
            from django.http import HttpResponseForbidden
            return HttpResponseForbidden('คุณไม่มีสิทธิ์ดำเนินการนี้')
        return view_func(request, *args, **kwargs)
    return _wrapped_view


@login_required
@owner_required
def edit_post(request, slug):
    pass
```

---

## 5. Class-Based Views (CBV)

### CBV พื้นฐาน

```python
from django.views import View
from django.http import HttpResponse, JsonResponse
from django.shortcuts import render, redirect, get_object_or_404


class PostListView(View):
    """
    Class-Based View สำหรับแสดงรายการ posts
    แยก logic ของ GET และ POST เป็น method
    """
    template_name = 'blog/post_list.html'
    
    def get(self, request, *args, **kwargs):
        """จัดการ GET request"""
        posts = Post.objects.filter(status='published')
        return render(request, self.template_name, {'posts': posts})
    
    def post(self, request, *args, **kwargs):
        """จัดการ POST request"""
        # ทำอะไรบางอย่างกับ POST data
        return redirect('blog:post_list')


class PostDetailView(View):
    """
    Class-Based View สำหรับแสดงรายละเอียด post
    """
    def get(self, request, slug):
        post = get_object_or_404(Post, slug=slug, status='published')
        context = {
            'post': post,
            'comments': post.comments.filter(is_approved=True),
        }
        return render(request, 'blog/post_detail.html', context)


# urls.py
# path('posts/', PostListView.as_view(), name='post_list'),
# path('posts/<slug:slug>/', PostDetailView.as_view(), name='post_detail'),
```

### CBV กับ Mixins

```python
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.views import View


class LoginRequired(LoginRequiredMixin):
    """Mixin ที่บังคับ login"""
    login_url = '/accounts/login/'
    raise_exception = False  # redirect แทน 403


class CreatePostView(LoginRequiredMixin, View):
    """
    View สร้างบทความ - ต้อง login
    """
    login_url = '/login/'
    template_name = 'blog/post_form.html'
    
    def get(self, request):
        form = PostForm()
        return render(request, self.template_name, {'form': form})
    
    def post(self, request):
        form = PostForm(request.POST, request.FILES)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            form.save_m2m()
            return redirect('blog:post_detail', slug=post.slug)
        return render(request, self.template_name, {'form': form})
```

---

## 6. Generic Class-Based Views

### ListView

```python
from django.views.generic import (
    ListView, DetailView, CreateView, UpdateView, DeleteView
)
from django.urls import reverse_lazy


class PostListView(ListView):
    """
    แสดงรายการ posts
    """
    model = Post
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'    # ชื่อ variable ใน template (default: object_list)
    paginate_by = 10                  # pagination
    ordering = ['-created_at']        # การเรียงลำดับ
    
    def get_queryset(self):
        """
        Override เพื่อกรอง queryset
        """
        queryset = super().get_queryset()
        # กรองเฉพาะ published
        queryset = queryset.filter(status='published')
        
        # Search
        search = self.request.GET.get('q', '')
        if search:
            queryset = queryset.filter(title__icontains=search)
        
        # Filter by category
        category = self.request.GET.get('category', '')
        if category:
            queryset = queryset.filter(category__slug=category)
        
        return queryset.select_related('author', 'category')
    
    def get_context_data(self, **kwargs):
        """
        Override เพื่อเพิ่ม context
        """
        context = super().get_context_data(**kwargs)
        # เพิ่ม data พิเศษ
        context['categories'] = Category.objects.all()
        context['search_query'] = self.request.GET.get('q', '')
        context['featured_posts'] = Post.objects.filter(
            is_featured=True,
            status='published'
        )[:3]
        return context
```

### DetailView

```python
class PostDetailView(DetailView):
    """
    แสดงรายละเอียด post
    """
    model = Post
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'
    slug_field = 'slug'              # field ที่ใช้ lookup (default: slug)
    slug_url_kwarg = 'slug'          # URL kwarg name (default: slug)
    
    def get_queryset(self):
        """กรองเฉพาะ published posts"""
        return super().get_queryset().filter(
            status='published'
        ).select_related('author', 'category').prefetch_related('tags')
    
    def get_object(self, queryset=None):
        """Override เพื่อเพิ่ม views count"""
        obj = super().get_object(queryset)
        obj.increment_views()
        return obj
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        post = self.get_object()
        context['comments'] = post.comments.filter(
            is_approved=True,
            parent__isnull=True
        ).select_related('author')
        context['comment_form'] = CommentForm()
        context['related_posts'] = Post.objects.filter(
            category=post.category,
            status='published'
        ).exclude(pk=post.pk)[:3]
        return context
```

### CreateView

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.contrib import messages


class PostCreateView(LoginRequiredMixin, CreateView):
    """
    สร้างบทความใหม่
    """
    model = Post
    form_class = PostForm
    template_name = 'blog/post_form.html'
    
    def form_valid(self, form):
        """
        ทำงานเมื่อ form valid
        กำหนด author ก่อน save
        """
        form.instance.author = self.request.user
        response = super().form_valid(form)
        messages.success(
            self.request,
            f'สร้างบทความ "{form.instance.title}" สำเร็จ!'
        )
        return response
    
    def form_invalid(self, form):
        """ทำงานเมื่อ form invalid"""
        messages.error(self.request, 'กรุณาตรวจสอบข้อมูล')
        return super().form_invalid(form)
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['title'] = 'สร้างบทความใหม่'
        context['action'] = 'สร้าง'
        return context
    
    def get_success_url(self):
        """URL หลังจาก success"""
        return self.object.get_absolute_url()
```

### UpdateView

```python
from django.core.exceptions import PermissionDenied


class PostUpdateView(LoginRequiredMixin, UpdateView):
    """
    แก้ไขบทความ (เฉพาะเจ้าของ)
    """
    model = Post
    form_class = PostForm
    template_name = 'blog/post_form.html'
    slug_field = 'slug'
    
    def get_queryset(self):
        """กรองเฉพาะ posts ของ user ที่ login"""
        queryset = super().get_queryset()
        if not self.request.user.is_staff:
            queryset = queryset.filter(author=self.request.user)
        return queryset
    
    def form_valid(self, form):
        response = super().form_valid(form)
        messages.success(self.request, 'อัปเดตบทความสำเร็จ!')
        return response
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['title'] = f'แก้ไข: {self.object.title}'
        context['action'] = 'แก้ไข'
        return context
    
    def get_success_url(self):
        return self.object.get_absolute_url()
```

### DeleteView

```python
class PostDeleteView(LoginRequiredMixin, DeleteView):
    """
    ลบบทความ (เฉพาะเจ้าของ)
    """
    model = Post
    template_name = 'blog/post_confirm_delete.html'
    success_url = reverse_lazy('blog:post_list')  # ใช้ reverse_lazy แทน reverse
    
    def get_queryset(self):
        """กรองเฉพาะ posts ของ user"""
        queryset = super().get_queryset()
        if not self.request.user.is_staff:
            queryset = queryset.filter(author=self.request.user)
        return queryset
    
    def form_valid(self, form):
        post_title = self.object.title
        response = super().form_valid(form)
        messages.success(self.request, f'ลบบทความ "{post_title}" สำเร็จ!')
        return response
```

---

## 7. URLs Configuration สำหรับ CBV

```python
# blog/urls.py
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    # FBV
    path('', views.post_list, name='post_list'),
    path('<slug:slug>/', views.post_detail, name='post_detail'),
    
    # CBV - ต้องใช้ .as_view()
    path('cbv/', views.PostListView.as_view(), name='post_list_cbv'),
    path('cbv/<slug:slug>/', views.PostDetailView.as_view(), name='post_detail_cbv'),
    path('new/', views.PostCreateView.as_view(), name='post_create'),
    path('<slug:slug>/edit/', views.PostUpdateView.as_view(), name='post_update'),
    path('<slug:slug>/delete/', views.PostDeleteView.as_view(), name='post_delete'),
    
    # Comments
    path('<slug:post_slug>/comment/', views.add_comment, name='add_comment'),
    
    # API
    path('api/posts/', views.post_list_api, name='post_list_api'),
]
```

---

## 8. Mixins สำหรับ CBV

### Custom Mixins

```python
# blog/mixins.py
from django.contrib.auth.mixins import LoginRequiredMixin
from django.core.exceptions import PermissionDenied
from django.contrib import messages


class OwnerRequiredMixin:
    """
    Mixin ที่ตรวจสอบว่าเป็นเจ้าของ object หรือไม่
    """
    def get_object(self, queryset=None):
        obj = super().get_object(queryset)
        if obj.author != self.request.user and not self.request.user.is_staff:
            raise PermissionDenied('คุณไม่มีสิทธิ์ดำเนินการนี้')
        return obj


class SuccessMessageMixin:
    """
    Mixin สำหรับแสดง success message
    """
    success_message = ''
    
    def form_valid(self, form):
        response = super().form_valid(form)
        if self.success_message:
            messages.success(self.request, self.success_message)
        return response


class AjaxResponseMixin:
    """
    Mixin สำหรับ JSON response เมื่อเป็น AJAX request
    """
    def form_invalid(self, form):
        if self.request.headers.get('X-Requested-With') == 'XMLHttpRequest':
            from django.http import JsonResponse
            return JsonResponse({'errors': form.errors}, status=400)
        return super().form_invalid(form)
    
    def form_valid(self, form):
        if self.request.headers.get('X-Requested-With') == 'XMLHttpRequest':
            from django.http import JsonResponse
            super().form_valid(form)
            return JsonResponse({'success': True, 'url': self.get_success_url()})
        return super().form_valid(form)


# ใช้งาน
class PostUpdateView(LoginRequiredMixin, OwnerRequiredMixin, SuccessMessageMixin, UpdateView):
    model = Post
    form_class = PostForm
    success_message = 'อัปเดตบทความสำเร็จ!'
    template_name = 'blog/post_form.html'
```

---

## 9. TemplateView และ RedirectView

```python
from django.views.generic import TemplateView, RedirectView


class HomeView(TemplateView):
    """
    แสดง template โดยไม่ต้องมี model
    """
    template_name = 'home.html'
    
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['recent_posts'] = Post.objects.filter(
            status='published'
        )[:5]
        context['categories'] = Category.objects.annotate(
            post_count=Count('posts')
        )
        return context


class OldBlogRedirect(RedirectView):
    """
    Redirect จาก URL เก่าไปใหม่
    """
    # กำหนด URL โดยตรง
    url = '/blog/posts/'
    
    # หรือกำหนดด้วย URL name
    pattern_name = 'blog:post_list'
    
    # permanent redirect (301) หรือ temporary (302)
    permanent = True
    
    # Query string จะถูก forward ไปด้วย
    query_string = True
```

---

## 10. View ที่จัดการหลาย Scenarios

```python
class PostManageView(LoginRequiredMixin, View):
    """
    View ที่จัดการทั้ง list, create, update, delete
    ผ่าน URL parameters
    """
    
    def get(self, request, action=None, slug=None):
        """จัดการ GET requests"""
        if action == 'create':
            return self._show_create_form(request)
        elif action == 'edit' and slug:
            return self._show_edit_form(request, slug)
        elif action == 'delete' and slug:
            return self._show_delete_confirm(request, slug)
        else:
            return self._show_list(request)
    
    def post(self, request, action=None, slug=None):
        """จัดการ POST requests"""
        if action == 'create':
            return self._handle_create(request)
        elif action == 'edit' and slug:
            return self._handle_update(request, slug)
        elif action == 'delete' and slug:
            return self._handle_delete(request, slug)
        return redirect('blog:post_list')
    
    def _show_list(self, request):
        posts = Post.objects.filter(author=request.user)
        return render(request, 'blog/manage/list.html', {'posts': posts})
    
    def _show_create_form(self, request):
        form = PostForm()
        return render(request, 'blog/manage/form.html', {
            'form': form,
            'action': 'create',
        })
    
    def _show_edit_form(self, request, slug):
        post = get_object_or_404(Post, slug=slug, author=request.user)
        form = PostForm(instance=post)
        return render(request, 'blog/manage/form.html', {
            'form': form,
            'post': post,
            'action': 'edit',
        })
    
    def _show_delete_confirm(self, request, slug):
        post = get_object_or_404(Post, slug=slug, author=request.user)
        return render(request, 'blog/manage/delete.html', {'post': post})
    
    def _handle_create(self, request):
        form = PostForm(request.POST, request.FILES)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            form.save_m2m()
            messages.success(request, 'สร้างบทความสำเร็จ!')
            return redirect('blog:post_list')
        return render(request, 'blog/manage/form.html', {
            'form': form,
            'action': 'create',
        })
    
    def _handle_update(self, request, slug):
        post = get_object_or_404(Post, slug=slug, author=request.user)
        form = PostForm(request.POST, request.FILES, instance=post)
        if form.is_valid():
            form.save()
            messages.success(request, 'อัปเดตบทความสำเร็จ!')
            return redirect('blog:post_detail', slug=post.slug)
        return render(request, 'blog/manage/form.html', {
            'form': form,
            'post': post,
            'action': 'edit',
        })
    
    def _handle_delete(self, request, slug):
        post = get_object_or_404(Post, slug=slug, author=request.user)
        title = post.title
        post.delete()
        messages.success(request, f'ลบบทความ "{title}" สำเร็จ!')
        return redirect('blog:post_list')
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: FBV สำหรับ E-commerce
สร้าง FBV สำหรับ product:
- `product_list` - แสดงรายการสินค้า, search, filter by category, pagination
- `product_detail` - แสดงรายละเอียดสินค้า
- `add_to_cart` - เพิ่มสินค้าลงตะกร้า (POST only, login required)

### แบบฝึกหัดที่ 2: CBV CRUD
สร้าง CBV ครบทุก CRUD operation สำหรับ `Comment` model:
- `CommentListView`
- `CommentCreateView`
- `CommentUpdateView`
- `CommentDeleteView`

### แบบฝึกหัดที่ 3: Custom Mixin
สร้าง `StaffRequiredMixin` ที่:
- ตรวจสอบว่า user เป็น staff หรือ admin
- ถ้าไม่ใช่ redirect ไปหน้า 403
- ใช้งานกับ CBV ใดก็ได้

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- FBV พื้นฐานและการใช้ decorators
- CBV และ Generic Views ทุกประเภท
- HttpRequest และ HttpResponse objects
- Pagination ด้วย Paginator
- Mixins สำหรับ reusable functionality
- การส่ง context และ render templates

---

## บทถัดไป

➡️ **[Part 55: Django Templates](part-055.md)** - เรียนรู้ Template syntax, tags, filters, และ inheritance
