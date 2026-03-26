# Персонализированные ответы на основе CV

> Файл для подготовки к интервью. Все ответы привязаны к реальному опыту из CV.
> Читай вслух — так лучше запоминается.

---

## 1. "Расскажи о себе" (elevator pitch, ~2 мин)

> **Используй:** О себе + последние 2 места работы + что ищешь
> **Акцент для high-load роли:** выдели опыт с нагрузкой, оптимизациями, инфраструктурой

Я backend-разработчик с 4+ годами коммерческого опыта на Go, специализируюсь на высоконагруженных системах в финтехе.

Начинал в Darwin Tech Labs — строил микросервисную торговую платформу: сервисы исполнения ордеров, управления балансами, интеграция с крипто-кошельками. Там разбирался с high-write нагрузкой в PostgreSQL: партиционирование таблиц с миллионами записей, BRIN-индексы для временных рядов, покрывающие индексы. Первый опыт с event-driven архитектурой через RabbitMQ.

Потом — Sergek Group, платформа мониторинга городского трафика. Обрабатывали поток событий от сотен камер и датчиков в реальном времени. Оптимизировал consumer'ы: заменил поштучную обработку на batch-вставки в ClickHouse (буфер 500 сообщений, flush каждые 100ms) — это дало 10x рост throughput. Там же столкнулся с задачами профилирования и поиска bottleneck'ов под нагрузкой.

Сейчас в eMoney.ge — цифровые платежи. Проектирую API с акцентом на идемпотентность (платежи нельзя задвоить), работаю с согласованностью данных при распределённых транзакциях, строю observability через Prometheus + Grafana. Слежу за p99 latency и error rate в реальном времени.

В сумме: у меня есть опыт и с архитектурой (event sourcing, идемпотентность, distributed systems), и с низким уровнем — профилирование Go через pprof, понимание как работает GMP-планировщик, memory allocator, escape analysis.

Ищу место, где есть реально высокая нагрузка и команда которая думает о производительности системно, а не только когда "всё упало".

---

## 2. "Как обеспечивал идемпотентность платёжных операций?" (eMoney)

> **Ключевые слова:** idempotency key, unique constraint, at-least-once

**Проблема:** внешний провайдер может не ответить, и клиент повторит запрос. Нужно гарантировать, что деньги спишутся ровно один раз.

**Решение которое применял:**

1. Клиент генерирует `idempotency_key` (UUID) и передаёт в заголовке запроса.
2. На уровне БД — таблица `payments` с `UNIQUE(idempotency_key)`.
3. При повторном запросе — `INSERT ... ON CONFLICT DO NOTHING` + возвращаем уже сохранённый результат.
4. Для операций через брокер (Kafka/RabbitMQ) — консьюмер перед обработкой проверяет наличие ключа в Redis/БД. Если уже обработано — пропускаем, делаем ack.

**Что это даёт:** система становится at-least-once на транспорте, но exactly-once на уровне бизнес-логики.

```go
// Пример: INSERT с idempotency key
query := `
    INSERT INTO payments (id, idempotency_key, amount, status, created_at)
    VALUES ($1, $2, $3, 'pending', NOW())
    ON CONFLICT (idempotency_key) DO NOTHING
    RETURNING id, status
`
```

---

## 3. "Как хранили события в ClickHouse в Sergek Group?"

> **Ключевые слова:** MergeTree, ORDER BY, батчевая вставка, TTL

**Контекст:** платформа мониторинга трафика — сотни тысяч событий от камер и датчиков в сутки, нужна аналитика.

**Схема:** использовали `MergeTree` с сортировкой по `(camera_id, timestamp)` — это ключ для эффективных range-запросов "все события с камеры X за сутки".

**Вставка:** Go-сервис накапливал события в буфере и делал батчевую вставку раз в секунду через `database/sql` с `INSERT INTO ... VALUES (batch)`. Одиночные INSERT в ClickHouse — антипаттерн, он плохо переваривает высокочастотные маленькие вставки.

