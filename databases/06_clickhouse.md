# ClickHouse: Аналитическая СУБД

---

## Лекция

### 1. Что такое ClickHouse и когда он нужен

ClickHouse — колоночная СУБД для аналитических запросов (OLAP). Создан в Яндексе, открытый исходный код с 2016.

**OLTP vs OLAP:**
| | OLTP (MySQL, PostgreSQL) | OLAP (ClickHouse) |
|--|--------------------------|-------------------|
| Задача | Транзакции, CRUD | Аналитика, агрегации |
| Запросы | Короткие, точечные | Длинные, по всей таблице |
| Данные | Отдельные строки | Миллионы строк, несколько колонок |
| Хранение | Построчное | Поколонное |
| Обновления | Часто | Редко (append-mostly) |

**Когда ClickHouse:**
- Аналитика событий (клики, просмотры, логи)
- Метрики и мониторинг
- A/B тесты, воронки конверсий
- Отчёты с агрегацией по большим данным
- Хранилище данных (Data Warehouse)

---

### 2. Колоночное хранение — ключевое преимущество

**Построчное хранение (OLTP):**
```
Row 1: [id=1, name="Alice", age=30, email="alice@...", country="US", ...]
Row 2: [id=2, name="Bob",   age=25, email="bob@...",   country="UK", ...]
```

**Поколонное хранение (OLAP):**
```
Column id:      [1, 2, 3, 4, 5, ...]
Column name:    ["Alice", "Bob", "Charlie", ...]
Column age:     [30, 25, 28, 35, ...]
Column country: ["US", "UK", "US", "DE", ...]
```

**Запрос аналитики:**
```sql
SELECT country, AVG(age) FROM users GROUP BY country;
```
- PostgreSQL читает ВСЕ колонки каждой строки, но нужны только `country` и `age`
- ClickHouse читает только два столбца — колоссальная экономия I/O

**Сжатие:** одна колонка = один тип данных → очень высокая степень сжатия (LZ4, ZSTD). Числа одного масштаба — дельта-кодирование. В среднем 10-20x сжатие.

---

### 3. MergeTree — основной движок

```sql
CREATE TABLE events (
    event_date  Date,
    user_id     UInt64,
    event_type  String,
    page_url    String,
    duration_ms UInt32,
    properties  String  -- JSON
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)   -- партиция по месяцу
ORDER BY (user_id, event_date)       -- первичный ключ для сортировки
SETTINGS index_granularity = 8192;  -- строк в одной гранулярности
```

**Как работает MergeTree:**
- Данные вставляются частями (parts)
- В фоне ClickHouse сливает (merges) мелкие части в большие — отсюда "MergeTree"
- Внутри каждой части данные отсортированы по `ORDER BY`
- `PARTITION BY` — физическое разделение по партициям

**Sparse Index (разреженный индекс):**
- Не хранит запись для каждой строки
- Хранит одну запись на `index_granularity = 8192` строк (одна "гранула")
- При запросе находит нужные гранулы → читает только их
- Очень компактный индекс, помещается в RAM

---

### 4. Семейство движков MergeTree

**ReplacingMergeTree** — дедупликация по PK при merge:
```sql
ENGINE = ReplacingMergeTree(version)
-- Оставляет строку с наибольшим version при merge
-- Используется для обновляемых данных (eventual consistency)
```

**SummingMergeTree** — суммирование числовых колонок при merge:
```sql
ENGINE = SummingMergeTree([columns_to_sum])
-- Автоматически суммирует числа при слиянии партиций
-- Используется для предагрегированных метрик
```

**AggregatingMergeTree** — продвинутая агрегация:
```sql
ENGINE = AggregatingMergeTree()
-- Хранит промежуточные состояния агрегатных функций
-- Используется с Materialized Views
```

**CollapsingMergeTree / VersionedCollapsingMergeTree** — для "удалений" через знаковые строки.

**ReplicatedMergeTree** — любой из вышеперечисленных с репликацией:
```sql
ENGINE = ReplicatedMergeTree('/clickhouse/tables/events', '{replica}')
-- Использует ZooKeeper/ClickHouse Keeper для координации
```

---

### 5. Materialized Views

```sql
-- Исходная таблица с сырыми данными
CREATE TABLE page_views (
    timestamp DateTime,
    user_id   UInt64,
    page      String,
    duration  UInt32
) ENGINE = MergeTree() ORDER BY timestamp;

-- Материализованное представление: предагрегируем по часам
CREATE MATERIALIZED VIEW page_views_hourly
ENGINE = SummingMergeTree()
ORDER BY (hour, page)
AS SELECT
    toStartOfHour(timestamp) AS hour,
    page,
    count() AS views,
    sum(duration) AS total_duration,
    uniq(user_id) AS unique_users
FROM page_views
GROUP BY hour, page;

-- Данные вставляются в page_views → автоматически агрегируются в page_views_hourly
-- Быстрые запросы по периодам:
SELECT page, sum(views), sum(unique_users)
FROM page_views_hourly
WHERE hour >= '2024-01-01' AND hour < '2024-02-01'
GROUP BY page
ORDER BY sum(views) DESC;
```

