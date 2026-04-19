# Go: Тестирование

---

## 1. Table-Driven Tests — основной паттерн

```go
func Add(a, b int) int { return a + b }

func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"positive", 2, 3, 5},
        {"negative", -1, -2, -3},
        {"zero", 0, 0, 0},
        {"mixed", -1, 1, 0},
    }

    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            got := Add(tc.a, tc.b)
            if got != tc.want {
                t.Errorf("Add(%d, %d) = %d, want %d", tc.a, tc.b, got, tc.want)
            }
        })
    }
}
```

**Запуск:**
```bash
go test ./...                        # все тесты
go test -run TestAdd                 # конкретный тест
go test -run TestAdd/positive        # конкретный subtest
go test -v ./...                     # verbose
go test -count=1 ./...               # без кеша
go test -timeout 30s ./...           # таймаут
```

---

## 2. testify — стандарт индустрии

```go
import (
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestUser(t *testing.T) {
    user, err := NewUser("alice", "alice@example.com")

    // require — останавливает тест при failure (fatal)
    require.NoError(t, err)
    require.NotNil(t, user)

    // assert — продолжает тест при failure
    assert.Equal(t, "alice", user.Name)
    assert.Equal(t, "alice@example.com", user.Email)
    assert.True(t, user.CreatedAt.Before(time.Now()))

    // Проверка ошибок
    _, err = NewUser("", "")
    assert.ErrorIs(t, err, ErrInvalidInput)
    assert.ErrorContains(t, err, "name")
}
```

**assert vs require:**
- `assert` — продолжает выполнение после провала (для независимых проверок)
- `require` — сразу завершает тест (для предусловий)

---

## 3. Интерфейсы для тестируемости

```go
// Плохо: прямая зависимость на конкретный тип
type UserService struct {
    db *sql.DB  // нельзя подменить в тесте
}

// Хорошо: зависимость на интерфейс
type UserRepository interface {
    GetUser(ctx context.Context, id int64) (*User, error)
    CreateUser(ctx context.Context, u *User) error
}

type UserService struct {
    repo UserRepository  // подменяем в тесте
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}
```

---

## 4. Моки — ручная реализация

```go
// Реализуем интерфейс вручную для теста
type mockUserRepo struct {
    users map[int64]*User
    err   error  // возвращать эту ошибку
}

func (m *mockUserRepo) GetUser(_ context.Context, id int64) (*User, error) {
    if m.err != nil {
        return nil, m.err
    }
    u, ok := m.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return u, nil
}

func (m *mockUserRepo) CreateUser(_ context.Context, u *User) error {
    if m.err != nil {
        return m.err
    }
    m.users[u.ID] = u
    return nil
}

// В тесте:
func TestUserService_GetUser(t *testing.T) {
    repo := &mockUserRepo{
        users: map[int64]*User{
            1: {ID: 1, Name: "Alice"},
        },
    }
    svc := NewUserService(repo)

    user, err := svc.GetUser(context.Background(), 1)
    require.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
}

func TestUserService_GetUser_NotFound(t *testing.T) {
    repo := &mockUserRepo{users: map[int64]*User{}}
    svc := NewUserService(repo)

    _, err := svc.GetUser(context.Background(), 999)
    assert.ErrorIs(t, err, ErrNotFound)
}
```

---

## 5. testify/mock — моки с проверкой вызовов

```go
import "github.com/stretchr/testify/mock"

type MockUserRepo struct {
    mock.Mock
}

func (m *MockUserRepo) GetUser(ctx context.Context, id int64) (*User, error) {
    args := m.Called(ctx, id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*User), args.Error(1)
}

func (m *MockUserRepo) CreateUser(ctx context.Context, u *User) error {
    args := m.Called(ctx, u)
    return args.Error(0)
}

func TestUserService_WithMock(t *testing.T) {
    repo := new(MockUserRepo)

    // Настраиваем ожидания
    repo.On("GetUser", mock.Anything, int64(1)).
        Return(&User{ID: 1, Name: "Alice"}, nil)

    // Можно указать сколько раз вызвать
    repo.On("CreateUser", mock.Anything, mock.AnythingOfType("*main.User")).
        Return(nil).Once()

    svc := NewUserService(repo)
    user, err := svc.GetUser(context.Background(), 1)

    require.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)

    // Проверяем что все ожидания выполнены
    repo.AssertExpectations(t)
}
```

---

## 6. HTTP тесты — httptest

