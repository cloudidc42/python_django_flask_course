# Part 041: Environment Configuration
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ python-dotenv โหลด environment variables
- จัดการ config ด้วย pydantic-settings
- แยก config สำหรับ dev/staging/production
- จัดการ secrets อย่างปลอดภัย
- ตรวจสอบ config ด้วย validation
- Patterns สำหรับ config management

---

## 1. Environment Variables พื้นฐาน

```python
import os

# === อ่าน Environment Variables ===
def read_env_vars():
    # อ่านค่า (ถ้าไม่มีจะ return None)
    db_url = os.environ.get("DATABASE_URL")
    
    # อ่านค่าพร้อม default
    debug = os.environ.get("DEBUG", "false").lower() == "true"
    port = int(os.environ.get("PORT", "8000"))
    
    # อ่านค่าที่ required (raise ถ้าไม่มี)
    try:
        secret_key = os.environ["SECRET_KEY"]
    except KeyError:
        raise EnvironmentError("SECRET_KEY environment variable is required")
    
    print(f"DB URL: {db_url}")
    print(f"Debug: {debug}")
    print(f"Port: {port}")


# === ตั้งค่าชั่วคราว ===
os.environ["MY_VAR"] = "hello"
print(os.environ.get("MY_VAR"))  # hello

# ลบ variable
del os.environ["MY_VAR"]
```

---

## 2. python-dotenv

```bash
pip install python-dotenv
```

### 2.1 สร้างไฟล์ .env

```bash
# .env - ไม่ควร commit เข้า git
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
SECRET_KEY=your-very-secret-key-here
DEBUG=true
PORT=8000
REDIS_URL=redis://localhost:6379/0

# API Keys
STRIPE_API_KEY=sk_test_xxxxxxxxxxxxx
SENDGRID_API_KEY=SG.xxxxxxxxxxxxx
AWS_ACCESS_KEY_ID=AKIAXXXXXXXXXXXXXXXX
AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Email Settings
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=noreply@example.com
SMTP_PASSWORD=app-password-here
```

```ini
# .env.example - commit อันนี้เข้า git เป็น template
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
SECRET_KEY=change-this-to-a-secure-random-string
DEBUG=false
PORT=8000
REDIS_URL=redis://localhost:6379/0

STRIPE_API_KEY=
SENDGRID_API_KEY=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=

SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASSWORD=
```

### 2.2 ใช้ python-dotenv

```python
from dotenv import load_dotenv, dotenv_values
import os

# === โหลด .env ไฟล์ ===
def setup_environment():
    # โหลดจาก .env (default path)
    load_dotenv()
    
    # โหลดจาก path เฉพาะ
    load_dotenv("/path/to/custom/.env")
    
    # โหลดและ override ค่าที่มีอยู่แล้ว
    load_dotenv(override=True)
    
    # โหลดตาม environment (dev/staging/prod)
    env = os.environ.get("APP_ENV", "development")
    load_dotenv(f".env.{env}")
    load_dotenv(".env")  # fallback defaults


# === อ่าน .env เป็น dict ===
def read_env_as_dict():
    # อ่านโดยไม่โหลดเข้า os.environ
    config = dotenv_values(".env")
    print(config)  # dict
    
    # อ่านจาก stream (สำหรับ testing)
    from io import StringIO
    stream = StringIO("KEY1=value1\nKEY2=value2")
    config = dotenv_values(stream=stream)
    print(config)


# === Interpolation ===
# .env supports variable interpolation
# BASE_DIR=/app
# LOG_DIR=${BASE_DIR}/logs
# DB_NAME=myapp
# DATABASE_URL=postgresql://localhost/${DB_NAME}


# โหลด .env
load_dotenv()

# อ่านค่า
db_url = os.getenv("DATABASE_URL", "sqlite:///default.db")
debug = os.getenv("DEBUG", "false").lower() == "true"
port = int(os.getenv("PORT", "8000"))

print(f"Database: {db_url}")
print(f"Debug mode: {debug}")
print(f"Port: {port}")
```

