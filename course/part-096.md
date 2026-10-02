# Part 096: Microservices Architecture

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจความแตกต่างระหว่าง Microservices กับ Monolith Architecture
- ออกแบบและสร้าง Microservices ด้วย Python/FastAPI
- จัดการ Service Communication ผ่าน REST, gRPC และ Message Queues
- ใช้งาน Service Discovery และ API Gateway Pattern
- สร้าง Multi-service environment ด้วย Docker Compose

---

## 1. Microservices vs Monolith Architecture

### Monolith Architecture

แอปพลิเคชันแบบ Monolith รวมทุกอย่างไว้ในโค้ดฐานเดียว ง่ายต่อการพัฒนาในช่วงแรก แต่ยากต่อการ scale และดูแลรักษาเมื่อโปรเจกต์ใหญ่ขึ้น

```python
# monolith_app.py - แอปพลิเคชันแบบ Monolith ทั่วไป
from fastapi import FastAPI, HTTPException, Depends
from sqlalchemy import create_engine, Column, Integer, String, Float, DateTime
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from datetime import datetime
from typing import List, Optional
import smtplib
import stripe  # payment processing
import boto3   # file storage

app = FastAPI(title="E-commerce Monolith")

# ทุกอย่างอยู่ในแอปเดียว
# - User management
# - Product catalog
# - Order processing
# - Payment
# - Notification
# - File storage
# ปัญหา: เมื่อระบบหนึ่งล่ม ทั้งหมดล่ม

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True)
    name = Column(String)

class Product(Base):
    __tablename__ = "products"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    price = Column(Float)
    stock = Column(Integer)

class Order(Base):
    __tablename__ = "orders"
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer)
    product_id = Column(Integer)
    quantity = Column(Integer)
    total = Column(Float)
    status = Column(String, default="pending")
    created_at = Column(DateTime, default=datetime.utcnow)

# ทุก logic รวมกันอยู่ใน endpoints เดียวกัน
@app.post("/orders/")
async def create_order(user_id: int, product_id: int, quantity: int, db: Session = Depends()):
    # 1. ตรวจสอบ user
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    
    # 2. ตรวจสอบ product และ stock
    product = db.query(Product).filter(Product.id == product_id).first()
    if not product or product.stock < quantity:
        raise HTTPException(status_code=400, detail="Product not available")
    
    # 3. คำนวณราคา
    total = product.price * quantity
    
    # 4. ประมวลผลการชำระเงิน (tight coupling)
    # stripe.charge(...)
    
    # 5. อัพเดต stock
    product.stock -= quantity
    
    # 6. สร้าง order
    order = Order(user_id=user_id, product_id=product_id, quantity=quantity, total=total)
    db.add(order)
    
    # 7. ส่ง email notification (tight coupling)
    # smtplib.send_email(user.email, ...)
    
    db.commit()
    return {"order_id": order.id, "total": total}
```

### Microservices Architecture

```python
# microservices_overview.py - แสดงโครงสร้าง Microservices

"""
โครงสร้าง Microservices สำหรับ E-commerce:

┌─────────────────────────────────────────────────────────┐
│                    API Gateway (Port 8000)                │
│            nginx / Kong / custom FastAPI gateway          │
└───────────────────────────┬─────────────────────────────┘
                             │
        ┌────────────────────┼─────────────────────┐
        │                    │                     │
┌───────▼──────┐   ┌────────▼──────┐   ┌─────────▼─────┐
│ User Service │   │Product Service│   │ Order Service  │
│  Port 8001   │   │   Port 8002   │   │   Port 8003    │
│  DB: users   │   │  DB: products │   │  DB: orders    │
└──────────────┘   └───────────────┘   └───────────────┘
        
┌──────────────┐   ┌───────────────┐   ┌───────────────┐
│Payment Svc   │   │Notification   │   │ File Storage  │
│  Port 8004   │   │   Port 8005   │   │   Port 8006   │
│  DB: payment │   │  DB: notifs   │   │  S3/MinIO     │
└──────────────┘   └───────────────┘   └───────────────┘

การสื่อสารระหว่าง Services:
- Synchronous: REST API / gRPC
- Asynchronous: Message Queue (RabbitMQ / Kafka)
"""

# ข้อดีของ Microservices
advantages = {
    "independent_deployment": "deploy แต่ละ service แยกกันได้",
    "technology_diversity": "แต่ละ service ใช้ tech stack ที่เหมาะสมได้",
    "fault_isolation": "service หนึ่งล่มไม่กระทบ service อื่น",
    "scalability": "scale เฉพาะ service ที่ต้องการได้",
    "team_autonomy": "แต่ละทีมดูแล service ของตัวเองได้อิสระ",
}

# ข้อเสียของ Microservices
disadvantages = {
    "complexity": "ซับซ้อนกว่า monolith มาก",
    "network_overhead": "การสื่อสารผ่าน network มี latency",
    "data_consistency": "ยากต่อการรักษา consistency ข้าม services",
    "debugging": "ยากต่อการ debug ปัญหาที่เกิดข้าม services",
    "operational_overhead": "ต้องการ infrastructure ที่ซับซ้อนกว่า",
}
```

