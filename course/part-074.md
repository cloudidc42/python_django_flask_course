# Part 074: Django Testing
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- เข้าใจความแตกต่างระหว่าง Django TestCase และ unittest.TestCase
- ใช้ setUp และ tearDown เพื่อเตรียมข้อมูลสำหรับการทดสอบ
- ทดสอบ Views, Models, และ Forms ใน Django ได้
- ใช้ Django Test Client สำหรับทดสอบ HTTP requests
- ทดสอบ REST API ด้วย DRF APITestCase
- ใช้ factory_boy สร้างข้อมูลทดสอบ
- ตั้งค่า pytest-django สำหรับการทดสอบด้วย pytest
- จัดการ Test Database ได้อย่างถูกต้อง
- Mock external services ในการทดสอบ
- วัด Code Coverage ด้วย pytest-cov
- สร้าง Complete Test Suite สำหรับ Blog Application

---

## 1. ทำไมต้องเขียน Tests?

การทดสอบ (Testing) เป็นส่วนสำคัญของการพัฒนา Software ที่มีคุณภาพ:

```
✅ ป้องกัน Regression Bugs - โค้ดเก่าไม่พังเมื่อเพิ่มโค้ดใหม่
✅ เอกสารประกอบโค้ด       - Tests บอกว่าโค้ดควรทำงานอย่างไร
✅ ออกแบบโค้ดดีขึ้น       - Code ที่ Test ได้ มักออกแบบดีกว่า
✅ Refactor อย่างมั่นใจ   - เปลี่ยนโค้ดได้โดยไม่กลัวพัง
✅ CI/CD Pipeline          - ตรวจสอบอัตโนมัติก่อน Deploy
```

### ประเภทของ Tests

```
Unit Tests       → ทดสอบ Function/Method เดียว
Integration Tests → ทดสอบการทำงานร่วมกันของหลาย Components
End-to-End Tests  → ทดสอบ User Journey ทั้งหมด
```

---

## 2. Django TestCase vs unittest.TestCase

Django มี TestCase class หลายตัวที่ขยายมาจาก Python unittest:

```python
# myapp/tests/test_basics.py

# Python standard library unittest
import unittest

# Django TestCase - สืบทอดจาก unittest.TestCase
from django.test import TestCase

# Django TransactionTestCase - สำหรับทดสอบ Database Transactions
from django.test import TransactionTestCase

# Django SimpleTestCase - ไม่ใช้ Database
from django.test import SimpleTestCase

# Django LiveServerTestCase - สำหรับ Selenium/Browser tests
from django.test import LiveServerTestCase
```

### ความแตกต่างหลัก

```python
# unittest.TestCase - ไม่รู้จัก Django
class PureUnittestExample(unittest.TestCase):
    def test_basic_math(self):
        # ทดสอบ Python ล้วนๆ ไม่เกี่ยวกับ Django
        self.assertEqual(2 + 2, 4)
        self.assertTrue(isinstance("hello", str))
        self.assertIn("h", "hello")


# django.test.SimpleTestCase - รู้จัก Django แต่ไม่ใช้ Database
from django.test import SimpleTestCase
from django.urls import reverse

class SimpleViewTest(SimpleTestCase):
    def test_homepage_url_resolves(self):
        # ทดสอบ URL routing โดยไม่ต้องใช้ Database
        url = reverse('home')
        self.assertEqual(url, '/')

    def test_template_rendering(self):
        # ทดสอบ Template rendering
        response = self.client.get('/')
        self.assertTemplateUsed(response, 'home.html')


# django.test.TestCase - รู้จัก Django และใช้ Database
from django.test import TestCase
from myapp.models import Post, User

class PostModelTest(TestCase):
    # Django จะสร้าง Test Database แยกต่างหาก
    # และ Rollback ทุก Transaction หลังแต่ละ Test

    def setUp(self):
        # สร้างข้อมูลก่อนแต่ละ Test
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )

    def test_create_post(self):
        post = Post.objects.create(
            title='Test Post',
            content='Test Content',
            author=self.user
        )
        # ตรวจสอบว่า Post ถูกสร้างจริง
        self.assertEqual(Post.objects.count(), 1)
        self.assertEqual(post.title, 'Test Post')

    def tearDown(self):
        # Django TestCase ทำ Rollback อัตโนมัติ
        # แต่เราสามารถเพิ่ม cleanup เพิ่มเติมได้
        pass
```

### การเลือกใช้ TestCase ที่เหมาะสม

```python
# ตัดสินใจเลือก TestCase

# SimpleTestCase → ใช้เมื่อ:
# - ทดสอบ URL patterns
# - ทดสอบ Template rendering ง่ายๆ
# - ไม่ต้องการ Database

# TestCase → ใช้เมื่อ:
# - ทดสอบ Models
# - ทดสอบ Views ที่ใช้ Database
# - ทดสอบ Forms
# - ใช้บ่อยที่สุด!

# TransactionTestCase → ใช้เมื่อ:
# - ทดสอบ Database Signals
# - ทดสอบ select_for_update()
# - ทดสอบ Raw SQL Transactions

# LiveServerTestCase → ใช้เมื่อ:
# - ทดสอบด้วย Selenium
# - ทดสอบ JavaScript interaction
# - End-to-End Testing
```

---

## 3. setUp และ tearDown

```python
# myapp/tests/test_setup_teardown.py
from django.test import TestCase
from django.contrib.auth.models import User
from myapp.models import Post, Category, Tag

class TestLifecycle(TestCase):
    """ตัวอย่างการใช้ setUp และ tearDown"""

    @classmethod
    def setUpClass(cls):
        """
        เรียกครั้งเดียวก่อน Tests ทั้งหมดใน Class นี้
        ใช้สำหรับ Setup ที่ค่อนข้าง expensive เช่น Connection
        ต้องเรียก super().setUpClass() ด้วย!
        """
        super().setUpClass()
        print("\n=== เริ่มต้น TestLifecycle ===")

    @classmethod
    def tearDownClass(cls):
        """เรียกครั้งเดียวหลัง Tests ทั้งหมดใน Class นี้"""
        super().tearDownClass()
        print("\n=== จบ TestLifecycle ===")

    def setUp(self):
        """
        เรียกก่อนแต่ละ Test Method
        Django TestCase จะ Rollback Database หลังแต่ละ Test
        ดังนั้น setUp จะถูกเรียกใหม่ทุกครั้ง
        """
        # สร้าง User สำหรับทดสอบ
        self.user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123',
            first_name='Test',
            last_name='User'
        )

        # สร้าง Admin User
        self.admin = User.objects.create_superuser(
            username='admin',
            email='admin@example.com',
            password='adminpass123'
        )

        # สร้าง Category
        self.category = Category.objects.create(
            name='Technology',
            slug='technology'
        )

        # สร้าง Post ตัวอย่าง
        self.post = Post.objects.create(
            title='Test Post',
            slug='test-post',
            content='This is test content',
            author=self.user,
            category=self.category,
            status='published'
        )

        # Login ผู้ใช้ (สำหรับ Tests ที่ต้องการ Authentication)
        self.client.login(username='testuser', password='testpass123')

    def tearDown(self):
        """
        เรียกหลังแต่ละ Test Method
        Django TestCase จะทำ Rollback Database อัตโนมัติ
        แต่เราสามารถ cleanup สิ่งอื่นๆ เช่น Files ได้
        """
        import os
        # ลบ Test Files ที่อาจถูกสร้างขึ้นในระหว่าง Test
        test_files = ['test_upload.jpg', 'test_doc.pdf']
        for filename in test_files:
            if os.path.exists(filename):
                os.remove(filename)

    def test_user_created(self):
        """ทดสอบว่า User ถูกสร้างใน setUp"""
        from django.contrib.auth.models import User as DjangoUser
        self.assertEqual(DjangoUser.objects.filter(is_superuser=False).count(), 1)
        self.assertEqual(self.user.username, 'testuser')

    def test_post_belongs_to_user(self):
        """ทดสอบความสัมพันธ์ระหว่าง Post และ User"""
        self.assertEqual(self.post.author, self.user)
        self.assertEqual(self.post.category, self.category)

    def test_post_is_published(self):
        """ทดสอบว่า Post มี status เป็น published"""
        self.assertEqual(self.post.status, 'published')


class TestWithFixtures(TestCase):
    """ใช้ Fixtures แทน setUp"""

    # โหลดข้อมูลจาก Fixture files
    fixtures = ['users.json', 'categories.json', 'posts.json']

    def test_fixtures_loaded(self):
        """ทดสอบว่า Fixtures โหลดได้"""
        from django.contrib.auth.models import User
        # ตรวจสอบว่า Users จาก Fixture ถูกโหลด
        self.assertGreater(User.objects.count(), 0)
```

---

## 4. Django Test Client สำหรับ HTTP Requests

