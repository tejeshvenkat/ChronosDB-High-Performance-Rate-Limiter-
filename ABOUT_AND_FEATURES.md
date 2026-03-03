# ChronosDB – What the Project Is About & Other Things

---

## What the Project Is About

**ChronosDB** is a **high-performance, distributed rate limiter and quota engine**. It enforces API limits accurately across many servers so that a user’s quota is never “double-spent” because of race conditions.

- **What it does:** A client asks: “Can this user make one more request now?” ChronosDB answers **allowed** or **denied** and, if denied, **when to retry** (`retry_after_ms`).
- **Why it exists:** In distributed systems, a simple “check then increment” can let two servers both allow a request and over-use the quota. ChronosDB does **check and record in one atomic step** (using Redis + Lua) so the limit is enforced correctly at scale.
- **Where it fits:** Suited for API gateways, platform quotas (e.g. Google-style API limits), or any service that needs strict, global rate limiting across instances and regions.

---

## Other Things (Features, Tech, What’s Included)

### Core behaviour
- **Sliding-window log** – Uses a sliding window (not fixed window) for accurate, fair limiting.
- **Atomic operations** – One Redis Lua script does: trim old entries → count → allow + record, or deny + compute retry time. No races.
- **gRPC API** – Single RPC: `CheckQuota(user_id, api_key, max_requests, window_seconds)` → `allowed`, `remaining_quota`, `retry_after_ms`.
- **L1 / L2 caching** – In-memory L1 cache for recent *denials* only (reduces Redis load); Redis (L2) is the single source of truth.
- **Stateless servers** – All quota state lives in Redis; you can run many ChronosDB instances behind a load balancer.

### Tech stack
- **Python 3.12+**, **asyncio** – Async gRPC and Redis.
- **gRPC + Protocol Buffers** – API defined in `proto/rate_limiter.proto`.
- **Redis** – Sorted sets + Lua for the sliding-window logic.
- **Prometheus** – Metrics: `requests_allowed_total`, `requests_denied_total`, `redis_latency_ms`, `grpc_latency_ms` on `/metrics`.
- **Docker & Kubernetes** – Dockerfile, docker-compose, and K8s manifests in `deploy/`.
- **Testing** – Pytest (unit tests), pytest-asyncio, and Locust (load tests).

### What’s in the repo
- **proto/** – gRPC API definition (`.proto`).
- **server/** – Async gRPC service, sliding-window logic, Redis client, L1 cache, metrics, config.
- **lua/** – Redis Lua script for atomic rate limiting.
- **client/** – Example gRPC client (`test_client.py`).
- **load_test/** – Locust load-test scenario.
- **deploy/docker** – Dockerfile and docker-compose (ChronosDB + Redis).
- **deploy/k8s** – Kubernetes deployment, service, and Redis manifests.
- **tests/** – Unit tests (service + sliding window).
- **README.md** – Architecture, design, how to run.
- **PROJECT_OVERVIEW.md** – Tech stack and project summary.

### How to run
1. Start Redis: `docker run --rm -p 6379:6379 redis:7-alpine`
2. Start server: `python -m server.main`
3. Call API: `python -m client.test_client`
4. (Optional) Open `http://localhost:8000/metrics` for Prometheus metrics.

---

You can use the “What the project is about” section for a GitHub description or About page, and the “Other things” section for features, tech stack, and repo layout.
