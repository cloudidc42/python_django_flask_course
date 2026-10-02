# Part 038: Design Patterns Advanced
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจและใช้ Repository Pattern ได้
- สร้าง Service Layer แยกจาก Business Logic
- ใช้ Dependency Injection ใน Python
- ใช้ Command Pattern สำหรับ undoable operations
- สร้าง Complex Objects ด้วย Builder Pattern
- เชื่อมโยง Patterns เข้าด้วยกันในโปรเจกต์จริง

---

## 1. Repository Pattern

Repository Pattern แยก Data Access Logic ออกจาก Business Logic ทำให้ง่ายต่อการเปลี่ยน Database หรือทดสอบ

### 1.1 ปัญหาที่ Repository Pattern แก้ไข

```python
# ❌ ไม่ดี - Business Logic คลุกเคล้ากับ Database Access
from database import db

class UserService:
    def get_active_users(self):
        # Business logic ผูกติดกับ SQL โดยตรง
        users = db.execute(
            "SELECT * FROM users WHERE is_active = 1 AND deleted_at IS NULL"
        ).fetchall()
        return users
    
    def create_user(self, name, email):
        # ยากต่อการเปลี่ยน Database หรือทดสอบ
        db.execute(
            "INSERT INTO users (name, email) VALUES (?, ?)", 
            (name, email)
        )
        db.commit()
```

```python
# ✅ ดี - ใช้ Repository Pattern แยก concerns
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List, Optional
from datetime import datetime

# === Domain Model (Entity) ===
@dataclass
class User:
    """Domain Entity - ไม่ขึ้นกับ Database"""
    id: Optional[int]
    name: str
    email: str
    is_active: bool = True
    created_at: datetime = None
    
    def __post_init__(self):
        if self.created_at is None:
            self.created_at = datetime.now()
    
    def deactivate(self):
        """Business logic อยู่ใน Entity"""
        self.is_active = False
    
    def is_valid_email(self) -> bool:
        """Validation logic"""
        return "@" in self.email and "." in self.email.split("@")[1]


# === Repository Interface (Abstract) ===
class UserRepository(ABC):
    """Interface ที่ Business Logic พึ่งพา"""
    
    @abstractmethod
    def find_by_id(self, user_id: int) -> Optional[User]:
        pass
    
    @abstractmethod
    def find_by_email(self, email: str) -> Optional[User]:
        pass
    
    @abstractmethod
    def find_all_active(self) -> List[User]:
        pass
    
    @abstractmethod
    def save(self, user: User) -> User:
        pass
    
    @abstractmethod
    def delete(self, user_id: int) -> bool:
        pass
    
    @abstractmethod
    def find_all(self, page: int = 1, per_page: int = 10) -> List[User]:
        pass


# === SQLite Implementation ===
import sqlite3

class SQLiteUserRepository(UserRepository):
    """Concrete implementation สำหรับ SQLite"""
    
    def __init__(self, db_path: str):
        self.db_path = db_path
        self._init_db()
    
    def _get_connection(self):
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row  # ให้ return dict-like objects
        return conn
    
    def _init_db(self):
        """สร้าง table ถ้ายังไม่มี"""
        with self._get_connection() as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS users (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    name TEXT NOT NULL,
                    email TEXT UNIQUE NOT NULL,
                    is_active INTEGER DEFAULT 1,
                    created_at TEXT NOT NULL
                )
            """)
    
    def _row_to_user(self, row) -> User:
        """แปลง database row เป็น User entity"""
        return User(
            id=row["id"],
            name=row["name"],
            email=row["email"],
            is_active=bool(row["is_active"]),
            created_at=datetime.fromisoformat(row["created_at"])
        )
    
    def find_by_id(self, user_id: int) -> Optional[User]:
        with self._get_connection() as conn:
            row = conn.execute(
                "SELECT * FROM users WHERE id = ?", (user_id,)
            ).fetchone()
            return self._row_to_user(row) if row else None
    
    def find_by_email(self, email: str) -> Optional[User]:
        with self._get_connection() as conn:
            row = conn.execute(
                "SELECT * FROM users WHERE email = ?", (email,)
            ).fetchone()
            return self._row_to_user(row) if row else None
    
    def find_all_active(self) -> List[User]:
        with self._get_connection() as conn:
            rows = conn.execute(
                "SELECT * FROM users WHERE is_active = 1"
            ).fetchall()
            return [self._row_to_user(row) for row in rows]
    
    def find_all(self, page: int = 1, per_page: int = 10) -> List[User]:
        offset = (page - 1) * per_page
        with self._get_connection() as conn:
            rows = conn.execute(
                "SELECT * FROM users LIMIT ? OFFSET ?", 
                (per_page, offset)
            ).fetchall()
            return [self._row_to_user(row) for row in rows]
    
    def save(self, user: User) -> User:
        with self._get_connection() as conn:
            if user.id is None:
                # Insert new user
                cursor = conn.execute(
                    """INSERT INTO users (name, email, is_active, created_at) 
                       VALUES (?, ?, ?, ?)""",
                    (user.name, user.email, int(user.is_active), 
                     user.created_at.isoformat())
                )
                user.id = cursor.lastrowid
            else:
                # Update existing user
                conn.execute(
                    """UPDATE users SET name=?, email=?, is_active=? 
                       WHERE id=?""",
                    (user.name, user.email, int(user.is_active), user.id)
                )
        return user
    
    def delete(self, user_id: int) -> bool:
        with self._get_connection() as conn:
            cursor = conn.execute(
                "DELETE FROM users WHERE id = ?", (user_id,)
            )
            return cursor.rowcount > 0


# === In-Memory Implementation สำหรับ Testing ===
class InMemoryUserRepository(UserRepository):
    """Implementation สำหรับ Unit Tests - ไม่ต้องการ Database จริง"""
    
    def __init__(self):
        self._users: dict[int, User] = {}
        self._next_id = 1
    
    def find_by_id(self, user_id: int) -> Optional[User]:
        return self._users.get(user_id)
    
    def find_by_email(self, email: str) -> Optional[User]:
        for user in self._users.values():
            if user.email == email:
                return user
        return None
    
    def find_all_active(self) -> List[User]:
        return [u for u in self._users.values() if u.is_active]
    
    def find_all(self, page: int = 1, per_page: int = 10) -> List[User]:
        users = list(self._users.values())
        start = (page - 1) * per_page
        return users[start:start + per_page]
    
    def save(self, user: User) -> User:
        if user.id is None:
            user.id = self._next_id
            self._next_id += 1
        self._users[user.id] = user
        return user
    
    def delete(self, user_id: int) -> bool:
        if user_id in self._users:
            del self._users[user_id]
            return True
        return False


# ทดสอบ Repository
def demo_repository():
    # ใช้ In-Memory สำหรับ development/testing
    repo = InMemoryUserRepository()
    
    # สร้าง users
    user1 = User(id=None, name="Alice", email="alice@example.com")
    user2 = User(id=None, name="Bob", email="bob@example.com")
    
    repo.save(user1)
    repo.save(user2)
    
    print(f"All active users: {repo.find_all_active()}")
    print(f"Find by email: {repo.find_by_email('alice@example.com')}")
    
    # Deactivate user
    user1.deactivate()
    repo.save(user1)
    
    print(f"Active after deactivate: {repo.find_all_active()}")

demo_repository()
```

