# Part 074: Django Testing

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เขียน Unit Tests ด้วย Django TestCase
- ทดสอบ Views ด้วย Django Test Client
- ทดสอบ API ด้วย APITestCase (DRF)
- ใช้ factory_boy สร้าง test data
- ใช้ fixtures
- เข้าใจ test database

---

## 1. ทำไมต้อง Test?

```
ประโยชน์ของ Testing:
1. หา bugs ก่อนถึง production
2. Refactor ได้อย่างมั่นใจ
3. Documentation ที่ executable
4. เพิ่ม confidence ในการ deploy

Test Pyramid:
        /\
       /E2E\        (น้อย, ช้า, แพง)
      /------\
     /Integration\  (ปานกลาง)
    /------------\
   /  Unit Tests  \ (เยอะ, เร็ว, ถูก)
  /________________\
```

---

## 2. Django TestCase พื้นฐาน

```python
# tests/test_models.py
from django.test import TestCase
from django.contrib.auth import get_user_model
from ..models import Article, Category

User = get_user_model()

class CategoryModelTest(TestCase):
    """Tests สำหรับ Category model"""
    
    def setUp(self):
        """สร้างข้อมูลก่อนทุก test"""
        self.category = Category.objects.create(
            name='Python',
            slug='python',
            description='บทความเกี่ยวกับ Python'
        )
    
    def tearDown(self):
        """ล้างข้อมูลหลังทุก test (ส่วนใหญ่ไม่ต้องเพราะ Django จัดการให้)"""
        pass
    
    def test_category_creation(self):
        """ทดสอบสร้าง Category"""
        self.assertEqual(self.category.name, 'Python')
        self.assertEqual(self.category.slug, 'python')
        self.assertIsNotNone(self.category.pk)
    
    def test_category_str(self):
        """ทดสอบ __str__ method"""
        self.assertEqual(str(self.category), 'Python')
    
    def test_category_unique_slug(self):
        """ทดสอบ slug ซ้ำ"""
        from django.db import IntegrityError
        with self.assertRaises(IntegrityError):
            Category.objects.create(name='Python 2', slug='python')
    
    def test_category_articles_count(self):
        """ทดสอบ property articles_count"""
        user = User.objects.create_user(username='testuser', password='testpass123')
        
        # สร้าง 3 บทความ
        for i in range(3):
            Article.objects.create(
                title=f'Article {i}',
                slug=f'article-{i}',
                author=user,
                category=self.category,
                content='Test content',
                status='published'
            )
        
        self.assertEqual(self.category.articles.filter(status='published').count(), 3)


class ArticleModelTest(TestCase):
    """Tests สำหรับ Article model"""
    
    @classmethod
    def setUpTestData(cls):
        """สร้างข้อมูล once สำหรับทุก test ใน class (เร็วกว่า setUp)"""
        cls.user = User.objects.create_user(
            username='author',
            email='author@test.com',
            password='testpass123'
        )
        cls.category = Category.objects.create(
            name='Django',
            slug='django'
        )
    
    def test_article_create(self):
        """ทดสอบสร้างบทความ"""
        article = Article.objects.create(
            title='Test Article',
            slug='test-article',
            author=self.user,
            category=self.category,
            content='This is test content',
            status='draft'
        )
        
        self.assertEqual(article.title, 'Test Article')
        self.assertEqual(article.author, self.user)
        self.assertEqual(article.status, 'draft')
        self.assertIsNotNone(article.created_at)
    
    def test_article_published_status(self):
        """ทดสอบ status"""
        article = Article.objects.create(
            title='Published',
            slug='published',
            author=self.user,
            content='Content',
            status='published'
        )
        self.assertTrue(article.status == 'published')
    
    def test_get_absolute_url(self):
        """ทดสอบ URL"""
        article = Article.objects.create(
            title='URL Test',
            slug='url-test',
            author=self.user,
            content='Content'
        )
        expected_url = f'/articles/{article.slug}/'
        self.assertEqual(article.get_absolute_url(), expected_url)
```

---

## 3. Django Test Client

