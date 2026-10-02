# Part 095: FastAPI WebSockets
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- สร้าง WebSocket endpoint
- จัดการ WebSocket connection lifecycle
- สร้าง ConnectionManager สำหรับหลาย clients
- Broadcast messages ไปยังทุก clients
- สร้าง room/channel concept
- Authentication ใน WebSockets
- สร้าง real-time chat application สมบูรณ์
- Error handling ใน WebSockets

---

## 1. WebSocket พื้นฐาน

WebSocket เป็น protocol ที่ช่วยให้สื่อสารแบบ bidirectional real-time ระหว่าง client และ server ต่างจาก HTTP ที่เป็น request-response

```python
# basic_websocket.py

from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import HTMLResponse

app = FastAPI()


# === WebSocket Endpoint พื้นฐาน ===

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    """
    WebSocket endpoint พื้นฐาน
    
    Lifecycle:
    1. accept() - รับการเชื่อมต่อ
    2. รับ/ส่งข้อความ
    3. disconnect เมื่อ client ปิด connection
    """
    # รับการเชื่อมต่อ
    await websocket.accept()

    try:
        while True:
            # รับข้อความจาก client
            data = await websocket.receive_text()

            # ส่งข้อความกลับ (echo)
            await websocket.send_text(f"Server รับว่า: {data}")

    except WebSocketDisconnect:
        # Client ปิด connection
        print("Client ตัดการเชื่อมต่อ")


# === รับ JSON Data ===

@app.websocket("/ws/json")
async def websocket_json(websocket: WebSocket):
    """WebSocket ที่รับและส่ง JSON"""
    await websocket.accept()

    try:
        while True:
            # รับ JSON
            data = await websocket.receive_json()

            # ประมวลผล
            message_type = data.get("type", "unknown")
            message_content = data.get("content", "")

            # ส่งกลับ
            response = {
                "type": "response",
                "original_type": message_type,
                "content": f"ได้รับ: {message_content}",
                "processed": True
            }
            await websocket.send_json(response)

    except WebSocketDisconnect:
        print(f"Client ตัดการเชื่อมต่อ")


# === รับ Binary Data ===

@app.websocket("/ws/binary")
async def websocket_binary(websocket: WebSocket):
    """WebSocket สำหรับ binary data (ไฟล์, รูปภาพ)"""
    await websocket.accept()

    try:
        while True:
            # รับ bytes
            data = await websocket.receive_bytes()

            # ประมวลผล binary data
            size = len(data)
            print(f"รับ binary data: {size} bytes")

            # ส่งกลับ (acknowledge)
            await websocket.send_json({
                "received": True,
                "size": size
            })

    except WebSocketDisconnect:
        pass


# === Demo HTML Page ===

@app.get("/", response_class=HTMLResponse)
def websocket_demo():
    """หน้าทดสอบ WebSocket"""
    return HTMLResponse("""
    <!DOCTYPE html>
    <html lang="th">
    <head>
        <meta charset="UTF-8">
        <title>WebSocket Demo</title>
        <style>
            body { font-family: sans-serif; max-width: 600px; margin: 40px auto; padding: 0 20px; }
            #messages { border: 1px solid #ccc; height: 300px; overflow-y: auto; padding: 10px; margin: 10px 0; }
            #messageInput { width: 70%; padding: 8px; }
            button { padding: 8px 16px; cursor: pointer; margin: 2px; }
            .received { color: blue; }
            .sent { color: green; }
            .system { color: orange; font-style: italic; }
        </style>
    </head>
    <body>
        <h1>WebSocket Demo</h1>
        <button onclick="connect()">เชื่อมต่อ</button>
        <button onclick="disconnect()">ตัดการเชื่อมต่อ</button>
        <p id="status">สถานะ: ยังไม่เชื่อมต่อ</p>

        <div id="messages"></div>
        <input type="text" id="messageInput" placeholder="พิมพ์ข้อความ..." />
        <button onclick="sendMessage()">ส่ง</button>

        <script>
            let ws = null;

            function connect() {
                ws = new WebSocket('ws://localhost:8000/ws');

                ws.onopen = () => {
                    document.getElementById('status').textContent = 'สถานะ: เชื่อมต่อแล้ว ✅';
                    addMessage('เชื่อมต่อสำเร็จ', 'system');
                };

                ws.onmessage = (event) => {
                    addMessage(`Server: ${event.data}`, 'received');
                };

                ws.onclose = () => {
                    document.getElementById('status').textContent = 'สถานะ: ตัดการเชื่อมต่อแล้ว';
                    addMessage('ตัดการเชื่อมต่อ', 'system');
                };

                ws.onerror = (error) => {
                    addMessage(`Error: ${error}`, 'system');
                };
            }

            function disconnect() {
                if (ws) ws.close();
            }

            function sendMessage() {
                const input = document.getElementById('messageInput');
                const message = input.value.trim();
                if (message && ws && ws.readyState === WebSocket.OPEN) {
                    ws.send(message);
                    addMessage(`You: ${message}`, 'sent');
                    input.value = '';
                }
            }

            function addMessage(text, type) {
                const div = document.getElementById('messages');
                const p = document.createElement('p');
                p.className = type;
                p.textContent = `[${new Date().toLocaleTimeString('th-TH')}] ${text}`;
                div.appendChild(p);
                div.scrollTop = div.scrollHeight;
            }

            // กด Enter เพื่อส่ง
            document.getElementById('messageInput').addEventListener('keypress', (e) => {
                if (e.key === 'Enter') sendMessage();
            });
        </script>
    </body>
    </html>
    """)
```

---

## 2. WebSocket Connection Lifecycle

