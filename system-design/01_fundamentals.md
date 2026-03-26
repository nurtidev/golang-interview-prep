# Урок 1: Основы проектирования систем

## Свойства информационных систем

Любую систему можно оценить по четырём ключевым свойствам.

**Scalability** — способность системы справляться с ростом нагрузки без деградации. Масштабируют вертикально (мощнее машина) или горизонтально (больше машин). Вертикальное — быстрый костыль, горизонтальное — долгосрочное решение.

**Performance** — это два числа: latency (время ответа на один запрос) и throughput (сколько запросов в секунду обрабатывает система). Они часто противоречат друг другу: батчинг увеличивает throughput, но увеличивает latency.

**Maintainability** — насколько легко систему поддерживать: читать код, вносить изменения, отлаживать инциденты. Распадается на operability, simplicity и evolvability.

**Security** — аутентификация, авторизация, шифрование данных в покое и в движении, защита от атак.

---

## Классификация систем

Прежде чем проектировать систему, нужно понять её характер — это определяет все архитектурные решения.

| Тип | Узкое место | Примеры |
|-----|------------|---------|
| **Data-intensive** | Объём / хранение данных | S3, HDFS, Kafka |
| **Compute-intensive** | CPU | ML-инференс, кодирование видео |
| **Read-intensive** | Читают >> пишут | Twitter-лента, Wikipedia |
| **Write-intensive** | Пишут >> читают | IoT-метрики, логи |
| **Low-latency** | Время ответа критично | Trading, игры, чаты |
| **High-throughput** | Важен суммарный поток, не скорость | Аналитика, batch jobs |

> Система может быть одновременно read-intensive и low-latency — это разные измерения.

---

## Требования: функциональные и нефункциональные

На собесе по System Design первые 5–10 минут — это уточнение требований. Никогда не начинай рисовать архитектуру, не зная масштаба.

**Функциональные** — что система делает:
- Пользователь может загрузить фото
- Система отправляет уведомление при новом сообщении

**Нефункциональные** — как система это делает:
- 99.9% availability (= 8.7 часов даунтайма в год)
- P99 latency < 200 мс
- 10 млн DAU
- Хранить данные 5 лет

**Шаблон вопросов на собесе:**
```
1. Масштаб: DAU? RPS? Объём данных?
2. Availability SLA: 99.9% или 99.99%?
3. Consistency или Availability важнее? (CAP)
4. Географическое распределение?
5. Есть ли пики нагрузки (10x в праздники)?
```

---

## Расчёт нагрузки (Back-of-the-envelope)

Это обязательный навык на System Design интервью. Интервьюер хочет видеть, что ты понимаешь порядок цифр.

**Полезные константы:**
```
1 день  = 86 400 сек ≈ 100K сек
1 год   = ~30M сек

1 KB = 10³ байт  |  1 MB = 10⁶  |  1 GB = 10⁹  |  1 TB = 10¹²
```

**Пример: социальная сеть, 10M DAU**
```
Среднее RPS  = 10M * 10 запросов / 86 400 ≈ 1 200 RPS
Пиковый RPS  = 1 200 * 3                  ≈ 3 600 RPS  ← правило x3

Фото: 10% DAU постят фото, 1 фото = 200 KB
Write throughput = 1M * 200 KB / 86 400   ≈ 2.3 GB/сек
Хранение за год  = 1M * 200 KB * 365      ≈ 73 PB  ← надо сжимать
```

---

## Балансировка нагрузки

Балансировщик — это первое, что встречает входящий трафик. Без него один сервер станет узким местом и точкой отказа.

### L4 vs L7

L4-балансировщик работает на уровне TCP/UDP — он видит только IP-адреса и порты, не смотрит в тело запроса. Это делает его очень быстрым, но функционально ограниченным.

L7-балансировщик парсит HTTP — он видит URL, заголовки, cookies. Это даёт гибкость: роутинг по `/api/v2/`, A/B тестирование, SSL termination, аутентификация.

