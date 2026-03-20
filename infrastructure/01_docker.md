# Docker: Контейнеризация

---

## Лекция

### 1. Что такое Docker и зачем нужен

Docker — платформа для создания, доставки и запуска приложений в контейнерах.

**Проблема без Docker:**
- "У меня работает!" — разные версии зависимостей на dev/staging/prod
- Сложная настройка окружения
- Конфликты зависимостей между приложениями на одном сервере

**Контейнер vs Виртуальная машина:**
```
Виртуальная машина:          Контейнер:
┌─────────────────┐          ┌──────┐ ┌──────┐ ┌──────┐
│   App + Libs    │          │ App  │ │ App  │ │ App  │
│ Guest OS (3GB!) │          │Libs  │ │Libs  │ │Libs  │
│   Hypervisor    │          └──────┘ └──────┘ └──────┘
│    Host OS      │          ┌─────────────────────────┐
└─────────────────┘          │    Container Runtime     │
                             │        Host OS           │
Overhead: высокий            └─────────────────────────┘
Изоляция: полная             Overhead: минимальный
Старт: минуты                Изоляция: процессная
                             Старт: секунды
```

---

### 2. Как работают контейнеры изнутри

Контейнер — это обычный Linux процесс с двумя ключевыми механизмами ядра:

**Namespaces** — изоляция ресурсов:
- `pid` namespace — процесс видит только свои процессы (PID 1 в контейнере)
- `net` namespace — свой сетевой стек, IP адрес
- `mnt` namespace — своя файловая система
- `uts` namespace — свой hostname
- `user` namespace — изоляция uid/gid

**cgroups (Control Groups)** — ограничение ресурсов:
- CPU: `--cpus="0.5"` — максимум 50% одного ядра
- Memory: `--memory="512m"` — максимум 512MB RAM
- При OOM (Out of Memory) — процесс убивается (OOM Killer)

**Union File System (OverlayFS):**
- Образ состоит из слоёв (layers)
- Каждый слой — read-only
- При запуске контейнера добавляется writable слой сверху (Copy-on-Write)
- Изменения в контейнере — только в верхнем слое
- Удаление контейнера = удаление верхнего слоя

---

### 3. Docker Image и Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

# Многоступенчатая сборка (multi-stage build)
# Этап 1: сборка
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Сначала копируем go.mod/go.sum для кеширования зависимостей
COPY go.mod go.sum ./
RUN go mod download

# Затем копируем код
COPY . .

# Сборка статического бинаря
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/server

# Этап 2: минимальный production образ
FROM scratch
# или FROM gcr.io/distroless/static для CA certificates

WORKDIR /app

# Копируем только бинарь из builder этапа
COPY --from=builder /app/server .
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Не запускать от root
USER 65534:65534

EXPOSE 8080

ENTRYPOINT ["./server"]
```

**Лучшие практики Dockerfile:**
- Многоступенчатая сборка — итоговый образ только с бинарём (5-20MB vs 1GB)
- Кешировать зависимости отдельным слоем (go mod download перед COPY . .)
- Не запускать от root (`USER`)
- Использовать конкретные теги образов (`golang:1.22`, не `golang:latest`)
- `.dockerignore` — исключить ненужные файлы

---

### 4. Docker Networking

**Типы сетей:**

```bash
# Bridge (по умолчанию) — NAT между контейнерами
docker network create my-net
docker run --network my-net --name api my-api
docker run --network my-net --name db postgres
# api может обращаться к db по имени: postgres://db:5432/mydb

# Host — контейнер использует сеть хоста напрямую (нет NAT)
docker run --network host nginx

# None — нет сети
docker run --network none alpine
```

**Port mapping:**
```bash
# Хост:8080 → контейнер:80
docker run -p 8080:80 nginx

# Только localhost
docker run -p 127.0.0.1:8080:80 nginx
```

---

### 5. Docker Volumes

```bash
# Named volume (управляется Docker)
docker run -v postgres-data:/var/lib/postgresql/data postgres

# Bind mount (папка с хоста)
docker run -v $(pwd)/config:/app/config:ro myapp

# tmpfs (только память, не сохраняется)
docker run --tmpfs /tmp myapp
```

**Когда что использовать:**
- Named volume — для данных БД, персистентных данных
- Bind mount — для разработки (live reload кода), конфигов
- tmpfs — для временных данных, которые не должны попасть на диск

---

### 6. Docker Compose

```yaml
# docker-compose.yml
version: '3.9'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data

volumes:
  postgres-data:
  redis-data:
```

---

## Практические примеры

### Пример 1: Оптимальный Dockerfile для Go сервиса

```dockerfile
FROM golang:1.22-alpine AS builder
RUN apk add --no-cache git ca-certificates tzdata

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-w -s" \
    -o server ./cmd/server

FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /build/server /server

EXPOSE 8080
ENTRYPOINT ["/server"]
```

### Пример 2: Health check в Go сервисе

```go
// Эндпоинт /healthz для Docker health check
func (s *Server) healthHandler(w http.ResponseWriter, r *http.Request) {
    // Проверяем соединение с БД
    if err := s.db.PingContext(r.Context()); err != nil {
        http.Error(w, "db unavailable", http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("ok"))
}
```

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/server", "-healthcheck"] || exit 1
```

### Пример 3: docker-compose для локальной разработки

```yaml
version: '3.9'

services:
  app:
    build: .
    volumes:
      - .:/app                    # bind mount для live reload
    ports:
      - "8080:8080"
      - "2345:2345"               # delve debugger port
    environment:
      - GO_ENV=development
    command: air                  # air — live reload для Go

  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"               # открываем для IDE
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - ./migrations:/docker-entrypoint-initdb.d  # авто-миграции
```

---

## Вопросы на собеседовании

### Q: Чем контейнер отличается от виртуальной машины?

**Ответ:** ВМ: полная гостевая ОС, гипервизор, изоляция на уровне железа. Старт минуты, размер гигабайты, высокий overhead. Контейнер: изолированный процесс ОС хоста через namespaces (изоляция ресурсов) и cgroups (ограничение ресурсов). Старт секунды, размер мегабайты, минимальный overhead. Изоляция слабее чем у ВМ — все контейнеры разделяют ядро хоста.

---

### Q: Как устроена файловая система Docker образа?

**Ответ:** Образ состоит из read-only слоёв (layers), каждый слой — результат одной инструкции Dockerfile. При запуске контейнера поверх добавляется тонкий writable слой (Copy-on-Write). При изменении файла из нижних слоёв — файл копируется в верхний слой и изменяется там. Удаление контейнера = удаление верхнего слоя. Этот механизм (OverlayFS) позволяет эффективно разделять слои между контейнерами.

---

### Q: Что такое multi-stage build? Зачем нужен?

**Ответ:** Multi-stage: несколько `FROM` в одном Dockerfile. В builder этапе собираем приложение (компилятор, зависимости — сотни МБ). В финальный образ копируем только скомпилированный бинарь (5-20MB). Итоговый образ минимален — меньше поверхность атаки, быстрее pull. Для Go: `FROM golang:1.22 AS builder` → компилируем → `FROM scratch` → копируем бинарь.

---

### Q: Как работает Docker Networking между контейнерами?

**Ответ:** По умолчанию — bridge сеть: Docker создаёт виртуальный коммутатор, каждый контейнер получает IP из подсети (172.17.0.0/16). Контейнеры в одной user-defined network могут обращаться друг к другу по имени контейнера (DNS resolution). Для доступа снаружи — port mapping (`-p 8080:80`). В Docker Compose все сервисы автоматически в одной сети и могут ходить по именам сервисов.
