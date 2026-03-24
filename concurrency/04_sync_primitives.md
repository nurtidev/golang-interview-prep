# Примитивы синхронизации

---

## Лекция

### Зачем нужна синхронизация

Когда несколько горутин обращаются к одной переменной, возникает **состояние гонки (race condition)**. Например, инкремент `x++` — это три операции: прочитать x, прибавить 1, записать x обратно. Если две горутины делают это одновременно, они могут прочитать одно и то же значение и оба запишут x+1 вместо x+2.

Go предоставляет несколько инструментов синхронизации. Выбор зависит от задачи.

---

### sync.Mutex — эксклюзивная блокировка

`Mutex` (mutual exclusion) — самый базовый примитив. В любой момент только одна горутина может держать блокировку.

```go
var mu sync.Mutex

mu.Lock()
// критическая секция — только одна горутина здесь
x++
mu.Unlock()
```

**Правила работы с Mutex:**
- `defer mu.Unlock()` сразу после `mu.Lock()` — безопасный паттерн, гарантирует разблокировку даже при панике
- Нельзя вызывать `Lock()` дважды из одной горутины — **нет реентерабельности**, будет deadlock
- Нельзя копировать после первого использования (Mutex содержит внутреннее состояние)
- Передавать в функции нужно по указателю: `func f(mu *sync.Mutex)`

---

### sync.RWMutex — читатели и писатели

Оптимизация для сценариев "много читателей, мало писателей". Позволяет нескольким горутинам читать одновременно, но запись — только эксклюзивная.

```go
var rw sync.RWMutex

// Чтение: много горутин могут держать RLock одновременно
rw.RLock()
v := cache["key"]
rw.RUnlock()

// Запись: полная эксклюзивность, ждёт пока все RLock не освободятся
rw.Lock()
cache["key"] = newValue
rw.Unlock()
```

**Когда RWMutex лучше Mutex:** когда чтений значительно больше, чем записей (например, кэш конфигурации: читается тысячи раз в секунду, обновляется раз в минуту).

**Когда RWMutex хуже:** при частых записях — overhead на управление очередью читателей/писателей выше чем у Mutex.

---

### sync/atomic — атомарные операции

Для простых счётчиков и флагов мьютекс избыточен. `sync/atomic` предоставляет атомарные операции на уровне процессора — быстрее мьютекса, но работает только с примитивными типами.

```go
import "sync/atomic"

var counter int64

atomic.AddInt64(&counter, 1)           // атомарный инкремент
v := atomic.LoadInt64(&counter)        // атомарное чтение
atomic.StoreInt64(&counter, 0)         // атомарная запись

// CAS (Compare-And-Swap) — атомарная условная замена
// "если counter == 5, замени на 10"
swapped := atomic.CompareAndSwapInt64(&counter, 5, 10)
// если counter==5: counter=10, swapped=true
// если counter!=5: ничего не меняется, swapped=false
```

**CAS** — основа lock-free алгоритмов. Паттерн "попытайся, если не получилось — повтори" (optimistic concurrency).

**atomic.Value** — для атомарного хранения произвольных значений (read-copy-update паттерн):
```go
var config atomic.Value

// Обновление конфига без блокировки читателей
config.Store(newConfig)

// Чтение (никогда не блокируется)
cfg := config.Load().(*Config)
```

---

### sync.WaitGroup — ожидание группы горутин

Счётчик для ожидания завершения группы горутин.

```go
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)            // ПЕРЕД запуском горутины, не внутри
    go func(id int) {
        defer wg.Done() // -1 при завершении
        work(id)
    }(i)
}

wg.Wait() // блокируется пока счётчик не станет 0
```

**Частая ошибка — `wg.Add(1)` внутри горутины:**
```go
// ❌ Race condition: горутина может не запуститься до wg.Wait()
go func() {
    wg.Add(1) // может выполниться после wg.Wait()!
    defer wg.Done()
}()
wg.Wait()
```

Между запуском горутины (`go func()`) и её реальным началом выполнения проходит время. Если `wg.Add(1)` внутри, то `wg.Wait()` может отработать до того, как Add вызовется.

**Go 1.25+: метод `wg.Go()`** — удобный shorthand, совмещает `Add(1)` + `go func()` + `defer Done()`:

```go
var wg sync.WaitGroup

// Старый способ:
wg.Add(1)
go func() {
    defer wg.Done()
    doWork()
}()

// Go 1.25+: эквивалентно, но компактнее и без ошибки Add-внутри:
wg.Go(func() {
    doWork()
})

wg.Wait()
```

---

### sync.Once — инициализация один раз

Гарантирует, что функция выполнится ровно один раз, даже при конкурентных вызовах. Все остальные вызовы `Do` блокируются, пока первый не завершится.

```go
var once sync.Once
var instance *Database

func getDB() *Database {
    once.Do(func() {
        instance = connectToDatabase() // вызовется только один раз
    })
    return instance
}
```

**Применение:** singleton-паттерн, ленивая инициализация дорогих ресурсов (соединение с БД, конфигурация).

**Важно:** если функция в `Do` паникует — `Once` считается выполненным. Повторный вызов `Do` не выполнит функцию ещё раз.

---

### sync.Cond — условная переменная

`Cond` для случаев, когда нужно ждать наступления условия. Более гибкий инструмент чем каналы для fan-out уведомлений.

```go
var mu sync.Mutex
cond := sync.NewCond(&mu)
ready := false

// Waiter: горутина, ожидающая условия
go func() {
    mu.Lock()
    for !ready {          // проверяем в цикле (spurious wakeups)
        cond.Wait()       // атомарно: снимает Lock и ждёт сигнала
                          // при пробуждении: снова берёт Lock
    }
    // работаем с данными под блокировкой
    mu.Unlock()
}()

// Notifier: горутина, сигнализирующая об изменении
mu.Lock()
ready = true
mu.Unlock()
cond.Signal()    // разбудить одну горутину
// или
cond.Broadcast() // разбудить все ожидающие горутины
```

`Wait()` всегда вызывается в цикле `for !condition` — защита от **spurious wakeups** (ложных пробуждений, которые могут происходить в ОС).

**Когда Cond лучше канала:** когда нужен broadcast (разбудить всех), а состояние может изменяться много раз. Каналы для этого менее удобны.

---

### Deadlock — взаимная блокировка

Deadlock — ситуация, когда горутины ждут друг друга и никто не может продолжить.

**Классический deadlock (cyclic dependency):**
```
G1 держит mu1, ждёт mu2
G2 держит mu2, ждёт mu1
→ оба ждут вечно
```

**Признаки:** программа "зависает", CPU не загружен, горутины не прогрессируют.

**Go детектирует полный deadlock** (когда заблокированы ВСЕ горутины):
```
fatal error: all goroutines are asleep - deadlock!
```

Но если есть хотя бы одна незаблокированная горутина — Go не детектирует частичный deadlock. Используйте `pprof` для диагностики.

**Предотвращение:** всегда захватывать мьютексы в одном и том же порядке во всех горутинах.

---

### Data Race и Livelock

**Data Race** — два конкурентных доступа к переменной, хотя бы один из них запись, без синхронизации. Результат непредсказуем (UB).

```go
var x int
go func() { x++ }() // ❌ data race
go func() { x++ }()
// x может быть 1, 2, или любым другим значением
```

**Детектор:** `go run -race ./...` или `go test -race ./...`. Встроен в Go — использовать обязательно.

**Livelock** — горутины активны (не заблокированы), но не прогрессируют. Например, два человека в коридоре постоянно уступают дорогу друг другу в одну сторону.

---

### False Sharing — скрытая проблема производительности

Когда две переменные находятся в одной **cache line** (64 байта на x86), конкурентное изменение любой из них инвалидирует кэш для обеих — даже если горутины работают с разными переменными.

```go
// ❌ c1 и c2 могут быть в одной cache line (16 байт < 64 байта)
type Counters struct {
    c1 int64 // offset 0
    c2 int64 // offset 8
}

// ✅ Padding гарантирует, что каждый счётчик в своей cache line
type Counters struct {
    c1  int64
    _   [56]byte // 56 + 8 = 64 байта = размер cache line
    c2  int64
}
```

Это важно только для **горячих путей** с высокой конкурентностью. Не оптимизируйте преждевременно.

---

### Как выбрать инструмент синхронизации

