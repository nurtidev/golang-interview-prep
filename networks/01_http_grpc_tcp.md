# Сети и протоколы: HTTP, gRPC, TCP

## TCP

### Как устроен TCP

TCP (Transmission Control Protocol) — надёжный протокол передачи с установкой соединения.

**Ключевые свойства:**
- Надёжность — гарантированная доставка и порядок пакетов
- Управление потоком — не перегружает получателя
- Управление перегрузкой — адаптируется к пропускной способности сети

### TCP Handshake (три шага)

```
Client                    Server
  |                          |
  |------- SYN ----------->  |   seq=x
  |  <------ SYN-ACK ------  |   seq=y, ack=x+1
  |------- ACK ----------->  |   ack=y+1
  |                          |
  |      CONNECTED           |
```

### TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Соединение | Да (handshake) | Нет |
| Надёжность | Гарантирована | Нет гарантий |
| Порядок | Гарантирован | Нет |
| Скорость | Медленнее | Быстрее |
| Overhead | Больше | Меньше |
| Использование | HTTP, gRPC, БД | DNS, видео, игры |

---

## HTTP

### Структура HTTP запроса

```
POST /api/users HTTP/1.1          ← Метод + Path + Версия
Host: api.example.com             ← Заголовки
Content-Type: application/json
Authorization: Bearer <token>
                                  ← Пустая строка
{"name": "Nurtilek", "age": 25}   ← Тело
```

### HTTP методы и их семантика

```
GET    — получить ресурс (idempotent, safe)
POST   — создать ресурс
PUT    — заменить ресурс целиком (idempotent)
PATCH  — частичное обновление
DELETE — удалить (idempotent)
HEAD   — как GET, но без тела ответа
```

### HTTP/1.1 vs HTTP/2

| | HTTP/1.1 | HTTP/2 |
|---|---|---|
| Протокол | text | binary |
| Мультиплексирование | нет (1 запрос/соединение) | да (много запросов/соединение) |
| Сжатие заголовков | нет | HPACK |
| Server Push | нет | да |
| TLS | опционально | практически обязательно |

### HTTP vs HTTPS

HTTPS = HTTP + TLS. TLS обеспечивает:
- **Шифрование** — данные нечитаемы при перехвате
- **Аутентификацию** — сертификат подтверждает личность сервера
- **Целостность** — данные не изменены в пути

TLS Handshake добавляет ~1 RTT (или 0-RTT с TLS 1.3).

### HTTP в Go

```go
// Клиент с таймаутами — ВСЕГДА устанавливай!
client := &http.Client{
    Timeout: 30 * time.Second,
    Transport: &http.Transport{
        DialContext: (&net.Dialer{
            Timeout:   5 * time.Second,
            KeepAlive: 30 * time.Second,
        }).DialContext,
        TLSHandshakeTimeout:   10 * time.Second,
        ResponseHeaderTimeout: 10 * time.Second,
        MaxIdleConns:          100,
        MaxIdleConnsPerHost:   10,
        IdleConnTimeout:       90 * time.Second,
    },
}

// Запрос с context
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := client.Do(req)
defer resp.Body.Close()
body, err := io.ReadAll(resp.Body)

// Сервер
mux := http.NewServeMux()
mux.HandleFunc("/api/users", handleUsers)

server := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  120 * time.Second,
}
```

### Middleware паттерн

```go
type Middleware func(http.Handler) http.Handler

func Logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

func Auth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if !validateToken(token) {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// Цепочка: запрос → Logger → Auth → Handler
handler := Logger(Auth(mux))
```

---

## gRPC

### Что такое gRPC

gRPC = Google Remote Procedure Call. Работает поверх HTTP/2, использует Protocol Buffers для сериализации.

**Преимущества перед REST:**
- Сильная типизация через .proto
- Эффективная бинарная сериализация
- HTTP/2 мультиплексирование
- Кодогенерация клиента/сервера
- Поддержка стриминга

### Типы RPC