```
Клиент → [L7 LB: Nginx] → [Backend 1]
                        → [Backend 2]
                        → [Backend 3]
         ↑
  SSL termination здесь,
  бэкенды получают plain HTTP — им не нужны сертификаты
```

| | L4 | L7 |
|---|---|---|
| Смотрит на | IP + TCP/UDP | HTTP headers, URL, body |
| Скорость | быстрее | медленнее (парсинг) |
| SSL termination | нет | да |
| Примеры | AWS NLB, HAProxy TCP | Nginx, AWS ALB, Envoy |

### DNS-балансировка и GeoDNS

DNS Round Robin — самый дешёвый способ балансировки: один домен резолвится в несколько IP-адресов, клиент получает следующий по кругу.

```
api.example.com → [1.1.1.1, 2.2.2.2, 3.3.3.3]
```

GeoDNS идёт дальше — отвечает разными IP в зависимости от геолокации клиента:
```
Европа → eu-servers.example.com
Азия   → as-servers.example.com
```

**Проблема DNS-балансировки:** TTL кэширует IP на клиенте. Если сервер упал — клиент будет ходить к нему до истечения TTL (минуты или часы).

### Алгоритмы балансировки

| Алгоритм | Когда использовать |
|----------|--------------------|
| **Round Robin** | Однородные серверы и запросы |
| **Weighted RR** | Серверы разной мощности |
| **Least Connections** | Долгие запросы: WebSocket, стриминг |
| **Least Response Time** | Latency сильно варьируется |
| **Sticky Sessions** | Stateful приложения (хранят сессию в памяти) |
| **Power of Two Choices** | Берём двух случайных, выбираем лучшего — хорошее качество при низком overhead |

---

## Проксирование

```
Forward Proxy                       Reverse Proxy
Клиент → [Proxy] → Интернет         Интернет → [Proxy] → Серверы

Скрывает клиента от сервера         Скрывает серверы от клиента
Примеры: VPN, корп. фильтр          Примеры: Nginx, CDN, API Gateway
```

Reverse proxy берёт на себя: SSL termination, кэширование статики, rate limiting, балансировку, защиту от DDoS — бэкенды работают с простым HTTP за закрытым периметром.

---

## Вопросы с собеса

**В чём разница между горизонтальным и вертикальным масштабированием?**

<details>
<summary>Ответ</summary>

**Вертикальное (Scale Up)** — добавляем ресурсы одной машине (RAM, CPU). Просто, без изменений в коде, но имеет физический потолок и остаётся single point of failure.

**Горизонтальное (Scale Out)** — добавляем новые машины. Теоретически безлимитно, отказоустойчиво, но требует stateless-сервисов, балансировки и усложняет операционку.

На практике: вертикальное — быстрый первый шаг, горизонтальное — долгосрочное решение.

</details>

---

**Как рассчитать сколько серверов нужно для 50 000 RPS?**

<details>
<summary>Ответ</summary>

1. Определяем пропускную способность одного сервера — например, Go-сервис на 4 CPU обрабатывает ~5 000 RPS (лёгкие запросы без IO)
2. 50 000 / 5 000 = 10 серверов
3. Добавляем запас на пики и rolling deploy: × 1.5–2 → **15–20 серверов**
4. Если нужен hot standby — ещё +N

Всегда уточнять: CPU-bound или IO-bound запросы? Есть кэш? Какой P99 latency требуется?

</details>

---

**Чем L4 балансировщик лучше L7 и когда его выбирать?**

<details>
<summary>Ответ</summary>

L4 не парсит HTTP — работает быстрее и с меньшей задержкой. Выбираем когда:
- Не-HTTP протоколы: gRPC поверх TCP, MQTT, кастомные бинарные протоколы
- Экстремально высокий throughput, каждая миллисекунда на счету
- Не нужна логика роутинга по содержимому запроса

L7 выбираем когда нужно: роутить по path/headers, делать SSL termination, A/B testing, аутентификацию на уровне прокси.

</details>

---

**Что такое sticky sessions и почему это антипаттерн?**

<details>
<summary>Ответ</summary>