```python
# myapp/tests/test_client.py
from django.test import TestCase, Client
from django.urls import reverse
from django.contrib.auth.models import User
from myapp.models import Post, Category

class TestClientBasics(TestCase):
    """ตัวอย่างการใช้ Django Test Client"""

    def setUp(self):
        self.client = Client()  # สร้าง Test Client
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.category = Category.objects.create(
            name='Tech',
            slug='tech'
        )
        self.post = Post.objects.create(
            title='Hello World',
            slug='hello-world',
            content='Content here',
            author=self.user,
            category=self.category,
            status='published'
        )

    # --- GET Requests ---

    def test_get_homepage(self):
        """ทดสอบ GET request ไปยัง Homepage"""
        response = self.client.get('/')
        # ตรวจสอบ Status Code
        self.assertEqual(response.status_code, 200)

    def test_get_post_list(self):
        """ทดสอบ POST list page"""
        url = reverse('post-list')  # ใช้ reverse() แทน hardcode URL
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)
        # ตรวจสอบว่า Post อยู่ใน Context
        self.assertIn('posts', response.context)
        self.assertQuerySetEqual(
            response.context['posts'],
            Post.objects.filter(status='published'),
            ordered=False
        )

    def test_get_post_detail(self):
        """ทดสอบ POST detail page"""
        url = reverse('post-detail', kwargs={'slug': self.post.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)
        # ตรวจสอบ Content ใน Response
        self.assertContains(response, 'Hello World')
        self.assertContains(response, 'Content here')

    def test_get_nonexistent_post(self):
        """ทดสอบ 404 สำหรับ Post ที่ไม่มี"""
        url = reverse('post-detail', kwargs={'slug': 'nonexistent-post'})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 404)

    # --- POST Requests ---

    def test_post_create_unauthenticated(self):
        """ทดสอบว่า ผู้ใช้ที่ไม่ได้ Login ถูก Redirect ไป Login page"""
        url = reverse('post-create')
        response = self.client.post(url, {
            'title': 'New Post',
            'content': 'New Content',
            'category': self.category.pk,
        })
        # ควร Redirect ไปยัง Login page
        self.assertEqual(response.status_code, 302)
        self.assertRedirects(response, f'/login/?next={url}')

    def test_post_create_authenticated(self):
        """ทดสอบสร้าง Post เมื่อ Login แล้ว"""
        # Login ก่อน
        self.client.login(username='testuser', password='testpass123')
        url = reverse('post-create')
        response = self.client.post(url, {
            'title': 'New Post',
            'slug': 'new-post',
            'content': 'New Content',
            'category': self.category.pk,
            'status': 'published',
        })
        # ตรวจสอบว่าสร้างสำเร็จและ Redirect ไปหน้า Detail
        self.assertEqual(response.status_code, 302)
        self.assertTrue(Post.objects.filter(title='New Post').exists())

    # --- Authentication ---

    def test_login(self):
        """ทดสอบการ Login"""
        response = self.client.post(reverse('login'), {
            'username': 'testuser',
            'password': 'testpass123',
        })
        self.assertEqual(response.status_code, 302)
        # ตรวจสอบว่า Session มี User ID
        self.assertIn('_auth_user_id', self.client.session)

    def test_login_invalid_credentials(self):
        """ทดสอบ Login ด้วย credentials ผิด"""
        response = self.client.post(reverse('login'), {
            'username': 'testuser',
            'password': 'wrongpassword',
        })
        self.assertEqual(response.status_code, 200)  # อยู่หน้าเดิม
        self.assertFormError(response, 'form', None,
                             'Please enter a correct username and password.')

    def test_force_login(self):
        """ใช้ force_login() เพื่อ Login โดยไม่ต้องใช้ Password"""
        self.client.force_login(self.user)
        response = self.client.get(reverse('profile'))
        self.assertEqual(response.status_code, 200)

    # --- Headers และ Content Type ---

    def test_ajax_request(self):
        """ทดสอบ AJAX request"""
        url = reverse('post-list')
        response = self.client.get(
            url,
            HTTP_X_REQUESTED_WITH='XMLHttpRequest'  # จำลอง AJAX
        )
        self.assertEqual(response.status_code, 200)

    def test_json_response(self):
        """ทดสอบ JSON response"""
        url = reverse('api-post-list')
        response = self.client.get(
            url,
            content_type='application/json'
        )
        self.assertEqual(response.status_code, 200)
        # Parse JSON response
        import json
        data = json.loads(response.content)
        self.assertIsInstance(data, list)

    def test_post_with_json(self):
        """ทดสอบส่ง JSON data"""
        import json
        self.client.force_login(self.user)
        url = reverse('api-post-create')
        data = {
            'title': 'JSON Post',
            'content': 'Content',
            'category': self.category.pk,
        }
        response = self.client.post(
            url,
            data=json.dumps(data),
            content_type='application/json'
        )
        self.assertEqual(response.status_code, 201)
```

---

## 5. Testing Models

```python
# myapp/tests/test_models.py
from django.test import TestCase
from django.contrib.auth.models import User
from django.core.exceptions import ValidationError
from django.utils import timezone
from django.db import IntegrityError
import datetime

from myapp.models import Post, Category, Tag, Comment


class CategoryModelTest(TestCase):
    """ทดสอบ Category Model"""

    def setUp(self):
        self.category = Category.objects.create(
            name='Technology',
            slug='technology',
            description='Tech articles'
        )

    def test_str_representation(self):
        """ทดสอบ __str__ method"""
        self.assertEqual(str(self.category), 'Technology')

    def test_slug_auto_generated(self):
        """ทดสอบว่า slug ถูกสร้างอัตโนมัติ"""
        category = Category.objects.create(name='My New Category')
        # ตรวจสอบว่า slug ถูกสร้างจาก name
        self.assertEqual(category.slug, 'my-new-category')

    def test_unique_slug(self):
        """ทดสอบว่า slug ต้อง unique"""
        with self.assertRaises(IntegrityError):
            Category.objects.create(
                name='Another Tech',
                slug='technology'  # slug ซ้ำ
            )

    def test_get_absolute_url(self):
        """ทดสอบ get_absolute_url method"""
        url = self.category.get_absolute_url()
        self.assertEqual(url, '/categories/technology/')


class PostModelTest(TestCase):
    """ทดสอบ Post Model"""

    def setUp(self):
        self.user = User.objects.create_user(
            username='author',
            password='pass123'
        )
        self.category = Category.objects.create(
            name='Tech',
            slug='tech'
        )
        self.post = Post.objects.create(
            title='Test Post',
            slug='test-post',
            content='Test content',
            author=self.user,
            category=self.category,
            status='published'
        )

    def test_post_creation(self):
        """ทดสอบการสร้าง Post"""
        self.assertIsNotNone(self.post.pk)
        self.assertEqual(self.post.title, 'Test Post')
        self.assertEqual(self.post.author, self.user)

    def test_post_ordering(self):
        """ทดสอบ Default Ordering ของ Posts"""
        # สร้าง Post เพิ่มเติม
        post2 = Post.objects.create(
            title='Post 2',
            slug='post-2',
            content='Content 2',
            author=self.user,
            category=self.category,
            status='published'
        )
        posts = Post.objects.all()
        # ตรวจสอบว่า Post ใหม่สุดอยู่ก่อน (ถ้า ordering = ['-created_at'])
        self.assertEqual(posts[0], post2)

    def test_published_manager(self):
        """ทดสอบ Custom Manager สำหรับ Published Posts"""
        # สร้าง Draft Post
        draft = Post.objects.create(
            title='Draft Post',
            slug='draft-post',
            content='Draft content',
            author=self.user,
            category=self.category,
            status='draft'
        )
        # ตรวจสอบว่า Published Manager คืนเฉพาะ Published Posts
        published = Post.published.all()
        self.assertIn(self.post, published)
        self.assertNotIn(draft, published)

    def test_word_count_property(self):
        """ทดสอบ word_count property"""
        post = Post.objects.create(
            title='Word Count Test',
            slug='word-count-test',
            content='This is a test post with ten words here',
            author=self.user,
            category=self.category,
        )
        # "This is a test post with ten words here" = 9 words
        self.assertEqual(post.word_count, 9)

    def test_read_time_property(self):
        """ทดสอบ read_time property (อ่านเฉลี่ย 200 words/min)"""
        # สร้างเนื้อหา 400 words = 2 นาที
        content = ' '.join(['word'] * 400)
        post = Post.objects.create(
            title='Long Post',
            slug='long-post',
            content=content,
            author=self.user,
            category=self.category,
        )
        self.assertEqual(post.read_time, 2)

    def test_post_validation(self):
        """ทดสอบ Model Validation"""
        post = Post(
            title='',  # Title ว่าง
            slug='empty-title',
            content='Content',
            author=self.user,
            category=self.category,
        )
        with self.assertRaises(ValidationError):
            post.full_clean()  # เรียก validation manually

    def test_published_date_set_on_publish(self):
        """ทดสอบว่า published_date ถูกตั้งเมื่อ status เปลี่ยนเป็น published"""
        post = Post.objects.create(
            title='Draft',
            slug='draft',
            content='Content',
            author=self.user,
            category=self.category,
            status='draft'
        )
        self.assertIsNone(post.published_date)

        # เปลี่ยน status เป็น published
        post.status = 'published'
        post.save()
        post.refresh_from_db()

        self.assertIsNotNone(post.published_date)


class CommentModelTest(TestCase):
    """ทดสอบ Comment Model"""

    def setUp(self):
        self.user = User.objects.create_user(
            username='commenter',
            password='pass123'
        )
        self.author = User.objects.create_user(
            username='author',
            password='pass123'
        )
        self.category = Category.objects.create(name='Tech', slug='tech')
        self.post = Post.objects.create(
            title='Post',
            slug='post',
            content='Content',
            author=self.author,
            category=self.category,
            status='published'
        )

    def test_comment_cascade_delete(self):
        """ทดสอบว่า Comment ถูกลบเมื่อ Post ถูกลบ"""
        Comment.objects.create(
            post=self.post,
            author=self.user,
            content='Test comment'
        )
        self.assertEqual(Comment.objects.count(), 1)

        # ลบ Post
        self.post.delete()
        # Comment ควรถูกลบไปด้วย (CASCADE)
        self.assertEqual(Comment.objects.count(), 0)

    def test_nested_comments(self):
        """ทดสอบ Nested Comments (Reply)"""
        parent = Comment.objects.create(
            post=self.post,
            author=self.user,
            content='Parent comment'
        )
        reply = Comment.objects.create(
            post=self.post,
            author=self.user,
            content='Reply comment',
            parent=parent
        )
        self.assertEqual(reply.parent, parent)
        self.assertIn(reply, parent.replies.all())
```

