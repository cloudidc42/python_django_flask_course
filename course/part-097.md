# Part 097: Message Queues and Event-Driven Architecture

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Event-Driven Architecture และ Message Queue concepts
- ใช้งาน RabbitMQ กับ pika library
- ใช้ Redis Pub/Sub สำหรับ real-time messaging
- ใช้งาน Apache Kafka กับ kafka-python
- ออกแบบ Event Sourcing pattern
- ใช้ Saga pattern สำหรับ distributed transactions

---

## 1. Event-Driven Architecture Overview

```python
# event_driven_concepts.py
"""
Event-Driven Architecture (EDA) คือ architectural pattern ที่ระบบสื่อสาร
ผ่าน events แทนการเรียก API โดยตรง

ข้อดี:
- Loose coupling: services ไม่รู้จักกันโดยตรง
- Scalability: producer และ consumer ขยายได้อิสระ
- Resilience: ถ้า consumer ล่ม message ยังอยู่ใน queue
- Audit trail: เก็บประวัติทุก event ได้

Components หลัก:
┌──────────┐    Event    ┌───────────────┐    Event    ┌──────────┐
│ Producer │ ──────────► │ Message Broker│ ──────────► │ Consumer │
│(Publisher)│            │ (Queue/Topic) │             │(Subscriber│
└──────────┘            └───────────────┘             └──────────┘

Message Brokers ที่นิยม:
1. RabbitMQ - AMQP protocol, เหมาะกับ task queue
2. Redis Pub/Sub - ง่าย, เร็ว, แต่ไม่ durable
3. Apache Kafka - High throughput, เหมาะกับ event streaming
4. AWS SQS/SNS - Managed service บน cloud
"""

# event_types.py - นิยาม event types
from dataclasses import dataclass, asdict
from datetime import datetime
from typing import Any, Dict, Optional
import json
import uuid


@dataclass
class Event:
    """Base event class"""
    event_id: str
    event_type: str
    timestamp: str
    data: Dict[str, Any]
    metadata: Optional[Dict[str, Any]] = None
    
    def to_json(self) -> str:
        return json.dumps(asdict(self))
    
    @classmethod
    def create(cls, event_type: str, data: dict, metadata: dict = None) -> "Event":
        return cls(
            event_id=str(uuid.uuid4()),
            event_type=event_type,
            timestamp=datetime.utcnow().isoformat(),
            data=data,
            metadata=metadata or {}
        )


# Event types สำหรับ E-commerce
class OrderEvents:
    ORDER_CREATED = "order.created"
    ORDER_PAID = "order.paid"
    ORDER_SHIPPED = "order.shipped"
    ORDER_DELIVERED = "order.delivered"
    ORDER_CANCELLED = "order.cancelled"

class UserEvents:
    USER_REGISTERED = "user.registered"
    USER_LOGGED_IN = "user.logged_in"
    PASSWORD_CHANGED = "user.password_changed"

class PaymentEvents:
    PAYMENT_PROCESSED = "payment.processed"
    PAYMENT_FAILED = "payment.failed"
    REFUND_INITIATED = "payment.refund_initiated"
```

---

## 2. RabbitMQ กับ pika

### การติดตั้งและตั้งค่า

```bash
# ติดตั้ง RabbitMQ ด้วย Docker
docker run -d \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=password \
  rabbitmq:3-management

# ติดตั้ง pika
pip install pika
```

### RabbitMQ Connection Manager

```python
# rabbitmq/connection.py
import pika
import time
import logging
from typing import Optional, Callable
import os

logger = logging.getLogger(__name__)


class RabbitMQConnection:
    """Connection manager สำหรับ RabbitMQ"""
    
    def __init__(
        self,
        host: str = None,
        port: int = 5672,
        username: str = None,
        password: str = None,
        virtual_host: str = "/"
    ):
        self.host = host or os.getenv("RABBITMQ_HOST", "localhost")
        self.port = port
        self.username = username or os.getenv("RABBITMQ_USER", "guest")
        self.password = password or os.getenv("RABBITMQ_PASS", "guest")
        self.virtual_host = virtual_host
        self._connection: Optional[pika.BlockingConnection] = None
        self._channel: Optional[pika.channel.Channel] = None
    
    def connect(self, max_retries: int = 5, retry_delay: float = 2.0):
        """เชื่อมต่อ RabbitMQ พร้อม retry logic"""
        credentials = pika.PlainCredentials(self.username, self.password)
        parameters = pika.ConnectionParameters(
            host=self.host,
            port=self.port,
            virtual_host=self.virtual_host,
            credentials=credentials,
            heartbeat=600,
            blocked_connection_timeout=300
        )
        
        for attempt in range(max_retries):
            try:
                self._connection = pika.BlockingConnection(parameters)
                self._channel = self._connection.channel()
                logger.info(f"Connected to RabbitMQ at {self.host}:{self.port}")
                return
            except pika.exceptions.AMQPConnectionError as e:
                if attempt < max_retries - 1:
                    logger.warning(f"Connection failed (attempt {attempt+1}). Retrying in {retry_delay}s...")
                    time.sleep(retry_delay)
                    retry_delay *= 2  # exponential backoff
                else:
                    raise ConnectionError(f"Failed to connect to RabbitMQ after {max_retries} attempts") from e
    
    def disconnect(self):
        """ปิดการเชื่อมต่อ"""
        if self._connection and not self._connection.is_closed:
            self._connection.close()
            logger.info("Disconnected from RabbitMQ")
    
    @property
    def channel(self) -> pika.channel.Channel:
        if not self._channel or self._channel.is_closed:
            self.connect()
        return self._channel
    
    def __enter__(self):
        self.connect()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.disconnect()
```

### Message Publisher

