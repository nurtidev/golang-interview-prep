# Go Deep: 10 задач на глубину языка

> Формат: читаешь задачу → пишешь решение сам → открываешь ответ
> Не подглядывай раньше времени — толку не будет

---

## Задача 1 — `errgroup` + параллельные запросы

```
Контекст: travel-сервис. При поиске рейсов нужно
запросить 3 провайдера параллельно.

Требования:
- Все 3 запроса выполняются параллельно
- Если хотя бы один вернул ошибку — возвращаем ошибку
- Если все успешны — возвращаем объединённый список
- Общий timeout — 3 секунды
```

```go
type Flight struct {
    ID       string
    Price    float64
    Provider string
}

type Provider interface {
    Search(ctx context.Context, from, to string) ([]Flight, error)
}

func SearchAll(ctx context.Context, providers []Provider, from, to string) ([]Flight, error) {
    // твоя реализация
}
```

<details>
<summary>Ответ</summary>

```go
import "golang.org/x/sync/errgroup"

func SearchAll(ctx context.Context, providers []Provider, from, to string) ([]Flight, error) {
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    g, ctx := errgroup.WithContext(ctx)

    results := make([][]Flight, len(providers))

    for i, p := range providers {
        i, p := i, p // capture loop variables
        g.Go(func() error {
            flights, err := p.Search(ctx, from, to)
            if err != nil {
                return fmt.Errorf("provider %d: %w", i, err)
            }
            results[i] = flights
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }

    var all []Flight
    for _, r := range results {
        all = append(all, r...)
    }
    return all, nil
}
```

**Технологии:**
- `errgroup.WithContext` — запускает горутины и отменяет контекст при первой ошибке. Альтернатива — `sync.WaitGroup` + канал для ошибок, но `errgroup` делает это чище
- `results[i]` — pre-allocated slice по индексу, без mutex (каждая горутина пишет в свой индекс)
- `i, p := i, p` — shadow переменных цикла до Go 1.22, иначе все горутины захватят последнее значение
- `%w` — wrapping ошибки, чтобы caller мог использовать `errors.Is/As`

</details>

---

## Задача 2 — `sync.Pool`

```
Контекст: сервис сериализует JSON для 50k rps.
Профилировщик показал: 40% времени — runtime.mallocgc.
Причина — каждый запрос создаёт новый bytes.Buffer.

Требования:
- Переиспользовать буферы через sync.Pool
- Функция Serialize должна быть goroutine-safe
- После использования буфер должен возвращаться в пул
```

```go
type Serializer struct {
    // твои поля
}

func NewSerializer() *Serializer {
    // твоя реализация
}

func (s *Serializer) Serialize(v any) ([]byte, error) {
    // твоя реализация
}
```

<details>
<summary>Ответ</summary>

```go
type Serializer struct {
    pool sync.Pool
}

func NewSerializer() *Serializer {
    return &Serializer{
        pool: sync.Pool{
            New: func() any {
                return new(bytes.Buffer)
            },
        },
    }
}

func (s *Serializer) Serialize(v any) ([]byte, error) {
    buf := s.pool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset() // очищаем перед возвратом
        s.pool.Put(buf)
    }()

    if err := json.NewEncoder(buf).Encode(v); err != nil {
        return nil, err
    }

    // Копируем — иначе вернём buf в пул, а caller держит ссылку на его память
    result := make([]byte, buf.Len())
    copy(result, buf.Bytes())
    return result, nil
}
```

**Технологии:**
- `sync.Pool` — переиспользование объектов между горутинами. GC может очистить пул в любой момент, поэтому `New` — обязательный fallback
- `buf.Reset()` перед `Put` — критично, иначе следующий `Get` получит буфер с чужими данными
- `copy(result, buf.Bytes())` — обязательно. Если вернуть `buf.Bytes()` напрямую, а потом положить buf обратно в пул — другая горутина перезапишет ту же память
- `json.NewEncoder` vs `json.Marshal` — энкодер пишет в writer, маршал создаёт новый буфер каждый раз

