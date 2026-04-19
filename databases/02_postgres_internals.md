# PostgreSQL Internals

---

## Лекция

### 1. MVCC — Multi-Version Concurrency Control

Проблема: как дать двум транзакциям читать и писать одновременно без блокировок?

Postgres решает это через **MVCC**: при UPDATE старая версия строки не удаляется сразу — создаётся новая версия. Каждая транзакция видит свой "снимок" данных на момент старта.

```
Строка в orders:
  xmin=100  xmax=0    status='shipped'   ← актуальная версия
  
После UPDATE в транзакции 200:
  xmin=100  xmax=200  status='shipped'   ← старая (мертвая) версия
  xmin=200  xmax=0    status='delivered' ← новая версия
```

- `xmin` — ID транзакции, которая создала версию
- `xmax` — ID транзакции, которая удалила/обновила (0 = жива)
- Транзакция видит строку только если `xmin` завершилась до её старта и `xmax` ещё не завершилась

**Следствие MVCC:**
- Читатели никогда не блокируют писателей
- Писатели никогда не блокируют читателей
- Но мёртвые версии строк накапливаются → bloat

---

### 2. Table Bloat и VACUUM

После массовых UPDATE/DELETE таблица разрастается из-за мёртвых tuple. Это называется **table bloat**.

```sql
-- Посмотреть bloat таблицы
SELECT relname, n_dead_tup, n_live_tup,
       round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

**VACUUM** — фоновый процесс, который помечает мёртвые tuple как доступные для повторного использования (не возвращает место ОС).

**VACUUM FULL** — перестраивает таблицу полностью, возвращает место. Но блокирует таблицу на всё время работы — на проде опасно.

**AUTOVACUUM** — запускается автоматически при превышении порога мёртвых tuple. Настраивается через `autovacuum_vacuum_threshold` и `autovacuum_vacuum_scale_factor`.

**Когда bloat критичен:**
- Запросы замедляются — seq scan читает лишние страницы
- Индексы тоже разрастаются (index bloat)
- После массового UPDATE 10M строк — запускать `VACUUM ANALYZE` вручную

---

### 3. WAL — Write-Ahead Log

**Проблема:** если Postgres упадёт в момент записи данных на диск — данные повредятся.

**Решение WAL:** перед изменением любых данных Postgres сначала записывает это изменение в WAL (журнал). При восстановлении — воспроизводит WAL с последнего checkpoint.

```
Транзакция → запись в WAL → COMMIT → применение на страницы данных
                                         ↑
                                   (может быть отложено)
```

**WAL и репликация:**
Реплики применяют WAL стрима от мастера. При массовом UPDATE 10M строк:
- Генерируется огромный WAL
- Реплики не успевают применять → **replication lag**
- Читатели на репликах видят устаревшие данные

```sql
-- Проверить replication lag
SELECT client_addr,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

---

### 4. Query Planner — как Postgres выбирает план выполнения

Postgres строит план запроса на основе **статистики** (pg_statistics). Основные операции:

| Операция | Когда используется |
|----------|-------------------|
| **Seq Scan** | Нет подходящего индекса или selectivity низкая (>10-20% таблицы) |
| **Index Scan** | Высокая selectivity, случайный доступ к heap |
| **Index Only Scan** | Все нужные колонки есть в индексе (covering index) |
| **Bitmap Heap Scan** | Средняя selectivity — сначала строит bitmap по индексу, потом читает heap блоками |

```sql
-- Смотреть план запроса
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE status = 'shipped';

-- Seq Scan → плохо для большой таблицы
-- Bitmap Heap Scan → ok для средней выборки
-- Index Only Scan → идеально
```

**Почему планировщик игнорирует индекс?**
- Статистика устарела → `ANALYZE` таблицы
- Selectivity слишком низкая (статус 'shipped' у 70% строк → seq scan выгоднее)
- `random_page_cost` слишком высокий → планировщик думает что диск медленный

---

### 5. Транзакции и уровни изоляции

| Уровень | Грязное чтение | Неповторяемое чтение | Фантомное чтение |
|---------|---------------|---------------------|-----------------|
| **Read Uncommitted** | да | да | да |
| **Read Committed** | нет | да | да |
| **Repeatable Read** | нет | нет | нет* |
| **Serializable** | нет | нет | нет |

*В Postgres Repeatable Read устраняет и фантомы — благодаря MVCC snapshot.

**Аномалии:**