```python
# rabbitmq/publisher.py
import pika
import json
import logging
from typing import Any, Dict, Optional
from .connection import RabbitMQConnection
from event_types import Event

logger = logging.getLogger(__name__)


class MessagePublisher:
    """Publisher สำหรับส่ง messages ไปยัง RabbitMQ"""
    
    def __init__(self, connection: RabbitMQConnection):
        self.conn = connection
    
    def setup_exchange(
        self,
        exchange_name: str,
        exchange_type: str = "topic",  # direct, fanout, topic, headers
        durable: bool = True
    ):
        """สร้าง exchange"""
        self.conn.channel.exchange_declare(
            exchange=exchange_name,
            exchange_type=exchange_type,
            durable=durable
        )
    
    def publish(
        self,
        exchange: str,
        routing_key: str,
        message: Any,
        persistent: bool = True,
        headers: Dict = None
    ):
        """ส่ง message ไปยัง exchange"""
        # แปลง message เป็น JSON
        if isinstance(message, Event):
            body = message.to_json().encode()
        elif isinstance(message, dict):
            body = json.dumps(message).encode()
        elif isinstance(message, str):
            body = message.encode()
        else:
            body = json.dumps(message).encode()
        
        # ตั้งค่า properties
        properties = pika.BasicProperties(
            delivery_mode=2 if persistent else 1,  # 2 = persistent
            content_type="application/json",
            headers=headers or {}
        )
        
        try:
            self.conn.channel.basic_publish(
                exchange=exchange,
                routing_key=routing_key,
                body=body,
                properties=properties
            )
            logger.info(f"Published message to {exchange}/{routing_key}")
        except pika.exceptions.UnroutableError as e:
            logger.error(f"Message unroutable: {e}")
            raise
    
    def publish_event(self, event: Event, exchange: str = "events"):
        """ส่ง event ไปยัง events exchange"""
        self.publish(
            exchange=exchange,
            routing_key=event.event_type,
            message=event
        )


# การใช้งาน Publisher
def order_created_publisher_example():
    """ตัวอย่างการ publish order created event"""
    conn = RabbitMQConnection()
    conn.connect()
    
    publisher = MessagePublisher(conn)
    
    # ตั้งค่า exchange
    publisher.setup_exchange("events", exchange_type="topic")
    
    # สร้างและส่ง event
    event = Event.create(
        event_type="order.created",
        data={
            "order_id": 12345,
            "user_id": 678,
            "items": [
                {"product_id": 1, "quantity": 2, "price": 299.99}
            ],
            "total": 599.98
        }
    )
    
    publisher.publish_event(event)
    conn.disconnect()
```

### Message Consumer

```python
# rabbitmq/consumer.py
import pika
import json
import logging
from typing import Callable, Dict, Any, Optional
from .connection import RabbitMQConnection

logger = logging.getLogger(__name__)


class MessageConsumer:
    """Consumer สำหรับรับ messages จาก RabbitMQ"""
    
    def __init__(self, connection: RabbitMQConnection):
        self.conn = connection
        self.handlers: Dict[str, Callable] = {}
    
    def setup_queue(
        self,
        queue_name: str,
        exchange: str,
        routing_keys: list,
        dead_letter_exchange: str = None,
        max_retries: int = 3
    ):
        """ตั้งค่า queue พร้อม dead letter exchange"""
        channel = self.conn.channel
        
        # ตั้งค่า arguments สำหรับ dead letter
        arguments = {}
        if dead_letter_exchange:
            arguments["x-dead-letter-exchange"] = dead_letter_exchange
        if max_retries:
            arguments["x-message-ttl"] = 30000  # 30 วินาที ก่อนส่งไป DLX
        
        # สร้าง queue
        channel.queue_declare(
            queue=queue_name,
            durable=True,
            arguments=arguments if arguments else None
        )
        
        # Bind queue กับ exchange และ routing keys
        for routing_key in routing_keys:
            channel.queue_bind(
                queue=queue_name,
                exchange=exchange,
                routing_key=routing_key
            )
        
        logger.info(f"Queue '{queue_name}' set up with {len(routing_keys)} routing keys")
    
    def register_handler(self, event_type: str, handler: Callable):
        """ลงทะเบียน handler สำหรับ event type"""
        self.handlers[event_type] = handler
        logger.info(f"Registered handler for '{event_type}'")
    
    def _process_message(self, channel, method, properties, body):
        """ประมวลผล message ที่ได้รับ"""
        try:
            # parse message
            message = json.loads(body.decode())
            event_type = message.get("event_type")
            
            logger.info(f"Received event: {event_type}")
            
            # หา handler ที่เหมาะสม
            handler = self.handlers.get(event_type)
            if handler:
                handler(message)
                # Acknowledge message เมื่อ process สำเร็จ
                channel.basic_ack(delivery_tag=method.delivery_tag)
                logger.info(f"Successfully processed event: {event_type}")
            else:
                logger.warning(f"No handler for event type: {event_type}")
                # ส่งไป DLX ถ้าไม่มี handler
                channel.basic_nack(
                    delivery_tag=method.delivery_tag,
                    requeue=False
                )
        
        except json.JSONDecodeError as e:
            logger.error(f"Failed to parse message: {e}")
            channel.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
        
        except Exception as e:
            logger.error(f"Error processing message: {e}")
            # Requeue message สำหรับ retry (แต่ระวัง infinite loop)
            channel.basic_nack(
                delivery_tag=method.delivery_tag,
                requeue=method.redelivered is False  # requeue ครั้งแรกเท่านั้น
            )
    
    def start_consuming(self, queue_name: str, prefetch_count: int = 10):
        """เริ่มรับ messages"""
        channel = self.conn.channel
        
        # QoS: รับ message ที่ละ prefetch_count ชิ้น
        channel.basic_qos(prefetch_count=prefetch_count)
        
        channel.basic_consume(
            queue=queue_name,
            on_message_callback=self._process_message
        )
        
        logger.info(f"Starting to consume from queue: {queue_name}")
        channel.start_consuming()


# ตัวอย่าง handlers สำหรับ Notification Service
class NotificationHandlers:
    """Handlers สำหรับ notification events"""
    
    def handle_order_created(self, event: dict):
        """ส่ง email เมื่อ order ถูกสร้าง"""
        data = event["data"]
        print(f"📧 Sending order confirmation email for order #{data['order_id']}")
        # ส่ง email จริง...
    
    def handle_payment_processed(self, event: dict):
        """ส่ง notification เมื่อชำระเงินสำเร็จ"""
        data = event["data"]
        print(f"💳 Payment {data.get('payment_id')} processed successfully")
    
    def handle_order_shipped(self, event: dict):
        """ส่ง tracking notification"""
        data = event["data"]
        print(f"🚚 Order #{data['order_id']} shipped. Tracking: {data.get('tracking_number')}")


# main_consumer.py
def run_notification_consumer():
    """เริ่ม notification consumer"""
    conn = RabbitMQConnection()
    conn.connect()
    
    consumer = MessageConsumer(conn)
    handlers = NotificationHandlers()
    
    # ตั้งค่า queue
    consumer.setup_queue(
        queue_name="notification-queue",
        exchange="events",
        routing_keys=["order.*", "payment.*"],
        dead_letter_exchange="events.dlx"
    )
    
    # ลงทะเบียน handlers
    consumer.register_handler("order.created", handlers.handle_order_created)
    consumer.register_handler("payment.processed", handlers.handle_payment_processed)
    consumer.register_handler("order.shipped", handlers.handle_order_shipped)
    
    # เริ่มรับ messages
    consumer.start_consuming("notification-queue")
```

