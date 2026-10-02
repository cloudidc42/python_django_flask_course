# Part 53: Django Migrations

## เป้าหมายของบทเรียน

- เข้าใจว่า Migrations คืออะไรและทำงานอย่างไร
- ใช้ `makemigrations` และ `migrate` commands
- อ่านและเข้าใจไฟล์ migration
- ย้อนกลับ (rollback) migrations
- สร้าง data migrations
- ใช้ `squashmigrations` เพื่อรวม migrations
- แก้ปัญหาที่พบบ่อยใน migrations

---

## 1. Migrations คืออะไร?

Migrations เป็นวิธีที่ Django ใช้ในการจัดการการเปลี่ยนแปลง database schema ตามการเปลี่ยนแปลงของ models

**เหมือนกับ version control สำหรับ database:**
- บันทึกการเปลี่ยนแปลง schema
- สามารถ apply หรือ revert ได้
- ทำงานร่วมกันในทีมได้

**วงจรชีวิต Migration:**
```
1. แก้ไข models.py
       |
2. makemigrations (สร้างไฟล์ migration)
       |
3. migrate (apply การเปลี่ยนแปลงลงฐานข้อมูล)
```

---

## 2. คำสั่ง makemigrations

```bash
# สร้าง migration สำหรับทุก apps
python manage.py makemigrations

# สร้าง migration สำหรับ app เฉพาะ
python manage.py makemigrations blog

# ระบุชื่อ migration
python manage.py makemigrations blog --name add_views_count

# ดูว่า SQL จะเป็นอะไร (ไม่สร้างไฟล์)
python manage.py makemigrations --dry-run

# แสดงรายละเอียดเพิ่มเติม
python manage.py makemigrations --verbosity 3

# ตรวจสอบว่า models และ migrations ตรงกัน
python manage.py makemigrations --check
# exit code 0 = ตรง, 1 = ไม่ตรง
```

---

## 3. คำสั่ง migrate

```bash
# Apply migrations ทั้งหมดที่ยังไม่ได้ apply
python manage.py migrate

# Apply migrations ของ app เฉพาะ
python manage.py migrate blog

# Apply ไปยัง migration เฉพาะ
python manage.py migrate blog 0002

# Apply ทั้งหมดรวมถึง initial migration
python manage.py migrate blog 0001_initial

# ดู migration status
python manage.py showmigrations
python manage.py showmigrations blog

# ดู SQL ของ migration
python manage.py sqlmigrate blog 0001_initial

# ดู SQL ของ migration หลายๆ อัน
python manage.py sqlmigrate blog 0002_add_field
```

### ตัวอย่าง showmigrations output

```
blog
 [X] 0001_initial              <- applied แล้ว
 [X] 0002_post_views_count     <- applied แล้ว
 [ ] 0003_comment_model        <- ยังไม่ apply
```

---

## 4. Migration Files

### Initial Migration ที่ Django สร้าง

