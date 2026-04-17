#  Delivery Checklist — Day 12 Lab Submission

> **Student Name:** Dau Van Quyen  
> **Student ID:** 2A202600359
> **Date:** 17/04/2026

---

##  Submission Requirements

Submit a **GitHub repository** containing: https://dauvanquyen-2a202600359-production.up.railway.app/

### 1. Mission Answers (40 points)

Create a file `MISSION_ANSWERS.md` with your answers to all exercises:

```markdown
# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
1. **Hardcoded API key**: `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` - This exposes secrets in source code
2. **Hardcoded database URL**: `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"` - Contains credentials in plain text
3. **No config management**: Variables like `DEBUG = True`, `MAX_TOKENS = 500` are hardcoded
4. **Improper logging**: Using `print()` instead of structured logging, and logging secrets: `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`
5. **No health check endpoint**: Missing `/health` endpoint required for cloud platforms
6. **Fixed port**: `port=8000` hardcoded instead of reading from `PORT` environment variable
7. **Localhost binding**: `host="localhost"` prevents external connections in containers
8. **Debug reload in production**: `reload=True` should only be used in development
9. **No graceful shutdown**: Missing signal handlers for clean shutdown
10. **No CORS configuration**: Missing security headers and origin restrictions

### Exercise 1.3: Comparison table
| Feature | Basic (Develop) | Advanced (Production) | Why Important? |
|---------|-----------------|----------------------|----------------|
| Config | Hardcoded values in code | Environment variables via pydantic | Secrets stay out of code, easy deployment across environments |
| Health check | Missing | `/health` and `/ready` endpoints | Cloud platforms need this to monitor service health and restart if needed |
| Logging | `print()` statements, logs secrets | Structured JSON logging, no secrets logged | Proper monitoring, debugging, and security compliance |
| Shutdown | Abrupt termination | Graceful shutdown with signal handling | Prevents data loss and ensures in-flight requests complete |
| Port binding | `localhost:8000` fixed | `0.0.0.0:$PORT` from env | Works in containers, accepts external connections |
| Security | No authentication | CORS, API key validation | Prevents unauthorized access and abuse |
| Error handling | Basic exceptions | Proper HTTP status codes, validation | Better API reliability and debugging |
| Lifecycle | No startup/shutdown logic | Lifespan management with async context | Proper resource initialization and cleanup |

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. Base image: python:3.11-slim - A lightweight Python image
2. Working directory: /app - All subsequent commands run in this directory
3. Why COPY requirements.txt first?: To leverage Docker layer caching - requirements installation only re-runs when requirements.txt changes, not when source code changes
4. CMD vs ENTRYPOINT difference: CMD provides default arguments that can be overridden, ENTRYPOINT defines the executable and arguments that cannot be overridden at runtime

### Exercise 2.3: Image size comparison
- Develop: 424 MB
- Production: 57 MB
- Difference: 87%

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- URL: https://dauvanquyen-2a202600359-production.up.railway.app
- Screenshot: ![alt text](image.png)

## Part 4: API Security

### Exercise 4.1-4.3: Test results
**API Key authentication:**
```bash
# Without API key - returns 401
curl http://localhost:8000/ask -X POST -H "Content-Type: application/json" -d '{"question": "Hello"}'
# Response: {"detail": "Unauthorized"}

# With correct API key - returns 200
curl http://localhost:8000/ask -X POST -H "X-API-Key: secret-key-123" -H "Content-Type: application/json" -d '{"question": "Hello"}'
# Response: {"answer": "Mock response for: Hello"}
```

**JWT authentication:**
```bash
# Get token
curl http://localhost:8000/token -X POST -H "Content-Type: application/json" -d '{"username": "admin", "password": "secret"}'
# Response: {"access_token": "eyJ...", "token_type": "bearer"}

# Use token
TOKEN="eyJ..."
curl http://localhost:8000/ask -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"question": "Explain JWT"}'
# Response: {"answer": "Mock response for: Explain JWT"}
```

**Rate limiting:**
```bash
# Send 15 requests quickly
for i in {1..15}; do
  curl -s http://localhost:8000/ask -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"question": "Test '$i'"}'
done

# Results: First 10 succeed (200), next 5 return 429 Too Many Requests
```

### Exercise 4.4: Cost guard implementation
**Approach implemented:**
```python
def check_budget(user_id: str, estimated_cost: float) -> bool:
    month_key = datetime.now().strftime("%Y-%m")
    key = f"budget:{user_id}:{month_key}"

    current = float(r.get(key) or 0)
    if current + estimated_cost > 10:  # $10 monthly limit
        return False

    r.incrbyfloat(key, estimated_cost)
    r.expire(key, 32 * 24 * 3600)  # 32 days expiry
    return True
```

**Features:**
- Monthly budget tracking per user
- Redis-based persistence
- Automatic key expiration
- Cost estimation before API calls

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes
**Health checks:**
```python
@app.get("/health")
def health():
    """Liveness probe - returns 200 if process is running"""
    return {"status": "ok"}

@app.get("/ready")
def ready():
    """Readiness probe - returns 200 if ready to serve traffic"""
    try:
        # Check Redis connection
        r.ping()
        return {"status": "ready"}
    except:
        raise HTTPException(status_code=503, detail="Not ready")
```