---

## 2. สร้าง Microservices ด้วย FastAPI

### User Service

```python
# services/user_service/main.py
from fastapi import FastAPI, HTTPException, Depends
from sqlalchemy import create_engine, Column, Integer, String, Boolean, DateTime
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from pydantic import BaseModel, EmailStr
from datetime import datetime
from passlib.context import CryptContext
from typing import Optional
import httpx
import os

app = FastAPI(title="User Service", version="1.0.0")

DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://user:password@user-db:5432/users")
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


class UserModel(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    name = Column(String, nullable=False)
    hashed_password = Column(String, nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)


Base.metadata.create_all(bind=engine)


# Pydantic schemas
class UserCreate(BaseModel):
    email: EmailStr
    name: str
    password: str


class UserResponse(BaseModel):
    id: int
    email: str
    name: str
    is_active: bool
    created_at: datetime
    
    class Config:
        from_attributes = True


class UserUpdate(BaseModel):
    name: Optional[str] = None
    is_active: Optional[bool] = None


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)


@app.get("/health")
async def health_check():
    """Health check endpoint สำหรับ service discovery"""
    return {"status": "healthy", "service": "user-service", "timestamp": datetime.utcnow()}


@app.post("/users/", response_model=UserResponse, status_code=201)
async def create_user(user_data: UserCreate, db: Session = Depends(get_db)):
    """สร้าง user ใหม่"""
    # ตรวจสอบ email ซ้ำ
    existing_user = db.query(UserModel).filter(UserModel.email == user_data.email).first()
    if existing_user:
        raise HTTPException(status_code=400, detail="Email already registered")
    
    user = UserModel(
        email=user_data.email,
        name=user_data.name,
        hashed_password=hash_password(user_data.password)
    )
    db.add(user)
    db.commit()
    db.refresh(user)
    return user


@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int, db: Session = Depends(get_db)):
    """ดึงข้อมูล user ด้วย ID"""
    user = db.query(UserModel).filter(UserModel.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user


@app.put("/users/{user_id}", response_model=UserResponse)
async def update_user(user_id: int, user_data: UserUpdate, db: Session = Depends(get_db)):
    """อัพเดทข้อมูล user"""
    user = db.query(UserModel).filter(UserModel.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    
    if user_data.name is not None:
        user.name = user_data.name
    if user_data.is_active is not None:
        user.is_active = user_data.is_active
    
    db.commit()
    db.refresh(user)
    return user


@app.post("/users/verify-password")
async def verify_user_password(email: str, password: str, db: Session = Depends(get_db)):
    """ตรวจสอบรหัสผ่าน (สำหรับ authentication service)"""
    user = db.query(UserModel).filter(UserModel.email == email).first()
    if not user or not verify_password(password, user.hashed_password):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    return {"user_id": user.id, "email": user.email, "name": user.name}
```

### Product Service

```python
# services/product_service/main.py
from fastapi import FastAPI, HTTPException, Depends, Query
from sqlalchemy import create_engine, Column, Integer, String, Float, Text, DateTime
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional, List
import os

app = FastAPI(title="Product Service", version="1.0.0")

DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://user:password@product-db:5432/products")
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()


class ProductModel(Base):
    __tablename__ = "products"
    
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, nullable=False)
    description = Column(Text)
    price = Column(Float, nullable=False)
    stock = Column(Integer, default=0)
    category = Column(String)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)


Base.metadata.create_all(bind=engine)


class ProductCreate(BaseModel):
    name: str
    description: Optional[str] = None
    price: float = Field(gt=0)
    stock: int = Field(ge=0, default=0)
    category: Optional[str] = None


class ProductResponse(BaseModel):
    id: int
    name: str
    description: Optional[str]
    price: float
    stock: int
    category: Optional[str]
    created_at: datetime
    
    class Config:
        from_attributes = True


class StockUpdate(BaseModel):
    quantity: int  # บวก = เพิ่ม stock, ลบ = ลด stock


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "product-service"}


@app.post("/products/", response_model=ProductResponse, status_code=201)
async def create_product(product_data: ProductCreate, db: Session = Depends(get_db)):
    product = ProductModel(**product_data.model_dump())
    db.add(product)
    db.commit()
    db.refresh(product)
    return product


@app.get("/products/", response_model=List[ProductResponse])
async def list_products(
    skip: int = 0,
    limit: int = 20,
    category: Optional[str] = None,
    db: Session = Depends(get_db)
):
    query = db.query(ProductModel)
    if category:
        query = query.filter(ProductModel.category == category)
    return query.offset(skip).limit(limit).all()


@app.get("/products/{product_id}", response_model=ProductResponse)
async def get_product(product_id: int, db: Session = Depends(get_db)):
    product = db.query(ProductModel).filter(ProductModel.id == product_id).first()
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    return product


@app.patch("/products/{product_id}/stock")
async def update_stock(product_id: int, stock_update: StockUpdate, db: Session = Depends(get_db)):
    """อัพเดท stock (ใช้โดย Order Service)"""
    product = db.query(ProductModel).filter(ProductModel.id == product_id).first()
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")
    
    new_stock = product.stock + stock_update.quantity
    if new_stock < 0:
        raise HTTPException(status_code=400, detail=f"Insufficient stock. Current: {product.stock}")
    
    product.stock = new_stock
    db.commit()
    return {"product_id": product_id, "new_stock": new_stock}


@app.post("/products/check-availability")
async def check_availability(
    items: List[dict],  # [{"product_id": 1, "quantity": 2}, ...]
    db: Session = Depends(get_db)
):
    """ตรวจสอบ availability ของหลาย products พร้อมกัน"""
    results = []
    for item in items:
        product = db.query(ProductModel).filter(ProductModel.id == item["product_id"]).first()
        if not product:
            results.append({"product_id": item["product_id"], "available": False, "reason": "not found"})
        elif product.stock < item["quantity"]:
            results.append({
                "product_id": item["product_id"], 
                "available": False,
                "reason": f"insufficient stock ({product.stock} available)"
            })
        else:
            results.append({
                "product_id": item["product_id"],
                "available": True,
                "price": product.price,
                "total": product.price * item["quantity"]
            })
    return results
```

