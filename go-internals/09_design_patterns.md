# Go: Паттерны проектирования

---

## 1. Functional Options — Go-специфичный паттерн

Самый популярный Go паттерн для конфигурации. Позволяет иметь необязательные параметры без breaking changes.

```go
type Server struct {
    host    string
    port    int
    timeout time.Duration
    maxConn int
}

// Тип опции — функция, которая модифицирует Server
type Option func(*Server)

// Конструктор принимает variadic опции
func NewServer(opts ...Option) *Server {
    s := &Server{
        host:    "localhost",  // defaults
        port:    8080,
        timeout: 30 * time.Second,
        maxConn: 100,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Каждая опция — отдельная функция
func WithHost(host string) Option {
    return func(s *Server) { s.host = host }
}

func WithPort(port int) Option {
    return func(s *Server) { s.port = port }
}

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

// Использование — читаемо, расширяемо
s := NewServer(
    WithHost("0.0.0.0"),
    WithPort(9090),
    WithTimeout(5*time.Second),
)
```

**Когда использовать:** конфигурация с >3 опциональными параметрами. Используется в grpc.Dial(), zap.New(), http.Server.

---

## 2. Singleton

```go
// Простой Singleton через sync.Once
type DB struct {
    conn *sql.DB
}

var (
    instance *DB
    once     sync.Once
)

func GetDB() *DB {
    once.Do(func() {
        conn, err := sql.Open("postgres", dsn)
        if err != nil {
            panic(err)
        }
        instance = &DB{conn: conn}
    })
    return instance
}

// Лучше: инжектировать зависимости через конструктор
// Singleton через wire/fx для DI (предпочтительно в продакшне)
```

**Осторожно:** Singleton усложняет тестирование (глобальное состояние). Предпочитай DI (dependency injection).

---

## 3. Factory Method

```go
type Notifier interface {
    Send(ctx context.Context, msg Message) error
}

type EmailNotifier struct{ smtp SMTPClient }
type SMSNotifier struct{ api SMSAPIClient }
type PushNotifier struct{ fcm FCMClient }

func (e *EmailNotifier) Send(ctx context.Context, msg Message) error { /* ... */ }
func (s *SMSNotifier) Send(ctx context.Context, msg Message) error   { /* ... */ }
func (p *PushNotifier) Send(ctx context.Context, msg Message) error  { /* ... */ }

// Фабрика создаёт нужную реализацию
func NewNotifier(notifierType string, cfg Config) (Notifier, error) {
    switch notifierType {
    case "email":
        return &EmailNotifier{smtp: NewSMTPClient(cfg.SMTP)}, nil
    case "sms":
        return &SMSNotifier{api: NewSMSClient(cfg.SMS)}, nil
    case "push":
        return &PushNotifier{fcm: NewFCMClient(cfg.FCM)}, nil
    default:
        return nil, fmt.Errorf("unknown notifier type: %s", notifierType)
    }
}

// Код не знает какая конкретная реализация
notifier, err := NewNotifier(cfg.NotifierType, cfg)
notifier.Send(ctx, msg)
```

---

## 4. Builder

```go
// Для сложных объектов с валидацией при построении
type Query struct {
    table      string
    conditions []string
    orderBy    string
    limit      int
    offset     int
}

type QueryBuilder struct {
    query Query
    errs  []error
}

func NewQueryBuilder(table string) *QueryBuilder {
    return &QueryBuilder{query: Query{table: table, limit: 100}}
}

func (b *QueryBuilder) Where(condition string) *QueryBuilder {
    if condition == "" {
        b.errs = append(b.errs, errors.New("empty condition"))
        return b
    }
    b.query.conditions = append(b.query.conditions, condition)
    return b
}

func (b *QueryBuilder) OrderBy(field string) *QueryBuilder {
    b.query.orderBy = field
    return b
}

func (b *QueryBuilder) Limit(n int) *QueryBuilder {
    if n <= 0 {
        b.errs = append(b.errs, fmt.Errorf("limit must be positive, got %d", n))
        return b
    }
    b.query.limit = n
    return b
}

func (b *QueryBuilder) Build() (*Query, error) {
    if len(b.errs) > 0 {
        return nil, errors.Join(b.errs...)
    }
    return &b.query, nil
}

// Использование — fluent interface
q, err := NewQueryBuilder("users").
    Where("active = true").
    Where("age > 18").
    OrderBy("created_at DESC").
    Limit(50).
    Build()
```

---

## 5. Adapter

```go
// Адаптируем внешний интерфейс к нашему
// Наш интерфейс
type Logger interface {
    Info(msg string, fields ...Field)
    Error(msg string, fields ...Field)
}

// Внешняя библиотека (logrus)
type logrusAdapter struct {
    log *logrus.Logger
}

func NewLogrusAdapter(l *logrus.Logger) Logger {
    return &logrusAdapter{log: l}
}

func (a *logrusAdapter) Info(msg string, fields ...Field) {
    entry := a.log.WithFields(toLogrusFields(fields))
    entry.Info(msg)
}

func (a *logrusAdapter) Error(msg string, fields ...Field) {
    entry := a.log.WithFields(toLogrusFields(fields))
    entry.Error(msg)
}

// Теперь можно менять логгер без изменения кода сервисов
```