Sticky sessions (session affinity) — балансировщик всегда отправляет запросы одного клиента на один сервер (по cookie или IP-хешу).

**Проблема:** если сервер падает — все его сессии теряются. Плюс неравномерная нагрузка: один сервер может получить много "тяжёлых" сессий.

**Правильное решение:** хранить состояние сессии не в памяти сервера, а в общем хранилище (Redis). Тогда любой сервер может обработать любой запрос — сервис становится stateless и sticky sessions не нужны.

</details>

---

## Шардирование и Consistent Hashing

### Зачем шардировать

Один сервер/БД имеет физический предел: RAM, CPU, дисковый I/O. При росте данных нужно делить нагрузку между несколькими узлами — это шардирование (горизонтальное партиционирование).

```
Без шардирования:          С шардированием:
[Один сервер]              [Shard 0] [Shard 1] [Shard 2] [Shard 3]
 10M users                  2.5M      2.5M       2.5M       2.5M
```

### Проблема с наивным хешированием

```
shard = hash(key) % N

N = 4 серверов:
key="user_1" → hash=1000 → 1000 % 4 = 0  → Shard 0
key="user_2" → hash=2001 → 2001 % 4 = 1  → Shard 1
```

**Проблема при добавлении/удалении сервера (N → N+1):**
```
N=4: key="user_1" → 1000 % 4 = 0  → Shard 0
N=5: key="user_1" → 1000 % 5 = 0  → Shard 0  (повезло)
     key="user_2" → 2001 % 5 = 1  → Shard 1  (совпало)
     key="user_3" → 3000 % 4 = 0, но 3000 % 5 = 0 ...

В среднем: ~(N-1)/N = 75% ключей меняют свой шард!
→ Огромный resharding, все данные нужно переносить
```

---

### Consistent Hashing — решение

**Идея:** и ключи, и узлы располагаются на одном "кольце" хешей (0 до 2^32-1). Ключ принадлежит ближайшему узлу по часовой стрелке.

```
          0
     ┌────┴────┐
  3/4│         │1/4
     │    ○    │
  Node C    Node A
     │         │
  1/2└────┬────┘
          │
        Node B

key → hash(key) → ищем ближайший узел по часовой стрелке

hash(key) = 0.1 → Node A (ближайший по часовой)
hash(key) = 0.4 → Node B
hash(key) = 0.8 → Node C
```

**При добавлении Node D (между B и C):**
```
До:   ключи 0.5-0.75 → Node C
После: ключи 0.5-0.6  → Node D (новый)
       ключи 0.6-0.75 → Node C (остались)

Перемещается только 1/N ключей — оптимально!
```

**При удалении Node B:**
```
Ключи B уходят к следующему узлу (Node C)
Остальные узлы не затронуты
```

---

### Virtual Nodes (vnodes)

**Проблема базового consistent hashing:** неравномерное распределение. Если узлы расположены неравномерно на кольце — один получит 60% ключей, другой 10%.

**Решение — виртуальные узлы:** каждый физический узел представлен множеством точек на кольце.

```
Node A (физический) → A-1, A-2, A-3, A-4, A-5  (виртуальные)
Node B (физический) → B-1, B-2, B-3, B-4, B-5
Node C (физический) → C-1, C-2, C-3, C-4, C-5

Кольцо: [A-3][B-1][C-2][A-1][B-4][C-5][A-5][B-2][C-1][A-2][B-3][C-4]
         ↑ перемешаны → равномерное распределение
```

**Количество vnodes:**
- Cassandra: 256 vnodes по умолчанию
- Redis Cluster: 16384 слота (hash slots), не vnodes
- Типично: 100-200 vnodes на узел

**Выгоды vnodes:**
1. Равномерное распределение независимо от количества узлов
2. При добавлении узла он забирает ключи от ВСЕХ существующих узлов (не только соседей)
3. Учёт мощности узлов: мощный узел получает больше vnodes

---

### Реализация в Go