**TTL:** ставили TTL 90 дней — старые сырые события автоматически удалялись, агрегаты хранились дольше.

**Когда ClickHouse — плохой выбор:** если нужны UPDATE/DELETE по отдельным строкам, или OLTP-нагрузка с маленькими транзакциями. ClickHouse оптимизирован под аналитику (scan большого количества строк), а не под точечные операции.

---

## 4. "Как делали reconciliation крипто-кошельков в Darwin?"

> **Ключевые слова:** сверка балансов, eventual consistency, scheduled job

**Проблема:** баланс в нашей БД может разойтись с реальным балансом в блокчейне из-за реорг блока, задержки ноды, или бага в коде.

**Решение:**

1. Основной флоу: при каждой транзакции пишем в БД `pending` запись + публикуем событие.
2. Фоновый `reconciliation job` (cron раз в час) — запрашивал реальные балансы с блокчейн-ноды и сравнивал с нашей БД.
3. При расхождении — алерт в мониторинг, баланс помечался `disputed`, ручной разбор.
4. Для транзакций — слушали webhook от ноды о confirmation и обновляли статус с `pending` → `confirmed`.

**Ключевое:** мы не доверяли балансу на слово — always verify against source of truth (блокчейн).

---

## 5. "Какие индексы создавал для high-write таблиц в Darwin?"

> **Ключевые слова:** partial index, BRIN, partition pruning, покрывающий индекс

**Контекст:** таблица `trades` — история сделок, тысячи записей в минуту.

**Что делали:**

1. **Партиционирование по времени (RANGE)** — `trades_2023_01`, `trades_2023_02` и т.д. Запросы за конкретный месяц идут только в один partition, не сканируют всё.

2. **BRIN индекс на `created_at`** — для временных рядов, где данные физически упорядочены по времени. Занимает в сотни раз меньше места чем B-tree, эффективен для range-сканов.

3. **Partial index** для активных ордеров:
   ```sql
   CREATE INDEX idx_orders_active ON orders (user_id, created_at)
   WHERE status = 'active';
   ```
   Большинство запросов — именно по активным ордерам. Индекс маленький и быстрый.

4. **Покрывающий индекс (INCLUDE)** — чтобы не ходить в heap:
   ```sql
   CREATE INDEX idx_orders_user ON orders (user_id) INCLUDE (status, amount);
   ```

**Результат:** EXPLAIN ANALYZE показывал Seq Scan → Index Scan, время запросов упало в 5-10 раз на горячих путях.

---

## 6. "Расскажи о самой сложной технической проблеме" (STAR)

> **Используй:** проблему с медленными запросами в Sergek Group

**Situation:** платформа городского трафика начала деградировать под нагрузкой — среднее время ответа API выросло с 50мс до 2+ секунд в пиковые часы.

**Task:** найти и устранить узкое место, не останавливая систему.

**Action:**
1. Включил `pg_stat_statements`, нашёл топ-5 медленных запросов — один JOIN без индекса давал Seq Scan на таблице в 50М строк.
2. Добавил составной индекс — время запроса упало с 3s до 80ms.
3. Обнаружил второй bottleneck: consumer читал из RabbitMQ по одному сообщению и делал INSERT в БД на каждое. Переписал на batch processing с буфером в 500 сообщений, flush каждые 100ms.
4. Настроил Prometheus метрики на consumer lag и latency percentiles, чтобы видеть проблему заранее.

**Result:** среднее время ответа вернулось к 40-60ms, consumer lag упал до нуля, добавили алерт на lag > 1000 сообщений.

---

## 7. "Как работает GMP-планировщик в Go?"

> **Это базовый вопрос — должен ответить чётко**

Go использует M:N модель планирования: M горутин на N потоков ОС.

- **G (Goroutine)** — лёгковесный поток, начинает со стека 2-8KB, может вырасти до 1GB. Не привязан к ОС-потоку.
- **M (Machine)** — реальный поток ОС. Выполняет горутины.
- **P (Processor)** — логический процессор, очередь runnable горутин. По умолчанию `GOMAXPROCS` = числу CPU.

