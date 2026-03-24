# Map

---

## Лекция

### Что такое map и зачем он нужен

Map (хеш-таблица, словарь, ассоциативный массив) — структура данных, позволяющая хранить пары ключ-значение и получать значение по ключу за O(1) в среднем. Это одна из самых часто используемых структур в программировании: кэш, реестр, подсчёт частот, индексирование данных.

В Go `map` встроен в язык — не нужно подключать библиотеки. Но у него есть важные особенности и ограничения, которые нужно знать.

---

### Внутреннее устройство

> **Версионное примечание:** Go 1.24 сменил реализацию map с bucket-based на **Swiss Tables**. Описание ниже актуально для Go 1.24+, затем — оригинальная реализация (Go ≤ 1.23) для понимания истории и вопросов на собеседовании.

---

#### Go 1.24+: Swiss Tables

Go 1.24 заменил внутреннюю реализацию map на **Swiss Tables** — алгоритм от Abseil/Google. Ключевые отличия:

```
Swiss Tables используют flat open-addressing (не цепочки):
- Metadata: компактный массив однобайтовых тегов (h2 = старшие 7 бит хеша)
- Slots: непрерывный массив пар ключ-значение
- Группы: 8 слотов + 8 байт метаданных — идеально для SIMD-операций

Поиск: сравниваем h2 сразу для 8 слотов параллельно через битовые операции
```

**Преимущества перед bmap:**
- Лучшая локальность кэша (метаданные и данные рядом в памяти)
- Поиск через SIMD-подобные операции (8 элементов за раз)
- Меньше overhead на overhead-бакеты → 2–3% CPU экономии в среднем
- Можно отключить: `GOEXPERIMENT=noswissmap`

---

#### Go ≤ 1.23: Bucket-based (hmap/bmap)

Go реализует map как **хеш-таблицу с бакетами (buckets)**. Структура `hmap` в рантайме:

```
hmap
├── count        int           // текущее количество элементов
├── B            uint8         // log₂ числа бакетов: 2^B бакетов всего
├── hash0        uint32        // seed для рандомизации хеша
├── buckets      *[]bmap       // массив бакетов
└── oldbuckets   *[]bmap       // старые бакеты во время эвакуации (rehash)

bmap (один бакет)
├── tophash      [8]uint8      // старшие 8 бит хеша каждого ключа (для быстрого поиска)
├── keys         [8]K          // ключи (8 штук)
└── values       [8]V          // значения (8 штук)
└── overflow     *bmap         // цепочка переполнения (если > 8 элементов в бакете)
```

**Как работает поиск ключа:**
1. Вычисляем хеш ключа
2. Младшие `B` бит хеша → номер бакета
3. Старшие 8 бит (tophash) → быстро сравниваем с `tophash[i]` в бакете
4. При совпадении tophash — полное сравнение ключа
5. Если в бакете > 8 элементов — идём по цепочке overflow-бакетов

```
key "alice"
    │
    ▼ hash(key) = 0b...10110101...11010
                         └──B бит──┘         └─ tophash ─┘
                         номер бакета=26      0b11010 = 26
```

**Зачем tophash?** Чтобы не делать полное сравнение строк/структур при каждом элементе в бакете. Сравнение 1 байта tophash гораздо быстрее.

---

### Rehash (эвакуация)

Когда map растёт и `count / buckets > 6.5` (load factor), Go создаёт новый массив бакетов вдвое больше и начинает **постепенную эвакуацию**: не сразу переносит всё (это бы заблокировало программу), а при каждой операции переносит 1–2 бакета. Во время эвакуации `oldbuckets` хранит старые бакеты.

Это означает: **одна запись в map может занять непредсказуемое время** — именно поэтому map не подходит для систем с жёсткими требованиями к latency без предварительного прогрева.

---

### Сложность операций

| Операция | Средняя | Худший случай |
|---|---|---|
| Чтение | O(1) | O(n) при плохом хеше |
| Запись | O(1) | O(n) при rehash |
| Удаление | O(1) | O(n) |

