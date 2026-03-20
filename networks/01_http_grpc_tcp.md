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

## Типичные вопросы на собесе

**Q: Зачем нужны таймауты и как их подобрать?**

Без таймаутов — зависшие соединения занимают goroutine и память. Подбор: p99 latency * 2-3 для read timeout; connect timeout — обычно 5с.

**Q: Чем gRPC stream отличается от unary?**

Unary — один запрос/ответ, закрытое соединение. Stream — открытое соединение HTTP/2 для передачи нескольких сообщений. Server stream — когда сервер отдаёт много данных (события, логи). Bidirectional — чат, real-time.

**Q: Что происходит при HTTP keep-alive?**

TCP соединение не закрывается после запроса — переиспользуется для следующих. Экономит время на TCP handshake.
