# Паттерны конкурентности

---

## Лекция

### Зачем нужны паттерны конкурентности

Go даёт мощные примитивы (горутины, каналы), но сами по себе они не решают архитектурные проблемы. Паттерны конкурентности — это проверенные рецепты для типичных задач: обработка потока данных несколькими воркерами, объединение результатов из разных источников, ограничение нагрузки. Знание этих паттернов — то, что отличает Go-разработчика среднего уровня от сильного.

---

### Pipeline (Конвейер)

**Идея:** разбить обработку данных на независимые стадии, каждая из которых работает в своей горутине. Стадии соединяются каналами.

```
generate → square → filter → вывод
   G1  ──chan──→ G2  ──chan──→ G3  ──chan──→ main
```

**Когда применять:** когда обработку данных можно разбить на независимые шаги (ETL, трансформации, фильтрация). Каждая стадия начинает работу как только получает данные — не ждёт завершения предыдущей стадии целиком.

**Ключевые принципы:**
- Каждая стадия принимает `<-chan T` и возвращает `<-chan T`
- `defer close(out)` в начале горутины — стадия закрывает свой выходной канал когда заканчивается входной
- Это создаёт автоматическую цепочку завершения

---

### Fan-Out (Разветвление)

**Идея:** распределить задачи из одного канала на несколько горутин-обработчиков для параллельной обработки.

```
input ──→ [worker 1]
      ──→ [worker 2]  → несколько выходных каналов
      ──→ [worker 3]
```

**Когда применять:** одна очередь задач, нужно обработать параллельно (HTTP запросы, CPU-интенсивные задачи).

**Важно:** при Fan-Out все воркеры читают из **одного** входного канала. Go гарантирует, что каждое значение прочитает только одна горутина — встроенная балансировка нагрузки.

---

### Fan-In (Слияние, Merge)

**Идея:** объединить несколько каналов в один.

```
[source 1] ──→
[source 2] ──→  merged output
[source 3] ──→
```

**Когда применять:** несколько независимых источников данных, нужно обработать их в одном месте.

**Ключевые принципы:**
- На каждый входной канал — своя горутина, которая перекладывает данные в выходной канал
- `sync.WaitGroup` отслеживает, когда все источники иссякли
- `close(out)` только после `wg.Wait()` — не раньше

---

### Worker Pool (Пул воркеров)

**Идея:** фиксированное количество горутин-воркеров читают задачи из общего канала.

```
jobs ──→ [worker 1]
     ──→ [worker 2]  → results
     ──→ [worker 3]
```

**Зачем ограничивать количество горутин?** Неограниченный Fan-Out может создать миллион горутин и перегрузить систему. Worker Pool даёт контроль над конкурентностью.

**Vs Fan-Out:** в Fan-Out на каждую задачу может создаваться новая горутина. В Worker Pool количество горутин фиксировано, они переиспользуются.

**Типичная структура:**
1. N горутин запускаются и читают из канала `jobs`
2. Каждый воркер обрабатывает задачу и пишет в `results`
3. `wg.Wait()` ждёт завершения всех воркеров
4. `close(results)` только после `wg.Wait()`

---

### Семафор через буферизованный канал

**Идея:** ограничить количество одновременно работающих горутин через буферизованный канал.

```go
sem := make(chan struct{}, N) // максимум N одновременных

sem <- struct{}{}   // захватить слот (блокируется если N уже занято)
// ... работа ...
<-sem               // освободить слот
```

Это простейшая реализация семафора. Размер буфера = максимальное количество параллельных операций.

**Применение:** ограничить количество параллельных HTTP-запросов, обращений к БД, файловых операций.

---

### Rate Limiter (Тротлер) через time.Ticker

**Идея:** выполнять операции не чаще N раз в секунду.

```go
ticker := time.NewTicker(time.Second / time.Duration(rps))
// каждый тик = одно разрешение на выполнение
```

