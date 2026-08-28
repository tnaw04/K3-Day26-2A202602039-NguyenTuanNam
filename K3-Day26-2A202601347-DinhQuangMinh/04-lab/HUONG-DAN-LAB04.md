# 🏆 Hướng Dẫn Hoàn Thiện Lab 04: Weather Agent với Remote MCP Server

## Mục lục
1. [Tổng quan bài lab](#1-tổng-quan-bài-lab)
2. [Kiến trúc hệ thống](#2-kiến-trúc-hệ-thống)
3. [Các bước chuẩn bị](#3-các-bước-chuẩn-bị)
4. [Bước 1: Cài đặt MCP Server](#4-bước-1-cài-đặt-mcp-server)
5. [Bước 2: Cài đặt ADK Agent (Client)](#5-bước-2-cài-đặt-adk-agent-client)
6. [Bước 3: Chạy và kiểm tra](#6-bước-3-chạy-và-kiểm-tra)
7. [Bước 4: Triển khai Cloud Run (tùy chọn)](#7-bước-4-triển-khai-cloud-run-tùy-chọn)
8. [Giải thích code chi tiết](#8-giải-thích-code-chi-tiết)
9. [Các lỗi thường gặp và cách khắc phục](#9-các-lỗi-thường-gặp-và-cách-khắc-phục)
10. [Bài tập mở rộng](#10-bài-tập-mở-rộng)

---

## 1. Tổng quan bài lab

### Mục tiêu
- Xây dựng **Weather Agent** sử dụng **Google ADK** (Agent Development Kit)
- Kết nối Agent với **Remote MCP Server** qua **Streamable HTTP**
- MCP Server gọi API thời tiết từ **WeatherAPI.com**
- Trải nghiệm kiến trúc **Client-Server** hoàn chỉnh với MCP

### Kết quả mong đợi
```
User hỏi: "Thời tiết Hà Nội như thế nào?"
    ↓
ADK Agent (Gemini) nhận câu hỏi
    ↓
Quyết định gọi tool get_current_weather
    ↓
MCP Client gửi request đến MCP Server (HTTP)
    ↓
MCP Server gọi WeatherAPI.com
    ↓
Trả về kết quả cho Agent
    ↓
Agent tổng hợp và trả lời User
```

### Các công nghệ sử dụng
| Thành phần | Công nghệ | Vai trò |
|------------|-----------|---------|
| MCP Server | FastMCP + httpx | Cung cấp tools thời tiết |
| MCP Client | Google ADK | Điều phối Agent + Tools |
| LLM | Gemini 2.5 Flash | Xử lý ngôn ngữ tự nhiên |
| Weather API | WeatherAPI.com | Nguồn dữ liệu thời tiết |
| Transport | Streamable HTTP | Giao tiếp Client-Server |

---

## 2. Kiến trúc hệ thống

### Sơ đồ kiến trúc

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           WEATHER AGENT SYSTEM                              │
│                                                                             │
│   ┌───────────────────────┐         ┌───────────────────────┐              │
│   │    ADK WEB UI         │         │    ADK WEB UI         │              │
│   │   (localhost:8000)    │         │   (Browser)          │              │
│   └───────────┬───────────┘         └───────────┬───────────┘              │
│               │                                  │                          │
│               │     Session Management          │                          │
│               └──────────────┬──────────────────┘                          │
│                              │                                               │
│                              ▼                                               │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │                        ADK AGENT                                   │     │
│   │  ┌─────────────────────────────────────────────────────────────┐  │     │
│   │  │                    Gemini 2.5 Flash                          │  │     │
│   │  │         (Function Calling Decision Making)                  │  │     │
│   │  └─────────────────────────────────────────────────────────────┘  │     │
│   │                              │                                      │     │
│   │                              ▼                                      │     │
│   │  ┌─────────────────────────────────────────────────────────────┐  │     │
│   │  │                    McpToolset                                │  │     │
│   │  │         (list_tools → call_tool orchestration)              │  │     │
│   │  └─────────────────────────────────────────────────────────────┘  │     │
│   └──────────────────────────────┬──────────────────────────────────────┘     │
│                                  │                                           │
│                                  │ Streamable HTTP                          │
│                                  │ (localhost:8085/mcp)                     │
│                                  ▼                                           │
│   ┌───────────────────────────────────────────────────────────────────┐     │
│   │                      MCP SERVER                                   │     │
│   │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │     │
│   │  │ get_current_    │  │ get_forecast   │  │ health_check   │    │     │
│   │  │   weather()     │  │   ()           │  │   ()           │    │     │
│   │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘    │     │
│   │           │                     │                     │              │     │
│   │           └─────────────────────┼─────────────────────┘              │     │
│   │                                 │                                    │     │
│   │                                 ▼                                    │     │
│   │  ┌─────────────────────────────────────────────────────────────┐  │     │
│   │  │                   httpx.AsyncClient                         │  │     │
│   │  │              (Gọi WeatherAPI.com API)                       │  │     │
│   │  └─────────────────────────────────────────────────────────────┘  │     │
│   └──────────────────────────────┬──────────────────────────────────────┘     │
│                                  │                                           │
│                                  ▼                                           │
│                         ┌─────────────────┐                                  │
│                         │ WeatherAPI.com  │                                  │
│                         │   (REST API)    │                                  │
│                         └─────────────────┘                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Luồng xử lý chi tiết

```
1. User mở http://localhost:8000 → Chọn "weather_agent"
                                    │
2. User nhập: "Thời tiết Hà Nội?" ──▶ ADK nhận message
                                    │
3. ADK gọi list_tools() ──────────▶ MCP Server
   GET /mcp ──────────────────────▶ Trả về tools schema
                                    │
4. ADK gửi tools + message ──────▶ Gemini
   cho Gemini quyết định            │
                                    │
5. Gemini quyết định: ────────────▶ Gọi get_current_weather
   "Hãy gọi tool này"              │ với city="Hanoi"
                                    │
6. ADK gọi call_tool() ──────────▶ MCP Server
   POST /mcp {tool, params}        │
                                    │
7. MCP Server thực thi: ──────────▶ Gọi WeatherAPI.com
   get_current_weather("Hanoi")     │ GET /current.json?q=Hanoi
                                    │
8. WeatherAPI trả dữ liệu ────────▶ MCP Server
   {temp_c, condition, ...}         │
                                    │
9. MCP Server format response ────▶ Trả về cho ADK
   "Nhiệt độ: 25°C, ..."            │
                                    │
10. ADK gửi kết quả ──────────────▶ Gemini
    để tổng hợp câu trả lời         │
                                    │
11. Gemini trả lời User ──────────▶ ADK hiển thị
    "Hà Nội hôm nay: 25°C, ..."     │
```

---

## 3. Các bước chuẩn bị

### Yêu cầu hệ thống

| Thành phần | Phiên bản tối thiểu |
|------------|---------------------|
| Python (MCP Server) | 3.10+ |
| Python (ADK Client) | 3.12+ |
| uv (package manager) | Latest |
| Git | Any |
| Web Browser | Chrome/Edge/Firefox |

### Chuẩn bị API Keys

#### 3.1. Lấy Google Gemini API Key

1. Truy cập: https://aistudio.google.com/apikey
2. Đăng nhập tài khoản Google
3. Click **"Create API Key"**
4. Copy API key (bắt đầu bằng `AIza...`)
5. **Lưu lại an toàn** - không chia sẻ công khai

#### 3.2. Lấy WeatherAPI.com API Key

1. Truy cập: https://www.weatherapi.com
2. Đăng ký tài khoản miễn phí
3. Sau khi đăng nhập, vào **Dashboard**
4. Copy API key (bắt đầu bằng `xxxx...`)
5. **Lưu lại an toàn** - key miễn phí có giới hạn

> **Lưu ý:** API key miễn phí của WeatherAPI cho phép 1 triệu calls/tháng và dự báo tối đa 3 ngày.

---

## 4. Bước 1: Cài đặt MCP Server

### 4.1. Cấu trúc thư mục

```
04-lab/
└── mcp-server/
    ├── weather.py          # Main server code
    ├── pyproject.toml      # Dependencies
    ├── .venv/              # Virtual environment
    └── .env                 # API keys (tạo mới)
```

### 4.2. Tạo thư mục và virtual environment

```bash
# Di chuyển vào thư mục MCP Server
cd 04-lab/mcp-server

# Tạo virtual environment với uv
uv venv .venv

# Kích hoạt virtual environment (Windows)
.venv\Scripts\activate

# Hoặc (Linux/Mac)
# source .venv/bin/activate
```

### 4.3. Cài đặt dependencies

```bash
# Cài đặt tất cả dependencies từ pyproject.toml
uv sync

# Hoặc cài đặt thủ công
uv pip install fastmcp httpx mcp uvicorn starlette
```

### 4.4. Tạo file .env cho MCP Server

```bash
# Tạo file .env (Windows PowerShell)
$env:WEATHERAPI_KEY = "YOUR_WEATHERAPI_KEY_HERE"
$env:PORT = "8085"

# Hoặc tạo file .env trực tiếp
# Windows Command
# echo WEATHERAPI_KEY=your_key > .env
# echo PORT=8085 >> .env
```

**Nội dung file `.env` (tạo bằng Notepad hoặc VS Code):**
```env
# WeatherAPI.com API Key
WEATHERAPI_KEY=your_weatherapi_key_here

# Server port (mặc định 8085)
PORT=8085
```

### 4.5. Chạy MCP Server

```bash
# Chạy server với Streamable HTTP transport
uv run python weather.py
```

**Kết quả mong đợi:**
```
✅ MCP server initialized with Streamable HTTP transport
🔧 Available tools: get_current_weather, get_forecast, health_check
🚀 Starting MCP server on http://0.0.0.0:8085/mcp
```

### 4.6. Kiểm tra MCP Server

Mở terminal mới và test:

```bash
# Test health check endpoint
curl http://localhost:8085/mcp

# Hoặc kiểm tra server có chạy không
# (sẽ trả về lỗi JSON-RPC vì đây là endpoint MCP)
```

---

## 5. Bước 2: Cài đặt ADK Agent (Client)

### 5.1. Cấu trúc thư mục

```
04-lab/mcp-client/
├── weather_agent/
│   ├── __init__.py          # Package init
│   └── agent.py             # Agent definition
├── .env                     # Gemini API key
├── .venv/                   # Virtual environment
└── pyproject.toml           # Dependencies
```

### 5.2. Tạo virtual environment và cài đặt

```bash
# Di chuyển vào thư mục MCP Client
cd 04-lab/mcp-client

# Tạo virtual environment với uv
uv venv .venv

# Kích hoạt virtual environment
.venv\Scripts\activate

# Cài đặt dependencies
uv sync
```

### 5.3. Tạo file .env cho ADK Client

**Tạo file `.env` với nội dung:**
```env
# Google Gemini API Key
GEMINI_API_KEY=your_gemini_api_key_here

# Model name (gemini-2.5-flash khuyến nghị)
GEMINI_MODEL=gemini-2.5-flash

# Alias cho GOOGLE_API_KEY (ADK cũng dùng)
GOOGLE_API_KEY=your_gemini_api_key_here
```

### 5.4. Tạo Agent Package

Đảm bảo cấu trúc thư mục `weather_agent/` tồn tại:

```bash
# Tạo thư mục nếu chưa có
mkdir -p weather_agent
```

**File `weather_agent/__init__.py`:**
```python
"""Weather Agent package - ADK agent with remote MCP tools"""
from .agent import root_agent

__all__ = ["root_agent"]
```

**File `weather_agent/agent.py`:**
```python
"""
Weather Agent - Connects to Remote MCP Server on Cloud Run
Successfully connects to custom MCP HTTP endpoints!
"""
from google.adk import Agent
from google.adk.tools.mcp_tool.mcp_toolset import McpToolset, StreamableHTTPConnectionParams
import logging
import os
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Set up logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# MCP Server URL - đổi thành Cloud Run URL nếu deploy
MCP_SERVER_URL = os.getenv("MCP_SERVER_URL", "http://localhost:8085/mcp")

logger.info(f"🌐 Initializing weather agent with remote MCP server")
logger.info(f"📡 MCP Server: {MCP_SERVER_URL}")

try:
    # Create connection parameters for the remote MCP server
    connection_params = StreamableHTTPConnectionParams(
        url=MCP_SERVER_URL,
        timeout=30.0,
    )
    
    # Create the MCP toolset - this will connect to the remote server
    logger.info("🔌 Connecting to MCP server...")
    weather_tools = McpToolset(
        connection_params=connection_params,
    )
    logger.info("✅ MCP toolset created successfully")
    
    # Create the agent with remote MCP tools
    root_agent = Agent(
        name="weather_agent",
        model=os.getenv("GEMINI_MODEL", "gemini-2.5-flash"),
        tools=[weather_tools],
        description="Weather assistant that can provide current weather and forecasts for cities worldwide",
        instruction="""Bạn là một trợ lý thời tiết thân thiện. Khi người dùng hỏi về thời tiết:
1. Xác định thành phố được hỏi
2. Sử dụng tool get_current_weather cho thời tiết hiện tại
3. Sử dụng tool get_forecast cho dự báo (1-3 ngày)
4. Trả lời bằng tiếng Việt, súc tích và hữu ích

Luôn trả lời bằng tiếng Việt."""
    )
    logger.info("✅ Weather agent initialized with remote MCP tools:")
    logger.info("   - get_current_weather(city)")
    logger.info("   - get_forecast(city, days)")
    logger.info("   - health_check()")
    logger.info("🎉 Remote MCP connection successful!")
    
except Exception as e:
    logger.error(f"❌ Failed to connect to remote MCP server: {e}")
    logger.error(f"   Server URL: {MCP_SERVER_URL}")
    import traceback
    traceback.print_exc()
    
    # Create a fallback agent without tools
    logger.warning("⚠️  Creating fallback agent without MCP tools")
    root_agent = Agent(
        name="weather_agent",
        model=os.getenv("GEMINI_MODEL", "gemini-2.5-flash"),
    )
```

---

## 6. Bước 3: Chạy và kiểm tra

### 6.1. Chạy MCP Server (Terminal 1)

```bash
# Terminal 1 - MCP Server
cd 04-lab/mcp-server
.venv\Scripts\activate
set WEATHERAPI_KEY=your_weatherapi_key
uv run python weather.py
```

**Giữ terminal này chạy!**

### 6.2. Chạy ADK Web Interface (Terminal 2)

```bash
# Terminal 2 - ADK Client
cd 04-lab/mcp-client
.venv\Scripts\activate
uv run adk web
```

**Kết quả mong đợi:**
```
🚀 ADK Web interface starting...
📍 Local URL: http://localhost:8000
```

### 6.3. Truy cập giao diện web

1. Mở trình duyệt
2. Truy cập: http://localhost:8000
3. Chọn **"weather_agent"** từ danh sách agents
4. Bắt đầu chat!

### 6.4. Các câu hỏi thử nghiệm

```text
1. "Thời tiết Hà Nội như thế nào?"
2. "Dự báo thời tiết Đà Nẵng 3 ngày tới"
3. "Cho tôi biết thời tiết Tokyo"
4. "So sánh thời tiết Sydney và Melbourne"
5. "Weather in London right now"
```

### 6.5. Kiểm tra bằng script

```bash
# Chạy script kiểm tra
cd 04-lab/mcp-client
uv run python verify_setup.py
```

**Kết quả mong đợi:**
```
============================================================
Weather Agent Setup Verification
============================================================

🔍 Checking environment configuration...
✅ GOOGLE_API_KEY configured (AIza...)

🔍 Checking dependencies...
✅ Google ADK
✅ Google Generative AI
✅ MCP
✅ FastMCP
...

🔍 Checking agent structure...
✅ weather_agent/agent.py
✅ weather_agent/__init__.py

✅ All checks passed!

🚀 Ready to start!
   Run: uv run adk web
```

---

## 7. Bước 4: Triển khai Cloud Run (tùy chọn)

### 7.1. Triển khai MCP Server lên Google Cloud Run

#### 7.1.1. Tạo Dockerfile

**File `04-lab/mcp-server/Dockerfile`:**
```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install uv
RUN pip install uv

# Copy dependencies
COPY pyproject.toml .

# Install dependencies
RUN uv sync

# Copy application code
COPY weather.py .

# Set environment variables
ENV PORT=8080
ENV HOST=0.0.0.0

# Run the server
CMD ["uv", "run", "python", "weather.py"]
```

#### 7.1.2. Build và Deploy

```bash
# Authenticate with Google Cloud
gcloud auth login
gcloud auth configure-docker

# Set project
gcloud config set project YOUR_PROJECT_ID

# Build container image
cd 04-lab/mcp-server
docker build -t gcr.io/YOUR_PROJECT_ID/weather-mcp-server:latest .

# Push to Container Registry
docker push gcr.io/YOUR_PROJECT_ID/weather-mcp-server:latest

# Deploy to Cloud Run
gcloud run deploy weather-mcp-server \
    --image gcr.io/YOUR_PROJECT_ID/weather-mcp-server:latest \
    --platform managed \
    --region asia-east1 \
    --allow-unauthenticated \
    --set-env-vars WEATHERAPI_KEY=your_api_key \
    --port 8080
```

#### 7.1.3. Cập nhật ADK Client để dùng Cloud Run

Sau khi deploy thành công, cập nhật `weather_agent/agent.py`:

```python
# Thay đổi URL từ localhost thành Cloud Run URL
MCP_SERVER_URL = "https://weather-mcp-server-xxxxx-as.a.run.app/mcp"
```

---

## 8. Giải thích code chi tiết

### 8.1. MCP Server (`weather.py`)

```python
# 1. Khởi tạo FastMCP server với Streamable HTTP
mcp = FastMCP("weather", host="0.0.0.0", port=port)

# 2. Định nghĩa tool với decorator
@mcp.tool()
async def get_current_weather(city: str) -> str:
    """
    Docstring này tự động trở thành tool description
    cho MCP Client khám phá
    """
    # ... implementation

# 3. Chạy server với HTTP transport
mcp.run(transport="streamable-http")
```

**Điểm quan trọng:**
- `FastMCP` tự động sinh tool schema từ type hints
- Transport `streamable-http` cho phép kết nối qua HTTP
- Server tự động expose endpoint `/mcp` cho MCP operations

### 8.2. ADK Agent (`agent.py`)

```python
# 1. Kết nối đến Remote MCP Server
connection_params = StreamableHTTPConnectionParams(
    url="http://localhost:8085/mcp",
    timeout=30.0,
)

# 2. Tạo MCP Toolset (Client side)
weather_tools = McpToolset(
    connection_params=connection_params,
)

# 3. ADK tự động:
#    - Gọi list_tools() để khám phá tools
#    - Truyền tools cho Gemini
#    - Điều phối vòng lặp function calling

# 4. Tạo Agent với tools
root_agent = Agent(
    name="weather_agent",
    model="gemini-2.5-flash",
    tools=[weather_tools],  # Tools được tự động dùng
)
```

### 8.3. ADK Web Interface

```bash
# Lệnh khởi động
uv run adk web

# ADK Web làm gì:
# 1. Khởi tạo session management
# 2. Load agent từ weather_agent package
# 3. Cung cấp giao diện chat tại localhost:8000
# 4. Quản lý multiple sessions
```

---

## 9. Các lỗi thường gặp và cách khắc phục

### Lỗi 1: MCP Server không chạy được

**Triệu chứng:**
```
ERROR: WeatherAPI key not set
```

**Nguyên nhân:** Chưa set `WEATHERAPI_KEY` environment variable

**Cách khắc phục:**
```bash
# Windows
set WEATHERAPI_KEY=your_api_key
uv run python weather.py

# Hoặc tạo file .env
echo WEATHERAPI_KEY=your_api_key > .env
```

---

### Lỗi 2: ADK Agent không kết nối được MCP Server

**Triệu chứng:**
```
Connection refused: [Errno 111] Connection refused
```

**Nguyên nhân:** MCP Server chưa chạy hoặc sai URL

**Cách khắc phục:**
```bash
# 1. Kiểm tra MCP Server có chạy không
curl http://localhost:8085/mcp

# 2. Kiểm tra URL trong agent.py
# Đảm bảo: MCP_SERVER_URL = "http://localhost:8085/mcp"

# 3. Khởi động lại MCP Server trước
```

---

### Lỗi 3: Gemini API Key không hợp lệ

**Triệu chứng:**
```
GoogleGenerativeAIError: Invalid API key
```

**Cách khắc phục:**
```bash
# 1. Kiểm tra API key trong .env
cat .env

# 2. Đảm bảo format đúng
GEMINI_API_KEY=AIza... (bắt đầu bằng AIza)

# 3. Tạo key mới tại: https://aistudio.google.com/apikey
```

---

### Lỗi 4: Import lỗi khi chạy ADK

**Triệu chứng:**
```
ModuleNotFoundError: No module named 'google.adk'
```

**Cách khắc phục:**
```bash
# 1. Kích hoạt virtual environment
.venv\Scripts\activate

# 2. Cài đặt lại dependencies
uv sync

# 3. Hoặc cài đặt thủ công
uv pip install google-adk google-generativeai
```

---

### Lỗi 5: WeatherAPI trả lỗi

**Triệu chứng:**
```
Unable to fetch current weather data for Hanoi
```

**Nguyên nhân:** API key hết hạn, hết quota, hoặc thành phố không tìm thấy

**Cách khắc phục:**
```bash
# 1. Kiểm tra API key còn active không
curl "https://api.weatherapi.com/v1/current.json?key=YOUR_KEY&q=Hanoi"

# 2. Thử thành phố khác
# 3. Kiểm tra quota: https://www.weatherapi.com/docs/
```

---

### Lỗi 6: Port bị chiếm

**Triệu chứng:**
```
OSError: [Errno 10048] Only one usage of each socket address
```

**Cách khắc phục:**
```bash
# 1. Tìm process chiếm port
netstat -ano | findstr :8085

# 2. Kill process (thay PID bằng số thực tế)
taskkill /PID 1234 /F

# 3. Hoặc đổi port khác
set PORT=8086
uv run python weather.py
```

---

## 10. Bài tập mở rộng

### Bài tập 1: Thêm tool mới

Thêm một tool để lấy thông tin chất lượng không khí (Air Quality):

```python
@mcp.tool()
async def get_air_quality(city: str) -> str:
    """Get air quality index for a city.
    
    Args:
        city: City name (e.g., "Hanoi", "Bangkok", "Tokyo")
    """
    params = {
        "q": city,
    }
    
    data = await make_weather_request("current.json", params)
    # ... xử lý AQI
```

### Bài tập 2: Thêm authentication

Thêm Bearer token authentication cho MCP Server:

```python
# Trong mcp-server/weather.py
from mcp.server.auth import BearerTokenVerifier

class MyAuthVerifier(BearerTokenVerifier):
    def __init__(self, token: str):
        self.token = token
    
    async def verify_token(self, token: str) -> bool:
        return token == self.token

# Cấu hình auth khi khởi tạo
mcp = FastMCP(
    "weather",
    auth_verifier=MyAuthVerifier(os.getenv("MCP_AUTH_TOKEN"))
)
```

### Bài tập 3: Caching

Thêm caching cho API calls:

```python
from functools import lru_cache
import time

# Cache weather data trong 10 phút
@lru_cache(maxsize=100)
def get_cached_weather(city: str):
    # ... implementation
```

### Bài tập 4: Multi-language support

Mở rộng agent để hỗ trợ nhiều ngôn ngữ:

```python
# Trong agent instruction
instruction="""Bạn là trợ lý thời tiết đa ngôn ngữ.
- Người dùng hỏi tiếng Việt → Trả lời tiếng Việt
- Người dùng hỏi tiếng Anh → Trả lời tiếng Anh
- Người dùng hỏi tiếng Nhật → Trả lời tiếng Nhật"""
```

### Bài tập 5: Error handling nâng cao

Thêm retry logic cho API calls:

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=2, max=10))
async def make_weather_request_with_retry(endpoint: str, params: dict):
    # ... implementation với retry logic
```

---

## Tóm tắt

Qua bài lab này, bạn đã học được:

| Kiến thức | Mô tả |
|-----------|-------|
| **MCP Protocol** | Hiểu cách MCP Client-Server hoạt động |
| **FastMCP** | Tạo MCP Server với FastMCP framework |
| **Google ADK** | Sử dụng ADK để xây dựng Agent |
| **Streamable HTTP** | Kết nối đến Remote MCP Server qua HTTP |
| **Tool Integration** | Tích hợp tools vào Agent một cách tự động |
| **Production Deployment** | Triển khai MCP Server lên Cloud Run |

**Cấu trúc hoàn chỉnh sau lab:**

```
04-lab/
├── mcp-server/
│   ├── weather.py           ✅ Đã hoàn thành
│   ├── pyproject.toml       ✅ Đã hoàn thành
│   └── .env                 ✅ Đã cấu hình
│
└── mcp-client/
    ├── weather_agent/
    │   ├── __init__.py      ✅ Đã hoàn thành
    │   └── agent.py         ✅ Đã hoàn thành
    ├── pyproject.toml       ✅ Đã hoàn thành
    └── .env                 ✅ Đã cấu hình
```

**Cách chạy hoàn chỉnh:**

```bash
# Terminal 1: MCP Server
cd 04-lab/mcp-server
.venv\Scripts\activate
set WEATHERAPI_KEY=your_api_key
uv run python weather.py

# Terminal 2: ADK Web
cd 04-lab/mcp-client
.venv\Scripts\activate
uv run adk web

# Mở browser: http://localhost:8000
```

---

## Liên kết hữu ích

- [MCP Documentation](https://modelcontextprotocol.io/)
- [Google ADK Documentation](https://google.github.io/adk-docs/)
- [FastMCP GitHub](https://github.com/jlowin/fastmcp)
- [WeatherAPI Documentation](https://www.weatherapi.com/docs/)
- [Gemini API Keys](https://aistudio.google.com/apikey)

---

*Hướng dẫn hoàn thành Lab 04 - Weather Agent with Remote MCP Server*
*Generated: 2026-08-28*