### Work Queue Pattern

```python
# rabbitmq/work_queue.py
"""
Work Queue Pattern:
- งาน (task) ถูกส่งไปยัง queue
- Workers หลายตัวรับงานจาก queue เดียวกัน
- แต่ละงานถูก process โดย Worker เดียวเท่านั้น

เหมาะสำหรับ:
- Image processing
- Email sending
- Report generation
- Data import/export
"""
import pika
import json
import time
import threading
from typing import Callable, Dict


class TaskQueue:
    """Work queue สำหรับกระจายงาน"""
    
    QUEUE_NAME = "task_queue"
    
    def __init__(self, rabbitmq_url: str = "amqp://guest:guest@localhost/"):
        self.url = rabbitmq_url
    
    def _get_channel(self):
        """สร้าง connection และ channel ใหม่"""
        params = pika.URLParameters(self.url)
        connection = pika.BlockingConnection(params)
        channel = connection.channel()
        
        # ประกาศ queue
        channel.queue_declare(queue=self.QUEUE_NAME, durable=True)
        return connection, channel
    
    def enqueue(self, task_type: str, payload: dict, priority: int = 0):
        """เพิ่มงานเข้า queue"""
        conn, channel = self._get_channel()
        
        message = {
            "task_type": task_type,
            "payload": payload,
            "priority": priority,
            "created_at": time.time()
        }
        
        channel.basic_publish(
            exchange="",
            routing_key=self.QUEUE_NAME,
            body=json.dumps(message),
            properties=pika.BasicProperties(
                delivery_mode=2,  # persistent
                priority=priority
            )
        )
        
        print(f"Enqueued task: {task_type}")
        conn.close()
    
    def start_worker(self, handlers: Dict[str, Callable], worker_id: str = "worker-1"):
        """เริ่ม worker สำหรับประมวลผลงาน"""
        conn, channel = self._get_channel()
        
        # รับงานทีละ 1 ชิ้น (fair dispatch)
        channel.basic_qos(prefetch_count=1)
        
        def callback(ch, method, properties, body):
            message = json.loads(body.decode())
            task_type = message["task_type"]
            payload = message["payload"]
            
            print(f"[{worker_id}] Processing task: {task_type}")
            start_time = time.time()
            
            try:
                handler = handlers.get(task_type)
                if handler:
                    handler(payload)
                    elapsed = time.time() - start_time
                    print(f"[{worker_id}] Task {task_type} completed in {elapsed:.2f}s")
                else:
                    print(f"[{worker_id}] Unknown task type: {task_type}")
                
                ch.basic_ack(delivery_tag=method.delivery_tag)
            
            except Exception as e:
                print(f"[{worker_id}] Task failed: {e}")
                ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
        
        channel.basic_consume(queue=self.QUEUE_NAME, on_message_callback=callback)
        print(f"[{worker_id}] Waiting for tasks...")
        channel.start_consuming()


# ตัวอย่างการใช้งาน
def simulate_email_task(payload: dict):
    """จำลองการส่ง email"""
    time.sleep(1)  # จำลอง I/O operation
    print(f"  Sent email to: {payload['to']}")

def simulate_resize_image(payload: dict):
    """จำลองการ resize รูปภาพ"""
    time.sleep(2)  # จำลอง CPU operation
    print(f"  Resized image: {payload['filename']}")

# Producer
queue = TaskQueue()
queue.enqueue("send_email", {"to": "user@example.com", "subject": "Hello"})
queue.enqueue("resize_image", {"filename": "photo.jpg", "width": 800})

# Consumer (Workers)
task_handlers = {
    "send_email": simulate_email_task,
    "resize_image": simulate_resize_image,
}

# เริ่ม multiple workers
# worker = TaskQueue()
# worker.start_worker(task_handlers, worker_id="worker-1")
```

---

## 3. Redis Pub/Sub