---

## 2. Service Layer Pattern

Service Layer รวม Business Logic และประสานงานระหว่าง Repositories ต่างๆ

```python
# === Service Layer ===
from dataclasses import dataclass
from typing import Optional
import hashlib
import secrets

@dataclass
class CreateUserDTO:
    """Data Transfer Object สำหรับรับข้อมูลจาก API"""
    name: str
    email: str
    password: str

@dataclass
class UserResponseDTO:
    """Data Transfer Object สำหรับส่งข้อมูลกลับ - ไม่มี password"""
    id: int
    name: str
    email: str
    is_active: bool

class UserAlreadyExistsError(Exception):
    pass

class UserNotFoundError(Exception):
    pass

class InvalidCredentialsError(Exception):
    pass


class PasswordHasher:
    """Helper class สำหรับ hash passwords"""
    
    @staticmethod
    def hash(password: str) -> str:
        salt = secrets.token_hex(16)
        hashed = hashlib.sha256(f"{salt}{password}".encode()).hexdigest()
        return f"{salt}:{hashed}"
    
    @staticmethod
    def verify(password: str, hashed: str) -> bool:
        salt, hash_value = hashed.split(":")
        expected = hashlib.sha256(f"{salt}{password}".encode()).hexdigest()
        return expected == hash_value


class UserService:
    """Service Layer - ประสานงาน Business Logic"""
    
    def __init__(
        self, 
        user_repository: UserRepository,
        password_hasher: PasswordHasher,
        email_service=None  # Optional dependency
    ):
        # Inject dependencies
        self._repo = user_repository
        self._hasher = password_hasher
        self._email = email_service
    
    def register_user(self, dto: CreateUserDTO) -> UserResponseDTO:
        """Register user ใหม่ - มี Business Rules"""
        
        # 1. Validate email format
        if "@" not in dto.email:
            raise ValueError(f"Invalid email: {dto.email}")
        
        # 2. Check for duplicate
        existing = self._repo.find_by_email(dto.email)
        if existing:
            raise UserAlreadyExistsError(
                f"User with email {dto.email} already exists"
            )
        
        # 3. Validate password strength
        if len(dto.password) < 8:
            raise ValueError("Password must be at least 8 characters")
        
        # 4. Hash password
        hashed_password = self._hasher.hash(dto.password)
        
        # 5. Create and save user
        user = User(
            id=None,
            name=dto.name,
            email=dto.email,
            is_active=True
        )
        # Note: ในระบบจริงจะเก็บ hashed_password ด้วย
        saved_user = self._repo.save(user)
        
        # 6. Send welcome email (optional side effect)
        if self._email:
            self._email.send_welcome(saved_user.email, saved_user.name)
        
        return self._to_response_dto(saved_user)
    
    def get_user(self, user_id: int) -> UserResponseDTO:
        """ดึง user โดย ID"""
        user = self._repo.find_by_id(user_id)
        if not user:
            raise UserNotFoundError(f"User {user_id} not found")
        return self._to_response_dto(user)
    
    def deactivate_user(self, user_id: int) -> UserResponseDTO:
        """ปิดการใช้งาน user"""
        user = self._repo.find_by_id(user_id)
        if not user:
            raise UserNotFoundError(f"User {user_id} not found")
        
        user.deactivate()
        saved_user = self._repo.save(user)
        return self._to_response_dto(saved_user)
    
    def get_active_users(self) -> list[UserResponseDTO]:
        """ดึง users ที่ active ทั้งหมด"""
        users = self._repo.find_all_active()
        return [self._to_response_dto(u) for u in users]
    
    def _to_response_dto(self, user: User) -> UserResponseDTO:
        """แปลง Entity เป็น DTO"""
        return UserResponseDTO(
            id=user.id,
            name=user.name,
            email=user.email,
            is_active=user.is_active
        )


# ทดสอบ Service Layer
def demo_service():
    repo = InMemoryUserRepository()
    hasher = PasswordHasher()
    service = UserService(repo, hasher)
    
    # Register users
    dto1 = CreateUserDTO("Alice", "alice@example.com", "password123")
    dto2 = CreateUserDTO("Bob", "bob@example.com", "securepass!")
    
    user1 = service.register_user(dto1)
    user2 = service.register_user(dto2)
    print(f"Registered: {user1}")
    print(f"Registered: {user2}")
    
    # Try duplicate registration
    try:
        service.register_user(dto1)
    except UserAlreadyExistsError as e:
        print(f"Expected error: {e}")
    
    # Get active users
    active = service.get_active_users()
    print(f"Active users: {len(active)}")
    
    # Deactivate
    service.deactivate_user(user1.id)
    active = service.get_active_users()
    print(f"Active after deactivate: {len(active)}")

demo_service()
```

