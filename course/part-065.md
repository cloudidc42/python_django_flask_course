# Part 065: DRF Serializers

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจการทำงานของ Serializers ใน DRF
- สร้าง ModelSerializer แบบต่างๆ
- ทำ Nested Serializers
- ใช้ SerializerMethodField
- ทำ validation ใน serializer
- Override create() และ update()

---

## 1. Serializer คืออะไร?

Serializer ทำหน้าที่:
1. **Serialization**: แปลง Python object (Model instance) เป็น JSON
2. **Deserialization**: แปลง JSON เป็น Python object พร้อม validate
3. **Validation**: ตรวจสอบข้อมูลที่รับเข้ามา

```
Client                    Django/DRF                   Database
  |                           |                            |
  |-- GET /api/articles/ -->  |                            |
  |                           |-- Article.objects.all() -> |
  |                           |<- QuerySet ----------------|
  |                           |
  |                    Serializer (→ JSON)
  |                           |
  |<-- JSON response ---------|
  
  |-- POST /api/articles/ --> |
  |    (JSON data)            |
  |                    Serializer (validate + → Python)
  |                           |-- article.save() --------> |
  |<-- 201 Created ---------- |
```

---

## 2. Serializer พื้นฐาน

```python
# serializers.py
from rest_framework import serializers

# Serializer ธรรมดา (ไม่ผูกกับ Model)
class ContactSerializer(serializers.Serializer):
    name = serializers.CharField(max_length=100)
    email = serializers.EmailField()
    message = serializers.CharField(min_length=10)
    phone = serializers.CharField(max_length=20, required=False, allow_blank=True)
    
    def validate_name(self, value):
        """Validate field name"""
        if len(value.strip()) < 2:
            raise serializers.ValidationError('ชื่อต้องมีอย่างน้อย 2 ตัวอักษร')
        return value.strip()
    
    def validate(self, attrs):
        """Cross-field validation"""
        name = attrs.get('name', '')
        message = attrs.get('message', '')
        if name.lower() in message.lower():
            raise serializers.ValidationError('ข้อความไม่ควรมีชื่อซ้ำ')
        return attrs
```

```python
# ใช้งาน Serializer
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class ContactAPIView(APIView):
    def post(self, request):
        serializer = ContactSerializer(data=request.data)
        
        if serializer.is_valid():
            # ดึงข้อมูลที่ผ่าน validate
            data = serializer.validated_data
            name = data['name']
            email = data['email']
            
            # ประมวลผล...
            return Response({'message': 'ส่งสำเร็จ'}, status=status.HTTP_200_OK)
        
        # ส่ง errors กลับ
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

---

## 3. ModelSerializer

```python
from rest_framework import serializers
from django.contrib.auth import get_user_model
from .models import Article, Category, Comment, Tag

User = get_user_model()

class UserSerializer(serializers.ModelSerializer):
    """Serializer สำหรับ User"""
    
    # เพิ่ม field ที่ไม่มีใน model
    full_name = serializers.SerializerMethodField()
    
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'first_name', 'last_name', 'full_name']
        # read_only_fields ป้องกันการแก้ไข
        read_only_fields = ['id']
        # extra_kwargs กำหนด options เพิ่มเติม
        extra_kwargs = {
            'email': {'required': True},
            'password': {'write_only': True},  # ไม่ส่งกลับไป client
        }
    
    def get_full_name(self, obj):
        return obj.get_full_name() or obj.username


class CategorySerializer(serializers.ModelSerializer):
    """Serializer สำหรับ Category"""
    
    # เพิ่ม field พิเศษ
    articles_count = serializers.IntegerField(read_only=True)
    
    class Meta:
        model = Category
        fields = ['id', 'name', 'slug', 'description', 'articles_count']
        read_only_fields = ['id']
    
    def validate_slug(self, value):
        """ตรวจสอบ slug"""
        import re
        if not re.match(r'^[a-z0-9-]+$', value):
            raise serializers.ValidationError('Slug ใช้ได้เฉพาะตัวพิมพ์เล็ก ตัวเลข และ -')
        return value