---

## 3. pydantic-settings

```bash
pip install pydantic-settings
```

### 3.1 Basic Settings

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field, validator, PostgresDsn, RedisDsn, SecretStr
from typing import Optional, List
import os

class AppSettings(BaseSettings):
    """
    pydantic-settings อ่านค่าจาก:
    1. Arguments ที่ส่งเข้ามาตอนสร้าง
    2. Environment variables
    3. .env files
    4. Default values
    """
    
    # ===  Application Settings ===
    app_name: str = "My Application"
    app_version: str = "1.0.0"
    debug: bool = False
    port: int = Field(default=8000, ge=1, le=65535)
    
    # === Security ===
    secret_key: SecretStr  # Required, no default
    allowed_hosts: List[str] = ["localhost", "127.0.0.1"]
    
    # === Database ===
    database_url: str = "sqlite:///./app.db"
    db_pool_size: int = Field(default=5, ge=1, le=100)
    db_max_overflow: int = 10
    
    # === Redis ===
    redis_url: str = "redis://localhost:6379/0"
    
    # === Email ===
    smtp_host: str = "localhost"
    smtp_port: int = 587
    smtp_user: Optional[str] = None
    smtp_password: Optional[SecretStr] = None
    
    # === Logging ===
    log_level: str = "INFO"
    log_format: str = "json"
    
    # pydantic-settings configuration
    model_config = SettingsConfigDict(
        env_file=".env",          # โหลดจาก .env
        env_file_encoding="utf-8",
        case_sensitive=False,     # DATABASE_URL == database_url
        extra="ignore",           # ไม่ error ถ้ามี env var เกิน
    )
    
    @property
    def is_production(self) -> bool:
        return not self.debug
    
    @property
    def secret_key_value(self) -> str:
        """ดึงค่าจาก SecretStr"""
        return self.secret_key.get_secret_value()


# สร้าง settings instance
# settings = AppSettings()  # จะ raise ValidationError ถ้าไม่มี SECRET_KEY

# Mock สำหรับ demo
os.environ["SECRET_KEY"] = "demo-secret-key-for-testing"
settings = AppSettings()

print(f"App: {settings.app_name} v{settings.app_version}")
print(f"Debug: {settings.debug}")
print(f"Port: {settings.port}")
print(f"Secret (masked): {settings.secret_key}")  # "**********"
print(f"DB URL: {settings.database_url}")
```

### 3.2 Nested Settings และ Validators

```python
from pydantic_settings import BaseSettings
from pydantic import BaseModel, Field, field_validator, model_validator
from typing import Optional, Literal
import secrets

# === Nested Config Models ===
class DatabaseSettings(BaseModel):
    url: str = "sqlite:///./app.db"
    pool_size: int = Field(default=5, ge=1)
    max_overflow: int = 10
    echo_sql: bool = False
    
    @field_validator("url")
    @classmethod
    def validate_db_url(cls, v: str) -> str:
        valid_prefixes = ["sqlite", "postgresql", "mysql", "oracle"]
        if not any(v.startswith(prefix) for prefix in valid_prefixes):
            raise ValueError(f"Invalid database URL prefix. Must start with: {valid_prefixes}")
        return v


class RedisSettings(BaseModel):
    url: str = "redis://localhost:6379/0"
    max_connections: int = 50
    decode_responses: bool = True


class EmailSettings(BaseModel):
    host: str = "localhost"
    port: int = Field(default=587, ge=1, le=65535)
    use_tls: bool = True
    username: Optional[str] = None
    password: Optional[str] = None
    from_email: str = "noreply@example.com"
    from_name: str = "My App"


