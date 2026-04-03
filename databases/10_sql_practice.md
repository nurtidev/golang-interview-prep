# SQL: Практика для интервью

## 1. Проектирование схемы (DDL)

### Домен: сервис поездок (inDrive-style)

```sql
-- Пользователи (и водители, и пассажиры)
CREATE TABLE users (
    id         BIGSERIAL    PRIMARY KEY,
    phone      VARCHAR(20)  NOT NULL UNIQUE,
    name       VARCHAR(100) NOT NULL,
    role       VARCHAR(10)  NOT NULL CHECK (role IN ('driver', 'passenger', 'both')),
    created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- Профиль водителя (1:1 с users)
CREATE TABLE driver_profiles (
    user_id     BIGINT      PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    car_model   VARCHAR(100) NOT NULL,
    car_plate   VARCHAR(20)  NOT NULL UNIQUE,
    rating      NUMERIC(3,2) NOT NULL DEFAULT 5.00,
    is_active   BOOLEAN      NOT NULL DEFAULT TRUE
);

-- Поездки
CREATE TABLE rides (
    id           BIGSERIAL    PRIMARY KEY,
    passenger_id BIGINT       NOT NULL REFERENCES users(id),
    driver_id    BIGINT       REFERENCES users(id),  -- NULL пока не принята
    status       VARCHAR(20)  NOT NULL DEFAULT 'searching'
                              CHECK (status IN ('searching', 'accepted', 'in_progress', 'completed', 'cancelled')),
    price        NUMERIC(10,2),
    distance_km  NUMERIC(6,2),
    origin       TEXT         NOT NULL,
    destination  TEXT         NOT NULL,
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);

-- Платежи
CREATE TABLE payments (
    id         BIGSERIAL    PRIMARY KEY,
    ride_id    BIGINT       NOT NULL REFERENCES rides(id),
    user_id    BIGINT       NOT NULL REFERENCES users(id),
    amount     NUMERIC(10,2) NOT NULL,
    method     VARCHAR(20)  NOT NULL CHECK (method IN ('card', 'cash', 'wallet')),
    status     VARCHAR(20)  NOT NULL DEFAULT 'pending'
                            CHECK (status IN ('pending', 'completed', 'failed', 'refunded')),
    created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- Рейтинги поездок
CREATE TABLE ride_reviews (
    id         BIGSERIAL    PRIMARY KEY,
    ride_id    BIGINT       NOT NULL REFERENCES rides(id),
    author_id  BIGINT       NOT NULL REFERENCES users(id),
    target_id  BIGINT       NOT NULL REFERENCES users(id),
    score      SMALLINT     NOT NULL CHECK (score BETWEEN 1 AND 5),
    comment    TEXT,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (ride_id, author_id)  -- один отзыв на поездку от одного автора
);
```

### Индексы к схеме

```sql
-- Поиск поездок пассажира / водителя — самые частые запросы
CREATE INDEX idx_rides_passenger_id ON rides(passenger_id);
CREATE INDEX idx_rides_driver_id    ON rides(driver_id);

-- Фильтрация по статусу + сортировка по дате
CREATE INDEX idx_rides_status_created ON rides(status, created_at DESC);

-- Платежи по поездке
CREATE INDEX idx_payments_ride_id ON payments(ride_id);

-- Поиск активных водителей
CREATE INDEX idx_driver_profiles_active ON driver_profiles(is_active) WHERE is_active = TRUE;
```

---

## 2. Типы JOIN — шпаргалка

```
INNER JOIN  — только строки, которые совпали в обеих таблицах
LEFT JOIN   — все из левой + совпавшие из правой (NULL если нет пары)
RIGHT JOIN  — все из правой + совпавшие из левой (редко используют, лучше LEFT)
FULL JOIN   — все из обеих таблиц, NULL где нет пары
CROSS JOIN  — декартово произведение (каждая с каждой)
```

