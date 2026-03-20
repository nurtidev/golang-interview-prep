# PostgreSQL: Архитектура и продвинутые возможности

---

## Лекция

### 1. Архитектура PostgreSQL

PostgreSQL — объектно-реляционная СУБД с процессной моделью (не тредовой). Каждое соединение = отдельный процесс ОС.

```
Client App
    |
    | TCP/Unix Socket
    |
Postmaster (главный процесс)
    |— Backend Process (для каждого соединения)
    |— WAL Writer
    |— Background Writer
    |— Checkpointer
    |— Autovacuum Launcher
    |— Stats Collector
```

**Shared Buffer Cache** — общая память для кеша страниц данных (аналог page cache в ОС). По умолчанию 128MB, рекомендуется 25% RAM.

---

### 2. MVCC и WAL

**MVCC (Multi-Version Concurrency Control):**
- Каждая строка содержит `xmin` (транзакция, создавшая строку) и `xmax` (транзакция, удалившая строку)
- При UPDATE: старая строка помечается `xmax`, создаётся новая с новым `xmin`
- Транзакция видит только строки, где `xmin <= txid < xmax`
- Читатели не блокируют писателей — главное преимущество MVCC

**WAL (Write-Ahead Log):**
- Перед изменением данных изменение записывается в журнал (WAL)
- Если сервер упадёт — WAL воспроизводится для восстановления
- Основа для репликации: WAL передаётся на реплики
- `wal_level`: minimal → replica → logical (нужен для логической репликации)

**VACUUM:**
- MVCC накапливает "мёртвые" версии строк (старые xmax)
- VACUUM их убирает и возвращает место в страницы
- AUTOVACUUM — автоматический фоновый процесс
- `VACUUM FULL` — полная реорганизация таблицы с блокировкой (осторожно на prod)

---

### 3. Типы данных PostgreSQL

```sql
-- Специфичные для PostgreSQL типы
uuid            -- UUID v4: gen_random_uuid()
jsonb           -- JSONB (бинарный JSON, индексируемый)
json            -- JSON текст (не индексируется напрямую)
text[]          -- массив текстов
int4range       -- диапазон целых чисел [1, 10)
tstzrange       -- диапазон временных меток с часовым поясом
tsvector        -- для full-text search
point, polygon  -- геометрические типы
inet, cidr      -- IP-адреса
```

```sql
-- JSONB операторы
SELECT data->>'name' FROM users;              -- текстовое значение
SELECT data->'address'->>'city' FROM users;   -- вложенный путь
SELECT * FROM users WHERE data @> '{"active": true}'; -- содержит
SELECT * FROM users WHERE data ? 'email';     -- ключ существует

-- Индекс для JSONB
CREATE INDEX idx_users_data ON users USING gin(data);
CREATE INDEX idx_users_data_path ON users USING gin(data jsonb_path_ops);
```

---

### 4. Репликация

**Streaming Replication (физическая):**
- Primary передаёт WAL файлы на реплики в реальном времени
- Реплика — точная физическая копия (байт в байт)
- Реплики read-only
- Синхронная (`synchronous_commit = on`) — Primary ждёт подтверждения от реплики
- Асинхронная — может быть lag, но выше производительность

**Logical Replication:**
- Передаёт логические изменения (INSERT/UPDATE/DELETE на уровне строк)
- Можно реплицировать отдельные таблицы
- Реплика может иметь другую схему
- Используется для zero-downtime миграций, CDC (Change Data Capture)

```sql
-- Настройка логической репликации
-- На primary:
CREATE PUBLICATION my_pub FOR TABLE orders, users;

-- На replica:
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=primary dbname=mydb user=rep'
    PUBLICATION my_pub;
```

---

### 5. Партиционирование

Разделение большой таблицы на меньшие физические части (партиции). Запросы идут только в нужные партиции (partition pruning).