```python
# blog/migrations/0001_initial.py
from django.conf import settings
from django.db import migrations, models
import django.db.models.deletion

class Migration(migrations.Migration):
    """
    Migration แรกสุดสำหรับ blog app
    สร้างตารางทั้งหมดที่กำหนดใน models.py
    """
    
    # dependencies บอกว่า migration นี้ต้องรอ migration อื่นก่อน
    dependencies = [
        migrations.swappable_dependency(settings.AUTH_USER_MODEL),
    ]
    
    # operations คือการเปลี่ยนแปลงที่จะทำ
    operations = [
        # สร้างตาราง Category
        migrations.CreateModel(
            name='Category',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True)),
                ('name', models.CharField(max_length=100, verbose_name='ชื่อหมวดหมู่')),
                ('slug', models.SlugField(unique=True)),
                ('description', models.TextField(blank=True, verbose_name='คำอธิบาย')),
                ('color', models.CharField(default='#007bff', max_length=7)),
                ('created_at', models.DateTimeField(auto_now_add=True)),
            ],
            options={
                'verbose_name': 'หมวดหมู่',
                'verbose_name_plural': 'หมวดหมู่ทั้งหมด',
                'ordering': ['name'],
            },
        ),
        # สร้างตาราง Post
        migrations.CreateModel(
            name='Post',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True)),
                ('title', models.CharField(max_length=200, verbose_name='หัวข้อ')),
                ('slug', models.SlugField(max_length=200, unique=True)),
                ('content', models.TextField(verbose_name='เนื้อหา')),
                ('status', models.CharField(
                    choices=[
                        ('draft', 'แบบร่าง'),
                        ('published', 'เผยแพร่'),
                        ('archived', 'เก็บเข้าคลัง'),
                    ],
                    default='draft',
                    max_length=20
                )),
                ('created_at', models.DateTimeField(auto_now_add=True)),
                ('updated_at', models.DateTimeField(auto_now=True)),
                ('author', models.ForeignKey(
                    on_delete=django.db.models.deletion.CASCADE,
                    related_name='posts',
                    to=settings.AUTH_USER_MODEL,
                )),
                ('category', models.ForeignKey(
                    blank=True,
                    null=True,
                    on_delete=django.db.models.deletion.SET_NULL,
                    related_name='posts',
                    to='blog.category',
                )),
            ],
        ),
    ]
```

### Migration ที่เพิ่ม Field ใหม่

```python
# blog/migrations/0002_post_views_count.py
from django.db import migrations, models


class Migration(migrations.Migration):
    """
    เพิ่ม field views_count ให้กับ Post model
    """
    
    dependencies = [
        # ต้องรอ 0001_initial ก่อน
        ('blog', '0001_initial'),
    ]
    
    operations = [
        migrations.AddField(
            model_name='post',
            name='views_count',
            field=models.PositiveIntegerField(default=0, verbose_name='จำนวนครั้งที่ดู'),
        ),
    ]
```

### Migration ที่เปลี่ยนแปลง Field

```python
# blog/migrations/0003_alter_post_excerpt.py
from django.db import migrations, models


class Migration(migrations.Migration):
    """
    เพิ่ม field excerpt และเปลี่ยน max_length ของ title
    """
    
    dependencies = [
        ('blog', '0002_post_views_count'),
    ]
    
    operations = [
        # เพิ่ม field ใหม่
        migrations.AddField(
            model_name='post',
            name='excerpt',
            field=models.TextField(blank=True, default='', verbose_name='บทคัดย่อ'),
            preserve_default=False,  # ลบ default หลัง migration
        ),
        # เปลี่ยน field ที่มีอยู่
        migrations.AlterField(
            model_name='post',
            name='title',
            field=models.CharField(max_length=300, verbose_name='หัวข้อ'),
        ),
        # เปลี่ยน ordering
        migrations.AlterModelOptions(
            name='post',
            options={
                'ordering': ['-created_at'],
                'verbose_name': 'บทความ',
                'verbose_name_plural': 'บทความทั้งหมด',
            },
        ),
    ]
```

### Migration ที่ลบ Field

```python
# blog/migrations/0004_remove_post_old_field.py
from django.db import migrations


class Migration(migrations.Migration):
    """
    ลบ field ที่ไม่ใช้แล้ว
    """
    
    dependencies = [
        ('blog', '0003_alter_post_excerpt'),
    ]
    
    operations = [
        migrations.RemoveField(
            model_name='post',
            name='old_field_name',
        ),
    ]
```

---

## 5. Migration Operations ทุกประเภท