# === Full Application Settings ===
class Settings(BaseSettings):
    # App
    environment: Literal["development", "staging", "production"] = "development"
    debug: bool = False
    secret_key: str = Field(default_factory=lambda: secrets.token_urlsafe(32))
    
    # Nested settings (ใช้ prefix)
    database: DatabaseSettings = DatabaseSettings()
    redis: RedisSettings = RedisSettings()
    email: EmailSettings = EmailSettings()
    
    # Features flags
    feature_new_ui: bool = False
    feature_beta_api: bool = False
    
    # CORS
    cors_origins: list[str] = ["http://localhost:3000", "http://localhost:8080"]
    
    model_config = SettingsConfigDict(
        env_file=".env",
        env_nested_delimiter="__",  # DATABASE__URL=... จะ map ไปที่ database.url
        case_sensitive=False,
    )
    
    @field_validator("secret_key")
    @classmethod
    def validate_secret_key(cls, v: str) -> str:
        if len(v) < 32:
            raise ValueError("Secret key must be at least 32 characters")
        return v
    
    @model_validator(mode="after")
    def validate_production_settings(self) -> "Settings":
        """ตรวจสอบ settings ที่จำเป็นสำหรับ production"""
        if self.environment == "production":
            if self.debug:
                raise ValueError("Debug must be False in production")
            if self.database.echo_sql:
                raise ValueError("echo_sql must be False in production")
        return self
    
    @property
    def is_development(self) -> bool:
        return self.environment == "development"
    
    @property
    def is_production(self) -> bool:
        return self.environment == "production"


# ทดสอบ Nested Settings
# ตั้ง env vars ด้วย __ (double underscore) เป็น delimiter
os.environ["DATABASE__ECHO_SQL"] = "true"
os.environ["DATABASE__POOL_SIZE"] = "10"

config = Settings()
print(f"Environment: {config.environment}")
print(f"DB pool size: {config.database.pool_size}")
print(f"DB echo: {config.database.echo_sql}")
print(f"Redis URL: {config.redis.url}")
```

---

## 4. Multi-Environment Configuration

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import SecretStr
from typing import Literal
from functools import lru_cache
import os

# === Base Settings ===
class BaseAppSettings(BaseSettings):
    """Settings ที่ทุก environments มีเหมือนกัน"""
    app_name: str = "MyApp"
    app_version: str = "1.0.0"
    environment: str = "development"
    secret_key: SecretStr
    
    # Database
    database_url: str
    db_pool_size: int = 5
    
    # Redis
    redis_url: str = "redis://localhost:6379/0"
    
    # Logging
    log_level: str = "INFO"


# === Development Settings ===
class DevelopmentSettings(BaseAppSettings):
    """Development-specific settings"""
    debug: bool = True
    database_url: str = "sqlite:///./dev.db"
    db_pool_size: int = 2
    log_level: str = "DEBUG"
    
    # Development tools
    reload: bool = True      # Hot reload
    
    model_config = SettingsConfigDict(
        env_file=".env.development",
        env_file_encoding="utf-8",
    )


# === Staging Settings ===
class StagingSettings(BaseAppSettings):
    """Staging-specific settings"""
    debug: bool = False
    database_url: str = "postgresql://user:pass@staging-db:5432/myapp_staging"
    db_pool_size: int = 5
    log_level: str = "INFO"
    
    # Staging-specific
    enable_debug_toolbar: bool = True
    
    model_config = SettingsConfigDict(
        env_file=".env.staging",
        env_file_encoding="utf-8",
    )


# === Production Settings ===
class ProductionSettings(BaseAppSettings):
    """Production-specific settings"""
    debug: bool = False
    db_pool_size: int = 20
    log_level: str = "WARNING"
    
    # Performance
    cache_ttl: int = 3600          # 1 hour
    rate_limit_requests: int = 100  # per minute
    
    # Security
    secure_cookies: bool = True
    ssl_required: bool = True
    
    model_config = SettingsConfigDict(
        env_file=".env.production",
        env_file_encoding="utf-8",
    )


# === Settings Factory ===
@lru_cache()  # Cache settings instance
def get_settings() -> BaseAppSettings:
    """
    สร้าง settings ตาม APP_ENV
    @lru_cache ทำให้ singleton pattern
    """
    app_env = os.environ.get("APP_ENV", "development").lower()
    
    settings_map = {
        "development": DevelopmentSettings,
        "staging": StagingSettings,
        "production": ProductionSettings,
    }
    
    settings_class = settings_map.get(app_env)
    if not settings_class:
        raise ValueError(
            f"Unknown environment: {app_env}. "
            f"Must be one of: {list(settings_map.keys())}"
        )
    
    return settings_class()


# ===  Usage Pattern ===
# FastAPI dependency injection
def demo_settings_pattern():
    """ตัวอย่าง pattern การใช้ settings"""
    
    # ตั้ง env สำหรับ demo
    os.environ["APP_ENV"] = "development"
    os.environ["SECRET_KEY"] = "development-secret-key-at-least-32-chars-long"
    os.environ["DATABASE_URL"] = "sqlite:///./dev.db"
    
    # ดึง settings (cached)
    settings = get_settings()
    
    print(f"App: {settings.app_name}")
    print(f"Environment: {settings.environment}")
    print(f"Debug: {settings.debug}")
    print(f"DB: {settings.database_url}")
    
    # ใน FastAPI:
    # from fastapi import Depends
    # async def api_endpoint(settings: Settings = Depends(get_settings)):
    #     return {"debug": settings.debug}
    
    return settings


settings = demo_settings_pattern()
```