class TagSerializer(serializers.ModelSerializer):
    class Meta:
        model = Tag
        fields = ['id', 'name', 'slug']
        read_only_fields = ['id']
```

---

## 4. Nested Serializers

```python
# Nested Serializer - แสดงข้อมูล related objects แบบ embedded

class CommentSerializer(serializers.ModelSerializer):
    # Nested user info (read-only)
    author = UserSerializer(read_only=True)
    
    class Meta:
        model = Comment
        fields = ['id', 'author', 'content', 'created_at']
        read_only_fields = ['id', 'created_at']


class ArticleDetailSerializer(serializers.ModelSerializer):
    """Serializer แสดงรายละเอียดบทความพร้อม nested objects"""
    
    # Nested objects (read-only)
    author = UserSerializer(read_only=True)
    category = CategorySerializer(read_only=True)
    tags = TagSerializer(many=True, read_only=True)
    comments = CommentSerializer(many=True, read_only=True)
    
    # Write-only fields สำหรับรับ IDs
    author_id = serializers.PrimaryKeyRelatedField(
        queryset=User.objects.all(),
        source='author',
        write_only=True
    )
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(),
        source='category',
        write_only=True,
        required=False,
        allow_null=True
    )
    tag_ids = serializers.PrimaryKeyRelatedField(
        queryset=Tag.objects.all(),
        source='tags',
        many=True,
        write_only=True,
        required=False
    )
    
    # Computed fields
    comments_count = serializers.SerializerMethodField()
    reading_time = serializers.SerializerMethodField()
    
    class Meta:
        model = Article
        fields = [
            'id', 'title', 'slug', 
            'author', 'author_id',
            'category', 'category_id',
            'tags', 'tag_ids',
            'content', 'excerpt', 'thumbnail',
            'status', 'views_count', 
            'comments', 'comments_count',
            'reading_time',
            'created_at', 'updated_at'
        ]
        read_only_fields = ['id', 'views_count', 'created_at', 'updated_at']
    
    def get_comments_count(self, obj):
        return obj.comments.count()
    
    def get_reading_time(self, obj):
        """คำนวณเวลาอ่าน (นาที)"""
        words = len(obj.content.split())
        minutes = max(1, round(words / 200))  # อ่านเฉลี่ย 200 คำ/นาที
        return minutes
```

---

## 5. SerializerMethodField

```python
class ArticleSerializer(serializers.ModelSerializer):
    # SerializerMethodField - ใช้ method ในการคำนวณค่า
    
    # ชื่อ field คือ 'thumbnail_url'
    # Django จะเรียก method 'get_thumbnail_url'
    thumbnail_url = serializers.SerializerMethodField()
    
    # หรือระบุ method_name เอง
    author_info = serializers.SerializerMethodField(method_name='get_author_details')
    
    # ส่ง context ไปด้วย
    can_edit = serializers.SerializerMethodField()
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'thumbnail_url', 'author_info', 'can_edit']
    
    def get_thumbnail_url(self, obj):
        """สร้าง absolute URL สำหรับ thumbnail"""
        if obj.thumbnail:
            request = self.context.get('request')
            if request:
                return request.build_absolute_uri(obj.thumbnail.url)
            return obj.thumbnail.url
        # ส่ง default thumbnail URL
        request = self.context.get('request')
        if request:
            return request.build_absolute_uri('/static/images/default.jpg')
        return None
    
    def get_author_details(self, obj):
        """ข้อมูล author แบบ custom"""
        author = obj.author
        return {
            'id': author.id,
            'name': author.get_full_name() or author.username,
            'avatar': self.get_avatar_url(author),
        }
    
    def get_avatar_url(self, user):
        """Helper method"""
        if hasattr(user, 'profile') and user.profile.avatar:
            return user.profile.avatar.url
        return None
    
    def get_can_edit(self, obj):
        """ตรวจสอบว่า user ปัจจุบันแก้ไขได้ไหม"""
        request = self.context.get('request')
        if not request or not request.user.is_authenticated:
            return False
        return obj.author == request.user or request.user.is_staff