```python
# ws_lifecycle.py

from fastapi import FastAPI, WebSocket, WebSocketDisconnect, WebSocketException, status
import json
import asyncio
from datetime import datetime
from typing import Optional

app = FastAPI()


@app.websocket("/ws/lifecycle")
async def websocket_lifecycle(websocket: WebSocket):
    """แสดง lifecycle ทั้งหมดของ WebSocket"""

    client_host = websocket.client.host if websocket.client else "unknown"
    client_port = websocket.client.port if websocket.client else 0
    connection_time = datetime.utcnow()

    print(f"[{connection_time}] Client เชื่อมต่อ: {client_host}:{client_port}")
    print(f"Headers: {dict(websocket.headers)}")
    print(f"Query params: {dict(websocket.query_params)}")
    print(f"Path params: {dict(websocket.path_params)}")

    # ขั้นตอน 1: Accept connection
    await websocket.accept()
    print("Connection accepted")

    # ขั้นตอน 2: ส่ง welcome message
    await websocket.send_json({
        "type": "connected",
        "client": f"{client_host}:{client_port}",
        "timestamp": connection_time.isoformat(),
        "message": "ยินดีต้อนรับ!"
    })

    try:
        message_count = 0

        while True:
            # รับข้อความ - รองรับทั้ง text และ binary
            message = await websocket.receive()

            # ตรวจสอบประเภทข้อความ
            if "text" in message:
                text = message["text"]
                message_count += 1

                print(f"Received text ({message_count}): {text[:50]}")

                # ประมวลผลและตอบกลับ
                try:
                    data = json.loads(text)
                    msg_type = data.get("type", "unknown")

                    if msg_type == "ping":
                        await websocket.send_json({
                            "type": "pong",
                            "timestamp": datetime.utcnow().isoformat()
                        })
                    elif msg_type == "close":
                        await websocket.send_json({
                            "type": "closing",
                            "message": "Server กำลังปิด connection..."
                        })
                        # ปิด connection อย่างปลอดภัย
                        await websocket.close(code=1000, reason="Client requested close")
                        break
                    else:
                        await websocket.send_json({
                            "type": "echo",
                            "data": data,
                            "count": message_count
                        })

                except json.JSONDecodeError:
                    # ข้อความไม่ใช่ JSON
                    await websocket.send_text(f"Echo: {text} (message #{message_count})")

            elif "bytes" in message:
                binary_data = message["bytes"]
                print(f"Received binary: {len(binary_data)} bytes")
                await websocket.send_json({
                    "type": "binary_received",
                    "size": len(binary_data)
                })

    except WebSocketDisconnect as e:
        # Client ตัดการเชื่อมต่อ
        disconnect_time = datetime.utcnow()
        duration = (disconnect_time - connection_time).total_seconds()
        print(f"Client ตัดการเชื่อมต่อ: code={e.code}, reason={e.reason}")
        print(f"เวลาเชื่อมต่อ: {duration:.1f} วินาที, รับข้อความ: {message_count} ข้อความ")

    except Exception as e:
        print(f"Unexpected error: {e}")
        try:
            await websocket.close(code=1011, reason="Internal server error")
        except:
            pass


# WebSocket พร้อม query parameters
@app.websocket("/ws/params")
async def websocket_with_params(
    websocket: WebSocket,
    token: Optional[str] = None,
    room: Optional[str] = "general"
):
    """WebSocket ที่รับ query parameters"""
    # ตรวจสอบ token (ง่ายๆ)
    if not token or token != "valid-token":
        # ปฏิเสธ connection
        await websocket.close(code=1008, reason="Unauthorized")
        return

    await websocket.accept()
    await websocket.send_json({
        "type": "connected",
        "room": room,
        "message": f"เข้าร่วมห้อง {room} แล้ว"
    })

    try:
        while True:
            data = await websocket.receive_json()
            await websocket.send_json({
                "type": "echo",
                "room": room,
                "data": data
            })
    except WebSocketDisconnect:
        pass
```

---

## 3. ConnectionManager สำหรับหลาย Clients

```python
# connection_manager.py

from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict, List, Set, Optional
import json
import asyncio
from datetime import datetime
import uuid

app = FastAPI()


class ConnectionManager:
    """
    จัดการ WebSocket connections หลายๆ อัน
    Thread-safe สำหรับ asyncio
    """

    def __init__(self):
        # เก็บ active connections: {connection_id: websocket}
        self.active_connections: Dict[str, WebSocket] = {}
        # เก็บข้อมูล client: {connection_id: metadata}
        self.client_data: Dict[str, dict] = {}

    async def connect(self, websocket: WebSocket, client_id: str = None) -> str:
        """
        รับ connection ใหม่
        Returns: connection_id
        """
        await websocket.accept()

        # สร้าง unique ID
        connection_id = client_id or str(uuid.uuid4())[:8]

        self.active_connections[connection_id] = websocket
        self.client_data[connection_id] = {
            "connected_at": datetime.utcnow().isoformat(),
            "message_count": 0,
            "host": websocket.client.host if websocket.client else "unknown"
        }

        print(f"Client {connection_id} เชื่อมต่อ | รวม: {self.count} clients")
        return connection_id

    def disconnect(self, connection_id: str):
        """ลบ connection"""
        if connection_id in self.active_connections:
            del self.active_connections[connection_id]
        if connection_id in self.client_data:
            del self.client_data[connection_id]
        print(f"Client {connection_id} ตัดการเชื่อมต่อ | เหลือ: {self.count} clients")

    @property
    def count(self) -> int:
        """จำนวน connections ที่ active"""
        return len(self.active_connections)

    @property
    def connection_ids(self) -> List[str]:
        """รายการ connection IDs"""
        return list(self.active_connections.keys())

    async def send_personal_message(self, message: str, connection_id: str):
        """ส่งข้อความไปยัง client คนเดียว"""
        if connection_id in self.active_connections:
            websocket = self.active_connections[connection_id]
            try:
                await websocket.send_text(message)
                # อัปเดต message count
                if connection_id in self.client_data:
                    self.client_data[connection_id]["message_count"] += 1
            except Exception as e:
                print(f"Error sending to {connection_id}: {e}")
                self.disconnect(connection_id)

    async def send_personal_json(self, data: dict, connection_id: str):
        """ส่ง JSON ไปยัง client คนเดียว"""
        await self.send_personal_message(json.dumps(data, ensure_ascii=False), connection_id)

    async def broadcast(self, message: str, exclude: Set[str] = None):
        """ส่งข้อความไปยังทุก clients"""
        exclude = exclude or set()
        disconnected = []

        for connection_id, websocket in self.active_connections.items():
            if connection_id in exclude:
                continue
            try:
                await websocket.send_text(message)
            except Exception:
                disconnected.append(connection_id)

        # ลบ connections ที่ fail
        for cid in disconnected:
            self.disconnect(cid)

    async def broadcast_json(self, data: dict, exclude: Set[str] = None):
        """Broadcast JSON ไปยังทุก clients"""
        await self.broadcast(json.dumps(data, ensure_ascii=False), exclude)

    def get_client_info(self, connection_id: str) -> Optional[dict]:
        """ดึงข้อมูลของ client"""
        return self.client_data.get(connection_id)

    def get_all_clients_info(self) -> dict:
        """ดึงข้อมูลของทุก clients"""
        return {
            "total": self.count,
            "clients": [
                {"id": cid, **info}
                for cid, info in self.client_data.items()
            ]
        }


# Singleton manager
manager = ConnectionManager()


@app.websocket("/ws/{client_id}")
async def websocket_with_manager(websocket: WebSocket, client_id: str):
    """WebSocket ที่ใช้ ConnectionManager"""
    connection_id = await manager.connect(websocket, client_id)

    # แจ้ง clients อื่นๆ ว่ามีคนเข้ามา
    await manager.broadcast_json(
        {
            "type": "user_joined",
            "user": connection_id,
            "online_count": manager.count,
            "online_users": manager.connection_ids
        },
        exclude={connection_id}  # ไม่ส่งให้ตัวเอง
    )

    # แจ้ง client ใหม่
    await manager.send_personal_json(
        {
            "type": "welcome",
            "your_id": connection_id,
            "online_users": manager.connection_ids
        },
        connection_id
    )

    try:
        while True:
            data = await websocket.receive_json()
            msg_type = data.get("type")

            if msg_type == "message":
                # Broadcast message ไปทุกคน
                await manager.broadcast_json({
                    "type": "message",
                    "from": connection_id,
                    "content": data.get("content", ""),
                    "timestamp": datetime.utcnow().isoformat()
                })

            elif msg_type == "private_message":
                # ส่ง private message
                target = data.get("to")
                if target in manager.active_connections:
                    await manager.send_personal_json(
                        {
                            "type": "private_message",
                            "from": connection_id,
                            "content": data.get("content", ""),
                            "timestamp": datetime.utcnow().isoformat()
                        },
                        target
                    )
                else:
                    await manager.send_personal_json(
                        {"type": "error", "message": f"ไม่พบผู้ใช้ {target}"},
                        connection_id
                    )

            elif msg_type == "get_users":
                # ดูรายการ users
                await manager.send_personal_json(
                    manager.get_all_clients_info(),
                    connection_id
                )

    except WebSocketDisconnect:
        manager.disconnect(connection_id)

        # แจ้ง clients อื่นๆ ว่ามีคนออกไป
        await manager.broadcast_json({
            "type": "user_left",
            "user": connection_id,
            "online_count": manager.count,
            "online_users": manager.connection_ids
        })


@app.get("/connections/")
def get_connections():
    """REST endpoint ดูจำนวน WebSocket connections"""
    return manager.get_all_clients_info()
```