### Order Service (Orchestrator)

```python
# services/order_service/main.py
from fastapi import FastAPI, HTTPException, Depends, BackgroundTasks
from sqlalchemy import create_engine, Column, Integer, String, Float, DateTime, JSON
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from pydantic import BaseModel
from datetime import datetime
from typing import List, Optional
import httpx
import os
import asyncio

app = FastAPI(title="Order Service", version="1.0.0")

DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://user:password@order-db:5432/orders")
USER_SERVICE_URL = os.getenv("USER_SERVICE_URL", "http://user-service:8001")
PRODUCT_SERVICE_URL = os.getenv("PRODUCT_SERVICE_URL", "http://product-service:8002")
PAYMENT_SERVICE_URL = os.getenv("PAYMENT_SERVICE_URL", "http://payment-service:8004")
NOTIFICATION_SERVICE_URL = os.getenv("NOTIFICATION_SERVICE_URL", "http://notification-service:8005")

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()


class OrderModel(Base):
    __tablename__ = "orders"
    
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, nullable=False, index=True)
    items = Column(JSON, nullable=False)  # [{"product_id": 1, "quantity": 2, "price": 100}]
    total_amount = Column(Float, nullable=False)
    status = Column(String, default="pending")  # pending, paid, failed, shipped, delivered
    payment_id = Column(String)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)


Base.metadata.create_all(bind=engine)


class OrderItem(BaseModel):
    product_id: int
    quantity: int


class OrderCreate(BaseModel):
    user_id: int
    items: List[OrderItem]
    payment_method: str = "credit_card"


class OrderResponse(BaseModel):
    id: int
    user_id: int
    items: list
    total_amount: float
    status: str
    payment_id: Optional[str]
    created_at: datetime
    
    class Config:
        from_attributes = True


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


async def call_service(client: httpx.AsyncClient, url: str, method: str = "GET", **kwargs):
    """Helper function สำหรับเรียก service อื่น"""
    try:
        response = await getattr(client, method.lower())(url, **kwargs)
        response.raise_for_status()
        return response.json()
    except httpx.HTTPStatusError as e:
        raise HTTPException(
            status_code=e.response.status_code,
            detail=f"Service call failed: {e.response.text}"
        )
    except httpx.RequestError as e:
        raise HTTPException(
            status_code=503,
            detail=f"Service unavailable: {str(e)}"
        )


@app.post("/orders/", response_model=OrderResponse, status_code=201)
async def create_order(
    order_data: OrderCreate,
    background_tasks: BackgroundTasks,
    db: Session = Depends(get_db)
):
    """
    สร้าง order โดยประสานงานกับ services อื่น
    
    Flow:
    1. ตรวจสอบ user
    2. ตรวจสอบ product availability
    3. คำนวณราคา
    4. สร้าง order ในสถานะ pending
    5. เรียก payment service
    6. อัพเดท stock
    7. ส่ง notification (async)
    """
    async with httpx.AsyncClient(timeout=10.0) as client:
        # 1. ตรวจสอบว่า user มีอยู่จริง
        user = await call_service(
            client, 
            f"{USER_SERVICE_URL}/users/{order_data.user_id}"
        )
        
        # 2. ตรวจสอบ product availability
        check_items = [
            {"product_id": item.product_id, "quantity": item.quantity}
            for item in order_data.items
        ]
        availability = await call_service(
            client,
            f"{PRODUCT_SERVICE_URL}/products/check-availability",
            method="POST",
            json=check_items
        )
        
        # ตรวจสอบว่าทุก product available
        unavailable = [item for item in availability if not item["available"]]
        if unavailable:
            raise HTTPException(
                status_code=400,
                detail=f"Items not available: {unavailable}"
            )
        
        # 3. คำนวณราคารวม
        price_map = {item["product_id"]: item["price"] for item in availability}
        order_items = []
        total_amount = 0
        
        for item in order_data.items:
            price = price_map[item.product_id]
            item_total = price * item.quantity
            total_amount += item_total
            order_items.append({
                "product_id": item.product_id,
                "quantity": item.quantity,
                "unit_price": price,
                "total": item_total
            })
        
        # 4. สร้าง order ในสถานะ pending
        order = OrderModel(
            user_id=order_data.user_id,
            items=order_items,
            total_amount=total_amount,
            status="pending"
        )
        db.add(order)
        db.commit()
        db.refresh(order)
        
        # 5. เรียก payment service
        try:
            payment_result = await call_service(
                client,
                f"{PAYMENT_SERVICE_URL}/payments/",
                method="POST",
                json={
                    "order_id": order.id,
                    "amount": total_amount,
                    "payment_method": order_data.payment_method,
                    "user_id": order_data.user_id
                }
            )
            
            order.status = "paid"
            order.payment_id = payment_result["payment_id"]
            
            # 6. อัพเดท stock
            for item in order_data.items:
                await call_service(
                    client,
                    f"{PRODUCT_SERVICE_URL}/products/{item.product_id}/stock",
                    method="PATCH",
                    json={"quantity": -item.quantity}
                )
            
        except HTTPException as e:
            order.status = "payment_failed"
            db.commit()
            raise
        
        db.commit()
        db.refresh(order)
        
        # 7. ส่ง notification แบบ async (ไม่ block response)
        background_tasks.add_task(
            send_order_notification,
            user_email=user["email"],
            order_id=order.id,
            total_amount=total_amount
        )
        
        return order


async def send_order_notification(user_email: str, order_id: int, total_amount: float):
    """ส่ง notification แบบ background task"""
    async with httpx.AsyncClient() as client:
        try:
            await client.post(
                f"{NOTIFICATION_SERVICE_URL}/notifications/email",
                json={
                    "to": user_email,
                    "subject": f"Order #{order_id} Confirmed",
                    "body": f"Your order has been confirmed. Total: {total_amount:.2f} THB"
                }
            )
        except Exception as e:
            # Log error แต่ไม่ fail order
            print(f"Failed to send notification: {e}")


@app.get("/orders/{order_id}", response_model=OrderResponse)
async def get_order(order_id: int, db: Session = Depends(get_db)):
    order = db.query(OrderModel).filter(OrderModel.id == order_id).first()
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")
    return order
```