### INNER JOIN — поездки с водителями

```sql
SELECT
    r.id          AS ride_id,
    p.name        AS passenger,
    d.name        AS driver,
    r.price,
    r.status
FROM rides r
INNER JOIN users p ON p.id = r.passenger_id
INNER JOIN users d ON d.id = r.driver_id  -- только поездки где driver_id NOT NULL
WHERE r.status = 'completed';
```

### LEFT JOIN — пассажиры без завершённых поездок

```sql
SELECT
    u.id,
    u.name,
    COUNT(r.id) AS completed_rides
FROM users u
LEFT JOIN rides r ON r.passenger_id = u.id AND r.status = 'completed'
WHERE u.role IN ('passenger', 'both')
GROUP BY u.id, u.name
HAVING COUNT(r.id) = 0;  -- только те у кого 0 завершённых поездок
```

### Самосоединение (self-join) — найти пользователей с тем же телефонным кодом

```sql
SELECT DISTINCT a.name, b.name, LEFT(a.phone, 4) AS area_code
FROM users a
JOIN users b ON LEFT(a.phone, 4) = LEFT(b.phone, 4) AND a.id < b.id;
```

---

## 3. Агрегация и GROUP BY

### Базовые агрегатные функции

```sql
-- Статистика по водителям за последние 30 дней
SELECT
    d.name                        AS driver,
    COUNT(r.id)                   AS total_rides,
    COUNT(r.id) FILTER (WHERE r.status = 'completed')  AS completed,
    COUNT(r.id) FILTER (WHERE r.status = 'cancelled')  AS cancelled,
    ROUND(AVG(r.price), 2)        AS avg_price,
    SUM(r.price)                  AS total_earnings,
    ROUND(AVG(rv.score), 2)       AS avg_rating
FROM users d
JOIN rides r       ON r.driver_id = d.id
LEFT JOIN ride_reviews rv ON rv.ride_id = r.id AND rv.target_id = d.id
WHERE d.role IN ('driver', 'both')
  AND r.created_at >= NOW() - INTERVAL '30 days'
GROUP BY d.id, d.name
HAVING COUNT(r.id) >= 5  -- только активных водителей
ORDER BY total_earnings DESC;
```

### WHERE vs HAVING

```sql
-- WHERE — фильтр ДО группировки (по строкам)
-- HAVING — фильтр ПОСЛЕ группировки (по группам/агрегатам)

-- ❌ Нельзя: COUNT() в WHERE
-- SELECT user_id FROM rides WHERE COUNT(*) > 10 GROUP BY user_id;

-- ✅ Правильно
SELECT passenger_id, COUNT(*) AS cnt
FROM rides
WHERE created_at >= '2024-01-01'   -- фильтр строк
GROUP BY passenger_id
HAVING COUNT(*) > 10;              -- фильтр групп
```

---

## 4. Оконные функции

```sql
-- ROW_NUMBER, RANK, DENSE_RANK
SELECT
    d.name,
    SUM(r.price)                                        AS earnings,
    RANK()       OVER (ORDER BY SUM(r.price) DESC)     AS rank,
    DENSE_RANK() OVER (ORDER BY SUM(r.price) DESC)     AS dense_rank,
    ROW_NUMBER() OVER (ORDER BY SUM(r.price) DESC)     AS row_num
FROM rides r
JOIN users d ON d.id = r.driver_id
WHERE r.status = 'completed'
GROUP BY d.id, d.name;

-- RANK vs DENSE_RANK:
-- RANK:       1, 2, 2, 4  (пропускает 3)
-- DENSE_RANK: 1, 2, 2, 3  (не пропускает)
```

```sql
-- LAG / LEAD — сравнение с предыдущей строкой
SELECT
    passenger_id,
    created_at,
    price,
    LAG(price) OVER (PARTITION BY passenger_id ORDER BY created_at) AS prev_price,
    price - LAG(price) OVER (PARTITION BY passenger_id ORDER BY created_at) AS price_diff
FROM rides
WHERE status = 'completed';
```