---

## 5. Secrets Management

```python
# === Secrets Best Practices ===
import os
import json
from pathlib import Path
from cryptography.fernet import Fernet  # pip install cryptography
import base64
import secrets


class SecretsManager:
    """จัดการ secrets อย่างปลอดภัย"""
    
    @staticmethod
    def generate_secret_key(length: int = 32) -> str:
        """สร้าง secret key ที่ปลอดภัย"""
        return secrets.token_urlsafe(length)
    
    @staticmethod
    def generate_fernet_key() -> str:
        """สร้าง Fernet encryption key"""
        return Fernet.generate_key().decode()
    
    @staticmethod
    def encrypt_value(value: str, key: str) -> str:
        """Encrypt string value"""
        f = Fernet(key.encode())
        return f.encrypt(value.encode()).decode()
    
    @staticmethod
    def decrypt_value(encrypted: str, key: str) -> str:
        """Decrypt encrypted string"""
        f = Fernet(key.encode())
        return f.decrypt(encrypted.encode()).decode()


# === Cloud Secrets Manager Integration ===
def get_secret_from_aws(secret_name: str, region: str = "ap-southeast-1") -> dict:
    """
    ดึง secrets จาก AWS Secrets Manager
    pip install boto3
    """
    import json
    try:
        import boto3
        from botocore.exceptions import ClientError
        
        client = boto3.client("secretsmanager", region_name=region)
        
        response = client.get_secret_value(SecretId=secret_name)
        
        if "SecretString" in response:
            return json.loads(response["SecretString"])
        else:
            # Binary secret
            return {"value": response["SecretBinary"]}
            
    except ImportError:
        print("boto3 not installed. Install with: pip install boto3")
        return {}
    except Exception as e:
        print(f"Error getting secret: {e}")
        return {}


# === .env.vault Pattern ===
"""
เก็บ secrets ใน encrypted vault
https://www.dotenv.org/docs/security/env-vault
"""


# === Local Secrets Store (สำหรับ Development) ===
class LocalSecretsStore:
    """Store secrets ใน encrypted file สำหรับ development"""
    
    def __init__(self, store_path: str = ".secrets.enc", key: str = None):
        self._path = Path(store_path)
        self._key = key or os.environ.get("SECRETS_ENCRYPTION_KEY")
        self._data = {}
        
        if self._path.exists() and self._key:
            self._load()
    
    def _load(self):
        """โหลด secrets จากไฟล์"""
        try:
            f = Fernet(self._key.encode())
            encrypted = self._path.read_bytes()
            decrypted = f.decrypt(encrypted)
            self._data = json.loads(decrypted)
        except Exception as e:
            print(f"Error loading secrets: {e}")
    
    def _save(self):
        """บันทึก secrets ลงไฟล์"""
        if not self._key:
            raise ValueError("Encryption key not set")
        
        f = Fernet(self._key.encode())
        data = json.dumps(self._data).encode()
        encrypted = f.encrypt(data)
        self._path.write_bytes(encrypted)
    
    def set(self, key: str, value: str):
        """เก็บ secret"""
        self._data[key] = value
        if self._key:
            self._save()
    
    def get(self, key: str, default: str = None) -> str:
        """ดึง secret"""
        return self._data.get(key, default)
    
    def delete(self, key: str):
        """ลบ secret"""
        self._data.pop(key, None)
        if self._key:
            self._save()


# Demo Secrets Manager
def demo_secrets():
    manager = SecretsManager()
    
    # สร้าง keys
    api_key = manager.generate_secret_key(24)
    enc_key = manager.generate_fernet_key()
    
    print(f"Generated API key: {api_key}")
    print(f"Generated Fernet key: {enc_key[:20]}...")
    
    # Encrypt/Decrypt
    original = "my-database-password"
    encrypted = manager.encrypt_value(original, enc_key)
    decrypted = manager.decrypt_value(encrypted, enc_key)
    
    print(f"Original: {original}")
    print(f"Encrypted: {encrypted[:30]}...")
    print(f"Decrypted: {decrypted}")
    print(f"Match: {original == decrypted}")


demo_secrets()
```