</details>

---

## Задача 3 — Generics

```
Контекст: нам нужен type-safe кеш который работает с любым типом.
Без generics пришлось бы делать interface{} и type assertion везде.

Требования:
- TTL на каждый элемент отдельно
- goroutine-safe
- Set(key, value, ttl) / Get(key) (value, bool)
- Expired элементы не возвращаются
```

```go
type Cache[K comparable, V any] struct {
    // твои поля
}

func NewCache[K comparable, V any]() *Cache[K, V] {
    // твоя реализация
}

func (c *Cache[K, V]) Set(key K, value V, ttl time.Duration) {
    // твоя реализация
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    // твоя реализация
}
```

<details>
<summary>Ответ</summary>

```go
type entry[V any] struct {
    value     V
    expiresAt time.Time
}

type Cache[K comparable, V any] struct {
    mu    sync.RWMutex
    items map[K]entry[V]
}

func NewCache[K comparable, V any]() *Cache[K, V] {
    return &Cache[K, V]{
        items: make(map[K]entry[V]),
    }
}

func (c *Cache[K, V]) Set(key K, value V, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = entry[V]{
        value:     value,
        expiresAt: time.Now().Add(ttl),
    }
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    e, ok := c.items[key]
    if !ok || time.Now().After(e.expiresAt) {
        var zero V
        return zero, false // zero value — правильный способ вернуть "ничего"
    }
    return e.value, true
}
```

**Технологии:**
- `[K comparable, V any]` — `comparable` нужен для ключей map (поддерживает `==`), `any` для значений
- `entry[V]` — вспомогательный generic тип для хранения значения + expiry
- `var zero V` — единственный способ вернуть zero value для generic типа. Нельзя написать `return nil, false` если V это `int`
- `RWMutex` — читатели не блокируют друг друга, что важно при высоком RPS

</details>

---

## Задача 4 — Pipeline с backpressure

```
Контекст: читаем события из Kafka, обогащаем данными из БД,
пишем в ClickHouse. Каждый этап — отдельная горутина.

Требования:
- 3 этапа: read → enrich → write
- Если write медленный — read должен замедлиться (backpressure)
- При ctx.Done — корректно завершить все этапы
- Вернуть первую ошибку из любого этапа
```

```go
type Event struct {
    ID   string
    Data string
}

func read(ctx context.Context) (<-chan Event, <-chan error)
func enrich(ctx context.Context, in <-chan Event) (<-chan Event, <-chan error)
func write(ctx context.Context, in <-chan Event) <-chan error

func RunPipeline(ctx context.Context) error {
    // твоя реализация
}
```

<details>
<summary>Ответ</summary>

```go
func RunPipeline(ctx context.Context) error {
    events, readErr := read(ctx)
    enriched, enrichErr := enrich(ctx, events)
    writeErr := write(ctx, enriched)

    for {
        select {
        case err, ok := <-readErr:
            if ok && err != nil {
                return fmt.Errorf("read: %w", err)
            }
            readErr = nil // nil канал в select никогда не срабатывает
        case err, ok := <-enrichErr:
            if ok && err != nil {
                return fmt.Errorf("enrich: %w", err)
            }
            enrichErr = nil
        case err, ok := <-writeErr:
            if ok && err != nil {
                return fmt.Errorf("write: %w", err)
            }
            writeErr = nil
        case <-ctx.Done():
            return ctx.Err()
        }

        if readErr == nil && enrichErr == nil && writeErr == nil {
            return nil
        }
    }
}

// Backpressure достигается размером буфера канала между этапами:
func read(ctx context.Context) (<-chan Event, <-chan error) {
    out := make(chan Event, 100) // если enrich медленный — read блокируется здесь
    errc := make(chan error, 1)
    go func() {
        defer close(out)
        defer close(errc)
        // ... читаем из Kafka
    }()
    return out, errc
}
```

