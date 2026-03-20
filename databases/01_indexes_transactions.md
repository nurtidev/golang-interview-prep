# Базы данных: Индексы и Транзакции

---

## Лекция

### 1. Что такое индекс и зачем он нужен

Представь, что у тебя книга на 1000 страниц без оглавления. Чтобы найти слово "транзакция", придётся читать каждую страницу — это **sequential scan**, O(n).

Индекс — это оглавление к таблице. Отдельная структура данных, которая хранит отсортированные значения колонки + указатель на строку в таблице. Поиск становится O(log n).

**Цена индекса:**
- Дополнительное место на диске (каждый индекс — отдельная структура)
- Замедление записи — при INSERT/UPDATE/DELETE все индексы обновляются
- Планировщик может выбрать неоптимальный индекс, если их слишком много

---

### 2. B-Tree индекс — основной тип

B-Tree (Balanced Tree) — сбалансированное дерево поиска. В PostgreSQL используется по умолчанию.

```
Корень:         [20 | 40]
               /    |    \
Листья:   [10|15]  [25|30]  [45|50]
```

**Свойства:**
- Высота дерева: O(log n), для таблицы в 1 млн строк — ~20 уровней
- Каждый узел = одна страница диска (8KB в PostgreSQL)
- Листовые узлы связаны в двусвязный список → эффективный range scan
- Поддерживает: `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `LIKE 'prefix%'`, `ORDER BY`
- НЕ работает с: `LIKE '%suffix'`, функциями над колонкой (`WHERE LOWER(email) = ...`)

---

### 3. Другие типы индексов в PostgreSQL

| Тип | Когда использовать |
|-----|-------------------|
| **B-Tree** | Универсальный, для большинства случаев |
| **Hash** | Только `=` сравнения, чуть быстрее B-Tree для точного поиска |
| **GIN** | Массивы, JSONB, full-text search (когда одна строка имеет много значений) |
| **GiST** | Геоданные, геометрия, full-text |
| **BRIN** | Огромные таблицы с физически упорядоченными данными (логи, временные ряды) |

---

### 4. Составной индекс и правило левого префикса

Индекс `(a, b, c)` — это как справочник, отсортированный сначала по `a`, внутри по `b`, внутри по `c`.

**Правило левого префикса:** индекс работает только если запрос начинает с самых левых колонок.

```sql
CREATE INDEX idx_orders ON orders(user_id, status, created_at);

-- Использует индекс:
WHERE user_id = 1
WHERE user_id = 1 AND status = 'active'
WHERE user_id = 1 AND status = 'active' AND created_at > '2024-01-01'

-- НЕ использует индекс (пропущен user_id):
WHERE status = 'active'
WHERE created_at > '2024-01-01'
```

---

### 5. Когда индекс не работает (антипаттерны)

```sql
-- Функция над колонкой — индекс обходится
WHERE LOWER(email) = 'user@example.com'        -- плохо
-- Решение: функциональный индекс
CREATE INDEX ON users(LOWER(email));

-- Неявное приведение типов
WHERE user_id = '123'  -- user_id bigint, '123' — text → cast → нет индекса

-- LIKE с левым wildcard
WHERE name LIKE '%john%'  -- не использует B-Tree

-- OR между разными колонками
WHERE a = 1 OR b = 2      -- могут использоваться два отдельных индекса через Bitmap Scan
```

---

### 6. EXPLAIN ANALYZE — как читать план запроса

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';
```

| Тип сканирования | Что означает |
|-----------------|--------------|
| **Seq Scan** | Полный перебор таблицы — плохо для больших таблиц |
| **Index Scan** | Использует индекс, читает строки по одной |
| **Index Only Scan** | Все данные берутся из индекса, таблица не трогается — лучший вариант |
| **Bitmap Heap Scan** | Сначала собирает битмап страниц через индекс, потом читает страницы — для диапазонов |

---

### 7. Что такое транзакция

Транзакция — группа операций, которые выполняются как единое целое. Либо все успешно, либо ни одна.