---

## 6. Testing Views

```python
# myapp/tests/test_views.py
from django.test import TestCase
from django.urls import reverse
from django.contrib.auth.models import User
from django.contrib.messages import get_messages
from myapp.models import Post, Category

class PostListViewTest(TestCase):
    """ทดสอบ Post List View"""

    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.category = Category.objects.create(
            name='Tech', slug='tech'
        )
        # สร้าง Posts หลายอัน
        for i in range(15):
            Post.objects.create(
                title=f'Post {i}',
                slug=f'post-{i}',
                content=f'Content {i}',
                author=self.user,
                category=self.category,
                status='published'
            )
        # สร้าง Draft Post (ไม่ควรแสดง)
        Post.objects.create(
            title='Draft',
            slug='draft',
            content='Draft content',
            author=self.user,
            category=self.category,
            status='draft'
        )

    def test_view_url_exists(self):
        """ทดสอบว่า URL มีอยู่"""
        response = self.client.get('/posts/')
        self.assertEqual(response.status_code, 200)

    def test_view_url_accessible_by_name(self):
        """ทดสอบ URL ด้วย name"""
        response = self.client.get(reverse('post-list'))
        self.assertEqual(response.status_code, 200)

    def test_view_uses_correct_template(self):
        """ทดสอบว่า View ใช้ Template ที่ถูกต้อง"""
        response = self.client.get(reverse('post-list'))
        self.assertTemplateUsed(response, 'blog/post_list.html')

    def test_pagination(self):
        """ทดสอบ Pagination"""
        response = self.client.get(reverse('post-list'))
        self.assertTrue('is_paginated' in response.context)
        self.assertTrue(response.context['is_paginated'])
        # ถ้าแต่ละหน้ามี 10 posts ควรมี 2 หน้า
        self.assertEqual(len(response.context['posts']), 10)

    def test_second_page(self):
        """ทดสอบหน้าที่ 2"""
        response = self.client.get(reverse('post-list') + '?page=2')
        self.assertEqual(response.status_code, 200)
        self.assertEqual(len(response.context['posts']), 5)

    def test_only_published_posts_shown(self):
        """ทดสอบว่าแสดงเฉพาะ Published Posts"""
        response = self.client.get(reverse('post-list'))
        posts = response.context['posts']
        for post in posts:
            self.assertEqual(post.status, 'published')


class PostDetailViewTest(TestCase):
    """ทดสอบ Post Detail View"""

    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.category = Category.objects.create(
            name='Tech', slug='tech'
        )
        self.published_post = Post.objects.create(
            title='Published Post',
            slug='published-post',
            content='Published content',
            author=self.user,
            category=self.category,
            status='published'
        )
        self.draft_post = Post.objects.create(
            title='Draft Post',
            slug='draft-post',
            content='Draft content',
            author=self.user,
            category=self.category,
            status='draft'
        )

    def test_published_post_accessible(self):
        """ทดสอบว่า Published Post เข้าถึงได้"""
        url = reverse('post-detail', kwargs={'slug': 'published-post'})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)

    def test_draft_post_not_accessible(self):
        """ทดสอบว่า Draft Post ไม่เข้าถึงได้ (404)"""
        url = reverse('post-detail', kwargs={'slug': 'draft-post'})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 404)

    def test_post_view_count_increases(self):
        """ทดสอบว่า View Count เพิ่มขึ้นเมื่อเข้าชม"""
        initial_views = self.published_post.views
        url = reverse('post-detail', kwargs={'slug': 'published-post'})
        self.client.get(url)
        self.published_post.refresh_from_db()
        self.assertEqual(self.published_post.views, initial_views + 1)


class PostCreateViewTest(TestCase):
    """ทดสอบ Post Create View"""

    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.category = Category.objects.create(
            name='Tech', slug='tech'
        )

    def test_create_view_requires_login(self):
        """ทดสอบว่า Create View ต้อง Login"""
        url = reverse('post-create')
        response = self.client.get(url)
        # ควร Redirect ไป Login
        self.assertEqual(response.status_code, 302)

    def test_create_post_success(self):
        """ทดสอบสร้าง Post สำเร็จ"""
        self.client.login(username='testuser', password='testpass123')
        url = reverse('post-create')
        data = {
            'title': 'New Post',
            'slug': 'new-post',
            'content': 'New content here',
            'category': self.category.pk,
            'status': 'published',
        }
        response = self.client.post(url, data)
        # ตรวจสอบว่า Redirect ไปยัง Post detail
        self.assertEqual(response.status_code, 302)
        # ตรวจสอบว่า Post ถูกสร้าง
        self.assertTrue(Post.objects.filter(title='New Post').exists())
        # ตรวจสอบว่า Author เป็น current user
        post = Post.objects.get(title='New Post')
        self.assertEqual(post.author, self.user)

    def test_create_post_invalid_form(self):
        """ทดสอบส่ง Form ที่ไม่ถูกต้อง"""
        self.client.login(username='testuser', password='testpass123')
        url = reverse('post-create')
        data = {
            'title': '',  # Title ว่าง - ไม่ถูกต้อง
            'content': 'Content',
        }
        response = self.client.post(url, data)
        # ควรอยู่หน้าเดิม (ไม่ Redirect)
        self.assertEqual(response.status_code, 200)
        # ควรมี Form errors
        self.assertFormError(response, 'form', 'title', 'This field is required.')

    def test_success_message_after_create(self):
        """ทดสอบว่ามี Success Message หลังสร้าง Post"""
        self.client.login(username='testuser', password='testpass123')
        url = reverse('post-create')
        data = {
            'title': 'Message Test',
            'slug': 'message-test',
            'content': 'Content',
            'category': self.category.pk,
            'status': 'published',
        }
        response = self.client.post(url, data, follow=True)
        # ดึง Messages จาก Response
        messages = list(get_messages(response.wsgi_request))
        self.assertEqual(len(messages), 1)
        self.assertEqual(str(messages[0]), 'Post created successfully!')
```

---

## 7. Testing Forms

