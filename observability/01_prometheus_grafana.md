# Prometheus и Grafana: Мониторинг и наблюдаемость

---

## Лекция

### 1. Observability — наблюдаемость системы

Три столпа наблюдаемости (Three Pillars of Observability):

| Столп | Что даёт | Инструменты |
|-------|----------|------------|
| **Metrics** | Числовые показатели во времени | Prometheus, InfluxDB |
| **Logs** | Текстовые события | ELK, Loki |
| **Traces** | Путь запроса через сервисы | Jaeger, Zipkin, Tempo |

**Prometheus + Grafana** — де-факто стандарт для метрик в Kubernetes-экосистеме.

---

### 2. Как работает Prometheus

**Pull model** — Prometheus сам опрашивает (scrape) эндпоинты `/metrics` у сервисов.

```
[Service A] /metrics ←─── Prometheus ───→ [Grafana]
[Service B] /metrics ←───/              ↑
[Service C] /metrics ←───/              └── Alertmanager
[Postgres Exporter] /metrics ←──/
[Node Exporter] /metrics ←──/
```

**Преимущества pull:**
- Легко проверить доступность цели
- Нет необходимости открывать порты в сервисах
- Centralized configuration

**Pushgateway** — для batch jobs, которые завершаются до scrape.

---

### 3. Типы метрик

**Counter** — только растёт (сбрасывается при рестарте):
```
http_requests_total{method="GET", status="200"} 14532
```
Используется для: количество запросов, ошибок, обработанных задач.
Производная: `rate(http_requests_total[5m])` — запросов в секунду.

**Gauge** — произвольное значение, может расти и убывать:
```
go_goroutines 42
memory_usage_bytes 1048576
queue_depth 15
```
Используется для: текущие значения (CPU, память, горутины, очередь).

**Histogram** — распределение значений по bucket'ам:
```
http_request_duration_seconds_bucket{le="0.1"} 8000
http_request_duration_seconds_bucket{le="0.5"} 9500
http_request_duration_seconds_bucket{le="1.0"} 9900
http_request_duration_seconds_bucket{le="+Inf"} 10000
http_request_duration_seconds_sum 1234.56
http_request_duration_seconds_count 10000
```
Используется для: latency, request size. Позволяет считать перцентили.

**Summary** — перцентили на стороне клиента (устарел, предпочитай Histogram):
```
rpc_duration_seconds{quantile="0.5"} 0.012
rpc_duration_seconds{quantile="0.95"} 0.085
```

---

### 4. PromQL — язык запросов

```promql
# Базовые запросы
http_requests_total                           # все временные ряды
http_requests_total{status="500"}             # фильтр по label
http_requests_total{status=~"5.."}            # regex фильтр (5xx ошибки)

# Функции для Counter
rate(http_requests_total[5m])                 # скорость изменения за 5 минут
irate(http_requests_total[1m])                # instant rate (последние два point)
increase(http_requests_total[1h])             # абсолютное увеличение за 1 час

# Агрегации
sum(rate(http_requests_total[5m]))                           # суммарный RPS
sum by (service) (rate(http_requests_total[5m]))             # RPS по сервисам
avg without (instance) (rate(http_requests_total[5m]))       # среднее, исключая label

# Перцентили из Histogram
histogram_quantile(0.95,
    rate(http_request_duration_seconds_bucket[5m])
)  # p95 latency за 5 минут

histogram_quantile(0.99,
    sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)  # p99 latency (суммарно по всем instance)

# Error rate
rate(http_requests_total{status=~"5.."}[5m])
/ rate(http_requests_total[5m])  # доля ошибок

# Использование ресурсов
container_memory_usage_bytes / container_spec_memory_limit_bytes * 100  # % памяти
```

---

### 5. Алёрты (Alerting Rules)

```yaml
# prometheus/rules/app.yml
groups:
- name: application
  rules:
  # Высокий error rate
  - alert: HighErrorRate
    expr: |
      rate(http_requests_total{status=~"5.."}[5m])
      / rate(http_requests_total[5m]) > 0.05
    for: 2m          # должно сохраняться 2 минуты
    labels:
      severity: critical
    annotations:
      summary: "High error rate on {{ $labels.service }}"
      description: "Error rate is {{ $value | humanizePercentage }}"

  # Высокая latency
  - alert: HighLatency
    expr: |
      histogram_quantile(0.95,
        rate(http_request_duration_seconds_bucket[5m])
      ) > 1.0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High p95 latency: {{ $value }}s"

  # Мало реплик
  - alert: LowReplicaCount
    expr: kube_deployment_status_replicas_available < 2
    for: 1m
    labels:
      severity: critical
```

**Alertmanager** — маршрутизация алёртов: email, Slack, PagerDuty, telegram.

```yaml
# alertmanager.yml
route:
  receiver: slack-critical
  routes:
  - match:
      severity: critical
    receiver: pagerduty
  - match:
      severity: warning
    receiver: slack-warning

receivers:
- name: slack-critical
  slack_configs:
  - api_url: '...'
    channel: '#alerts-critical'
    title: 'CRITICAL: {{ .GroupLabels.alertname }}'
```

---

### 6. Grafana Dashboards

**Ключевые панели для Go сервиса:**
- RPS (requests per second): `rate(http_requests_total[5m])`
- Error rate: `rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])`
- Latency p50/p95/p99: `histogram_quantile(0.99, ...)`
- Goroutines: `go_goroutines`
- Memory: `go_memstats_heap_inuse_bytes`
- GC pause: `rate(go_gc_duration_seconds_sum[5m])`

**USE метод (для инфраструктуры):**
- Utilization (загрузка)
- Saturation (насыщение)
- Errors (ошибки)