```go
func TestHandler_GetUser(t *testing.T) {
    // Создаём тестовый сервер — без реального порта
    handler := NewUserHandler(mockService)
    ts := httptest.NewServer(handler)
    defer ts.Close()

    resp, err := http.Get(ts.URL + "/users/1")
    require.NoError(t, err)
    defer resp.Body.Close()

    assert.Equal(t, http.StatusOK, resp.StatusCode)

    var user User
    require.NoError(t, json.NewDecoder(resp.Body).Decode(&user))
    assert.Equal(t, int64(1), user.ID)
}

// Или через httptest.NewRecorder — без TCP:
func TestHandler_Direct(t *testing.T) {
    req := httptest.NewRequest(http.MethodGet, "/users/1", nil)
    w := httptest.NewRecorder()

    handler := NewUserHandler(mockService)
    handler.ServeHTTP(w, req)

    resp := w.Result()
    assert.Equal(t, http.StatusOK, resp.StatusCode)
}
```

---

## 7. Интеграционные тесты с testcontainers

```go
import "github.com/testcontainers/testcontainers-go/modules/postgres"

func TestUserRepo_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    ctx := context.Background()

    // Запускаем реальный PostgreSQL в Docker
    container, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:15"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready").
                WithStartupTimeout(30*time.Second),
        ),
    )
    require.NoError(t, err)
    defer container.Terminate(ctx)

    connStr, _ := container.ConnectionString(ctx, "sslmode=disable")
    db, err := sql.Open("pgx", connStr)
    require.NoError(t, err)

    // Запускаем миграции
    runMigrations(db)

    repo := NewUserRepo(db)
    user := &User{Name: "Alice", Email: "alice@example.com"}

    err = repo.CreateUser(ctx, user)
    require.NoError(t, err)
    assert.NotZero(t, user.ID)

    found, err := repo.GetUser(ctx, user.ID)
    require.NoError(t, err)
    assert.Equal(t, "Alice", found.Name)
}
```

```bash
# Запуск с интеграционными тестами
go test -v ./...

# Пропустить интеграционные (только unit)
go test -short ./...
```

---

## 8. Goleak — обнаружение goroutine утечек

```go
import "go.uber.org/goleak"

func TestMain(m *testing.M) {
    // Проверяет что после каждого теста нет лишних горутин
    goleak.VerifyTestMain(m)
}

// Или для конкретного теста:
func TestWorker(t *testing.T) {
    defer goleak.VerifyNone(t)

    w := NewWorker()
    w.Start()
    // если w.Stop() не вызван — goleak обнаружит утечку горутины
    w.Stop()
}
```

---

## 9. Бенчмарки

```go
func BenchmarkJSON(b *testing.B) {
    users := generateUsers(100)
    b.ResetTimer()
    b.ReportAllocs()

    for i := 0; i < b.N; i++ {
        data, err := json.Marshal(users)
        if err != nil {
            b.Fatal(err)
        }
        _ = data
    }
}

// Параллельный benchmark
func BenchmarkConcurrent(b *testing.B) {
    cache := NewCache()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            cache.Get("key")
        }
    })
}
```

```bash
go test -bench=. -benchmem -benchtime=5s
# BenchmarkJSON-8    50000    24531 ns/op    4096 B/op    12 allocs/op
#                    ^N       ^время/op      ^байт/op     ^аллокаций/op
```

---

## 10. Coverage

```bash
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out     # открыть в браузере
go tool cover -func=coverage.out     # построчно в терминале

# В CI: проверить минимальный порог
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%'
```

---

## Вопросы на собеседовании

### Q: Как ты структурируешь тесты в Go проекте?

**Ответ:** Table-driven tests как основной паттерн — описываем тест-кейсы в слайсе структур, запускаем через t.Run(). Разделяю unit и integration тесты флагом `testing.Short()`. Unit тесты — быстрые, без I/O, зависимости через интерфейсы. Интеграционные — через testcontainers (реальная БД в Docker). Всегда `b.ReportAllocs()` в benchmark'ах. Goleak для проверки goroutine утечек в тестах.

---

### Q: Чем мок отличается от стаба?

**Ответ:** Стаб (stub) — просто возвращает заготовленные данные, не проверяет как вызван. Мок (mock) — дополнительно проверяет что вызван с правильными аргументами, нужное количество раз. В Go: ручная реализация интерфейса = стаб. testify/mock или gomock = мок с `AssertExpectations()`. Для большинства тестов достаточно стаба — моки нужны когда важно что именно и сколько раз вызвали (например, что платёжный сервис вызван ровно один раз).

---

### Q: Как тестировать код с time.Now()?

**Ответ:** Инжектировать clock через интерфейс или функцию:
```go
type Clock interface { Now() time.Time }
type RealClock struct{}
func (RealClock) Now() time.Time { return time.Now() }

// В тесте:
type MockClock struct{ T time.Time }
func (m MockClock) Now() time.Time { return m.T }

svc := NewService(MockClock{T: time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC)})
```

---

## 9. Testcontainers — реальная БД в тестах