**Как работает:**
1. Каждый P имеет локальную очередь горутин (до 256 штук).
2. M берёт горутину из P и выполняет.
3. При системном вызове (I/O) — M блокируется, P уходит к другому M.
4. Work stealing: если у P пустая очередь — крадёт горутины у других P.

**Почему горутины дешевле потоков:** context switch горутины ~100ns vs ~1μs для ОС-потока. Стек растёт динамически. Можно запустить миллион горутин там, где потоков — только тысячи.

---

## 8. "Как использовал context.Context?"

> **Привяжи к реальному опыту из eMoney**

В eMoney каждый HTTP-запрос несёт свой `context` с `deadline`. Мы пробрасываем его через весь стек:

```go
// Handler → Service → Repository → DB
func (h *PaymentHandler) Create(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // уже содержит deadline от клиента

    result, err := h.svc.CreatePayment(ctx, req)
    // если клиент отвалился — ctx.Done() сработает,
    // и БД-запрос будет отменён автоматически
}
```

**WithTimeout для внешних вызовов:**
```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

resp, err := h.externalProvider.Charge(ctx, payment)
```

**Правило:** никогда не передаём `context.Background()` внутрь хендлера — только пробрасываем родительский. `context.Background()` — только в `main()` или при запуске фоновых джобов.

**Ловушка:** не забывать `defer cancel()` — иначе утечка горутин если goroutine висит и ждёт `ctx.Done()`.

---

## 9. "Kafka vs RabbitMQ — чем отличаются?"

> **У тебя есть опыт с обоими — скажи когда что использовал**

| | Kafka | RabbitMQ |
|---|---|---|
| Модель | Pull (consumer сам читает) | Push (broker толкает) |
| Хранение | Лог с retention (дни/недели) | Очередь (удаляется после ack) |
| Ordering | Гарантия в пределах партиции | Нет гарантии при конкурентных consumer |
| Throughput | Очень высокий (млн/сек) | Средний |
| Replay | Да (перечитать с любого offset) | Нет |

**В Darwin** использовали RabbitMQ для event-driven нотификаций (ордер исполнен → уведомить пользователя). Простая pub/sub логика, невысокая нагрузка.

**В Sergek** обрабатывали поток событий от камер — там важен высокий throughput и возможность перечитать события при сбое. Здесь Kafka была бы правильнее (если бы мигрировали).

**Когда Kafka:** аудит-лог, event sourcing, stream processing, высокая нагрузка.
**Когда RabbitMQ:** task queue, сложный routing (topic/fanout exchange), когда нужна гибкая маршрутизация.

---

## 10. "Как делал graceful shutdown?"

> **Это частая живая задача — знай наизусть**

```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: router}

    // Запускаем сервер в горутине
    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    // Ждём сигнал от ОС
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    <-quit

    log.Println("shutting down...")

    // Даём 30 секунд на завершение текущих запросов
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("forced shutdown:", err)
    }

    log.Println("server stopped")
}
```

**Почему SIGTERM, не SIGKILL:** SIGTERM — "пожалуйста, завершись корректно". SIGKILL — немедленное убийство процесса, нет шанса закрыть соединения/освободить ресурсы. Kubernetes посылает SIGTERM, ждёт `terminationGracePeriodSeconds`, потом SIGKILL.

---

## 11. "Как тестировал критические сервисы в Darwin?"

> **Ключевые слова:** testify, table-driven tests, mock интерфейсов

**Структура тестов:**

```go
func TestCreateOrder(t *testing.T) {
    tests := []struct {
        name    string
        input   CreateOrderReq
        mockFn  func(repo *MockOrderRepo)
        wantErr bool
    }{
        {
            name:  "success",
            input: CreateOrderReq{UserID: 1, Amount: 100},
            mockFn: func(repo *MockOrderRepo) {
                repo.On("Save", mock.Anything, mock.AnythingOfType("Order")).
                    Return(nil)
            },
        },
        {
            name:    "insufficient balance",
            input:   CreateOrderReq{UserID: 1, Amount: 999999},
            wantErr: true,
            mockFn:  func(repo *MockOrderRepo) {}, // не должен дойти до repo
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := new(MockOrderRepo)
            tt.mockFn(repo)

            svc := NewOrderService(repo)
            err := svc.Create(context.Background(), tt.input)

            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
            }
            repo.AssertExpectations(t)
        })
    }
}
```

