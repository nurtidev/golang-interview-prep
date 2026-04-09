# Live Coding: Fintech & Travel задачи

> Формат: задача → что проверяют → решение с объяснением
> Все задачи реалистичны — такие встречаются на Senior Go интервью

---

## Задача 1: Token Bucket Rate Limiter

### Постановка
Реализуй thread-safe `RateLimiter` по алгоритму Token Bucket.

**Требования:**
- `NewRateLimiter(rate int, burst int)` — rate токенов/сек, burst — максимум
- `Allow() bool` — можно ли выполнить запрос прямо сейчас
- `Wait(ctx context.Context) error` — ждать пока не появится токен

**Контекст:** у нас API к внешнему провайдеру авиабилетов — 100 rps лимит.

### Что проверяют
- Понимание алгоритма Token Bucket vs Leaky Bucket
- Работа с `time.Now()`, вычисление прошедшего времени
- Mutex + правильная логика пополнения токенов
- Context cancellation

### Решение

```go
package ratelimiter

import (
    "context"
    "sync"
    "time"
)

type RateLimiter struct {
    mu       sync.Mutex
    tokens   float64
    maxBurst float64
    rate     float64 // токенов в секунду
    lastTime time.Time
}

func NewRateLimiter(rate, burst int) *RateLimiter {
    return &RateLimiter{
        tokens:   float64(burst),
        maxBurst: float64(burst),
        rate:     float64(rate),
        lastTime: time.Now(),
    }
}

// пополняем токены на основе прошедшего времени
func (rl *RateLimiter) refill() {
    now := time.Now()
    elapsed := now.Sub(rl.lastTime).Seconds()
    rl.tokens += elapsed * rl.rate
    if rl.tokens > rl.maxBurst {
        rl.tokens = rl.maxBurst
    }
    rl.lastTime = now
}

func (rl *RateLimiter) Allow() bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    rl.refill()
    if rl.tokens >= 1 {
        rl.tokens--
        return true
    }
    return false
}

func (rl *RateLimiter) Wait(ctx context.Context) error {
    for {
        rl.mu.Lock()
        rl.refill()
        if rl.tokens >= 1 {
            rl.tokens--
            rl.mu.Unlock()
            return nil
        }
        // сколько ждать до следующего токена
        waitDuration := time.Duration((1 - rl.tokens) / rl.rate * float64(time.Second))
        rl.mu.Unlock()

        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(waitDuration):
            // попробуем снова
        }
    }
}
```

**Тест:**
```go
func TestRateLimiter(t *testing.T) {
    rl := NewRateLimiter(10, 1) // 10 rps, burst=1
    
    assert.True(t, rl.Allow())  // первый — ок
    assert.False(t, rl.Allow()) // сразу второй — нет токенов
    
    time.Sleep(200 * time.Millisecond)
    assert.True(t, rl.Allow())  // пополнилось
}
```

**Вопрос на интервью:** *"Чем Token Bucket отличается от Fixed Window?"*
> Fixed Window считает запросы в окне (напр. 100/мин). Проблема — burst на границе окон: 100 в конце + 100 в начале следующего = 200 за 2 секунды. Token Bucket сглаживает burst: нельзя потратить больше чем накоплено.

---

## Задача 2: Concurrent Flight Search (Fan-Out / Fan-In)

### Постановка
Реализуй `SearchFlights` — параллельный поиск по нескольким провайдерам (Amadeus, Sabre, собственный кеш).

**Требования:**
- Запросы к провайдерам — параллельно
- Общий timeout 3 секунды
- Если провайдер не ответил — пропускаем, не падаем
- Дедупликация результатов по `(flightID)`
- Возвращаем все найденные рейсы, отсортированные по цене

### Что проверяют
- Fan-out / fan-in паттерн
- Context propagation и отмена
- Обработка ошибок без panic
- Дедупликация + сортировка

### Решение