```sql
-- Партиционирование по диапазону (самое популярное — для временных рядов)
CREATE TABLE orders (
    id BIGINT,
    created_at TIMESTAMPTZ NOT NULL,
    user_id BIGINT,
    amount DECIMAL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Партиционирование по списку (например, по региону)
CREATE TABLE users PARTITION BY LIST (region);
CREATE TABLE users_eu PARTITION OF users FOR VALUES IN ('DE', 'FR', 'PL');
CREATE TABLE users_us PARTITION OF users FOR VALUES IN ('US', 'CA');

-- Партиционирование по hash (равномерное распределение)
CREATE TABLE events PARTITION BY HASH (user_id);
CREATE TABLE events_0 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE events_1 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 1);
```

---

### 6. CTE и рекурсивные запросы

```sql
-- Common Table Expression (WITH)
WITH monthly_stats AS (
    SELECT
        DATE_TRUNC('month', created_at) as month,
        SUM(amount) as total
    FROM orders
    GROUP BY 1
),
running_total AS (
    SELECT
        month,
        total,
        SUM(total) OVER (ORDER BY month) as cumulative
    FROM monthly_stats
)
SELECT * FROM running_total;

-- Рекурсивный CTE — для обхода деревьев и иерархий
WITH RECURSIVE category_tree AS (
    -- Базовый случай: корневые категории
    SELECT id, name, parent_id, 0 AS depth
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Рекурсивный случай: дочерние категории
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY depth, name;
```

---

### 7. Connection Pooling

Каждое соединение с PostgreSQL = процесс ОС. 1000 соединений = 1000 процессов — это дорого.

**PgBouncer** — легковесный пул соединений:

```
Приложения → PgBouncer → PostgreSQL
(1000 conn)    (пул)     (50-100 conn)
```

Режимы работы:
- **Session mode** — клиент держит соединение всё время сессии
- **Transaction mode** — соединение отдаётся обратно в пул после каждой транзакции (рекомендуется)
- **Statement mode** — после каждого запроса (несовместим с транзакциями)

---

### 8. Полнотекстовый поиск

```sql
-- Создание tsvector
SELECT to_tsvector('russian', 'Быстрая коричневая лиса');
-- 'бистр':1 'коричнев':2 'лис':3

-- Поиск
SELECT title
FROM articles
WHERE to_tsvector('russian', content) @@ to_tsquery('russian', 'PostgreSQL & индекс');

-- Индекс для full-text search
ALTER TABLE articles ADD COLUMN search_vector tsvector;
UPDATE articles SET search_vector = to_tsvector('russian', content);
CREATE INDEX idx_articles_fts ON articles USING gin(search_vector);

-- Автоматическое обновление через триггер
CREATE TRIGGER articles_search_update
BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION
    tsvector_update_trigger(search_vector, 'pg_catalog.russian', content, title);
```

---

## Практические примеры

### Пример 1: Upsert (INSERT ON CONFLICT)

```sql
-- Вставить или обновить при конфликте
INSERT INTO user_stats (user_id, login_count, last_login)
VALUES ($1, 1, NOW())
ON CONFLICT (user_id) DO UPDATE
SET
    login_count = user_stats.login_count + 1,
    last_login = EXCLUDED.last_login
WHERE user_stats.last_login < EXCLUDED.last_login;

-- EXCLUDED — это таблица со значениями, которые пытались вставить
```

### Пример 2: Advisory Locks (распределённые блокировки через PostgreSQL)

```go
// Получить эксклюзивный advisory lock (например, для cron job)
func acquireLock(ctx context.Context, db *sql.DB, lockID int64) (bool, error) {
    var acquired bool
    err := db.QueryRowContext(ctx,
        "SELECT pg_try_advisory_lock($1)", lockID,
    ).Scan(&acquired)
    return acquired, err
}

func releaseLock(ctx context.Context, db *sql.DB, lockID int64) error {
    _, err := db.ExecContext(ctx, "SELECT pg_advisory_unlock($1)", lockID)
    return err
}

// Использование: гарантирует что только один instance обрабатывает задачу
const CRON_JOB_LOCK = 12345
if acquired, _ := acquireLock(ctx, db, CRON_JOB_LOCK); acquired {
    defer releaseLock(ctx, db, CRON_JOB_LOCK)
    runCronJob()
}
```

### Пример 3: Оптимизация с covering index

```sql
-- Запрос: выбрать email и created_at для активных пользователей
SELECT email, created_at FROM users WHERE status = 'active';

-- Без индекса: Seq Scan
-- С обычным индексом на status: Index Scan + Heap fetch
-- С covering index: Index Only Scan (не трогает основную таблицу)
CREATE INDEX idx_users_status_covering ON users(status)
INCLUDE (email, created_at);
```