`time.Ticker` создаёт канал, в который рантайм периодически кладёт значения. Ожидание на `ticker.C` создаёт естественное ограничение скорости.

---

### Graceful Shutdown — корректное завершение

Паттерн для завершения всех воркеров без потери данных:

1. Получаем сигнал остановки (от ОС или через `context.WithCancel`)
2. Перестаём принимать новые задачи (закрываем `jobs` канал или отменяем контекст)
3. Ждём завершения текущих задач (`wg.Wait()`)
4. Освобождаем ресурсы (БД-соединения, файлы)

Ключевой момент: **close(jobs) != kill workers**. Закрытие jobs-канала сигнализирует воркерам "новых задач не будет", но они дочитают всё что уже в канале.

---

## Примеры

### Пример 1: Pipeline

```go
package main

import "fmt"

// Стадия 1: генерируем числа
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

// Стадия 2: возводим в квадрат
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

// Стадия 3: фильтруем чётные
func filterEven(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if n%2 == 0 {
                out <- n
            }
        }
    }()
    return out
}

func main() {
    // Строим конвейер: generate → square → filterEven
    nums := generate(1, 2, 3, 4, 5)
    squares := square(nums)
    evens := filterEven(squares)

    for v := range evens {
        fmt.Println(v) // 4 (2²), 16 (4²)
    }
}
```

### Пример 2: Fan-In (Merge каналов)

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func producer(name string, vals ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, v := range vals {
            time.Sleep(50 * time.Millisecond)
            out <- v
        }
    }()
    return out
}