| Задача | Инструмент |
|---|---|
| Защита критической секции | `sync.Mutex` |
| Много читателей, мало писателей | `sync.RWMutex` |
| Счётчики, флаги (простые типы) | `sync/atomic` |
| Ожидание группы горутин | `sync.WaitGroup` |
| Запуск горутины с учётом WaitGroup | `wg.Go(func(){...})` (Go 1.25+) |
| Инициализация один раз | `sync.Once` |
| Ожидание условия с broadcast | `sync.Cond` |
| Передача данных между горутинами | `chan` |
| Ограничение конкурентности | буферизованный `chan` (семафор) |

---

## Примеры

### Пример 1: Безопасный счётчик с Mutex

```go
package main

import (
    "fmt"
    "sync"
)

type SafeCounter struct {
    mu    sync.Mutex
    value int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *SafeCounter) Get() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}

func main() {
    counter := &SafeCounter{}
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Increment()
        }()
    }

    wg.Wait()
    fmt.Println("итог:", counter.Get()) // всегда 1000
}
```

### Пример 2: RWMutex для кэша конфигурации

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Config struct {
    mu       sync.RWMutex
    settings map[string]string
}

func NewConfig() *Config {
    return &Config{
        settings: map[string]string{"timeout": "30s", "retries": "3"},
    }
}

// Много горутин могут читать одновременно
func (c *Config) Get(key string) string {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.settings[key]
}

// Запись — эксклюзивная
func (c *Config) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.settings[key] = value
}

func main() {
    cfg := NewConfig()

    // 10 горутин читают конфиг
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            fmt.Printf("goroutine %d: timeout=%s\n", id, cfg.Get("timeout"))
        }(i)
    }

    // Обновляем конфиг
    time.Sleep(time.Millisecond)
    cfg.Set("timeout", "60s")

    wg.Wait()
}
```

### Пример 3: Атомарный счётчик vs Mutex-счётчик

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

func main() {
    const N = 100_000
    var wg sync.WaitGroup

    // Атомарный счётчик (быстрее)
    var atomicCounter int64
    for i := 0; i < N; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&atomicCounter, 1)
        }()
    }
    wg.Wait()
    fmt.Println("atomic:", atomicCounter) // 100000

    // Mutex счётчик
    var mu sync.Mutex
    var mutexCounter int
    for i := 0; i < N; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            mutexCounter++
            mu.Unlock()
        }()
    }
    wg.Wait()
    fmt.Println("mutex:", mutexCounter) // 100000
}
```

### Пример 4: sync.Once — ленивая инициализация

```go
package main

import (
    "fmt"
    "sync"
)

type Database struct {
    dsn string
}

func (db *Database) Query(q string) string {
    return fmt.Sprintf("result of '%s' from %s", q, db.dsn)
}

var (
    once sync.Once
    db   *Database
)

func getDB() *Database {
    once.Do(func() {
        fmt.Println("инициализируем соединение с БД...")
        db = &Database{dsn: "postgres://localhost/mydb"}
    })
    return db
}

func main() {
    var wg sync.WaitGroup

    // 5 горутин одновременно запрашивают DB
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            result := getDB().Query(fmt.Sprintf("SELECT * FROM users LIMIT %d", id))
            fmt.Println(result)
        }(i)
    }

    wg.Wait()
    // "инициализируем соединение с БД..." выведется ОДИН раз
}
```

### Пример 5: Deadlock и как его избежать

```go
package main

import (
    "fmt"
    "sync"
)

// ❌ DEADLOCK: горутины захватывают мьютексы в разном порядке
func deadlockExample() {
    var mu1, mu2 sync.Mutex
    done := make(chan struct{})

    go func() {
        mu1.Lock()
        // time.Sleep(1ms) — тут второй успевает захватить mu2
        mu2.Lock() // ждёт горутину 2, которая ждёт нас
        mu2.Unlock()
        mu1.Unlock()
        close(done)
    }()

    mu2.Lock()
    mu1.Lock() // ждёт горутину выше — DEADLOCK
    mu1.Unlock()
    mu2.Unlock()

    <-done
}

// ✅ Правильно: всегда захватываем мьютексы в одном порядке (mu1, затем mu2)
func noDeadlock() {
    var mu1, mu2 sync.Mutex
    var wg sync.WaitGroup
    wg.Add(2)

    go func() {
        defer wg.Done()
        mu1.Lock()
        mu2.Lock()
        fmt.Println("горутина 1: работаю")
        mu2.Unlock()
        mu1.Unlock()
    }()

    go func() {
        defer wg.Done()
        mu1.Lock() // тот же порядок: сначала mu1, потом mu2
        mu2.Lock()
        fmt.Println("горутина 2: работаю")
        mu2.Unlock()
        mu1.Unlock()
    }()

    wg.Wait()
}

func main() {
    noDeadlock()
}
```