```sql
-- Накопительная сумма по дням
SELECT
    DATE(created_at)                          AS day,
    SUM(price)                                AS daily_revenue,
    SUM(SUM(price)) OVER (ORDER BY DATE(created_at)) AS cumulative_revenue
FROM rides
WHERE status = 'completed'
GROUP BY DATE(created_at)
ORDER BY day;
```

---

## 5. Типовые задачи

### Топ-N: топ 5 водителей по выручке

```sql
SELECT driver_id, SUM(price) AS total
FROM rides
WHERE status = 'completed'
GROUP BY driver_id
ORDER BY total DESC
LIMIT 5;
```

### Топ-N в каждой группе (топ 3 водителя в каждом городе)

```sql
WITH ranked AS (
    SELECT
        driver_id,
        origin,         -- допустим origin = город
        SUM(price)      AS total,
        ROW_NUMBER() OVER (PARTITION BY origin ORDER BY SUM(price) DESC) AS rn
    FROM rides
    WHERE status = 'completed'
    GROUP BY driver_id, origin
)
SELECT * FROM ranked WHERE rn <= 3;
```

### Найти дубликаты (один телефон у нескольких пользователей)

```sql
SELECT phone, COUNT(*) AS cnt
FROM users
GROUP BY phone
HAVING COUNT(*) > 1;

-- С деталями
SELECT u.*
FROM users u
JOIN (
    SELECT phone FROM users GROUP BY phone HAVING COUNT(*) > 1
) dups ON dups.phone = u.phone;
```

### Пользователи без поездок (LEFT JOIN + IS NULL)

```sql
-- Вариант 1: LEFT JOIN
SELECT u.id, u.name
FROM users u
LEFT JOIN rides r ON r.passenger_id = u.id
WHERE r.id IS NULL;

-- Вариант 2: NOT EXISTS (часто быстрее)
SELECT id, name FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM rides r WHERE r.passenger_id = u.id
);

-- Вариант 3: NOT IN (осторожно с NULL!)
SELECT id, name FROM users
WHERE id NOT IN (SELECT passenger_id FROM rides WHERE passenger_id IS NOT NULL);
```

### Найти поездки без оплаты

```sql
SELECT r.id, r.passenger_id, r.price, r.status
FROM rides r
LEFT JOIN payments p ON p.ride_id = r.id AND p.status = 'completed'
WHERE r.status = 'completed'
  AND p.id IS NULL;  -- нет успешной оплаты
```

### Процент выполненных поездок по водителю

```sql
SELECT
    driver_id,
    COUNT(*)                                                   AS total,
    COUNT(*) FILTER (WHERE status = 'completed')               AS completed,
    ROUND(
        COUNT(*) FILTER (WHERE status = 'completed') * 100.0 / COUNT(*),
        1
    )                                                          AS completion_rate
FROM rides
WHERE driver_id IS NOT NULL
GROUP BY driver_id
ORDER BY completion_rate DESC;
```

---

## 6. Анализ запросов — EXPLAIN

```sql
EXPLAIN ANALYZE
SELECT r.*, u.name
FROM rides r
JOIN users u ON u.id = r.passenger_id
WHERE r.status = 'completed'
  AND r.created_at >= '2024-01-01';
```

**Что смотреть:**

| Узел плана | Что означает | Хорошо/Плохо |
|---|---|---|
| `Seq Scan` | Полный перебор таблицы | Плохо для больших таблиц |
| `Index Scan` | Использует индекс | Хорошо |
| `Index Only Scan` | Все данные из индекса, таблица не читается | Отлично |
| `Bitmap Heap Scan` | Индекс + батч чтение страниц | Норм для диапазонов |
| `Hash Join` | Хеш-соединение | Норм для больших таблиц |
| `Nested Loop` | Вложенный цикл | Хорошо только для малых наборов |