---

## 4. Broadcasting Messages

```python
# broadcasting.py

from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict, List, Set
import json
import asyncio
from datetime import datetime

app = FastAPI()


class BroadcastManager:
    """Manager สำหรับ broadcasting กับ features เพิ่มเติม"""

    def __init__(self):
        self.connections: Dict[str, WebSocket] = {}
        self.message_history: List[dict] = []  # เก็บประวัติข้อความ
        self.max_history = 50  # เก็บแค่ 50 ข้อความล่าสุด

    async def connect(self, websocket: WebSocket, user_id: str):
        await websocket.accept()
        self.connections[user_id] = websocket

        # ส่งประวัติข้อความให้ client ใหม่
        if self.message_history:
            await websocket.send_json({
                "type": "history",
                "messages": self.message_history[-20:]  # 20 ข้อความล่าสุด
            })

    def disconnect(self, user_id: str):
        self.connections.pop(user_id, None)

    def add_to_history(self, message: dict):
        """เพิ่มข้อความลง history"""
        self.message_history.append(message)
        # ตัด history เก่าออก
        if len(self.message_history) > self.max_history:
            self.message_history = self.message_history[-self.max_history:]

    async def broadcast(self, message: dict, exclude: Set[str] = None):
        """Broadcast ไปทุก clients"""
        exclude = exclude or set()
        message_str = json.dumps(message, ensure_ascii=False)
        failed = []

        for user_id, ws in self.connections.items():
            if user_id in exclude:
                continue
            try:
                await ws.send_text(message_str)
            except Exception:
                failed.append(user_id)

        for user_id in failed:
            self.disconnect(user_id)

    async def broadcast_to_list(self, message: dict, user_ids: List[str]):
        """Broadcast ไปยัง users ที่ระบุ"""
        message_str = json.dumps(message, ensure_ascii=False)
        for user_id in user_ids:
            if user_id in self.connections:
                try:
                    await self.connections[user_id].send_text(message_str)
                except Exception:
                    self.disconnect(user_id)

    async def send_to(self, user_id: str, message: dict):
        """ส่งไปยัง user คนเดียว"""
        if user_id in self.connections:
            try:
                await self.connections[user_id].send_json(message)
            except Exception:
                self.disconnect(user_id)

    @property
    def online_users(self) -> List[str]:
        return list(self.connections.keys())


broadcast_manager = BroadcastManager()


@app.websocket("/chat/{username}")
async def chat_endpoint(websocket: WebSocket, username: str):
    """Chat endpoint พร้อม broadcast"""
    await broadcast_manager.connect(websocket, username)

    # แจ้ง online users ว่ามีคนเข้า
    join_msg = {
        "type": "system",
        "content": f"{username} เข้าร่วมห้องสนทนา",
        "online_users": broadcast_manager.online_users,
        "timestamp": datetime.utcnow().isoformat()
    }
    broadcast_manager.add_to_history(join_msg)
    await broadcast_manager.broadcast(join_msg)

    try:
        while True:
            data = await websocket.receive_json()
            content = data.get("content", "").strip()

            if not content:
                continue

            # สร้าง message
            message = {
                "type": "chat",
                "username": username,
                "content": content,
                "timestamp": datetime.utcnow().isoformat()
            }

            # บันทึกประวัติและ broadcast
            broadcast_manager.add_to_history(message)
            await broadcast_manager.broadcast(message)

    except WebSocketDisconnect:
        broadcast_manager.disconnect(username)

        leave_msg = {
            "type": "system",
            "content": f"{username} ออกจากห้องสนทนา",
            "online_users": broadcast_manager.online_users,
            "timestamp": datetime.utcnow().isoformat()
        }
        broadcast_manager.add_to_history(leave_msg)
        await broadcast_manager.broadcast(leave_msg)


@app.get("/chat/history/")
def get_history():
    """ดูประวัติข้อความ (REST endpoint)"""
    return {
        "messages": broadcast_manager.message_history,
        "online_users": broadcast_manager.online_users
    }
```