```python
# myapp/tests/test_forms.py
from django.test import TestCase
from django.contrib.auth.models import User
from myapp.forms import PostForm, CommentForm, UserRegistrationForm
from myapp.models import Category


class PostFormTest(TestCase):
    """ทดสอบ Post Form"""

    def setUp(self):
        self.category = Category.objects.create(
            name='Tech', slug='tech'
        )
        # ข้อมูลที่ถูกต้อง
        self.valid_data = {
            'title': 'Test Post',
            'slug': 'test-post',
            'content': 'This is a test post with enough content.',
            'category': self.category.pk,
            'status': 'published',
            'tags': 'python, django',
        }

    def test_form_valid_data(self):
        """ทดสอบ Form ด้วยข้อมูลที่ถูกต้อง"""
        form = PostForm(data=self.valid_data)
        self.assertTrue(form.is_valid())

    def test_form_missing_title(self):
        """ทดสอบ Form ขาด Title"""
        data = self.valid_data.copy()
        data['title'] = ''
        form = PostForm(data=data)
        self.assertFalse(form.is_valid())
        self.assertIn('title', form.errors)

    def test_form_missing_content(self):
        """ทดสอบ Form ขาด Content"""
        data = self.valid_data.copy()
        data['content'] = ''
        form = PostForm(data=data)
        self.assertFalse(form.is_valid())
        self.assertIn('content', form.errors)

    def test_form_invalid_status(self):
        """ทดสอบ Form ด้วย Status ที่ไม่ถูกต้อง"""
        data = self.valid_data.copy()
        data['status'] = 'invalid_status'
        form = PostForm(data=data)
        self.assertFalse(form.is_valid())
        self.assertIn('status', form.errors)

    def test_form_title_too_long(self):
        """ทดสอบ Title ที่ยาวเกินไป"""
        data = self.valid_data.copy()
        data['title'] = 'A' * 300  # เกิน max_length
        form = PostForm(data=data)
        self.assertFalse(form.is_valid())
        self.assertIn('title', form.errors)

    def test_form_slug_auto_generated(self):
        """ทดสอบว่า Slug ถูกสร้างอัตโนมัติเมื่อไม่ได้ระบุ"""
        data = self.valid_data.copy()
        data.pop('slug')  # ไม่ระบุ slug
        form = PostForm(data=data)
        if form.is_valid():
            post = form.save(commit=False)
            # ตรวจสอบว่า slug ถูกสร้างจาก title
            self.assertIsNotNone(post.slug)
            self.assertNotEqual(post.slug, '')

    def test_form_with_file_upload(self):
        """ทดสอบ Form ที่มีการ Upload ไฟล์"""
        import io
        from PIL import Image
        from django.core.files.uploadedfile import SimpleUploadedFile

        # สร้าง Test Image
        img = Image.new('RGB', (100, 100), color='red')
        img_io = io.BytesIO()
        img.save(img_io, format='JPEG')
        img_io.seek(0)

        image_file = SimpleUploadedFile(
            name='test_image.jpg',
            content=img_io.read(),
            content_type='image/jpeg'
        )

        data = self.valid_data.copy()
        form = PostForm(
            data=data,
            files={'featured_image': image_file}
        )
        self.assertTrue(form.is_valid())


class UserRegistrationFormTest(TestCase):
    """ทดสอบ User Registration Form"""

    def test_registration_valid(self):
        """ทดสอบ Registration Form ด้วยข้อมูลถูกต้อง"""
        form = UserRegistrationForm(data={
            'username': 'newuser',
            'email': 'new@example.com',
            'password1': 'StrongPass123!',
            'password2': 'StrongPass123!',
        })
        self.assertTrue(form.is_valid())

    def test_passwords_dont_match(self):
        """ทดสอบเมื่อ Password ไม่ตรงกัน"""
        form = UserRegistrationForm(data={
            'username': 'newuser',
            'email': 'new@example.com',
            'password1': 'StrongPass123!',
            'password2': 'DifferentPass456!',
        })
        self.assertFalse(form.is_valid())
        self.assertIn('password2', form.errors)

    def test_duplicate_username(self):
        """ทดสอบ Username ซ้ำ"""
        User.objects.create_user(
            username='existinguser',
            password='pass123'
        )
        form = UserRegistrationForm(data={
            'username': 'existinguser',
            'email': 'new@example.com',
            'password1': 'StrongPass123!',
            'password2': 'StrongPass123!',
        })
        self.assertFalse(form.is_valid())
        self.assertIn('username', form.errors)

    def test_weak_password(self):
        """ทดสอบ Password ที่อ่อนแอ"""
        form = UserRegistrationForm(data={
            'username': 'newuser',
            'email': 'new@example.com',
            'password1': '123456',  # Password ง่ายเกินไป
            'password2': '123456',
        })
        self.assertFalse(form.is_valid())
```

---

## 8. DRF APITestCase สำหรับ REST API

```python
# myapp/tests/test_api.py
from rest_framework.test import APITestCase, APIClient
from rest_framework import status
from rest_framework.authtoken.models import Token
from django.urls import reverse
from django.contrib.auth.models import User
from myapp.models import Post, Category


class PostAPITest(APITestCase):
    """ทดสอบ Post REST API"""

    def setUp(self):
        # สร้าง Users
        self.user = User.objects.create_user(
            username='apiuser',
            password='apipass123',
            email='api@example.com'
        )
        self.admin = User.objects.create_superuser(
            username='apiadmin',
            password='adminpass123',
            email='admin@example.com'
        )

        # สร้าง Token สำหรับ Authentication
        self.user_token = Token.objects.create(user=self.user)
        self.admin_token = Token.objects.create(user=self.admin)

        # สร้าง Category และ Post
        self.category = Category.objects.create(
            name='Tech', slug='tech'
        )
        self.post = Post.objects.create(
            title='API Test Post',
            slug='api-test-post',
            content='API test content',
            author=self.user,
            category=self.category,
            status='published'
        )

    # --- GET Tests ---

    def test_list_posts_unauthenticated(self):
        """ทดสอบ List Posts โดยไม่ต้อง Authenticate"""
        url = reverse('api:post-list')
        response = self.client.get(url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_list_posts_returns_correct_count(self):
        """ทดสอบว่า List Posts คืนข้อมูลถูกจำนวน"""
        url = reverse('api:post-list')
        response = self.client.get(url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['count'], 1)

    def test_get_post_detail(self):
        """ทดสอบ Get Post Detail"""
        url = reverse('api:post-detail', kwargs={'pk': self.post.pk})
        response = self.client.get(url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], 'API Test Post')
        self.assertEqual(response.data['author']['username'], 'apiuser')

    # --- POST Tests ---

    def test_create_post_unauthenticated(self):
        """ทดสอบสร้าง Post โดยไม่ Authenticate"""
        url = reverse('api:post-list')
        data = {
            'title': 'New Post',
            'content': 'Content',
            'category': self.category.pk,
        }
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_create_post_with_token_auth(self):
        """ทดสอบสร้าง Post ด้วย Token Authentication"""
        # ตั้งค่า Token Authentication
        self.client.credentials(
            HTTP_AUTHORIZATION=f'Token {self.user_token.key}'
        )
        url = reverse('api:post-list')
        data = {
            'title': 'Token Auth Post',
            'slug': 'token-auth-post',
            'content': 'Content here',
            'category': self.category.pk,
            'status': 'published',
        }
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(response.data['title'], 'Token Auth Post')
        # ตรวจสอบว่า Author ถูกตั้งเป็น current user
        self.assertEqual(response.data['author']['username'], 'apiuser')

    def test_create_post_with_jwt(self):
        """ทดสอบ JWT Authentication"""
        # ขอ JWT Token
        login_url = reverse('api:token-obtain-pair')
        login_response = self.client.post(login_url, {
            'username': 'apiuser',
            'password': 'apipass123'
        }, format='json')
        self.assertEqual(login_response.status_code, status.HTTP_200_OK)

        # ใช้ JWT Token
        access_token = login_response.data['access']
        self.client.credentials(
            HTTP_AUTHORIZATION=f'Bearer {access_token}'
        )

        # สร้าง Post
        url = reverse('api:post-list')
        data = {
            'title': 'JWT Post',
            'slug': 'jwt-post',
            'content': 'JWT content',
            'category': self.category.pk,
        }
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)

    # --- PUT/PATCH Tests ---

    def test_update_own_post(self):
        """ทดสอบแก้ไข Post ของตัวเอง"""
        self.client.credentials(
            HTTP_AUTHORIZATION=f'Token {self.user_token.key}'
        )
        url = reverse('api:post-detail', kwargs={'pk': self.post.pk})
        data = {'title': 'Updated Title'}
        response = self.client.patch(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], 'Updated Title')

    def test_update_others_post(self):
        """ทดสอบว่าไม่สามารถแก้ไข Post ของคนอื่น"""
        other_user = User.objects.create_user(
            username='other',
            password='other123'
        )
        other_token = Token.objects.create(user=other_user)
        self.client.credentials(
            HTTP_AUTHORIZATION=f'Token {other_token.key}'
        )
        url = reverse('api:post-detail', kwargs={'pk': self.post.pk})
        data = {'title': 'Hacked Title'}
        response = self.client.patch(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)

    # --- DELETE Tests ---

    def test_delete_own_post(self):
        """ทดสอบลบ Post ของตัวเอง"""
        self.client.credentials(
            HTTP_AUTHORIZATION=f'Token {self.user_token.key}'
        )
        url = reverse('api:post-detail', kwargs={'pk': self.post.pk})
        response = self.client.delete(url)
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertFalse(Post.objects.filter(pk=self.post.pk).exists())

    def test_admin_can_delete_any_post(self):
        """ทดสอบว่า Admin ลบ Post ของใครก็ได้"""
        self.client.credentials(
            HTTP_AUTHORIZATION=f'Token {self.admin_token.key}'
        )
        url = reverse('api:post-detail', kwargs={'pk': self.post.pk})
        response = self.client.delete(url)
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)

    # --- Filtering/Search Tests ---

    def test_filter_by_category(self):
        """ทดสอบ Filter โดย Category"""
        other_category = Category.objects.create(
            name='Science', slug='science'
        )
        Post.objects.create(
            title='Science Post',
            slug='science-post',
            content='Science content',
            author=self.user,
            category=other_category,
            status='published'
        )
        url = reverse('api:post-list') + f'?category={self.category.pk}'
        response = self.client.get(url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        # ควรเห็นเฉพาะ Tech posts
        for post in response.data['results']:
            self.assertEqual(post['category']['name'], 'Tech')

    def test_search_posts(self):
        """ทดสอบ Search Posts"""
        url = reverse('api:post-list') + '?search=API+Test'
        response = self.client.get(url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['count'], 1)


class AuthAPITest(APITestCase):
    """ทดสอบ Authentication API"""

    def test_register_user(self):
        """ทดสอบการสมัครสมาชิก"""
        url = reverse('api:register')
        data = {
            'username': 'newuser',
            'email': 'new@example.com',
            'password': 'StrongPass123!',
            'password_confirm': 'StrongPass123!',
        }
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertTrue(User.objects.filter(username='newuser').exists())

    def test_obtain_token(self):
        """ทดสอบการขอ Token"""
        User.objects.create_user(
            username='tokenuser',
            password='pass123'
        )
        url = reverse('api:api-token-auth')
        response = self.client.post(url, {
            'username': 'tokenuser',
            'password': 'pass123'
        }, format='json')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertIn('token', response.data)
```