```go
package search

import (
    "context"
    "sort"
    "sync"
    "time"
)

type Flight struct {
    ID       string
    Provider string
    Price    float64
    From, To string
    DepartAt time.Time
}

type Provider interface {
    Search(ctx context.Context, from, to string, date time.Time) ([]Flight, error)
}

func SearchFlights(
    ctx context.Context,
    providers []Provider,
    from, to string,
    date time.Time,
) []Flight {
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    type result struct {
        flights []Flight
        err     error
    }

    resultCh := make(chan result, len(providers)) // буферизованный — не блокируем горутины

    var wg sync.WaitGroup
    for _, p := range providers {
        wg.Add(1)
        go func(provider Provider) {
            defer wg.Done()
            flights, err := provider.Search(ctx, from, to, date)
            resultCh <- result{flights, err}
        }(p)
    }

    // Закрываем канал когда все закончили
    go func() {
        wg.Wait()
        close(resultCh)
    }()

    // Собираем результаты + дедупликация
    seen := make(map[string]struct{})
    var allFlights []Flight

    for res := range resultCh {
        if res.err != nil {
            // логируем, но продолжаем
            continue
        }
        for _, f := range res.flights {
            if _, ok := seen[f.ID]; !ok {
                seen[f.ID] = struct{}{}
                allFlights = append(allFlights, f)
            }
        }
    }

    // Сортируем по цене
    sort.Slice(allFlights, func(i, j int) bool {
        return allFlights[i].Price < allFlights[j].Price
    })

    return allFlights
}
```

**Подвох:** почему канал буферизованный?
> Если `resultCh` небуферизованный и мы вышли по timeout (ctx.Done), горутины провайдеров попытаются писать в канал, но никто не читает → goroutine leak. Буфер `len(providers)` гарантирует что каждая горутина сможет записать результат и завершиться.

---

## Задача 3: Idempotent Booking с distributed lock

### Постановка
Реализуй `BookFlight` — идемпотентное бронирование. Клиент может повторить запрос при сетевой ошибке.

**Требования:**
- `idempotencyKey` из заголовка запроса
- Если бронирование с этим ключом уже есть — вернуть тот же результат
- Защита от одновременных запросов с одним ключом (distributed lock через Redis)
- Транзакция: списание баланса + создание брони — атомарно

### Что проверяют
- Понимание идемпотентности
- Redis SETNX / SET NX EX для distributed lock
- SQL транзакция в Go
- Обработка race condition

### Решение

```go
package booking

import (
    "context"
    "database/sql"
    "errors"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

type BookingResult struct {
    BookingID string
    Status    string
    CreatedAt time.Time
}

type BookingService struct {
    db    *sql.DB
    redis *redis.Client
}

var ErrInsufficientBalance = errors.New("insufficient balance")
var ErrFlightNotAvailable = errors.New("flight not available")

func (s *BookingService) BookFlight(
    ctx context.Context,
    userID, flightID, idempotencyKey string,
    amount float64,
) (*BookingResult, error) {
    // 1. Проверяем кеш — может уже обработали?
    existing, err := s.getExistingBooking(ctx, idempotencyKey)
    if err == nil {
        return existing, nil // возвращаем тот же результат
    }

    // 2. Берём distributed lock на idempotencyKey
    lockKey := fmt.Sprintf("booking:lock:%s", idempotencyKey)
    locked, err := s.redis.SetNX(ctx, lockKey, "1", 30*time.Second).Result()
    if err != nil {
        return nil, fmt.Errorf("redis lock: %w", err)
    }
    if !locked {
        return nil, errors.New("request already in progress, retry later")
    }
    defer s.redis.Del(ctx, lockKey)

    // 3. Двойная проверка после лока (другой запрос мог успеть)
    existing, err = s.getExistingBooking(ctx, idempotencyKey)
    if err == nil {
        return existing, nil
    }

    // 4. Выполняем бронирование в транзакции
    result, err := s.createBookingTx(ctx, userID, flightID, idempotencyKey, amount)
    if err != nil {
        return nil, err
    }

    // 5. Сохраняем в Redis на 24 часа (для будущих повторных запросов)
    s.cacheBookingResult(ctx, idempotencyKey, result)

    return result, nil
}

func (s *BookingService) createBookingTx(
    ctx context.Context,
    userID, flightID, idempotencyKey string,
    amount float64,
) (*BookingResult, error) {
    tx, err := s.db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelSerializable, // для финансовых операций
    })
    if err != nil {
        return nil, err
    }
    defer tx.Rollback() // no-op если CommitTx уже вызван

    // Проверяем баланс и блокируем строку (SELECT FOR UPDATE)
    var balance float64
    err = tx.QueryRowContext(ctx,
        `SELECT balance FROM wallets WHERE user_id = $1 FOR UPDATE`,
        userID,
    ).Scan(&balance)
    if err != nil {
        return nil, err
    }
    if balance < amount {
        return nil, ErrInsufficientBalance
    }

    // Проверяем наличие мест (SELECT FOR UPDATE)
    var availableSeats int
    err = tx.QueryRowContext(ctx,
        `SELECT available_seats FROM flights WHERE id = $1 FOR UPDATE`,
        flightID,
    ).Scan(&availableSeats)
    if err != nil {
        return nil, err
    }
    if availableSeats <= 0 {
        return nil, ErrFlightNotAvailable
    }

    // Списываем баланс
    _, err = tx.ExecContext(ctx,
        `UPDATE wallets SET balance = balance - $1 WHERE user_id = $2`,
        amount, userID,
    )
    if err != nil {
        return nil, err
    }

    // Уменьшаем количество мест
    _, err = tx.ExecContext(ctx,
        `UPDATE flights SET available_seats = available_seats - 1 WHERE id = $1`,
        flightID,
    )
    if err != nil {
        return nil, err
    }

    // Создаём бронирование
    var bookingID string
    var createdAt time.Time
    err = tx.QueryRowContext(ctx,
        `INSERT INTO bookings (user_id, flight_id, amount, idempotency_key, status)
         VALUES ($1, $2, $3, $4, 'confirmed')
         RETURNING id, created_at`,
        userID, flightID, amount, idempotencyKey,
    ).Scan(&bookingID, &createdAt)
    if err != nil {
        return nil, err
    }

    if err := tx.Commit(); err != nil {
        return nil, err
    }

    return &BookingResult{
        BookingID: bookingID,
        Status:    "confirmed",
        CreatedAt: createdAt,
    }, nil
}
```