---

## 3. Dependency Injection

Dependency Injection (DI) ทำให้ Code ยืดหยุ่นและทดสอบได้ง่าย

### 3.1 Manual DI

```python
# Manual Dependency Injection
from typing import Protocol

# ใช้ Protocol แทน Abstract Class (duck typing)
class EmailServiceProtocol(Protocol):
    def send_welcome(self, to: str, name: str) -> None: ...
    def send_reset_password(self, to: str, token: str) -> None: ...

class DatabaseProtocol(Protocol):
    def execute(self, query: str, params: tuple = ()) -> any: ...
    def commit(self) -> None: ...


# === Implementations ===
class SmtpEmailService:
    """Production email service"""
    def __init__(self, host: str, port: int, username: str, password: str):
        self.host = host
        self.port = port
        # จริงๆ จะ connect SMTP server
    
    def send_welcome(self, to: str, name: str) -> None:
        print(f"[SMTP] Sending welcome email to {to}")
    
    def send_reset_password(self, to: str, token: str) -> None:
        print(f"[SMTP] Sending reset email to {to}")


class MockEmailService:
    """Mock สำหรับ Testing"""
    def __init__(self):
        self.sent_emails = []
    
    def send_welcome(self, to: str, name: str) -> None:
        self.sent_emails.append({"type": "welcome", "to": to, "name": name})
    
    def send_reset_password(self, to: str, token: str) -> None:
        self.sent_emails.append({"type": "reset", "to": to, "token": token})


# === Simple DI Container ===
class Container:
    """Simple Dependency Injection Container"""
    
    def __init__(self):
        self._services = {}
        self._factories = {}
    
    def register(self, name: str, factory, singleton: bool = True):
        """ลงทะเบียน service"""
        if singleton:
            self._services[name] = None  # Lazy initialization
            self._factories[name] = factory
        else:
            self._factories[name] = factory
    
    def get(self, name: str):
        """ดึง service"""
        if name in self._services:
            # Singleton - สร้างครั้งเดียว
            if self._services[name] is None:
                self._services[name] = self._factories[name](self)
            return self._services[name]
        elif name in self._factories:
            # Transient - สร้างใหม่ทุกครั้ง
            return self._factories[name](self)
        raise KeyError(f"Service '{name}' not registered")
    
    def __getattr__(self, name: str):
        """Syntactic sugar: container.user_service"""
        if name.startswith('_'):
            raise AttributeError(name)
        return self.get(name)


# === Setup Container สำหรับ Production ===
def create_production_container() -> Container:
    container = Container()
    
    # Register repositories
    container.register(
        "user_repository",
        lambda c: SQLiteUserRepository("production.db")
    )
    
    # Register services
    container.register(
        "password_hasher",
        lambda c: PasswordHasher()
    )
    
    container.register(
        "email_service",
        lambda c: SmtpEmailService(
            host="smtp.example.com",
            port=587,
            username="noreply@example.com",
            password="secret"
        )
    )
    
    container.register(
        "user_service",
        lambda c: UserService(
            user_repository=c.get("user_repository"),
            password_hasher=c.get("password_hasher"),
            email_service=c.get("email_service")
        )
    )
    
    return container


# === Setup Container สำหรับ Testing ===
def create_test_container() -> Container:
    container = Container()
    
    mock_email = MockEmailService()
    
    container.register("user_repository", lambda c: InMemoryUserRepository())
    container.register("password_hasher", lambda c: PasswordHasher())
    container.register("email_service", lambda c: mock_email)
    container.register(
        "user_service",
        lambda c: UserService(
            user_repository=c.get("user_repository"),
            password_hasher=c.get("password_hasher"),
            email_service=c.get("email_service")
        )
    )
    
    return container


# ทดสอบ DI Container
def demo_di():
    # สร้าง container สำหรับ testing
    container = create_test_container()
    
    # ดึง service จาก container
    service = container.get("user_service")
    
    # ใช้ service
    dto = CreateUserDTO("Charlie", "charlie@example.com", "password123")
    user = service.register_user(dto)
    print(f"Created: {user}")
    
    # ตรวจสอบ email ถูกส่ง
    email_service = container.get("email_service")
    print(f"Emails sent: {email_service.sent_emails}")

demo_di()
```

