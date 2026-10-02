# Part 095: FastAPI WebSockets
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้
- สร้าง WebSocket endpoint ใน FastAPI
- จัดการ WebSocket connections
- สร้าง Connection Manager สำหรับ broadcasting
- สร้าง real-time chat application
- จัดการ authentication ใน WebSocket

---

## 1. WebSocket คืออะไร?

WebSocket เป็น protocol ที่ช่วยให้ browser และ server สื่อสารแบบ two-way (bi-directional) ได้แบบ real-time

```
HTTP (request-response):
Client → Request → Server → Response → Client (จบ)

WebSocket (persistent connection):
Client ←→ Server (คุยไปมาได้ตลอด)
```

ใช้สำหรับ:
- Chat applications
- Real-time notifications
- Live updates (stock prices, sports scores)
- Collaborative editing
- Online games

---

## 2. WebSocket พื้นฐาน

```python
# websocket_basic.py

from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()


@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    """
    WebSocket endpoint พื้นฐาน
    URL: ws://localhost:8000/ws
    """
    
    # รับ connection จาก client
    await websocket.accept()
    
    try:
        while True:
            # รอรับ message จาก client
            data = await websocket.receive_text()
            
            # ส่ง message กลับ
            await websocket.send_text(f"คุณส่งมาว่า: {data}")
    
    except WebSocketDisconnect:
        # Client ตัด connection
        print("Client disconnected")


@app.websocket("/ws/json")
async def websocket_json(websocket: WebSocket):
    """WebSocket ที่รับส่ง JSON"""
    await websocket.accept()
    
    try:
        while True:
            # รับ JSON
            data = await websocket.receive_json()
            
            # ประมวลผล
            response = {
                "status": "received",
                "echo": data,
                "message_count": 1
            }
            
            # ส่ง JSON กลับ
            await websocket.send_json(response)
    
    except WebSocketDisconnect:
        print("Client disconnected")


@app.websocket("/ws/bytes")
async def websocket_bytes(websocket: WebSocket):
    """WebSocket ที่รับส่ง bytes (สำหรับไฟล์)"""
    await websocket.accept()
    
    try:
        while True:
            data = await websocket.receive_bytes()
            await websocket.send_bytes(data)  # echo
    
    except WebSocketDisconnect:
        pass
```

### HTML Client สำหรับทดสอบ
```html
<!-- test_client.html -->
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Test</title>
</head>
<body>
    <h1>WebSocket Test</h1>
    <input id="message" type="text" placeholder="พิมพ์ข้อความ">
    <button onclick="sendMessage()">ส่ง</button>
    <div id="output"></div>
    
    <script>
        const ws = new WebSocket("ws://localhost:8000/ws");
        
        ws.onopen = () => {
            console.log("Connected!");
            appendMessage("✅ เชื่อมต่อสำเร็จ");
        };
        
        ws.onmessage = (event) => {
            appendMessage("Server: " + event.data);
        };
        
        ws.onclose = () => {
            appendMessage("❌ ตัดการเชื่อมต่อ");
        };
        
        function sendMessage() {
            const input = document.getElementById("message");
            ws.send(input.value);
            appendMessage("You: " + input.value);
            input.value = "";
        }
        
        function appendMessage(msg) {
            const div = document.getElementById("output");
            div.innerHTML += `<p>${msg}</p>`;
        }
    </script>
</body>
</html>
```

---

## 3. Connection Manager

