# System Design: Travel Platform (Halyk Travel)

> Контекст: Halyk Travel — travel-продукт от Halyk Bank. Бронирование авиабилетов, отелей,
> интеграция с платёжной системой банка. Аудитория — клиенты Halyk Bank (~8M пользователей).

---

## Вопрос: "Спроектируй систему поиска и бронирования авиабилетов"

Это классический **open-ended system design** вопрос. Структура ответа важнее деталей.

---

## 1. Clarifying Questions (всегда начинай с этого)

Перед проектированием уточни scope:

- Только авиабилеты или отели/трансферы тоже?
- Сколько пользователей одновременно? DAU?
- Интегрируемся с внешними GDS (Amadeus, Sabre) или имеем свой инвентарь?
- Нужен ли real-time поиск или можно кешировать?
- Мультивалютность? Только тенге?
- Какая SLA на поиск? (< 3s? < 1s?)

**Примерные параметры для Halyk Travel:**
- 50k DAU (клиенты Halyk Bank), 500 rps в пик
- Интеграция с 2-3 внешними провайдерами (Amadeus + локальные)
- Поиск < 3 секунды, бронирование < 5 секунд
- Платежи через Halyk Pay (уже существующая система)

---

## 2. Высокоуровневая архитектура

```
Client (iOS/Android/Web)
         │
    API Gateway (auth, rate limiting, routing)
         │
    ┌────┴────────────────────────────┐
    │                                 │
Search Service              Booking Service
    │                                 │
    ├── Cache (Redis)        ├── Payment Service (Halyk Pay)
    │                        ├── Notification Service
    └── Provider Adapter     └── DB (PostgreSQL)
         ├── Amadeus GDS
         ├── Local Provider
         └── Cache Warmer
                              ┌────────────────┐
                              │  Kafka          │
                              │  (events bus)   │
                              └────────────────┘
```

---

## 3. Search Service — детальный дизайн

### Проблема: внешние провайдеры медленные и ненадёжные

Amadeus/Sabre API: latency 500ms-2s, доступность ~99.5%.

**Решение — многоуровневый кеш:**

```
Запрос: MAD → ALA, 15 мая

L1: Redis (горячий кеш, 10 мин TTL)
    ключ: search:{from}:{to}:{date}:{class}
    ├── HIT → сразу возвращаем (< 10ms)
    └── MISS ↓

L2: Параллельный запрос к провайдерам (fan-out)
    ├── Amadeus (таймаут 2.5s)
    ├── Local Provider (таймаут 2.5s)
    └── Aggregate + Dedup → Redis L1
```

**Схема Redis ключа:**
```
search:ALA:MAD:2026-05-15:economy → JSON([]Flight) TTL=10min
```

**Cache Warmer** — фоновый процесс, который прогревает кеш для популярных маршрутов:
```go
// Топ 100 маршрутов из Казахстана — обновляем каждые 5 минут
popularRoutes := []Route{
    {From: "ALA", To: "NQZ"},
    {From: "ALA", To: "TSE"},
    {From: "ALA", To: "MOW"},
    // ...
}
```

### Почему не храним рейсы в своей БД?

Цены на авиабилеты меняются каждые минуты. Хранить — значит постоянно синхронизировать. Дешевле кешировать результаты поиска с коротким TTL.

---

## 4. Booking Service — детальный дизайн

### Критичные требования для финтех:
1. **Идемпотентность** — клиент может повторить запрос
2. **Атомарность** — списание денег + бронь = одна транзакция
3. **Seat lock** — место нельзя продать дважды
4. **Audit log** — каждое действие логируется (требование регулятора)

### Seat Locking — временная блокировка места

Пользователь нашёл билет, идёт на страницу оплаты. Нужно "придержать" место на 15 минут.

```
Flow:
1. SearchResult → клиент выбирает рейс
2. POST /bookings/hold → создаём hold (TTL 15 мин в Redis)
3. Пользователь вводит данные / оплачивает
4. POST /bookings/confirm → финализируем бронь
5. Если не подтвердил за 15 мин → hold истекает автоматически
```

```go
// Redis key для hold
holdKey := fmt.Sprintf("flight:hold:%s:%s", flightID, seatID)
redis.SetNX(ctx, holdKey, userID, 15*time.Minute)
```

**Проблема:** что если Redis упал в момент подтверждения?
> Финальное бронирование — только через PostgreSQL транзакцию (`SELECT FOR UPDATE`). Redis hold — лишь UX-слой. Истина — в базе.

### Saga Pattern для распределённой транзакции

Бронирование включает:
1. Hold места у внешнего провайдера
2. Списание денег через Halyk Pay
3. Запись в нашу БД
4. Отправка электронного билета

Если шаг 3 упал после шага 2 → нужен rollback (возврат денег).

```
Saga:
STEP 1: hold_seat(flightID, seatID)
    compensate: release_seat(flightID, seatID)

STEP 2: charge_payment(userID, amount)
    compensate: refund_payment(userID, amount)

STEP 3: create_booking_record(...)
    compensate: mark_booking_failed(bookingID)

STEP 4: send_ticket(email, bookingID)
    compensate: (нет — письмо уже отправлено, логируем)
```

```go
type SagaStep struct {
    Execute    func(ctx context.Context) error
    Compensate func(ctx context.Context) error
}

func runSaga(ctx context.Context, steps []SagaStep) error {
    var executed []SagaStep
    for _, step := range steps {
        if err := step.Execute(ctx); err != nil {
            // Откатываем в обратном порядке
            for i := len(executed) - 1; i >= 0; i-- {
                if cErr := executed[i].Compensate(ctx); cErr != nil {
                    // Логируем — manual intervention needed
                    log.Error("compensation failed", "err", cErr)
                }
            }
            return err
        }
        executed = append(executed, step)
    }
    return nil
}
```