**Что тестировали в первую очередь:** business logic сервисного слоя (через моки), reconciliation логику, edge cases с балансами (нулевой, отрицательный).

**Integration тесты:** поднимали тестовую БД через Docker Compose в CI, прогоняли CRUD операции против реальной схемы. Это поймало баг с миграцией который unit тесты пропустили.

---

## 12. "Что такое data race и как обнаружить?"

```go
// Пример data race:
var counter int

go func() { counter++ }()
go func() { counter++ }()
// Чтение и запись из разных горутин без синхронизации — UB
```

**Как обнаружить:**
```bash
go test -race ./...
go run -race main.go
```

Race detector добавляет ~5-10x замедление, используем только в dev/CI, не в production.

**Как чинить:**
- `sync.Mutex` / `sync.RWMutex` для защиты shared state
- `sync/atomic` для счётчиков и флагов
- Channels — "share memory by communicating"

---

## 13. "Уровни изоляции транзакций PostgreSQL"

> **Базовый вопрос для любого backend-разработчика**

| Уровень | Dirty Read | Non-repeatable Read | Phantom Read |
|---------|-----------|---------------------|--------------|
| Read Committed (default) | ✗ | возможно | возможно |
| Repeatable Read | ✗ | ✗ | возможно |
| Serializable | ✗ | ✗ | ✗ |

**Read Committed** — видим только закоммиченные данные, но два SELECT в одной транзакции могут вернуть разный результат (другая транзакция успела закоммитить между ними).

**Repeatable Read** — снимок данных берётся на начало транзакции, повторный SELECT вернёт то же самое.

**Serializable** — полная изоляция, как будто транзакции выполняются последовательно. Максимальный overhead, может завершиться с ошибкой сериализации.

**В eMoney:** используем `Read Committed` для большинства операций, `Repeatable Read` при критических финансовых операциях (списание + проверка баланса в одной транзакции).

---

## 14. "Как настраивал Prometheus/Grafana в eMoney?"

**Метрики которые добавляли:**

```go
var (
    paymentDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "payment_processing_duration_seconds",
            Buckets: []float64{0.01, 0.05, 0.1, 0.5, 1, 5},
        },
        []string{"provider", "status"},
    )

    paymentTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "payments_total"},
        []string{"status"},
    )
)
```

**Дашборды:** смотрели на p50/p95/p99 latency, error rate, consumer lag для очередей. Алерты на p99 > 2s и error_rate > 1%.

**Structured logging:** `slog` (или `zap`) с полями `request_id`, `user_id`, `payment_id` — чтобы трейсить конкретный платёж через все логи.

---

---

## 15. "Расскажи про высоконагруженный проект который ты делал" (STAR)

> **Используй:** batch consumer в Sergek + платёжный сервис в eMoney

**Situation:** В Sergek Group система мониторинга трафика получала события от 200+ камер. Каждая камера слала 5-10 событий в секунду → 1000-2000 событий/сек в пике. Consumer читал из очереди и делал INSERT в ClickHouse на каждое событие.

**Task:** система начала отставать — consumer lag рос, ClickHouse давал ошибки too many parts (признак частых мелких вставок).

**Action:**
1. Измерил: через Prometheus смотрел consumer lag и latency вставок. pprof показал что 90% CPU — в syscall от частых мелких write.
2. Переписал consumer на батч-обработку: буфер `[]Event` на 500 элементов, flush по таймеру (100ms) или по заполнению. Один `INSERT INTO ... VALUES (batch)` вместо 500 одиночных.
3. Добавил backpressure через buffered channel: если буфер флашера заполнен — consumer замедляется (Go channel backpressure бесплатно).
4. Настроил алерт на lag > 5000 сообщений и на время флаша > 200ms.