```python
# connection_manager.py

from fastapi import WebSocket
from typing import List, Dict
import json


class ConnectionManager:
    """จัดการ WebSocket connections ทั้งหมด"""
    
    def __init__(self):
        # รายการ connections ที่ active
        self.active_connections: List[WebSocket] = []
        # Map user_id → websocket
        self.user_connections: Dict[int, WebSocket] = {}
        # Map room → list of websockets
        self.room_connections: Dict[str, List[WebSocket]] = {}
    
    async def connect(self, websocket: WebSocket, user_id: int = None, room: str = None):
        """รับ connection ใหม่"""
        await websocket.accept()
        self.active_connections.append(websocket)
        
        if user_id:
            self.user_connections[user_id] = websocket
        
        if room:
            if room not in self.room_connections:
                self.room_connections[room] = []
            self.room_connections[room].append(websocket)
    
    def disconnect(self, websocket: WebSocket, user_id: int = None, room: str = None):
        """ลบ connection"""
        if websocket in self.active_connections:
            self.active_connections.remove(websocket)
        
        if user_id and user_id in self.user_connections:
            del self.user_connections[user_id]
        
        if room and room in self.room_connections:
            if websocket in self.room_connections[room]:
                self.room_connections[room].remove(websocket)
    
    async def send_personal(self, message: str, websocket: WebSocket):
        """ส่ง message ไปยัง connection เดียว"""
        await websocket.send_text(message)
    
    async def send_to_user(self, message: str, user_id: int):
        """ส่ง message ไปยัง user เฉพาะ"""
        ws = self.user_connections.get(user_id)
        if ws:
            await ws.send_text(message)
    
    async def broadcast(self, message: str):
        """ส่ง message ไปยังทุก connection"""
        disconnected = []
        for connection in self.active_connections:
            try:
                await connection.send_text(message)
            except Exception:
                disconnected.append(connection)
        
        # ลบ connections ที่ตัดไปแล้ว
        for conn in disconnected:
            self.active_connections.remove(conn)
    
    async def broadcast_to_room(self, message: str, room: str):
        """ส่ง message ไปยัง room เฉพาะ"""
        if room not in self.room_connections:
            return
        
        disconnected = []
        for connection in self.room_connections[room]:
            try:
                await connection.send_text(message)
            except Exception:
                disconnected.append(connection)
        
        for conn in disconnected:
            self.room_connections[room].remove(conn)
    
    async def broadcast_json(self, data: dict):
        """ส่ง JSON ไปยังทุก connection"""
        message = json.dumps(data, ensure_ascii=False)
        await self.broadcast(message)
    
    @property
    def connection_count(self) -> int:
        return len(self.active_connections)
    
    @property
    def online_users(self) -> List[int]:
        return list(self.user_connections.keys())


# Global instance
manager = ConnectionManager()
```

---

## 4. Chat Application

