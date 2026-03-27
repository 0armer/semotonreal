  from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.responses import HTMLResponse

app = FastAPI()

html = """
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>실시간 채팅방</title>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Pretendard:wght@300;400;500;600;700&display=swap');
        
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
            font-family: 'Pretendard', sans-serif;
        }

        body {
            background-color: #0f1115;
            color: #ffffff;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            overflow: hidden;
            margin: 0;
        }

        #chat-container {
            background-color: #1a1d24;
            width: 100%;
            max-width: 480px;
            height: 100vh; /* 모바일을 위해 기본 100% */
            border-radius: 0;
            display: flex;
            flex-direction: column;
            position: relative;
        }

        @media (min-width: 481px) {
            #chat-container {
                height: 90vh;
                max-height: 800px;
                border-radius: 24px;
                box-shadow: 0 20px 40px rgba(0,0,0,0.4), 0 0 0 1px rgba(255,255,255,0.05);
            }
        }

        #header {
            padding: 24px;
            text-align: center;
            font-weight: 600;
            font-size: 1.25rem;
            border-bottom: 1px solid rgba(255,255,255,0.08);
            backdrop-filter: blur(10px);
            background: linear-gradient(180deg, rgba(26,29,36,0.9) 0%, rgba(26,29,36,0.8) 100%);
            z-index: 10;
        }

        @media (min-width: 481px) {
            #header { border-radius: 24px 24px 0 0; }
        }

        .header-status {
            font-size: 0.8rem;
            color: #10b981;
            font-weight: 400;
            margin-top: 6px;
            display: flex;
            align-items: center;
            justify-content: center;
            gap: 6px;
        }

        .status-dot {
            width: 8px;
            height: 8px;
            background-color: #10b981;
            border-radius: 50%;
            box-shadow: 0 0 8px #10b981;
        }

        #messages {
            flex-grow: 1;
            padding: 24px;
            overflow-y: auto;
            display: flex;
            flex-direction: column;
            gap: 16px;
        }

        #messages::-webkit-scrollbar {
            width: 6px;
        }
        #messages::-webkit-scrollbar-track {
            background: transparent;
        }
        #messages::-webkit-scrollbar-thumb {
            background: rgba(255,255,255,0.1);
            border-radius: 10px;
        }

        .message-wrapper {
            display: flex;
            flex-direction: column;
            max-width: 85%;
            animation: fadeIn 0.3s ease-out forwards;
            opacity: 0;
            transform: translateY(10px);
        }

        .message-wrapper.self {
            align-self: flex-end;
        }

        .message-wrapper.other {
            align-self: flex-start;
        }
        
        .message-wrapper.system {
            align-self: center;
            max-width: 100%;
        }

        .sender-name {
            font-size: 0.75rem;
            color: rgba(255,255,255,0.4);
            margin-bottom: 4px;
            margin-left: 2px;
            margin-right: 2px;
        }

        .message-wrapper.self .sender-name {
            text-align: right;
        }

        .message {
            padding: 12px 16px;
            border-radius: 20px;
            line-height: 1.5;
            font-size: 0.95rem;
            word-break: break-all;
            position: relative;
        }

        .message.self {
            background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
            border-bottom-right-radius: 4px;
            color: white;
            box-shadow: 0 4px 12px rgba(37, 99, 235, 0.2);
        }

        .message.other {
            background-color: #2a2d35;
            border-bottom-left-radius: 4px;
            color: #e2e8f0;
            border: 1px solid rgba(255,255,255,0.05);
        }
        
        .message.system {
            background-color: rgba(255, 255, 255, 0.05);
            border-radius: 12px;
            color: rgba(255,255,255,0.6);
            font-size: 0.85rem;
            text-align: center;
            padding: 6px 16px;
        }

        @keyframes fadeIn {
            to {
                opacity: 1;
                transform: translateY(0);
            }
        }

        #input-container {
            padding: 16px 24px 24px;
            background: transparent;
            display: flex;
            gap: 12px;
            position: relative;
        }

        .input-wrapper {
            flex-grow: 1;
            position: relative;
            background-color: #2a2d35;
            border-radius: 24px;
            border: 1px solid rgba(255,255,255,0.08);
            transition: all 0.2s ease;
        }

        .input-wrapper:focus-within {
            border-color: rgba(59, 130, 246, 0.5);
            box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.1);
        }

        #messageText {
            width: 100%;
            padding: 14px 20px;
            background: transparent;
            border: none;
            color: #ffffff;
            font-size: 0.95rem;
            outline: none;
        }

        #messageText::placeholder {
            color: rgba(255,255,255,0.3);
        }

        button {
            width: 48px;
            height: 48px;
            border-radius: 50%;
            background: linear-gradient(135deg, #3b82f6 0%, #2563eb 100%);
            color: white;
            border: none;
            cursor: pointer;
            display: flex;
            justify-content: center;
            align-items: center;
            transition: transform 0.2s ease, box-shadow 0.2s ease;
            box-shadow: 0 4px 12px rgba(37, 99, 235, 0.2);
            flex-shrink: 0;
        }

        button:hover {
            transform: translateY(-2px);
            box-shadow: 0 6px 16px rgba(37, 99, 235, 0.3);
        }
        
        button:active {
            transform: translateY(0);
        }

        button svg {
            width: 20px;
            height: 20px;
            margin-left: 2px;
            margin-top: 1px;
            fill: none;
            stroke: currentColor;
            stroke-width: 2.5;
            stroke-linecap: round;
            stroke-linejoin: round;
        }

        /* Glassmorphism blobs */
        .blob-1 { position: absolute; top: 10%; left: -100px; width: 300px; height: 300px; border-radius: 50%; background: radial-gradient(circle, rgba(59,130,246,0.15) 0%, rgba(0,0,0,0) 70%); z-index: -1; filter: blur(40px); }
        .blob-2 { position: absolute; bottom: 10%; right: -50px; width: 250px; height: 250px; border-radius: 50%; background: radial-gradient(circle, rgba(16,185,129,0.1) 0%, rgba(0,0,0,0) 70%); z-index: -1; filter: blur(40px); }
    </style>
</head>
<body>
    <div class="blob-1"></div>
    <div class="blob-2"></div>
    <div id="chat-container">
        <div id="header">
            실시간 랜덤
            <div class="header-status">
                <div class="status-dot"></div>
                온라인
            </div>
        </div>
        <div id="messages"></div>
        <form action="" onsubmit="sendMessage(event)" id="input-container">
            <div class="input-wrapper">
                <input type="text" id="messageText" autocomplete="off" placeholder="메시지를 입력하세요..." required/>
            </div>
            <button type="submit">
                <svg viewBox="0 0 24 24">
                    <line x1="22" y1="2" x2="11" y2="13"></line>
                    <polygon points="22 2 15 22 11 13 2 9 22 2"></polygon>
                </svg>
            </button>
        </form>
    </div>
    <script>
        var client_id = Math.floor(Math.random() * 10000);
        var protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
        var ws = new WebSocket(`${protocol}//${location.host}/ws/${client_id}`);
        
        ws.onopen = function() {
            var messages = document.getElementById('messages');
            var wrapper = document.createElement('div');
            wrapper.classList.add('message-wrapper', 'system');
            var msgDiv = document.createElement('div');
            msgDiv.classList.add('message', 'system');
            msgDiv.textContent = `채팅 서버에 연결되었습니다. (내 ID: ${client_id})`;
            wrapper.appendChild(msgDiv);
            messages.appendChild(wrapper);
        };

        ws.onmessage = function(event) {
            var messages = document.getElementById('messages');
            var messageData = JSON.parse(event.data);
            
            var wrapper = document.createElement('div');
            wrapper.classList.add('message-wrapper');
            
            if (messageData.client_id === "System") {
                wrapper.classList.add('system');
                var msgDiv = document.createElement('div');
                msgDiv.classList.add('message', 'system');
                msgDiv.textContent = messageData.message;
                wrapper.appendChild(msgDiv);
            } else {
                var isSelf = (messageData.client_id == client_id);
                wrapper.classList.add(isSelf ? 'self' : 'other');
                
                var nameDiv = document.createElement('div');
                nameDiv.classList.add('sender-name');
                nameDiv.textContent = isSelf ? '나' : `User ${messageData.client_id}`;
                wrapper.appendChild(nameDiv);
                
                var msgDiv = document.createElement('div');
                msgDiv.classList.add('message', isSelf ? 'self' : 'other');
                msgDiv.textContent = messageData.message;
                wrapper.appendChild(msgDiv);
            }
            
            messages.appendChild(wrapper);
            
            // Smooth scroll to bottom
            messages.scrollTo({
                top: messages.scrollHeight,
                behavior: 'smooth'
            });
        };
        
        function sendMessage(event) {
            event.preventDefault();
            var input = document.getElementById("messageText");
            var message = input.value.trim();
            if(message !== "") {
                ws.send(message);
                input.value = '';
                input.focus();
            }
        }
    </script>
</body>
</html>
"""

class ConnectionManager:
    def __init__(self):
        self.active_connections: list[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.active_connections.append(websocket)

    def disconnect(self, websocket: WebSocket):
        self.active_connections.remove(websocket)

    async def broadcast(self, message: dict):
        for connection in self.active_connections:
            await connection.send_json(message)

manager = ConnectionManager()

@app.get("/")
async def get():
    return HTMLResponse(html)

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await manager.connect(websocket)
    try:
        # 환영 메시지는 모든 사람에게 보냅니다.
        await manager.broadcast({"client_id": "System", "message": f"User {client_id} 님이 입장하셨습니다."})
        while True:
            data = await websocket.receive_text()
            await manager.broadcast({"client_id": client_id, "message": data})
    except WebSocketDisconnect:
        manager.disconnect(websocket)
        await manager.broadcast({"client_id": "System", "message": f"User {client_id} 님이 퇴장하셨습니다."})