### Пример 4: Мониторинг медленных запросов

```sql
-- Включить логирование медленных запросов (postgresql.conf)
log_min_duration_statement = 1000  -- запросы дольше 1 секунды

-- pg_stat_statements — статистика по всем запросам
SELECT
    query,
    calls,
    total_exec_time / calls AS avg_ms,
    rows / calls AS avg_rows,
    shared_blks_hit / (shared_blks_hit + shared_blks_read + 1.0) AS cache_hit_ratio
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Найти неиспользуемые индексы
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## Вопросы на собеседовании

### Q: Что такое MVCC и как он реализован в PostgreSQL?

**Ответ:** MVCC — Multi-Version Concurrency Control. Каждая строка хранит `xmin` (id транзакции, создавшей) и `xmax` (id транзакции, удалившей). При UPDATE создаётся новая версия строки, старая помечается xmax. Транзакция видит только строки, созданные до её начала и не удалённые (или удалённые после). Это позволяет читателям не блокировать писателей. Минус: накапливаются мёртвые версии, их убирает VACUUM.

---

### Q: Зачем нужен VACUUM? Когда запускать VACUUM FULL?

**Ответ:** VACUUM убирает мёртвые версии строк (накопленные из-за MVCC), возвращает место в страницы и обновляет статистику. AUTOVACUUM делает это автоматически. VACUUM FULL — полная реорганизация таблицы, возвращает место ОС, но блокирует таблицу целиком. На продакшне `VACUUM FULL` нужно использовать осторожно, в окна обслуживания. Альтернатива — `pg_repack` (без полной блокировки).

---

### Q: Чем JSONB отличается от JSON в PostgreSQL?

**Ответ:** JSON хранится как текст (как есть), сохраняет порядок ключей и дублирующиеся ключи. JSONB хранится в бинарном формате, удаляет дублирующиеся ключи, не сохраняет порядок, но зато поддерживает индексы (GIN) и значительно быстрее при поиске. Для большинства случаев используй JSONB.

---

### Q: Что такое WAL и зачем он нужен?

**Ответ:** WAL (Write-Ahead Log) — журнал изменений. Перед записью в основные файлы данных изменение пишется в WAL. При сбое сервера WAL воспроизводится — данные не теряются (Durability из ACID). WAL также основа репликации: Primary передаёт WAL-сегменты на реплики. Параметр `wal_level` определяет уровень детализации: `replica` для стримминг-репликации, `logical` для логической.

---

### Q: В чём разница между физической и логической репликацией?

**Ответ:** Физическая (Streaming Replication) передаёт WAL-байты — реплика является точной физической копией. Реплики только для чтения. Логическая репликация передаёт логические изменения (INSERT/UPDATE/DELETE), можно реплицировать отдельные таблицы, реплика может иметь другую схему, другую СУБД. Логическая репликация используется для CDC (Change Data Capture), zero-downtime миграций между версиями PostgreSQL.

---

### Q: Как работает партиционирование? Когда стоит применять?

**Ответ:** Партиционирование делит таблицу на физические части (партиции) по определённому критерию. При запросе PostgreSQL делает partition pruning — обращается только к нужным партициям. Типы: RANGE (по дате — для временных рядов), LIST (по значению — по региону), HASH (равномерное распределение). Применять когда: таблица > 10-50GB, большинство запросов идут в "горячий" диапазон (последние N дней), нужно быстро удалять старые данные (DROP PARTITION вместо DELETE).

---

### Q: Что такое Connection Pooling? Зачем нужен PgBouncer?

**Ответ:** Каждое соединение с PostgreSQL создаёт процесс ОС. При 1000 соединениях — 1000 процессов, это огромные накладные расходы на память и переключение контекста. PgBouncer — пул соединений перед PostgreSQL: приложения подключаются к PgBouncer (может быть 1000 соединений), а PgBouncer держит небольшой пул реальных соединений к PostgreSQL (50-100). Рекомендуемый режим — transaction mode: соединение возвращается в пул после каждой транзакции.