```python
# chat_app.py

from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Query, HTTPException
from fastapi.responses import HTMLResponse
from typing import Optional
from datetime import datetime, timezone
import json

from connection_manager import manager

app = FastAPI()


# ---- REST Endpoints ----

@app.get("/chat", response_class=HTMLResponse)
async def chat_page():
    """หน้า Chat"""
    return """
    <!DOCTYPE html>
    <html>
    <head>
        <title>Chat App</title>
        <style>
            body { font-family: Arial; max-width: 800px; margin: 0 auto; padding: 20px; }
            #messages { height: 400px; overflow-y: scroll; border: 1px solid #ccc; padding: 10px; margin-bottom: 10px; }
            #input-area { display: flex; gap: 10px; }
            input { flex: 1; padding: 8px; }
            button { padding: 8px 16px; background: #007bff; color: white; border: none; cursor: pointer; border-radius: 4px; }
            .my-message { text-align: right; color: #007bff; }
            .other-message { text-align: left; color: #333; }
            .system-message { text-align: center; color: #999; font-style: italic; }
        </style>
    </head>
    <body>
        <h1>💬 Chat Room</h1>
        <div id="messages"></div>
        <div id="input-area">
            <input id="username" placeholder="ชื่อของคุณ" value="User1">
            <input id="message" placeholder="พิมพ์ข้อความ..." onkeypress="handleEnter(event)">
            <button onclick="sendMessage()">ส่ง</button>
        </div>
        
        <script>
            let ws;
            let username = "";
            
            function connect() {
                username = document.getElementById("username").value;
                const room = "general";
                ws = new WebSocket(`ws://localhost:8000/ws/chat/${room}?username=${username}`);
                
                ws.onmessage = (event) => {
                    const data = JSON.parse(event.data);
                    displayMessage(data);
                };
                
                ws.onclose = () => {
                    displayMessage({type: "system", content: "Disconnected"});
                };
            }
            
            function sendMessage() {
                const input = document.getElementById("message");
                if (input.value && ws) {
                    ws.send(JSON.stringify({
                        type: "message",
                        content: input.value
                    }));
                    input.value = "";
                }
            }
            
            function handleEnter(event) {
                if (event.key === "Enter") sendMessage();
            }
            
            function displayMessage(data) {
                const div = document.getElementById("messages");
                const p = document.createElement("p");
                
                if (data.type === "system") {
                    p.className = "system-message";
                    p.textContent = data.content;
                } else if (data.username === username) {
                    p.className = "my-message";
                    p.textContent = `${data.content} :คุณ`;
                } else {
                    p.className = "other-message";
                    p.textContent = `${data.username}: ${data.content}`;
                }
                
                div.appendChild(p);
                div.scrollTop = div.scrollHeight;
            }
            
            // Auto-connect
            window.onload = connect;
        </script>
    </body>
    </html>
    """


# ---- WebSocket Endpoint ----

@app.websocket("/ws/chat/{room}")
async def chat_websocket(
    websocket: WebSocket,
    room: str,
    username: str = Query(...)  # รับ username จาก query parameter
):
    """WebSocket endpoint สำหรับ chat"""
    
    # เชื่อมต่อ
    await manager.connect(websocket, room=room)
    
    # แจ้งว่ามีคนเข้ามา
    join_message = json.dumps({
        "type": "system",
        "content": f"⬆️ {username} เข้ามาใน {room}",
        "timestamp": datetime.now(timezone.utc).isoformat()
    }, ensure_ascii=False)
    await manager.broadcast_to_room(join_message, room)
    
    try:
        while True:
            # รับ message
            data = await websocket.receive_text()
            
            try:
                message_data = json.loads(data)
            except json.JSONDecodeError:
                message_data = {"type": "message", "content": data}
            
            # สร้าง response
            response = json.dumps({
                "type": "message",
                "username": username,
                "content": message_data.get("content", ""),
                "room": room,
                "timestamp": datetime.now(timezone.utc).isoformat()
            }, ensure_ascii=False)
            
            # ส่งไปยังทุกคนใน room
            await manager.broadcast_to_room(response, room)
    
    except WebSocketDisconnect:
        manager.disconnect(websocket, room=room)
        
        # แจ้งว่ามีคนออกไป
        leave_message = json.dumps({
            "type": "system",
            "content": f"⬇️ {username} ออกจาก {room}",
            "timestamp": datetime.now(timezone.utc).isoformat()
        }, ensure_ascii=False)
        await manager.broadcast_to_room(leave_message, room)


# ---- Status Endpoint ----

@app.get("/ws/status")
async def websocket_status():
    """ดูสถานะ WebSocket connections"""
    return {
        "total_connections": manager.connection_count,
        "online_users": manager.online_users,
        "rooms": {
            room: len(conns)
            for room, conns in manager.room_connections.items()
        }
    }
```

---

## 5. Real-time Notifications

```python
# notifications.py

from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from connection_manager import manager
import json
import asyncio
from datetime import datetime, timezone

app = FastAPI()


@app.websocket("/ws/notifications/{user_id}")
async def notification_websocket(
    websocket: WebSocket,
    user_id: int
):
    """WebSocket สำหรับ real-time notifications"""
    await manager.connect(websocket, user_id=user_id)
    
    # ส่ง notification ต้อนรับ
    await websocket.send_json({
        "type": "connected",
        "message": "เชื่อมต่อสำเร็จ",
        "timestamp": datetime.now(timezone.utc).isoformat()
    })
    
    try:
        while True:
            # รอรับ acknowledgment จาก client
            # หรือ keepalive messages
            data = await websocket.receive_text()
            
            if data == "ping":
                await websocket.send_text("pong")
    
    except WebSocketDisconnect:
        manager.disconnect(websocket, user_id=user_id)
        print(f"User {user_id} disconnected")


# API endpoint สำหรับส่ง notification ผ่าน REST
@app.post("/notifications/send/{user_id}")
async def send_notification(user_id: int, message: str, notification_type: str = "info"):
    """ส่ง notification ไปยัง user ที่ระบุผ่าน WebSocket"""
    
    notification = json.dumps({
        "type": notification_type,
        "message": message,
        "timestamp": datetime.now(timezone.utc).isoformat()
    }, ensure_ascii=False)
    
    await manager.send_to_user(notification, user_id)
    
    return {"status": "sent", "user_id": user_id}


@app.post("/notifications/broadcast")
async def broadcast_notification(message: str):
    """ส่ง notification ไปยังทุกคน"""
    notification = json.dumps({
        "type": "broadcast",
        "message": message,
        "timestamp": datetime.now(timezone.utc).isoformat()
    }, ensure_ascii=False)
    
    await manager.broadcast(notification)
    return {"status": "broadcast sent", "connections": manager.connection_count}
```

---

## 6. WebSocket Authentication

```python
# websocket_auth.py

from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Query, status
import jwt

SECRET_KEY = "your-secret-key"
ALGORITHM = "HS256"

app = FastAPI()


async def get_websocket_user(token: str):
    """ตรวจสอบ JWT token สำหรับ WebSocket"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id = int(payload.get("sub"))
        username = payload.get("username")
        return {"id": user_id, "username": username}
    except Exception:
        return None