---

## 6. .gitignore สำหรับ Secrets

```bash
# .gitignore - สำคัญมาก!

# Environment files
.env
.env.local
.env.*.local
.env.development
.env.staging
.env.production

# Keep template
!.env.example

# Secrets
*.enc
.secrets
secrets/
credentials/

# Cloud credentials
.aws/credentials
gcloud-credentials.json
service-account.json

# SSL certificates
*.pem
*.key
*.crt
*.p12
```

---

## 7. Config Validation และ Testing

```python
from pydantic_settings import BaseSettings
from pydantic import validator, ValidationError
from typing import Optional
import pytest

class TestableSettings(BaseSettings):
    debug: bool = False
    database_url: str = "sqlite:///./test.db"
    secret_key: str = "test-secret-key-at-least-32-chars!!"
    max_connections: int = 10
    
    model_config = {"env_file": None}  # ไม่โหลด .env ในการทดสอบ


def test_default_settings():
    """Test default values"""
    settings = TestableSettings()
    assert settings.debug == False
    assert settings.max_connections == 10
    print("✅ test_default_settings passed")


def test_override_settings():
    """Test overriding settings"""
    settings = TestableSettings(debug=True, max_connections=20)
    assert settings.debug == True
    assert settings.max_connections == 20
    print("✅ test_override_settings passed")


def test_env_override(monkeypatch=None):
    """Test environment variable override"""
    # ใช้ monkeypatch ใน pytest
    os.environ["DEBUG"] = "true"
    os.environ["MAX_CONNECTIONS"] = "50"
    
    settings = TestableSettings()
    assert settings.debug == True
    assert settings.max_connections == 50
    
    # Cleanup
    del os.environ["DEBUG"]
    del os.environ["MAX_CONNECTIONS"]
    print("✅ test_env_override passed")


def test_invalid_settings():
    """Test validation errors"""
    try:
        settings = TestableSettings(max_connections=-1)
        # Should fail
    except ValidationError as e:
        print(f"✅ Validation caught invalid settings: {e.error_count()} errors")


# Run tests
test_default_settings()
test_override_settings()
test_env_override()
test_invalid_settings()
```

---

## 8. Complete Configuration Setup

