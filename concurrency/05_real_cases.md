# Паттерны параллельного программирования: живые кейсы на Go

---

## Лекция

### 1. Worker Pool — пул воркеров

**Кейс:** есть 10 000 URL для парсинга. Нельзя запустить 10 000 горутин — перегрузим сеть и память. Нужно ограничить параллелизм.

```go
func WorkerPool(ctx context.Context, jobs <-chan string, numWorkers int) <-chan Result {
    results := make(chan Result)

    var wg sync.WaitGroup
    wg.Add(numWorkers)

    for i := 0; i < numWorkers; i++ {
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case job, ok := <-jobs:
                    if !ok {
                        return
                    }
                    results <- process(job)
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}

// Использование:
jobs := make(chan string, 100)
results := WorkerPool(ctx, jobs, 10) // 10 воркеров

go func() {
    defer close(jobs)
    for _, url := range urls {
        jobs <- url
    }
}()

for r := range results {
    fmt.Println(r)
}
```

**Ловушки:**
- Закрывать `jobs` должен отправитель, не воркеры
- `results` закрывается только после завершения всех воркеров через `wg.Wait()`
- Буфер у `jobs` предотвращает блокировку отправителя

---

### 2. Fan-In — слияние каналов

**Кейс:** читаем данные из нескольких источников (Kafka топики, микросервисы) и обрабатываем в одном месте.

```go
func Merge(ctx context.Context, channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    wg.Add(len(channels))

    for _, ch := range channels {
        go func(c <-chan int) {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case v, ok := <-c:
                    if !ok {
                        return
                    }
                    out <- v
                }
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

---

### 3. Fan-Out — раздача работы

**Кейс:** один источник данных, несколько обработчиков. Например, входящие заказы распределяем по воркерам по регионам.

```go
func FanOut(ctx context.Context, in <-chan Order, numWorkers int) []<-chan Order {
    outputs := make([]chan Order, numWorkers)
    for i := range outputs {
        outputs[i] = make(chan Order, 10)
    }

    go func() {
        defer func() {
            for _, out := range outputs {
                close(out)
            }
        }()

        i := 0
        for {
            select {
            case <-ctx.Done():
                return
            case order, ok := <-in:
                if !ok {
                    return
                }
                // round-robin распределение
                outputs[i%numWorkers] <- order
                i++
            }
        }
    }()

    result := make([]<-chan Order, numWorkers)
    for i, ch := range outputs {
        result[i] = ch
    }
    return result
}
```

---

### 4. Pipeline — конвейер обработки

**Кейс:** обработка данных в несколько этапов: читаем из БД → фильтруем → трансформируем → записываем. Каждый этап — отдельная горутина.

```go
func generate(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case <-ctx.Done():
                return
            case out <- n:
            }
        }
    }()
    return out
}

func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            select {
            case <-ctx.Done():
                return
            case out <- v * v:
            }
        }
    }()
    return out
}

func filter(ctx context.Context, in <-chan int, pred func(int) bool) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            select {
            case <-ctx.Done():
                return
            default:
                if pred(v) {
                    out <- v
                }
            }
        }
    }()
    return out
}

// Использование:
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

nums := generate(ctx, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
squared := square(ctx, nums)
evens := filter(ctx, squared, func(v int) bool { return v%2 == 0 })

for v := range evens {
    fmt.Println(v) // 4, 16, 36, 64, 100
}
```

---

### 5. Rate Limiter — ограничение частоты запросов

**Кейс:** не более 100 запросов в секунду к внешнему API.

**Простой rate limiter через тикер:**
```go
type RateLimiter struct {
    ticker *time.Ticker
    quit   chan struct{}
}

func NewRateLimiter(rps int) *RateLimiter {
    return &RateLimiter{
        ticker: time.NewTicker(time.Second / time.Duration(rps)),
        quit:   make(chan struct{}),
    }
}

func (rl *RateLimiter) Wait(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    case <-rl.ticker.C:
        return nil
    }
}

func (rl *RateLimiter) Stop() {
    rl.ticker.Stop()
    close(rl.quit)
}
```

**Token Bucket — более гибкий вариант:**
```go
type TokenBucket struct {
    tokens   int
    capacity int
    refill   int           // токенов за интервал
    interval time.Duration
    mu       sync.Mutex
    last     time.Time
}

func NewTokenBucket(capacity, refillPerSec int) *TokenBucket {
    return &TokenBucket{
        tokens:   capacity,
        capacity: capacity,
        refill:   refillPerSec,
        interval: time.Second,
        last:     time.Now(),
    }
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    now := time.Now()
    elapsed := now.Sub(tb.last)
    tb.last = now

    // Добавляем токены за прошедшее время
    newTokens := int(elapsed.Seconds() * float64(tb.refill))
    tb.tokens = min(tb.capacity, tb.tokens+newTokens)

    if tb.tokens <= 0 {
        return false
    }
    tb.tokens--
    return true
}
```

---

### 6. Semaphore — ограничение параллелизма

**Кейс:** не более N одновременных запросов к БД.

```go
type Semaphore struct {
    ch chan struct{}
}

