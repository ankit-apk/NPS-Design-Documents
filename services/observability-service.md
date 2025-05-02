# ✅ Observability LLD – LGTM Stack on Kubernetes

**Microservices in Go (Fiber) – No Vendor Lock-In**

---

## 🔷 1. Developer Responsibilities (Go + Fiber)

### 🔧 A. Structured Logging (Loki via Promtail)

#### Use Logrus with JSON format:

```go
import (
  "github.com/sirupsen/logrus"
  "os"
)

var log = logrus.New()

func initLogger() {
  log.SetFormatter(&logrus.JSONFormatter{})
  log.SetOutput(os.Stdout) // Important for Promtail
  log.SetLevel(logrus.InfoLevel)
}
```

#### Log Example:

```go
log.WithFields(logrus.Fields{
  "service": "user-service",
  "event":   "login_attempt",
  "user_id": "1234",
}).Info("User login attempt")
```

---

### 📊 B. Metrics (Mimir via Prometheus or OTEL SDK)

#### Option 1: Use Prometheus metrics

```go
import (
  "github.com/prometheus/client_golang/prometheus/promhttp"
  "github.com/gofiber/adaptor/v2"
)

app.Get("/metrics", adaptor.HTTPHandler(promhttp.Handler()))
```

#### Option 2: OTEL SDK metrics (optional):

You can push custom counters, gauges, and histograms using OpenTelemetry-Go metrics SDK (recommended for advanced tracing and histogram correlation).

---

### 🧵 C. Distributed Tracing (Tempo via OpenTelemetry)

#### 1. Add OTEL SDK

```go
import (
  "context"
  "go.opentelemetry.io/otel"
  "go.opentelemetry.io/otel/sdk/resource"
  "go.opentelemetry.io/otel/sdk/trace"
  semconv "go.opentelemetry.io/otel/semconv/v1.12.0"
  "go.opentelemetry.io/otel/exporters/otlp/otlpgrpc"
)

func initTracer() func(context.Context) error {
  ctx := context.Background()
  exp, _ := otlpgrpc.New(ctx,
    otlpgrpc.WithEndpoint("otel-collector.observability.svc.cluster.local:4317"),
    otlpgrpc.WithInsecure(),
  )

  tp := trace.NewTracerProvider(
    trace.WithBatcher(exp),
    trace.WithResource(resource.NewWithAttributes(
      semconv.SchemaURL,
      semconv.ServiceNameKey.String("user-service"),
    )),
  )

  otel.SetTracerProvider(tp)
  return tp.Shutdown
}
```

#### 2. Use OTEL Middleware in Go Fiber

```go
import otelfiber "go.opentelemetry.io/contrib/instrumentation/github.com/gofiber/fiber/otelfiber"

app.Use(otelfiber.Middleware())
```

---

### 🛠️ D. Summary for Developers

| Task           | Required Action                                        |
| -------------- | ------------------------------------------------------ |
| Logging        | Use Logrus with JSON to stdout                         |
| Metrics        | Expose `/metrics` or use OTEL SDK                      |
| Tracing        | Initialize OTEL SDK + Wrap routes with OTEL middleware |
| Error Tracking | Log errors as structured logs                          |
| Correlation    | Use trace/span IDs in logs (OTEL auto injects headers) |

---

## 🧰 2. DevOps Responsibilities (Kubernetes + LGTM Infra)

### 🧱 A. Deploy Loki + Promtail

#### Promtail (DaemonSet)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: promtail
spec:
  containers:
    - name: promtail
      image: grafana/promtail:latest
      args:
        - -config.file=/etc/promtail/promtail.yaml
      volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: dockercontainers
          mountPath: /var/lib/docker/containers
```

---

### 📊 B. Deploy Mimir for Metrics

- Run Mimir in **monolithic mode** or HA setup.
- Use OTEL Collector to scrape metrics and send to Mimir.

---

### 🧵 C. Deploy Tempo for Tracing

- Deploy Tempo StatefulSet (simple single binary).
- Expose GRPC endpoint `4317` for OTLP.

---

### 📦 D. Deploy OpenTelemetry Collector (Central Router)

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

exporters:
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
  mimir:
    endpoint: http://mimir:9009
  tempo:
    endpoint: tempo:4317

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [tempo]
    metrics:
      receivers: [otlp]
      exporters: [mimir]
    logs:
      receivers: [otlp]
      exporters: [loki]
```

---

### 📺 E. Deploy Grafana

- Connect **Loki** as log source.
- Connect **Tempo** as trace source.
- Connect **Mimir** as Prometheus data source.

---

### 🚨 F. Setup Alerts and Dashboards

- Use LogQL, PromQL, and TraceQL for queries.
- Set alerting rules via Grafana or Alertmanager.

Example alert (log-based):

```logql
sum(rate({service="user-service", level="error"}[5m])) > 5
```

Example alert (metric-based):

```promql
rate(http_requests_total{status="500"}[1m]) > 10
```

---

## 🔄 E2E Request Flow

```text
User → Go Fiber Microservice
   → Log (stdout) → Promtail → Loki
   → Trace → OTEL SDK → OTEL Collector → Tempo
   → /metrics → Prometheus or OTEL → Mimir
                          ↘︎ Grafana
                        Dashboards + Alerts
```

---

## 🧩 TL;DR Responsibility Matrix

| Role      | Task                   | Tools / Details                |
| --------- | ---------------------- | ------------------------------ |
| Developer | Add structured logging | Logrus JSON stdout             |
| Developer | Expose metrics         | `/metrics` or OTEL SDK         |
| Developer | Instrument tracing     | OTEL SDK + Go Fiber middleware |
| DevOps    | Deploy Loki & Promtail | DaemonSet, Pod log scraping    |
| DevOps    | Deploy Tempo, Mimir    | StatefulSets + services        |
| DevOps    | Deploy OTEL Collector  | Routes logs/metrics/traces     |
| DevOps    | Setup Grafana          | Dashboards and alerts          |
| DevOps    | Configure alerting     | LogQL / PromQL / TraceQL       |