// Объединяем несколько каналов в один
func merge(inputs ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    forward := func(ch <-chan int) {
        defer wg.Done()
        for v := range ch {
            out <- v
        }
    }

    wg.Add(len(inputs))
    for _, ch := range inputs {
        go forward(ch)
    }

    // Закрываем выходной канал когда все источники завершились
    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

func main() {
    ch1 := producer("A", 1, 2, 3)
    ch2 := producer("B", 10, 20, 30)
    ch3 := producer("C", 100, 200)

    for v := range merge(ch1, ch2, ch3) {
        fmt.Println(v) // числа из всех трёх каналов, порядок непредсказуем
    }
}
```

### Пример 3: Worker Pool

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

func workerPool(ctx context.Context, jobs <-chan int, numWorkers int) <-chan int {
    results := make(chan int, numWorkers)
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    fmt.Printf("worker %d: остановлен по context\n", workerID)
                    return
                case job, ok := <-jobs:
                    if !ok {
                        fmt.Printf("worker %d: канал jobs закрыт, выхожу\n", workerID)
                        return
                    }
                    // Симулируем работу
                    time.Sleep(10 * time.Millisecond)
                    result := job * job
                    select {
                    case results <- result:
                    case <-ctx.Done():
                        return
                    }
                }
            }
        }(i)
    }

    // Закрываем results только после завершения всех воркеров
    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    jobs := make(chan int, 20)
    go func() {
        defer close(jobs)
        for i := 1; i <= 10; i++ {
            jobs <- i
        }
    }()

    for result := range workerPool(ctx, jobs, 3) {
        fmt.Println("результат:", result)
    }
}
```

### Пример 4: Семафор — ограничение параллельных HTTP-запросов

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func fetchURL(url string) string {
    time.Sleep(100 * time.Millisecond) // симуляция HTTP-запроса
    return fmt.Sprintf("response from %s", url)
}

func fetchAll(urls []string, maxConcurrent int) []string {
    sem := make(chan struct{}, maxConcurrent) // семафор
    results := make([]string, len(urls))
    var wg sync.WaitGroup

    for i, url := range urls {
        wg.Add(1)
        go func(i int, url string) {
            defer wg.Done()
            sem <- struct{}{}        // захватить слот
            defer func() { <-sem }() // освободить слот при завершении
            results[i] = fetchURL(url)
        }(i, url)
    }

    wg.Wait()
    return results
}

func main() {
    urls := []string{
        "https://example.com/1",
        "https://example.com/2",
        "https://example.com/3",
        "https://example.com/4",
        "https://example.com/5",
    }

    // Максимум 2 параллельных запроса
    results := fetchAll(urls, 2)
    for _, r := range results {
        fmt.Println(r)
    }
}
```

### Пример 5: Rate Limiter

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// Выполняет fn не чаще rps раз в секунду
func rateLimited(ctx context.Context, rps int, tasks []string) {
    ticker := time.NewTicker(time.Second / time.Duration(rps))
    defer ticker.Stop()

    i := 0
    for i < len(tasks) {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            fmt.Printf("[%s] обрабатываю: %s\n", time.Now().Format("15:04:05.000"), tasks[i])
            i++
        }
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    tasks := []string{"task1", "task2", "task3", "task4", "task5", "task6"}
    rateLimited(ctx, 2, tasks) // 2 задачи в секунду
}
```

---

## Вопросы с собеседования

**Q: Напишите функцию merge, которая объединяет N каналов в один.**

```go
func merge(inputs ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    wg.Add(len(inputs))
    for _, ch := range inputs {
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

Ключевые моменты: одна горутина на каждый входной канал, WaitGroup ждёт всех, close только после wg.Wait().

---

**Q: Напишите worker pool на N воркеров.**

Ключевые элементы реализации:
1. N горутин запускаются и читают из одного канала `jobs`
2. `wg.Add(1)` перед каждым `go func()`
3. `defer wg.Done()` в каждом воркере
4. `close(results)` в отдельной горутине после `wg.Wait()`
5. Обработка `ctx.Done()` для graceful shutdown

---

**Q: Как ограничить количество одновременных горутин?**

Через семафор на основе буферизованного канала:
```go
sem := make(chan struct{}, maxConcurrent)
// перед работой:
sem <- struct{}{}
defer func() { <-sem }()
```

---

**Q: В чём разница между Fan-Out и Worker Pool?**

**Fan-Out:** на каждую задачу/источник может создаваться новая горутина. Количество горутин растёт вместе с количеством задач.

**Worker Pool:** фиксированное количество горутин. Они переиспользуются. Контроль над потреблением ресурсов. Подходит для долгоживущих сервисов.

---

**Q: Почему нельзя делать `close(results)` сразу после запуска воркеров в worker pool?**

Потому что воркеры ещё не завершили работу. Если закрыть `results` раньше — запись воркеров в закрытый канал вызовет панику. Нужно дождаться `wg.Wait()` в отдельной горутине и только потом делать `close(results)`.

---

**Q: Как gracefully завершить worker pool?**

Два способа:
1. **Закрыть канал jobs** — воркеры дочитают оставшиеся задачи и завершатся при `ok=false`
2. **Отменить context** — воркеры проверяют `ctx.Done()` и выходят немедленно

Первый способ — мягкое завершение (обрабатываем всё что уже в очереди). Второй — жёсткое (бросаем незавершённые задачи). Часто используют оба: сначала закрывают jobs, если не завершились за таймаут — отменяют context.

---

**Q: Как работает Pipeline паттерн и в чём его преимущество?**

Pipeline — цепочка стадий, соединённых каналами. Каждая стадия — горутина, читающая из входного канала и пишущая в выходной.

Преимущества:
- Стадии работают параллельно: пока стадия 2 обрабатывает элемент 1, стадия 1 уже готовит элемент 2
- Легко добавлять/убирать стадии
- Каждая стадия проста и тестируема отдельно
- При `close` первой стадии завершение автоматически распространяется по цепочке

---

**Q: Почему нельзя читать из канала results до завершения всех воркеров в паттерне "параллельный fetch"?**

Можно и нужно — это одно из преимуществ worker pool. Consumer читает из `results` по мере появления результатов. `close(results)` сигнализирует что результатов больше не будет, и `for range results` завершится. Не надо ждать всех воркеров перед первым чтением.