---

## 5. Room/Channel Concept

```python
# rooms.py

from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict, Set, List, Optional
import json
from datetime import datetime

app = FastAPI()


class Room:
    """ห้องสนทนา"""

    def __init__(self, room_id: str, name: str, is_private: bool = False):
        self.room_id = room_id
        self.name = name
        self.is_private = is_private
        self.members: Dict[str, WebSocket] = {}  # {username: websocket}
        self.message_history: List[dict] = []

    async def add_member(self, username: str, websocket: WebSocket):
        """เพิ่มสมาชิกในห้อง"""
        self.members[username] = websocket

        # ส่งประวัติข้อความ
        if self.message_history:
            await websocket.send_json({
                "type": "room_history",
                "room_id": self.room_id,
                "messages": self.message_history[-30:]
            })

        # แจ้งสมาชิกอื่น
        await self.broadcast({
            "type": "member_joined",
            "room_id": self.room_id,
            "username": username,
            "member_count": len(self.members),
            "members": list(self.members.keys())
        }, exclude={username})

    async def remove_member(self, username: str):
        """ลบสมาชิกออกจากห้อง"""
        self.members.pop(username, None)

        await self.broadcast({
            "type": "member_left",
            "room_id": self.room_id,
            "username": username,
            "member_count": len(self.members),
            "members": list(self.members.keys())
        })

    async def broadcast(self, message: dict, exclude: Set[str] = None):
        """ส่งข้อความไปทุกคนในห้อง"""
        exclude = exclude or set()
        message_str = json.dumps(message, ensure_ascii=False)
        disconnected = []

        for username, ws in self.members.items():
            if username in exclude:
                continue
            try:
                await ws.send_text(message_str)
            except:
                disconnected.append(username)

        for username in disconnected:
            await self.remove_member(username)

    async def send_to_member(self, username: str, message: dict):
        """ส่งข้อความไปยังสมาชิกคนเดียว"""
        if username in self.members:
            try:
                await self.members[username].send_json(message)
            except:
                await self.remove_member(username)

    def add_to_history(self, message: dict):
        self.message_history.append(message)
        if len(self.message_history) > 100:
            self.message_history = self.message_history[-100:]

    @property
    def is_empty(self) -> bool:
        return len(self.members) == 0

    def to_info(self) -> dict:
        return {
            "room_id": self.room_id,
            "name": self.name,
            "is_private": self.is_private,
            "member_count": len(self.members),
            "members": list(self.members.keys())
        }


class RoomManager:
    """จัดการห้องสนทนาทั้งหมด"""

    def __init__(self):
        self.rooms: Dict[str, Room] = {}

        # สร้างห้องเริ่มต้น
        self.create_room("general", "ห้องทั่วไป")
        self.create_room("tech", "ห้องเทคโนโลยี")
        self.create_room("random", "ห้องสุ่ม")

    def create_room(self, room_id: str, name: str, is_private: bool = False) -> Room:
        """สร้างห้องใหม่"""
        room = Room(room_id, name, is_private)
        self.rooms[room_id] = room
        return room

    def get_room(self, room_id: str) -> Optional[Room]:
        """ดึงห้องตาม ID"""
        return self.rooms.get(room_id)

    def get_or_create_room(self, room_id: str, name: str = None) -> Room:
        """ดึงหรือสร้างห้อง"""
        if room_id not in self.rooms:
            self.create_room(room_id, name or room_id)
        return self.rooms[room_id]

    def list_rooms(self) -> List[dict]:
        """รายการห้องทั้งหมด (ไม่รวม private)"""
        return [
            room.to_info()
            for room in self.rooms.values()
            if not room.is_private
        ]

    def delete_empty_rooms(self, protected_rooms: Set[str] = None):
        """ลบห้องที่ว่างเปล่า (ยกเว้นห้องที่ protected)"""
        protected = protected_rooms or {"general", "tech", "random"}
        to_delete = [
            room_id for room_id, room in self.rooms.items()
            if room.is_empty and room_id not in protected
        ]
        for room_id in to_delete:
            del self.rooms[room_id]


room_manager = RoomManager()


@app.websocket("/ws/room/{room_id}/{username}")
async def room_websocket(websocket: WebSocket, room_id: str, username: str):
    """WebSocket endpoint สำหรับเข้าร่วมห้อง"""
    # ดึงหรือสร้างห้อง
    room = room_manager.get_or_create_room(room_id, f"ห้อง {room_id}")

    # เข้าร่วมห้อง
    await room.add_member(username, websocket)

    # ส่ง room info ให้ user
    await room.send_to_member(username, {
        "type": "joined_room",
        "room": room.to_info()
    })

    try:
        while True:
            data = await websocket.receive_json()
            msg_type = data.get("type")

            if msg_type == "message":
                message = {
                    "type": "message",
                    "room_id": room_id,
                    "username": username,
                    "content": data.get("content", ""),
                    "timestamp": datetime.utcnow().isoformat()
                }
                room.add_to_history(message)
                await room.broadcast(message)

            elif msg_type == "switch_room":
                # ย้ายไปห้องอื่น
                new_room_id = data.get("room_id")
                if new_room_id:
                    # ออกจากห้องเดิม
                    await room.remove_member(username)

                    # เข้าร่วมห้องใหม่
                    room = room_manager.get_or_create_room(new_room_id)
                    await room.add_member(username, websocket)
                    await room.send_to_member(username, {
                        "type": "switched_room",
                        "room": room.to_info()
                    })

            elif msg_type == "list_rooms":
                await room.send_to_member(username, {
                    "type": "room_list",
                    "rooms": room_manager.list_rooms()
                })

    except WebSocketDisconnect:
        await room.remove_member(username)
        room_manager.delete_empty_rooms()


@app.get("/rooms/")
def list_rooms():
    """REST endpoint ดูรายการห้อง"""
    return room_manager.list_rooms()


@app.post("/rooms/")
def create_room(room_id: str, name: str):
    """REST endpoint สร้างห้องใหม่"""
    if room_id in room_manager.rooms:
        from fastapi import HTTPException
        raise HTTPException(status_code=400, detail="ห้องนี้มีอยู่แล้ว")
    room = room_manager.create_room(room_id, name)
    return room.to_info()
```