---

## Вопросы с собеседования

**Q: Чем Mutex отличается от RWMutex? Когда что использовать?**

`Mutex` — полная эксклюзивность: только одна горутина в критической секции.

`RWMutex` — несколько горутин могут одновременно держать `RLock` (чтение), но `Lock` (запись) — эксклюзивен.

Использовать `RWMutex` когда чтений значительно больше, чем записей (кэш, конфигурация, registry). Если записи часты — обычный `Mutex` может быть быстрее из-за меньшего overhead.

---

**Q: Что такое CAS и зачем он нужен?**

Compare-And-Swap — атомарная операция: "если текущее значение равно old, замени на new". Возвращает `true` если замена произошла.

Основа lock-free алгоритмов. Паттерн использования:
```go
for {
    old := atomic.LoadInt64(&x)
    new := compute(old)
    if atomic.CompareAndSwapInt64(&x, old, new) {
        break // успешно обновили
    }
    // кто-то успел изменить x раньше нас — повторяем
}
```

---

**Q: Можно ли копировать sync.Mutex?**

Нет. `Mutex` содержит внутреннее состояние (флаг заблокированности). Копирование заблокированного мьютекса скопирует заблокированное состояние, что приведёт к deadlock или некорректному поведению. `go vet` предупреждает об этом. Передавать нужно по указателю.

---

**Q: Почему `wg.Add(1)` нужно вызывать ДО запуска горутины?**

Между `go func()` и реальным началом выполнения горутины есть задержка. Если `wg.Add(1)` внутри горутины, то `wg.Wait()` может выполниться до того, как `Add` будет вызван. Счётчик останется 0, `Wait` вернётся, а горутина ещё не завершила работу — race condition.

---

**Q: В чём разница между `cond.Signal()` и `cond.Broadcast()`?**

`Signal()` — будит одну произвольную горутину из очереди ожидающих.
`Broadcast()` — будит все ожидающие горутины.

Использовать `Broadcast()` когда несколько горутин ожидают одного события (например, завершение загрузки данных). `Signal()` — когда одно событие = одна задача (например, в пуле воркеров появилась задача).

---

**Q: Что такое data race и как его обнаружить?**

Data race — конкурентный доступ к переменной, хотя бы один доступ — запись, без синхронизации. Результат undefined behavior: значение может быть любым, программа может падать в случайных местах.

Обнаружить: `go test -race ./...` или `go run -race main.go`. Race detector инструментирует код во время компиляции и обнаруживает нарушения в runtime. Overhead ~5-10x по времени и ~2-20x по памяти — не для продакшна, но обязательно в CI.

---

**Q: Что такое deadlock и как его избежать?**

Deadlock — взаимная блокировка: горутина A ждёт ресурс у B, B ждёт ресурс у A. Никто не может продолжить.

Как избежать:
1. **Порядок захвата:** всегда захватывать мьютексы в одном и том же порядке
2. **Таймауты:** использовать `context.WithTimeout` вместо бесконечного ожидания
3. **Минимизировать критические секции:** не держать блокировку дольше необходимого
4. **Каналы вместо мьютексов** для коммуникации между горутинами

---

**Q: Чем sync.Once отличается от `init()` функции?**

`init()` выполняется при инициализации пакета — всегда, даже если объект никогда не используется. `sync.Once` выполняется **лениво** — только при первом обращении. Это полезно для дорогих инициализаций (открытие БД-соединения), которые нужны не всегда.

---

**Q: Что такое false sharing и когда это важно?**

Два поля структуры могут попасть в одну cache line (64 байта). Если две горутины на разных CPU ядрах изменяют разные поля, но они в одной cache line — каждое изменение инвалидирует кэш другого ядра. Это приводит к "cache ping-pong" и снижает производительность.

Решение — padding до границы cache line. Актуально только для очень горячих путей с высокой конкурентностью. Измеряйте перед оптимизацией.