---

## 9. factory_boy สำหรับสร้าง Test Data

```bash
# ติดตั้ง factory_boy
pip install factory-boy
```

```python
# myapp/tests/factories.py
import factory
import factory.django
from django.contrib.auth.models import User
from myapp.models import Post, Category, Tag, Comment


class UserFactory(factory.django.DjangoModelFactory):
    """Factory สำหรับสร้าง User"""

    class Meta:
        model = User
        # ป้องกัน duplicate username
        django_get_or_create = ('username',)

    # ใช้ Sequence เพื่อให้ username ไม่ซ้ำ
    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.LazyAttribute(lambda obj: f'{obj.username}@example.com')
    first_name = factory.Faker('first_name')
    last_name = factory.Faker('last_name')
    password = factory.PostGenerationMethodCall('set_password', 'testpass123')
    is_active = True


class AdminUserFactory(UserFactory):
    """Factory สำหรับ Admin User"""
    is_staff = True
    is_superuser = True
    username = factory.Sequence(lambda n: f'admin{n}')


class CategoryFactory(factory.django.DjangoModelFactory):
    """Factory สำหรับ Category"""

    class Meta:
        model = Category
        django_get_or_create = ('slug',)

    name = factory.Sequence(lambda n: f'Category {n}')
    # slug สร้างจาก name อัตโนมัติ
    slug = factory.LazyAttribute(
        lambda obj: obj.name.lower().replace(' ', '-')
    )
    description = factory.Faker('sentence')


class TagFactory(factory.django.DjangoModelFactory):
    """Factory สำหรับ Tag"""

    class Meta:
        model = Tag
        django_get_or_create = ('slug',)

    name = factory.Sequence(lambda n: f'tag{n}')
    slug = factory.LazyAttribute(lambda obj: obj.name.lower())


class PostFactory(factory.django.DjangoModelFactory):
    """Factory สำหรับ Post"""

    class Meta:
        model = Post

    title = factory.Faker('sentence', nb_words=6)
    # slug สร้างจาก title
    slug = factory.LazyAttribute(
        lambda obj: obj.title.lower().replace(' ', '-').replace('.', '')[:50]
    )
    content = factory.Faker('paragraphs', nb=3, extended=False)
    author = factory.SubFactory(UserFactory)   # สร้าง User อัตโนมัติ
    category = factory.SubFactory(CategoryFactory)
    status = 'published'

    # Many-to-Many field (Tags)
    @factory.post_generation
    def tags(self, create, extracted, **kwargs):
        if not create:
            return
        if extracted:
            # ถ้าส่ง tags มา ใช้ tags นั้น
            for tag in extracted:
                self.tags.add(tag)
        else:
            # สร้าง Tags อัตโนมัติ 2 อัน
            self.tags.add(TagFactory(), TagFactory())


class DraftPostFactory(PostFactory):
    """Factory สำหรับ Draft Post"""
    status = 'draft'


class CommentFactory(factory.django.DjangoModelFactory):
    """Factory สำหรับ Comment"""

    class Meta:
        model = Comment

    post = factory.SubFactory(PostFactory)
    author = factory.SubFactory(UserFactory)
    content = factory.Faker('paragraph')
    is_approved = True
```

```python
# myapp/tests/test_with_factories.py
from django.test import TestCase
from myapp.tests.factories import (
    UserFactory, PostFactory, DraftPostFactory,
    CategoryFactory, CommentFactory
)


class TestWithFactoryBoy(TestCase):
    """ทดสอบด้วย factory_boy"""

    def test_create_single_post(self):
        """สร้าง Post เดี่ยว"""
        post = PostFactory()
        self.assertIsNotNone(post.pk)
        self.assertEqual(post.status, 'published')
        # Author และ Category ถูกสร้างอัตโนมัติ
        self.assertIsNotNone(post.author.pk)
        self.assertIsNotNone(post.category.pk)

    def test_create_post_with_specific_author(self):
        """สร้าง Post พร้อมระบุ Author"""
        from django.contrib.auth.models import User
        specific_author = UserFactory(username='specificauthor')
        post = PostFactory(author=specific_author)
        self.assertEqual(post.author.username, 'specificauthor')

    def test_create_multiple_posts(self):
        """สร้าง Posts หลายอัน"""
        posts = PostFactory.create_batch(10)
        self.assertEqual(len(posts), 10)
        from myapp.models import Post
        self.assertEqual(Post.objects.count(), 10)

    def test_draft_post_factory(self):
        """ทดสอบ DraftPostFactory"""
        draft = DraftPostFactory()
        self.assertEqual(draft.status, 'draft')

    def test_post_with_custom_tags(self):
        """สร้าง Post พร้อม Tags ที่กำหนด"""
        from myapp.tests.factories import TagFactory
        tag1 = TagFactory(name='python')
        tag2 = TagFactory(name='django')
        post = PostFactory(tags=[tag1, tag2])
        self.assertIn(tag1, post.tags.all())
        self.assertIn(tag2, post.tags.all())

    def test_create_comment_with_post(self):
        """สร้าง Comment พร้อม Post"""
        comment = CommentFactory()
        self.assertIsNotNone(comment.post.pk)
        self.assertIsNotNone(comment.author.pk)

    def test_build_without_saving(self):
        """สร้าง Object โดยไม่บันทึก Database"""
        post = PostFactory.build()
        # Post ไม่ถูกบันทึกใน Database
        self.assertIsNone(post.pk)

    def test_stub_for_lightweight_testing(self):
        """ใช้ Stub สำหรับ Test เบาๆ"""
        post = PostFactory.stub()
        # Stub มี attributes แต่ไม่ใช่ Django Model instance
        self.assertEqual(post.status, 'published')
```

---

## 10. pytest-django Integration

```bash
# ติดตั้ง pytest-django
pip install pytest pytest-django pytest-cov

# สร้างไฟล์ pytest.ini หรือ pyproject.toml
```

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = myproject.settings.test
python_files = tests.py test_*.py *_tests.py
python_classes = Test*
python_functions = test_*
addopts = 
    --strict-markers
    -v
    --tb=short
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
    unit: marks tests as unit tests
```

```toml
# pyproject.toml (วิธีอื่น)
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "myproject.settings.test"
python_files = ["tests.py", "test_*.py", "*_tests.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = ["-v", "--tb=short"]
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
]
```

```python
# myapp/tests/test_with_pytest.py
"""
ทดสอบด้วย pytest style
pytest ใช้ functions แทน classes (แต่ใช้ class ก็ได้)
"""
import pytest
from django.contrib.auth.models import User
from myapp.models import Post, Category
from myapp.tests.factories import PostFactory, UserFactory


# --- Fixtures ---

@pytest.fixture
def user(db):
    """Fixture: สร้าง User"""
    return UserFactory()


@pytest.fixture
def admin_user(db):
    """Fixture: สร้าง Admin User"""
    return UserFactory(is_staff=True, is_superuser=True)


