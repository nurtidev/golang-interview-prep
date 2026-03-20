# Функции и Хранимые Процедуры

---

## Лекция

### 1. Что это и зачем

**Хранимая процедура (Stored Procedure)** — именованный блок SQL-кода, сохранённый в БД. Вызывается по имени, выполняется на стороне сервера БД.

**Функция (Function)** — то же самое, но обязана вернуть значение и может использоваться прямо в SQL-запросе.

**Зачем нужны:**
- Логика выполняется рядом с данными — нет сетевых round-trip между приложением и БД
- Переиспользование сложной логики
- Безопасность — приложение не имеет прямого доступа к таблицам, только вызывает процедуры
- Атомарность — несколько операций в одной транзакции

---

### 2. Функция vs Процедура

| | Функция | Процедура |
|--|---------|-----------|
| Возвращает значение | Обязательно | Необязательно (OUT параметры) |
| Использование в SELECT | Да | Нет |
| Транзакция внутри | Нельзя управлять | Можно COMMIT/ROLLBACK |
| Вызов | `SELECT func()` | `CALL proc()` |
| Побочные эффекты | Нежелательны | Нормально |

---

### 3. Функции в PostgreSQL

**Простая функция:**
```sql
CREATE OR REPLACE FUNCTION add_numbers(a INT, b INT)
RETURNS INT
LANGUAGE sql
AS $$
    SELECT a + b;
$$;

-- Использование прямо в запросе
SELECT add_numbers(3, 5);  -- 8
SELECT id, add_numbers(price, tax) AS total FROM orders;
```

**Функция с логикой (PL/pgSQL):**
```sql
CREATE OR REPLACE FUNCTION get_discount(price NUMERIC, user_level INT)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
DECLARE
    discount NUMERIC := 0;
BEGIN
    IF user_level >= 3 THEN
        discount := price * 0.20;  -- 20% для VIP
    ELSIF user_level >= 1 THEN
        discount := price * 0.10;  -- 10% для обычных
    END IF;

    RETURN discount;
END;
$$;

SELECT price, get_discount(price, user_level) FROM users;
```

**Функция возвращающая таблицу (RETURNS TABLE):**
```sql
CREATE OR REPLACE FUNCTION get_user_orders(p_user_id INT)
RETURNS TABLE(order_id INT, amount NUMERIC, status TEXT)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
        SELECT o.id, o.amount, o.status
        FROM orders o
        WHERE o.user_id = p_user_id
        ORDER BY o.created_at DESC;
END;
$$;

-- Используется как таблица
SELECT * FROM get_user_orders(42);
SELECT * FROM get_user_orders(42) WHERE status = 'paid';
```

---

### 4. Хранимые процедуры в PostgreSQL (с PostgreSQL 11)

```sql
CREATE OR REPLACE PROCEDURE transfer_money(
    from_account INT,
    to_account   INT,
    amount       NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
    -- Списываем
    UPDATE accounts
    SET balance = balance - amount
    WHERE id = from_account;

    -- Проверяем что баланс не ушёл в минус
    IF (SELECT balance FROM accounts WHERE id = from_account) < 0 THEN
        RAISE EXCEPTION 'Недостаточно средств на счёте %', from_account;
        -- RAISE EXCEPTION автоматически делает ROLLBACK
    END IF;

    -- Зачисляем
    UPDATE accounts
    SET balance = balance + amount
    WHERE id = to_account;

    COMMIT;  -- процедура может управлять транзакцией
END;
$$;

-- Вызов
CALL transfer_money(1, 2, 1000.00);
```

---

### 5. Триггеры (Triggers)

Триггер — функция которая вызывается **автоматически** при INSERT / UPDATE / DELETE.

```sql
-- Шаг 1: создаём функцию-триггер (возвращает TRIGGER)
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = NOW();  -- NEW — новая версия строки
    RETURN NEW;
END;
$$;

-- Шаг 2: вешаем триггер на таблицу
CREATE TRIGGER set_updated_at
    BEFORE UPDATE ON users          -- BEFORE: до изменения
    FOR EACH ROW                    -- для каждой изменённой строки
    EXECUTE FUNCTION update_updated_at();
```