```python
# tests/test_views.py
from django.test import TestCase, Client
from django.urls import reverse
from django.contrib.auth import get_user_model

User = get_user_model()

class ArticleViewTest(TestCase):
    
    def setUp(self):
        self.client = Client()
        self.user = User.objects.create_user(
            username='testuser',
            email='test@test.com',
            password='testpass123'
        )
        self.article = Article.objects.create(
            title='Test Article',
            slug='test-article',
            author=self.user,
            content='Test content',
            status='published'
        )
    
    # GET requests
    def test_article_list_view(self):
        """ทดสอบ list view"""
        response = self.client.get(reverse('article_list'))
        
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'articles/list.html')
        self.assertContains(response, 'Test Article')
        self.assertIn('articles', response.context)
    
    def test_article_detail_view(self):
        """ทดสอบ detail view"""
        url = reverse('article_detail', kwargs={'slug': self.article.slug})
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, self.article.title)
    
    def test_article_not_found(self):
        """ทดสอบ 404"""
        url = reverse('article_detail', kwargs={'slug': 'not-exist'})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 404)
    
    # Authentication
    def test_login_required(self):
        """ทดสอบว่า login required view redirect"""
        url = reverse('article_create')
        response = self.client.get(url)
        
        # ควร redirect ไป login
        self.assertRedirects(response, f'/login/?next={url}')
    
    def test_login_and_access(self):
        """ทดสอบ login แล้วเข้าได้"""
        self.client.login(username='testuser', password='testpass123')
        
        url = reverse('article_create')
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, 200)
    
    # POST requests
    def test_create_article(self):
        """ทดสอบสร้างบทความ"""
        self.client.login(username='testuser', password='testpass123')
        
        data = {
            'title': 'New Article',
            'slug': 'new-article',
            'content': 'New content',
            'status': 'draft'
        }
        
        response = self.client.post(reverse('article_create'), data)
        
        # ควร redirect หลัง create สำเร็จ
        self.assertEqual(response.status_code, 302)
        
        # ตรวจสอบว่าสร้างจริง
        self.assertTrue(Article.objects.filter(slug='new-article').exists())
    
    def test_create_article_invalid_data(self):
        """ทดสอบสร้างบทความด้วยข้อมูลไม่ถูกต้อง"""
        self.client.login(username='testuser', password='testpass123')
        
        # ไม่มี title
        data = {'content': 'Content only'}
        response = self.client.post(reverse('article_create'), data)
        
        # ควร return form with errors
        self.assertEqual(response.status_code, 200)
        self.assertFormError(response, 'form', 'title', 'This field is required.')
    
    def test_delete_article_by_non_owner(self):
        """ทดสอบว่า non-owner ลบไม่ได้"""
        other_user = User.objects.create_user(
            username='other',
            password='pass123'
        )
        self.client.login(username='other', password='pass123')
        
        url = reverse('article_delete', kwargs={'pk': self.article.pk})
        response = self.client.post(url)
        
        self.assertEqual(response.status_code, 403)
        self.assertTrue(Article.objects.filter(pk=self.article.pk).exists())
```

---

## 4. DRF APITestCase