```python
# redis_pubsub/manager.py
import redis
import json
import threading
import asyncio
from typing import Callable, Dict, List, Optional
import os


class RedisPubSub:
    """Redis Pub/Sub สำหรับ real-time messaging"""
    
    def __init__(self, redis_url: str = None):
        self.redis_url = redis_url or os.getenv("REDIS_URL", "redis://localhost:6379")
        self.client = redis.from_url(self.redis_url, decode_responses=True)
        self.pubsub = self.client.pubsub()
        self._handlers: Dict[str, List[Callable]] = {}
        self._thread: Optional[threading.Thread] = None
    
    def publish(self, channel: str, message: dict) -> int:
        """ส่ง message ไปยัง channel"""
        json_message = json.dumps(message)
        receivers = self.client.publish(channel, json_message)
        return receivers  # จำนวน subscribers ที่รับ message
    
    def subscribe(self, channel: str, handler: Callable):
        """Subscribe channel และลงทะเบียน handler"""
        if channel not in self._handlers:
            self._handlers[channel] = []
        self._handlers[channel].append(handler)
        
        self.pubsub.subscribe(**{channel: self._dispatch_message})
    
    def psubscribe(self, pattern: str, handler: Callable):
        """Subscribe pattern (wildcard) สำหรับ multiple channels"""
        self._handlers[pattern] = [handler]
        self.pubsub.psubscribe(**{pattern: self._dispatch_pattern_message})
    
    def _dispatch_message(self, message):
        """ส่ง message ไปยัง handlers ที่ลงทะเบียน"""
        if message["type"] != "message":
            return
        
        channel = message["channel"]
        try:
            data = json.loads(message["data"])
        except json.JSONDecodeError:
            data = message["data"]
        
        for handler in self._handlers.get(channel, []):
            try:
                handler(channel, data)
            except Exception as e:
                print(f"Handler error for channel {channel}: {e}")
    
    def _dispatch_pattern_message(self, message):
        """ส่ง message สำหรับ pattern subscription"""
        if message["type"] != "pmessage":
            return
        
        pattern = message["pattern"]
        channel = message["channel"]
        try:
            data = json.loads(message["data"])
        except json.JSONDecodeError:
            data = message["data"]
        
        for handler in self._handlers.get(pattern, []):
            try:
                handler(channel, data)
            except Exception as e:
                print(f"Pattern handler error: {e}")
    
    def start_listening(self, daemon: bool = True):
        """เริ่ม background thread สำหรับฟัง messages"""
        self._thread = threading.Thread(
            target=self.pubsub.run_forever,
            daemon=daemon
        )
        self._thread.start()
        print("Redis Pub/Sub listener started")
    
    def stop_listening(self):
        """หยุดฟัง messages"""
        self.pubsub.close()
        if self._thread:
            self._thread.join()
        print("Redis Pub/Sub listener stopped")


# redis_pubsub/realtime_service.py - Real-time notification service
class RealtimeNotificationService:
    """บริการ real-time notification ผ่าน Redis Pub/Sub"""
    
    def __init__(self):
        self.pubsub = RedisPubSub()
        self.client = redis.from_url(
            os.getenv("REDIS_URL", "redis://localhost:6379"),
            decode_responses=True
        )
    
    def notify_user(self, user_id: int, notification_type: str, data: dict):
        """ส่ง notification ไปยัง user เฉพาะ"""
        channel = f"user:{user_id}:notifications"
        message = {
            "type": notification_type,
            "data": data,
            "timestamp": time.time()
        }
        receivers = self.pubsub.publish(channel, message)
        
        # เก็บ notification ไว้สำหรับ user ที่ offline
        self.client.lpush(
            f"user:{user_id}:pending_notifications",
            json.dumps(message)
        )
        self.client.expire(f"user:{user_id}:pending_notifications", 86400)  # 24 ชั่วโมง
        
        return receivers
    
    def broadcast(self, channel: str, message: dict):
        """ส่ง broadcast ไปยังทุกคนใน channel"""
        return self.pubsub.publish(channel, message)
    
    def get_pending_notifications(self, user_id: int) -> list:
        """ดึง notifications ที่ยังไม่ได้อ่านสำหรับ user"""
        notifications = self.client.lrange(
            f"user:{user_id}:pending_notifications", 0, -1
        )
        # ลบ notifications ที่ดึงมาแล้ว
        self.client.delete(f"user:{user_id}:pending_notifications")
        return [json.loads(n) for n in notifications]


# WebSocket integration กับ FastAPI
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Set
import asyncio

app = FastAPI()

class ConnectionManager:
    """จัดการ WebSocket connections"""
    
    def __init__(self):
        self.active_connections: Dict[int, Set[WebSocket]] = {}  # user_id -> websockets
        self.pubsub = RedisPubSub()
        self._setup_subscriptions()
    
    def _setup_subscriptions(self):
        """ตั้งค่า Redis subscriptions"""
        self.pubsub.psubscribe("user:*:notifications", self._handle_user_notification)
        self.pubsub.start_listening()
    
    async def connect(self, websocket: WebSocket, user_id: int):
        """รับ WebSocket connection"""
        await websocket.accept()
        if user_id not in self.active_connections:
            self.active_connections[user_id] = set()
        self.active_connections[user_id].add(websocket)
    
    def disconnect(self, websocket: WebSocket, user_id: int):
        """ตัด WebSocket connection"""
        if user_id in self.active_connections:
            self.active_connections[user_id].discard(websocket)
    
    async def send_to_user(self, user_id: int, message: dict):
        """ส่ง message ไปยัง user"""
        if user_id in self.active_connections:
            dead_sockets = set()
            for ws in self.active_connections[user_id]:
                try:
                    await ws.send_json(message)
                except Exception:
                    dead_sockets.add(ws)
            
            # ลบ dead connections
            self.active_connections[user_id] -= dead_sockets
    
    def _handle_user_notification(self, channel: str, message: dict):
        """จัดการ notification จาก Redis"""
        # Extract user_id จาก channel name "user:123:notifications"
        parts = channel.split(":")
        if len(parts) >= 2:
            user_id = int(parts[1])
            # ใช้ asyncio.create_task ในที่ที่มี event loop
            loop = asyncio.new_event_loop()
            loop.run_until_complete(self.send_to_user(user_id, message))


manager = ConnectionManager()


@app.websocket("/ws/{user_id}")
async def websocket_endpoint(websocket: WebSocket, user_id: int):
    await manager.connect(websocket, user_id)
    try:
        while True:
            # รอรับ messages จาก client
            data = await websocket.receive_json()
            # ประมวลผล client messages ถ้าจำเป็น
            print(f"Received from user {user_id}: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket, user_id)
        print(f"User {user_id} disconnected")
```

---

## 4. Apache Kafka กับ kafka-python