```python
# === config.py - Complete Setup ===
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import SecretStr, Field, field_validator
from typing import Optional, List, Literal
from functools import lru_cache
import os

class DatabaseConfig(BaseSettings):
    url: str = "sqlite:///./app.db"
    pool_size: int = Field(5, ge=1, le=100)
    max_overflow: int = 10
    echo: bool = False
    
    model_config = SettingsConfigDict(env_prefix="DB_")


class RedisConfig(BaseSettings):
    url: str = "redis://localhost:6379"
    db: int = 0
    password: Optional[SecretStr] = None
    
    model_config = SettingsConfigDict(env_prefix="REDIS_")


class SecurityConfig(BaseSettings):
    secret_key: SecretStr = Field(default_factory=lambda: __import__("secrets").token_urlsafe(32))
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 7
    bcrypt_rounds: int = 12
    
    model_config = SettingsConfigDict(env_prefix="SECURITY_")


class AppConfig(BaseSettings):
    # Basic info
    name: str = "My Application"
    version: str = "1.0.0"
    description: str = ""
    env: Literal["development", "testing", "staging", "production"] = "development"
    
    # Network
    host: str = "0.0.0.0"
    port: int = Field(8000, ge=1, le=65535)
    workers: int = Field(1, ge=1)
    
    # Features
    debug: bool = False
    testing: bool = False
    docs_enabled: bool = True
    
    # Nested configs
    db: DatabaseConfig = Field(default_factory=DatabaseConfig)
    redis: RedisConfig = Field(default_factory=RedisConfig)
    security: SecurityConfig = Field(default_factory=SecurityConfig)
    
    # CORS
    cors_origins: List[str] = ["*"]
    cors_methods: List[str] = ["*"]
    cors_headers: List[str] = ["*"]
    
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        env_nested_delimiter="__",
        case_sensitive=False,
        extra="ignore",
    )
    
    @field_validator("env", mode="before")
    @classmethod
    def validate_env(cls, v: str) -> str:
        return v.lower().strip()
    
    @property
    def is_debug(self) -> bool:
        return self.debug or self.env == "development"
    
    @property
    def database_url(self) -> str:
        return self.db.url
    
    def display(self) -> str:
        """แสดง config สำหรับ logging (ไม่แสดง secrets)"""
        return (
            f"App: {self.name} v{self.version}\n"
            f"Environment: {self.env}\n"
            f"Debug: {self.debug}\n"
            f"Host: {self.host}:{self.port}\n"
            f"Database: {self.db.url.split('@')[-1] if '@' in self.db.url else self.db.url}\n"
            f"Redis: {self.redis.url}\n"
        )


# Singleton
@lru_cache
def get_app_config() -> AppConfig:
    return AppConfig()


# Usage
def demo_complete_config():
    # Mock env vars
    os.environ.setdefault("SECURITY_SECRET_KEY", "demo-secret-key-for-testing-must-be-long")
    
    config = get_app_config()
    print(config.display())
    
    # Use in application
    if config.is_debug:
        print("Running in debug mode")
    
    print(f"Database pool: {config.db.pool_size}")
    print(f"Token expire: {config.security.access_token_expire_minutes} min")


demo_complete_config()
```

---

## 9. สรุป Part 041

✅ **python-dotenv** - โหลด .env files, interpolation  
✅ **pydantic-settings** - Validated settings จาก environment  
✅ **Nested Settings** - จัดกลุ่ม config ด้วย nested models  
✅ **Multi-Environment** - dev/staging/prod settings  
✅ **Secrets Management** - การเก็บ secrets อย่างปลอดภัย  
✅ **.gitignore** - ป้องกัน secrets หลุดเข้า git  
✅ **Config Validation** - ตรวจสอบ config ก่อน startup  
✅ **Singleton Pattern** - @lru_cache สำหรับ config instance  

**Best Practices:**
- ไม่ commit .env ลง git
- ใช้ .env.example เป็น template
- ตรวจสอบ required vars ตอน startup
- ใช้ SecretStr สำหรับ passwords/keys
- แยก config ตาม environment

## ➡️ ถัดไป: Part 042 - Celery Task Queue
*Part 041/100+ | Python Course - Beginner to World-Class*