**Graceful shutdown:**
```python
def shutdown_handler(signum, frame):
    logger.info("Received SIGTERM - initiating graceful shutdown")
    # Stop accepting new requests
    server.should_exit = True
    # Wait for in-flight requests to complete
    time.sleep(1)
    # Close connections
    logger.info("Shutdown complete")

signal.signal(signal.SIGTERM, shutdown_handler)
```

**Stateless design:**
```python
# Anti-pattern (stateful):
conversation_history = {}  # In memory - lost on restart

# Correct (stateless):
history = r.lrange(f"history:{user_id}", 0, -1)  # From Redis
r.rpush(f"history:{user_id}", question, response)
```

**Load balancing test:**
```bash
docker compose up --scale agent=3
```
- 3 agent instances started successfully
- Nginx distributes requests across instances
- Load balancing verified through logs
- Individual instance failures don't affect overall service

**Stateless scaling test:**
- Created conversation with instance 1
- Killed instance 1
- Subsequent requests routed to instance 2
- Conversation history preserved in Redis
- No data loss during scaling events
```

---

### 2. Full Source Code - Lab 06 Complete (60 points)

Your final production-ready agent with all files:

```
your-repo/
├── app/
│   ├── main.py              # Main application
│   ├── config.py            # Configuration
│   ├── auth.py              # Authentication
│   ├── rate_limiter.py      # Rate limiting
│   └── cost_guard.py        # Cost protection
├── utils/
│   └── mock_llm.py          # Mock LLM (provided)
├── Dockerfile               # Multi-stage build
├── docker-compose.yml       # Full stack
├── requirements.txt         # Dependencies
├── .env.example             # Environment template
├── .dockerignore            # Docker ignore
├── railway.toml             # Railway config (or render.yaml)
└── README.md                # Setup instructions
```

**Requirements:**
-  All code runs without errors
-  Multi-stage Dockerfile (image < 500 MB)
-  API key authentication
-  Rate limiting (10 req/min)
-  Cost guard ($10/month)
-  Health + readiness checks
-  Graceful shutdown
-  Stateless design (Redis)
-  No hardcoded secrets

---

### 3. Service Domain Link

Create a file `DEPLOYMENT.md` with your deployed service information:

```markdown
# Deployment Information

## Public URL
https://dauvanquyen-2a202600359-production.up.railway.app

## Platform
Railway / Render / Cloud Run

## Test Commands

### Health Check
```bash
curl https://dauvanquyen-2a202600359-production.up.railway.app/health
# Expected: {"status": "ok"}
```

### API Test (with authentication)
```bash
curl -X POST https://dauvanquyen-2a202600359-production.up.railway.app/ask \
  -H "X-API-Key: prod-2a202600359" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
```

## Environment Variables Set
- PORT
- REDIS_URL
- AGENT_API_KEY
- LOG_LEVEL

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
```

##  Pre-Submission Checklist

- [x] Repository is public (or instructor has access)
- [x] `MISSION_ANSWERS.md` completed with all exercises
- [x] `DEPLOYMENT.md` has working public URL
- [x] All source code in `app/` directory
- [x] `README.md` has clear setup instructions
- [x] No `.env` file committed (only `.env.example`)
- [x] No hardcoded secrets in code
- [x] Public URL is accessible and working
- [x] Screenshots included in `screenshots/` folder
- [x] Repository has clear commit history

---

##  Self-Test

Before submitting, verify your deployment:

```bash
# 1. Health check
curl https://dauvanquyen-agent-production.up.railway.app/health

# 2. Authentication required
curl https://dauvanquyen-agent-production.up.railway.app/ask
# Should return 401

# 3. With API key works
curl -H "X-API-Key: prod-2a202600359" https://dauvanquyen-agent-production.up.railway.app/ask \
  -X POST -d '{"user_id":"test","question":"Hello"}'
# Should return 200

# 4. Rate limiting
for i in {1..15}; do 
  curl -H "X-API-Key: prod-2a202600359" https://dauvanquyen-agent-production.up.railway.app/ask \
    -X POST -d '{"user_id":"test","question":"test"}'; 
done
# Should eventually return 429
```

---

##  Submission

**Submit your GitHub repository URL:**

```
https://github.com/thayquyendau/lab12-DauVanQuyen-2A202600359/
```

**Deadline:** 17/4/2026

---

##  Quick Tips

1.  Test your public URL from a different device
2.  Make sure repository is public or instructor has access
3.  Include screenshots of working deployment
4.  Write clear commit messages
5.  Test all commands in DEPLOYMENT.md work
6.  No secrets in code or commit history

---

##  Need Help?

- Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
- Review [CODE_LAB.md](CODE_LAB.md)
- Ask in office hours
- Post in discussion forum

---

**Good luck! **