```protobuf
service UserService {
    // Unary — один запрос, один ответ
    rpc GetUser(GetUserRequest) returns (User);

    // Server Streaming — один запрос, поток ответов
    rpc ListUsers(ListUsersRequest) returns (stream User);

    // Client Streaming — поток запросов, один ответ
    rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);

    // Bidirectional Streaming — поток запросов и ответов
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

### Unary vs Stream

| | Unary | Stream |
|---|---|---|
| Паттерн | req → resp | поток данных |
| Использование | CRUD, запросы | лог, чат, большие данные |
| Таймаут | per-call | per-stream или per-message |

### gRPC в Go

```go
// Сервер
type userServer struct {
    pb.UnimplementedUserServiceServer
}

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    // проверяем контекст
    if err := ctx.Err(); err != nil {
        return nil, status.Error(codes.Canceled, "request canceled")
    }
    user, err := db.GetUser(ctx, req.Id)
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "user %d not found", req.Id)
    }
    return user, nil
}

// Запуск
lis, _ := net.Listen("tcp", ":50051")
grpcServer := grpc.NewServer(
    grpc.UnaryInterceptor(loggingInterceptor),
)
pb.RegisterUserServiceServer(grpcServer, &userServer{})
grpcServer.Serve(lis)

// Клиент
conn, _ := grpc.Dial("localhost:50051",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
    grpc.WithBlock(),
)
defer conn.Close()
client := pb.NewUserServiceClient(conn)

ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 42})
```

---

## DNS

### Что происходит при запросе google.com

```
1. Browser cache → OS cache → /etc/hosts
2. Запрос к Recursive Resolver (обычно провайдер или 8.8.8.8)
3. Resolver → Root nameserver (.) → "спроси .com"
4. Resolver → TLD nameserver (.com) → "спроси google.com NS"
5. Resolver → Authoritative nameserver (google.com) → IP: 142.250.74.46
6. Resolver кэширует TTL, отвечает браузеру
7. Browser → TCP connect → TLS → HTTP GET
```

DNS работает по **UDP** (порт 53), при больших ответах переключается на TCP.

---

---

## TCP Congestion Control (Управление перегрузкой)

### Зачем это нужно

Без управления перегрузкой отправитель заполнит сеть пакетами → роутеры начнут дропать → ACK не придут → отправитель шлёт ещё → коллапс сети. TCP решает это через динамическое управление окном передачи.

**Два окна:**
```
CWND (Congestion Window)  — сколько байт можно "в полёте" без ACK, контролируется отправителем
RWND (Receive Window)     — сколько буфера у получателя (из TCP заголовка)

Реальный лимит = min(CWND, RWND)
```

**"В полёте" (in-flight)** — пакеты отправлены, но ACK ещё не получен.

---

### Фазы TCP Congestion Control

#### 1. Slow Start (Медленный старт)

```
CWND начинается с 1 MSS (обычно 1460 байт, или 10 MSS в Linux с RFC 6928)

При каждом ACK: CWND += 1 MSS → экспоненциальный рост

CWND: 1 → 2 → 4 → 8 → 16 → ... → ssthresh
```

```
Время:  RTT1  RTT2  RTT3  RTT4
CWND:     1     2     4     8    ← удваивается каждый RTT
```

Рост продолжается до достижения `ssthresh` (slow start threshold).

#### 2. Congestion Avoidance (Предотвращение перегрузки)

После достижения ssthresh — линейный рост:
```
При каждом ACK: CWND += MSS * (MSS / CWND)  ≈ +1 MSS за RTT (аддитивный рост)
```

```
CWND:  16 → 17 → 18 → 19 → ...  ← +1 MSS каждый RTT
```

**AIMD (Additive Increase, Multiplicative Decrease)** — ключевой принцип:
- Хорошо: +1 MSS/RTT
- Потеря пакета: CWND / 2 (мультипликативное уменьшение)

#### 3. Обнаружение потери (два механизма)

**Triple Duplicate ACK (Fast Retransmit):**
```
Отправитель получил 3 одинаковых ACK →
"Пакет потерян, но сеть ещё работает" →
Fast Retransmit: отправить потерянный сразу →
ssthresh = CWND / 2
CWND = ssthresh  (остаёмся в Congestion Avoidance — Fast Recovery)
```

**RTO Timeout (Retransmission Timeout):**
```
ACK не пришёл за RTO (обычно 200ms-1s) →
"Серьёзная потеря или недоступность сети" →
CWND = 1 MSS  ← сбрасываем полностью!
ssthresh = max(CWND/2, 2*MSS)
Начинаем Slow Start заново
```

RTO timeout гораздо разрушительнее для throughput чем triple duplicate ACK.

---

### TCP CUBIC (Linux default с 2.6.19)

Стандартный AIMD работал плохо на каналах с большим BDP (Bandwidth-Delay Product): трансатлантический канал 10 Гбит/с × 150ms RTT → BDP = 1.5 Гбит. После потери слишком долго "разгоняться" обратно.

**CUBIC идея:** форма восстановления CWND — кубическая функция времени, не зависящая от RTT:

```
W(t) = C * (t - K)³ + Wmax