**Ключевые моменты для обсуждения:**
1. **Почему `SELECT FOR UPDATE`?** — предотвращает race condition когда два запроса одновременно проверяют баланс (оба видят достаточно денег, оба списывают)
2. **Почему `Serializable` isolation?** — для финансовых операций нужна полная изоляция. Phantom reads недопустимы
3. **Зачем double-check после лока?** — между первой проверкой и получением лока другой экземпляр сервиса мог уже обработать запрос

---

## Задача 4: Circuit Breaker

### Постановка
Внешний провайдер периодически падает. Реализуй Circuit Breaker, чтобы не ждать timeout при каждом запросе.

**Состояния:**
- `Closed` — всё ок, пропускаем запросы
- `Open` — провайдер упал, сразу возвращаем ошибку (fast fail)
- `HalfOpen` — пробуем один запрос, если ок — переходим в Closed

### Что проверяют
- State machine в Go
- Atomics vs Mutex для state
- Таймаут для восстановления

### Решение

```go
package circuitbreaker

import (
    "errors"
    "sync"
    "time"
)

type State int

const (
    Closed   State = iota // работаем нормально
    Open                  // fast fail
    HalfOpen              // тестируем восстановление
)

var ErrCircuitOpen = errors.New("circuit breaker is open")

type CircuitBreaker struct {
    mu            sync.Mutex
    state         State
    failures      int
    maxFailures   int
    lastFailure   time.Time
    resetTimeout  time.Duration
}

func NewCircuitBreaker(maxFailures int, resetTimeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        maxFailures:  maxFailures,
        resetTimeout: resetTimeout,
        state:        Closed,
    }
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    cb.mu.Lock()
    state := cb.currentState()
    
    if state == Open {
        cb.mu.Unlock()
        return ErrCircuitOpen
    }
    if state == HalfOpen {
        cb.state = Open // будем осторожны, откроем только если успех
    }
    cb.mu.Unlock()

    err := fn()

    cb.mu.Lock()
    defer cb.mu.Unlock()

    if err != nil {
        cb.onFailure()
        return err
    }
    cb.onSuccess()
    return nil
}

func (cb *CircuitBreaker) currentState() State {
    if cb.state == Open {
        if time.Since(cb.lastFailure) > cb.resetTimeout {
            cb.state = HalfOpen
        }
    }
    return cb.state
}

func (cb *CircuitBreaker) onFailure() {
    cb.failures++
    cb.lastFailure = time.Now()
    if cb.failures >= cb.maxFailures {
        cb.state = Open
    }
}

func (cb *CircuitBreaker) onSuccess() {
    cb.failures = 0
    cb.state = Closed
}
```