**Технологии:**
- Размер буфера канала = механизм backpressure. Небуферизованный — жёсткий backpressure. Буферизованный — мягкий
- `select` с несколькими error каналами — ждём первую ошибку из любого этапа
- `nil` канал в `select` никогда не срабатывает — способ "отключить" кейс после закрытия канала
- Каждый этап закрывает свой output канал при завершении — сигнал следующему этапу

</details>

---

## Задача 5 — Lock-free счётчик метрик

```
Контекст: счётчики запросов, ошибок, суммарной latency.
Вызывается из тысяч горутин одновременно.

Требования:
- Increment(name string)
- Add(name string, delta int64)
- Snapshot() map[string]int64 — атомарный снимок всех метрик
- Без mutex на hot path (используй atomic)
```

```go
type Metrics struct {
    // твои поля
}

func NewMetrics() *Metrics
func (m *Metrics) Increment(name string)
func (m *Metrics) Add(name string, delta int64)
func (m *Metrics) Snapshot() map[string]int64
```

<details>
<summary>Ответ</summary>

```go
type Metrics struct {
    mu     sync.RWMutex
    values map[string]*atomic.Int64
}

func NewMetrics() *Metrics {
    return &Metrics{
        values: make(map[string]*atomic.Int64),
    }
}

func (m *Metrics) getOrCreate(name string) *atomic.Int64 {
    m.mu.RLock()
    v, ok := m.values[name]
    m.mu.RUnlock()
    if ok {
        return v
    }

    m.mu.Lock()
    defer m.mu.Unlock()
    // double-check: пока ждали Lock, другая горутина могла создать
    if v, ok = m.values[name]; ok {
        return v
    }
    v = &atomic.Int64{}
    m.values[name] = v
    return v
}

func (m *Metrics) Increment(name string) {
    m.getOrCreate(name).Add(1)
}

func (m *Metrics) Add(name string, delta int64) {
    m.getOrCreate(name).Add(delta)
}

func (m *Metrics) Snapshot() map[string]int64 {
    m.mu.RLock()
    defer m.mu.RUnlock()

    snap := make(map[string]int64, len(m.values))
    for k, v := range m.values {
        snap[k] = v.Load()
    }
    return snap
}
```

**Технологии:**
- `atomic.Int64` (Go 1.19+) — атомарные операции без mutex для hot path (Add/Increment)
- `sync.RWMutex` — только для map (добавление новых ключей). Сама map не thread-safe, но `*atomic.Int64` указатели после записи не меняются
- double-check в `getOrCreate` — между RUnlock и Lock другая горутина могла создать ключ
- Разделение ответственности: mutex защищает структуру map, atomic защищает значения

</details>

---

## Задача 6 — Nil interface gotcha

```
Контекст: классическая ловушка Go которую спрашивают на Senior интервью.
Найди баг и объясни почему он происходит.
```

```go
type MyError struct {
    Code    int
    Message string
}

func (e *MyError) Error() string {
    return fmt.Sprintf("code=%d msg=%s", e.Code, e.Message)
}

func getUser(id int) *MyError {
    if id <= 0 {
        return &MyError{Code: 400, Message: "invalid id"}
    }
    return nil
}

func process(id int) error {
    err := getUser(id)
    if err != nil {
        return err
    }
    return nil
}

func main() {
    err := process(1)
    if err != nil {
        fmt.Println("got error:", err) // почему это печатается при id=1?
    } else {
        fmt.Println("ok")
    }
}

// Вопросы:
// 1. Что выведет программа при id=1? Почему?
// 2. Как исправить?
```

<details>
<summary>Ответ</summary>

**Что выведет:** `got error: <nil>` — ошибка не nil, хотя `getUser` вернул nil.

**Почему:**

