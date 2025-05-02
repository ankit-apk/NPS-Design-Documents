# 🧠 Caching Service (LLD)

Supports: **REST (Fiber)** & **gRPC** | Cache store: **Redis / KeyDB**
Environment: **Kubernetes-native**, no vendor lock-in

---

## ⚙️ Functional Goals

- Central cache for infrequently changing or reusable data across services (e.g., templates, user preferences, feature flags).
- Uniform API to set, get, delete, and clear cache.
- TTL-based key expiration.
- Fast internal access via gRPC, external/internal fallback via REST.

---

## 🔁 API Interfaces

### 🔗 REST (Go + Fiber)

| Method   | Route              | Description           |
| -------- | ------------------ | --------------------- |
| `GET`    | `/cache/:ns/:key`  | Get cached value      |
| `POST`   | `/cache/:ns/:key`  | Set cache value + TTL |
| `DELETE` | `/cache/:ns/:key`  | Delete key            |
| `POST`   | `/cache/clear/:ns` | Clear namespace       |

### 🔗 gRPC

```protobuf
service CacheService {
  rpc Get (CacheKey) returns (CacheValue);
  rpc Set (CacheSetRequest) returns (CacheStatus);
  rpc Delete (CacheKey) returns (CacheStatus);
  rpc ClearNamespace (Namespace) returns (CacheStatus);
}
```

---

## 🏗️ Internal Architecture

```
         ┌────────────────────┐
         │  Notification Svc │
         └────────────────────┘
               ▲      ▲
        gRPC   │      │   REST
               ▼      ▼
      ┌────────────────────────┐
      │     Caching Service    │
      │ ┌────────┐ ┌─────────┐ │
      │ │ gRPC   │ │ REST API│ │
      │ └────────┘ └─────────┘ │
      └────────────────────────┘
               ▲
               │
               ▼
        ┌──────────────┐
        │ Redis/KeyDB  │
        └──────────────┘
```

---

## 🧵 Developer Responsibility

| Task                     | Description                                          |
| ------------------------ | ---------------------------------------------------- |
| **REST API**             | Implement using Go + Fiber                           |
| **gRPC Server**          | Use `google.golang.org/grpc`                         |
| **Redis Integration**    | Use `go-redis`                                       |
| **TTL Support**          | Use `SET key value EX` with Redis                    |
| **Namespace Management** | Abstract into key-prefix pattern                     |
| **Logging**              | Hook into Loki-compatible logs                       |
| **gRPC/REST Adapter**    | Keep business logic shared, expose via dual handlers |
| **Go SDK**               | Create a common client for both REST and gRPC        |
| **Unit + Load Testing**  | Test performance at high concurrency levels          |

---

## 🛠️ DevOps Responsibility

| Task                         | Description                                           |
| ---------------------------- | ----------------------------------------------------- |
| **Redis HA Deployment**      | Deploy Redis or KeyDB on Kubernetes via Helm          |
| **Service Mesh (optional)**  | mTLS + Load balancing                                 |
| **Ingress/Service Exposure** | ClusterIP for internal use, optional ingress for REST |
| **Metrics Exporter**         | Redis Exporter, gRPC server stats                     |
| **LGTM Integration**         | Loki (logs), Grafana (dashboards), Tempo (traces)     |
| **Secrets Management**       | Redis creds via Kubernetes Secrets                    |
| **Rate Limiting**            | Use API Gateway or Envoy filters                      |
| **CI/CD Pipelines**          | Separate jobs for gRPC & REST tests and deployments   |

---

## 🧾 Sample Redis Key Convention

```
namespace:key
```

Example:

- `template:order_email_en`
- `user_pref:U12345`
- `auth:jwt_public_keys`

---

## 🔐 Security Considerations

| Scope                  | Action                                                      |
| ---------------------- | ----------------------------------------------------------- |
| Internal Services      | Use internal service DNS + gRPC mutual TLS (if mesh exists) |
| REST Exposure          | Optional — protect with IP allowlist, token auth            |
| Redis Protection       | No direct Redis access from other services                  |
| Authn/Authz (optional) | Include service-token middleware                            |

---

## 🎯 Cache TTL Best Practices

| Data Type         | TTL Example |
| ----------------- | ----------- |
| Email Templates   | 12–24h      |
| Feature Flags     | 6–12h       |
| Public Keys       | 1h          |
| Static User Prefs | 2–6h        |

---

## 📊 Observability Hooks (via LGTM)

- **Loki**: Log cache hit/miss, TTL expiry, eviction
- **Grafana**: Redis memory, key count, command rate
- **Tempo**: End-to-end trace from app → cache
- **Mimir**: Redis CPU, latency, cluster health

---

## 🧰 Redis HA Deployment Example (Helm)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install cache-redis bitnami/redis \
  --set auth.enabled=true \
  --set architecture=replication
```

Kubernetes Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: caching-service
spec:
  selector:
    app: caching-service
  ports:
    - name: grpc
      port: 50051
    - name: rest
      port: 8080
```

---

## 📁 File Structure (Suggestion)

```
/caching-service
│
├── grpc/
│   ├── proto/
│   └── server.go
│
├── rest/
│   └── handlers.go
│
├── redis/
│   └── client.go
│
├── shared/
│   └── logic.go
│
├── main.go
└── go.mod
```
