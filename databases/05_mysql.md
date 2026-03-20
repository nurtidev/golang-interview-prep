# MySQL: Архитектура и особенности

---

## Лекция

### 1. Архитектура MySQL

MySQL имеет слоистую архитектуру:

```
Клиент
    |
Connection Layer (аутентификация, пул потоков)
    |
SQL Layer (парсер, оптимизатор запросов, кеш)
    |
Storage Engine Layer
    |
    ├── InnoDB (по умолчанию) — ACID, FK, MVCC
    ├── MyISAM (устаревший) — быстрый, нет транзакций, нет FK
    └── Memory — только RAM, нет персистентности
```

**Ключевое отличие от PostgreSQL:** сменные Storage Engine. InnoDB — де-факто стандарт с MySQL 5.5.

---

### 2. InnoDB: внутреннее устройство

**Страницы:** базовая единица I/O — 16KB (vs 8KB в PostgreSQL).

**Buffer Pool:**
- Кеш страниц данных и индексов в памяти
- Аналог Shared Buffer Cache в PostgreSQL
- Рекомендуется: 70-80% RAM
- `innodb_buffer_pool_size = 8G`

**Tablespace:**
- system tablespace: `ibdata1` — системные данные
- file-per-table: каждая таблица в `.ibd` файле (рекомендуется, `innodb_file_per_table=ON`)

**Clustered Index (кластерный индекс):**
- В InnoDB строки данных физически хранятся в B-Tree индексе PRIMARY KEY
- PK = кластерный индекс. Данные и индекс — одна структура
- Secondary index хранит: значение индексируемой колонки + значение PK (не указатель на строку)

```
PRIMARY KEY (id) — clustered:
B-Tree: [id: 1 → {все данные строки}, id: 2 → {...}, ...]

INDEX ON (email) — secondary:
B-Tree: [email: alice@... → pk=1, email: bob@... → pk=5, ...]
```

**Следствие:** secondary index lookup = 2 поиска (secondary → PK значение → clustered). Это называется "double lookup" или "bookmark lookup".

---

### 3. Особенности индексов в MySQL

**Выбор PRIMARY KEY — критически важен:**
```sql
-- ПЛОХО: UUID как PK (random вставки → фрагментация B-Tree)
CREATE TABLE users (
    id CHAR(36) PRIMARY KEY,  -- случайный UUID
    ...
);

-- ХОРОШО: автоинкрементный INT/BIGINT
CREATE TABLE users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid CHAR(36) UNIQUE,  -- UUID как secondary unique index
    ...
);

-- Компромисс: UUID v7 (временно-сортированный) или ULID
```

**Covering Index:** если secondary index содержит все нужные колонки запроса — не нужен double lookup.

```sql
-- Запрос: SELECT email, created_at FROM users WHERE status = 'active'
-- Covering index: все нужные колонки есть в индексе
CREATE INDEX idx_status_email_date ON users(status, email, created_at);
-- MySQL сделает Index Only Scan (нет double lookup)
```

**Prefix Index для текстовых колонок:**
```sql
-- Вместо индекса по всему TEXT/VARCHAR(255)
CREATE INDEX idx_email ON users(email(20));  -- только первые 20 символов
```

---

### 4. MVCC в MySQL/InnoDB

InnoDB реализует MVCC через **undo log:**
- При UPDATE строки: текущая версия остаётся в таблице, старая версия записывается в undo log
- Транзакция читает нужную версию из undo log по transaction ID
- Undo log очищается когда нет транзакций, которым нужна старая версия (purge thread)

**Read View:** в начале транзакции создаётся список активных транзакций (read view). Транзакция видит данные, зафиксированные до создания read view.

---

### 5. Репликация в MySQL

**Statement-Based Replication (SBR):** реплицируется SQL запрос.
- Компактно
- Проблема: недетерминированные функции (`NOW()`, `UUID()`, `RAND()`) дают разные результаты

**Row-Based Replication (RBR):** реплицируются изменённые строки.
- Надёжно, работает с любыми запросами
- Больший объём binlog
- По умолчанию с MySQL 5.7.7

**Mixed:** автоматически выбирает SBR или RBR.

**GTID (Global Transaction ID):**
- Уникальный ID для каждой транзакции: `server_uuid:transaction_id`
- Позволяет легко определить, что реплика получила
- Упрощает failover и настройку репликации

**Group Replication / InnoDB Cluster:**
- Multi-master репликация с автоматическим failover
- Требует `group_replication_consensus` (Paxos-based)

---

### 6. MySQL vs PostgreSQL: ключевые различия

| Аспект | MySQL | PostgreSQL |
|--------|-------|-----------|
| Архитектура | Подключаемые движки | Единый движок |
| Кластерный индекс | Да (InnoDB) | Нет (heap tables) |
| Партиционирование | Базовое | Продвинутое |
| JSONB | JSON (хуже) | JSONB (бинарный, индексируемый) |
| Full-text search | Встроен | Встроен + tsvector |
| Репликация | Statement/Row/Mixed | WAL streaming + logical |
| Window functions | MySQL 8.0+ | Давно есть |
| CTE | MySQL 8.0+ | Давно есть |
| Типы данных | Меньше | Богаче (arrays, ranges, etc.) |
| Производительность чтения | Быстрее на простых запросах | Лучше на сложных |

---

### 7. Оптимизация и диагностика