```python
# tests/test_api.py
from rest_framework.test import APITestCase, APIClient
from rest_framework import status
from rest_framework.authtoken.models import Token
from django.urls import reverse
from django.contrib.auth import get_user_model

User = get_user_model()

class ArticleAPITest(APITestCase):
    """Tests สำหรับ Article API"""
    
    def setUp(self):
        # สร้าง users
        self.user = User.objects.create_user(
            username='testuser',
            email='test@test.com',
            password='testpass123'
        )
        self.admin = User.objects.create_user(
            username='admin',
            email='admin@test.com',
            password='adminpass123',
            is_staff=True
        )
        
        # สร้าง token
        self.token = Token.objects.create(user=self.user)
        
        # สร้าง test data
        self.article = Article.objects.create(
            title='Test Article',
            slug='test-article',
            author=self.user,
            content='Test content',
            status='published'
        )
    
    def authenticate(self, user=None):
        """Helper: set authentication"""
        if user is None:
            user = self.user
        token, _ = Token.objects.get_or_create(user=user)
        self.client.credentials(HTTP_AUTHORIZATION=f'Token {token.key}')
    
    def test_list_articles_unauthenticated(self):
        """ดู list โดยไม่ login ได้"""
        url = reverse('article-list')
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertIn('results', response.data)  # pagination
    
    def test_list_articles_authenticated(self):
        """ดู list เมื่อ login"""
        self.authenticate()
        url = reverse('article-list')
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
    
    def test_retrieve_article(self):
        """ดู article เดียว"""
        url = reverse('article-detail', kwargs={'pk': self.article.pk})
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], 'Test Article')
        self.assertEqual(response.data['author']['username'], 'testuser')
    
    def test_create_article_requires_auth(self):
        """สร้าง article ต้อง login"""
        url = reverse('article-list')
        data = {'title': 'New', 'content': 'Content'}
        response = self.client.post(url, data)
        
        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)
    
    def test_create_article(self):
        """สร้าง article สำเร็จ"""
        self.authenticate()
        url = reverse('article-list')
        data = {
            'title': 'New Article',
            'slug': 'new-article',
            'content': 'New content',
            'status': 'draft'
        }
        response = self.client.post(url, data, format='json')
        
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(response.data['title'], 'New Article')
        self.assertEqual(response.data['author']['username'], 'testuser')
        
        # ตรวจสอบใน DB
        self.assertTrue(Article.objects.filter(slug='new-article').exists())
    
    def test_update_article_by_owner(self):
        """แก้ไข article โดย owner"""
        self.authenticate()
        url = reverse('article-detail', kwargs={'pk': self.article.pk})
        data = {'title': 'Updated Title'}
        response = self.client.patch(url, data, format='json')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response.data['title'], 'Updated Title')
        
        # ตรวจสอบใน DB
        self.article.refresh_from_db()
        self.assertEqual(self.article.title, 'Updated Title')
    
    def test_update_article_by_non_owner(self):
        """แก้ไข article โดย user อื่น - ต้องไม่ได้"""
        other_user = User.objects.create_user(
            username='other', password='pass123'
        )
        self.authenticate(other_user)
        
        url = reverse('article-detail', kwargs={'pk': self.article.pk})
        response = self.client.patch(url, {'title': 'Hacked'}, format='json')
        
        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
    
    def test_delete_article(self):
        """ลบ article"""
        self.authenticate()
        url = reverse('article-detail', kwargs={'pk': self.article.pk})
        response = self.client.delete(url)
        
        self.assertEqual(response.status_code, status.HTTP_204_NO_CONTENT)
        self.assertFalse(Article.objects.filter(pk=self.article.pk).exists())
    
    def test_custom_action_publish(self):
        """ทดสอบ custom action: publish"""
        self.authenticate()
        
        # สร้าง draft article
        draft = Article.objects.create(
            title='Draft',
            slug='draft',
            author=self.user,
            content='Content',
            status='draft'
        )
        
        url = reverse('article-publish', kwargs={'pk': draft.pk})
        response = self.client.post(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        
        draft.refresh_from_db()
        self.assertEqual(draft.status, 'published')
    
    def test_filter_by_status(self):
        """ทดสอบ filter"""
        Article.objects.create(
            title='Draft Article', slug='draft-art',
            author=self.user, content='Content', status='draft'
        )
        
        url = reverse('article-list')
        response = self.client.get(url, {'status': 'published'})
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        # ควรเห็นแค่ published
        for article in response.data['results']:
            self.assertEqual(article['status'], 'published')
    
    def test_search(self):
        """ทดสอบ search"""
        url = reverse('article-list')
        response = self.client.get(url, {'search': 'Test'})
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertGreater(len(response.data['results']), 0)
    
    def test_pagination(self):
        """ทดสอบ pagination"""
        # สร้าง articles เยอะๆ
        for i in range(25):
            Article.objects.create(
                title=f'Article {i}',
                slug=f'article-{i}',
                author=self.user,
                content='Content',
                status='published'
            )
        
        url = reverse('article-list')
        response = self.client.get(url)
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertIn('count', response.data)
        self.assertIn('next', response.data)
        self.assertIn('results', response.data)
        
        # ตรวจสอบ page size
        self.assertLessEqual(len(response.data['results']), 20)
```

---

## 5. factory_boy

```bash
pip install factory_boy
```

```python
# factories.py
import factory
from factory.django import DjangoModelFactory
from faker import Faker
from django.contrib.auth import get_user_model
from .models import Article, Category, Comment

fake = Faker('th_TH')  # ใช้ภาษาไทย
User = get_user_model()

class UserFactory(DjangoModelFactory):
    """Factory สำหรับสร้าง User test data"""
    
    class Meta:
        model = User
    
    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.LazyAttribute(lambda obj: f'{obj.username}@test.com')
    first_name = factory.Faker('first_name', locale='th_TH')
    last_name = factory.Faker('last_name', locale='th_TH')
    password = factory.PostGenerationMethodCall('set_password', 'testpass123')
    is_active = True


class AdminFactory(UserFactory):
    """Factory สำหรับ Admin user"""
    is_staff = True
    is_superuser = True


class CategoryFactory(DjangoModelFactory):
    """Factory สำหรับสร้าง Category"""
    
    class Meta:
        model = Category
    
    name = factory.Sequence(lambda n: f'Category {n}')
    slug = factory.LazyAttribute(lambda obj: obj.name.lower().replace(' ', '-'))
    description = factory.Faker('paragraph')


class ArticleFactory(DjangoModelFactory):
    """Factory สำหรับสร้าง Article"""
    
    class Meta:
        model = Article
    
    title = factory.Faker('sentence', nb_words=6)
    slug = factory.LazyAttribute(
        lambda obj: obj.title.lower().replace(' ', '-')[:50]
    )
    author = factory.SubFactory(UserFactory)
    category = factory.SubFactory(CategoryFactory)
    content = factory.Faker('paragraphs', nb=5, as_text=True)
    excerpt = factory.Faker('paragraph')
    status = 'published'
    views_count = factory.Faker('random_int', min=0, max=10000)
    
    # ManyToMany
    @factory.post_generation
    def tags(self, create, extracted, **kwargs):
        if not create:
            return
        if extracted:
            for tag in extracted:
                self.tags.add(tag)


class DraftArticleFactory(ArticleFactory):
    """Factory สำหรับ draft article"""
    status = 'draft'


class CommentFactory(DjangoModelFactory):
    """Factory สำหรับสร้าง Comment"""
    
    class Meta:
        model = Comment
    
    article = factory.SubFactory(ArticleFactory)
    author = factory.SubFactory(UserFactory)
    content = factory.Faker('paragraph')
```

