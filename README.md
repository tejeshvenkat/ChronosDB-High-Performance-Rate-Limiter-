ChronosDB – A High-Performance Distributed Rate Limiter & Quota Engine
======================================================================

### Overview

ChronosDB is a production-inspired, high-accuracy, distributed rate limiting
and quota enforcement service. It is designed to run as a horizontally
scalable gRPC service backed by Redis and to demonstrate how to build
Google-style infrastructure in Python:

- **Async gRPC server** using `grpc.aio`
- **Sliding Window Log** algorithm backed by **Redis Sorted Sets**
- **Atomic quota updates** via **Redis Lua scripting**
- **L1/L2 cache strategy** (in-memory + Redis)
- **Prometheus metrics** with Grafana-friendly exports
- **Kubernetes-ready** with Docker + K8s manifests

---

### Architecture Overview

- **Clients** call the `RateLimiter.CheckQuota` gRPC method.
- **ChronosDB server** (async Python, `grpc.aio`) receives the request and:
  - Derives a stable Redis key from `(user_id, api_key)`.
  - Checks a **local L1 negative cache** to short-circuit obviously denied keys.
  - Executes a **Lua script in Redis** that:
    - Maintains a **sliding window log** in a sorted set.
    - Performs **check + increment atomically**.
  - Updates **Prometheus metrics** for allowed/denied requests and latencies.
- **Redis** acts as:
  - The **L2 cache** and **source of truth** for quota state.
  - The execution environment for the sliding window logic.

This architecture is **stateless at the application layer**; any number of
ChronosDB instances can be added or removed without impacting correctness as
long as they point at the same Redis cluster.

---

### Sliding Window Log Algorithm

ChronosDB uses a **Sliding Window Log** instead of a fixed window:

- For each `(user_id, api_key)` pair, we maintain a Redis **sorted set**:
  - **Score** = request timestamp in milliseconds.
  - **Member** = the same timestamp (we only need ordering).
- When a request arrives at time \( t \):
  1. Remove all entries with score `< t - window_ms`.
  2. Count remaining entries.
  3. If `count < max_requests`:
     - Insert a new entry `(t, t)` into the sorted set.
     - Set a TTL on the key to allow Redis GC.
     - Allow the request.
  4. Otherwise:
     - Compute the **earliest** timestamp in the set.
     - Compute `retry_after_ms = window_ms - (t - earliest_ts)`.
     - Deny the request and tell the caller when to retry.

Because the logic runs entirely inside a single Lua script, the window
maintenance, count, and insert operations are **atomic**.

---

### Why Redis Lua Is Used

In a distributed system with many application servers:

- A naive implementation that does:
  1. `ZRANGE` / `ZCARD` to check usage
  2. `ZADD` to add a new entry
  …is vulnerable to **race conditions**. Two servers could both see
  `count < max_requests` and both increment, **double-spending** quota.

Using **Lua scripts in Redis** solves this:

- Redis executes each Lua script **atomically**.
- All steps of the algorithm (cleanup, count, insert, TTL) occur inside a
  **single transaction boundary**, ensuring:
  - No double-spend.
  - Correct sliding window semantics.

---

### L1 vs L2 Cache Trade-offs

- **L2 (Redis)**:
  - Source of truth.
  - Guarantees correctness and atomicity via Lua.
  - Higher per-request latency due to network hop.

- **L1 (in-memory)**
  - Lives inside each ChronosDB instance.
  - Used **only as a negative cache** for over-quota keys.
  - Stores a short-lived entry with `retry_after_ms` when Redis denies
    a request.
  - Subsequent requests can be denied **locally**, avoiding extra
    Redis round-trips.

**Correctness considerations:**

- By caching only **denials** and not **grants**, we avoid the risk
  of **under-enforcing** quotas:
  - Over-caching grants could allow more requests than intended.
  - Over-caching denials may deny a few extra requests but never
    exceed the quota.
- L1 TTLs are **capped** to a maximum (configurable) to avoid
  long-lived black-holes due to misconfigurations.

---

### Failure Scenarios

