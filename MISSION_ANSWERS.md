> **Student Name:** Hà Huy Hoàng
> **Student ID:** 2A202600054  
> **Date:** 18-04-2026

---


# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
1. API key hardcode trong code
2. Database URL + password hardcode
3. DEBUG = True cứng trong code
4. MAX_TOKENS hardcode
5. Dùng print() thay vì logging
6. Log ra cả secret key
7. Không có /health endpoint
8. Không có /health endpoint
9. host="localhost"
...


### Exercise 1.3: Comparison table
| Feature | Basic | Advanced | Tại sao quan trọng? |
|---------|-------|----------|---------------------|
| Config | Hardcode OPENAI_API_KEY = "sk-..." | os.getenv("OPENAI_API_KEY") qua config.py | Secret không rò rỉ, đổi giữa dev/prod không cần sửa code |
| Health check |port=8000 cứng  |settings.port từ env PORT  | Railway/Render inject PORT tự động, app cần đọc đúng |
| Logging |print() + log cả API key | logging.basicConfig format JSON, không log secret | JSON parse được bằng Datadog/Loki; không leak credential |
| Shutdown | Ctrl+C cắt đột  | lifespan + handler SIGTERM | Request đang chạy được hoàn thành trước khi tắt |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. Base image: python:3.11
2. Working directory: WORKDIR /app
3. Tại sao COPY requirements.txt trước? 
- Mục đích chính: tận dụng Docker layer cache
- Cơ chế:
    * Docker build theo từng layer
    * Nếu requirements.txt không đổi → layer pip install được cache
    * Khi chỉ sửa app.py → không cần cài lại thư viện
4. CMD vs ENTRYPOINT khác nhau thế nào?
- CMD: bị thay thế
    * Là default command
    * Có thể bị override khi chạy container

- ENTRYPOINT
    * Là command chính, bắt buộc chạy
    * Không bị override dễ dàng
    * Thường dùng khi container đóng vai trò như 1 “tool”

### Exercise 2.3: Image size comparison
- Develop: 424 MB (content size), 1.66GB (disk usage) 
- Production: 56.6 MB, 236 MB
- Difference: ~87%

### Exercise 2.4: Phân tích `docker-compose.yml` và architecture

Services được start:
1. `agent`: FastAPI AI agent.
2. `redis`: cache/session/rate limiting store.
3. `qdrant`: vector database cho RAG/search.
4. `nginx`: reverse proxy + load balancer public-facing.

Cách các service communicate:
- Client chỉ gọi vào `nginx` qua port `80` (và `443` nếu có cert).
- `nginx` proxy request vào upstream `agent_backend` (`agent:8000`).
- `agent` gọi nội bộ qua network `internal` đến:
	- `redis:6379`
	- `qdrant:6333`
- `depends_on` + `healthcheck` đảm bảo thứ tự sẵn sàng cơ bản trước khi nhận traffic.

Architecture tóm tắt:
- `Client -> Nginx -> Agent`
- `Agent -> Redis`
- `Agent -> Qdrant`


PS D:\AI-in-Action\2A202600054-HaHuyHoang-Day12> docker images | Select-String "my-agent"
WARNING: This output is designed for human readability. For machine-readable output, please use --format.

my-agent:advanced   cd2815f54102        236MB         56.6MB        
my-agent:develop    8397fac0445b       1.66GB          424MB   U    

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- URL: https://perfect-beauty-production-bb61.up.railway.app
- Screenshot: [screenshots\railway-app.png](screenshots\railway-app.png)

### Exercise 3.2: So sánh `render.yaml` vs `railway.toml`

| Điểm so sánh | `railway.toml` | `render.yaml` |
|---|---|---|
| Format | TOML | YAML |
| Build system | Nixpacks (auto-detect) hoặc Dockerfile | Chỉ định rõ `buildCommand` |
| Health check | `healthcheckPath` + `healthcheckTimeout` | `healthCheckPath` |
| Restart policy | `restartPolicyType = "ON_FAILURE"` (tường minh) | Render tự xử lý, không cần khai báo |
| Secrets | Set qua CLI `railway variables set` hoặc Dashboard | `sync: false` (set tay trên Dashboard) hoặc `generateValue: true` |
| Redis | Không khai báo trong toml, add riêng trên Dashboard | Khai báo service `type: redis` ngay trong file |
| Region | Không có trong toml cơ bản | Có `region: singapore` |
| Auto-deploy | Mặc định khi push | `autoDeploy: true` tường minh |