**Пример:** перевод денег между счетами. Нельзя списать с одного счёта и не зачислить на другой.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- или ROLLBACK в случае ошибки
```

---

### 8. ACID

| Буква | Свойство | Объяснение простыми словами |
|-------|----------|-----------------------------|
| **A** | Atomicity (Атомарность) | Всё или ничего. Нет частичного выполнения. |
| **C** | Consistency (Согласованность) | БД переходит из одного валидного состояния в другое. Ограничения (FK, CHECK) не нарушаются. |
| **I** | Isolation (Изоляция) | Транзакции не видят незафиксированные изменения друг друга. |
| **D** | Durability (Долговечность) | После COMMIT данные не потеряются даже при сбое питания. Обеспечивается WAL (журнал). |

---

### 9. Уровни изоляции и аномалии

**Аномалии** — нежелательные эффекты при параллельном выполнении транзакций:

- **Dirty Read** — Транзакция А читает незафиксированные изменения транзакции Б. Если Б откатится — А работала с "грязными" данными.
- **Non-repeatable Read** — Транзакция А дважды читает одну строку и получает разные значения (Б успела изменить и зафиксировать между чтениями).
- **Phantom Read** — Транзакция А дважды выполняет одинаковый SELECT и получает разные наборы строк (Б вставила новые строки).

| Уровень изоляции | Dirty Read | Non-repeatable Read | Phantom Read |
|-----------------|-----------|---------------------|--------------|
| READ UNCOMMITTED | возможен | возможен | возможен |
| READ COMMITTED | защита | возможен | возможен |
| REPEATABLE READ | защита | защита | возможен (в большинстве БД) |
| SERIALIZABLE | защита | защита | защита |

**Важно про PostgreSQL:**
- `READ UNCOMMITTED` ведёт себя как `READ COMMITTED` — грязного чтения нет в принципе
- `REPEATABLE READ` в PostgreSQL защищает и от Phantom Read (MVCC)
- По умолчанию используется `READ COMMITTED`

---

### 10. Блокировки и Deadlock

PostgreSQL использует **MVCC (Multi-Version Concurrency Control)** — читатели не блокируют писателей, писатели не блокируют читателей. Каждая транзакция видит снапшот данных на момент начала.

**Deadlock** — ситуация, когда две транзакции ждут друг друга:

```
Транзакция 1: заблокировала строку 1, ждёт строку 2
Транзакция 2: заблокировала строку 2, ждёт строку 1
→ Круговое ожидание = Deadlock
```

PostgreSQL автоматически детектирует дедлок и откатывает одну из транзакций (ту, что дешевле откатить).

**Как избежать:** всегда обновлять строки в одном и том же порядке (например, по возрастанию id).

---

### 11. Оконные функции

Оконные функции выполняются над набором строк, связанных с текущей строкой, без группировки (строки не схлопываются).

```sql
функция() OVER (
    PARTITION BY колонка  -- разделить на группы (как GROUP BY, но без схлопывания)
    ORDER BY колонка       -- порядок внутри окна
    ROWS/RANGE BETWEEN ... -- размер окна
)
```

Основные функции: `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `LAG()`, `LEAD()`, `SUM()`, `AVG()`, `FIRST_VALUE()`, `LAST_VALUE()`

---

## Практические примеры

### Пример 1: Поиск медленных запросов и добавление индекса

```sql
-- Проверяем план без индекса
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 42 AND status = 'pending';
-- Seq Scan on orders (cost=0.00..45000.00 rows=3 width=100) actual time=150ms

-- Добавляем составной индекс
CREATE INDEX CONCURRENTLY idx_orders_user_status
ON orders(user_id, status);
-- CONCURRENTLY — создание без блокировки таблицы (важно для prod)

-- Проверяем снова
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 42 AND status = 'pending';
-- Index Scan using idx_orders_user_status (cost=0.43..8.45 rows=3 width=100) actual time=0.3ms
```

### Пример 2: Функциональный индекс для case-insensitive поиска

```sql
-- Без индекса — Seq Scan
WHERE LOWER(email) = LOWER('User@Example.COM')

-- Создаём функциональный индекс
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Теперь работает с индексом
WHERE LOWER(email) = 'user@example.com'
```

### Пример 3: Транзакция в Go

```go
func transfer(ctx context.Context, db *sql.DB, fromID, toID int64, amount float64) error {
    tx, err := db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelReadCommitted,
    })
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer tx.Rollback() // безопасно: если уже Commit — ничего не делает

    _, err = tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
        amount, fromID,
    )
    if err != nil {
        return fmt.Errorf("debit: %w", err)
    }

    _, err = tx.ExecContext(ctx,
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount, toID,
    )
    if err != nil {
        return fmt.Errorf("credit: %w", err)
    }

    return tx.Commit()
}
```

### Пример 4: Оконные функции

```sql
-- Ранжирование продаж по отделам
SELECT
    department,
    employee,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS overall_rank
FROM employees;

-- Скользящее среднее за 7 дней
SELECT
    date,
    revenue,
    AVG(revenue) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d
FROM daily_revenue;

-- Изменение относительно предыдущего периода
SELECT
    date,
    revenue,
    LAG(revenue) OVER (ORDER BY date) AS prev_revenue,
    ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY date))
          / LAG(revenue) OVER (ORDER BY date), 2) AS pct_change
FROM daily_revenue;
```

### Пример 5: Избежание Deadlock

```go
// ПЛОХО: разный порядок обновлений → возможен deadlock
func badTransfer(tx *sql.Tx, from, to int64, amount float64) error {
    tx.Exec("UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, from)
    tx.Exec("UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, to)
    return nil
}

// ХОРОШО: всегда обновляем в порядке возрастания id
func goodTransfer(tx *sql.Tx, from, to int64, amount float64) error {
    first, second := from, to
    firstAmount, secondAmount := -amount, amount
    if from > to {
        first, second = to, from
        firstAmount, secondAmount = amount, -amount
    }
    tx.Exec("UPDATE accounts SET balance = balance + $1 WHERE id = $2", firstAmount, first)
    tx.Exec("UPDATE accounts SET balance = balance + $1 WHERE id = $2", secondAmount, second)
    return nil
}
```