---

### Базовые операции

```go
// Создание
m := make(map[string]int)
m := make(map[string]int, 100) // с hint — меньше rehash-ов при росте

// Запись
m["key"] = 42

// Чтение с проверкой наличия (comma ok idiom)
v, ok := m["key"]
if !ok {
    // ключа нет, v == zero value для типа
}

// Чтение без проверки — возвращает zero value если ключа нет (не паникует)
v := m["missing"] // v == 0 для int

// Удаление
delete(m, "key") // безопасно даже если ключа нет

// Итерация — порядок ВСЕГДА непредсказуем
for k, v := range m {
    fmt.Println(k, v)
}
```

---

### Рандомизация порядка итерации

Начиная с Go 1, порядок итерации по map **намеренно рандомизирован**. При каждом запуске `range` выбирается случайный начальный бакет. Это сделано специально — чтобы разработчики не полагались на порядок (который технически может меняться между версиями Go и компиляциями).

Если нужен порядок — собери ключи в слайс и сортируй:
```go
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
    fmt.Println(k, m[k])
}
```

---

### Конкурентный доступ к map — DATA RACE

Map в Go **не потокобезопасна**. Конкурентная запись (или чтение+запись) — это data race, который приводит к падению с сообщением:

```
fatal error: concurrent map writes
```

```go
// ❌ DATA RACE — программа упадёт
var m = make(map[string]int)
go func() { m["a"] = 1 }()
go func() { m["b"] = 2 }()
```

**Решение 1: sync.RWMutex** — универсально, подходит для любой нагрузки

```go
var mu sync.RWMutex
var m = make(map[string]int)

// Запись
mu.Lock()
m["key"] = value
mu.Unlock()

// Чтение
mu.RLock()
v := m["key"]
mu.RUnlock()
```

**Решение 2: sync.Map** — специализированная конкурентная map

```go
var sm sync.Map

sm.Store("key", 42)             // запись
v, ok := sm.Load("key")         // чтение
sm.Delete("key")                // удаление
sm.LoadOrStore("key", "default") // атомарный get-or-set

sm.Range(func(k, v any) bool {
    fmt.Println(k, v)
    return true // false = остановить итерацию
})
```

---

### sync.Map — когда использовать, а когда нет

`sync.Map` использует два словаря внутри:
- `read` — атомарный указатель на read-only map, без блокировок
- `dirty` — обычная map с мьютексом, для новых ключей

При чтении: сначала проверяем `read` (без блокировки). При промахе — обращаемся к `dirty` (с блокировкой). При достаточном количестве промахов `dirty` "продвигается" в `read`.

**Когда sync.Map выигрывает:**
- **Read-heavy:** большинство операций — чтения существующих ключей
- **Disjoint keys:** каждая горутина работает со своим набором ключей

**Когда sync.Map проигрывает обычной map + mutex:**
- Частые записи новых ключей — постоянный mutex overhead
- Маленькая map с равномерными чтениями/записями

**Правило:** по умолчанию используй `map + sync.RWMutex`. Переходи на `sync.Map` только если профилировщик показывает bottleneck.

---

### Ключи map — comparability

Ключом может быть любой **comparable** тип — тот, который можно сравнить через `==`:

```go
map[string]int          // ✅ string
map[int]string          // ✅ int, float, bool, pointer
map[[3]int]string       // ✅ массив из comparable элементов
map[struct{ X, Y int }]string  // ✅ структура из comparable полей

map[[]int]string        // ❌ compile error: слайс не comparable
map[map[string]int]string  // ❌ compile error: map не comparable
map[func()]string       // ❌ compile error: func не comparable
```

**Интерфейс как ключ:** компилируется, но паникует в runtime если конкретный тип не comparable:
```go
var m = map[interface{}]int{}
m[[]int{1, 2}] = 1 // panic: runtime error: hash of unhashable type []int
```

---

### Set на основе map

В Go нет встроенного типа Set. Стандартный способ — `map[T]struct{}`:

```go
set := make(map[string]struct{})

// Добавить
set["apple"] = struct{}{}

// Проверить наличие
_, exists := set["apple"] // exists = true

// Удалить
delete(set, "apple")

// Размер
fmt.Println(len(set))
```

`struct{}` — zero-size тип, не занимает памяти (в отличие от `map[string]bool`, где bool занимает 1 байт на элемент).

---

### Утечка памяти через map

В отличие от слайса, map не уменьшается при удалении элементов. Бакеты остаются выделенными.

```go
m := make(map[int][128]byte)

// Наполняем
for i := 0; i < 1_000_000; i++ {
    m[i] = [128]byte{}
}
// Используется ~128 МБ

// Очищаем
for i := 0; i < 1_000_000; i++ {
    delete(m, i)
}
// m пустая, но ~128 МБ ВСЁРАВНО удерживаются!
// Бакеты не освобождены.
```

**Решение:** пересоздать map — `m = make(map[int][128]byte)`. Или хранить указатели `map[int]*[128]byte` — тогда GC освободит сами значения.

---

### Изменение map во время итерации

Go допускает изменение map во время `range`, но с оговорками:
- Удалённые ключи: если ещё не посещены — могут не появиться. Если посещены — уже обработаны.
- Добавленные ключи: могут встретиться или нет — неопределённо.
- Программа не паникует, но результат нестабилен.

---

## Примеры

### Пример 1: Подсчёт частот слов

```go
package main

import (
    "fmt"
    "strings"
)

func wordCount(text string) map[string]int {
    counts := make(map[string]int)
    for _, word := range strings.Fields(text) {
        counts[word]++ // если ключа нет — zero value (0) + 1 = 1
    }
    return counts
}

func main() {
    text := "go is great go is fast go"
    counts := wordCount(text)

    fmt.Println("go:", counts["go"])     // 3
    fmt.Println("is:", counts["is"])     // 2
    fmt.Println("great:", counts["great"]) // 1
}
```

### Пример 2: Безопасный кэш с RWMutex

```go
package main

import (
    "fmt"
    "sync"
)

type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func NewCache() *Cache {
    return &Cache{data: make(map[string]string)}
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()         // много горутин могут читать одновременно
    defer c.mu.RUnlock()
    v, ok := c.data[key]
    return v, ok
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()          // запись эксклюзивна
    defer c.mu.Unlock()
    c.data[key] = value
}

func main() {
    cache := NewCache()
    var wg sync.WaitGroup

    // 5 горутин пишут
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            cache.Set(fmt.Sprintf("key%d", id), fmt.Sprintf("value%d", id))
        }(i)
    }

    wg.Wait()

    // Читаем результаты
    for i := 0; i < 5; i++ {
        v, ok := cache.Get(fmt.Sprintf("key%d", i))
        fmt.Printf("key%d: %s (found: %v)\n", i, v, ok)
    }
}
```

### Пример 3: sync.Map для конкурентного кэша

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var sm sync.Map
    var wg sync.WaitGroup

    // Записываем конкурентно
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            sm.Store(id, id*id)
        }(i)
    }
    wg.Wait()

    // Читаем
    sm.Range(func(k, v any) bool {
        fmt.Printf("%d² = %d\n", k, v)
        return true
    })

    // LoadOrStore — атомарный get-or-set
    actual, loaded := sm.LoadOrStore(10, 100)
    fmt.Printf("key=10: value=%v, existed=%v\n", actual, loaded)
}
```

### Пример 4: Граф на основе map

```go
package main

import "fmt"

type Graph map[string][]string

func (g Graph) AddEdge(from, to string) {
    g[from] = append(g[from], to)
}

func (g Graph) Neighbors(node string) []string {
    return g[node] // nil если узла нет — безопасно для range
}

func (g Graph) BFS(start string) []string {
    visited := make(map[string]bool)
    queue := []string{start}
    var result []string

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]

        if visited[node] { // map возвращает false для отсутствующего ключа
            continue
        }
        visited[node] = true
        result = append(result, node)

        for _, neighbor := range g.Neighbors(node) {
            if !visited[neighbor] {
                queue = append(queue, neighbor)
            }
        }
    }
    return result
}

