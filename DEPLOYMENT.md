> **Student Name:** Hà Huy Hoàng

> **Student ID:** 2A202600054

> **Date:** 17/04/2026

---

# Deployment Information

## Public URL
https://serene-analysis-production-a125.up.railway.app

## Platform
Railway

## Test Commands

### Health Check
```bash
curl https://day12-hoang-agent-production.up.railway.app/health
```
Expected: `{"status":"ok","version":"1.0.0","environment":"production","uptime_seconds":...,"checks":{"llm":"mock"},"timestamp":"..."}`

### Readiness Check
```bash
curl https://day12-hoang-agent-production.up.railway.app/ready
```
Expected: `{"ready":true}`

### Authentication Required (should return 401)
```bash
curl -X POST https://day12-hoang-agent-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
```
Expected: `401 {"detail":"Invalid or missing API key. Include header: X-API-Key: <key>"}`

### API Test with authentication (should return 200)
```bash
curl -X POST https://day12-hoang-agent-production.up.railway.app/ask \
  -H "X-API-Key: hoang-secret-key-2026-vinuni" \
  -H "Content-Type: application/json" \
  -d '{"question":"What is cloud deployment?"}'
```
Expected: `200 {"question":"...","answer":"...","model":"gpt-4o-mini","timestamp":"..."}`

### Rate Limiting Test (20 req/min → gets 429 after limit)
```bash
for i in {1..25}; do
  curl -X POST https://day12-hoang-agent-production.up.railway.app/ask \
    -H "X-API-Key: hoang-secret-key-2026-vinuni" \
    -H "Content-Type: application/json" \
    -d '{"question":"test"}'
done
```
Expected: First 20 requests return 200, then 429 with `Retry-After: 60` header.

## Environment Variables Set on Railway
- `PORT` — auto-injected by Railway
- `REDIS_URL` — auto-injected from Railway Redis plugin
- `AGENT_API_KEY` — API authentication key
- `JWT_SECRET` — JWT signing secret
- `ENVIRONMENT=production`
- `APP_NAME=Day12 Hoang Agent`
- `APP_VERSION=1.0.0`
- `RATE_LIMIT_PER_MINUTE=20`
- `DAILY_BUDGET_USD=5.0`