```sql
-- Грязное чтение: видишь незакоммиченные данные другой транзакции
-- (в Postgres невозможно даже на Read Uncommitted)

-- Неповторяемое чтение: один и тот же SELECT возвращает разные данные
BEGIN; -- TX1
SELECT balance FROM accounts WHERE id = 1; -- 1000
-- TX2 делает UPDATE balance = 500, COMMIT
SELECT balance FROM accounts WHERE id = 1; -- 500 (другой результат!)
COMMIT;

-- Фантомное чтение: появляются новые строки между двумя SELECT
BEGIN; -- TX1
SELECT COUNT(*) FROM orders WHERE status = 'new'; -- 10
-- TX2 вставляет новый order, COMMIT
SELECT COUNT(*) FROM orders WHERE status = 'new'; -- 11 (фантом!)
COMMIT;
```

**В продакшене чаще всего используют:**
- `Read Committed` — дефолт в Postgres, подходит для большинства задач
- `Repeatable Read` — финансовые операции, отчёты
- `Serializable` — критичные транзакции (перевод денег)

---

### 6. Шардирование и партиционирование

**Партиционирование** — разбивка одной таблицы на части внутри одной БД.

```sql
-- Партиционирование по дате
CREATE TABLE orders (
    id SERIAL,
    created_at TIMESTAMP,
    status VARCHAR(20)
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

**Плюсы партиционирования:**
- Запросы с фильтром по дате читают только нужную партицию (partition pruning)
- Старые данные легко архивировать/удалять (DROP PARTITION — мгновенно)
- VACUUM работает на меньших таблицах

**Шардирование** — разбивка данных по разным физическим серверам. Делается на уровне приложения или через Citus/pg_partman.

---

### 7. Практика: массовый UPDATE без боли

**Плохо (убивает прод):**
```sql
UPDATE orders SET status = 'delivered'
WHERE status = 'shipped' AND created_at < NOW() - INTERVAL '14 days';
-- Блокирует 10M строк, огромный WAL, bloat, replication lag
```

**Правильно — батчами:**
```sql
DO $$
DECLARE
    batch_size INT := 10000;
    updated    INT;
BEGIN
    LOOP
        UPDATE orders SET status = 'delivered'
        WHERE id IN (
            SELECT id FROM orders
            WHERE status = 'shipped'
              AND created_at < NOW() - INTERVAL '14 days'
            LIMIT batch_size
            FOR UPDATE SKIP LOCKED  -- не ждём залоченные строки
        );
        GET DIAGNOSTICS updated = ROW_COUNT;
        EXIT WHEN updated = 0;
        PERFORM pg_sleep(0.05);  -- пауза снижает нагрузку на реплики
    END LOOP;
END $$;
```

**Нужный индекс:**
```sql
CREATE INDEX CONCURRENTLY idx_orders_status_created
ON orders(status, created_at)
WHERE status = 'shipped';  -- partial index — меньше размер
```

---

## Вопросы с интервью

**Q: Что такое MVCC и почему в Postgres нет грязного чтения?**
A: MVCC хранит несколько версий строки. Каждая транзакция работает со снимком на момент старта и никогда не видит незакоммиченные данные других транзакций. Грязное чтение невозможно даже на уровне Read Uncommitted — Postgres просто игнорирует этот уровень и использует Read Committed.

**Q: Почему VACUUM не возвращает место операционной системе?**
A: Обычный VACUUM только помечает мёртвые tuple как доступные для повторного использования внутри тех же страниц. VACUUM FULL перестраивает таблицу и возвращает место — но блокирует таблицу целиком. Для проде используют `pg_repack` как альтернативу без блокировки.

**Q: Чем Index Only Scan отличается от Index Scan?**
A: Index Scan находит TID (tuple id) в индексе, потом идёт в heap за строкой — два чтения. Index Only Scan берёт все нужные данные прямо из индекса, heap не читает. Работает только если все SELECT-колонки входят в индекс (covering index).

**Q: Когда планировщик выбирает Seq Scan вместо индекса?**
A: Когда selectivity низкая — например, статус 'active' у 80% строк. Случайный доступ по индексу к 80% строк дороже, чем последовательное чтение всей таблицы. Также если статистика устарела — помогает `ANALYZE`.

**Q: Как массовый UPDATE влияет на реплики?**
A: Генерирует большой WAL, реплики не успевают применять → replication lag растёт. Читатели на репликах видят устаревшие данные. Решение — батчи с паузами между ними.

**Q: В чём разница между партиционированием и шардированием?**
A: Партиционирование — внутри одного сервера Postgres, прозрачно для приложения, планировщик делает partition pruning. Шардирование — данные на разных серверах, приложение само маршрутизирует запросы, нет join между шардами.
