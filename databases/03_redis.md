# Redis: Архитектура и паттерны использования

---

## Лекция

### 1. Что такое Redis и как он устроен

Redis — in-memory хранилище данных. Все данные хранятся в RAM → операции очень быстрые (100k+ ops/sec). Сохранение на диск опционально.

**Ключевые свойства:**
- Однопоточная модель выполнения команд (нет race conditions внутри Redis)
- Команды атомарны по умолчанию
- Поддерживает несколько структур данных, не только key-value
- Pub/Sub, Streams, Lua скрипты, транзакции

**Когда использовать Redis:**
- Кеширование (самый частый кейс)
- Сессии пользователей
- Rate limiting
- Распределённые блокировки (distributed locks)
- Очереди задач
- Лидерборды (sorted sets)
- Pub/Sub сообщения

---

### 2. Основные структуры данных

**String** — базовый тип. Строка, число, бинарные данные. Max 512MB.
```
GET key, SET key value, INCR key, EXPIRE key seconds
```

**List** — двусвязный список строк. Порядок вставки сохраняется.
```
LPUSH (добавить в начало), RPUSH (в конец)
LPOP, RPOP (извлечь из начала/конца)
LRANGE key 0 -1 (получить все элементы)
Использование: очереди, стеки, последние N событий
```

**Hash** — хэш-таблица полей. Как объект.
```
HSET user:1 name "Alice" age 30
HGET user:1 name
HGETALL user:1
Использование: объекты, профили пользователей
```

**Set** — неупорядоченное множество уникальных строк.
```
SADD, SREM, SISMEMBER, SMEMBERS
SUNION, SINTER, SDIFF (операции над множествами)
Использование: теги, уникальные посетители, друзья пользователя
```

**Sorted Set (ZSet)** — множество с float score для сортировки.
```
ZADD leaderboard 100 "Alice"
ZRANGE leaderboard 0 -1 WITHSCORES   -- по возрастанию
ZREVRANGE leaderboard 0 9            -- top 10
ZRANK leaderboard "Alice"            -- позиция
Использование: лидерборды, очереди с приоритетом, временные окна
```

**Bitmap** — битовая карта. Эффективно для больших множеств флагов.
```
SETBIT user:logins:2024-01 42 1  -- user 42 вошёл в январе
BITCOUNT user:logins:2024-01     -- сколько уникальных пользователей вошло
Использование: ежедневная активность, A/B тесты
```

**HyperLogLog** — вероятностная структура для подсчёта уникальных значений.
```
PFADD page_views:home user1 user2 user3
PFCOUNT page_views:home  -- ~точное, но не 100%
Использование: подсчёт уникальных просмотров с ~1% погрешностью, занимает только 12KB
```

---

### 3. Персистентность

Redis предлагает два механизма сохранения данных:

**RDB (Redis Database) — снапшоты:**
- Периодически сохраняет дамп всех данных на диск (`.rdb` файл)
- Настраивается: "сохранять каждые 60 секунд если изменилось > 1000 ключей"
- Быстрый рестарт (загрузить один файл)
- Минус: при сбое теряются данные за интервал между снапшотами

**AOF (Append Only File) — журнал:**
- Каждая команда записи добавляется в файл
- Варианты: `always` (записать до ответа клиенту), `everysec` (раз в секунду), `no` (ОС решает)
- Потеря данных: max 1 секунда при `everysec`
- Минус: файл растёт, периодически нужна перезапись (AOF rewrite)

**Рекомендация:** Использовать оба: RDB для быстрого восстановления, AOF для минимальной потери данных.

---

### 4. Репликация и кластер

**Master-Replica репликация:**
- Реплики асинхронно получают все команды от мастера
- Реплики read-only (по умолчанию)
- Автоматический failover через Redis Sentinel

**Redis Sentinel:**
- Мониторит мастер и реплики
- При падении мастера → выбирает новый мастер из реплик
- Обновляет клиентов о новом мастере

**Redis Cluster:**
- Горизонтальное масштабирование
- Данные шардируются по 16384 слотам (hash slots)
- `HASH_SLOT = CRC16(key) % 16384`
- Каждый мастер-узел отвечает за диапазон слотов
- Минимум: 3 мастера + 3 реплики

---

### 5. Паттерны кеширования

**Cache-Aside (Lazy Loading):**
```
Чтение: нет в кеше → читаем из БД → кладём в кеш → возвращаем
Запись: пишем в БД → инвалидируем кеш (или просто пишем в кеш)
```
- Самый популярный паттерн
- Минус: Cache Miss = 2 запроса (в Redis + в БД)