```python
from django.db import migrations, models
import django.db.models.deletion


class Migration(migrations.Migration):
    dependencies = [...]
    
    operations = [
        # === สร้าง/ลบ Model ===
        
        # สร้าง Model ใหม่
        migrations.CreateModel(
            name='NewModel',
            fields=[...],
            options={},
        ),
        
        # ลบ Model
        migrations.DeleteModel(name='OldModel'),
        
        # เปลี่ยนชื่อ Model
        migrations.RenameModel(
            old_name='OldName',
            new_name='NewName',
        ),
        
        # === จัดการ Fields ===
        
        # เพิ่ม field
        migrations.AddField(
            model_name='post',
            name='new_field',
            field=models.CharField(max_length=100, default=''),
        ),
        
        # ลบ field
        migrations.RemoveField(
            model_name='post',
            name='old_field',
        ),
        
        # เปลี่ยนแปลง field
        migrations.AlterField(
            model_name='post',
            name='existing_field',
            field=models.TextField(),
        ),
        
        # เปลี่ยนชื่อ field
        migrations.RenameField(
            model_name='post',
            old_name='old_name',
            new_name='new_name',
        ),
        
        # === จัดการ Indexes ===
        
        # เพิ่ม index
        migrations.AddIndex(
            model_name='post',
            index=models.Index(fields=['title'], name='post_title_idx'),
        ),
        
        # ลบ index
        migrations.RemoveIndex(
            model_name='post',
            name='post_title_idx',
        ),
        
        # === จัดการ Constraints ===
        
        # เพิ่ม constraint
        migrations.AddConstraint(
            model_name='post',
            constraint=models.UniqueConstraint(
                fields=['slug', 'author'],
                name='unique_post_slug_author',
            ),
        ),
        
        # ลบ constraint
        migrations.RemoveConstraint(
            model_name='post',
            name='unique_post_slug_author',
        ),
        
        # === จัดการ Meta ===
        
        # เปลี่ยน model options
        migrations.AlterModelOptions(
            name='post',
            options={'ordering': ['-created_at']},
        ),
        
        # เปลี่ยน table name
        migrations.AlterModelTable(
            name='post',
            table='blog_articles',
        ),
        
        # === SQL ตรงๆ ===
        
        # รัน SQL โดยตรง
        migrations.RunSQL(
            sql='UPDATE blog_post SET status = "published" WHERE status = "active"',
            reverse_sql='UPDATE blog_post SET status = "active" WHERE status = "published"',
        ),
        
        # รัน Python function
        migrations.RunPython(
            code=some_function,
            reverse_code=reverse_function,
        ),
    ]
```

---

## 6. ย้อนกลับ (Rollback) Migration

```bash
# ย้อนกลับไปยัง migration เฉพาะ
python manage.py migrate blog 0002
# จะ rollback จาก 0003 กลับไปที่ 0002

# ย้อนกลับทั้งหมดของ app
python manage.py migrate blog zero
# จะ rollback ทุก migration ของ blog

# ดู SQL ที่จะใช้ใน rollback
python manage.py sqlmigrate blog 0003 --backwards
```

**สิ่งสำคัญ:** Migration บางอย่าง rollback ไม่ได้ เช่น:
- `DeleteModel` - ข้อมูลหายไปแล้ว
- `RemoveField` - ข้อมูลใน field นั้นหายไปแล้ว

เพื่อให้ rollback ได้ ควรมี `reverse_code` ใน `RunPython` และ `reverse_sql` ใน `RunSQL`

---

## 7. Data Migrations

Data migrations ใช้สำหรับย้ายหรือแปลงข้อมูลใน database

### ตัวอย่าง 1: เติมข้อมูลเริ่มต้น (Initial Data)

```python
# blog/migrations/0005_add_default_categories.py
from django.db import migrations


def add_default_categories(apps, schema_editor):
    """
    เพิ่มหมวดหมู่เริ่มต้น
    ใช้ apps.get_model แทน direct import เพื่อความปลอดภัย
    (เพราะ model อาจเปลี่ยนแปลงในอนาคต)
    """
    # ใช้ historical model (model ณ ตอน migration นี้)
    Category = apps.get_model('blog', 'Category')
    
    default_categories = [
        {'name': 'เทคโนโลยี', 'slug': 'technology', 'color': '#007bff'},
        {'name': 'การเขียนโปรแกรม', 'slug': 'programming', 'color': '#28a745'},
        {'name': 'ไลฟ์สไตล์', 'slug': 'lifestyle', 'color': '#dc3545'},
        {'name': 'อาหาร', 'slug': 'food', 'color': '#fd7e14'},
    ]
    
    for cat_data in default_categories:
        Category.objects.get_or_create(
            slug=cat_data['slug'],
            defaults=cat_data
        )


def remove_default_categories(apps, schema_editor):
    """
    ลบหมวดหมู่เริ่มต้น (ใช้ใน rollback)
    """
    Category = apps.get_model('blog', 'Category')
    slugs = ['technology', 'programming', 'lifestyle', 'food']
    Category.objects.filter(slug__in=slugs).delete()


class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0004_remove_post_old_field'),
    ]
    
    operations = [
        migrations.RunPython(
            code=add_default_categories,
            reverse_code=remove_default_categories,
        ),
    ]
```

