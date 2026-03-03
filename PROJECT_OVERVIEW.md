# ChronosDB – What We Used & About the Project

## What Was Used (Tech Stack & Tools)

### Language & Runtime
- **Python 3.12+** (project targets 3.12+; tested on 3.14)
- **asyncio** – All I/O (gRPC, Redis) is async for high concurrency and low latency.

### Communication & API
- **gRPC** – High-performance RPC instead of REST.
- **Protocol Buffers (proto3)** – Schema for the API (`proto/rate_limiter.proto`).
- **grpcio** – Python gRPC runtime.
- **grpcio-tools** – Compiles `.proto` files to Python (e.g. `rate_limiter_pb2.py`, `rate_limiter_pb2_grpc.py`).

### Storage & Logic
- **Redis** – Primary store for rate-limit state.
- **Redis Sorted Sets** – Used to implement the sliding-window log (timestamps as scores).
- **Lua** – Scripts run inside Redis so “check + increment” is a single atomic operation (`lua/rate_limit.lua`).
- **redis** (Python package) – Async Redis client (`redis.asyncio`).

### Caching
- **L1 cache** – In-process, in-memory cache in the ChronosDB server (see `server/l1_cache.py`). Used only as a *negative* cache (recent denials) to cut down Redis calls for over-quota keys.
- **L2** – Redis is the global source of truth; the Lua script and sorted sets implement the real quota logic.

### Observability
- **Prometheus** – Metrics format and scraping.
- **prometheus-client** (Python) – Exposes counters and histograms; HTTP server serves `/metrics`.

### Testing & Load
- **pytest** – Unit tests.
- **pytest-asyncio** – Runs async tests.
- **Locust** – Load testing (e.g. many concurrent gRPC `CheckQuota` calls); see `load_test/locustfile.py`.

### Infrastructure
- **Docker** – Container image for the server (`deploy/docker/Dockerfile`).
- **docker-compose** – Runs ChronosDB + Redis together (`deploy/docker/docker-compose.yml`).
- **Kubernetes** – Manifests for deployment, service, and Redis (`deploy/k8s/`).

### Other
- **Scripts** – `scripts/generate_grpc.sh` (or `python -m grpc_tools.protoc`) to generate gRPC stubs from the proto.

---

## About the Project

### What It Is
**ChronosDB** is a **distributed rate limiter and quota engine**. It answers: “Is this user allowed to make one more request right now, and if not, when can they retry?” It is built to run across many servers and geographies without “double-spending” a user’s API quota (no race conditions).

### The Problem It Solves
In distributed systems, if you only “check then increment” in two steps, two servers can both see “under limit” and both increment, allowing more requests than the limit. ChronosDB avoids that by doing the entire check-and-record in **one atomic step** inside Redis (Lua script + sorted sets).

### How It Works (Short Version)
1. **Client** calls the gRPC method `CheckQuota` with `user_id`, `api_key`, `max_requests`, and `window_seconds`.
2. **Server** (async Python):
   - Builds a Redis key from `(api_key, user_id)`.
   - Optionally consults the **L1 cache** for recent denials (to reduce Redis load).
   - Runs the **Lua script** in Redis, which:
     - Keeps a **sliding window** of request timestamps in a sorted set.
     - Removes old timestamps, counts current ones, and either allows and records the new request or denies and returns `retry_after_ms`.
   - Records **Prometheus metrics** (allowed/denied, Redis and gRPC latency).
3. **Response** is `allowed` (true/false), `remaining_quota`, and `retry_after_ms`.

### Main Design Choices
- **Sliding window log** (not fixed window) – More accurate and fair; no burst at window boundaries.
- **gRPC + Protobuf** – Efficient serialization and streaming; common in large-scale systems.
- **Redis Lua** – Atomicity without distributed locks; one script = one logical operation.
- **L1 negative cache only** – We never cache “allowed”; that would risk over-allowing. Caching “denied” only may over-deny slightly but never exceeds the quota.
- **Stateless app servers** – All quota state is in Redis; you can scale ChronosDB horizontally.

### Project Structure (Summary)
- **proto/** – API definition.
- **server/** – gRPC service, sliding-window logic, Redis client, L1 cache, metrics, config.
- **lua/** – Atomic rate-limit script.
- **client/** – Example gRPC client.
- **load_test/** – Locust scenario.
- **deploy/** – Docker and Kubernetes.
- **tests/** – Pytest tests.

### How to Run It
1. Start Redis: `docker run --rm -p 6379:6379 redis:7-alpine`
2. Start server: `python -m server.main`
3. Call API: `python -m client.test_client`
4. (Optional) Open `http://localhost:8000/metrics` in a browser to see Prometheus metrics.

---

This document summarizes what was used to build ChronosDB and what the project does. For more detail (algorithm, failure modes, scaling), see **README.md**.