**Проблема моков для БД:** мок репозитория не проверяет реальные SQL-запросы, индексы, транзакции. Мокированные тесты проходят, а на проде падают из-за неправильного JOIN или нарушения constraint.

**Testcontainers** запускает реальный Docker-контейнер с PostgreSQL прямо в тесте.

```go
import (
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
)

func TestOrderRepo_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    ctx := context.Background()

    // Запускаем PostgreSQL в Docker
    pgContainer, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:16-alpine"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).WithStartupTimeout(30*time.Second),
        ),
    )
    require.NoError(t, err)
    defer pgContainer.Terminate(ctx)

    connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    require.NoError(t, err)

    db, err := sqlx.Connect("postgres", connStr)
    require.NoError(t, err)

    // Применяем миграции
    _, err = db.Exec(`
        CREATE TABLE orders (
            id SERIAL PRIMARY KEY,
            user_id INT NOT NULL,
            status VARCHAR(20) DEFAULT 'pending',
            amount DECIMAL(10,2),
            created_at TIMESTAMP DEFAULT NOW()
        )
    `)
    require.NoError(t, err)

    repo := NewOrderRepo(db)

    // Тест с реальной БД
    order, err := repo.Create(ctx, &Order{UserID: 1, Amount: 999.99})
    require.NoError(t, err)
    assert.NotZero(t, order.ID)

    fetched, err := repo.GetByID(ctx, order.ID)
    require.NoError(t, err)
    assert.Equal(t, order.ID, fetched.ID)
    assert.Equal(t, "pending", fetched.Status)
}
```

**Паттерн TestMain — один контейнер на весь пакет:**

```go
var testDB *sqlx.DB

func TestMain(m *testing.M) {
    ctx := context.Background()

    pgContainer, _ := postgres.RunContainer(ctx, ...)
    connStr, _ := pgContainer.ConnectionString(ctx, "sslmode=disable")
    testDB, _ = sqlx.Connect("postgres", connStr)

    runMigrations(testDB)

    code := m.Run() // запускаем все тесты

    pgContainer.Terminate(ctx)
    os.Exit(code)
}
```

**Когда интеграционные тесты важнее unit:**
- Сложные SQL-запросы с JOIN, оконными функциями, индексами
- Транзакционная логика (ACID, deadlock detection)
- Миграции схемы
- Специфичные фичи БД (JSONB, array operators, upsert)

---

## 10. Property-Based Testing

**Идея:** вместо конкретных примеров задаём **свойства** которые должны выполняться для любых входных данных. Фреймворк генерирует тысячи случайных кейсов.

Библиотека: `github.com/leanovate/gopter`

```go
import (
    "github.com/leanovate/gopter"
    "github.com/leanovate/gopter/gen"
    "github.com/leanovate/gopter/prop"
)

// Свойство: сортировка идемпотентна
// sort(sort(x)) == sort(x)
func TestSort_Idempotent(t *testing.T) {
    properties := gopter.NewProperties(nil)

    properties.Property("sort is idempotent", prop.ForAll(
        func(nums []int) bool {
            first := make([]int, len(nums))
            copy(first, nums)
            sort.Ints(first)

            second := make([]int, len(first))
            copy(second, first)
            sort.Ints(second)

            return reflect.DeepEqual(first, second)
        },
        gen.SliceOf(gen.Int()),
    ))

    properties.TestingRun(t)
}

// Свойство: encode→decode возвращает исходный объект
func TestJSON_RoundTrip(t *testing.T) {
    properties := gopter.NewProperties(nil)

    properties.Property("json roundtrip", prop.ForAll(
        func(id int64, amount float64, status string) bool {
            original := Order{ID: id, Amount: amount, Status: status}

            data, err := json.Marshal(original)
            if err != nil {
                return true // невалидный input — пропускаем
            }

            var decoded Order
            if err := json.Unmarshal(data, &decoded); err != nil {
                return false
            }

            return original == decoded
        },
        gen.Int64(),
        gen.Float64(),
        gen.AlphaString(),
    ))

    properties.TestingRun(t)
}
```

**Когда применять:**
- Математические инварианты (сортировка, поиск, хэширование)
- Encode/Decode, Marshal/Unmarshal пары
- Парсеры и валидаторы — граничные значения находит автоматически
- Бизнес-правила с широким диапазоном входных данных

**Shrinking:** при нахождении падающего кейса фреймворк автоматически упрощает входные данные до минимального воспроизводящего примера.

---

## 11. Нагрузочное тестирование с k6

**k6** — инструмент нагрузочного тестирования, скрипты на JavaScript.