- **Redis unavailable**:
  - The core Lua script cannot execute.
  - Reasonable strategies (left for extension) include:
    - Fail-closed (deny all requests to protect downstream systems).
    - Fail-open (allow all requests to preserve availability).
  - In this reference implementation, failures will surface as gRPC
    errors; production deployments should add explicit policy.

- **Network partitions / high latency**:
  - Redis latency is exported as `redis_latency_ms`.
  - SREs can alert when this metric spikes and take action (e.g.
    failover, traffic shifting).

- **Application instance crashes**:
  - No local state (beyond L1 cache) is required.
  - A crashed instance loses its L1 cache, but Redis state remains.

---

### Scaling Strategy

- **Horizontal scale**:
  - Add more ChronosDB pods / containers.
  - They are stateless with respect to quotas; concurrency is
    handled by Redis + Lua.

- **Redis scaling**:
  - Start with a single Redis instance for simplicity.
  - Scale up to Redis Cluster / managed Redis as QPS grows.

- **Sharding keys**:
  - Redis keys are namespaced as `chronosdb:quota:{api_key}:{user_id}`.
  - This pattern lends itself well to Redis Cluster’s hash slot
    distribution.

---

### Metrics & Observability

ChronosDB exports the following **Prometheus metrics**:

- **`requests_allowed_total{api_key=...}`**
- **`requests_denied_total{api_key=...}`**
- **`redis_latency_ms`** (histogram)
- **`grpc_latency_ms`** (histogram)

The metrics HTTP server is exposed on `/metrics` (default port `8000`)
and is suitable for Grafana dashboards such as:

- Allowed vs Denied over time.
- Tail latencies (p95, p99) for Redis and gRPC.

---

### gRPC API

- **Service**: `RateLimiter`
- **RPC**: `CheckQuota(CheckQuotaRequest) returns (CheckQuotaResponse)`

`CheckQuotaRequest`:
- `string user_id`
- `string api_key`
- `int32 max_requests`
- `int32 window_seconds`

`CheckQuotaResponse`:
- `bool allowed`
- `int32 remaining_quota`
- `int64 retry_after_ms`

---

### Running Locally

1. **Install dependencies**

```bash
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

2. **Generate gRPC stubs**

```bash
chmod +x scripts/generate_grpc.sh
./scripts/generate_grpc.sh
```

3. **Start Redis**

```bash
docker run --rm -p 6379:6379 redis:7-alpine
```

4. **Run the server**

```bash
python -m server.main
```

5. **Call it from the test client**

```bash
python -m client.test_client
```

You should see some requests allowed and then denied as the quota
is exhausted within the sliding window.

---

### Docker & Kubernetes

- **Docker**:
  - `deploy/docker/Dockerfile` builds a self-contained image.
  - `deploy/docker/docker-compose.yml` runs:
    - `redis`
    - `chronosdb` (server + metrics)

- **Kubernetes**:
  - `deploy/k8s/redis.yaml` – Redis Deployment + Service.
  - `deploy/k8s/deployment.yaml` – ChronosDB Deployment (3 replicas).
  - `deploy/k8s/service.yaml` – ClusterIP Service for gRPC and metrics.

---

### Load Testing with Locust

ChronosDB includes a basic **Locust** load test in `load_test/locustfile.py`.

1. Ensure Redis and the ChronosDB server are running.
2. Set the gRPC target if needed:

```bash
export CHRONOSDB_GRPC_TARGET=localhost:50051
```

3. Run Locust:

```bash
locust -f load_test/locustfile.py
```

4. Open the Locust web UI (by default at `http://localhost:8089`) and
   start a test, e.g.:
   - Users: 200
   - Spawn rate: 50

---

### Example Load Testing Results (Template)

These values are **placeholders**; you should overwrite them with
numbers from your environment after running Locust.

- **Environment**: 1 Redis (container), 3 ChronosDB instances (K8s or Docker).
- **Test config**: 10,000 RPS target, 60-second window, 100 requests per window.

Sample (replace with your actual numbers):

- **Sustained RPS**: ~10k
- **gRPC p95 latency**: ~5–10 ms
- **Redis p95 latency**: ~2–5 ms
- **Error rate**: < 0.1%

Documenting your own results here (hardware, config, and graphs) will
make this project look very strong on a resume or in an interview.