### 3.2 ใช้ inject library

```python
# pip install inject
# สำหรับโปรเจกต์ขนาดใหญ่

import inject

class Config:
    db_url: str = "sqlite:///app.db"
    smtp_host: str = "localhost"


def configure_inject(binder: inject.Binder):
    """Configure injector"""
    config = Config()
    
    binder.bind(Config, config)
    binder.bind_to_constructor(
        UserRepository,
        lambda: InMemoryUserRepository()
    )
    binder.bind_to_constructor(
        PasswordHasher,
        lambda: PasswordHasher()
    )


# inject.configure(configure_inject)

class ModernUserService:
    """Service ที่ใช้ inject decorator"""
    
    # @inject.attr(UserRepository)  # Uncomment เมื่อ configure inject แล้ว
    # repo: UserRepository
    
    def __init__(self):
        # In real code with inject:
        # self.repo = inject.instance(UserRepository)
        self.repo = InMemoryUserRepository()
    
    def get_all_users(self):
        return self.repo.find_all()
```

---

## 4. Command Pattern

Command Pattern ห่อหุ้ม operation เป็น object ทำให้ undo/redo ได้

```python
from abc import ABC, abstractmethod
from typing import List, Any
from copy import deepcopy

# === Command Interface ===
class Command(ABC):
    """Base class สำหรับทุก Commands"""
    
    @abstractmethod
    def execute(self) -> Any:
        """ทำงาน"""
        pass
    
    @abstractmethod
    def undo(self) -> Any:
        """ยกเลิก"""
        pass
    
    @property
    def description(self) -> str:
        return self.__class__.__name__


# === Concrete Commands ===
class CreateUserCommand(Command):
    """Command สำหรับสร้าง User"""
    
    def __init__(self, repo: UserRepository, name: str, email: str):
        self._repo = repo
        self._name = name
        self._email = email
        self._created_user: Optional[User] = None
    
    def execute(self) -> User:
        user = User(id=None, name=self._name, email=self._email)
        self._created_user = self._repo.save(user)
        print(f"[Command] Created user: {self._created_user.name}")
        return self._created_user
    
    def undo(self) -> bool:
        if self._created_user and self._created_user.id:
            result = self._repo.delete(self._created_user.id)
            print(f"[Undo] Deleted user: {self._created_user.name}")
            return result
        return False
    
    @property
    def description(self) -> str:
        return f"Create user '{self._name}'"


class DeactivateUserCommand(Command):
    """Command สำหรับ deactivate User"""
    
    def __init__(self, repo: UserRepository, user_id: int):
        self._repo = repo
        self._user_id = user_id
        self._previous_state: Optional[User] = None
    
    def execute(self) -> Optional[User]:
        user = self._repo.find_by_id(self._user_id)
        if not user:
            return None
        
        # เก็บ state เดิมไว้สำหรับ undo
        self._previous_state = deepcopy(user)
        
        user.deactivate()
        self._repo.save(user)
        print(f"[Command] Deactivated user: {user.name}")
        return user
    
    def undo(self) -> Optional[User]:
        if self._previous_state:
            self._repo.save(self._previous_state)
            print(f"[Undo] Restored user: {self._previous_state.name}")
            return self._previous_state
        return None
    
    @property
    def description(self) -> str:
        return f"Deactivate user ID={self._user_id}"


class BatchCommand(Command):
    """Command ที่รวมหลาย Commands เป็น Transaction"""
    
    def __init__(self, commands: List[Command]):
        self._commands = commands
        self._executed: List[Command] = []
    
    def execute(self) -> List[Any]:
        results = []
        for cmd in self._commands:
            try:
                result = cmd.execute()
                results.append(result)
                self._executed.append(cmd)
            except Exception as e:
                # Rollback ทุก command ที่ execute ไปแล้ว
                print(f"[Batch] Error in {cmd.description}: {e}")
                self.undo()
                raise
        return results
    
    def undo(self) -> None:
        # Undo ย้อนกลับ (LIFO order)
        for cmd in reversed(self._executed):
            cmd.undo()
        self._executed.clear()
    
    @property
    def description(self) -> str:
        return f"Batch({', '.join(c.description for c in self._commands)})"


# === Command Invoker (Command History) ===
class CommandHistory:
    """เก็บประวัติ Commands และจัดการ Undo/Redo"""
    
    def __init__(self, max_history: int = 50):
        self._history: List[Command] = []
        self._redo_stack: List[Command] = []
        self._max_history = max_history
    
    def execute(self, command: Command) -> Any:
        """Execute command และเพิ่มเข้า history"""
        result = command.execute()
        
        self._history.append(command)
        self._redo_stack.clear()  # Clear redo stack เมื่อมี new command
        
        # จำกัดขนาด history
        if len(self._history) > self._max_history:
            self._history.pop(0)
        
        return result
    
    def undo(self) -> bool:
        """Undo command ล่าสุด"""
        if not self._history:
            print("[History] Nothing to undo")
            return False
        
        command = self._history.pop()
        command.undo()
        self._redo_stack.append(command)
        return True
    
    def redo(self) -> bool:
        """Redo command ที่ถูก undo"""
        if not self._redo_stack:
            print("[History] Nothing to redo")
            return False
        
        command = self._redo_stack.pop()
        command.execute()
        self._history.append(command)
        return True
    
    def get_history(self) -> List[str]:
        """ดู history ของ commands"""
        return [cmd.description for cmd in self._history]


# ทดสอบ Command Pattern
def demo_command():
    repo = InMemoryUserRepository()
    history = CommandHistory()
    
    # Execute commands
    cmd1 = CreateUserCommand(repo, "Dave", "dave@example.com")
    cmd2 = CreateUserCommand(repo, "Eve", "eve@example.com")
    
    user1 = history.execute(cmd1)
    user2 = history.execute(cmd2)
    
    print(f"Users: {[u.name for u in repo.find_all_active()]}")
    print(f"History: {history.get_history()}")
    
    # Deactivate
    cmd3 = DeactivateUserCommand(repo, user1.id)
    history.execute(cmd3)
    print(f"After deactivate: {[u.name for u in repo.find_all_active()]}")
    
    # Undo deactivation
    history.undo()
    print(f"After undo: {[u.name for u in repo.find_all_active()]}")
    
    # Redo deactivation
    history.redo()
    print(f"After redo: {[u.name for u in repo.find_all_active()]}")
    
    # Batch command
    batch = BatchCommand([
        CreateUserCommand(repo, "Frank", "frank@example.com"),
        CreateUserCommand(repo, "Grace", "grace@example.com"),
    ])
    history.execute(batch)
    print(f"After batch: {[u.name for u in repo.find_all_active()]}")
    
    # Undo batch
    history.undo()
    print(f"After undo batch: {[u.name for u in repo.find_all_active()]}")

demo_command()
```