---

## 5. Schema дизайн (PostgreSQL)

```sql
-- Бронирования
CREATE TABLE bookings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,
    flight_id       VARCHAR(50) NOT NULL,    -- ID у провайдера
    provider        VARCHAR(50) NOT NULL,     -- amadeus, local
    idempotency_key VARCHAR(255) UNIQUE NOT NULL,
    status          VARCHAR(20) NOT NULL,     -- pending, confirmed, cancelled, refunded
    amount          NUMERIC(12, 2) NOT NULL,
    currency        VARCHAR(3) DEFAULT 'KZT',
    passenger_data  JSONB NOT NULL,           -- имена, паспорта (encrypted)
    provider_pnr    VARCHAR(20),              -- локатор у провайдера
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Индексы
CREATE INDEX idx_bookings_user_id ON bookings(user_id);
CREATE INDEX idx_bookings_status ON bookings(status) WHERE status IN ('pending', 'confirmed');
CREATE INDEX idx_bookings_created_at ON bookings(created_at DESC);

-- Audit log (append-only, никогда не удаляем)
CREATE TABLE booking_events (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id  UUID NOT NULL REFERENCES bookings(id),
    event_type  VARCHAR(50) NOT NULL,  -- created, payment_initiated, confirmed, cancelled
    payload     JSONB,
    actor_id    UUID,                  -- userID или system
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_booking_events_booking_id ON booking_events(booking_id);
```

---

## 6. Provider Integration — Adapter Pattern

Разные GDS имеют разные API. Абстрагируемся:

```go
// Единый интерфейс для всех провайдеров
type FlightProvider interface {
    Search(ctx context.Context, req SearchRequest) ([]Flight, error)
    Hold(ctx context.Context, flightID string, passengers []Passenger) (HoldToken, error)
    Confirm(ctx context.Context, holdToken HoldToken, paymentRef string) (*Booking, error)
    Cancel(ctx context.Context, pnr string) error
}

// Amadeus реализация
type AmadeusProvider struct {
    client  *amadeus.Client
    limiter *RateLimiter // 100 rps limit от Amadeus
}

func (a *AmadeusProvider) Search(ctx context.Context, req SearchRequest) ([]Flight, error) {
    if err := a.limiter.Wait(ctx); err != nil {
        return nil, err
    }
    // Маппинг нашей модели → Amadeus API формат
    amadeusReq := mapToAmadeusRequest(req)
    resp, err := a.client.FlightOffersSearch(ctx, amadeusReq)
    if err != nil {
        return nil, fmt.Errorf("amadeus search: %w", err)
    }
    return mapFromAmadeusResponse(resp), nil
}
```

**Retry с exponential backoff:**
```go
func withRetry(ctx context.Context, maxAttempts int, fn func() error) error {
    backoff := 100 * time.Millisecond
    for i := 0; i < maxAttempts; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        // Не ретраим если контекст отменён или бизнес-ошибка
        if errors.Is(err, context.Canceled) || isBusinessError(err) {
            return err
        }
        select {
        case <-time.After(backoff):
            backoff *= 2 // 100ms → 200ms → 400ms
        case <-ctx.Done():
            return ctx.Err()
        }
    }
    return fmt.Errorf("max attempts reached")
}
```

---

## 7. Observability

**Метрики (Prometheus):**
```go
var (
    searchDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "search_duration_seconds",
            Buckets: []float64{0.1, 0.5, 1.0, 2.0, 3.0, 5.0},
        },
        []string{"provider", "status"},
    )
    
    bookingTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "bookings_total"},
        []string{"status", "provider"},
    )
    
    cacheHitRatio = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{Name: "search_cache_hit_ratio"},
        []string{"route_type"}, // popular vs rare
    )
)
```

**Важные алерты:**
- `search_p99_latency > 3s` → будим дежурного
- `booking_failure_rate > 1%` → критично (деньги)
- `provider_error_rate > 10%` → провайдер деградирует

**Structured logging:**
```go
logger.Info("booking created",
    "booking_id", booking.ID,
    "user_id", userID,
    "flight_id", flightID,
    "amount", amount,
    "provider", provider,
    "duration_ms", time.Since(start).Milliseconds(),
)
```

---

## 8. Вопросы которые могут задать

**Q: Как обрабатываешь ситуацию когда Amadeus вернул билет, но при подтверждении место уже занято?**
> Это называется "price/availability guarantee window". Amadeus даёт `offer validity` — обычно 5-10 минут. Если истёк — нужно повторить поиск и показать пользователю актуальную цену. На уровне UX — показываем таймер обратного отсчёта.

**Q: Как масштабировать Search Service при Black Friday?**
> 1. Horizontal scaling Search Service (stateless) 2. Агрессивнее прогреваем кеш (Cache Warmer работает чаще) 3. Rate limiting на уровне API Gateway 4. Circuit Breaker перед провайдерами 5. Очередь запросов с backpressure вместо прямых вызовов

**Q: Как хранить данные паспортов безопасно?**
> Шифрование на уровне приложения (AES-256) перед записью в БД. Ключи в Vault. В логах — только masked данные (первые/последние 3 символа). Отдельная таблица с жёстким ACL. Соответствие требованиям GDPR/местного регулятора.

**Q: Нужен ли нам Event Sourcing для бронирований?**
> Для Halyk Travel — нет, это over-engineering. Достаточно `booking_events` таблицы (append-only audit log) + current state в `bookings`. Event Sourcing оправдан когда нужно воспроизводить историческое состояние или сложная проекция событий.