---

## 3. Service Communication ด้วย gRPC

```python
# proto/user.proto
"""
syntax = "proto3";

package user;

service UserService {
  rpc GetUser (GetUserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
  rpc VerifyPassword (VerifyPasswordRequest) returns (VerifyPasswordResponse);
}

message GetUserRequest {
  int32 user_id = 1;
}

message CreateUserRequest {
  string email = 1;
  string name = 2;
  string password = 3;
}

message UserResponse {
  int32 id = 1;
  string email = 2;
  string name = 3;
  bool is_active = 4;
}

message VerifyPasswordRequest {
  string email = 1;
  string password = 2;
}

message VerifyPasswordResponse {
  bool valid = 1;
  int32 user_id = 2;
}
"""

# services/user_service/grpc_server.py
import grpc
from concurrent import futures
import user_pb2
import user_pb2_grpc
from sqlalchemy.orm import Session
from main import UserModel, SessionLocal, verify_password, hash_password


class UserServicer(user_pb2_grpc.UserServiceServicer):
    """gRPC server implementation สำหรับ User Service"""
    
    def GetUser(self, request, context):
        db = SessionLocal()
        try:
            user = db.query(UserModel).filter(UserModel.id == request.user_id).first()
            if not user:
                context.set_code(grpc.StatusCode.NOT_FOUND)
                context.set_details("User not found")
                return user_pb2.UserResponse()
            
            return user_pb2.UserResponse(
                id=user.id,
                email=user.email,
                name=user.name,
                is_active=user.is_active
            )
        finally:
            db.close()
    
    def CreateUser(self, request, context):
        db = SessionLocal()
        try:
            # ตรวจสอบ email ซ้ำ
            existing = db.query(UserModel).filter(UserModel.email == request.email).first()
            if existing:
                context.set_code(grpc.StatusCode.ALREADY_EXISTS)
                context.set_details("Email already registered")
                return user_pb2.UserResponse()
            
            user = UserModel(
                email=request.email,
                name=request.name,
                hashed_password=hash_password(request.password)
            )
            db.add(user)
            db.commit()
            db.refresh(user)
            
            return user_pb2.UserResponse(
                id=user.id,
                email=user.email,
                name=user.name,
                is_active=user.is_active
            )
        finally:
            db.close()
    
    def VerifyPassword(self, request, context):
        db = SessionLocal()
        try:
            user = db.query(UserModel).filter(UserModel.email == request.email).first()
            if user and verify_password(request.password, user.hashed_password):
                return user_pb2.VerifyPasswordResponse(valid=True, user_id=user.id)
            return user_pb2.VerifyPasswordResponse(valid=False, user_id=0)
        finally:
            db.close()


def serve():
    """เริ่ม gRPC server"""
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    user_pb2_grpc.add_UserServiceServicer_to_server(UserServicer(), server)
    server.add_insecure_port("[::]:50051")
    server.start()
    print("gRPC User Service started on port 50051")
    server.wait_for_termination()


if __name__ == "__main__":
    serve()


# services/order_service/grpc_client.py
import grpc
import user_pb2
import user_pb2_grpc


class UserServiceClient:
    """gRPC client สำหรับเรียก User Service"""
    
    def __init__(self, host: str = "user-service", port: int = 50051):
        self.channel = grpc.insecure_channel(f"{host}:{port}")
        self.stub = user_pb2_grpc.UserServiceStub(self.channel)
    
    def get_user(self, user_id: int) -> dict:
        try:
            response = self.stub.GetUser(
                user_pb2.GetUserRequest(user_id=user_id),
                timeout=5.0
            )
            return {
                "id": response.id,
                "email": response.email,
                "name": response.name,
                "is_active": response.is_active
            }
        except grpc.RpcError as e:
            if e.code() == grpc.StatusCode.NOT_FOUND:
                return None
            raise
    
    def verify_password(self, email: str, password: str) -> dict:
        try:
            response = self.stub.VerifyPassword(
                user_pb2.VerifyPasswordRequest(email=email, password=password),
                timeout=5.0
            )
            return {"valid": response.valid, "user_id": response.user_id}
        except grpc.RpcError as e:
            raise Exception(f"gRPC error: {e.details()}")
    
    def close(self):
        self.channel.close()
```