```javascript
// load_test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
    stages: [
        { duration: '30s', target: 50 },   // ramp-up: 0 → 50 VU за 30 сек
        { duration: '1m',  target: 50 },   // держим 50 VU 1 минуту
        { duration: '30s', target: 200 },  // spike: 50 → 200 VU
        { duration: '1m',  target: 200 },  // держим пиковую нагрузку
        { duration: '30s', target: 0 },    // ramp-down
    ],
    thresholds: {
        http_req_duration: ['p(95)<500'],  // 95% запросов < 500ms
        http_req_failed:   ['rate<0.01'],  // ошибок < 1%
        errors:            ['rate<0.05'],
    },
};

export default function () {
    const res = http.post('http://localhost:8080/api/orders',
        JSON.stringify({ user_id: 1, product_id: 42, quantity: 1 }),
        { headers: { 'Content-Type': 'application/json' } },
    );

    const ok = check(res, {
        'status is 201':      (r) => r.status === 201,
        'response time < 1s': (r) => r.timings.duration < 1000,
    });

    errorRate.add(!ok);
    sleep(1);
}
```

```bash
# Запуск
k6 run load_test.js

# С выводом метрик в InfluxDB/Prometheus
k6 run --out influxdb=http://localhost:8086/k6 load_test.js
```

**Ключевые метрики в результатах k6:**

| Метрика | Что означает |
|---------|-------------|
| `http_req_duration` | Полное время запроса (latency) |
| `http_reqs` | Throughput — запросов в секунду |
| `http_req_failed` | Доля ошибок |
| `http_req_waiting` | Time To First Byte (TTFB) |
| `vus` | Активные виртуальные пользователи |

---

## 12. Throughput vs Latency под нагрузкой

**Throughput** — сколько запросов в секунду обрабатывает система (RPS).

**Latency** — время ответа на один запрос (мс). Смотрим на перцентили:
- `p50` — медиана, типичный пользователь
- `p95` — 95% пользователей получают ответ быстрее этого значения
- `p99` — хвостовая латентность, "самые медленные"

**Почему они конфликтуют под нагрузкой:**

```
                    Throughput
                        ↑
        saturation ─────┤ ←── throughput выходит на плато
        point      ─────┤
                        │         ↗ latency резко растёт
                        │      ↗
                        │   ↗
                        └──────────────────→ VU (нагрузка)
```

При низкой нагрузке: latency низкая, throughput растёт линейно.
При насыщении: очереди в сервисе заполняются → latency растёт экспоненциально, throughput выходит на плато.

**Закон Литтла:** `L = λ × W`
- L = среднее число запросов в системе
- λ = throughput (RPS)
- W = среднее время ответа (latency)

Если throughput не растёт, а latency растёт — система насыщена. Нужно масштабировать или оптимизировать.

**Что смотреть при анализе результатов k6:**

```
✓ p95 latency < 500ms      → норма
✗ p95 latency = 2000ms     → система перегружена
✓ error rate < 1%           → норма
✗ error rate = 15%          → connection pool исчерпан или OOM

Если throughput вырос, но p99 резко выросла:
→ горячий путь в коде, GC паузы, lock contention, медленные запросы в БД
```

**Типичные причины деградации:**
- Исчерпан пул соединений к БД — запросы встают в очередь
- GC давление — частые паузы на сборку мусора
- Lock contention — горутины ждут мьютекс
- Slow queries — один медленный запрос блокирует воркер

---

## Вопросы с интервью (дополнение)

**Q: Когда использовать Testcontainers вместо моков для БД?**

**Ответ:** Моки для БД — когда тестируем бизнес-логику сервисного слоя, БД не суть. Testcontainers — когда тестируем репозиторный слой: реальные SQL-запросы, миграции, транзакции, constraint'ы. Нас однажды обожгло: мокированные тесты проходили, но на проде падала вставка из-за UNIQUE violation которую мок не проверял. После перехода на Testcontainers такие баги ловятся в CI.

**Q: Что такое property-based testing и когда он лучше unit-тестов?**

**Ответ:** Property-based тест задаёт инвариант (свойство) которое должно выполняться для любых входных данных, фреймворк генерирует тысячи случайных кейсов. Лучше unit-тестов когда: тестируем парсеры, кодеки, математические функции — сложно вручную придумать все edge cases. При нахождении бага — автоматически находит минимальный воспроизводящий пример (shrinking).

**Q: p95 latency выросла с 200ms до 800ms под нагрузкой. Как диагностировать?**

**Ответ:** Смотрим по порядку: 1) `EXPLAIN ANALYZE` на медленные запросы в БД — возможно full scan без индекса, 2) размер пула соединений к БД — если запросы ждут соединения, 3) `go tool pprof` — CPU профиль под нагрузкой, горячие пути, 4) метрики GC (`go_gc_duration_seconds`) — паузы, 5) lock contention — `runtime.SetMutexProfileFraction`. Чаще всего виновата БД или исчерпанный connection pool.