---

## 5. Builder Pattern

Builder Pattern สร้าง Complex Objects ทีละขั้นตอน

```python
from dataclasses import dataclass, field
from typing import Optional, Dict, Any, List

# === Complex Object ===
@dataclass
class DatabaseConfig:
    """Complex configuration object"""
    host: str
    port: int
    database: str
    username: str
    password: str
    pool_size: int = 5
    max_overflow: int = 10
    pool_timeout: int = 30
    echo_sql: bool = False
    ssl_enabled: bool = False
    ssl_cert_path: Optional[str] = None
    charset: str = "utf8mb4"
    extra_params: Dict[str, Any] = field(default_factory=dict)
    
    def get_connection_string(self) -> str:
        """สร้าง connection string"""
        base = f"postgresql://{self.username}:{self.password}@{self.host}:{self.port}/{self.database}"
        params = []
        if self.ssl_enabled:
            params.append("sslmode=require")
        if params:
            base += "?" + "&".join(params)
        return base


# === Builder ===
class DatabaseConfigBuilder:
    """Builder สำหรับ DatabaseConfig"""
    
    def __init__(self):
        self._config = {}
        # Default values
        self._config['pool_size'] = 5
        self._config['max_overflow'] = 10
        self._config['pool_timeout'] = 30
        self._config['echo_sql'] = False
        self._config['ssl_enabled'] = False
        self._config['charset'] = "utf8mb4"
        self._config['extra_params'] = {}
    
    def host(self, host: str) -> 'DatabaseConfigBuilder':
        self._config['host'] = host
        return self  # Method chaining
    
    def port(self, port: int) -> 'DatabaseConfigBuilder':
        if port < 1 or port > 65535:
            raise ValueError(f"Invalid port: {port}")
        self._config['port'] = port
        return self
    
    def database(self, database: str) -> 'DatabaseConfigBuilder':
        self._config['database'] = database
        return self
    
    def credentials(self, username: str, password: str) -> 'DatabaseConfigBuilder':
        self._config['username'] = username
        self._config['password'] = password
        return self
    
    def pool(self, size: int = 5, max_overflow: int = 10, timeout: int = 30) -> 'DatabaseConfigBuilder':
        self._config['pool_size'] = size
        self._config['max_overflow'] = max_overflow
        self._config['pool_timeout'] = timeout
        return self
    
    def with_ssl(self, cert_path: Optional[str] = None) -> 'DatabaseConfigBuilder':
        self._config['ssl_enabled'] = True
        self._config['ssl_cert_path'] = cert_path
        return self
    
    def debug_mode(self) -> 'DatabaseConfigBuilder':
        self._config['echo_sql'] = True
        return self
    
    def extra(self, **kwargs) -> 'DatabaseConfigBuilder':
        self._config['extra_params'].update(kwargs)
        return self
    
    def build(self) -> DatabaseConfig:
        """Build และ validate"""
        required = ['host', 'port', 'database', 'username', 'password']
        missing = [f for f in required if f not in self._config]
        if missing:
            raise ValueError(f"Missing required fields: {missing}")
        
        return DatabaseConfig(**self._config)


# === Director (Optional) - รู้จักการสร้าง config แต่ละแบบ ===
class DatabaseConfigDirector:
    """Director รู้จัก recipe สำหรับ configurations ต่างๆ"""
    
    @staticmethod
    def development() -> DatabaseConfig:
        return (DatabaseConfigBuilder()
                .host("localhost")
                .port(5432)
                .database("myapp_dev")
                .credentials("dev_user", "dev_pass")
                .debug_mode()
                .pool(size=2, max_overflow=3)
                .build())
    
    @staticmethod
    def production(host: str, password: str) -> DatabaseConfig:
        return (DatabaseConfigBuilder()
                .host(host)
                .port(5432)
                .database("myapp_prod")
                .credentials("prod_user", password)
                .with_ssl()
                .pool(size=20, max_overflow=10, timeout=60)
                .build())
    
    @staticmethod
    def test() -> DatabaseConfig:
        return (DatabaseConfigBuilder()
                .host("localhost")
                .port(5432)
                .database("myapp_test")
                .credentials("test_user", "test_pass")
                .pool(size=1, max_overflow=0)
                .build())


# ทดสอบ Builder Pattern
def demo_builder():
    # Manual builder
    config = (DatabaseConfigBuilder()
              .host("db.example.com")
              .port(5432)
              .database("myapp")
              .credentials("admin", "secret")
              .with_ssl()
              .pool(size=10, max_overflow=5)
              .extra(application_name="MyApp", connect_timeout=10)
              .build())
    
    print(f"Connection: {config.get_connection_string()}")
    print(f"Pool size: {config.pool_size}")
    print(f"SSL: {config.ssl_enabled}")
    
    # ใช้ Director
    dev_config = DatabaseConfigDirector.development()
    prod_config = DatabaseConfigDirector.production("prod-db.example.com", "prod_secret")
    test_config = DatabaseConfigDirector.test()
    
    print(f"\nDev: {dev_config.database}, echo={dev_config.echo_sql}")
    print(f"Prod: {prod_config.database}, ssl={prod_config.ssl_enabled}")
    print(f"Test: {test_config.database}, pool={test_config.pool_size}")

demo_builder()
```

