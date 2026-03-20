# Kubernetes: Оркестрация контейнеров

---

## Лекция

### 1. Зачем нужен Kubernetes

Docker решает упаковку приложения. Kubernetes решает оркестрацию — управление множеством контейнеров в production:

- Автоматический перезапуск упавших контейнеров
- Масштабирование (scaling) под нагрузку
- Rolling updates без даунтайма
- Service discovery и load balancing
- Управление конфигами и секретами
- Распределение нагрузки по нодам

---

### 2. Архитектура Kubernetes

```
Control Plane (мастер):                Worker Nodes:
┌────────────────────────┐            ┌──────────────────┐
│  kube-apiserver        │  ←──────→  │  kubelet         │
│  etcd (хранилище)      │            │  kube-proxy      │
│  kube-scheduler        │            │  container runtime│
│  kube-controller-manager│           │  (containerd)    │
└────────────────────────┘            │                  │
                                      │  Pod Pod Pod      │
                                      └──────────────────┘
```

**kube-apiserver** — единая точка входа для всего. Все компоненты общаются через API.

**etcd** — распределённое хранилище ключ-значение. Хранит всё состояние кластера.

**kube-scheduler** — решает, на какую ноду поместить Pod.

**kube-controller-manager** — набор контроллеров: ReplicaSet controller, Deployment controller, etc. Обеспечивают desired state.

**kubelet** — агент на каждой ноде. Запускает контейнеры, проверяет health.

**kube-proxy** — управляет сетевыми правилами (iptables/ipvs) для Services.

---

### 3. Основные объекты

**Pod** — наименьшая единица деплоя. Один или несколько контейнеров с общей сетью и хранилищем.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: my-app:1.0
    ports:
    - containerPort: 8080
    resources:
      requests:              # минимально гарантировано
        memory: "64Mi"
        cpu: "100m"          # 100 millicores = 0.1 CPU
      limits:                # максимально допустимо
        memory: "128Mi"
        cpu: "500m"
```

**Deployment** — управляет набором Pod. Обеспечивает desired state.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # max 1 pod может быть недоступен
      maxSurge: 1          # max 1 pod сверх replicas во время update
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:1.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Service** — стабильный DNS/IP для доступа к Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP      # только внутри кластера
  # type: NodePort     — доступен через порт каждой ноды
  # type: LoadBalancer — внешний балансировщик (cloud)
```

---

### 4. Liveness и Readiness Probes

```yaml
livenessProbe:     # "жив ли Pod?" → если нет: перезапустить
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15  # подождать перед первой проверкой
  periodSeconds: 20
  failureThreshold: 3      # 3 неудачи → restart

readinessProbe:    # "готов ли Pod принимать трафик?" → если нет: убрать из балансировки
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

**Разница:** Liveness — перезапустить если завис. Readiness — не слать трафик если не готов (например, грузит кеш при старте).

**startup Probe** — для медленно стартующих приложений. Пока startupProbe не прошла — liveness не проверяется.

---

### 5. ConfigMap и Secret

```yaml
# ConfigMap — обычные конфигурации
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  DATABASE_HOST: "postgres-service"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s

# Secret — чувствительные данные (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DATABASE_PASSWORD: cGFzc3dvcmQ=  # base64("password")
  JWT_SECRET: c2VjcmV0a2V5         # base64("secretkey")
```

```yaml
# Использование в Pod
spec:
  containers:
  - name: app
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secrets
    volumeMounts:
    - name: config-volume
      mountPath: /app/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

---

### 6. Ingress — внешний HTTP доступ

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

---

### 7. HPA — автоскейлинг

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70   # масштабировать при CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

### 8. Паттерны деплоя

**Rolling Update** (по умолчанию) — постепенно заменяет старые поды новыми. Нет даунтайма, но старая и новая версии работают одновременно → API должен быть обратно совместим.

**Blue-Green** — два идентичных окружения (blue = текущее, green = новое). Переключение трафика мгновенное. Требует 2x ресурсов.

**Canary** — небольшой % трафика идёт на новую версию. Постепенно увеличиваем. Позволяет обнаружить проблемы на малом % пользователей.

---

## Практические примеры

### Пример 1: Полный манифест Go сервиса

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: order-service
        image: registry.example.com/order-service:v1.2.3
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: order-service-secrets
              key: database-url
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /healthz
            port: http
          initialDelaySeconds: 10
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"] # дать время балансировщику убрать pod
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  selector:
    app: order-service
  ports:
  - name: http
    port: 80
    targetPort: http
```

### Пример 2: Graceful shutdown в Go

```go
func main() {
    server := &http.Server{Addr: ":8080", Handler: router}

    // Канал для сигналов от ОС
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)

    go func() {
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    // Ждём SIGTERM (Kubernetes посылает при завершении Pod)
    <-quit
    log.Println("shutting down...")

    // Graceful shutdown: завершить текущие запросы, max 25 секунд
    ctx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Printf("shutdown error: %v", err)
    }
    log.Println("server stopped")
}
```

---

## Вопросы на собеседовании

### Q: Что такое Pod? Почему не просто контейнер?

**Ответ:** Pod — минимальная единица деплоя в Kubernetes. Может содержать несколько контейнеров, которые: разделяют одну сеть (localhost между собой), разделяют volumes, запускаются и умирают вместе. Паттерны multi-container: sidecar (логи, прокси, мониторинг рядом с основным), init containers (подготовка перед стартом основного). Pod — эфемерен: Kubernetes свободно его убивает и создаёт заново.

---

### Q: Чем Deployment отличается от StatefulSet?

**Ответ:** Deployment: поды идентичны и взаимозаменяемы, случайные имена (pod-abc123), storage не привязан к конкретному поду. StatefulSet: каждый под уникален, стабильные имена (pod-0, pod-1, pod-2), каждый под получает свой PersistentVolume, поды запускаются/удаляются в порядке. StatefulSet для: баз данных (PostgreSQL, MongoDB), Kafka, любых stateful приложений где важна идентичность.

---

### Q: В чём разница между liveness и readiness probe?

**Ответ:** Liveness probe — "жив ли контейнер?". Если нет — kubelet перезапускает контейнер. Защита от deadlock и зависших процессов. Readiness probe — "готов ли принимать трафик?". Если нет — Pod убирается из балансировки Service (EndpointSlice), но не перезапускается. Используй readiness при прогреве кеша, ожидании зависимостей, временной перегрузке. Оба нужны: liveness = выживаемость, readiness = доступность.

---

### Q: Как Kubernetes обеспечивает zero-downtime деплой?

**Ответ:** Rolling Update: новые поды поднимаются → readiness probe проходит → старые поды удаляются. Параметры `maxUnavailable=0, maxSurge=1` — никогда не уменьшать число работающих подов. Также важно: 1) readiness probe отвечает только когда приложение готово, 2) PodDisruptionBudget — гарантирует минимум доступных подов, 3) terminationGracePeriodSeconds + preStop hook — дать время балансировщику убрать pod до закрытия соединений.

---

### Q: Что такое requests и limits в Kubernetes?

**Ответ:** Resources requests — минимальное количество ресурсов, которое scheduler гарантирует поду при размещении на ноду. Scheduler не поместит Pod на ноду, если там не хватает ресурсов для requests. Resources limits — максимальный потолок. При превышении CPU limit — throttling (замедление). При превышении memory limit — OOM Kill (контейнер убивается). Requests должны быть реалистичными (для правильного scheduling), limits — с запасом (защита от runaway процессов).