**Result:** consumer lag упал с 50k до <500 сообщений. ClickHouse CPU с 80% до 15%. Throughput вырос с 200 событий/сек до 2000+ событий/сек.

```go
// Схема batch consumer
type BatchConsumer struct {
    queue   <-chan Event
    flusher chan []Event
    buf     []Event
    maxSize int
    ticker  *time.Ticker
}

func (c *BatchConsumer) run(ctx context.Context) {
    for {
        select {
        case event := <-c.queue:
            c.buf = append(c.buf, event)
            if len(c.buf) >= c.maxSize {
                c.flush()
            }
        case <-c.ticker.C:
            if len(c.buf) > 0 {
                c.flush()
            }
        case <-ctx.Done():
            c.flush()  // финальный flush при shutdown
            return
        }
    }
}
```

---

## 16. "Как профилировал и оптимизировал Go приложение?"

> **Ключевые слова:** pprof, escape analysis, sync.Pool, benchmarks

**Процесс который использую:**

1. **Измерить сначала** — запустить pprof CPU профиль под реальной нагрузкой (не microbenchmark):
```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
(pprof) top10 -cum   # топ по cumulative time
(pprof) list hotFunc # построчный анализ
(pprof) web          # flamegraph
```

2. **Найти аллокации** — heap профиль + benchmarks с `-benchmem`:
```bash
go test -bench=. -benchmem -memprofile=mem.prof
go tool pprof mem.prof
# ищем inuse_objects — живые объекты, не просто аллокации
```

3. **Escape analysis** — понять что улетает в heap:
```bash
go build -gcflags="-m" ./... 2>&1 | grep "escapes to heap"
```

**Что реально оптимизировал:**

- **Убрал лишние аллокации в hot path** — парсер JSON-событий создавал `map[string]interface{}` на каждый запрос. Заменил на typed struct → 0 аллокаций на горячем пути, -40% CPU.

- **sync.Pool для буферов** — HTTP handler создавал `[]byte` буфер для сборки ответа. Добавил sync.Pool → аллокации в этом хендлере упали с 3/op до 0/op:
```go
var bufPool = sync.Pool{New: func() interface{} { return make([]byte, 0, 4096) }}
buf := bufPool.Get().([]byte)[:0]
defer bufPool.Put(buf)
```

- **Предаллокация слайсов** — `make([]T, 0, expectedSize)` там где размер известен → избавились от N реаллокаций при append.

- **strings.Builder вместо конкатенации** — в генераторе SQL-запросов была конкатенация строк в цикле → каждая итерация аллокация. Builder с одним Grow(n) → 1 аллокация.

---

## 17. "Как понимаешь работу Go под капотом?"

> **Покажи глубину: GMP, memory allocator, GC**

**GMP-планировщик:**
Go использует M:N модель — M горутин на N OS потоков. P (processor) — это очередь runnable горутин, количество P = GOMAXPROCS (по умолчанию = CPU ядра). При блокирующем syscall M "паркуется", P уходит к другому M — OS поток не простаивает. Work stealing: пустой P крадёт половину очереди соседнего P.

**Memory allocator:**
tcmalloc-подобный, три уровня: mcache (per-P, без блокировок) → mcentral (с блокировкой) → mheap (mmap из ОС). Объекты по size classes (8, 16, 24 ... 32768 байт). Большие (>32KB) — сразу mheap. Горячий путь аллокации — только атомарная операция в mcache.

**Почему это важно на практике:**
- GOMAXPROCS в контейнере надо ставить по cpu.quota, не NumCPU → `go.uber.org/automaxprocs`
- GOMEMLIMIT (Go 1.19+) — ставить 90% от container memory limit чтобы GC успевал до OOM
- Много горутин в syscall → M потоков растёт → контекст свитчи OS → `GOMAXPROCS` не помогает → нужен netpoller (async I/O)

---

## 18. "Как работал с OS-уровнем в production?"

> **Для ролей где требуют знание OS**