---

## 6. Decorator / Middleware

Самый распространённый паттерн в Go — обёртка вокруг интерфейса.

```go
// Базовый интерфейс
type UserRepository interface {
    GetUser(ctx context.Context, id int64) (*User, error)
}

// Декоратор: добавляет кеширование
type cachedUserRepo struct {
    repo  UserRepository
    cache *redis.Client
    ttl   time.Duration
}

func WithCache(repo UserRepository, cache *redis.Client, ttl time.Duration) UserRepository {
    return &cachedUserRepo{repo: repo, cache: cache, ttl: ttl}
}

func (c *cachedUserRepo) GetUser(ctx context.Context, id int64) (*User, error) {
    key := fmt.Sprintf("user:%d", id)

    // Попробуем из кеша
    cached, err := c.cache.Get(ctx, key).Bytes()
    if err == nil {
        var user User
        if err = json.Unmarshal(cached, &user); err == nil {
            return &user, nil
        }
    }

    // Промах — идём в репозиторий
    user, err := c.repo.GetUser(ctx, id)
    if err != nil {
        return nil, err
    }

    // Сохраняем в кеш
    data, _ := json.Marshal(user)
    c.cache.Set(ctx, key, data, c.ttl)
    return user, nil
}

// Декоратор: добавляет метрики
type instrumentedUserRepo struct {
    repo    UserRepository
    metrics *prometheus.CounterVec
    latency *prometheus.HistogramVec
}

func WithMetrics(repo UserRepository, reg prometheus.Registerer) UserRepository {
    // ... регистрация метрик
    return &instrumentedUserRepo{repo: repo}
}

func (r *instrumentedUserRepo) GetUser(ctx context.Context, id int64) (*User, error) {
    start := time.Now()
    user, err := r.repo.GetUser(ctx, id)
    r.latency.WithLabelValues("GetUser").Observe(time.Since(start).Seconds())
    if err != nil {
        r.metrics.WithLabelValues("GetUser", "error").Inc()
    } else {
        r.metrics.WithLabelValues("GetUser", "ok").Inc()
    }
    return user, err
}

// Цепочка декораторов
repo := NewPostgresUserRepo(db)
repo = WithCache(repo, redisClient, 5*time.Minute)
repo = WithMetrics(repo, prometheus.DefaultRegisterer)
// Порядок: Metrics → Cache → Postgres
```

---

## 7. Observer / Pub-Sub через каналы

```go
type EventBus struct {
    mu          sync.RWMutex
    subscribers map[string][]chan Event
}

func NewEventBus() *EventBus {
    return &EventBus{
        subscribers: make(map[string][]chan Event),
    }
}

func (b *EventBus) Subscribe(topic string, bufSize int) <-chan Event {
    ch := make(chan Event, bufSize)
    b.mu.Lock()
    b.subscribers[topic] = append(b.subscribers[topic], ch)
    b.mu.Unlock()
    return ch
}

func (b *EventBus) Publish(topic string, event Event) {
    b.mu.RLock()
    subs := b.subscribers[topic]
    b.mu.RUnlock()

    for _, ch := range subs {
        select {
        case ch <- event:
        default:
            // подписчик не успевает — дроп или лог
        }
    }
}

func (b *EventBus) Unsubscribe(topic string, ch <-chan Event) {
    b.mu.Lock()
    defer b.mu.Unlock()
    subs := b.subscribers[topic]
    for i, sub := range subs {
        if sub == ch {
            b.subscribers[topic] = append(subs[:i], subs[i+1:]...)
            close(sub)
            return
        }
    }
}
```

---

## 8. Strategy

```go
// Алгоритм инжектируется как зависимость
type SortStrategy func([]int)

func BubbleSort(data []int) { /* ... */ }
func QuickSort(data []int)  { /* ... */ }
func MergeSort(data []int)  { /* ... */ }

type Sorter struct {
    strategy SortStrategy
}

func (s *Sorter) Sort(data []int) {
    s.strategy(data)
}

// Через интерфейс (если нужно состояние):
type Compressor interface {
    Compress(data []byte) ([]byte, error)
}

type GzipCompressor struct{ level int }
type ZstdCompressor struct{ level int }
type LZ4Compressor struct{}

func (g *GzipCompressor) Compress(data []byte) ([]byte, error) { /* ... */ }

type Storage struct {
    compressor Compressor
}

func NewStorage(compressor Compressor) *Storage {
    return &Storage{compressor: compressor}
}
```

---

## 9. Repository Pattern