---

## 6. Authentication ใน WebSockets

```python
# ws_auth.py

from fastapi import FastAPI, WebSocket, WebSocketDisconnect, HTTPException, Query, Header
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt
from typing import Optional, Dict
import json
from datetime import datetime

app = FastAPI()

SECRET_KEY = "secret-key-for-websocket"
ALGORITHM = "HS256"

# Mock user data
USERS = {
    "alice": {"id": 1, "username": "alice", "role": "admin"},
    "bob": {"id": 2, "username": "bob", "role": "user"},
}


def verify_ws_token(token: str) -> Optional[dict]:
    """ตรวจสอบ JWT token สำหรับ WebSocket"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username = payload.get("sub")
        if username and username in USERS:
            return USERS[username]
    except JWTError:
        pass
    return None


# === Method 1: Token ใน Query Parameter ===
@app.websocket("/ws/auth/query")
async def ws_auth_query(
    websocket: WebSocket,
    token: str = Query(..., description="JWT token")
):
    """Authentication ด้วย token ใน query parameter"""
    user = verify_ws_token(token)

    if not user:
        # ปฏิเสธ connection
        await websocket.close(code=4001, reason="Unauthorized: Invalid token")
        return

    await websocket.accept()
    await websocket.send_json({
        "type": "authenticated",
        "user": user["username"],
        "role": user["role"]
    })

    try:
        while True:
            data = await websocket.receive_json()
            await websocket.send_json({
                "type": "echo",
                "user": user["username"],
                "data": data
            })
    except WebSocketDisconnect:
        pass


# === Method 2: Token ใน Header (ผ่าน Sec-WebSocket-Protocol) ===
@app.websocket("/ws/auth/header")
async def ws_auth_header(websocket: WebSocket):
    """Authentication ด้วย token ใน header"""
    # WebSocket ส่ง token ผ่าน Sec-WebSocket-Protocol subprotocol
    token = None

    # ดึงจาก headers
    auth_header = websocket.headers.get("authorization")
    if auth_header and auth_header.startswith("Bearer "):
        token = auth_header.split(" ")[1]

    # หรือจาก Sec-WebSocket-Protocol
    if not token:
        protocols = websocket.headers.get("sec-websocket-protocol", "")
        for protocol in protocols.split(","):
            protocol = protocol.strip()
            if protocol.startswith("access_token."):
                token = protocol.replace("access_token.", "")
                break

    user = verify_ws_token(token) if token else None

    if not user:
        await websocket.close(code=4001, reason="Unauthorized")
        return

    await websocket.accept()

    try:
        while True:
            data = await websocket.receive_json()
            await websocket.send_json({
                "type": "response",
                "authenticated_as": user["username"],
                "data": data
            })
    except WebSocketDisconnect:
        pass


# === Method 3: First Message Authentication ===
@app.websocket("/ws/auth/first-message")
async def ws_auth_first_message(websocket: WebSocket):
    """
    Authentication ด้วยข้อความแรก
    Client ต้องส่ง token เป็นข้อความแรกก่อนเสมอ
    """
    await websocket.accept()

    # รอ authentication message
    try:
        auth_data = await websocket.receive_json()

        if auth_data.get("type") != "auth":
            await websocket.send_json({
                "type": "error",
                "message": "ต้อง authenticate ก่อน"
            })
            await websocket.close(code=4001)
            return

        token = auth_data.get("token")
        user = verify_ws_token(token) if token else None

        if not user:
            await websocket.send_json({
                "type": "error",
                "message": "Token ไม่ถูกต้อง"
            })
            await websocket.close(code=4001)
            return

        # Authentication สำเร็จ
        await websocket.send_json({
            "type": "authenticated",
            "user": user["username"],
            "message": "ยืนยันตัวตนสำเร็จ"
        })

        # รับข้อความต่อไป
        while True:
            data = await websocket.receive_json()
            await websocket.send_json({
                "type": "message",
                "from": user["username"],
                "role": user["role"],
                "data": data
            })

    except WebSocketDisconnect:
        pass


# === Protected Room Manager ===

class AuthenticatedRoomManager:
    """Room Manager ที่ต้องการ authentication"""

    def __init__(self):
        self.rooms: Dict[str, Dict[str, dict]] = {}  # {room: {username: {ws, user_info}}}

    async def join(self, room: str, username: str, websocket: WebSocket, user_info: dict):
        if room not in self.rooms:
            self.rooms[room] = {}
        self.rooms[room][username] = {
            "websocket": websocket,
            "user_info": user_info
        }
        await self.broadcast_to_room(room, {
            "type": "user_joined",
            "username": username,
            "role": user_info["role"],
            "online": list(self.rooms[room].keys())
        })

    async def leave(self, room: str, username: str):
        if room in self.rooms and username in self.rooms[room]:
            del self.rooms[room][username]
            if not self.rooms[room]:
                del self.rooms[room]
            else:
                await self.broadcast_to_room(room, {
                    "type": "user_left",
                    "username": username,
                    "online": list(self.rooms.get(room, {}).keys())
                })

    async def broadcast_to_room(self, room: str, message: dict):
        if room not in self.rooms:
            return
        msg_str = json.dumps(message, ensure_ascii=False)
        for username, data in list(self.rooms[room].items()):
            try:
                await data["websocket"].send_text(msg_str)
            except:
                await self.leave(room, username)


auth_room_manager = AuthenticatedRoomManager()


@app.websocket("/ws/secure-room/{room_id}")
async def secure_room(
    websocket: WebSocket,
    room_id: str,
    token: str = Query(...)
):
    """ห้องที่ต้องการ authentication"""
    user = verify_ws_token(token)

    if not user:
        await websocket.close(code=4001, reason="Unauthorized")
        return

    # Admin-only rooms
    if room_id.startswith("admin-") and user["role"] != "admin":
        await websocket.close(code=4003, reason="Forbidden: Admin only")
        return

    await websocket.accept()
    await auth_room_manager.join(room_id, user["username"], websocket, user)

    await websocket.send_json({
        "type": "joined",
        "room": room_id,
        "user": user["username"]
    })

    try:
        while True:
            data = await websocket.receive_json()
            if data.get("type") == "message":
                await auth_room_manager.broadcast_to_room(room_id, {
                    "type": "message",
                    "room": room_id,
                    "from": user["username"],
                    "role": user["role"],
                    "content": data.get("content", ""),
                    "timestamp": datetime.utcnow().isoformat()
                })
    except WebSocketDisconnect:
        await auth_room_manager.leave(room_id, user["username"])
```