```python
# kafka/producer.py
from kafka import KafkaProducer
from kafka.errors import KafkaError
import json
import logging
from typing import Optional, Callable
import os

logger = logging.getLogger(__name__)


class KafkaEventProducer:
    """Kafka producer สำหรับส่ง events"""
    
    def __init__(
        self,
        bootstrap_servers: str = None,
        topic_prefix: str = ""
    ):
        self.bootstrap_servers = bootstrap_servers or os.getenv(
            "KAFKA_BOOTSTRAP_SERVERS", 
            "localhost:9092"
        )
        self.topic_prefix = topic_prefix
        
        self.producer = KafkaProducer(
            bootstrap_servers=self.bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode("utf-8"),
            key_serializer=lambda k: k.encode("utf-8") if k else None,
            acks="all",              # รอ acknowledgment จาก all replicas
            retries=3,               # retry 3 ครั้งถ้า fail
            max_in_flight_requests_per_connection=1,  # prevent reordering
            compression_type="gzip"  # compress messages
        )
    
    def send(
        self,
        topic: str,
        value: dict,
        key: Optional[str] = None,
        partition: Optional[int] = None,
        headers: Optional[dict] = None,
        callback: Optional[Callable] = None
    ):
        """ส่ง message ไปยัง Kafka topic"""
        full_topic = f"{self.topic_prefix}{topic}" if self.topic_prefix else topic
        
        kafka_headers = [
            (k, v.encode("utf-8")) 
            for k, v in (headers or {}).items()
        ]
        
        future = self.producer.send(
            topic=full_topic,
            value=value,
            key=key,
            partition=partition,
            headers=kafka_headers
        )
        
        if callback:
            future.add_callback(callback)
            future.add_errback(lambda e: logger.error(f"Kafka send error: {e}"))
        else:
            try:
                record_metadata = future.get(timeout=10)
                logger.info(
                    f"Sent to {record_metadata.topic} "
                    f"partition {record_metadata.partition} "
                    f"offset {record_metadata.offset}"
                )
                return record_metadata
            except KafkaError as e:
                logger.error(f"Failed to send message: {e}")
                raise
    
    def send_event(self, event_type: str, data: dict, key: str = None):
        """ส่ง event ในรูปแบบ standard"""
        import uuid
        from datetime import datetime
        
        event = {
            "event_id": str(uuid.uuid4()),
            "event_type": event_type,
            "timestamp": datetime.utcnow().isoformat(),
            "data": data
        }
        
        # ใช้ event_type เป็น topic
        topic = event_type.replace(".", "-")  # order.created -> order-created
        
        return self.send(topic, event, key=key)
    
    def flush(self, timeout: float = None):
        """รอให้ส่ง messages ทั้งหมดก่อนปิด"""
        self.producer.flush(timeout=timeout)
    
    def close(self):
        """ปิด producer"""
        self.producer.close()
    
    def __enter__(self):
        return self
    
    def __exit__(self, *args):
        self.flush()
        self.close()


# kafka/consumer.py
from kafka import KafkaConsumer
from kafka.errors import KafkaError
import json
import logging
from typing import Callable, Dict, List, Optional
import threading
import os

logger = logging.getLogger(__name__)


class KafkaEventConsumer:
    """Kafka consumer สำหรับรับ events"""
    
    def __init__(
        self,
        topics: List[str],
        group_id: str,
        bootstrap_servers: str = None,
        auto_offset_reset: str = "latest"
    ):
        self.bootstrap_servers = bootstrap_servers or os.getenv(
            "KAFKA_BOOTSTRAP_SERVERS",
            "localhost:9092"
        )
        self.handlers: Dict[str, Callable] = {}
        self._running = False
        
        self.consumer = KafkaConsumer(
            *topics,
            bootstrap_servers=self.bootstrap_servers,
            group_id=group_id,
            auto_offset_reset=auto_offset_reset,  # "latest" หรือ "earliest"
            enable_auto_commit=False,  # manual commit สำหรับ exactly-once
            value_deserializer=lambda v: json.loads(v.decode("utf-8")),
            key_deserializer=lambda k: k.decode("utf-8") if k else None,
            max_poll_interval_ms=300000,  # 5 นาที
            session_timeout_ms=30000
        )
    
    def register_handler(self, event_type: str, handler: Callable):
        """ลงทะเบียน handler สำหรับ event type"""
        self.handlers[event_type] = handler
    
    def start(self):
        """เริ่มรับ messages"""
        self._running = True
        
        logger.info(f"Starting Kafka consumer for topics: {self.consumer.subscription()}")
        
        for message in self.consumer:
            if not self._running:
                break
            
            try:
                event = message.value
                event_type = event.get("event_type")
                
                logger.info(
                    f"Received event: {event_type} "
                    f"from partition {message.partition} "
                    f"offset {message.offset}"
                )
                
                handler = self.handlers.get(event_type)
                if handler:
                    handler(event)
                    # Commit offset หลังจาก process สำเร็จ
                    self.consumer.commit()
                else:
                    logger.warning(f"No handler for event type: {event_type}")
                    self.consumer.commit()  # commit ต่อไปแม้ไม่มี handler
                    
            except json.JSONDecodeError as e:
                logger.error(f"Failed to decode message: {e}")
                self.consumer.commit()
            except Exception as e:
                logger.error(f"Error processing message: {e}")
                # ไม่ commit เพื่อให้ reprocess
    
    def stop(self):
        """หยุด consumer"""
        self._running = False
        self.consumer.close()


# kafka/admin.py - สร้าง Kafka topics
from kafka.admin import KafkaAdminClient, NewTopic


def create_topics(bootstrap_servers: str, topics: List[dict]):
    """สร้าง Kafka topics"""
    admin_client = KafkaAdminClient(
        bootstrap_servers=bootstrap_servers,
        client_id="admin"
    )
    
    new_topics = [
        NewTopic(
            name=topic["name"],
            num_partitions=topic.get("partitions", 3),
            replication_factor=topic.get("replication", 1),
            topic_configs={
                "retention.ms": str(topic.get("retention_days", 7) * 86400000),
                "cleanup.policy": topic.get("cleanup_policy", "delete")
            }
        )
        for topic in topics
    ]
    
    try:
        admin_client.create_topics(new_topics=new_topics, validate_only=False)
        print(f"Created {len(new_topics)} topics")
    except Exception as e:
        print(f"Error creating topics (may already exist): {e}")
    finally:
        admin_client.close()


# ตัวอย่างการตั้งค่า topics
KAFKA_TOPICS = [
    {"name": "order-created", "partitions": 6, "replication": 1},
    {"name": "order-paid", "partitions": 6, "replication": 1},
    {"name": "payment-processed", "partitions": 3, "replication": 1},
    {"name": "user-registered", "partitions": 3, "replication": 1},
    {"name": "inventory-updated", "partitions": 6, "replication": 1},
]
```

---

## 5. Event Sourcing Pattern