Wmax — CWND в момент последней потери
t    — время с последней потери
K    — точка где W(t) = Wmax (быстрое восстановление до предыдущего пика)
C    — scaling factor (0.4 по умолчанию)
```

```
CWND
 │              ╭──────── (новые пики)
 │         ╭───╯
Wmax──────╯ ← здесь была потеря
 │    ╰──── быстрый рост → замедление у Wmax → медленный рост дальше
 └─────────────────────── время
       ↑
    потеря
```

**Почему лучше:** RTT-независимость = fairness на каналах с разными задержками. Быстрое восстановление до Wmax (cubic вогнутая часть). Осторожный рост после Wmax (cubic выпуклая часть).

---

### TCP BBR (Google, 2016) — другая парадигма

CUBIC и классические алгоритмы реагируют на **потери** (loss-based). Проблема: буферы роутеров большие (bufferbloat) → пакеты не дропаются, но latency растёт до сотен ms.

**BBR (Bottleneck Bandwidth and RTT)** моделирует физические свойства канала:

```
Две метрики:
  BtlBw  — bottleneck bandwidth (максимальная пропускная способность)
  RTprop — round-trip propagation delay (минимальный RTT = RTT без очередей)

Оптимальная скорость = BtlBw × RTprop (BDP)
```

**Фазы BBR:**
```
STARTUP      — экспоненциальный рост пока BtlBw растёт (как Slow Start)
DRAIN        — уменьшить CWND, слить очередь в роутере
PROBE_BW     — основной режим: пробуем увеличить скорость каждые 8 RTT
PROBE_RTT    — раз в 10с снижаем нагрузку чтобы измерить чистый RTprop
```

**Сравнение CUBIC vs BBR:**

| | CUBIC | BBR |
|---|---|---|
| Триггер | Потери пакетов | Модель сети (BtlBw + RTprop) |
| Latency | Высокая при bufferbloat | Низкая |
| Throughput на дальних каналах | Хуже | Лучше (до 2700x в тестах Google) |
| CPU overhead | Низкий | Чуть выше |
| Fairness с другими потоками | Хорошая | Проблемы с CUBIC-потоками |
| Использование | Linux default | YouTube, Google CDN, Cloudflare |

```bash
# Посмотреть текущий алгоритм:
sysctl net.ipv4.tcp_congestion_control
# tcp_cubic

# Включить BBR (Linux 4.9+):
sysctl -w net.ipv4.tcp_congestion_control=bbr
sysctl -w net.core.default_qdisc=fq  # нужен для BBR
```

---

### BDP — Bandwidth-Delay Product

Ключевое понятие для high-throughput систем:

```
BDP = Bandwidth × RTT

Пример: 1 Гбит/с канал, RTT = 100ms
BDP = 1,000,000,000 бит/с × 0.1с = 100,000,000 бит = 12.5 MB

→ Чтобы полностью загрузить канал, нужно держать 12.5 MB "в полёте"
→ CWND должен достичь 12.5 MB
→ При потере и откате к 1 MSS — нужны десятки секунд для восстановления!
```

**Почему это важно для backend разработчика:**
- Междатацентровый трафик (300ms RTT): CWND нужен ~37 MB для 1 Гбит/с
- TCP SO_SNDBUF / SO_RCVBUF должны быть достаточно большими
- BBR справляется лучше CUBIC на high-BDP каналах

---

### Практика в Go: TCP socket tuning

```go
import (
    "net"
    "syscall"
)

// Настройка TCP keepalive (обнаружение мёртвых соединений)
dialer := &net.Dialer{
    KeepAlive: 30 * time.Second,  // Go устанавливает SO_KEEPALIVE
    Timeout:   5 * time.Second,
}