### ตัวอย่าง 2: แปลงข้อมูล (Transform Data)

สมมติว่าเราต้องการแยก field `full_name` เป็น `first_name` และ `last_name`

```python
# สมมติว่า UserProfile model มีการเปลี่ยนแปลง
# 0006_split_name.py

from django.db import migrations, models


def split_full_name(apps, schema_editor):
    """
    แยก full_name เป็น first_name และ last_name
    """
    UserProfile = apps.get_model('accounts', 'UserProfile')
    
    for profile in UserProfile.objects.all():
        if profile.full_name:
            parts = profile.full_name.split(' ', 1)
            profile.first_name = parts[0]
            profile.last_name = parts[1] if len(parts) > 1 else ''
            profile.save()


def merge_names(apps, schema_editor):
    """
    รวม first_name และ last_name กลับเป็น full_name (rollback)
    """
    UserProfile = apps.get_model('accounts', 'UserProfile')
    
    for profile in UserProfile.objects.all():
        profile.full_name = f'{profile.first_name} {profile.last_name}'.strip()
        profile.save()


class Migration(migrations.Migration):
    dependencies = [
        ('accounts', '0005_add_first_last_name'),
    ]
    
    operations = [
        # ทำ data migration
        migrations.RunPython(
            code=split_full_name,
            reverse_code=merge_names,
        ),
    ]
```

### ตัวอย่าง 3: สร้าง Slug จาก Title

```python
# blog/migrations/0007_generate_slugs.py
from django.db import migrations
from django.utils.text import slugify


def generate_slugs(apps, schema_editor):
    """
    สร้าง slug สำหรับ posts ที่ยังไม่มี slug
    """
    Post = apps.get_model('blog', 'Post')
    
    for post in Post.objects.filter(slug=''):
        # สร้าง slug จาก title
        base_slug = slugify(post.title)
        slug = base_slug
        counter = 1
        
        # ตรวจสอบว่า slug ซ้ำหรือไม่
        while Post.objects.filter(slug=slug).exclude(pk=post.pk).exists():
            slug = f'{base_slug}-{counter}'
            counter += 1
        
        post.slug = slug
        post.save()


class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0006_post_slug_not_required'),
    ]
    
    operations = [
        migrations.RunPython(
            code=generate_slugs,
            reverse_code=migrations.RunPython.noop,  # ไม่ต้อง reverse
        ),
    ]
```

---

## 8. Squash Migrations

เมื่อ migration files มีจำนวนมาก สามารถรวมให้เป็นไฟล์เดียวได้

```bash
# รวม migrations ตั้งแต่ 0001 ถึง 0010 ให้เป็น 1 ไฟล์
python manage.py squashmigrations blog 0010

# กำหนด start migration ด้วย
python manage.py squashmigrations blog 0005 0010
# รวมตั้งแต่ 0005 ถึง 0010

# ระบุชื่อไฟล์ใหม่
python manage.py squashmigrations blog 0010 --squashed-name combined_migration
```

หลังจาก squash:
1. ไฟล์ใหม่จะถูกสร้าง เช่น `0001_squashed_0010_...`
2. ทดสอบว่า migration ทำงานถูกต้อง
3. ลบไฟล์ migration เก่า (0001-0010)
4. ลบ `replaces` attribute ในไฟล์ที่ squash

---

## 9. Fake Migrations

ใช้เมื่อต้องการบันทึกว่า migration apply แล้วโดยไม่ต้องรัน operations จริง