```go
import (
    "crypto/md5"
    "fmt"
    "sort"
    "sync"
)

type ConsistentHash struct {
    mu       sync.RWMutex
    ring     map[uint32]string  // hash → node
    sorted   []uint32           // отсортированные хеши для binary search
    replicas int                // количество виртуальных узлов на один физический
}

func New(replicas int) *ConsistentHash {
    return &ConsistentHash{
        ring:     make(map[uint32]string),
        replicas: replicas,
    }
}

func (c *ConsistentHash) hash(key string) uint32 {
    h := md5.Sum([]byte(key))
    return uint32(h[0])<<24 | uint32(h[1])<<16 | uint32(h[2])<<8 | uint32(h[3])
}

func (c *ConsistentHash) Add(node string) {
    c.mu.Lock()
    defer c.mu.Unlock()

    for i := 0; i < c.replicas; i++ {
        virtualKey := fmt.Sprintf("%s#%d", node, i)
        h := c.hash(virtualKey)
        c.ring[h] = node
        c.sorted = append(c.sorted, h)
    }
    sort.Slice(c.sorted, func(i, j int) bool {
        return c.sorted[i] < c.sorted[j]
    })
}

func (c *ConsistentHash) Remove(node string) {
    c.mu.Lock()
    defer c.mu.Unlock()

    for i := 0; i < c.replicas; i++ {
        virtualKey := fmt.Sprintf("%s#%d", node, i)
        h := c.hash(virtualKey)
        delete(c.ring, h)
    }
    // Пересобираем sorted
    c.sorted = c.sorted[:0]
    for h := range c.ring {
        c.sorted = append(c.sorted, h)
    }
    sort.Slice(c.sorted, func(i, j int) bool {
        return c.sorted[i] < c.sorted[j]
    })
}

func (c *ConsistentHash) Get(key string) string {
    c.mu.RLock()
    defer c.mu.RUnlock()

    if len(c.ring) == 0 {
        return ""
    }

    h := c.hash(key)

    // Binary search: найти первый узел >= h
    idx := sort.Search(len(c.sorted), func(i int) bool {
        return c.sorted[i] >= h
    })

    // Если не нашли — берём первый (wrap around кольца)
    if idx == len(c.sorted) {
        idx = 0
    }

    return c.ring[c.sorted[idx]]
}

// Использование:
// ch := New(150)  // 150 vnodes на узел
// ch.Add("cache-server-1:6379")
// ch.Add("cache-server-2:6379")
// ch.Add("cache-server-3:6379")
//
// node := ch.Get("user:12345")  // → "cache-server-2:6379"
```

---

### Где используется

| Система | Реализация |
|---------|-----------|
| **Redis Cluster** | 16384 hash slots, consistent hashing по слотам |
| **Cassandra** | Virtual nodes (vnodes), 256 по умолчанию |
| **Amazon Dynamo** | Оригинальная статья 2007, consistent hashing + vnodes |
| **Memcached (клиентский)** | Ketama: consistent hashing на клиенте |
| **CDN (Nginx, Varnish)** | Consistent hashing для cache routing |
| **Kafka** | Партиции по ключу (не consistent hashing, но похожая идея) |

---

### Backpressure — управление давлением при высоком throughput

Когда producer быстрее consumer — очередь растёт → OOM. Backpressure — механизм сигнализации "притормози".

```
Producer → [Buffer/Queue] → Consumer

Без backpressure:
Producer: 100k msg/sec
Consumer: 50k msg/sec
Через 10s: очередь 500k сообщений → OOM

С backpressure:
Consumer заполнен → сигнал Producer'у → Producer замедляется / блокируется
```

**Паттерны backpressure в Go:**

```go
// 1. Блокирующий канал (простейший backpressure)
queue := make(chan Job, 1000)  // буфер 1000

// Producer блокируется при заполнении буфера:
queue <- job  // ← backpressure здесь

// 2. Drop с метрикой (сброс при перегрузке)
select {
case queue <- job:
    // отправлено
default:
    // очередь полна — дропаем и метрику
    droppedCounter.Inc()
    return ErrOverloaded
}

// 3. Semaphore для ограничения concurrent обработки
sem := make(chan struct{}, maxConcurrent)

func processRequest(req Request) error {
    sem <- struct{}{}        // acquire
    defer func() { <-sem }() // release

    return handle(req)
}

// 4. Rate limiting через token bucket
import "golang.org/x/time/rate"

limiter := rate.NewLimiter(rate.Limit(1000), 100)  // 1000 rps, burst 100

func handleRequest(ctx context.Context, req Request) error {
    if err := limiter.Wait(ctx); err != nil {
        return fmt.Errorf("rate limited: %w", err)
    }
    return process(req)
}
```