// Низкоуровневая настройка через Control
dialer.Control = func(network, address string, c syscall.RawConn) error {
    return c.Control(func(fd uintptr) {
        // Увеличить буфер отправки (помогает на high-BDP каналах)
        syscall.SetsockoptInt(int(fd), syscall.SOL_SOCKET, syscall.SO_SNDBUF, 4*1024*1024)
        // Увеличить буфер приёма
        syscall.SetsockoptInt(int(fd), syscall.SOL_SOCKET, syscall.SO_RCVBUF, 4*1024*1024)
        // TCP_NODELAY — отключить алгоритм Nagle (важно для низкой latency)
        syscall.SetsockoptInt(int(fd), syscall.IPPROTO_TCP, syscall.TCP_NODELAY, 1)
    })
}

conn, err := dialer.DialContext(ctx, "tcp", "remote:8080")
```

**TCP_NODELAY vs Nagle:**
```
Алгоритм Nagle (по умолчанию): буферизует маленькие пакеты, ждёт ACK или накопит MSS
→ Снижает количество пакетов (хорошо для throughput)
→ Добавляет latency 1-200ms для маленьких сообщений (плохо для rpc/чат)

TCP_NODELAY = 1: отправлять сразу, без буферизации
→ Нужен для gRPC, Redis, any RPC protocol
→ Go net/http и grpc устанавливают TCP_NODELAY автоматически
```

---

### TLS Handshake детально

```
TLS 1.3 (2018):

Client                              Server
  |                                   |
  |──── ClientHello ────────────────►|   (поддерживаемые алгоритмы, key_share)
  |◄─── ServerHello ─────────────────|   (выбранный алгоритм, key_share)
  |◄─── {Certificate} ───────────────|   (зашифровано уже!)
  |◄─── {CertificateVerify} ─────────|
  |◄─── {Finished} ──────────────────|
  |──── {Finished} ────────────────►|
  |                                   |
  |════ Application Data ════════════|   (1 RTT всего!)

TLS 1.2 (старый): 2 RTT
TLS 1.3 с 0-RTT (session resumption): 0 RTT для повторных соединений
  (но replay attack возможен — использовать только для idempotent запросов!)
```

**Алгоритмы в TLS 1.3:**
```
Key Exchange:  X25519 (ECDH на Curve25519) — быстрый, безопасный
               или FFDHE (Finite Field DH)

Authentication: Ed25519, RSA-PSS, ECDSA

AEAD Cipher:   AES-128-GCM, AES-256-GCM, ChaCha20-Poly1305
               (ChaCha20 быстрее без AES-NI — мобильные устройства)

Hash: SHA-256, SHA-384
```

**TLS в Go:**
```go
// Настройка TLS клиента
tlsConfig := &tls.Config{
    MinVersion: tls.VersionTLS12,  // минимум TLS 1.2 (лучше 1.3)
    // CipherSuites: nil → Go выбирает безопасные автоматически
    ServerName: "api.example.com",  // SNI (Server Name Indication)

    // Для mTLS (mutual TLS):
    Certificates: []tls.Certificate{clientCert},
    RootCAs:      certPool,
}

transport := &http.Transport{
    TLSClientConfig: tlsConfig,
    // TLS сессии кэшируются автоматически в Go → 0-RTT resumption
}
```

**SNI (Server Name Indication):** расширение TLS, позволяет серверу держать несколько сертификатов на одном IP (как virtual hosting для HTTPS). Клиент указывает имя хоста в ClientHello → сервер выбирает нужный сертификат.

---

## Типичные вопросы на собесе

**Q: Зачем нужны таймауты и как их подобрать?**

Без таймаутов — зависшие соединения занимают goroutine и память. Подбор: p99 latency * 2-3 для read timeout; connect timeout — обычно 5с.

**Q: Чем gRPC stream отличается от unary?**

Unary — один запрос/ответ, закрытое соединение. Stream — открытое соединение HTTP/2 для передачи нескольких сообщений. Server stream — когда сервер отдаёт много данных (события, логи). Bidirectional — чат, real-time.

**Q: Что происходит при HTTP keep-alive?**

TCP соединение не закрывается после запроса — переиспользуется для следующих. Экономит время на TCP handshake.