```bash
# Mark migration ว่า apply แล้ว (ไม่รัน SQL)
python manage.py migrate blog 0003 --fake

# Mark ทุก migrations ว่า apply แล้ว
python manage.py migrate --fake

# ใช้กรณี: database มี schema อยู่แล้ว แต่ยังไม่มี migration records
python manage.py migrate --fake-initial
```

---

## 10. Multi-Database Migrations

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'primary_db',
    },
    'secondary': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'secondary_db',
    }
}
```

```python
# สร้าง router
# blog/db_router.py
class BlogRouter:
    """
    Router สำหรับ blog app ให้ใช้ secondary database
    """
    route_app_labels = {'blog'}
    
    def db_for_read(self, model, **hints):
        if model._meta.app_label in self.route_app_labels:
            return 'secondary'
        return None
    
    def db_for_write(self, model, **hints):
        if model._meta.app_label in self.route_app_labels:
            return 'secondary'
        return None
    
    def allow_migrate(self, db, app_label, model_name=None, **hints):
        if app_label in self.route_app_labels:
            return db == 'secondary'
        return None
```

```python
# settings.py
DATABASE_ROUTERS = ['blog.db_router.BlogRouter']
```

```bash
# migrate เฉพาะ secondary database
python manage.py migrate --database secondary
```

---

## 11. ปัญหาที่พบบ่อยและวิธีแก้

### ปัญหา 1: Migration Conflicts

```
CommandError: Conflicting migrations detected; multiple leaf nodes in the 
migration graph: (0003_a, 0003_b in blog).
```

**สาเหตุ:** เกิดจากทำงานหลาย branch พร้อมกันและสร้าง migration

**วิธีแก้:**
```bash
# สร้าง merge migration
python manage.py makemigrations --merge

# หรือสร้างด้วยตัวเอง
# blog/migrations/0004_merge.py
from django.db import migrations

class Migration(migrations.Migration):
    dependencies = [
        ('blog', '0003_a'),
        ('blog', '0003_b'),
    ]
    operations = []
```

### ปัญหา 2: Circular Dependency

```
ValueError: Dependency on app with no migrations: auth
```

**วิธีแก้:**
```python
# แทนที่จะใช้ model โดยตรง
from django.contrib.auth.models import User

# ใช้ string reference
author = models.ForeignKey(
    'auth.User',  # 'app_label.ModelName'
    on_delete=models.CASCADE,
)
```

### ปัญหา 3: Cannot Alter to Non-Null Field

```
django.db.utils.IntegrityError: NOT NULL constraint failed
```

**วิธีแก้:** เพิ่ม field แบบ nullable ก่อน แล้วค่อย migrate data แล้วจึง make it non-null

```bash
# Step 1: เพิ่ม field แบบ nullable
python manage.py makemigrations

# Step 2: สร้าง data migration เพื่อกรอกข้อมูล
python manage.py makemigrations blog --empty --name fill_new_field

# Step 3: เปลี่ยน field เป็น non-null
python manage.py makemigrations
```

### ปัญหา 4: Migration ไม่ detect การเปลี่ยนแปลง

**ตรวจสอบว่า:**
1. App ลงทะเบียนใน `INSTALLED_APPS` แล้ว
2. Model import ถูกต้อง
3. ไม่มี syntax error ใน models.py

```bash
python manage.py check  # ตรวจสอบ project
```

### ปัญหา 5: Database ไม่ตรงกับ Migration

```bash
# ตรวจสอบ migration status
python manage.py showmigrations

# ดู SQL ที่จะ apply
python manage.py sqlmigrate blog 0001

# ถ้า database มี schema แล้วแต่ไม่มี migration record
python manage.py migrate --fake-initial
```

### ปัญหา 6: Renaming Model อาจทำให้ข้อมูลหาย

```python
# อันตราย! Django อาจตีความเป็น Delete + Create
class OldName(models.Model):
    pass

# เปลี่ยนเป็น
class NewName(models.Model):
    pass