**Backpressure в распределённых системах:**
```
HTTP:    503 Service Unavailable + Retry-After header
gRPC:    RESOURCE_EXHAUSTED status code
Kafka:   Producer max.block.ms — блокировка при заполнении буфера
TCP:     Zero Window — встроенный backpressure на уровне протокола
```

---

### Rate Limiting — алгоритмы

| Алгоритм | Принцип | Burst | Использование |
|----------|---------|-------|---------------|
| **Token Bucket** | Токены накапливаются с фиксированной скоростью | Да (burst = ёмкость bucket) | Большинство API rate limiter |
| **Leaky Bucket** | Запросы обрабатываются с фиксированной скоростью | Нет | Сглаживание трафика |
| **Fixed Window** | Счётчик сбрасывается каждые N секунд | Да (граница окна) | Простые API quotas |
| **Sliding Window Log** | Точный лог всех запросов | Нет | Точный rate limit, дорогой по памяти |
| **Sliding Window Counter** | Fixed window × два, интерполяция | Небольшой | Redis rate limiter, баланс точность/память |

```go
// Token Bucket в Redis (атомарный через Lua)
const luaTokenBucket = `
local key = KEYS[1]
local rate = tonumber(ARGV[1])      -- токенов в секунду
local capacity = tonumber(ARGV[2])  -- ёмкость bucket
local now = tonumber(ARGV[3])       -- текущее время (unix ms)
local requested = tonumber(ARGV[4]) -- нужно токенов

local last_time = tonumber(redis.call('hget', key, 'last_time') or now)
local tokens = tonumber(redis.call('hget', key, 'tokens') or capacity)

-- Пополнить токены
local elapsed = (now - last_time) / 1000.0
tokens = math.min(capacity, tokens + elapsed * rate)

if tokens >= requested then
    tokens = tokens - requested
    redis.call('hset', key, 'tokens', tokens, 'last_time', now)
    redis.call('expire', key, math.ceil(capacity / rate) + 1)
    return 1  -- разрешить
else
    return 0  -- отказать
end
`
```

---

### Вопросы на собеседовании

**Q: Почему consistent hashing лучше modulo hashing при горизонтальном масштабировании?**

**Ответ:** При modulo hashing (`key % N`) добавление/удаление узла инвалидирует (N-1)/N ≈ 75-80% маппингов — почти все данные нужно перераспределять. При consistent hashing меняется только ~1/N ключей — только те, что принадлежали удалённому узлу или должны принадлежать новому. Virtual nodes добавляют равномерность распределения: каждый узел представлен 100-300 точками на кольце, что нивелирует случайность позиций. Это критично для кешей (Redis Cluster, Memcached) — минимальный resharding = минимальный cache miss spike при масштабировании.

---

**Q: Что такое backpressure? Как реализовать в Go микросервисе?**

**Ответ:** Backpressure — механизм передачи сигнала "я перегружен" от consumer к producer. Без него: producer генерирует быстрее consumer обрабатывает → очередь растёт → OOM или деградация. В Go: блокирующий buffered channel (`queue <- job` блокируется при заполнении), semaphore pattern (`make(chan struct{}, N)` ограничивает concurrent обработку), rate limiter (`golang.org/x/time/rate`). На уровне HTTP — возвращать 503 с `Retry-After`, gRPC — `RESOURCE_EXHAUSTED`. В Kafka consumer — не читать быстрее чем обрабатываешь (pull модель даёт backpressure бесплатно). Главное: дропать запросы gracefully с метрикой, а не накапливать до OOM.