@pytest.fixture
def category(db):
    """Fixture: สร้าง Category"""
    return Category.objects.create(name='Tech', slug='tech')


@pytest.fixture
def post(db, user, category):
    """Fixture: สร้าง Post (ต้องการ user และ category)"""
    return Post.objects.create(
        title='Test Post',
        slug='test-post',
        content='Test content',
        author=user,
        category=category,
        status='published'
    )


@pytest.fixture
def authenticated_client(client, user):
    """Fixture: Client ที่ Login แล้ว"""
    client.force_login(user)
    return client


# --- Test Functions ---

@pytest.mark.django_db
def test_post_creation(user, category):
    """ทดสอบสร้าง Post"""
    post = Post.objects.create(
        title='Pytest Post',
        slug='pytest-post',
        content='Pytest content',
        author=user,
        category=category,
    )
    assert post.pk is not None
    assert post.title == 'Pytest Post'


@pytest.mark.django_db
def test_post_list_view(client, post):
    """ทดสอบ Post List View"""
    from django.urls import reverse
    url = reverse('post-list')
    response = client.get(url)
    assert response.status_code == 200


@pytest.mark.django_db
def test_create_post_authenticated(authenticated_client, category):
    """ทดสอบสร้าง Post เมื่อ Login แล้ว"""
    from django.urls import reverse
    url = reverse('post-create')
    data = {
        'title': 'New Pytest Post',
        'slug': 'new-pytest-post',
        'content': 'Content here',
        'category': category.pk,
        'status': 'published',
    }
    response = authenticated_client.post(url, data)
    assert response.status_code == 302
    assert Post.objects.filter(title='New Pytest Post').exists()


@pytest.mark.slow
@pytest.mark.django_db
def test_bulk_post_creation():
    """ทดสอบสร้าง Posts จำนวนมาก (Slow Test)"""
    posts = PostFactory.create_batch(100)
    assert Post.objects.count() == 100


# --- Parametrize Tests ---

@pytest.mark.django_db
@pytest.mark.parametrize("status,expected_visible", [
    ('published', True),
    ('draft', False),
    ('archived', False),
])
def test_post_visibility_by_status(client, user, category, status, expected_visible):
    """ทดสอบ Visibility ของ Post ตาม Status"""
    from django.urls import reverse
    post = Post.objects.create(
        title=f'{status.capitalize()} Post',
        slug=f'{status}-post',
        content='Content',
        author=user,
        category=category,
        status=status
    )
    url = reverse('post-detail', kwargs={'slug': post.slug})
    response = client.get(url)
    if expected_visible:
        assert response.status_code == 200
    else:
        assert response.status_code == 404


# --- Class-based pytest Tests ---

class TestPostModel:
    """Tests แบบ Class ใน pytest"""

    @pytest.mark.django_db
    def test_str_representation(self, post):
        assert str(post) == 'Test Post'

    @pytest.mark.django_db
    def test_get_absolute_url(self, post):
        url = post.get_absolute_url()
        assert url == f'/posts/{post.slug}/'
```

---

## 11. Test Database Handling

```python
# myproject/settings/test.py
"""
Settings เฉพาะสำหรับ Testing
"""
from .base import *  # Import base settings

# ใช้ In-memory SQLite สำหรับ Tests เร็วขึ้น
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': ':memory:',  # In-memory database
    }
}

# ปิด Password Hashing เพื่อให้ Tests เร็วขึ้น
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.MD5PasswordHasher',
]

# ปิด Email จริง
EMAIL_BACKEND = 'django.core.mail.backends.locmem.EmailBackend'

# ปิด Celery tasks
CELERY_TASK_ALWAYS_EAGER = True
CELERY_TASK_EAGER_PROPAGATES = True

# Media files สำหรับ Tests
import tempfile
MEDIA_ROOT = tempfile.mkdtemp()

# ปิด Caching
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.dummy.DummyCache',
    }
}
```

```python
# myapp/tests/test_database.py
from django.test import TestCase, TransactionTestCase
from django.db import connection
from myapp.models import Post, Category
from myapp.tests.factories import UserFactory, PostFactory


class TestDatabaseRollback(TestCase):
    """
    Django TestCase ใช้ Transaction Rollback
    ข้อมูลจะหายหลังแต่ละ Test
    """

    def test_first_test(self):
        """Test แรก สร้าง Post"""
        PostFactory()
        self.assertEqual(Post.objects.count(), 1)

    def test_second_test(self):
        """Test สอง - ไม่เห็น Post จาก test_first_test"""
        # Database ถูก Rollback แล้ว
        self.assertEqual(Post.objects.count(), 0)


class TestTransactionHandling(TransactionTestCase):
    """
    TransactionTestCase ใช้ TRUNCATE แทน ROLLBACK
    ใช้เมื่อต้องการทดสอบ Database Transactions
    """

    def test_atomic_transaction(self):
        """ทดสอบ Atomic Transaction"""
        from django.db import transaction

        user = UserFactory()
        category_name = 'Test Category'
        category_slug = 'test-category'

        # Atomic block - ถ้า error ข้อมูลทั้งหมดจะ Rollback
        with transaction.atomic():
            category = Category.objects.create(
                name=category_name,
                slug=category_slug
            )
            post = Post.objects.create(
                title='Atomic Post',
                slug='atomic-post',
                content='Content',
                author=user,
                category=category,
            )

        self.assertEqual(Category.objects.count(), 1)
        self.assertEqual(Post.objects.count(), 1)

    def test_transaction_rollback_on_error(self):
        """ทดสอบว่า Transaction Rollback เมื่อเกิด Error"""
        from django.db import transaction

        user = UserFactory()
        initial_count = Post.objects.count()

        try:
            with transaction.atomic():
                category = Category.objects.create(
                    name='Rollback Category',
                    slug='rollback'
                )
                Post.objects.create(
                    title='Rollback Post',
                    slug='rollback-post',
                    content='Content',
                    author=user,
                    category=category,
                )
                # จงใจทำให้ Error เกิดขึ้น
                raise ValueError("Intentional error")
        except ValueError:
            pass

        # ข้อมูลควรถูก Rollback กลับ
        self.assertEqual(Post.objects.count(), initial_count)


class TestDatabasePerformance(TestCase):
    """ทดสอบ Database Query Performance"""

    def setUp(self):
        user = UserFactory()
        category = Category.objects.create(name='Tech', slug='tech')
        # สร้าง Posts พร้อม Comments
        for i in range(10):
            post = PostFactory(author=user, category=category)

    def test_no_n_plus_one_queries(self):
        """ทดสอบว่าไม่มี N+1 Query problem"""
        from django.test.utils import CaptureQueriesContext

        with CaptureQueriesContext(connection) as ctx:
            # ดึง Posts พร้อม select_related
            posts = list(
                Post.objects.select_related('author', 'category').all()
            )
            # Access related fields
            for post in posts:
                _ = post.author.username
                _ = post.category.name

        # ควรมี Queries น้อยกว่า len(posts) + 1
        # (ถ้ามี N+1 จะมี query เพิ่มสำหรับแต่ละ post)
        self.assertLess(len(ctx.captured_queries), len(posts))
```

---

## 12. Mocking External Services

```python
# myapp/tests/test_mocking.py
from django.test import TestCase
from unittest.mock import Mock, MagicMock, patch, call
from myapp.services import (
    EmailService, PaymentService, WeatherService, S3Service
)
from myapp.tests.factories import UserFactory, PostFactory


class TestEmailMocking(TestCase):
    """ทดสอบ Email Service ด้วย Mock"""

    @patch('django.core.mail.send_mail')
    def test_send_welcome_email(self, mock_send_mail):
        """Mock send_mail function"""
        user = UserFactory()
        EmailService.send_welcome_email(user)

        # ตรวจสอบว่า send_mail ถูกเรียก
        mock_send_mail.assert_called_once()

        # ตรวจสอบ Arguments ที่ส่งไป
        args = mock_send_mail.call_args
        self.assertIn(user.email, args[0])  # positional args
        self.assertIn('Welcome', args[0][0])  # subject

    @patch('django.core.mail.send_mail')
    def test_send_post_notification(self, mock_send_mail):
        """Mock Email Notification เมื่อ Post ใหม่"""
        post = PostFactory()
        EmailService.notify_subscribers(post)

        # ตรวจสอบว่าถูกเรียก
        self.assertTrue(mock_send_mail.called)

    def test_email_backend_in_test(self):
        """ใช้ locmem Email Backend"""
        from django.core import mail
        from django.core.mail import send_mail

        send_mail(
            subject='Test Subject',
            message='Test message',
            from_email='from@example.com',
            recipient_list=['to@example.com'],
        )

        # ตรวจสอบ Email ที่ถูกส่ง
        self.assertEqual(len(mail.outbox), 1)
        self.assertEqual(mail.outbox[0].subject, 'Test Subject')
        self.assertEqual(mail.outbox[0].to, ['to@example.com'])