```

**วิธีแก้:** ใช้ `RenameModel` ใน migration โดยตรง หรือตอบ `y` เมื่อ Django ถามว่า rename หรือ delete+create

---

## 12. Best Practices

### 1. ใช้ Atomic Migrations

```python
class Migration(migrations.Migration):
    # หยุดทุกอย่างถ้าเกิด error (default)
    atomic = True
    
    # หรือปิดถ้า database ไม่รองรับ DDL transactions
    # atomic = False  # PostgreSQL รองรับ, MySQL บางส่วน
```

### 2. หลีกเลี่ยง Import Model ตรงๆ ใน Migration

```python
# ไม่ดี!
from blog.models import Post

def migration_function(apps, schema_editor):
    Post.objects.all()  # ใช้ model ปัจจุบัน ไม่ใช่ historical

# ดี!
def migration_function(apps, schema_editor):
    Post = apps.get_model('blog', 'Post')  # historical model
    Post.objects.all()
```

### 3. ทดสอบ Migration ก่อน Deploy

```bash
# ทดสอบ apply
python manage.py migrate

# ทดสอบ rollback
python manage.py migrate blog 0005  # back to 0005

# ทดสอบ apply อีกครั้ง
python manage.py migrate
```

### 4. Documentation ใน Migration

```python
class Migration(migrations.Migration):
    """
    Migration นี้:
    1. เพิ่ม field excerpt ใน Post model
    2. กรอกข้อมูล excerpt จาก content (100 ตัวแรก)
    3. ทำให้ excerpt เป็น required field
    
    ต้องรัน: python manage.py migrate blog 0008
    สร้างเมื่อ: 2026-10-01
    ผู้สร้าง: สมชาย ใจดี
    """
    ...
```

### 5. ไม่แก้ไข Migration ที่ Apply แล้ว

ถ้า migration ถูก apply ใน production แล้ว ห้ามแก้ไข ให้สร้าง migration ใหม่แทน

---

## 13. ตัวอย่างเต็ม: Migration Workflow

```bash
# 1. สร้าง initial models
# models.py มี Category, Post

python manage.py makemigrations blog --name initial
python manage.py migrate

# 2. เพิ่ม field
# เพิ่ม views_count ใน Post

python manage.py makemigrations blog --name add_views_count
python manage.py migrate

# 3. สร้าง data migration
python manage.py makemigrations blog --empty --name seed_categories
# แก้ไขไฟล์ migration ที่สร้าง

python manage.py migrate

# 4. ตรวจสอบ
python manage.py showmigrations blog

# 5. ทดสอบ rollback
python manage.py migrate blog 0002
python manage.py showmigrations blog  # 0003 ควรยังไม่ apply

# 6. Apply กลับ
python manage.py migrate
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Schema Evolution
เริ่มจาก model ง่ายๆ แล้วค่อยๆ เพิ่ม:
1. สร้าง `Product` model ที่มี name, price, stock
2. เพิ่ม field `description` และ `category` (ForeignKey)
3. เพิ่ม index ที่ `price` field
4. เปลี่ยน `stock` จาก IntegerField เป็น PositiveIntegerField
5. ทดสอบ rollback ไปแต่ละขั้น

### แบบฝึกหัดที่ 2: Data Migration
สร้าง data migration ที่:
1. เพิ่มหมวดหมู่สินค้าเริ่มต้น 5 หมวด
2. สร้าง slug สำหรับ products ที่ไม่มี slug
3. ทดสอบทั้ง forward และ reverse migration

### แบบฝึกหัดที่ 3: Squash
1. สร้าง migrations 5-10 ไฟล์จาก blog app
2. Squash ทั้งหมดเป็นไฟล์เดียว
3. ทดสอบว่า squashed migration ทำงานถูกต้อง

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- Migrations คืออะไรและวิธีการทำงาน
- คำสั่ง `makemigrations` และ `migrate`
- โครงสร้างและ operations ของไฟล์ migration
- การย้อนกลับ migration
- Data migrations สำหรับ transform ข้อมูล
- Squash migrations เพื่อรวมไฟล์
- ปัญหาที่พบบ่อยและวิธีแก้

---

## บทถัดไป

➡️ **[Part 54: Django Views](part-054.md)** - เรียนรู้ Function-Based Views, Class-Based Views, และ Generic Views