**Nhận xét:** Railway ưu tiên CLI-driven workflow với cấu hình tối giản; Render ưu tiên Infrastructure-as-Code khai báo toàn bộ stack (app + Redis) trong 1 file.


## Part 4: API Security

### Exercise 4.1: Test results
```bash
a) No Key: 401 {'detail': 'Missing API key. Include header: X-API-Key: <your-key>'}
b) Wrong Key: 403 {'detail': 'Invalid API key.'}
c) Correct Key: 200 answer_present=True
```

### Exercise 4.2: Test results
```bash
d) Auth: 200 TokenLen: 168 Prefix: eyJhbGciOi...
e) No Token: 401 Detail: Authentication required. Include: Authorization: Bearer <token>
f) With Token: 200 Keys: ['question', 'answer', 'usage']
```

### Exercise 4.3: Test results
```text
Statuses: [200, 200, 200, 200, 200, 200, 200, 200, 200, 429, 429, 429]
First 429 index: 9
```


### Exercise 4.4: Cost guard implementation
Phần cost guard nằm ở `04-api-gateway/production/cost_guard.py`, cách làm:

- Track usage theo user theo ngày: input tokens, output tokens, số request.
- Tính chi phí từ token theo bảng giá mock (`PRICE_PER_1K_INPUT_TOKENS`, `PRICE_PER_1K_OUTPUT_TOKENS`).
- Chặn theo ngân sách user: mặc định `$1/ngày/user`, vượt thì trả `402`.
- Chặn theo ngân sách global: mặc định `$10/ngày`, vượt thì trả `503`.
- Cảnh báo khi chạm ngưỡng `80%` budget.

Flow bảo vệ request:

1. Check auth (API Key hoặc JWT tùy mode).
2. Check rate limit.
3. Check budget trước khi gọi model.
4. Gọi model xong thì ghi usage + cộng chi phí.


## Part 5: Scaling & Reliability

### Exercise 5.1 - Health checks và readiness:

- Ứng dụng develop (`05-scaling-reliability/develop/app.py`) có các endpoint:
	- `GET /health` cho liveness.
	- `GET /ready` cho readiness.
	- Có xử lý graceful shutdown (`SIGTERM`, `SIGINT`) và ghi `pid.txt`.
- Ứng dụng production (`05-scaling-reliability/production/app.py`) có:
	- `GET /health` và `GET /ready`.
	- Readiness kiểm tra kết nối Redis bằng `PING`.

### Exercise 5.2 - Kiến trúc stateless với Redis:

- Production lưu lịch sử hội thoại theo `session_id` trong Redis:
	- Key format: `chat:<session_id>`.
	- Mỗi message được append vào Redis list (JSON), không phụ thuộc bộ nhớ local của container.
- Biến `instance_id = socket.gethostname()` được trả về trong response (`served_by`) để chứng minh request có thể đi qua nhiều replica.
- Endpoint `POST /chat` và endpoint tương thích `POST /ask` đều dùng session state trong Redis.

### Exercise 5.3 - Cân bằng tải với Nginx:

- `05-scaling-reliability/production/nginx.conf` cấu hình upstream `agent_backend` cho 3 replica.
- Nginx expose cổng `8080` qua `docker-compose.yml` và route request vào upstream backend.

### Exercise 5.4 - Scale stack bằng Docker Compose:

- File `05-scaling-reliability/production/docker-compose.yml` mô tả stack gồm:
	- `redis` (có healthcheck),
	- `agent` (phụ thuộc `redis` healthy),
	- `nginx` (đứng trước agent).
- Bằng chứng chạy thực tế (log gần nhất):
	- `docker compose up -d --build --scale agent=3`
	- Kết quả: `up 7/7`, trong đó `production-redis-1 Healthy`, `production-agent-1/2/3 Started`, `production-nginx-1 Started`.

### Exercise 5.5 - Kiểm thử stateless:

- Script kiểm thử: `05-scaling-reliability/production/test_stateless.py`.
- Kết quả thực tế trong phiên làm việc:
	- Đã có bằng chứng request thành công qua Nginx:
		- `Invoke-WebRequest http://localhost:8080/health` trả về `StatusCode: 200`.
		- Body có `instance_id`, `storage: redis`, `redis_connected: true`.
		- Header có `X-Served-By` (backend phía sau Nginx).
	- Tuy nhiên, ở một số lần chạy chuỗi lệnh tự động, vẫn xuất hiện lỗi ngắt quãng:
		- `No connection could be made because the target machine actively refused it`.
		- `502 Bad Gateway`.
		- `test_stateless.py` có thể fail với `HTTPError 502` hoặc `URLError WinError 10061`.