В работе сталкивался с несколькими OS-уровневыми аспектами:

**Процессы и syscalls:**
- Go runtime использует epoll (Linux) для async I/O — горутины паркуются без блокировки OS потока. Это позволяет держать тысячи goroutine на 8 потоках.
- Настраивал `ulimit -n` (открытые файловые дескрипторы) для сервисов с большим числом соединений. По умолчанию 1024 — убивает при >1000 concurrent connections.
- Graceful shutdown через `SIGTERM` — Kubernetes посылает SIGTERM, потом через terminationGracePeriodSeconds — SIGKILL. Важно поймать SIGTERM и завершить текущие запросы.

**Память:**
- OOM Killer в Kubernetes убивал наш сервис. Диагностика: `dmesg | grep -i "killed process"`. Решение: правильный GOMEMLIMIT + memory request/limit в K8s манифесте.
- mmap для чтения больших файлов конфигурации/сертификатов — zero-copy, ядро подгружает страницы лениво.

**Сеть:**
- `ss -tnp` и `netstat` для диагностики соединений. Видел кейс с накоплением TIME_WAIT при высоком трафике — решается через `SO_REUSEADDR` и keep-alive.
- tcpdump в production для поимки странного поведения внешнего провайдера — снял pcap, открыл в Wireshark, увидел что провайдер присылает RST через 30 секунд бездействия. Решение: HTTP keep-alive с `IdleTimeout < 30s`.

---

## 19. "Как проектировал API для высоконагруженной системы?"

> **Ключевые слова:** идемпотентность, rate limiting, connection pooling, timeouts**

В eMoney проектировал payment API, где нагрузка до 500 RPS в пике:

**Идемпотентность:** каждый платёжный запрос несёт `Idempotency-Key` в заголовке. Храним в Redis с TTL 24h. При дубликате — возвращаем cached response. Это критично при retry логике клиентов.

**Таймауты везде:**
```go
// Внешний провайдер платежей
ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
defer cancel()

// HTTP клиент
client := &http.Client{
    Transport: &http.Transport{
        MaxIdleConnsPerHost: 50,        // connection pool per host
        IdleConnTimeout:     90*time.Second,
        TLSHandshakeTimeout: 5*time.Second,
    },
    Timeout: 15 * time.Second,
}
```

**Rate limiting:** token bucket на уровне user_id через Redis Lua script. 100 req/min per user, burst 20. Возвращаем 429 с `Retry-After` хедером.

**Circuit breaker для внешних провайдеров:** если провайдер отвечает с ошибкой 5 раз подряд — открываем circuit на 30 секунд, не шлём запросы, возвращаем клиенту fail-fast. Использовали `github.com/sony/gobreaker`.

**Connection pooling к БД:**
```go
db.SetMaxOpenConns(50)        // не больше 50 соединений
db.SetMaxIdleConns(10)        // держать 10 в idle
db.SetConnMaxLifetime(5 * time.Minute)  // переоткрывать соединения
```

---

## Быстрые ответы на стандартные вопросы

**— Почему Go, а не Java/Python?**
Статическая типизация + скомпилированный бинарь + горутины из коробки. Для микросервисов: маленький Docker image (scratch + бинарь), быстрый старт. Я в нём 4+ лет — это моя основная специализация.

**— Что такое nil interface trap?**
```go
var err *MyError = nil
var iface error = err
fmt.Println(iface == nil) // false! интерфейс содержит type=*MyError, value=nil
```
Интерфейс == nil только если и type, и value равны nil.

**— Чем отличается make от new?**
`new(T)` — выделяет память, возвращает `*T` (указатель на нулевое значение).
`make([]T, n)` — создаёт slice/map/chan с инициализацией внутренних структур. Для этих типов нельзя использовать `new`.

**— Что такое escape analysis?**
Компилятор анализирует, переживёт ли переменная свою функцию. Если да — уходит в heap (GC за ней следит). Если нет — остаётся в stack (автоматически освобождается). `go build -gcflags="-m"` покажет что куда ушло.