func NewSemaphore(n int) *Semaphore {
    return &Semaphore{ch: make(chan struct{}, n)}
}

func (s *Semaphore) Acquire(ctx context.Context) error {
    select {
    case s.ch <- struct{}{}:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func (s *Semaphore) Release() {
    <-s.ch
}

// Использование:
sem := NewSemaphore(5) // максимум 5 параллельных запросов

for _, id := range orderIDs {
    if err := sem.Acquire(ctx); err != nil {
        break
    }
    go func(id int) {
        defer sem.Release()
        processOrder(ctx, id)
    }(id)
}
```

---

### 7. Graceful Shutdown — корректное завершение

**Кейс:** сервис получил SIGTERM, нужно дождаться завершения активных запросов и закрыть соединения.

```go
func main() {
    server := &http.Server{Addr: ":8080", Handler: handler}

    // Канал для сигналов ОС
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

    // Запускаем сервер в горутине
    go func() {
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    // Ждём сигнала
    <-quit
    log.Println("shutting down...")

    // Даём 30 секунд на завершение активных запросов
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Fatal("forced shutdown:", err)
    }

    log.Println("server stopped")
}
```

---

### 8. Параллельное чтение из кэша и БД

**Кейс (реальный вопрос из Ozon):** хочем получить данные — сначала пробуем кэш, но если он медленный, параллельно идём в БД. Берём что пришло первым.

```go
func GetOrder(ctx context.Context, id int) (Order, error) {
    results := make(chan Order, 2)
    errs := make(chan error, 2)

    // Запрос в кэш
    go func() {
        order, err := cache.Get(ctx, id)
        if err != nil {
            errs <- err
            return
        }
        results <- order
    }()

    // Параллельно — запрос в БД (с задержкой, чтобы дать кэшу шанс)
    go func() {
        // Hedged request: ждём немного, потом идём в БД
        timer := time.NewTimer(5 * time.Millisecond)
        defer timer.Stop()
        select {
        case <-ctx.Done():
            errs <- ctx.Err()
            return
        case <-timer.C:
        }

        order, err := db.Get(ctx, id)
        if err != nil {
            errs <- err
            return
        }
        results <- order
    }()

    // Ждём первый успешный результат
    var errCount int
    for {
        select {
        case order := <-results:
            return order, nil
        case <-errs:
            errCount++
            if errCount == 2 {
                return Order{}, errors.New("both cache and db failed")
            }
        case <-ctx.Done():
            return Order{}, ctx.Err()
        }
    }
}
```

Этот паттерн называется **Hedged Requests** — посылаем дублирующий запрос если первый не ответил за порог.

---

## Вопросы с интервью

**Q: Чем Worker Pool лучше чем просто `go func()` для каждой задачи?**
A: Неограниченное количество горутин может исчерпать память и перегрузить внешние системы (БД, API). Worker Pool даёт предсказуемое потребление ресурсов. 10 воркеров по 10K задач vs 10K горутин одновременно — разница в нагрузке на систему кратная.

**Q: Как не допустить утечки горутин в Fan-In?**
A: Всегда передавать context и обрабатывать `ctx.Done()`. Без context горутина будет вечно ждать данных из закрытого канала или зависнет если канал никогда не закроется. Второй момент — закрывать `out` только после `wg.Wait()`.

**Q: Мьютекс или канал — когда что?**
A: Канал — когда передаёшь данные между горутинами или сигнализируешь о событии. Мьютекс — когда защищаешь общее состояние (кэш, счётчик). Семафор через канал — когда ограничиваешь параллелизм.

**Q: Что такое data race и как найти?**
A: Несинхронизированный доступ к переменной из нескольких горутин где хотя бы один доступ — запись. Находим через `go test -race` или `go run -race`. Фиксим через mutex, atomic или передачу владения через канал.

**Q: Как правильно завершить pipeline если один из этапов вернул ошибку?**
A: Передать context через все этапы. При ошибке вызвать `cancel()` — все горутины увидят `ctx.Done()` и корректно завершатся. Канал с ошибками `chan error` возвращать отдельно от канала с данными.

**Q: Что такое Hedged Requests?**
A: Паттерн для снижения хвостовых задержек (p99). Если запрос не ответил за порог (5-10ms) — посылаем дублирующий запрос на другой экземпляр. Берём первый ответ, второй отменяем через context. Google использует этот паттерн повсеместно.