**Write-Through:**
```
Запись: пишем в кеш → кеш пишет в БД
```
- Данные всегда консистентны
- Минус: задержка при записи

**Write-Behind (Write-Back):**
```
Запись: пишем в кеш → асинхронно кеш пишет в БД
```
- Быстрая запись
- Риск потери данных при падении

---

### 6. Distributed Lock (Распределённая блокировка)

```
SET lock_key "owner_id" NX PX 30000
```
- `NX` — установить только если ключ не существует
- `PX 30000` — TTL 30 секунд (защита от deadlock)

**Redlock алгоритм** (для высокой надёжности):
- Попытаться захватить блокировку на N/2+1 независимых Redis узлах
- Если захвачено большинство — блокировка получена

---

### 7. Eviction Policies (политики вытеснения)

Когда Redis достигает `maxmemory`, он должен что-то удалять:

| Политика | Поведение |
|----------|-----------|
| `noeviction` | Возвращает ошибку при записи (по умолчанию) |
| `allkeys-lru` | Удаляет наименее недавно использованные ключи |
| `volatile-lru` | LRU только среди ключей с TTL |
| `allkeys-lfu` | Удаляет наименее часто используемые |
| `allkeys-random` | Случайные ключи |
| `volatile-ttl` | Ключи с наименьшим TTL |

Для кеша рекомендуется `allkeys-lru` или `allkeys-lfu`.

---

### 8. Pub/Sub и Streams

**Pub/Sub:**
```
SUBSCRIBE channel      -- подписаться
PUBLISH channel msg    -- опубликовать
```
- Fire-and-forget: если получатель не слушает — сообщение теряется
- Нет истории сообщений
- Подходит для real-time уведомлений

**Streams (Redis 5.0+):**
- Персистентный лог сообщений с группами потребителей
- Как Kafka, но в Redis
- `XADD stream * field value` — добавить
- `XREAD COUNT 10 STREAMS stream 0` — читать
- Подходит для event sourcing, аудит лога

---

## Практические примеры

### Пример 1: Кеш с Go (go-redis)

```go
import "github.com/redis/go-redis/v9"

type UserCache struct {
    client *redis.Client
    db     UserRepository
}

func (c *UserCache) GetUser(ctx context.Context, id int64) (*User, error) {
    key := fmt.Sprintf("user:%d", id)

    // Пробуем кеш
    data, err := c.client.Get(ctx, key).Bytes()
    if err == nil {
        var user User
        json.Unmarshal(data, &user)
        return &user, nil
    }
    if err != redis.Nil {
        // Ошибка Redis — логируем, но продолжаем (fallback к БД)
        log.Printf("redis get error: %v", err)
    }

    // Cache Miss — идём в БД
    user, err := c.db.GetByID(ctx, id)
    if err != nil {
        return nil, err
    }

    // Кладём в кеш на 1 час
    data, _ = json.Marshal(user)
    c.client.Set(ctx, key, data, time.Hour)

    return user, nil
}

func (c *UserCache) InvalidateUser(ctx context.Context, id int64) {
    c.client.Del(ctx, fmt.Sprintf("user:%d", id))
}
```

### Пример 2: Rate Limiter (скользящее окно)

```go
// Ограничение: max 100 запросов за 60 секунд
func (rl *RateLimiter) Allow(ctx context.Context, userID string) (bool, error) {
    key := fmt.Sprintf("ratelimit:%s", userID)
    now := time.Now().UnixMilli()
    windowMs := int64(60 * 1000) // 60 секунд
    limit := 100

    pipe := rl.client.Pipeline()
    // Удалить устаревшие записи
    pipe.ZRemRangeByScore(ctx, key, "0", fmt.Sprintf("%d", now-windowMs))
    // Добавить текущий запрос
    pipe.ZAdd(ctx, key, redis.Z{Score: float64(now), Member: now})
    // Посчитать запросы в окне
    countCmd := pipe.ZCard(ctx, key)
    // Установить TTL
    pipe.Expire(ctx, key, 70*time.Second)

    _, err := pipe.Exec(ctx)
    if err != nil {
        return true, err // при ошибке Redis — пропускаем (fail open)
    }

    return countCmd.Val() <= int64(limit), nil
}
```

### Пример 3: Distributed Lock