```go
// Абстрагирует хранилище от бизнес-логики
type UserRepository interface {
    GetByID(ctx context.Context, id int64) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    Create(ctx context.Context, user *User) error
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id int64) error
    List(ctx context.Context, filter UserFilter) ([]*User, error)
}

// PostgreSQL реализация
type pgUserRepo struct {
    db *pgxpool.Pool
}

func NewPGUserRepo(db *pgxpool.Pool) UserRepository {
    return &pgUserRepo{db: db}
}

func (r *pgUserRepo) GetByID(ctx context.Context, id int64) (*User, error) {
    var u User
    err := r.db.QueryRow(ctx,
        `SELECT id, name, email, created_at FROM users WHERE id = $1`, id,
    ).Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)
    if errors.Is(err, pgx.ErrNoRows) {
        return nil, ErrNotFound
    }
    return &u, err
}

// Бизнес-логика не знает о PostgreSQL
type UserService struct {
    repo UserRepository
}

func (s *UserService) RegisterUser(ctx context.Context, email, name string) (*User, error) {
    existing, err := s.repo.GetByEmail(ctx, email)
    if err != nil && !errors.Is(err, ErrNotFound) {
        return nil, fmt.Errorf("check existing: %w", err)
    }
    if existing != nil {
        return nil, ErrUserAlreadyExists
    }
    u := &User{Name: name, Email: email}
    return u, s.repo.Create(ctx, u)
}
```

---

## 10. Circuit Breaker

```go
// Защита от каскадных отказов
type State int

const (
    StateClosed   State = iota // Нормальная работа
    StateOpen                  // Цепь разомкнута, запросы блокируются
    StateHalfOpen              // Пробный запрос
)

type CircuitBreaker struct {
    mu           sync.Mutex
    state        State
    failures     int
    maxFailures  int
    resetTimeout time.Duration
    openedAt     time.Time
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    cb.mu.Lock()
    switch cb.state {
    case StateOpen:
        if time.Since(cb.openedAt) > cb.resetTimeout {
            cb.state = StateHalfOpen
        } else {
            cb.mu.Unlock()
            return ErrCircuitOpen
        }
    }
    cb.mu.Unlock()

    err := fn()

    cb.mu.Lock()
    defer cb.mu.Unlock()

    if err != nil {
        cb.failures++
        if cb.failures >= cb.maxFailures {
            cb.state = StateOpen
            cb.openedAt = time.Now()
        }
        return err
    }

    // Успех — сбросить счётчик
    cb.failures = 0
    cb.state = StateClosed
    return nil
}
```

---

## 11. Outbox Pattern (для гарантированной доставки событий)

```go
// Проблема: как атомарно сохранить в БД И отправить в Kafka?
// Решение: пишем событие в ту же БД транзакционно, отдельный воркер читает и отправляет

// В транзакции: создаём заказ + пишем событие в outbox таблицу
func (s *OrderService) CreateOrder(ctx context.Context, order *Order) error {
    return s.db.WithTransaction(ctx, func(tx pgx.Tx) error {
        // 1. Создаём заказ
        if err := s.orderRepo.CreateTx(ctx, tx, order); err != nil {
            return err
        }

        // 2. Пишем событие в outbox (атомарно с заказом!)
        event := OutboxEvent{
            Topic:   "orders.created",
            Payload: mustMarshal(OrderCreatedEvent{OrderID: order.ID}),
        }
        return s.outboxRepo.CreateTx(ctx, tx, event)
    })
}

// Отдельный воркер публикует события из outbox в Kafka
func (w *OutboxWorker) Run(ctx context.Context) {
    ticker := time.NewTicker(100 * time.Millisecond)
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            events, _ := w.outboxRepo.GetUnpublished(ctx, 100)
            for _, e := range events {
                if err := w.kafka.Publish(ctx, e.Topic, e.Payload); err != nil {
                    continue
                }
                w.outboxRepo.MarkPublished(ctx, e.ID)
            }
        }
    }
}
```

---

## Вопросы на собеседовании

### Q: Какой паттерн используешь для конфигурации в Go?

**Ответ:** Functional Options — функции типа `func(*Config)` передаются как variadic аргументы в конструктор. Преимущества: backwards compatible (новые опции не ломают существующий код), читаемый API, возможность валидации в Build(). Альтернатива — Builder паттерн если нужна явная валидация при построении. Config struct как параметр — проще, но все поля становятся публичными.

---

### Q: Как реализуешь middleware цепочку в Go?

**Ответ:** Через Decorator паттерн — каждый middleware реализует тот же интерфейс и оборачивает следующий. Для HTTP: `func(http.Handler) http.Handler`. Для бизнес-логики: оборачиваем интерфейс репозитория (кеш, метрики, трейсинг). Цепочка читается снаружи внутрь: `Metrics(Cache(Postgres))` — первым вызывается Metrics, последним — Postgres. Это позволяет добавлять cross-cutting concerns (логирование, кеш, трейсинг) без изменения бизнес-логики.

---

### Q: Когда использовать Repository паттерн?

**Ответ:** Когда нужно: 1) тестировать бизнес-логику без реальной БД — мокаем интерфейс, 2) поменять хранилище (PostgreSQL → MongoDB) без изменения сервисов, 3) добавить кеш-слой прозрачно через Decorator. Не нужен если: простой CRUD без бизнес-логики, скрипт или маленький сервис. Ошибка — делать репозиторий слишком "умным" (бизнес-логика должна быть в сервисе, репозиторий — тонкий слой доступа к данным).