---

## Вопросы на собеседовании

### Q: Что такое индекс и как он устроен внутри?

**Ответ:** Индекс — отдельная структура данных (обычно B-Tree), которая хранит отсортированные значения колонки и указатели на строки в таблице. B-Tree — сбалансированное дерево высотой O(log n). Каждый узел — страница диска (8KB). Листовые узлы связаны в список для эффективного range scan. Ускоряет SELECT, замедляет INSERT/UPDATE/DELETE (нужно обновить все индексы).

---

### Q: Как работает составной индекс? Что такое правило левого префикса?

**Ответ:** Составной индекс `(a, b, c)` сортирует сначала по `a`, затем по `b`, затем по `c`. Работает только если запрос использует колонки слева направо без пропусков. `WHERE a = 1` — работает, `WHERE a = 1 AND b = 2` — работает, `WHERE b = 2` — не работает (пропущен `a`). Порядок колонок важен: высококардинальные колонки (email, id) ставить первыми, колонки в WHERE условиях `=` — перед колонками диапазонов (`>`, `<`, `BETWEEN`).

---

### Q: Объясни ACID. Зачем нужна каждая буква?

**Ответ:**
- **A (Atomicity)** — транзакция неделима: либо все операции выполнились, либо ни одна. Реализуется через WAL (Write-Ahead Log) и возможность rollback.
- **C (Consistency)** — транзакция переводит БД из одного консистентного состояния в другое. Ограничения (FK, CHECK, NOT NULL) не нарушаются.
- **I (Isolation)** — транзакции изолированы друг от друга. В PostgreSQL реализуется через MVCC — каждая транзакция видит снапшот данных.
- **D (Durability)** — после COMMIT данные сохранены на диске. Реализуется через WAL: сначала запись в журнал, потом в основные файлы.

---

### Q: Какие уровни изоляции есть? Какой по умолчанию в PostgreSQL?

**Ответ:** Четыре уровня: READ UNCOMMITTED, READ COMMITTED (по умолчанию в PostgreSQL), REPEATABLE READ, SERIALIZABLE. В PostgreSQL READ UNCOMMITTED ведёт себя как READ COMMITTED (грязного чтения нет). REPEATABLE READ в PostgreSQL через MVCC защищает и от Phantom Read. Чем выше уровень изоляции — тем меньше аномалий, но больше конфликтов и overhead.

---

### Q: Что такое Deadlock и как его избежать?

**Ответ:** Deadlock — ситуация, когда транзакция A ждёт ресурс, заблокированный транзакцией B, а B ждёт ресурс, заблокированный A. Круговое ожидание. PostgreSQL автоматически детектирует и откатывает одну транзакцию. Как избежать: всегда захватывать блокировки в одном и том же порядке (например, по возрастанию id), держать транзакции короткими, не делать внешних вызовов внутри транзакции.

---

### Q: Чем WHERE отличается от HAVING?

**Ответ:** `WHERE` фильтрует строки **до** группировки — работает с отдельными строками, не может использовать агрегатные функции. `HAVING` фильтрует **после** группировки — работает с результатами GROUP BY, может использовать `COUNT()`, `SUM()` и т.д.

```sql
SELECT user_id, COUNT(*) as orders
FROM orders
WHERE status = 'completed'   -- фильтр строк до группировки
GROUP BY user_id
HAVING COUNT(*) > 5;          -- фильтр групп после группировки
```

---

### Q: Что такое Index Only Scan? Когда он возникает?

**Ответ:** Index Only Scan — PostgreSQL берёт все нужные данные прямо из индекса, не обращаясь к основной таблице. Возникает когда все запрашиваемые колонки есть в индексе (covering index). Это самый быстрый вид сканирования. Пример: `SELECT email FROM users WHERE email LIKE 'a%'` при наличии индекса на `email` — данные уже в индексе.

---

### Q: Что такое MVCC?

**Ответ:** Multi-Version Concurrency Control — механизм изоляции, при котором каждая транзакция видит снапшот данных на момент начала. При изменении строки PostgreSQL не перезаписывает её, а создаёт новую версию (с transaction id). Старые версии видны транзакциям, которые начались раньше. Это позволяет читателям не блокировать писателей и наоборот. Старые версии очищает процесс VACUUM.

---

### Q: Что делает EXPLAIN ANALYZE? Чем отличается от EXPLAIN?

**Ответ:** `EXPLAIN` показывает **предполагаемый** план выполнения с оценочными стоимостями (cost). `EXPLAIN ANALYZE` **реально выполняет** запрос и показывает фактическое время и количество строк. Используй EXPLAIN ANALYZE для диагностики медленных запросов. Смотри на расхождение между `rows=` (оценка) и `actual rows=` (факт) — большое расхождение говорит об устаревшей статистике, нужен `ANALYZE`.