class TestExternalAPIsMocking(TestCase):
    """ทดสอบ External APIs ด้วย Mock"""

    @patch('requests.get')
    def test_weather_service(self, mock_get):
        """Mock requests.get สำหรับ Weather API"""
        # กำหนด Response ที่ Mock จะคืน
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
            'weather': 'sunny',
            'temperature': 28.5
        }
        mock_get.return_value = mock_response

        weather = WeatherService.get_weather('Bangkok')

        # ตรวจสอบว่า requests.get ถูกเรียกด้วย URL ที่ถูกต้อง
        mock_get.assert_called_once_with(
            'https://api.weather.com/v1/current',
            params={'city': 'Bangkok', 'units': 'metric'}
        )

        self.assertEqual(weather['weather'], 'sunny')
        self.assertEqual(weather['temperature'], 28.5)

    @patch('requests.get')
    def test_weather_service_api_error(self, mock_get):
        """ทดสอบเมื่อ API คืน Error"""
        mock_response = Mock()
        mock_response.status_code = 500
        mock_response.raise_for_status.side_effect = Exception('API Error')
        mock_get.return_value = mock_response

        with self.assertRaises(Exception):
            WeatherService.get_weather('Bangkok')

    @patch('myapp.services.stripe.Charge.create')
    def test_payment_service(self, mock_charge):
        """Mock Stripe Payment"""
        mock_charge.return_value = {
            'id': 'ch_test123',
            'status': 'succeeded',
            'amount': 1000
        }

        result = PaymentService.charge_card(
            token='tok_test',
            amount=1000,
            currency='thb'
        )

        self.assertEqual(result['status'], 'succeeded')
        mock_charge.assert_called_once_with(
            amount=1000,
            currency='thb',
            source='tok_test'
        )


class TestS3MockService(TestCase):
    """ทดสอบ S3 File Upload ด้วย Mock"""

    @patch('boto3.client')
    def test_upload_file(self, mock_boto3_client):
        """Mock S3 Upload"""
        # Mock boto3 client
        mock_s3 = MagicMock()
        mock_boto3_client.return_value = mock_s3
        mock_s3.upload_file.return_value = None

        service = S3Service()
        result = service.upload_file(
            file_path='/tmp/test.jpg',
            bucket='my-bucket',
            key='uploads/test.jpg'
        )

        # ตรวจสอบว่า upload_file ถูกเรียก
        mock_s3.upload_file.assert_called_once_with(
            '/tmp/test.jpg',
            'my-bucket',
            'uploads/test.jpg'
        )

    @patch('boto3.client')
    def test_s3_upload_failure(self, mock_boto3_client):
        """ทดสอบเมื่อ S3 Upload ล้มเหลว"""
        mock_s3 = MagicMock()
        mock_boto3_client.return_value = mock_s3
        mock_s3.upload_file.side_effect = Exception('Connection Error')

        service = S3Service()
        with self.assertRaises(Exception) as ctx:
            service.upload_file('/tmp/test.jpg', 'bucket', 'key')

        self.assertIn('Connection Error', str(ctx.exception))


class TestCeleryTaskMocking(TestCase):
    """Mock Celery Tasks"""

    @patch('myapp.tasks.send_notification_task.delay')
    def test_notification_task_called(self, mock_task):
        """ทดสอบว่า Celery Task ถูกเรียก"""
        post = PostFactory()

        from myapp.services import NotificationService
        NotificationService.notify_on_publish(post)

        # ตรวจสอบว่า Task ถูกเรียกด้วย argument ที่ถูกต้อง
        mock_task.assert_called_once_with(post_id=post.pk)
```

---

## 13. Coverage ด้วย pytest-cov

```bash
# รัน Tests พร้อม Coverage
pytest --cov=myapp --cov-report=html --cov-report=term-missing

# หรือกำหนดใน pytest.ini
# addopts = --cov=myapp --cov-report=html --cov-fail-under=80
```

```ini
# .coveragerc - ตั้งค่า Coverage
[run]
source = myapp
omit =
    */migrations/*
    */tests/*
    manage.py
    */settings/*
    */wsgi.py
    */asgi.py

[report]
exclude_lines =
    pragma: no cover
    def __repr__
    if self.debug:
    if settings.DEBUG
    raise AssertionError
    raise NotImplementedError
    if 0:
    if __name__ == .__main__.:
    class.*\bProtocol\b:
    @(abc\.)?abstractmethod

show_missing = True
precision = 2

[html]
directory = htmlcov
title = Blog App Coverage Report
```

```python
# myapp/tests/test_coverage_example.py
"""
ตัวอย่างโค้ดที่ครอบคลุมทุก Branch
เพื่อให้ Coverage ครบ
"""
from django.test import TestCase
from myapp.models import Post
from myapp.tests.factories import UserFactory, PostFactory, CategoryFactory


class TestAllBranches(TestCase):
    """ทดสอบทุก Branch ใน Code"""

    def setUp(self):
        self.user = UserFactory()
        self.category = CategoryFactory()

    def test_post_published_status(self):
        """ทดสอบ Post ที่ Published"""
        post = PostFactory(
            author=self.user,
            category=self.category,
            status='published'
        )
        # Branch: status == 'published'
        self.assertTrue(post.is_published)

    def test_post_draft_status(self):
        """ทดสอบ Post ที่เป็น Draft"""
        post = PostFactory(
            author=self.user,
            category=self.category,
            status='draft'
        )
        # Branch: status != 'published'
        self.assertFalse(post.is_published)

    def test_post_with_no_tags(self):
        """ทดสอบ Post ที่ไม่มี Tags"""
        post = PostFactory(
            author=self.user,
            category=self.category,
            tags=[]  # ไม่มี Tags
        )
        # Branch: tags is empty
        self.assertEqual(post.tags.count(), 0)
        self.assertEqual(post.get_tags_display(), 'ไม่มี Tags')

    def test_post_with_tags(self):
        """ทดสอบ Post ที่มี Tags"""
        from myapp.tests.factories import TagFactory
        tag = TagFactory(name='python')
        post = PostFactory(
            author=self.user,
            category=self.category,
            tags=[tag]
        )
        # Branch: tags is not empty
        self.assertNotEqual(post.get_tags_display(), 'ไม่มี Tags')
```

---

## 14. Complete Test Suite: Blog App

```python
# myapp/tests/__init__.py
# (ไฟล์ว่าง)
```

```python
# myapp/tests/test_blog_complete.py
"""
Complete Test Suite สำหรับ Blog Application
รวม Tests ทุกประเภท
"""
from django.test import TestCase
from django.urls import reverse
from django.contrib.auth.models import User
from django.core import mail
from unittest.mock import patch
from rest_framework.test import APITestCase
from rest_framework import status

from myapp.models import Post, Category, Comment, Tag
from myapp.tests.factories import (
    UserFactory, PostFactory, CategoryFactory,
    CommentFactory, TagFactory
)


# ============================================================
# MODEL TESTS
# ============================================================

class BlogModelTests(TestCase):
    """Tests สำหรับ Blog Models"""

    def setUp(self):
        self.user = UserFactory(username='blogauthor')
        self.category = CategoryFactory(name='Programming', slug='programming')

    def test_post_str(self):
        post = PostFactory(title='Hello Django', author=self.user)
        self.assertEqual(str(post), 'Hello Django')

    def test_post_get_absolute_url(self):
        post = PostFactory(slug='hello-django', author=self.user)
        self.assertEqual(post.get_absolute_url(), '/posts/hello-django/')

    def test_post_published_manager(self):
        published = PostFactory(author=self.user, status='published')
        draft = PostFactory(author=self.user, status='draft')
        published_posts = Post.published.all()
        self.assertIn(published, published_posts)
        self.assertNotIn(draft, published_posts)

    def test_comment_str(self):
        post = PostFactory(author=self.user)
        comment = CommentFactory(post=post, author=self.user)
        self.assertIn(self.user.username, str(comment))

    def test_category_post_count(self):
        PostFactory.create_batch(5, category=self.category, author=self.user)
        self.assertEqual(self.category.post_count, 5)

    def test_tag_cloud(self):
        tags = TagFactory.create_batch(3)
        for tag in tags:
            PostFactory(author=self.user, tags=[tag])
        # ตรวจสอบว่า Tag Cloud ทำงานได้
        from myapp.models import Tag as TagModel
        cloud = TagModel.objects.annotate_post_count()
        self.assertEqual(cloud.count(), 3)


# ============================================================
# VIEW TESTS
# ============================================================