### ใช้ Factory ใน Tests

```python
# tests/test_with_factory.py
from django.test import TestCase
from rest_framework.test import APITestCase
from .factories import UserFactory, ArticleFactory, CategoryFactory, CommentFactory

class ArticleWithFactoryTest(TestCase):
    
    def setUp(self):
        self.user = UserFactory()
        self.category = CategoryFactory(name='Python', slug='python')
    
    def test_create_multiple_articles(self):
        """สร้าง articles หลายชิ้น"""
        # สร้าง 10 articles
        articles = ArticleFactory.create_batch(10, author=self.user)
        
        self.assertEqual(len(articles), 10)
        for article in articles:
            self.assertEqual(article.author, self.user)
    
    def test_draft_articles(self):
        """สร้าง draft articles"""
        from .factories import DraftArticleFactory
        draft = DraftArticleFactory(author=self.user)
        
        self.assertEqual(draft.status, 'draft')
    
    def test_article_with_comments(self):
        """สร้าง article พร้อม comments"""
        article = ArticleFactory()
        comments = CommentFactory.create_batch(5, article=article)
        
        self.assertEqual(article.comments.count(), 5)
    
    def test_build_vs_create(self):
        """build ไม่บันทึก DB, create บันทึก"""
        # build - สร้าง object แต่ไม่บันทึก DB (เร็วกว่า)
        article = ArticleFactory.build()
        self.assertIsNone(article.pk)
        
        # create - บันทึก DB
        article = ArticleFactory.create()
        self.assertIsNotNone(article.pk)
```

---

## 6. Fixtures

```bash
# สร้าง fixture จาก database ปัจจุบัน
python manage.py dumpdata articles.Article --indent 2 > articles/fixtures/articles.json
python manage.py dumpdata auth.User --indent 2 > articles/fixtures/users.json

# โหลด fixture
python manage.py loaddata articles.json
```

```json
// articles/fixtures/test_articles.json
[
    {
        "model": "articles.category",
        "pk": 1,
        "fields": {
            "name": "Python",
            "slug": "python",
            "description": "บทความ Python"
        }
    },
    {
        "model": "articles.article",
        "pk": 1,
        "fields": {
            "title": "Python Basics",
            "slug": "python-basics",
            "author": 1,
            "category": 1,
            "content": "Python content here...",
            "status": "published"
        }
    }
]
```

```python
# ใช้ fixtures ใน tests
class ArticleFixtureTest(TestCase):
    fixtures = ['users.json', 'test_articles.json']
    
    def test_article_loaded(self):
        article = Article.objects.get(slug='python-basics')
        self.assertEqual(article.title, 'Python Basics')
```

---

## 7. Test Coverage

```bash
pip install coverage

# รัน tests พร้อม coverage
coverage run manage.py test
coverage report
coverage html  # สร้าง HTML report

# .coveragerc
[run]
source = .
omit =
    */migrations/*
    */tests/*
    manage.py
    myproject/wsgi.py
    myproject/asgi.py
```

---

## 8. รัน Tests

```bash
# รัน tests ทั้งหมด
python manage.py test

# รัน tests ของ app เดียว
python manage.py test articles

# รัน test class เดียว
python manage.py test articles.tests.ArticleModelTest

# รัน test method เดียว
python manage.py test articles.tests.ArticleModelTest.test_article_create

# รัน แบบ verbose
python manage.py test --verbosity=2

# รัน แบบ parallel (เร็วกว่า)
python manage.py test --parallel
```

---

## 9. สรุป Part 074

✅ **TestCase** สำหรับ model tests มี setUp/setUpTestData/tearDown
✅ **Client** ทดสอบ views ผ่าน HTTP requests (get/post)
✅ **APITestCase** ทดสอบ DRF API endpoints
✅ **assertEqual, assertContains, assertRedirects** asserts ที่ใช้บ่อย
✅ **factory_boy** สร้าง test data ได้ง่ายและยืดหยุ่น
✅ **Fixtures** โหลดข้อมูลสำเร็จรูปสำหรับ tests
✅ **coverage** วัดว่า code ถูก test กี่ %
✅ **setUpTestData** สร้างข้อมูลครั้งเดียวสำหรับทั้ง class (เร็วกว่า setUp)

## ➡️ ถัดไป: Part 075 - Django Deployment

*Part 074/100+ | Python Course - Beginner to World-Class*