```

---

## 6. Validation ใน Serializer

```python
from rest_framework import serializers
from django.contrib.auth import get_user_model

User = get_user_model()

class UserRegistrationSerializer(serializers.ModelSerializer):
    """Serializer สำหรับ registration"""
    
    password = serializers.CharField(
        write_only=True,
        min_length=8,
        style={'input_type': 'password'}  # สำหรับ Browsable API
    )
    password_confirm = serializers.CharField(
        write_only=True,
        style={'input_type': 'password'}
    )
    
    class Meta:
        model = User
        fields = ['username', 'email', 'first_name', 'last_name', 'password', 'password_confirm']
    
    # Field-level validation
    def validate_username(self, value):
        """Validate username"""
        if User.objects.filter(username__iexact=value).exists():
            raise serializers.ValidationError('Username นี้ถูกใช้แล้ว')
        
        if len(value) < 3:
            raise serializers.ValidationError('Username ต้องมีอย่างน้อย 3 ตัวอักษร')
        
        import re
        if not re.match(r'^[a-zA-Z0-9_]+$', value):
            raise serializers.ValidationError('Username ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _')
        
        return value.lower()
    
    def validate_email(self, value):
        """Validate email"""
        if User.objects.filter(email__iexact=value).exists():
            raise serializers.ValidationError('อีเมลนี้ถูกใช้แล้ว')
        return value.lower()
    
    def validate_password(self, value):
        """Validate password strength"""
        import re
        errors = []
        
        if not re.search(r'[A-Z]', value):
            errors.append('ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว')
        if not re.search(r'[a-z]', value):
            errors.append('ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว')
        if not re.search(r'[0-9]', value):
            errors.append('ต้องมีตัวเลขอย่างน้อย 1 ตัว')
        
        if errors:
            raise serializers.ValidationError(errors)
        
        return value
    
    # Object-level validation (cross-field)
    def validate(self, attrs):
        """ตรวจสอบ password match"""
        password = attrs.get('password')
        password_confirm = attrs.pop('password_confirm', None)  # ลบออกจาก attrs
        
        if password != password_confirm:
            raise serializers.ValidationError({
                'password_confirm': 'รหัสผ่านไม่ตรงกัน'
            })
        
        return attrs
    
    def create(self, validated_data):
        """สร้าง user พร้อม hash password"""
        user = User.objects.create_user(
            username=validated_data['username'],
            email=validated_data['email'],
            first_name=validated_data.get('first_name', ''),
            last_name=validated_data.get('last_name', ''),
            password=validated_data['password']
        )
        return user
```

---

## 7. Override create() และ update()

```python
class ArticleSerializer(serializers.ModelSerializer):
    tags = serializers.PrimaryKeyRelatedField(
        queryset=Tag.objects.all(),
        many=True,
        required=False
    )
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'category', 'tags', 'status']
    
    def create(self, validated_data):
        """Override create สำหรับ ManyToMany fields"""
        # ดึง tags ออกมาก่อน (ManyToMany ต้องจัดการแยก)
        tags = validated_data.pop('tags', [])
        
        # กำหนด author จาก context
        validated_data['author'] = self.context['request'].user
        
        # สร้าง article
        article = Article.objects.create(**validated_data)
        
        # กำหนด tags
        article.tags.set(tags)
        
        return article
    
    def update(self, instance, validated_data):
        """Override update"""
        tags = validated_data.pop('tags', None)
        
        # update fields ทั้งหมด
        for attr, value in validated_data.items():
            setattr(instance, attr, value)
        instance.save()
        
        # update tags ถ้ามี
        if tags is not None:
            instance.tags.set(tags)
        
        return instance