---

## 4. Service Discovery

```python
# service_discovery/consul_client.py
import consul
import socket
import os
from typing import Optional, List


class ServiceRegistry:
    """Service Registry ใช้ Consul สำหรับ service discovery"""
    
    def __init__(self, consul_host: str = "consul", consul_port: int = 8500):
        self.client = consul.Consul(host=consul_host, port=consul_port)
        self.service_id = None
        self.service_name = None
    
    def register(
        self,
        service_name: str,
        service_port: int,
        health_check_url: str,
        tags: List[str] = None
    ):
        """ลงทะเบียน service กับ Consul"""
        hostname = socket.gethostname()
        service_ip = socket.gethostbyname(hostname)
        self.service_id = f"{service_name}-{hostname}-{service_port}"
        self.service_name = service_name
        
        self.client.agent.service.register(
            name=service_name,
            service_id=self.service_id,
            address=service_ip,
            port=service_port,
            tags=tags or [],
            check={
                "http": f"http://{service_ip}:{service_port}{health_check_url}",
                "interval": "10s",
                "timeout": "5s",
                "deregister_critical_service_after": "30s"
            }
        )
        print(f"Registered {service_name} at {service_ip}:{service_port}")
    
    def deregister(self):
        """ยกเลิกการลงทะเบียน service"""
        if self.service_id:
            self.client.agent.service.deregister(self.service_id)
            print(f"Deregistered {self.service_name}")
    
    def discover(self, service_name: str) -> Optional[str]:
        """ค้นหา service URL จาก Consul"""
        _, services = self.client.health.service(service_name, passing=True)
        
        if not services:
            return None
        
        # เลือก service แบบ round-robin หรือ random
        import random
        service = random.choice(services)
        address = service["Service"]["Address"]
        port = service["Service"]["Port"]
        return f"http://{address}:{port}"
    
    def get_all_instances(self, service_name: str) -> List[str]:
        """ดึง URL ของ service ทุก instance"""
        _, services = self.client.health.service(service_name, passing=True)
        return [
            f"http://{s['Service']['Address']}:{s['Service']['Port']}"
            for s in services
        ]


# ใช้งาน service discovery ใน Order Service
class ServiceClient:
    """HTTP client ที่ใช้ service discovery"""
    
    def __init__(self, registry: ServiceRegistry):
        self.registry = registry
    
    async def call(self, service_name: str, path: str, method: str = "GET", **kwargs):
        """เรียก service โดยใช้ service discovery"""
        import httpx
        
        service_url = self.registry.discover(service_name)
        if not service_url:
            raise Exception(f"Service '{service_name}' not available")
        
        async with httpx.AsyncClient() as client:
            url = f"{service_url}{path}"
            response = await getattr(client, method.lower())(url, **kwargs)
            response.raise_for_status()
            return response.json()


# main.py - เริ่ม service พร้อม registration
import asyncio
from contextlib import asynccontextmanager
import uvicorn


registry = ServiceRegistry()


@asynccontextmanager
async def lifespan(app):
    # Startup: ลงทะเบียน service
    registry.register(
        service_name="order-service",
        service_port=8003,
        health_check_url="/health",
        tags=["api", "order"]
    )
    yield
    # Shutdown: ยกเลิกการลงทะเบียน
    registry.deregister()
```