---

### 6. Дистрибутивные запросы и Distributed движок

```sql
-- Distributed таблица — прокси над шардами
CREATE TABLE events_distributed AS events
ENGINE = Distributed(
    'cluster_name',  -- имя кластера из конфига
    'database',      -- база данных на шардах
    'events',        -- локальная таблица на каждом шарде
    user_id          -- ключ шардирования
);

-- Запросы к events_distributed автоматически распределяются по шардам
-- и результаты объединяются
```

---

### 7. Типичные паттерны использования

**Событийная аналитика:**
```sql
-- Воронка конверсий (Funnel Analysis)
SELECT
    countIf(has(groupArray(event_type), 'page_view')) AS step1,
    countIf(has(groupArray(event_type), 'add_to_cart')) AS step2,
    countIf(has(groupArray(event_type), 'purchase')) AS step3
FROM events
WHERE event_date >= today() - 30
GROUP BY user_id;

-- Retention: пользователи, вернувшиеся через N дней
SELECT
    start_date,
    countIf(datediff('day', start_date, return_date) = 1) AS day1_retention
FROM (
    SELECT user_id, min(event_date) AS start_date FROM events GROUP BY user_id
) users
JOIN events ON users.user_id = events.user_id
GROUP BY start_date;
```

---

## Практические примеры

### Пример 1: Вставка данных из Go

```go
import "github.com/ClickHouse/clickhouse-go/v2"

type Event struct {
    EventDate time.Time
    UserID    uint64
    EventType string
    PageURL   string
    DurationMs uint32
}

func (r *EventRepository) BatchInsert(ctx context.Context, events []Event) error {
    batch, err := r.conn.PrepareBatch(ctx, "INSERT INTO events")
    if err != nil {
        return err
    }

    for _, e := range events {
        if err := batch.Append(
            e.EventDate,
            e.UserID,
            e.EventType,
            e.PageURL,
            e.DurationMs,
        ); err != nil {
            return err
        }
    }

    return batch.Send()
}
```

### Пример 2: Аналитика за период

```sql
-- Топ страниц за последние 30 дней с метриками
SELECT
    page_url,
    count() AS views,
    uniqExact(user_id) AS unique_users,
    avg(duration_ms) AS avg_duration_ms,
    quantile(0.95)(duration_ms) AS p95_duration_ms
FROM events
WHERE
    event_type = 'page_view'
    AND event_date >= today() - 30
GROUP BY page_url
ORDER BY views DESC
LIMIT 20;

-- Дневная активность с оконными функциями
SELECT
    event_date,
    uniq(user_id) AS dau,
    avg(uniq(user_id)) OVER (
        ORDER BY event_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS wau_rolling
FROM events
WHERE event_date >= today() - 90
GROUP BY event_date
ORDER BY event_date;
```

---

## Вопросы на собеседовании

### Q: Почему ClickHouse быстрый? В чём суть колоночного хранения?

**Ответ:** Колоночное хранение: данные одного столбца хранятся последовательно на диске. При аналитическом запросе (например, `SELECT country, SUM(amount) GROUP BY country`) нужно прочитать только 2 столбца из 50 — это 4% от всех данных против 100% при построчном хранении. Плюс однотипные данные в колонке отлично сжимаются (10-20x). ClickHouse также использует SIMD инструкции для векторизованной обработки данных и параллельное выполнение.

---

### Q: Что такое MergeTree? Как работают вставки?

**Ответ:** MergeTree — основной движок ClickHouse. При INSERT данные записываются как отдельная "часть" (part) — небольшой отсортированный кусок данных. Периодически в фоне мелкие части сливаются в большие (merge) — данные пересортируются, дедуплицируются (ReplacingMergeTree), агрегируются (SummingMergeTree). INSERT всегда быстрый (write-once), merge происходит асинхронно. Поэтому ClickHouse не подходит для частых UPDATE/DELETE.

---

### Q: Чем ClickHouse отличается от PostgreSQL? Когда что использовать?

**Ответ:** PostgreSQL — OLTP: строковое хранение, ACID, быстрые точечные операции, частые обновления. ClickHouse — OLAP: колоночное хранение, нет полноценных ACID транзакций, медленные UPDATE/DELETE, но агрегации по миллиардам строк — за секунды. Используй PostgreSQL для бизнес-данных (заказы, пользователи, транзакции). ClickHouse — для аналитики, логов, событий, где нужны быстрые агрегации по большим данным.

---

### Q: Что такое Materialized View в ClickHouse?

**Ответ:** Материализованное представление в ClickHouse — это триггер: при INSERT в исходную таблицу автоматически выполняется запрос и результат вставляется в целевую таблицу. Это позволяет поддерживать предагрегированные данные в реальном времени. Например: сырые события → автоматически суммируются по часам в отдельной таблице. Запросы к агрегированной таблице работают в тысячи раз быстрее, чем агрегация по сырым данным.