Интерфейс в Go — это пара `(тип, значение)`. `error` — интерфейс.

```
getUser(1) возвращает *MyError(nil)

При присвоении в error интерфейс:
err := getUser(id)  →  err = (type=*MyError, value=nil)

Проверка err != nil:
(type=*MyError, value=nil) != (nil, nil)  →  TRUE
```

Интерфейс `nil` только когда оба компонента nil: `(nil, nil)`.

**Исправление:**
```go
// Вариант 1: функция сразу возвращает error интерфейс
func getUser(id int) error {
    if id <= 0 {
        return &MyError{Code: 400, Message: "invalid id"}
    }
    return nil // теперь (nil, nil)
}

// Вариант 2: явная проверка конкретного типа
func process(id int) error {
    err := getUser(id)
    if err == nil { // сравниваем *MyError с nil — корректно
        return nil
    }
    return err
}
```

**Технологии / концепции:**
- Интерфейс в Go = `(itab *type, data unsafe.Pointer)` — два поля
- Nil interface только когда оба поля nil
- Правило: **никогда не возвращай конкретный тип там где ожидается интерфейс**, если значение может быть nil
- Одна из самых частых ловушек на Senior Go интервью

</details>

---

## Задача 7 — HTTP Middleware chain

```
Контекст: цепочка middleware для HTTP сервера.

Требования:
- Тип Middleware и функция Chain(middlewares...)
- RequestID middleware — добавляет UUID в context и заголовок ответа
- Logger middleware — логирует метод, путь, статус, длительность
- Middleware должны быть composable (легко добавлять новые)
```

```go
type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, middlewares ...Middleware) http.Handler {
    // твоя реализация
}

func RequestID() Middleware {
    // твоя реализация
}

func Logger(log *slog.Logger) Middleware {
    // твоя реализация
}

type contextKey string
const RequestIDKey contextKey = "request_id"
```

<details>
<summary>Ответ</summary>

```go
type Middleware func(http.Handler) http.Handler

// Chain применяет middleware в порядке слева направо
// Chain(h, A, B, C) → A(B(C(h))) — A выполняется первым
func Chain(h http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

type contextKey string

const RequestIDKey contextKey = "request_id"

func RequestID() Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            id := r.Header.Get("X-Request-ID")
            if id == "" {
                id = uuid.New().String()
            }
            ctx := context.WithValue(r.Context(), RequestIDKey, id)
            w.Header().Set("X-Request-ID", id)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// responseWriter-обёртка чтобы перехватить статус код
type statusWriter struct {
    http.ResponseWriter
    status int
}

func (sw *statusWriter) WriteHeader(code int) {
    sw.status = code
    sw.ResponseWriter.WriteHeader(code)
}

func Logger(log *slog.Logger) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            sw := &statusWriter{ResponseWriter: w, status: http.StatusOK}

            next.ServeHTTP(sw, r)

            log.Info("request",
                "method",      r.Method,
                "path",        r.URL.Path,
                "status",      sw.status,
                "duration_ms", time.Since(start).Milliseconds(),
                "request_id",  r.Context().Value(RequestIDKey),
            )
        })
    }
}
```

**Технологии:**
- `func(http.Handler) http.Handler` — стандартный тип middleware в Go экосистеме
- Обратный порядок в `Chain` — чтобы первый middleware оборачивал снаружи (выполнялся первым)
- `statusWriter` обёртка — `http.ResponseWriter` не отдаёт статус код обратно, нужно перехватывать `WriteHeader`
- `context.WithValue` с типизированным ключом (`contextKey`) — защита от коллизий с другими пакетами
- `slog` (Go 1.21+) — стандартный structured logger

</details>

---

## Задача 8 — Retry с exponential backoff