---

## 5. API Gateway Pattern

```python
# api_gateway/main.py
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.responses import JSONResponse
from fastapi.middleware.cors import CORSMiddleware
import httpx
import jwt
import time
import os
from typing import Optional
from functools import lru_cache
import asyncio
from collections import defaultdict

app = FastAPI(title="API Gateway", version="1.0.0")

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# Service URLs (ในการใช้งานจริงใช้ service discovery)
SERVICES = {
    "users": os.getenv("USER_SERVICE_URL", "http://user-service:8001"),
    "products": os.getenv("PRODUCT_SERVICE_URL", "http://product-service:8002"),
    "orders": os.getenv("ORDER_SERVICE_URL", "http://order-service:8003"),
    "payments": os.getenv("PAYMENT_SERVICE_URL", "http://payment-service:8004"),
}

JWT_SECRET = os.getenv("JWT_SECRET", "your-secret-key")

# Rate limiting storage
rate_limit_store = defaultdict(list)


class RateLimiter:
    """Rate limiter แบบ sliding window"""
    
    def __init__(self, max_requests: int = 100, window_seconds: int = 60):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests = defaultdict(list)
    
    def is_allowed(self, client_id: str) -> bool:
        now = time.time()
        window_start = now - self.window_seconds
        
        # ลบ requests ที่หมดอายุแล้ว
        self.requests[client_id] = [
            req_time for req_time in self.requests[client_id]
            if req_time > window_start
        ]
        
        if len(self.requests[client_id]) >= self.max_requests:
            return False
        
        self.requests[client_id].append(now)
        return True


rate_limiter = RateLimiter(max_requests=100, window_seconds=60)


def verify_token(token: str) -> Optional[dict]:
    """ตรวจสอบ JWT token"""
    try:
        payload = jwt.decode(token, JWT_SECRET, algorithms=["HS256"])
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")


async def proxy_request(
    service: str,
    path: str,
    request: Request,
    add_user_header: bool = False,
    user_id: Optional[int] = None
):
    """Proxy request ไปยัง service"""
    if service not in SERVICES:
        raise HTTPException(status_code=404, detail=f"Service '{service}' not found")
    
    service_url = SERVICES[service]
    target_url = f"{service_url}{path}"
    
    # Copy headers จาก original request
    headers = dict(request.headers)
    headers.pop("host", None)  # ลบ host header
    
    # เพิ่ม user info ถ้ามี
    if add_user_header and user_id:
        headers["X-User-ID"] = str(user_id)
    
    # เพิ่ม request ID สำหรับ tracing
    import uuid
    headers["X-Request-ID"] = str(uuid.uuid4())
    
    # อ่าน request body
    body = await request.body()
    
    async with httpx.AsyncClient(timeout=30.0) as client:
        response = await client.request(
            method=request.method,
            url=target_url,
            headers=headers,
            content=body,
            params=dict(request.query_params)
        )
    
    return JSONResponse(
        content=response.json() if response.content else None,
        status_code=response.status_code,
        headers=dict(response.headers)
    )


@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    """Rate limiting middleware"""
    client_ip = request.client.host
    
    if not rate_limiter.is_allowed(client_ip):
        return JSONResponse(
            status_code=429,
            content={"detail": "Too many requests. Please try again later."}
        )
    
    response = await call_next(request)
    return response


@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    """Request logging middleware"""
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time
    
    print(f"{request.method} {request.url.path} - {response.status_code} - {process_time:.3f}s")
    response.headers["X-Process-Time"] = str(process_time)
    return response


# Public routes (ไม่ต้อง authentication)
@app.api_route("/api/v1/products/{path:path}", methods=["GET"])
async def products_public(path: str, request: Request):
    return await proxy_request("products", f"/products/{path}", request)


@app.api_route("/api/v1/products", methods=["GET"])
async def products_list(request: Request):
    return await proxy_request("products", "/products/", request)


# Auth routes
@app.post("/api/v1/auth/login")
async def login(credentials: dict):
    """Login และรับ JWT token"""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{SERVICES['users']}/users/verify-password",
            params={"email": credentials["email"], "password": credentials["password"]}
        )
        
        if response.status_code != 200:
            raise HTTPException(status_code=401, detail="Invalid credentials")
        
        user_data = response.json()
        
        # สร้าง JWT token
        payload = {
            "user_id": user_data["user_id"],
            "email": user_data["email"],
            "exp": time.time() + 3600  # หมดอายุใน 1 ชั่วโมง
        }
        token = jwt.encode(payload, JWT_SECRET, algorithm="HS256")
        
        return {"access_token": token, "token_type": "bearer"}


# Protected routes (ต้อง authentication)
def get_current_user(request: Request) -> dict:
    """ดึง user จาก JWT token"""
    authorization = request.headers.get("Authorization")
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing or invalid token")
    
    token = authorization.split(" ")[1]
    return verify_token(token)


@app.api_route("/api/v1/orders/{path:path}", methods=["GET", "POST", "PUT", "PATCH"])
async def orders_protected(path: str, request: Request):
    current_user = get_current_user(request)
    return await proxy_request(
        "orders", 
        f"/orders/{path}", 
        request,
        add_user_header=True,
        user_id=current_user["user_id"]
    )


@app.get("/api/v1/health")
async def gateway_health():
    """ตรวจสอบ health ของทุก services"""
    health_status = {}
    
    async with httpx.AsyncClient(timeout=5.0) as client:
        for service_name, service_url in SERVICES.items():
            try:
                response = await client.get(f"{service_url}/health")
                health_status[service_name] = {
                    "status": "healthy" if response.status_code == 200 else "unhealthy",
                    "response_time": response.elapsed.total_seconds()
                }
            except Exception as e:
                health_status[service_name] = {"status": "unavailable", "error": str(e)}
    
    overall = "healthy" if all(s["status"] == "healthy" for s in health_status.values()) else "degraded"
    return {"overall": overall, "services": health_status}
```

