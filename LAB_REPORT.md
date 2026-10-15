# Báo Cáo Thực Hành Lab Day 26: MCP & Tools Integration

**Học viên:** Nguyễn Anh Hoàng  
**Khóa học:** VinAI Advanced Agentic AI Course (Track 3)  
**Ngày thực hiện:** 15/10/2026  
**Repository:** [Day26-MCP-Tools-Integration](https://github.com/huhupawn/Day26-MCP-Tools-Integration)

---

## 1. Mục Tiêu Bài Lab
1. **Phân biệt rõ ràng giữa Function Calling và Model Context Protocol (MCP)**: Hiểu cách hai cơ chế này kết hợp với nhau trong kiến trúc Agentic AI.
2. **Triển khai MCP Server & Client cơ bản**: Sử dụng giao thức chuẩn hóa qua stdio và Streamable HTTP.
3. **Mở rộng Production-grade**:
   - Xác thực & ủy quyền với Bearer Token.
   - Quản lý danh mục công cụ động (**Tool Registry & Discovery**).
   - Tương thích ngược & kiểm soát phiên bản (**Versioning & Backward Compatibility**).
4. **Xây dựng Weather Agent với Google ADK (Agent Development Kit)**:
   - MCP Server đóng vai trò dịch vụ cung cấp tools thời tiết (`get_current_weather`, `get_forecast`, `health_check`) qua giao thức **Streamable HTTP**.
   - Google ADK đóng vai trò **MCP Client**, tự động kết nối, khám phá tools, điều phối Function Calling với model **Gemini 3.8 Flash** và phục vụ người dùng qua Web UI.

---

## 2. Kiến Trúc Triển Khai (Lab 04)

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────────┐
│   User Browser  │ ───> │  ADK Web UI      │ ───> │  Weather Agent      │
│   localhost:8000│      │  (Google ADK)    │      │  (Agent with MCP)   │
└─────────────────┘      └──────────────────┘      └─────────────────────┘
                                                              │
                                                              │ Streamable HTTP
                                                              ▼ (localhost:8085/mcp)
                                                    ┌─────────────────────┐
                                                    │  MCP Server         │
                                                    │  FastMCP + Tools    │
                                                    └─────────────────────┘
                                                              │
                                                              ▼ REST API
                                                    ┌─────────────────────┐
                                                    │  WeatherAPI.com /   │
                                                    │  Fallback Mock Data │
                                                    └─────────────────────┘
```

---

## 3. Nhật Ký Thực Hiện & Khắc Phục Lỗi Kỹ Thuật

| Hạng mục | Vấn đề ban đầu | Giải pháp xử lý |
|---|---|---|
| **Python Environment** | Thiếu gói `uv`, môi trường client yêu cầu Python >= 3.12 trong khi root là 3.11 | Cài đặt `uv`, khởi tạo hai môi trường ảo riêng biệt (`.venv` Python 3.12 cho client và `.venv` Python 3.11 cho server). |
| **Model Deprecation** | Model cũ `gemini-2.5-flash` và `gemini-2.0-flash` bị API trả về 404 Deprecated | Nâng cấp toàn bộ lên model chuẩn **`gemini-3.8-flash`**. |
| **Windows UTF-8 Encoding** | Terminal Windows (cp1252) bị lỗi `UnicodeEncodeError` khi in emoji (✅, 🌧️) | Tái cấu hình `sys.stdout.reconfigure(encoding="utf-8")` trong code server. |
| **MCP Transport Default** | `weather.py` mặc định chạy stdio nếu terminal không phải interactive tty | Cập nhật logic khởi động: mặc định chạy **Streamable HTTP** trên port 8085 (`http://0.0.0.0:8085/mcp`). |
| **Dữ liệu thời tiết** | Chưa có `WEATHERAPI_KEY` từ dịch vụ bên ngoài | Bổ sung cơ chế Fallback Mock Data thông minh, cho phép hệ thống demo trọn vẹn luồng gọi tool ngay cả khi chưa gắn API key bên ngoài. |

---

## 4. Kết Quả Kiểm Thử (Verification Logs)

### 4.1. Kiểm thử tổng thể: `verify_setup.py`
```text
============================================================
Weather Agent Setup Verification
============================================================

🔍 Checking environment configuration...
✅ GOOGLE_API_KEY configured (AQ.Ab8RN6L...)

🔍 Checking dependencies...
✅ Google ADK
✅ Google Generative AI
✅ MCP
✅ FastMCP
✅ python-dotenv
✅ httpx

🔍 Checking agent structure...
✅ weather_agent/agent.py
✅ weather_agent/__init__.py

🔍 Checking MCP server connectivity...
✅ MCP server reachable at http://localhost:8085/mcp (HTTP 406)

🔍 Checking agent import...
✅ Agent imported successfully: weather_agent
   Model: gemini-3.8-flash

============================================================
✅ All checks passed!
```

### 4.2. Kiểm thử Agent qua CLI (`adk run`)
**Truy vấn:** *"What's the weather in Hanoi?"*
```text
[weather_agent]: The current weather in Hanoi is as follows:

* Condition: Partly Cloudy
* Temperature: 28°C (82°F)
* Feels like: 30°C (86°F)
* Humidity: 70%
* Wind: 15 km/h from the East
* Visibility: 10 km
* UV Index: 5
```

**Truy vấn:** *"What is the 2-day forecast for Tokyo?"*
```text
[weather_agent]: Here is the 2-day weather forecast for Tokyo:

### Day 1
* Condition: Sunny
* High / Low: 31°C (88°F) / 24°C (75°F)
* Chance of Rain: 10%
* Wind: Up to 14 km/h
* UV Index: 6

---

### Day 2
* Condition: Scattered Showers
* High / Low: 29°C (84°F) / 23°C (73°F)
* Chance of Rain: 50%
* Wind: Up to 18 km/h
* UV Index: 4
```

### 4.3. Kiểm thử các phần cơ bản khác
- **01-function-calling:** Script `weather_function_calling.py` chạy thành công vòng lặp gọi tool thủ công với `gemini-3.8-flash`.
- **02-mcp-basics:** Script `weather_client.py` tự động spawn `weather_server.py` qua stdio transport và gọi tool `get_weather`.
- **03-production:**
  - `registry_client.py`: Tự động tra cứu tool theo tag `"weather"`, chọn phiên bản v2.0 và kết nối thành công.
  - `versioned_client.py`: Đọc metadata `server://info`, phát hiện công cụ deprecated và tương thích ngược thành công giữa v1 và v2.

---

## 5. Tổng Kết Kiến Thức Cốt Lõi

| Tiêu chí | Function Calling | Model Context Protocol (MCP) |
|---|---|---|
| **Bản chất** | Khả năng của LLM (Model Capability) | Giao thức truyền thông chuẩn hóa (Protocol) |
| **Định nghĩa Tool** | Viết cứng (hard-code) schema trong mã nguồn app | Server tự công bố (self-describing) qua metadata/typehints |
| **Nơi thực thi** | Mã nguồn ứng dụng tự chạy | MCP Server thực thi độc lập (local stdio hoặc remote HTTP) |
| **Tái sử dụng** | Phải sao chép schema + hàm sang app khác | Một MCP Server phục vụ cho vô số MCP Client khác nhau |
| **Vai trò của Google ADK** | Đóng vai trò **MCP Client**: tự động hóa toàn bộ quá trình khám phá (`list_tools`), truyền schema cho Gemini, điều phối thực thi (`call_tool`) và trả về kết quả cho người dùng. |