**Аудит через триггер:**
```sql
CREATE OR REPLACE FUNCTION audit_changes()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO audit_log(table_name, operation, old_data, new_data, changed_at)
    VALUES (
        TG_TABLE_NAME,           -- имя таблицы
        TG_OP,                   -- 'INSERT', 'UPDATE', 'DELETE'
        row_to_json(OLD),        -- старая строка
        row_to_json(NEW),        -- новая строка
        NOW()
    );
    RETURN NEW;
END;
$$;

CREATE TRIGGER orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION audit_changes();
```

**Переменные триггера:**
```
NEW  — новая версия строки (доступна в INSERT, UPDATE)
OLD  — старая версия строки (доступна в UPDATE, DELETE)
TG_OP        — операция: 'INSERT' | 'UPDATE' | 'DELETE'
TG_TABLE_NAME — имя таблицы
```

---

### 6. Языки написания функций

```sql
LANGUAGE sql      -- чистый SQL, простые случаи, компилятор может инлайнить
LANGUAGE plpgsql  -- процедурный язык PostgreSQL: IF/ELSE, циклы, переменные
LANGUAGE plpython -- Python внутри PostgreSQL
LANGUAGE plv8     -- JavaScript (расширение)
```

**Когда что использовать:**
- `sql` — простые вычисления без условий, максимальная производительность
- `plpgsql` — сложная логика, условия, циклы, обработка ошибок

---

### 7. Плюсы и минусы

**Плюсы:**
- Нет сетевых round-trip — логика выполняется рядом с данными
- Атомарность сложных операций
- Безопасность — можно закрыть прямой доступ к таблицам
- Переиспользование логики между разными сервисами

**Минусы:**
- Логика размазана между приложением и БД — сложнее поддерживать
- Сложно тестировать и дебажить
- Версионирование — нет нормального git diff для SQL в БД
- Масштабирование — логика на стороне БД нагружает БД, а не приложение
- Переносимость — PL/pgSQL не работает в MySQL

---

### 8. Вызов из Go

```go
// Вызов функции
var total float64
err := db.QueryRowContext(ctx,
    "SELECT get_discount($1, $2)",
    price, userLevel,
).Scan(&total)

// Вызов функции возвращающей таблицу
rows, err := db.QueryContext(ctx,
    "SELECT * FROM get_user_orders($1)",
    userID,
)
defer rows.Close()
for rows.Next() {
    var order Order
    rows.Scan(&order.ID, &order.Amount, &order.Status)
}

// Вызов процедуры
_, err = db.ExecContext(ctx,
    "CALL transfer_money($1, $2, $3)",
    fromAccount, toAccount, amount,
)
```

---

## Вопросы на собеседовании

### Q: В чём разница между функцией и хранимой процедурой?

**Ответ:** Функция обязана вернуть значение и может использоваться в SELECT прямо как выражение. Процедура не обязана возвращать значение, зато может управлять транзакциями (COMMIT/ROLLBACK внутри). Функцию вызывают через SELECT, процедуру через CALL. Функции нежелательно иметь побочные эффекты (INSERT/UPDATE), хотя технически возможно.

---

### Q: Что такое триггер? Приведи пример использования.

**Ответ:** Триггер — функция которая автоматически вызывается при INSERT, UPDATE или DELETE. BEFORE-триггер может модифицировать данные до записи (например, автоматически заполнить `updated_at`). AFTER-триггер срабатывает после изменения (например, записать в таблицу аудита). Внутри доступны `NEW` (новая версия строки) и `OLD` (старая версия).

---

### Q: Почему функции и процедуры считаются антипаттерном в некоторых командах?

**Ответ:** Бизнес-логика размазывается между приложением и БД — непонятно где искать проблему. Сложно тестировать: нет unit-тестов в привычном смысле. Версионирование неудобное: нет нормального git diff, миграции нужны. Масштабирование страдает: можно добавить инстансы приложения, но БД одна. Переносимость нулевая: PL/pgSQL привязан к PostgreSQL. При этом для аудита и атомарных финансовых операций они оправданы.

---

### Q: Когда стоит использовать хранимые процедуры?

**Ответ:** Оправданы когда: нужна атомарность сложной операции с несколькими таблицами (перевод денег), нужен аудит изменений через триггеры, несколько разных приложений/сервисов работают с одной БД и логика должна быть централизована, критична минимальная latency и нельзя позволить лишние round-trip до БД.