class BlogViewTests(TestCase):
    """Tests สำหรับ Blog Views"""

    def setUp(self):
        self.user = UserFactory(username='viewer')
        self.category = CategoryFactory()
        self.post = PostFactory(
            author=self.user,
            category=self.category,
            status='published'
        )

    # Homepage
    def test_homepage_status_200(self):
        response = self.client.get(reverse('home'))
        self.assertEqual(response.status_code, 200)

    def test_homepage_shows_recent_posts(self):
        response = self.client.get(reverse('home'))
        self.assertIn('recent_posts', response.context)

    # Post List
    def test_post_list_status_200(self):
        response = self.client.get(reverse('post-list'))
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'blog/post_list.html')

    def test_post_list_shows_only_published(self):
        draft = PostFactory(author=self.user, status='draft')
        response = self.client.get(reverse('post-list'))
        posts = response.context['posts']
        self.assertNotIn(draft, posts)

    # Post Detail
    def test_post_detail_status_200(self):
        url = reverse('post-detail', kwargs={'slug': self.post.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)

    def test_post_detail_context(self):
        url = reverse('post-detail', kwargs={'slug': self.post.slug})
        response = self.client.get(url)
        self.assertEqual(response.context['post'], self.post)
        self.assertIn('comments', response.context)
        self.assertIn('related_posts', response.context)

    # Comment Submission
    def test_submit_comment_authenticated(self):
        self.client.force_login(self.user)
        url = reverse('post-comment', kwargs={'slug': self.post.slug})
        response = self.client.post(url, {
            'content': 'Great post!'
        })
        self.assertEqual(response.status_code, 302)
        self.assertEqual(Comment.objects.count(), 1)

    def test_submit_comment_unauthenticated(self):
        url = reverse('post-comment', kwargs={'slug': self.post.slug})
        response = self.client.post(url, {'content': 'Test'})
        self.assertEqual(response.status_code, 302)
        # Redirect ไป Login
        self.assertRedirects(response, f'/login/?next={url}')

    # Search
    def test_search_posts(self):
        PostFactory(title='Python Tutorial', author=self.user, status='published')
        url = reverse('post-search') + '?q=Python'
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, 'Python Tutorial')

    def test_search_empty_query(self):
        url = reverse('post-search') + '?q='
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)

    # Category
    def test_category_posts(self):
        url = reverse('category-posts', kwargs={'slug': self.category.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)


# ============================================================
# FORM TESTS
# ============================================================

class BlogFormTests(TestCase):
    """Tests สำหรับ Blog Forms"""

    def setUp(self):
        self.category = CategoryFactory()

    def test_post_form_valid(self):
        from myapp.forms import PostForm
        form = PostForm(data={
            'title': 'Valid Post',
            'slug': 'valid-post',
            'content': 'Valid content here',
            'category': self.category.pk,
            'status': 'published',
        })
        self.assertTrue(form.is_valid())

    def test_comment_form_valid(self):
        from myapp.forms import CommentForm
        form = CommentForm(data={'content': 'Great article!'})
        self.assertTrue(form.is_valid())

    def test_comment_form_empty(self):
        from myapp.forms import CommentForm
        form = CommentForm(data={'content': ''})
        self.assertFalse(form.is_valid())


# ============================================================
# API TESTS
# ============================================================

class BlogAPITests(APITestCase):
    """Tests สำหรับ Blog REST API"""

    def setUp(self):
        self.user = UserFactory()
        self.category = CategoryFactory()
        self.post = PostFactory(
            author=self.user,
            category=self.category,
            status='published'
        )
        self.client.force_authenticate(user=self.user)

    def test_api_post_list(self):
        url = reverse('api:post-list')
        response = self.client.get(url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_api_post_create(self):
        url = reverse('api:post-list')
        data = {
            'title': 'API Post',
            'slug': 'api-post',
            'content': 'API content',
            'category': self.category.pk,
            'status': 'published',
        }
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)

    def test_api_post_detail(self):
        url = reverse('api:post-detail', kwargs={'pk': self.post.pk})
        response = self.client.get(url)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], self.post.title)

    def test_api_post_update(self):
        url = reverse('api:post-detail', kwargs={'pk': self.post.pk})
        response = self.client.patch(url, {'title': 'Updated'}, format='json')
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.post.refresh_from_db()
        self.assertEqual(self.post.title, 'Updated')

    def test_api_post_delete(self):
        url = reverse('api:post-detail', kwargs={'pk': self.post.pk})
        response = self.client.delete(url)
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)

    def test_api_unauthenticated_access(self):
        self.client.force_authenticate(user=None)
        url = reverse('api:post-list')
        response = self.client.post(url, {}, format='json')
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)


# ============================================================
# EMAIL TESTS
# ============================================================

class BlogEmailTests(TestCase):
    """Tests สำหรับ Email Notifications"""

    def setUp(self):
        self.user = UserFactory()
        self.post = PostFactory(author=self.user, status='published')

    @patch('myapp.services.send_new_post_notification')
    def test_notification_sent_on_publish(self, mock_notify):
        """ทดสอบว่า Notification ถูกส่งเมื่อ Publish"""
        draft = PostFactory(author=self.user, status='draft')
        draft.status = 'published'
        draft.save()
        mock_notify.assert_called_once_with(draft)

    def test_comment_notification_email(self):
        """ทดสอบ Email เมื่อมี Comment ใหม่"""
        self.client.force_login(self.user)
        url = reverse('post-comment', kwargs={'slug': self.post.slug})
        self.client.post(url, {'content': 'New comment!'})
        # ตรวจสอบ Email
        self.assertEqual(len(mail.outbox), 1)
        self.assertIn(self.post.author.email, mail.outbox[0].to)


# ============================================================
# SIGNAL TESTS
# ============================================================

class BlogSignalTests(TestCase):
    """Tests สำหรับ Django Signals"""

    def test_user_profile_created_on_register(self):
        """ทดสอบว่า Profile ถูกสร้างเมื่อ User ลงทะเบียน"""
        from myapp.models import UserProfile
        user = User.objects.create_user(
            username='signaltest',
            password='pass123'
        )
        # Signal ควรสร้าง Profile อัตโนมัติ
        self.assertTrue(UserProfile.objects.filter(user=user).exists())

    def test_slug_auto_generated_on_save(self):
        """ทดสอบว่า Slug ถูกสร้างอัตโนมัติเมื่อบันทึก"""
        user = UserFactory()
        category = CategoryFactory()
        post = Post(
            title='Auto Slug Test',
            content='Content',
            author=user,
            category=category,
        )
        post.save()
        # Slug ควรถูกสร้างจาก title
        self.assertEqual(post.slug, 'auto-slug-test')
```

---

## 15. การรัน Tests

```bash
# รัน Tests ทั้งหมด
python manage.py test

# รัน Tests เฉพาะ App
python manage.py test myapp

# รัน Tests เฉพาะ Module
python manage.py test myapp.tests.test_models

# รัน Tests เฉพาะ Class
python manage.py test myapp.tests.test_models.PostModelTest

# รัน Tests เฉพาะ Method
python manage.py test myapp.tests.test_models.PostModelTest.test_post_creation

# รัน Tests พร้อม Verbose output
python manage.py test --verbosity=2

# รัน Tests พร้อม Coverage (ด้วย pytest)
pytest --cov=myapp --cov-report=html

# รัน เฉพาะ Slow tests (pytest markers)
pytest -m slow

# รัน Tests ยกเว้น Slow tests
pytest -m "not slow"

# รัน Tests แบบ Parallel (ต้องติดตั้ง pytest-xdist)
pytest -n auto
```

---

## 16. สรุป Part 074

✅ **Django TestCase vs unittest** - TestCase มี Database support, SimpleTestCase ไม่มี  
✅ **setUp/tearDown** - เตรียมและทำความสะอาด Test Data ก่อน/หลังแต่ละ Test  
✅ **Test Client** - จำลอง HTTP requests สำหรับทดสอบ Views  
✅ **Model Tests** - ทดสอบ str, validation, relationships, managers  
✅ **View Tests** - ทดสอบ status code, template, context, redirect  
✅ **Form Tests** - ทดสอบ valid/invalid data, field errors  
✅ **DRF APITestCase** - ทดสอบ REST API endpoints พร้อม authentication  
✅ **factory_boy** - สร้าง Test Data อัตโนมัติด้วย Factories  
✅ **pytest-django** - รัน Tests ด้วย pytest style พร้อม fixtures  
✅ **Database Handling** - Rollback, TransactionTestCase, Performance  
✅ **Mocking** - Mock External Services ด้วย unittest.mock  
✅ **Coverage** - วัด Code Coverage ด้วย pytest-cov  
✅ **Complete Test Suite** - Tests ครบถ้วนสำหรับ Blog App  

---

## ➡️ ถัดไป: Part 075 - Django Deployment and Production

*Part 074/100+ | Python Course - Beginner to World-Class*