```python
# event_sourcing/event_store.py
"""
Event Sourcing Pattern:
- เก็บ state ของ entity เป็น sequence of events
- แทนที่จะเก็บ current state, เก็บ history ของการเปลี่ยนแปลง
- สามารถ reconstruct state ณ จุดเวลาใดก็ได้

ข้อดี:
- Complete audit trail
- สามารถ replay events ได้
- เหมาะกับระบบที่ต้อง track history

ตัวอย่าง: Order Aggregate
"""
from dataclasses import dataclass, field, asdict
from typing import List, Dict, Any, Optional, Type
from datetime import datetime
import json
import uuid
import sqlite3


@dataclass
class DomainEvent:
    """Base class สำหรับ domain events"""
    event_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    aggregate_id: str = ""
    aggregate_type: str = ""
    event_type: str = ""
    version: int = 0
    timestamp: str = field(default_factory=lambda: datetime.utcnow().isoformat())
    data: Dict[str, Any] = field(default_factory=dict)
    
    def to_dict(self) -> dict:
        return asdict(self)


# Order domain events
@dataclass
class OrderCreated(DomainEvent):
    def __init__(self, aggregate_id: str, user_id: int, items: list):
        super().__init__()
        self.aggregate_id = aggregate_id
        self.aggregate_type = "Order"
        self.event_type = "OrderCreated"
        self.data = {"user_id": user_id, "items": items}


@dataclass
class OrderItemAdded(DomainEvent):
    def __init__(self, aggregate_id: str, product_id: int, quantity: int, price: float):
        super().__init__()
        self.aggregate_id = aggregate_id
        self.aggregate_type = "Order"
        self.event_type = "OrderItemAdded"
        self.data = {"product_id": product_id, "quantity": quantity, "price": price}


@dataclass
class OrderPaid(DomainEvent):
    def __init__(self, aggregate_id: str, payment_id: str, amount: float):
        super().__init__()
        self.aggregate_id = aggregate_id
        self.aggregate_type = "Order"
        self.event_type = "OrderPaid"
        self.data = {"payment_id": payment_id, "amount": amount}


@dataclass
class OrderShipped(DomainEvent):
    def __init__(self, aggregate_id: str, tracking_number: str):
        super().__init__()
        self.aggregate_id = aggregate_id
        self.aggregate_type = "Order"
        self.event_type = "OrderShipped"
        self.data = {"tracking_number": tracking_number}


class EventStore:
    """Event store สำหรับเก็บ domain events"""
    
    def __init__(self, db_path: str = ":memory:"):
        self.conn = sqlite3.connect(db_path, check_same_thread=False)
        self._init_schema()
    
    def _init_schema(self):
        """สร้างตาราง events"""
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS events (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                event_id TEXT UNIQUE NOT NULL,
                aggregate_id TEXT NOT NULL,
                aggregate_type TEXT NOT NULL,
                event_type TEXT NOT NULL,
                version INTEGER NOT NULL,
                timestamp TEXT NOT NULL,
                data TEXT NOT NULL,
                UNIQUE(aggregate_id, version)
            )
        """)
        self.conn.execute("""
            CREATE INDEX IF NOT EXISTS idx_aggregate 
            ON events(aggregate_id, version)
        """)
        self.conn.commit()
    
    def append(self, event: DomainEvent, expected_version: int = None):
        """เพิ่ม event เข้า store"""
        # ตรวจสอบ version conflict (Optimistic Concurrency)
        if expected_version is not None:
            current_version = self._get_current_version(event.aggregate_id)
            if current_version != expected_version:
                raise ValueError(
                    f"Version conflict for {event.aggregate_id}: "
                    f"expected {expected_version}, got {current_version}"
                )
        
        # หา version ถัดไป
        current_version = self._get_current_version(event.aggregate_id)
        event.version = current_version + 1
        
        self.conn.execute("""
            INSERT INTO events 
            (event_id, aggregate_id, aggregate_type, event_type, version, timestamp, data)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """, (
            event.event_id,
            event.aggregate_id,
            event.aggregate_type,
            event.event_type,
            event.version,
            event.timestamp,
            json.dumps(event.data)
        ))
        self.conn.commit()
    
    def get_events(
        self,
        aggregate_id: str,
        from_version: int = 0,
        to_version: int = None
    ) -> List[dict]:
        """ดึง events สำหรับ aggregate"""
        query = """
            SELECT event_id, aggregate_id, aggregate_type, event_type, 
                   version, timestamp, data
            FROM events
            WHERE aggregate_id = ? AND version > ?
        """
        params = [aggregate_id, from_version]
        
        if to_version:
            query += " AND version <= ?"
            params.append(to_version)
        
        query += " ORDER BY version ASC"
        
        cursor = self.conn.execute(query, params)
        rows = cursor.fetchall()
        
        return [
            {
                "event_id": row[0],
                "aggregate_id": row[1],
                "aggregate_type": row[2],
                "event_type": row[3],
                "version": row[4],
                "timestamp": row[5],
                "data": json.loads(row[6])
            }
            for row in rows
        ]
    
    def _get_current_version(self, aggregate_id: str) -> int:
        """ดึง version ล่าสุดของ aggregate"""
        cursor = self.conn.execute(
            "SELECT MAX(version) FROM events WHERE aggregate_id = ?",
            [aggregate_id]
        )
        result = cursor.fetchone()[0]
        return result or 0


class OrderAggregate:
    """Order aggregate ที่ใช้ Event Sourcing"""
    
    def __init__(self, order_id: str = None):
        self.order_id = order_id or str(uuid.uuid4())
        self.user_id: Optional[int] = None
        self.items: List[dict] = []
        self.status: str = "draft"
        self.total_amount: float = 0.0
        self.payment_id: Optional[str] = None
        self.tracking_number: Optional[str] = None
        self.version: int = 0
        self._pending_events: List[DomainEvent] = []
    
    @classmethod
    def from_events(cls, events: List[dict]) -> "OrderAggregate":
        """Reconstruct aggregate จาก events"""
        order = cls.__new__(cls)
        order.order_id = events[0]["aggregate_id"] if events else str(uuid.uuid4())
        order.user_id = None
        order.items = []
        order.status = "draft"
        order.total_amount = 0.0
        order.payment_id = None
        order.tracking_number = None
        order.version = 0
        order._pending_events = []
        
        for event_data in events:
            order._apply_event(event_data)
        
        return order
    
    def _apply_event(self, event_data: dict):
        """Apply event เพื่ออัพเดท state"""
        event_type = event_data["event_type"]
        data = event_data["data"]
        
        if event_type == "OrderCreated":
            self.user_id = data["user_id"]
            self.items = data.get("items", [])
            self.status = "created"
        
        elif event_type == "OrderItemAdded":
            self.items.append({
                "product_id": data["product_id"],
                "quantity": data["quantity"],
                "price": data["price"],
                "total": data["price"] * data["quantity"]
            })
            self.total_amount = sum(item["total"] for item in self.items)
        
        elif event_type == "OrderPaid":
            self.status = "paid"
            self.payment_id = data["payment_id"]
        
        elif event_type == "OrderShipped":
            self.status = "shipped"
            self.tracking_number = data["tracking_number"]
        
        self.version = event_data.get("version", self.version + 1)
    
    def create(self, user_id: int) -> "OrderAggregate":
        """สร้าง order ใหม่"""
        event = OrderCreated(
            aggregate_id=self.order_id,
            user_id=user_id,
            items=[]
        )
        self._apply_event(event.to_dict())
        self._pending_events.append(event)
        return self
    
    def add_item(self, product_id: int, quantity: int, price: float) -> "OrderAggregate":
        """เพิ่ม item ใน order"""
        if self.status != "created":
            raise ValueError(f"Cannot add item to order in status: {self.status}")
        
        event = OrderItemAdded(
            aggregate_id=self.order_id,
            product_id=product_id,
            quantity=quantity,
            price=price
        )
        self._apply_event(event.to_dict())
        self._pending_events.append(event)
        return self
    
    def pay(self, payment_id: str) -> "OrderAggregate":
        """บันทึกการชำระเงิน"""
        if self.status != "created":
            raise ValueError(f"Cannot pay order in status: {self.status}")
        
        event = OrderPaid(
            aggregate_id=self.order_id,
            payment_id=payment_id,
            amount=self.total_amount
        )
        self._apply_event(event.to_dict())
        self._pending_events.append(event)
        return self
    
    def ship(self, tracking_number: str) -> "OrderAggregate":
        """บันทึกการจัดส่ง"""
        if self.status != "paid":
            raise ValueError(f"Cannot ship order in status: {self.status}")
        
        event = OrderShipped(
            aggregate_id=self.order_id,
            tracking_number=tracking_number
        )
        self._apply_event(event.to_dict())
        self._pending_events.append(event)
        return self
    
    def get_pending_events(self) -> List[DomainEvent]:
        """ดึง events ที่ยังไม่ได้บันทึก"""
        return self._pending_events.copy()
    
    def clear_pending_events(self):
        """ล้าง pending events หลังจากบันทึกแล้ว"""
        self._pending_events.clear()


# ตัวอย่างการใช้งาน Event Sourcing
def event_sourcing_demo():
    store = EventStore()
    
    # สร้าง order
    order = OrderAggregate()
    order.create(user_id=123)
    order.add_item(product_id=1, quantity=2, price=299.99)
    order.add_item(product_id=3, quantity=1, price=499.99)
    
    # บันทึก events
    for event in order.get_pending_events():
        store.append(event)
    order.clear_pending_events()
    
    print(f"Order {order.order_id}: {order.status}, Total: {order.total_amount}")
    
    # Reconstruct order จาก events
    events = store.get_events(order.order_id)
    reconstructed = OrderAggregate.from_events(events)
    
    print(f"Reconstructed order: {reconstructed.status}, Items: {len(reconstructed.items)}")
    
    # ดู order ณ version 1 (ก่อนเพิ่ม items)
    old_events = store.get_events(order.order_id, to_version=1)
    old_order = OrderAggregate.from_events(old_events)
    print(f"Order at version 1: {old_order.status}, Items: {len(old_order.items)}")
```