class ArticleWithCommentsSerializer(serializers.ModelSerializer):
    """Serializer พร้อม writable nested objects"""
    
    comments = CommentSerializer(many=True, required=False)
    
    class Meta:
        model = Article
        fields = ['id', 'title', 'content', 'comments']
    
    def create(self, validated_data):
        """สร้าง article พร้อม comments"""
        comments_data = validated_data.pop('comments', [])
        
        article = Article.objects.create(**validated_data)
        
        for comment_data in comments_data:
            Comment.objects.create(
                article=article,
                author=self.context['request'].user,
                **comment_data
            )
        
        return article
    
    def update(self, instance, validated_data):
        """แก้ไข article พร้อม comments"""
        comments_data = validated_data.pop('comments', None)
        
        # update article fields
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.save()
        
        if comments_data is not None:
            # ลบ comments เดิมและสร้างใหม่
            instance.comments.all().delete()
            for comment_data in comments_data:
                Comment.objects.create(
                    article=instance,
                    author=self.context['request'].user,
                    **comment_data
                )
        
        return instance
```

---

## 8. Dynamic Fields Serializer

```python
class DynamicFieldsModelSerializer(serializers.ModelSerializer):
    """Serializer ที่ client เลือก fields ได้
    
    ใช้งาน:
    GET /api/articles/?fields=id,title,excerpt
    """
    
    def __init__(self, *args, **kwargs):
        # ดึง fields parameter
        fields = kwargs.pop('fields', None)
        
        super().__init__(*args, **kwargs)
        
        if fields is not None:
            # กรองเฉพาะ fields ที่ต้องการ
            allowed = set(fields)
            existing = set(self.fields)
            for field_name in existing - allowed:
                self.fields.pop(field_name)
    
    @classmethod
    def from_request(cls, request, *args, **kwargs):
        """สร้าง serializer จาก request"""
        fields_param = request.query_params.get('fields')
        if fields_param:
            fields = [f.strip() for f in fields_param.split(',')]
            kwargs['fields'] = fields
        return cls(*args, **kwargs)


class ArticleSerializer(DynamicFieldsModelSerializer):
    class Meta:
        model = Article
        fields = ['id', 'title', 'slug', 'excerpt', 'content', 'status', 'created_at']


# ใช้งานใน view
class ArticleViewSet(viewsets.ModelViewSet):
    def get_serializer(self, *args, **kwargs):
        kwargs['context'] = self.get_serializer_context()
        return ArticleSerializer.from_request(self.request, *args, **kwargs)
```

---

## 9. Serializer Relations

```python
class ArticleSerializer(serializers.ModelSerializer):
    # วิธีต่างๆ ในการ represent relations:
    
    # 1. PrimaryKeyRelatedField - แสดง/รับ ID
    category = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all()
    )
    
    # 2. StringRelatedField - แสดง __str__() 
    author = serializers.StringRelatedField(read_only=True)
    
    # 3. SlugRelatedField - แสดง field ใดก็ได้
    category_slug = serializers.SlugRelatedField(
        queryset=Category.objects.all(),
        slug_field='slug',
        source='category'
    )
    
    # 4. HyperlinkedRelatedField - แสดง URL
    author_url = serializers.HyperlinkedRelatedField(
        view_name='user-detail',
        read_only=True,
        source='author'
    )
    
    # 5. Nested Serializer - แสดงข้อมูล object ทั้งหมด
    category_detail = CategorySerializer(source='category', read_only=True)
    
    class Meta:
        model = Article
        fields = [
            'id', 'title',
            'category',          # ID
            'author',            # __str__
            'category_slug',     # slug
            'author_url',        # URL
            'category_detail',   # nested object
        ]
```

---

## 10. สรุป Part 065

✅ **Serializer** แปลง Model instances เป็น JSON และ validate ข้อมูล
✅ **ModelSerializer** สร้าง serializer จาก Model อัตโนมัติ ลดโค้ด
✅ **Nested Serializers** แสดงข้อมูล related objects แบบ embedded
✅ **SerializerMethodField** เพิ่ม computed fields ด้วย method
✅ **Field-level validation** ใช้ `validate_<fieldname>` method
✅ **Object-level validation** ใช้ `validate()` method สำหรับ cross-field
✅ **create()/update()** override เพื่อจัดการ ManyToMany และ nested objects
✅ **write_only** ป้องกันการส่ง sensitive data กลับ เช่น password

## ➡️ ถัดไป: Part 066 - DRF Views

*Part 065/100+ | Python Course - Beginner to World-Class*