---

## 6. รวม Patterns เข้าด้วยกัน - Real World Example

```python
# === Full Example: Order Processing System ===
from dataclasses import dataclass, field
from typing import List, Optional
from enum import Enum
from decimal import Decimal

class OrderStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

@dataclass
class Product:
    id: int
    name: str
    price: Decimal
    stock: int

@dataclass
class OrderItem:
    product: Product
    quantity: int
    
    @property
    def subtotal(self) -> Decimal:
        return self.product.price * self.quantity

@dataclass
class Order:
    id: Optional[int]
    customer_id: int
    items: List[OrderItem] = field(default_factory=list)
    status: OrderStatus = OrderStatus.PENDING
    
    @property
    def total(self) -> Decimal:
        return sum(item.subtotal for item in self.items)
    
    def add_item(self, product: Product, quantity: int):
        if quantity > product.stock:
            raise ValueError(f"Insufficient stock for {product.name}")
        self.items.append(OrderItem(product, quantity))
    
    def can_cancel(self) -> bool:
        return self.status in [OrderStatus.PENDING, OrderStatus.CONFIRMED]


# Repository Interface
class OrderRepository(ABC):
    @abstractmethod
    def save(self, order: Order) -> Order: pass
    
    @abstractmethod
    def find_by_id(self, order_id: int) -> Optional[Order]: pass
    
    @abstractmethod
    def find_by_customer(self, customer_id: int) -> List[Order]: pass


# In-Memory Order Repository
class InMemoryOrderRepository(OrderRepository):
    def __init__(self):
        self._orders: dict[int, Order] = {}
        self._next_id = 1
    
    def save(self, order: Order) -> Order:
        if order.id is None:
            order.id = self._next_id
            self._next_id += 1
        self._orders[order.id] = order
        return order
    
    def find_by_id(self, order_id: int) -> Optional[Order]:
        return self._orders.get(order_id)
    
    def find_by_customer(self, customer_id: int) -> List[Order]:
        return [o for o in self._orders.values() 
                if o.customer_id == customer_id]


# Builder สำหรับ Order
class OrderBuilder:
    """Builder Pattern สำหรับสร้าง Order"""
    
    def __init__(self, customer_id: int):
        self._order = Order(id=None, customer_id=customer_id)
    
    def add_product(self, product: Product, quantity: int) -> 'OrderBuilder':
        self._order.add_item(product, quantity)
        return self
    
    def build(self) -> Order:
        if not self._order.items:
            raise ValueError("Order must have at least one item")
        return self._order


# Command สำหรับ Order Operations
class PlaceOrderCommand(Command):
    def __init__(self, repo: OrderRepository, order: Order):
        self._repo = repo
        self._order = order
        self._saved_order: Optional[Order] = None
    
    def execute(self) -> Order:
        self._order.status = OrderStatus.CONFIRMED
        self._saved_order = self._repo.save(self._order)
        print(f"[Order] Placed order #{self._saved_order.id}, total: {self._saved_order.total}")
        return self._saved_order
    
    def undo(self):
        if self._saved_order:
            self._saved_order.status = OrderStatus.CANCELLED
            self._repo.save(self._saved_order)
            print(f"[Undo] Cancelled order #{self._saved_order.id}")


class CancelOrderCommand(Command):
    def __init__(self, repo: OrderRepository, order_id: int):
        self._repo = repo
        self._order_id = order_id
        self._previous_status: Optional[OrderStatus] = None
    
    def execute(self) -> Optional[Order]:
        order = self._repo.find_by_id(self._order_id)
        if not order or not order.can_cancel():
            print(f"[Order] Cannot cancel order #{self._order_id}")
            return None
        
        self._previous_status = order.status
        order.status = OrderStatus.CANCELLED
        self._repo.save(order)
        print(f"[Order] Cancelled order #{self._order_id}")
        return order
    
    def undo(self):
        if self._previous_status:
            order = self._repo.find_by_id(self._order_id)
            if order:
                order.status = self._previous_status
                self._repo.save(order)
                print(f"[Undo] Restored order #{self._order_id}")


# Service Layer
class OrderService:
    """Service ที่ประสานงานทุก components"""
    
    def __init__(self, order_repo: OrderRepository):
        self._repo = order_repo
        self._history = CommandHistory()
    
    def create_order(self, customer_id: int, items: list[tuple]) -> Order:
        """สร้าง Order ใหม่ด้วย Builder"""
        # items = [(product, quantity), ...]
        builder = OrderBuilder(customer_id)
        for product, quantity in items:
            builder.add_product(product, quantity)
        
        order = builder.build()
        cmd = PlaceOrderCommand(self._repo, order)
        return self._history.execute(cmd)
    
    def cancel_order(self, order_id: int) -> Optional[Order]:
        cmd = CancelOrderCommand(self._repo, order_id)
        return self._history.execute(cmd)
    
    def undo_last_action(self) -> bool:
        return self._history.undo()
    
    def get_customer_orders(self, customer_id: int) -> List[Order]:
        return self._repo.find_by_customer(customer_id)


# Demo
def demo_full_example():
    # Setup
    products = [
        Product(1, "Laptop", Decimal("35000"), 10),
        Product(2, "Mouse", Decimal("800"), 50),
        Product(3, "Keyboard", Decimal("1500"), 30),
    ]
    
    order_repo = InMemoryOrderRepository()
    service = OrderService(order_repo)
    
    # Create order
    order1 = service.create_order(
        customer_id=1,
        items=[(products[0], 1), (products[1], 2)]
    )
    print(f"Order total: {order1.total}")
    
    # Create another order
    order2 = service.create_order(
        customer_id=1,
        items=[(products[2], 1)]
    )
    
    # Show customer orders
    customer_orders = service.get_customer_orders(1)
    print(f"Customer has {len(customer_orders)} orders")
    
    # Cancel an order
    service.cancel_order(order1.id)
    
    # Undo cancellation
    service.undo_last_action()
    
    # Check status
    order = order_repo.find_by_id(order1.id)
    print(f"Order status after undo: {order.status}")

demo_full_example()
```

---

## 7. สรุป Part 038

✅ **Repository Pattern** - แยก Data Access ออกจาก Business Logic  
✅ **Service Layer** - รวม Business Logic และประสาน Repositories  
✅ **Dependency Injection** - ทำให้ Code ยืดหยุ่นและทดสอบได้ง่าย  
✅ **Command Pattern** - ห่อหุ้ม operations เป็น Objects สำหรับ Undo/Redo  
✅ **Builder Pattern** - สร้าง Complex Objects ทีละขั้นตอน  
✅ **การรวม Patterns** - นำ Patterns มาใช้ร่วมกันในโปรเจกต์จริง  

**หลักการสำคัญ:**
- Patterns ไม่ใช่ Silver Bullet - ใช้เมื่อจำเป็น
- ทดสอบง่ายขึ้นเมื่อใช้ DI และ Repository Pattern
- Command Pattern เหมาะสำหรับระบบที่ต้องการ Audit Trail

## ➡️ ถัดไป: Part 039 - Web Scraping
*Part 038/100+ | Python Course - Beginner to World-Class*