**Использование:**
```go
cb := NewCircuitBreaker(5, 30*time.Second)

result, err := cb.Execute(func() error {
    return externalProvider.Search(ctx, params)
})
if errors.Is(err, ErrCircuitOpen) {
    // Используем кеш или отвечаем "сервис временно недоступен"
    return getCachedResults(params)
}
```

---

## Задача 5: Worker Pool с graceful shutdown

### Постановка
Нужно обработать очередь уведомлений об успешном бронировании (email + push). Реализуй worker pool с корректным завершением.

**Требования:**
- N воркеров, обрабатывают jobs конкурентно
- Graceful shutdown: при сигнале — дождаться завершения текущих jobs
- Метрики: количество обработанных / упавших jobs

### Что проверяют
- WaitGroup правильное использование
- Channel как очередь задач
- Graceful shutdown паттерн
- Recover в горутинах (нельзя уронить весь пул)

### Решение

```go
package workerpool

import (
    "context"
    "log"
    "sync"
    "sync/atomic"
)

type Job struct {
    ID      string
    Payload interface{}
}

type WorkerPool struct {
    jobs      chan Job
    wg        sync.WaitGroup
    processed atomic.Int64
    failed    atomic.Int64
    handler   func(Job) error
}

func NewWorkerPool(workers int, queueSize int, handler func(Job) error) *WorkerPool {
    p := &WorkerPool{
        jobs:    make(chan Job, queueSize),
        handler: handler,
    }

    for i := 0; i < workers; i++ {
        p.wg.Add(1)
        go p.worker(i)
    }

    return p
}

func (p *WorkerPool) worker(id int) {
    defer p.wg.Done()

    for job := range p.jobs { // выйдет когда канал закрыт
        p.process(job)
    }
}

func (p *WorkerPool) process(job Job) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("worker panic on job %s: %v", job.ID, r)
            p.failed.Add(1)
        }
    }()

    if err := p.handler(job); err != nil {
        log.Printf("job %s failed: %v", job.ID, err)
        p.failed.Add(1)
        return
    }
    p.processed.Add(1)
}

// Submit отправляет job, блокируется если очередь полна
func (p *WorkerPool) Submit(ctx context.Context, job Job) error {
    select {
    case p.jobs <- job:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

// Shutdown ждёт завершения всех текущих jobs
func (p *WorkerPool) Shutdown() {
    close(p.jobs) // сигнал воркерам: новых jobs не будет
    p.wg.Wait()   // ждём завершения текущих
    log.Printf("pool shutdown: processed=%d failed=%d",
        p.processed.Load(), p.failed.Load())
}
```

**Использование в main:**
```go
pool := NewWorkerPool(10, 100, sendNotification)
defer pool.Shutdown()

// Из Kafka consumer
for msg := range kafkaMessages {
    if err := pool.Submit(ctx, Job{ID: msg.Key, Payload: msg.Value}); err != nil {
        log.Printf("submit failed: %v", err) // context cancelled
        break
    }
}
```

---

## Частые follow-up вопросы

**Q: В задаче 2, что если нам нужен только первый ответ (fastest provider)?**
```go
// Используем канал с буфером 1 + return при первом результате
resultCh := make(chan []Flight, len(providers))
// ... запускаем горутины ...
// Берём первый результат
select {
case flights := <-resultCh:
    cancel() // отменяем остальные
    return flights
case <-ctx.Done():
    return nil
}
```

**Q: Как тестировать RateLimiter без `time.Sleep`?**
> Инъекция зависимости: вместо `time.Now()` принимаем `func() time.Time` в конструкторе. В тестах передаём mock-функцию которая возвращает нужное время.

**Q: Что лучше для счётчиков в WorkerPool — `atomic` или `mutex`?**
> `atomic` для простых счётчиков (Add, Load) — быстрее, нет блокировки. `mutex` нужен когда несколько полей надо изменить атомарно (например, state + lastFailure в Circuit Breaker).