func main() {
    g := make(Graph)
    g.AddEdge("A", "B")
    g.AddEdge("A", "C")
    g.AddEdge("B", "D")
    g.AddEdge("C", "D")

    fmt.Println(g.BFS("A")) // [A B C D]
}
```

### Пример 5: Set операции

```go
package main

import "fmt"

type Set[T comparable] map[T]struct{}

func NewSet[T comparable](items ...T) Set[T] {
    s := make(Set[T])
    for _, item := range items {
        s[item] = struct{}{}
    }
    return s
}

func (s Set[T]) Add(item T)            { s[item] = struct{}{} }
func (s Set[T]) Has(item T) bool       { _, ok := s[item]; return ok }
func (s Set[T]) Remove(item T)         { delete(s, item) }

func (s Set[T]) Intersect(other Set[T]) Set[T] {
    result := make(Set[T])
    for k := range s {
        if other.Has(k) {
            result.Add(k)
        }
    }
    return result
}

func main() {
    a := NewSet("go", "python", "java")
    b := NewSet("go", "rust", "python")

    intersection := a.Intersect(b)
    for k := range intersection {
        fmt.Println(k) // go, python (в любом порядке)
    }
}
```

---

## Вопросы с собеседования

**Q: Что произойдёт при чтении несуществующего ключа из map?**

Вернётся zero value для типа значения. Паники нет. Для `map[string]int` несуществующий ключ вернёт `0`, для `map[string][]int` — `nil`. Чтобы различить "ключ есть, значение 0" и "ключа нет" — использовать `v, ok := m[key]`.

---

**Q: Потокобезопасна ли map в Go?**

Нет. Конкурентная запись (или запись+чтение) вызывает `fatal error: concurrent map writes` и падение программы. Решения: `sync.RWMutex` (универсально) или `sync.Map` (для read-heavy или disjoint-keys сценариев).

---

**Q: Почему порядок итерации по map случаен?**

Go намеренно рандомизирует начало итерации начиная с Go 1. Цель: не дать разработчикам полагаться на порядок, который является деталью реализации и может меняться. Если нужен порядок — собери ключи в слайс и отсортируй.

---

**Q: Можно ли изменять map во время итерации range?**

Формально да — паники не будет. Но результат нестабилен: удалённые ключи могут исчезнуть из итерации, добавленные — появиться или нет. Для безопасного удаления — сначала собери ключи в слайс, потом удаляй.

---

**Q: Почему map занимает память даже после удаления всех элементов?**

Бакеты не освобождаются при `delete` — только помечаются как пустые. Физическая память остаётся выделенной для возможного повторного использования. Чтобы вернуть память — пересоздать map: `m = make(map[K]V)`.

---

**Q: Чем sync.Map отличается от map + mutex? Когда что использовать?**

`sync.Map` оптимизирована для read-heavy нагрузки: чтения существующих ключей происходят без блокировки через атомарный read-словарь. При частых записях новых ключей sync.Map работает медленнее, чем простой `map + sync.RWMutex`.

Правило: по умолчанию — `map + RWMutex`. `sync.Map` — только если профилировщик показывает, что это bottleneck, и нагрузка read-heavy.

---

**Q: Почему нельзя использовать слайс как ключ map?**

Ключом может быть только comparable тип — тот, который поддерживает `==`. Слайсы, map-ы, функции — не comparable. Попытка использовать их как ключ — compile-time ошибка. Если нужен составной ключ — используй структуру из comparable полей или строку.

---

**Q: Что такое `map[string]struct{}` и зачем так делают?**

Это идиоматический способ реализовать Set (множество). `struct{}` — zero-size тип, не занимает памяти. В отличие от `map[string]bool`, значения `struct{}` не занимают места. Операции: добавить — `m[k] = struct{}{}`, проверить — `_, ok := m[k]`.