---

## 7. Complete Real-Time Chat Application

```python
# chat_app.py - Real-time Chat Application สมบูรณ์

from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Query, HTTPException
from fastapi.responses import HTMLResponse
from pydantic import BaseModel
from typing import Dict, List, Optional, Set
from datetime import datetime
import json
import uuid
import asyncio
from enum import Enum

app = FastAPI(title="Real-Time Chat Application")


# === Enums ===

class MessageType(str, Enum):
    TEXT = "text"
    IMAGE = "image"
    FILE = "file"
    SYSTEM = "system"


class UserStatus(str, Enum):
    ONLINE = "online"
    AWAY = "away"
    OFFLINE = "offline"


# === Models ===

class ChatMessage(BaseModel):
    id: str
    type: MessageType
    content: str
    username: str
    room_id: str
    timestamp: datetime
    edited: bool = False
    reply_to: Optional[str] = None  # message ID ที่ reply


class ChatUser(BaseModel):
    id: str
    username: str
    status: UserStatus = UserStatus.ONLINE
    joined_at: datetime
    rooms: List[str] = []


# === Chat Manager ===

class ChatApplication:
    """ระบบ Chat สมบูรณ์"""

    def __init__(self):
        # Users: {user_id: {ws, user_info, rooms}}
        self.users: Dict[str, dict] = {}

        # Rooms: {room_id: {name, members, history, typing}}
        self.rooms: Dict[str, dict] = {
            "general": {
                "id": "general",
                "name": "ห้องทั่วไป",
                "description": "ห้องสนทนาสำหรับทุกคน",
                "members": set(),
                "history": [],
                "typing_users": set()
            },
            "random": {
                "id": "random",
                "name": "ห้องสุ่ม",
                "description": "พูดอะไรก็ได้",
                "members": set(),
                "history": [],
                "typing_users": set()
            }
        }

    # === User Management ===

    async def register_user(self, websocket: WebSocket, username: str) -> str:
        """ลงทะเบียน user ใหม่"""
        await websocket.accept()
        user_id = str(uuid.uuid4())[:8]

        self.users[user_id] = {
            "id": user_id,
            "username": username,
            "websocket": websocket,
            "status": UserStatus.ONLINE,
            "joined_at": datetime.utcnow(),
            "rooms": set(),
            "last_seen": datetime.utcnow()
        }

        return user_id

    def get_user_by_username(self, username: str) -> Optional[dict]:
        for user in self.users.values():
            if user["username"] == username:
                return user
        return None

    async def remove_user(self, user_id: str):
        """ลบ user ออกจากระบบ"""
        if user_id not in self.users:
            return

        user = self.users[user_id]

        # ออกจากทุกห้อง
        for room_id in list(user["rooms"]):
            await self.leave_room(user_id, room_id)

        del self.users[user_id]

    def get_online_users(self) -> List[dict]:
        return [
            {
                "id": u["id"],
                "username": u["username"],
                "status": u["status"]
            }
            for u in self.users.values()
            if u["status"] != UserStatus.OFFLINE
        ]

    # === Room Management ===

    def create_room(self, room_id: str, name: str, description: str = "") -> dict:
        """สร้างห้องใหม่"""
        if room_id in self.rooms:
            return self.rooms[room_id]

        room = {
            "id": room_id,
            "name": name,
            "description": description,
            "members": set(),
            "history": [],
            "typing_users": set()
        }
        self.rooms[room_id] = room
        return room

    async def join_room(self, user_id: str, room_id: str):
        """เข้าร่วมห้อง"""
        if room_id not in self.rooms:
            self.create_room(room_id, f"ห้อง {room_id}")

        user = self.users[user_id]
        room = self.rooms[room_id]

        if user_id not in room["members"]:
            room["members"].add(user_id)
            user["rooms"].add(room_id)

            # ส่งประวัติข้อความ
            await self.send_to_user(user_id, {
                "type": "room_joined",
                "room": {
                    "id": room_id,
                    "name": room["name"],
                    "members": [
                        self.users[uid]["username"]
                        for uid in room["members"]
                        if uid in self.users
                    ]
                },
                "history": room["history"][-30:]
            })

            # แจ้งสมาชิกอื่น
            await self.broadcast_to_room(room_id, {
                "type": "user_joined_room",
                "room_id": room_id,
                "username": user["username"],
                "timestamp": datetime.utcnow().isoformat()
            }, exclude={user_id})

    async def leave_room(self, user_id: str, room_id: str):
        """ออกจากห้อง"""
        if room_id not in self.rooms:
            return

        user = self.users.get(user_id)
        room = self.rooms[room_id]

        room["members"].discard(user_id)
        room["typing_users"].discard(user_id)
        if user:
            user["rooms"].discard(room_id)

            # แจ้งสมาชิกอื่น
            await self.broadcast_to_room(room_id, {
                "type": "user_left_room",
                "room_id": room_id,
                "username": user["username"],
                "timestamp": datetime.utcnow().isoformat()
            })

    # === Messaging ===

    async def send_message(self, user_id: str, room_id: str, content: str, reply_to: str = None):
        """ส่งข้อความไปยังห้อง"""
        user = self.users.get(user_id)
        room = self.rooms.get(room_id)

        if not user or not room:
            return

        if user_id not in room["members"]:
            await self.send_to_user(user_id, {
                "type": "error",
                "message": "คุณไม่ได้อยู่ในห้องนี้"
            })
            return

        # สร้าง message
        message = {
            "id": str(uuid.uuid4()),
            "type": "message",
            "content": content,
            "username": user["username"],
            "user_id": user_id,
            "room_id": room_id,
            "timestamp": datetime.utcnow().isoformat(),
            "reply_to": reply_to
        }

        # บันทึกประวัติ
        room["history"].append(message)
        if len(room["history"]) > 200:
            room["history"] = room["history"][-200:]

        # หยุด typing indicator
        room["typing_users"].discard(user_id)
        await self.broadcast_typing_status(room_id)

        # Broadcast ไปทุกคนในห้อง
        await self.broadcast_to_room(room_id, message)

    async def send_direct_message(self, from_user_id: str, to_username: str, content: str):
        """ส่ง direct message"""
        from_user = self.users.get(from_user_id)
        to_user = self.get_user_by_username(to_username)

        if not from_user:
            return

        if not to_user:
            await self.send_to_user(from_user_id, {
                "type": "error",
                "message": f"ไม่พบผู้ใช้ {to_username}"
            })
            return

        dm_message = {
            "id": str(uuid.uuid4()),
            "type": "direct_message",
            "content": content,
            "from": from_user["username"],
            "to": to_username,
            "timestamp": datetime.utcnow().isoformat()
        }

        # ส่งให้ทั้งสองคน
        await self.send_to_user(to_user["id"], dm_message)
        await self.send_to_user(from_user_id, {**dm_message, "sent": True})

    # === Typing Indicators ===

    async def set_typing(self, user_id: str, room_id: str, is_typing: bool):
        """อัปเดต typing status"""
        if room_id not in self.rooms:
            return

        room = self.rooms[room_id]
        if is_typing:
            room["typing_users"].add(user_id)
        else:
            room["typing_users"].discard(user_id)

        await self.broadcast_typing_status(room_id)

    async def broadcast_typing_status(self, room_id: str):
        """Broadcast typing status"""
        room = self.rooms.get(room_id)
        if not room:
            return

        typing_usernames = [
            self.users[uid]["username"]
            for uid in room["typing_users"]
            if uid in self.users
        ]

        await self.broadcast_to_room(room_id, {
            "type": "typing",
            "room_id": room_id,
            "typing_users": typing_usernames
        })

    # === Helpers ===

    async def send_to_user(self, user_id: str, message: dict):
        """ส่งข้อความไปยัง user คนเดียว"""
        user = self.users.get(user_id)
        if user:
            try:
                await user["websocket"].send_text(
                    json.dumps(message, ensure_ascii=False, default=str)
                )
            except Exception:
                pass

    async def broadcast_to_room(self, room_id: str, message: dict, exclude: Set[str] = None):
        """Broadcast ไปทุกคนในห้อง"""
        room = self.rooms.get(room_id)
        if not room:
            return

        exclude = exclude or set()
        msg_str = json.dumps(message, ensure_ascii=False, default=str)
        failed = []

        for user_id in room["members"]:
            if user_id in exclude:
                continue
            user = self.users.get(user_id)
            if user:
                try:
                    await user["websocket"].send_text(msg_str)
                except:
                    failed.append(user_id)

        for user_id in failed:
            await self.remove_user(user_id)


# Singleton
chat_app = ChatApplication()


# === WebSocket Endpoint ===

@app.websocket("/chat/{username}")
async def chat_websocket(
    websocket: WebSocket,
    username: str,
    auto_join: str = Query(default="general")
):
    """Main chat WebSocket endpoint"""
    # ลงทะเบียน
    user_id = await chat_app.register_user(websocket, username)

    # ส่ง welcome
    await chat_app.send_to_user(user_id, {
        "type": "welcome",
        "user_id": user_id,
        "username": username,
        "online_users": chat_app.get_online_users(),
        "rooms": [
            {"id": rid, "name": r["name"], "description": r["description"]}
            for rid, r in chat_app.rooms.items()
        ]
    })

    # เข้าห้องอัตโนมัติ
    await chat_app.join_room(user_id, auto_join)

    try:
        while True:
            raw_data = await websocket.receive_text()

            try:
                data = json.loads(raw_data)
            except json.JSONDecodeError:
                await chat_app.send_to_user(user_id, {
                    "type": "error",
                    "message": "รูปแบบ JSON ไม่ถูกต้อง"
                })
                continue

            msg_type = data.get("type")

            # ===== Message Handlers =====

            if msg_type == "message":
                await chat_app.send_message(
                    user_id=user_id,
                    room_id=data.get("room_id", auto_join),
                    content=data.get("content", ""),
                    reply_to=data.get("reply_to")
                )

            elif msg_type == "direct_message":
                await chat_app.send_direct_message(
                    from_user_id=user_id,
                    to_username=data.get("to", ""),
                    content=data.get("content", "")
                )

            elif msg_type == "join_room":
                await chat_app.join_room(user_id, data.get("room_id", ""))

            elif msg_type == "leave_room":
                await chat_app.leave_room(user_id, data.get("room_id", ""))

            elif msg_type == "typing":
                await chat_app.set_typing(
                    user_id=user_id,
                    room_id=data.get("room_id", auto_join),
                    is_typing=data.get("is_typing", False)
                )

            elif msg_type == "create_room":
                new_room = chat_app.create_room(
                    room_id=data.get("room_id", str(uuid.uuid4())[:8]),
                    name=data.get("name", "ห้องใหม่"),
                    description=data.get("description", "")
                )
                await chat_app.send_to_user(user_id, {
                    "type": "room_created",
                    "room": new_room
                })
                await chat_app.join_room(user_id, new_room["id"])

            elif msg_type == "get_online_users":
                await chat_app.send_to_user(user_id, {
                    "type": "online_users",
                    "users": chat_app.get_online_users()
                })

            elif msg_type == "ping":
                await chat_app.send_to_user(user_id, {
                    "type": "pong",
                    "timestamp": datetime.utcnow().isoformat()
                })

    except WebSocketDisconnect:
        await chat_app.remove_user(user_id)
```