---

## 6. Docker Compose Multi-Service Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  # API Gateway
  api-gateway:
    build:
      context: ./api_gateway
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - USER_SERVICE_URL=http://user-service:8001
      - PRODUCT_SERVICE_URL=http://product-service:8002
      - ORDER_SERVICE_URL=http://order-service:8003
      - PAYMENT_SERVICE_URL=http://payment-service:8004
      - JWT_SECRET=your-super-secret-jwt-key
    depends_on:
      - user-service
      - product-service
      - order-service
    networks:
      - microservices-net
    restart: unless-stopped

  # User Service
  user-service:
    build:
      context: ./services/user_service
      dockerfile: Dockerfile
    ports:
      - "8001:8001"
    environment:
      - DATABASE_URL=postgresql://postgres:password@user-db:5432/users
      - PORT=8001
    depends_on:
      user-db:
        condition: service_healthy
    networks:
      - microservices-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  user-db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=users
    volumes:
      - user-db-data:/var/lib/postgresql/data
    networks:
      - microservices-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Product Service
  product-service:
    build:
      context: ./services/product_service
      dockerfile: Dockerfile
    ports:
      - "8002:8002"
    environment:
      - DATABASE_URL=postgresql://postgres:password@product-db:5432/products
      - PORT=8002
    depends_on:
      product-db:
        condition: service_healthy
    networks:
      - microservices-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8002/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  product-db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=products
    volumes:
      - product-db-data:/var/lib/postgresql/data
    networks:
      - microservices-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Order Service
  order-service:
    build:
      context: ./services/order_service
      dockerfile: Dockerfile
    ports:
      - "8003:8003"
    environment:
      - DATABASE_URL=postgresql://postgres:password@order-db:5432/orders
      - USER_SERVICE_URL=http://user-service:8001
      - PRODUCT_SERVICE_URL=http://product-service:8002
      - PAYMENT_SERVICE_URL=http://payment-service:8004
      - NOTIFICATION_SERVICE_URL=http://notification-service:8005
      - PORT=8003
    depends_on:
      order-db:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks:
      - microservices-net
    restart: unless-stopped

  order-db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=orders
    volumes:
      - order-db-data:/var/lib/postgresql/data
    networks:
      - microservices-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Payment Service
  payment-service:
    build:
      context: ./services/payment_service
      dockerfile: Dockerfile
    ports:
      - "8004:8004"
    environment:
      - DATABASE_URL=postgresql://postgres:password@payment-db:5432/payments
      - STRIPE_SECRET_KEY=${STRIPE_SECRET_KEY}
      - PORT=8004
    depends_on:
      payment-db:
        condition: service_healthy
    networks:
      - microservices-net
    restart: unless-stopped

  payment-db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=payments
    volumes:
      - payment-db-data:/var/lib/postgresql/data
    networks:
      - microservices-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Notification Service
  notification-service:
    build:
      context: ./services/notification_service
      dockerfile: Dockerfile
    ports:
      - "8005:8005"
    environment:
      - SMTP_HOST=${SMTP_HOST}
      - SMTP_PORT=587
      - SMTP_USER=${SMTP_USER}
      - SMTP_PASSWORD=${SMTP_PASSWORD}
      - RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
      - PORT=8005
    depends_on:
      rabbitmq:
        condition: service_healthy
    networks:
      - microservices-net
    restart: unless-stopped

  # Message Queue (RabbitMQ)
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"  # Management UI
    environment:
      - RABBITMQ_DEFAULT_USER=guest
      - RABBITMQ_DEFAULT_PASS=guest
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    networks:
      - microservices-net
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 10s
      timeout: 10s
      retries: 5

  # Redis (Cache + Session)
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - microservices-net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Consul (Service Discovery)
  consul:
    image: consul:1.15
    ports:
      - "8500:8500"
    command: "agent -server -bootstrap-expect=1 -ui -client=0.0.0.0"
    networks:
      - microservices-net