---

## 6. Saga Pattern

```python
# saga/choreography_saga.py
"""
Saga Pattern สำหรับ Distributed Transactions

2 แบบหลัก:
1. Choreography: แต่ละ service react ต่อ events โดยตรง
2. Orchestration: มี central coordinator (Saga Orchestrator)

ตัวอย่าง: Order Processing Saga

Choreography Flow:
OrderService ──OrderCreated──► PaymentService
PaymentService ──PaymentProcessed──► InventoryService
InventoryService ──InventoryUpdated──► ShippingService
ShippingService ──OrderShipped──► NotificationService

Compensation Flow (เมื่อเกิด failure):
PaymentFailed ──► OrderService.cancelOrder()
InventoryUpdateFailed ──► PaymentService.refundPayment() + OrderService.cancelOrder()
"""
import asyncio
from typing import Callable, Dict, List, Optional
from dataclasses import dataclass
from enum import Enum
import uuid


class SagaStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    COMPENSATING = "compensating"
    FAILED = "failed"


@dataclass
class SagaStep:
    """ขั้นตอนหนึ่งใน Saga"""
    name: str
    action: Callable          # function ที่ execute
    compensation: Callable    # function สำหรับ rollback
    timeout: float = 30.0


class SagaOrchestrator:
    """
    Orchestration-based Saga
    - Central coordinator จัดการ flow ทั้งหมด
    - Explicit transaction management
    """
    
    def __init__(self, saga_id: str = None):
        self.saga_id = saga_id or str(uuid.uuid4())
        self.steps: List[SagaStep] = []
        self.completed_steps: List[str] = []
        self.status = SagaStatus.PENDING
        self.context: Dict = {}
    
    def add_step(self, step: SagaStep) -> "SagaOrchestrator":
        """เพิ่มขั้นตอนใน Saga"""
        self.steps.append(step)
        return self
    
    async def execute(self) -> bool:
        """Execute Saga ทุกขั้นตอน"""
        self.status = SagaStatus.RUNNING
        
        for step in self.steps:
            try:
                print(f"[Saga {self.saga_id}] Executing step: {step.name}")
                result = await asyncio.wait_for(
                    step.action(self.context),
                    timeout=step.timeout
                )
                
                # เก็บผลลัพธ์ไว้ใน context
                if isinstance(result, dict):
                    self.context.update(result)
                
                self.completed_steps.append(step.name)
                print(f"[Saga {self.saga_id}] Step {step.name} completed")
            
            except asyncio.TimeoutError:
                print(f"[Saga {self.saga_id}] Step {step.name} timed out")
                await self._compensate()
                return False
            
            except Exception as e:
                print(f"[Saga {self.saga_id}] Step {step.name} failed: {e}")
                await self._compensate()
                return False
        
        self.status = SagaStatus.COMPLETED
        print(f"[Saga {self.saga_id}] Completed successfully")
        return True
    
    async def _compensate(self):
        """Rollback ทุกขั้นตอนที่ทำไปแล้ว"""
        self.status = SagaStatus.COMPENSATING
        
        # Compensate ย้อนกลับจาก step ล่าสุด
        for step_name in reversed(self.completed_steps):
            step = next((s for s in self.steps if s.name == step_name), None)
            if step:
                try:
                    print(f"[Saga {self.saga_id}] Compensating step: {step.name}")
                    await step.compensation(self.context)
                    print(f"[Saga {self.saga_id}] Compensated step: {step.name}")
                except Exception as e:
                    print(f"[Saga {self.saga_id}] Compensation failed for {step.name}: {e}")
        
        self.status = SagaStatus.FAILED


# ตัวอย่าง Order Processing Saga
class OrderSagaActions:
    """Actions สำหรับ Order Processing Saga"""
    
    async def reserve_inventory(self, context: dict) -> dict:
        """จอง inventory"""
        order_id = context["order_id"]
        items = context["items"]
        
        # เรียก Inventory Service
        print(f"  Reserving inventory for order {order_id}")
        # reservation_id = await inventory_service.reserve(items)
        reservation_id = f"RES-{uuid.uuid4().hex[:8]}"
        
        return {"reservation_id": reservation_id}
    
    async def cancel_reservation(self, context: dict):
        """ยกเลิกการจอง inventory"""
        reservation_id = context.get("reservation_id")
        if reservation_id:
            print(f"  Cancelling inventory reservation: {reservation_id}")
            # await inventory_service.cancel_reservation(reservation_id)
    
    async def process_payment(self, context: dict) -> dict:
        """ประมวลผลการชำระเงิน"""
        order_id = context["order_id"]
        amount = context["total_amount"]
        
        print(f"  Processing payment {amount:.2f} THB for order {order_id}")
        # payment_id = await payment_service.charge(amount)
        payment_id = f"PAY-{uuid.uuid4().hex[:8]}"
        
        return {"payment_id": payment_id}
    
    async def refund_payment(self, context: dict):
        """คืนเงิน"""
        payment_id = context.get("payment_id")
        if payment_id:
            print(f"  Refunding payment: {payment_id}")
            # await payment_service.refund(payment_id)
    
    async def create_shipment(self, context: dict) -> dict:
        """สร้างการจัดส่ง"""
        order_id = context["order_id"]
        
        print(f"  Creating shipment for order {order_id}")
        # shipment_id = await shipping_service.create(order_id)
        shipment_id = f"SHIP-{uuid.uuid4().hex[:8]}"
        
        return {"shipment_id": shipment_id}
    
    async def cancel_shipment(self, context: dict):
        """ยกเลิกการจัดส่ง"""
        shipment_id = context.get("shipment_id")
        if shipment_id:
            print(f"  Cancelling shipment: {shipment_id}")
            # await shipping_service.cancel(shipment_id)
    
    async def send_confirmation(self, context: dict):
        """ส่ง confirmation notification"""
        order_id = context["order_id"]
        print(f"  Sending confirmation for order {order_id}")
        # await notification_service.send_confirmation(order_id)


async def process_order_with_saga(order_id: str, items: list, total_amount: float):
    """Process order โดยใช้ Saga pattern"""
    actions = OrderSagaActions()
    
    saga = SagaOrchestrator(saga_id=f"order-{order_id}")
    
    saga.add_step(SagaStep(
        name="reserve_inventory",
        action=actions.reserve_inventory,
        compensation=actions.cancel_reservation
    ))
    
    saga.add_step(SagaStep(
        name="process_payment",
        action=actions.process_payment,
        compensation=actions.refund_payment
    ))
    
    saga.add_step(SagaStep(
        name="create_shipment",
        action=actions.create_shipment,
        compensation=actions.cancel_shipment
    ))
    
    saga.add_step(SagaStep(
        name="send_confirmation",
        action=actions.send_confirmation,
        compensation=lambda ctx: None  # ไม่ต้อง compensate notification
    ))
    
    # ตั้งค่า context เริ่มต้น
    saga.context = {
        "order_id": order_id,
        "items": items,
        "total_amount": total_amount
    }
    
    success = await saga.execute()
    
    if success:
        print(f"\n✅ Order {order_id} processed successfully!")
        print(f"   Payment: {saga.context.get('payment_id')}")
        print(f"   Shipment: {saga.context.get('shipment_id')}")
    else:
        print(f"\n❌ Order {order_id} failed. All compensations executed.")
    
    return success


# ทดสอบ Saga
if __name__ == "__main__":
    asyncio.run(process_order_with_saga(
        order_id="ORD-12345",
        items=[{"product_id": 1, "quantity": 2}],
        total_amount=599.98
    ))
```

---

## 7. สรุป Part 097

✅ เข้าใจ Event-Driven Architecture และ Message Queue concepts
✅ ใช้งาน RabbitMQ ด้วย pika สำหรับ task queue และ publish/subscribe
✅ ใช้ Redis Pub/Sub สำหรับ real-time messaging และ WebSocket integration
✅ ใช้ Apache Kafka กับ kafka-python สำหรับ high-throughput event streaming
✅ ออกแบบ Event Sourcing pattern ด้วย event store และ aggregate reconstruction
✅ ใช้ Saga Orchestration pattern สำหรับ distributed transactions พร้อม compensation

## ➡️ ถัดไป: Part 098 - System Design for Python Apps

*Part 097/105 | Python Course - World-Class Level*