```go
type RedisLock struct {
    client *redis.Client
    key    string
    value  string // уникальный ID владельца
    ttl    time.Duration
}

func (l *RedisLock) Acquire(ctx context.Context) (bool, error) {
    l.value = uuid.New().String()
    ok, err := l.client.SetNX(ctx, l.key, l.value, l.ttl).Result()
    return ok, err
}

// Release только если мы владельцы (атомарно через Lua)
var releaseScript = redis.NewScript(`
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    else
        return 0
    end
`)

func (l *RedisLock) Release(ctx context.Context) error {
    return releaseScript.Run(ctx, l.client, []string{l.key}, l.value).Err()
}
```

### Пример 4: Лидерборд (Sorted Set)

```go
// Обновить счёт пользователя
func (lb *Leaderboard) UpdateScore(ctx context.Context, userID string, score float64) error {
    return lb.client.ZAdd(ctx, "leaderboard:global", redis.Z{
        Score:  score,
        Member: userID,
    }).Err()
}

// Получить топ-10
func (lb *Leaderboard) GetTop10(ctx context.Context) ([]redis.Z, error) {
    return lb.client.ZRevRangeWithScores(ctx, "leaderboard:global", 0, 9).Result()
}

// Получить ранг пользователя
func (lb *Leaderboard) GetRank(ctx context.Context, userID string) (int64, error) {
    rank, err := lb.client.ZRevRank(ctx, "leaderboard:global", userID).Result()
    return rank + 1, err // rank 0-based, возвращаем 1-based
}
```

---

## Вопросы на собеседовании

### Q: Почему Redis так быстр? В чём его ограничения?

**Ответ:** Redis хранит все данные в RAM — операции идут со скоростью памяти, без дисковых I/O. Однопоточная модель выполнения команд — нет overhead на блокировки. Простые структуры данных с оптимизированными операциями. Ограничения: объём данных ограничен RAM, при падении возможна потеря данных (зависит от персистентности), однопоточность означает что одна тяжёлая операция (например `KEYS *`) блокирует всё.

---

### Q: В чём разница между RDB и AOF?

**Ответ:** RDB создаёт периодические снапшоты всей базы — быстрый рестарт, но при сбое теряются данные за интервал (минуты). AOF логирует каждую команду записи — при `everysec` теряем максимум 1 секунду данных, но файл большой и рестарт медленнее. На практике используют оба: RDB для быстрого восстановления, AOF для минимальной потери данных.

---

### Q: Как реализовать распределённую блокировку в Redis?

**Ответ:** `SET key value NX PX 30000` — атомарно устанавливает ключ только если не существует, с TTL 30 секунд. Value должен быть уникальным (UUID) — чтобы при освобождении мы удаляли только свою блокировку. Освобождение — Lua скрипт: проверить что value совпадает и удалить атомарно. TTL защищает от deadlock если держатель упал. Для высокой надёжности — Redlock: захватить блокировку на большинстве из N независимых Redis узлов.

---

### Q: Что такое Cache Stampede и как с ним бороться?

**Ответ:** Cache Stampede (thundering herd) — когда много запросов одновременно получают Cache Miss (например, истёк TTL популярного ключа) и все идут в БД. БД может не выдержать нагрузку. Решения: 1) Probabilistic Early Reexpiration — обновлять кеш случайно перед истечением TTL. 2) Distributed lock на обновление — только один поток обновляет, остальные ждут. 3) Stale-while-revalidate — возвращать устаревшие данные, обновлять асинхронно. 4) Jitter к TTL — добавлять случайный offset к TTL чтобы ключи не истекали одновременно.

---

### Q: Когда использовать Redis Cluster? Как данные распределяются?

**Ответ:** Cluster нужен когда данные не помещаются в RAM одной машины или нужна горизонтальная масштабируемость. Данные шардируются по 16384 hash slots: `slot = CRC16(key) % 16384`. Каждый мастер-узел отвечает за диапазон слотов. Минимальная конфигурация: 3 мастера + 3 реплики. Ограничение: multi-key операции работают только если все ключи в одном слоте (используй hash tags: `{user:42}:profile`, `{user:42}:settings`).

---

### Q: Чем Sorted Set отличается от Set? Как он реализован?

**Ответ:** Set — неупорядоченное множество уникальных строк. Sorted Set — каждый элемент имеет float score, элементы упорядочены по score. Реализован как skip list (для быстрого range-запроса) + hash table (для быстрого поиска по value). Все операции O(log n). Использование: лидерборды, очереди с приоритетом, временные окна (score = timestamp).