```
Контекст: вызываем внешний платёжный провайдер.
Нужна retry-логика которая не спамит провайдера при деградации.

Требования:
- Максимум N попыток
- Задержка: 100ms → 200ms → 400ms → 800ms (exponential)
- Jitter ±20% — чтобы не создавать thundering herd
- Не ретраить если ошибка бизнес-логики (NonRetryableError)
- Уважать ctx отмену
```

```go
type RetryableFunc func(ctx context.Context) error

type NonRetryableError struct {
    Err error
}

func (e *NonRetryableError) Error() string { return e.Err.Error() }

func Retry(ctx context.Context, maxAttempts int, fn RetryableFunc) error {
    // твоя реализация
}
```

<details>
<summary>Ответ</summary>

```go
func Retry(ctx context.Context, maxAttempts int, fn RetryableFunc) error {
    backoff := 100 * time.Millisecond

    for attempt := 1; attempt <= maxAttempts; attempt++ {
        err := fn(ctx)
        if err == nil {
            return nil
        }

        // Бизнес-ошибка — не ретраим
        var nonRetryable *NonRetryableError
        if errors.As(err, &nonRetryable) {
            return nonRetryable.Err
        }

        // Последняя попытка — возвращаем ошибку сразу
        if attempt == maxAttempts {
            return fmt.Errorf("after %d attempts: %w", maxAttempts, err)
        }

        // Jitter: ±20% от backoff
        jitter := time.Duration(rand.Int63n(int64(backoff/5)*2) - int64(backoff/5))
        wait := backoff + jitter

        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(wait):
        }

        backoff *= 2
    }
    return nil
}
```

**Технологии:**
- Exponential backoff — каждая попытка ждёт вдвое дольше. Снижает нагрузку на деградирующий сервис
- Jitter — случайное смещение. Без него все инстансы ретраят одновременно → thundering herd → провайдер не восстанавливается
- `errors.As` — проверяем тип ошибки через unwrapping. Лучше чем type assertion напрямую
- `select { case <-ctx.Done() }` — прерываем ожидание при отмене контекста

</details>

---

## Задача 9 — Pub/Sub in-memory

```
Контекст: внутренний event bus для сервиса.
Сервисы подписываются на события, другие — публикуют.

Требования:
- Subscribe(topic string) <-chan string
- Publish(topic string, msg string) — отправляет всем подписчикам
- Unsubscribe(topic string, ch <-chan string)
- Медленный подписчик НЕ должен блокировать остальных
```

```go
type PubSub struct {
    // твои поля
}

func NewPubSub() *PubSub
func (ps *PubSub) Subscribe(topic string) <-chan string
func (ps *PubSub) Publish(topic string, msg string)
func (ps *PubSub) Unsubscribe(topic string, ch <-chan string)
```

<details>
<summary>Ответ</summary>

```go
type PubSub struct {
    mu   sync.RWMutex
    subs map[string][]chan string
}

func NewPubSub() *PubSub {
    return &PubSub{subs: make(map[string][]chan string)}
}

func (ps *PubSub) Subscribe(topic string) <-chan string {
    ch := make(chan string, 10) // буфер — медленный подписчик не блокирует Publish
    ps.mu.Lock()
    ps.subs[topic] = append(ps.subs[topic], ch)
    ps.mu.Unlock()
    return ch
}

func (ps *PubSub) Publish(topic string, msg string) {
    ps.mu.RLock()
    subs := make([]chan string, len(ps.subs[topic]))
    copy(subs, ps.subs[topic]) // копируем чтобы быстро отпустить lock
    ps.mu.RUnlock()

    for _, ch := range subs {
        select {
        case ch <- msg:
        default:
            // подписчик не успевает — drop (можно заменить на метрику)
        }
    }
}

func (ps *PubSub) Unsubscribe(topic string, ch <-chan string) {
    ps.mu.Lock()
    defer ps.mu.Unlock()

    subs := ps.subs[topic]
    for i, sub := range subs {
        if sub == ch {
            ps.subs[topic] = append(subs[:i], subs[i+1:]...)
            close(sub) // сигнал подписчику что канал закрыт
            return
        }
    }
}
```