```sql
-- Медленные запросы
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- > 1 секунды

-- Анализ запроса
EXPLAIN FORMAT=JSON SELECT ...;

-- Статистика InnoDB
SHOW ENGINE INNODB STATUS;

-- Статистика таблиц
SELECT
    table_name,
    table_rows,
    ROUND(data_length / 1024 / 1024, 2) AS data_mb,
    ROUND(index_length / 1024 / 1024, 2) AS index_mb
FROM information_schema.tables
WHERE table_schema = 'mydb'
ORDER BY data_length DESC;
```

---

## Практические примеры

### Пример 1: Работа с MySQL из Go (sqlx)

```go
import (
    "github.com/jmoiron/sqlx"
    _ "github.com/go-sql-driver/mysql"
)

type UserRepository struct {
    db *sqlx.DB
}

func NewUserRepo(dsn string) (*UserRepository, error) {
    db, err := sqlx.Open("mysql", dsn)
    if err != nil {
        return nil, err
    }

    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(25)
    db.SetConnMaxLifetime(5 * time.Minute)

    return &UserRepository{db: db}, nil
}

type User struct {
    ID        int64     `db:"id"`
    Email     string    `db:"email"`
    Status    string    `db:"status"`
    CreatedAt time.Time `db:"created_at"`
}

func (r *UserRepository) GetActiveUsers(ctx context.Context, limit int) ([]User, error) {
    var users []User
    err := r.db.SelectContext(ctx, &users,
        `SELECT id, email, status, created_at
         FROM users
         WHERE status = 'active'
         ORDER BY created_at DESC
         LIMIT ?`,
        limit,
    )
    return users, err
}
```

### Пример 2: Пагинация — правильный подход

```sql
-- ПЛОХО: OFFSET медленный на больших данных
-- OFFSET 10000 → MySQL читает 10000 строк и выбрасывает
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 10000;

-- ХОРОШО: keyset pagination (cursor-based)
-- Первая страница
SELECT * FROM orders ORDER BY id LIMIT 20;

-- Следующая страница (cursor = последний id предыдущей страницы)
SELECT * FROM orders WHERE id > :last_id ORDER BY id LIMIT 20;
```

### Пример 3: Fulltext search в MySQL

```sql
-- Создать FULLTEXT индекс
ALTER TABLE articles ADD FULLTEXT INDEX idx_fts (title, content);

-- Поиск в natural language mode
SELECT *, MATCH(title, content) AGAINST ('mysql performance') AS relevance
FROM articles
WHERE MATCH(title, content) AGAINST ('mysql performance')
ORDER BY relevance DESC;

-- Boolean mode
SELECT * FROM articles
WHERE MATCH(title, content) AGAINST ('+mysql -oracle performance*' IN BOOLEAN MODE);
-- + обязательное слово
-- - исключить
-- * префикс
```

---

## Вопросы на собеседовании

### Q: Что такое кластерный индекс в InnoDB? В чём отличие от вторичного?

**Ответ:** Кластерный индекс — данные строки физически хранятся в B-Tree самого индекса (PRIMARY KEY). Один кластерный индекс на таблицу. Если PK не задан, InnoDB создаёт скрытый автоинкрементный PK. Вторичный индекс хранит значение индексируемых колонок + значение PK (не rowid). Lookup по вторичному индексу = 2 поиска: сначала в secondary index → получить PK → потом в clustered index. Это double lookup, который можно избежать через covering index.

---

### Q: Почему UUID — плохой выбор для PRIMARY KEY в InnoDB?

**Ответ:** UUID случаен — новые строки вставляются в случайные места B-Tree. Это вызывает: 1) частые page splits (страница делится при переполнении), 2) сильную фрагментацию, 3) buffer pool промахи (нужная страница редко в кеше). Автоинкрементный PK всегда вставляется в конец → нет splits, нет фрагментации. Если нужен UUID — используй UUID v7 (time-sortable) или хранить как BIGINT + отдельный VARCHAR column для UUID.

---

### Q: В чём разница между MyISAM и InnoDB?

**Ответ:** MyISAM — старый движок: нет транзакций, нет Foreign Keys, нет MVCC. Блокировки на уровне таблицы (не строки). Быстрое чтение для read-heavy workloads. Таблица повреждается при сбое. InnoDB — современный стандарт: ACID транзакции, Foreign Keys, MVCC (row-level locking), crash recovery через redo log. Используй только InnoDB.

---

### Q: Как работает репликация в MySQL?

**Ответ:** Primary записывает все изменения в binary log (binlog). Replica запускает два потока: IO thread — читает binlog с primary и сохраняет в relay log. SQL thread — воспроизводит события из relay log. По умолчанию асинхронная — replica может отставать. Semisynchronous replication: primary ждёт подтверждения хотя бы от одной реплики перед commit. GTID (Global Transaction ID) — уникальный ID для каждой транзакции, упрощает failover.

---

### Q: Как MySQL оптимизатор выбирает индекс?

**Ответ:** Оптимизатор использует статистику (cardinality — кол-во уникальных значений) для оценки стоимости разных планов. Иногда выбирает неоптимальный индекс из-за устаревшей статистики. Как помочь: `ANALYZE TABLE` — обновить статистику. `FORCE INDEX (idx_name)` — принудить использовать конкретный индекс (крайнее средство). `EXPLAIN` — посмотреть выбранный план. Оптимизатор может не использовать индекс если: функция над колонкой, неявный cast типов, низкая cardinality (< ~20% уникальных значений).