**Красные флаги в EXPLAIN:**
```
-- Плохо: большой rows × loops
->  Seq Scan on rides  (cost=0.00..15420.00 rows=500000 ...)
      Filter: (status = 'completed')
      Rows Removed by Filter: 450000   ← убрал 90% строк без индекса

-- Хорошо: индекс работает
->  Index Scan using idx_rides_status_created on rides
      Index Cond: ((status = 'completed') AND (created_at >= '2024-01-01'))
```

**Как исправить:**
```sql
-- Добавить недостающий индекс
CREATE INDEX CONCURRENTLY idx_rides_status_created
    ON rides(status, created_at DESC);

-- CONCURRENTLY — создание без блокировки таблицы (важно для prod)
```

---

## 7. PRIMARY KEY vs UNIQUE INDEX

| | PRIMARY KEY | UNIQUE INDEX |
|---|---|---|
| Количество | Один на таблицу | Сколько угодно |
| NULL | Запрещён | Разрешён (несколько NULL считаются уникальными в PostgreSQL) |
| Кластеризация | В некоторых СУБД — да (MySQL InnoDB) | Нет |
| Внешние ключи | Обычно ссылаются на PK | Могут ссылаться и на UNIQUE |
| Создаёт индекс | Да (автоматически) | Да (это и есть индекс) |

```sql
-- PRIMARY KEY = NOT NULL + UNIQUE + автоматический индекс
ALTER TABLE users ADD CONSTRAINT users_pkey PRIMARY KEY (id);

-- UNIQUE INDEX — можно на несколько колонок
CREATE UNIQUE INDEX idx_ride_reviews_unique
    ON ride_reviews(ride_id, author_id);

-- Отличие от UNIQUE CONSTRAINT: индекс гибче (можно добавить WHERE)
CREATE UNIQUE INDEX idx_active_plate
    ON driver_profiles(car_plate) WHERE is_active = TRUE;
```

---

## 8. Частые вопросы на интервью (Q&A)

**Q: Чем LEFT JOIN отличается от INNER JOIN?**

INNER JOIN возвращает только строки с совпадением в обеих таблицах. LEFT JOIN — все строки из левой таблицы, и NULL в колонках правой если совпадения нет. Используй LEFT JOIN когда нужно сохранить все записи из одной таблицы независимо от наличия связанных данных.

**Q: Когда NOT EXISTS лучше NOT IN?**

NOT IN опасен при NULL в подзапросе: `WHERE id NOT IN (SELECT user_id FROM ...)` вернёт пустой результат если хотя бы один `user_id = NULL` (NULL IN (...) = UNKNOWN, NOT UNKNOWN = UNKNOWN). NOT EXISTS безопасен с NULL и часто лучше оптимизируется.

**Q: Как найти N-ю по величине запись?**

```sql
-- Топ-3 поездки по цене
SELECT price FROM rides ORDER BY price DESC LIMIT 1 OFFSET 2;  -- 3-я

-- Через оконную функцию (надёжнее при дублях)
SELECT * FROM (
    SELECT *, DENSE_RANK() OVER (ORDER BY price DESC) AS rk FROM rides
) t WHERE rk = 3;
```

**Q: Как оптимизировать медленный запрос?**

1. `EXPLAIN ANALYZE` — смотрю план, ищу Seq Scan на больших таблицах
2. Проверяю есть ли индекс на колонках в WHERE/JOIN/ORDER BY
3. Смотрю нет ли функций над индексируемой колонкой (`WHERE LOWER(email) = ...` — индекс не используется)
4. Проверяю типы данных в JOIN (несовпадение типов ломает индекс)
5. Если запрос сложный — разбиваю на CTE, смотрю на каком этапе тормоз
6. Для агрегаций по большим таблицам — рассматриваю партиционирование или материализованные представления