**RED метод (для сервисов):**
- Rate (RPS)
- Errors (error rate)
- Duration (latency)

---

### 7. SLI, SLO, SLA

**SLI (Service Level Indicator)** — метрика качества сервиса:
- Availability: `uptime / total_time`
- Latency: `requests_under_100ms / total_requests`
- Error rate: `successful_requests / total_requests`

**SLO (Service Level Objective)** — целевое значение SLI:
- "99.9% запросов успешны" (три девятки = 8.7 часов даунтайма/год)
- "95% запросов выполняются менее чем за 200мс"

**SLA (Service Level Agreement)** — юридическое обязательство перед клиентами с штрафами.

**Error Budget** — допустимое количество ошибок:
- SLO = 99.9% → Error Budget = 0.1% = 43.8 минут/месяц
- Если Error Budget исчерпан → фриз новых фич, фокус на надёжность

---

## Практические примеры

### Пример 1: Инструментация Go сервиса

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "path", "status"},
    )

    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets, // .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10
        },
        []string{"method", "path"},
    )

    activeConnections = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "http_active_connections",
            Help: "Currently active HTTP connections",
        },
    )

    queueDepth = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "queue_depth",
            Help: "Number of items in processing queue",
        },
        []string{"queue_name"},
    )
)

// Middleware для HTTP
func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        activeConnections.Inc()
        defer activeConnections.Dec()

        rw := &statusRecorder{ResponseWriter: w, status: 200}
        next.ServeHTTP(rw, r)

        duration := time.Since(start).Seconds()
        status := strconv.Itoa(rw.status)

        httpRequestsTotal.WithLabelValues(r.Method, r.URL.Path, status).Inc()
        httpRequestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
    })
}

type statusRecorder struct {
    http.ResponseWriter
    status int
}

func (r *statusRecorder) WriteHeader(status int) {
    r.status = status
    r.ResponseWriter.WriteHeader(status)
}

func main() {
    // Эндпоинт для Prometheus scrape
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":9090", nil)
}
```

### Пример 2: Бизнес-метрики

```go
var (
    ordersCreated = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "orders_created_total",
            Help: "Total orders created",
        },
        []string{"status", "payment_method"},
    )

    orderAmount = promauto.NewHistogram(
        prometheus.HistogramOpts{
            Name:    "order_amount_rub",
            Help:    "Order amount distribution in rubles",
            Buckets: []float64{100, 500, 1000, 5000, 10000, 50000},
        },
    )

    kafkaLag = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "kafka_consumer_lag",
            Help: "Kafka consumer group lag",
        },
        []string{"topic", "partition"},
    )
)

func (s *OrderService) CreateOrder(ctx context.Context, req *CreateOrderRequest) (*Order, error) {
    order, err := s.repo.Create(ctx, req)
    if err != nil {
        ordersCreated.WithLabelValues("failed", req.PaymentMethod).Inc()
        return nil, err
    }

    ordersCreated.WithLabelValues("success", req.PaymentMethod).Inc()
    orderAmount.Observe(order.Amount)
    return order, nil
}
```

---

## Вопросы на собеседовании

### Q: Чем Histogram отличается от Summary? Какой предпочесть?

**Ответ:** Summary вычисляет перцентили на стороне клиента (в приложении). Нельзя агрегировать по нескольким инстанциям. Histogram хранит распределение по bucket'ам, перцентили вычисляются в Prometheus через `histogram_quantile()`. Можно суммировать bucket'ы по нескольким инстанциям и считать агрегированные перцентили. Почти всегда используй Histogram. Summary только если нужна очень точная percentile accuracy и одна инстанция.

---

### Q: Что такое rate() и irate()? В чём разница?

**Ответ:** `rate(counter[5m])` — среднее значение скорости изменения counter за 5 минут. Сглаживает кратковременные пики. `irate(counter[1m])` — instant rate, считается по последним двум точкам в window. Более чувствителен к резким изменениям. Для алертов — `rate()` (устойчивее). Для дашбордов — `irate()` если нужна более детальная картина. Оба работают только с Counter.

---

### Q: Что такое SLO и Error Budget?

**Ответ:** SLO (Service Level Objective) — целевой показатель надёжности сервиса, например 99.9% успешных запросов. Error Budget — допустимое количество ошибок: 100% - 99.9% = 0.1% = 43 минуты даунтайма в месяц. Если Error Budget исчерпан раньше срока — замораживаем деплои и фичи, фокусируемся на надёжности. Если Error Budget ещё большой — можно деплоить смелее. Это инструмент баланса между скоростью разработки и надёжностью.

---

### Q: Чем отличается pull-модель Prometheus от push-модели?

**Ответ:** Pull: Prometheus сам опрашивает сервисы по расписанию. Просто конфигурировать, легко обнаруживать проблемы (если endpoint недоступен — сразу видно). Минус: сервисы должны быть доступны Prometheus'у. Push: сервисы сами отправляют метрики в aggregator. Хорошо для batch jobs, serverless, short-lived процессов. Минус: сложнее обнаружить "тихую смерть" сервиса. Prometheus использует pull, но предоставляет Pushgateway для batch jobs.

---

### Q: Что такое Observability? Чем отличается от мониторинга?

**Ответ:** Мониторинг — заранее задаёшь что проверять (алёрты, дашборды). Работает для известных проблем. Observability — способность системы отвечать на произвольные вопросы о своём внутреннем состоянии без изменения кода. Три столпа: Metrics (агрегированные числа), Logs (детальные события), Traces (путь запроса). Для observability нужно хорошо инструментировать систему заранее, но тогда можно исследовать любые инциденты постфактум.