---

## 8. Error Handling ใน WebSockets

```python
# ws_error_handling.py

from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.websockets import WebSocketState
import json
import logging
import traceback
from typing import Optional
from enum import IntEnum

app = FastAPI()
logger = logging.getLogger(__name__)


# WebSocket Close Codes
class WSCloseCode(IntEnum):
    NORMAL = 1000           # ปิดปกติ
    GOING_AWAY = 1001       # Browser หรือ server กำลัง shutdown
    PROTOCOL_ERROR = 1002   # Protocol error
    UNSUPPORTED_DATA = 1003 # ประเภทข้อมูลไม่รองรับ
    POLICY_VIOLATION = 1008 # ละเมิด policy
    MESSAGE_TOO_BIG = 1009  # ข้อความใหญ่เกินไป
    INTERNAL_ERROR = 1011   # Internal server error

    # Custom codes (4000-4999 สำหรับ application use)
    UNAUTHORIZED = 4001
    FORBIDDEN = 4003
    NOT_FOUND = 4004
    RATE_LIMITED = 4029


class SafeWebSocket:
    """Wrapper สำหรับ WebSocket ที่มี error handling"""

    def __init__(self, websocket: WebSocket):
        self.ws = websocket
        self.is_connected = False

    async def accept(self):
        await self.ws.accept()
        self.is_connected = True

    async def send_json(self, data: dict) -> bool:
        """ส่ง JSON อย่างปลอดภัย - returns True ถ้าสำเร็จ"""
        if not self.is_connected:
            return False

        try:
            await self.ws.send_json(data)
            return True
        except Exception as e:
            logger.error(f"Error sending message: {e}")
            self.is_connected = False
            return False

    async def send_error(self, code: str, message: str, details: dict = None):
        """ส่ง error message"""
        await self.send_json({
            "type": "error",
            "code": code,
            "message": message,
            "details": details or {}
        })

    async def close(self, code: int = WSCloseCode.NORMAL, reason: str = ""):
        """ปิด connection อย่างปลอดภัย"""
        if self.is_connected:
            try:
                await self.ws.close(code=code, reason=reason)
            except Exception:
                pass
            finally:
                self.is_connected = False

    async def receive_json(self, timeout: float = 30.0) -> Optional[dict]:
        """รับ JSON พร้อม timeout"""
        import asyncio
        try:
            data = await asyncio.wait_for(
                self.ws.receive_json(),
                timeout=timeout
            )
            return data
        except asyncio.TimeoutError:
            logger.warning("WebSocket receive timeout")
            return None
        except Exception as e:
            logger.error(f"Error receiving: {e}")
            self.is_connected = False
            return None


@app.websocket("/ws/safe")
async def safe_websocket(websocket: WebSocket):
    """WebSocket ที่มี comprehensive error handling"""
    safe_ws = SafeWebSocket(websocket)

    try:
        await safe_ws.accept()
        logger.info(f"Client connected: {websocket.client}")

        await safe_ws.send_json({
            "type": "connected",
            "message": "เชื่อมต่อสำเร็จ"
        })

        consecutive_errors = 0
        MAX_ERRORS = 5

        while True:
            data = await safe_ws.receive_json(timeout=60.0)

            # Timeout
            if data is None:
                if not safe_ws.is_connected:
                    break
                # ส่ง ping เพื่อตรวจสอบ
                success = await safe_ws.send_json({"type": "ping"})
                if not success:
                    break
                continue

            # ตรวจสอบโครงสร้าง
            if not isinstance(data, dict):
                await safe_ws.send_error("INVALID_FORMAT", "ต้องส่ง JSON object")
                consecutive_errors += 1
                if consecutive_errors >= MAX_ERRORS:
                    await safe_ws.close(WSCloseCode.POLICY_VIOLATION, "Too many errors")
                    break
                continue

            # Reset error count เมื่อได้ข้อความที่ถูกต้อง
            consecutive_errors = 0

            msg_type = data.get("type")

            try:
                # ===== Process message =====
                if msg_type == "ping":
                    await safe_ws.send_json({"type": "pong"})

                elif msg_type == "divide":
                    # ตัวอย่าง: operation ที่อาจ fail
                    a = data.get("a", 0)
                    b = data.get("b", 0)

                    if b == 0:
                        await safe_ws.send_error("DIVISION_BY_ZERO", "ไม่สามารถหารด้วย 0 ได้")
                        continue

                    result = a / b
                    await safe_ws.send_json({
                        "type": "result",
                        "value": result
                    })

                elif msg_type == "large_payload":
                    content = data.get("content", "")
                    if len(content) > 10000:
                        await safe_ws.close(WSCloseCode.MESSAGE_TOO_BIG, "Message too large")
                        break

                    await safe_ws.send_json({
                        "type": "processed",
                        "size": len(content)
                    })

                else:
                    await safe_ws.send_error(
                        "UNKNOWN_TYPE",
                        f"ไม่รู้จัก message type: {msg_type}"
                    )

            except Exception as e:
                # จัดการ errors ภายใน message processing
                logger.error(f"Error processing message: {e}\n{traceback.format_exc()}")
                await safe_ws.send_error(
                    "PROCESSING_ERROR",
                    "เกิดข้อผิดพลาดในการประมวลผล"
                )
                consecutive_errors += 1

    except WebSocketDisconnect as e:
        logger.info(f"WebSocket disconnected: code={e.code}")

    except Exception as e:
        logger.error(f"Unexpected WebSocket error: {e}\n{traceback.format_exc()}")
        await safe_ws.close(WSCloseCode.INTERNAL_ERROR, "Internal server error")

    finally:
        logger.info("WebSocket connection closed")
        safe_ws.is_connected = False
```

---

## 9. สรุป Part 095

✅ **WebSocket พื้นฐาน** - `@app.websocket()`, accept, receive_text/json/bytes, send_text/json

✅ **Connection Lifecycle** - accept → message loop → disconnect, WebSocketDisconnect exception

✅ **ConnectionManager** - จัดการหลาย connections, เก็บ metadata, ส่งแบบ broadcast/personal

✅ **Broadcasting** - ส่งข้อความไปทุก clients, message history, exclude sender

✅ **Rooms** - Room class, RoomManager, เข้า/ออกห้อง, switch room

✅ **Authentication** - Token ใน query param, header, หรือ first message pattern

✅ **Real-time Chat** - ChatApplication สมบูรณ์พร้อม rooms, DMs, typing indicators

✅ **Error Handling** - SafeWebSocket wrapper, custom close codes, timeout handling, consecutive error tracking

---

## ➡️ ถัดไป: Part 096 - FastAPI Deployment

ใน Part ถัดไปเราจะเรียนรู้:
- Deploy FastAPI ด้วย Uvicorn และ Gunicorn
- Docker containerization
- Nginx reverse proxy
- Environment variables และ configuration management
- CI/CD pipeline

*Part 095/100+ | Python Course - Beginner to World-Class*