volumes:
  user-db-data:
  product-db-data:
  order-db-data:
  payment-db-data:
  rabbitmq-data:
  redis-data:

networks:
  microservices-net:
    driver: bridge
```

```dockerfile
# services/user_service/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# ติดตั้ง dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy source code
COPY . .

# สร้าง non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8001

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8001", "--workers", "4"]
```

---

## 7. Inter-Service Communication Patterns

```python
# patterns/circuit_breaker.py
import asyncio
import time
from enum import Enum
from typing import Callable, Any
import httpx


class CircuitState(Enum):
    CLOSED = "closed"      # ปกติ - ส่ง request ได้
    OPEN = "open"          # เปิด - ไม่ส่ง request
    HALF_OPEN = "half_open"  # ลองส่ง request ดู


class CircuitBreaker:
    """
    Circuit Breaker Pattern
    ป้องกันการ cascade failure ใน microservices
    """
    
    def __init__(
        self,
        failure_threshold: int = 5,
        success_threshold: int = 2,
        timeout: float = 60.0
    ):
        self.failure_threshold = failure_threshold  # จำนวน failure ก่อน OPEN
        self.success_threshold = success_threshold  # จำนวน success ก่อน CLOSED
        self.timeout = timeout  # เวลา (วินาที) ก่อนลอง HALF_OPEN
        
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
    
    def _should_attempt(self) -> bool:
        """ตรวจสอบว่าควร attempt หรือไม่"""
        if self.state == CircuitState.CLOSED:
            return True
        
        if self.state == CircuitState.OPEN:
            # ตรวจสอบว่าหมด timeout แล้วหรือยัง
            if time.time() - self.last_failure_time > self.timeout:
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
                print(f"Circuit Breaker: OPEN -> HALF_OPEN")
                return True
            return False
        
        if self.state == CircuitState.HALF_OPEN:
            return True
        
        return False
    
    def _on_success(self):
        """จัดการเมื่อ success"""
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state = CircuitState.CLOSED
                self.failure_count = 0
                print(f"Circuit Breaker: HALF_OPEN -> CLOSED")
        elif self.state == CircuitState.CLOSED:
            self.failure_count = 0
    
    def _on_failure(self):
        """จัดการเมื่อ failure"""
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.OPEN
            print(f"Circuit Breaker: HALF_OPEN -> OPEN")
        elif self.state == CircuitState.CLOSED and self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
            print(f"Circuit Breaker: CLOSED -> OPEN (failures: {self.failure_count})")
    
    async def call(self, func: Callable, *args, **kwargs) -> Any:
        """เรียก function ผ่าน circuit breaker"""
        if not self._should_attempt():
            raise Exception(f"Circuit breaker is OPEN. Service unavailable.")
        
        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise


# การใช้งาน Circuit Breaker
user_service_cb = CircuitBreaker(failure_threshold=5, timeout=30.0)

async def get_user_with_circuit_breaker(user_id: int) -> dict:
    async def _get_user():
        async with httpx.AsyncClient(timeout=5.0) as client:
            response = await client.get(f"http://user-service:8001/users/{user_id}")
            response.raise_for_status()
            return response.json()
    
    try:
        return await user_service_cb.call(_get_user)
    except Exception as e:
        # Fallback: ส่งค่า default หรือ raise error
        print(f"User service unavailable: {e}")
        raise HTTPException(status_code=503, detail="User service temporarily unavailable")


# patterns/retry.py
import asyncio
import functools
from typing import Type, Tuple


def retry(
    max_attempts: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    exceptions: Tuple[Type[Exception], ...] = (Exception,)
):
    """Decorator สำหรับ retry logic"""
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            current_delay = delay
            last_exception = None
            
            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e
                    if attempt < max_attempts - 1:
                        print(f"Attempt {attempt + 1} failed: {e}. Retrying in {current_delay}s...")
                        await asyncio.sleep(current_delay)
                        current_delay *= backoff
                    
            raise last_exception
        return wrapper
    return decorator


# การใช้งาน retry
@retry(max_attempts=3, delay=0.5, backoff=2.0, exceptions=(httpx.RequestError,))
async def resilient_service_call(url: str) -> dict:
    async with httpx.AsyncClient(timeout=5.0) as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()
```

---

## 8. สรุป Part 096

✅ เข้าใจความแตกต่างระหว่าง Microservices และ Monolith Architecture อย่างลึกซึ้ง
✅ สร้าง Microservices แต่ละตัว (User, Product, Order Service) ด้วย FastAPI
✅ จัดการ Service Communication ผ่าน REST API และ gRPC
✅ ใช้งาน Service Discovery ด้วย Consul
✅ สร้าง API Gateway ที่มี Rate Limiting, Authentication และ Request Proxying
✅ ตั้งค่า Multi-service environment ด้วย Docker Compose
✅ ใช้ Circuit Breaker และ Retry patterns สำหรับ fault tolerance

## ➡️ ถัดไป: Part 097 - Message Queues and Event-Driven Architecture

*Part 096/105 | Python Course - World-Class Level*