**Технологии:**
- Буферизованный канал подписчика — ключевое решение. Без буфера `Publish` блокируется на медленном подписчике
- `select { default: }` в Publish — non-blocking send. Если буфер полон — drop
- `copy(subs, ...)` под RLock — копируем срез и отпускаем lock. Отправка в каналы может быть медленной, не держим lock
- `close(sub)` в Unsubscribe — подписчик узнаёт о закрытии через `v, ok := <-ch` где `ok == false`

</details>

---

## Задача 10 — Memory leak: найди и исправь

```
Контекст: сервис обрабатывает WebSocket соединения.
Через 24 часа работы — OOM. Найди утечку и исправь.
```

```go
type ConnectionManager struct {
    mu    sync.Mutex
    conns map[string]*websocket.Conn
    stats map[string][]time.Time // история подключений для аналитики
}

func (cm *ConnectionManager) Connect(userID string, conn *websocket.Conn) {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    cm.conns[userID] = conn
    cm.stats[userID] = append(cm.stats[userID], time.Now())
}

func (cm *ConnectionManager) Disconnect(userID string) {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    delete(cm.conns, userID)
    // stats намеренно не трогаем — нужна история
}

func (cm *ConnectionManager) GetHistory(userID string) []time.Time {
    cm.mu.Lock()
    defer cm.mu.Unlock()
    return cm.stats[userID] // возвращаем напрямую
}

// Вопросы:
// 1. Где утечки? (их несколько)
// 2. Как исправить не теряя аналитику?
// 3. Что ещё здесь можно улучшить?
```

<details>
<summary>Ответ</summary>

**Утечки:**

**1. `stats` растёт бесконечно** — каждое подключение добавляет элемент, никогда не удаляется. При 1000 подключений/сутки за год — миллионы записей.

**2. `GetHistory` возвращает внутренний slice** — caller может его модифицировать, нарушая инварианты (data race при чтении без lock).

**3. `sync.Mutex` вместо `sync.RWMutex`** — `GetHistory` только читает, но блокирует на запись.

```go
type ConnectionManager struct {
    mu         sync.RWMutex
    conns      map[string]*websocket.Conn
    stats      map[string][]time.Time
    maxHistory int
    historyTTL time.Duration
}

func NewConnectionManager(maxHistory int, historyTTL time.Duration) *ConnectionManager {
    return &ConnectionManager{
        conns:      make(map[string]*websocket.Conn),
        stats:      make(map[string][]time.Time),
        maxHistory: maxHistory,
        historyTTL: historyTTL,
    }
}

func (cm *ConnectionManager) Connect(userID string, conn *websocket.Conn) {
    cm.mu.Lock()
    defer cm.mu.Unlock()

    cm.conns[userID] = conn

    history := append(cm.stats[userID], time.Now())

    // Удаляем записи старше TTL
    cutoff := time.Now().Add(-cm.historyTTL)
    start := 0
    for start < len(history) && history[start].Before(cutoff) {
        start++
    }
    history = history[start:]

    // Ограничиваем максимальный размер
    if len(history) > cm.maxHistory {
        history = history[len(history)-cm.maxHistory:]
    }

    cm.stats[userID] = history
}

func (cm *ConnectionManager) GetHistory(userID string) []time.Time {
    cm.mu.RLock()
    defer cm.mu.RUnlock()

    src := cm.stats[userID]
    result := make([]time.Time, len(src))
    copy(result, src) // возвращаем копию
    return result
}
```

**Технологии:**
- Sliding window — храним только последние N записей или за последние X времени
- `copy` при возврате slice — защита от непреднамеренной мутации внешним кодом
- `RWMutex` вместо `Mutex` — `GetHistory` только читает, можно параллельно
- Trim в момент записи — простое решение без background goroutine

</details>