@app.websocket("/ws/authenticated")
async def authenticated_websocket(
    websocket: WebSocket,
    token: str = Query(...)  # รับ token จาก query parameter
):
    """WebSocket ที่ต้องการ authentication"""
    
    # ตรวจสอบ token ก่อน accept
    user = await get_websocket_user(token)
    
    if not user:
        # ปฏิเสธ connection
        await websocket.close(code=status.WS_1008_POLICY_VIOLATION)
        return
    
    await websocket.accept()
    
    # ส่งข้อมูล user กลับ
    await websocket.send_json({
        "type": "authenticated",
        "user": user,
        "message": f"ยินดีต้อนรับ {user['username']}"
    })
    
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_json({
                "type": "echo",
                "from": user["username"],
                "message": data
            })
    
    except WebSocketDisconnect:
        print(f"User {user['username']} disconnected")
```

---

## 7. Complete Chat App

```python
# main.py — Complete Chat Application

from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Query
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from connection_manager import ConnectionManager
from typing import Optional
from datetime import datetime, timezone
import json

app = FastAPI(title="Real-time Chat")
manager = ConnectionManager()


@app.websocket("/ws/chat/{room_id}")
async def chat_endpoint(
    websocket: WebSocket,
    room_id: str,
    username: str = Query(..., min_length=1, max_length=50)
):
    """Chat WebSocket endpoint"""
    await manager.connect(websocket, room=room_id)
    
    # แจ้งคนอื่น
    await manager.broadcast_to_room(
        json.dumps({
            "type": "join",
            "username": username,
            "room": room_id,
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "online_count": len(manager.room_connections.get(room_id, []))
        }, ensure_ascii=False),
        room_id
    )
    
    try:
        while True:
            raw_data = await websocket.receive_text()
            
            try:
                data = json.loads(raw_data)
                msg_type = data.get("type", "message")
                content = data.get("content", "")
            except json.JSONDecodeError:
                msg_type = "message"
                content = raw_data
            
            response_data = {
                "type": msg_type,
                "username": username,
                "room": room_id,
                "content": content,
                "timestamp": datetime.now(timezone.utc).isoformat()
            }
            
            if msg_type == "message":
                await manager.broadcast_to_room(
                    json.dumps(response_data, ensure_ascii=False),
                    room_id
                )
            elif msg_type == "typing":
                # ส่ง typing indicator ไปคนอื่น (ไม่ส่งกลับ sender)
                await manager.broadcast_to_room(
                    json.dumps(response_data, ensure_ascii=False),
                    room_id
                )
    
    except WebSocketDisconnect:
        manager.disconnect(websocket, room=room_id)
        
        await manager.broadcast_to_room(
            json.dumps({
                "type": "leave",
                "username": username,
                "room": room_id,
                "timestamp": datetime.now(timezone.utc).isoformat(),
                "online_count": len(manager.room_connections.get(room_id, []))
            }, ensure_ascii=False),
            room_id
        )


@app.get("/rooms/{room_id}/status")
async def room_status(room_id: str):
    """สถานะของ room"""
    connections = manager.room_connections.get(room_id, [])
    return {
        "room": room_id,
        "online_count": len(connections)
    }


if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

---

## 8. สรุป Part 095

✅ **WebSocket** สร้างด้วย `@app.websocket()` decorator  
✅ **websocket.accept()** รับ connection จาก client  
✅ **receive_text() / send_text()** รับส่ง text messages  
✅ **receive_json() / send_json()** รับส่ง JSON data  
✅ **ConnectionManager** จัดการ connections หลายตัว  
✅ **Broadcasting** ส่ง message ไปยัง connections ทั้งหมด  
✅ **Room-based messaging** แยก groups ด้วย rooms  
✅ **WebSocket authentication** ตรวจสอบ token ก่อน accept  
✅ **WebSocketDisconnect** จัดการเมื่อ client ตัด connection  

---

## ✅ จบ FastAPI Section!

คุณได้เรียนรู้ FastAPI ครบทั้ง:
- **Part 086**: FastAPI Basics
- **Part 087**: Path & Query Parameters  
- **Part 088**: Request Body & File Upload
- **Part 089**: Dependencies
- **Part 090**: Authentication (JWT)
- **Part 091**: Database Integration (async SQLAlchemy)
- **Part 092**: Background Tasks (BackgroundTasks + Celery)
- **Part 093**: Middleware & CORS
- **Part 094**: Testing
- **Part 095**: WebSockets

*Part 095/100+ | Python Course - Beginner to World-Class*
