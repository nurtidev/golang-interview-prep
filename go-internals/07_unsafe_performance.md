# Go: unsafe, pprof, Высокопроизводительный код

---

## 1. Пакет unsafe

### Что такое unsafe

`unsafe` — пакет Go, дающий доступ к низкоуровневым операциям, которые обходят систему типов. Это не "небезопасный" в смысле security — это "не даёт гарантий типобезопасности".

```go
import "unsafe"

// Три основные функции:
unsafe.Sizeof(x)    // размер переменной в байтах (без косвенных ссылок)
unsafe.Alignof(x)   // требуемое выравнивание
unsafe.Offsetof(x.Field)  // смещение поля в структуре

// Базовый тип:
unsafe.Pointer  // тип указателя, совместимый с любым *T
```

### unsafe.Pointer правила конвертации

Go разрешает только 6 паттернов использования unsafe.Pointer:

```go
// 1. Конвертация *T1 → unsafe.Pointer → *T2 (reinterpret cast)
func Float64bits(f float64) uint64 {
    return *(*uint64)(unsafe.Pointer(&f))
}

// 2. Конвертация unsafe.Pointer → uintptr (только для вычислений, НЕ хранить!)
offset := unsafe.Offsetof(s.Field)
ptr := unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + offset)

// 3. Арифметика указателей для slice/array элементов
a := []int{1, 2, 3, 4, 5}
// Доступ к элементу без bounds check:
elem := *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&a[0])) + 2*unsafe.Sizeof(a[0])))

// ОПАСНО: сохранять uintptr в переменную — GC может переместить объект!
// BAD:
p := uintptr(unsafe.Pointer(&x))  // GC может переместить x!
// ...позже...
*(*int)(unsafe.Pointer(p)) = 42   // undefined behavior!

// GOOD: одна выражение без промежуточных uintptr переменных
*(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Offsetof(s.b))) = 42
```

### Реальный use case: zero-copy string ↔ []byte

```go
// Стандартный способ — выделяет память:
b := []byte(s)    // копирование
s := string(b)    // копирование

// unsafe: zero-copy (только для чтения!)
func StringToBytes(s string) []byte {
    // string header: {ptr, len}
    // slice header:  {ptr, len, cap}
    sp := (*[2]uintptr)(unsafe.Pointer(&s))
    bp := [3]uintptr{sp[0], sp[1], sp[1]}
    return *(*[]byte)(unsafe.Pointer(&bp))
}

func BytesToString(b []byte) string {
    return *(*string)(unsafe.Pointer(&b))
}

// Реальная имплементация (Go 1.20+):
func StringToBytes(s string) []byte {
    return unsafe.Slice(unsafe.StringData(s), len(s))
}

func BytesToString(b []byte) string {
    return unsafe.String(unsafe.SliceData(b), len(b))
}
```

**Когда использовать:** горячий путь с миллионами конвертаций (парсинг, сериализация). Нельзя мутировать байты — string неизменяем.

### Чтение/запись структур в бинарный формат (zero-copy сериализация)

```go
type Header struct {
    Magic   uint32
    Version uint16
    Flags   uint16
    Length  uint64
}

// Читаем бинарный буфер напрямую в структуру
func ParseHeader(data []byte) *Header {
    if len(data) < int(unsafe.Sizeof(Header{})) {
        return nil
    }
    return (*Header)(unsafe.Pointer(&data[0]))
    // Внимание: выравнивание должно совпадать!
}

// Пишем структуру как байты
func SerializeHeader(h *Header) []byte {
    size := int(unsafe.Sizeof(*h))
    return unsafe.Slice((*byte)(unsafe.Pointer(h)), size)
}
```

### sync/atomic с unsafe для lock-free структур

```go
// Атомарное хранение указателя на произвольный тип
type AtomicValue[T any] struct {
    p unsafe.Pointer
}

func (v *AtomicValue[T]) Load() *T {
    return (*T)(atomic.LoadPointer(&v.p))
}

func (v *AtomicValue[T]) Store(val *T) {
    atomic.StorePointer(&v.p, unsafe.Pointer(val))
}

func (v *AtomicValue[T]) CompareAndSwap(old, new *T) bool {
    return atomic.CompareAndSwapPointer(&v.p,
        unsafe.Pointer(old), unsafe.Pointer(new))
}
```

---

## 2. Выравнивание структур и padding

### Как компилятор добавляет padding

```go
// Плохо: 24 байта
type Bad struct {
    a bool    // 1 байт + 7 байт padding
    b float64 // 8 байт
    c bool    // 1 байт + 7 байт padding
}
// Итого: 1+7+8+1+7 = 24 байта

// Хорошо: 10 байт (с padding — 16)
type Good struct {
    b float64 // 8 байт
    a bool    // 1 байт
    c bool    // 1 байт
              // 6 байт padding до выравнивания 8
}
// Итого: 8+1+1+6 = 16 байт

// Правило: располагай поля от большего к меньшему
```

```go
// Проверка размеров
fmt.Println(unsafe.Sizeof(Bad{}))   // 24
fmt.Println(unsafe.Sizeof(Good{}))  // 16
fmt.Println(unsafe.Alignof(Bad{}))  // 8
```

### False sharing — cache line проблема

```go
// Плохо: x и y на одной cache line (64 байта)
// Два ядра обновляют разные поля → постоянный cache invalidation
type Counters struct {
    hits   int64
    misses int64
}

// Хорошо: разделить cache lines
type Counter struct {
    value int64
    _     [56]byte  // padding до 64 байт (cache line size)
}

type Counters struct {
    hits   Counter
    misses Counter
}
```

---

## 3. Escape Analysis — что уходит в heap

### Почему важно

Stack allocation — быстро (нет GC pressure, нет malloc). Heap allocation — медленно (GC overhead). Задача: максимум на стеке, минимум на куче.

```go
// Посмотреть escape analysis:
go build -gcflags="-m -m" ./...
// или для конкретного файла:
go build -gcflags="-m" main.go
```

### Правила escape на heap

```go
// 1. Указатель возвращается из функции
func newFoo() *Foo {
    f := Foo{}   // "f escapes to heap"
    return &f
}

// 2. Интерфейс boxing
var i interface{} = 42  // int уходит на heap при хранении в interface{}
fmt.Println(42)         // 42 escapes to heap! (Println принимает interface{})

// 3. Слайс растёт (append за пределы capacity)
s := make([]int, 0, 10)
for i := 0; i < 100; i++ {
    s = append(s, i)  // после capacity=10 → переаллокация на heap
}

// 4. Closure захватывает переменную по ссылке
x := 0
go func() { x++ }()  // x escapes to heap

// 5. Переменная живёт дольше функции (отправляется по каналу)
ch := make(chan *int)
n := 42
ch <- &n  // n escapes to heap
```

### Как избежать лишних аллокаций

```go
// BAD: boxing в interface{}
func process(v interface{}) { ... }

// GOOD: generics
func process[T any](v T) { ... }

// BAD: каждый вызов Printf аллоцирует
log.Printf("user %d logged in", userID)

// GOOD: fmt.Fprintf в bytes.Buffer, или структурированное логирование
// (slog, zerolog используют sync.Pool для буферов)

// BAD: конкатенация строк в цикле
result := ""
for _, s := range items {
    result += s  // каждая итерация — новая аллокация
}

// GOOD:
var b strings.Builder
for _, s := range items {
    b.WriteString(s)
}
result := b.String()
```

---

## 4. sync.Pool — переиспользование объектов

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 0, 4096)
    },
}

func process(data []byte) []byte {
    buf := bufPool.Get().([]byte)
    defer func() {
        buf = buf[:0]  // reset без освобождения памяти
        bufPool.Put(buf)
    }()

    buf = append(buf, data...)
    // ... обработка ...
    return append([]byte{}, buf...)  // копируем результат перед возвратом в pool
}
```

**Реальный пример — zerolog использует pool для JSON буферов:**
```go
// Zerolog: ~0 аллокаций на горячем пути
log.Info().
    Str("user", username).
    Int("id", userID).
    Msg("login")
```

### Pool + bytes.Buffer паттерн

```go
var pool = sync.Pool{
    New: func() interface{} { return new(bytes.Buffer) },
}

func encode(v interface{}) ([]byte, error) {
    buf := pool.Get().(*bytes.Buffer)
    buf.Reset()
    defer pool.Put(buf)

    if err := json.NewEncoder(buf).Encode(v); err != nil {
        return nil, err
    }

    // Копируем — нельзя вернуть buf.Bytes() напрямую (buf уйдёт в pool)
    result := make([]byte, buf.Len())
    copy(result, buf.Bytes())
    return result, nil
}
```

---

## 5. pprof — профилирование

### Виды профилей

| Профиль | Что измеряет |
|---------|-------------|
| **CPU** | Где процессор тратит время |
| **Memory (heap)** | Аллокации — сколько и откуда |
| **Goroutine** | Стек всех живых горутин |
| **Block** | Время ожидания на mutex/channel |
| **Mutex** | Contention на мьютексах |
| **Trace** | Полная трасса выполнения (goroutine scheduling, GC, syscalls) |

### Добавить pprof в HTTP сервер

```go
import _ "net/http/pprof"  // регистрирует /debug/pprof/* handlers

// В main:
go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

```bash
# Собрать CPU профиль (30 секунд)
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Memory профиль
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine дамп
curl http://localhost:6060/debug/pprof/goroutine?debug=2

# Block профиль (нужно включить: runtime.SetBlockProfileRate(1))
go tool pprof http://localhost:6060/debug/pprof/block

# Mutex профиль (нужно включить: runtime.SetMutexProfileFraction(1))
go tool pprof http://localhost:6060/debug/pprof/mutex
```

### Команды в pprof интерактивном режиме

```
(pprof) top10          — топ 10 функций по CPU/памяти
(pprof) top10 -cum     — с учётом вызовов (cumulative)
(pprof) list funcName  — построчный анализ функции
(pprof) web            — открыть flamegraph в браузере
(pprof) png            — сохранить граф как PNG
(pprof) tree           — дерево вызовов
(pprof) disasm funcName — дизассемблер
```

### Flamegraph — как читать

```
───────────────────────────────────────────
        http.(*ServeMux).ServeHTTP          ← вершина стека (то что вызвало)
 ────────────────────────────────────────
  myHandler    │   json.Marshal
 ────────────  │  ─────────────────────────
  db.Query     │  encoding/json.(*enc)...
 ────────────  │  ──────────────────
               │  reflect.Value.Field
───────────────────────────────────────────
                     main
```

Широкий прямоугольник = много времени. Ищи "плато" наверху — там нет вызовов вниз, это реальная работа.

### Написать benchmark + профилирование

```go
// bench_test.go
func BenchmarkProcess(b *testing.B) {
    data := generateTestData()
    b.ResetTimer()
    b.ReportAllocs()  // показывать аллокации

    for i := 0; i < b.N; i++ {
        result := Process(data)
        _ = result
    }
}
```

```bash
# Запустить benchmark с CPU профилем
go test -bench=BenchmarkProcess -benchmem -cpuprofile=cpu.prof -memprofile=mem.prof

# Анализировать
go tool pprof cpu.prof
go tool pprof mem.prof
```

### Запись trace

```go
import "runtime/trace"

f, _ := os.Create("trace.out")
trace.Start(f)
defer trace.Stop()

// ... код ...
```

```bash
go tool trace trace.out
# Открывает веб-интерфейс с:
# - Timeline всех goroutines
# - GC events
# - Network I/O
# - Syscalls
```

---

## 6. Техники оптимизации

### GOMAXPROCS и CPU affinity

```go
// По умолчанию GOMAXPROCS = количество CPU ядер
// Для CPU-intensive workloads — оставить как есть
// Для I/O-intensive — можно увеличить

runtime.GOMAXPROCS(runtime.NumCPU())

// В контейнерах: Go 1.19+ автоматически читает cgroup CPU limits
// Но для старых версий нужно:
import "go.uber.org/automaxprocs"
// или вручную:
// GOMAXPROCS = cpu.quota / cpu.period
```

### Memory-mapped I/O для больших файлов

```go
import "golang.org/x/sys/unix"

func processLargeFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close()

    fi, _ := f.Stat()
    size := fi.Size()

    // mmap вместо read() — ядро подгружает страницы по требованию
    data, err := unix.Mmap(int(f.Fd()), 0, int(size),
        unix.PROT_READ, unix.MAP_SHARED|unix.MAP_POPULATE)
    if err != nil {
        return err
    }
    defer unix.Munmap(data)

    // Обрабатываем data как []byte — zero-copy!
    // Ядро управляет page cache, не нужно читать весь файл в RAM
    return parseData(data)
}
```

### Избегать reflect в горячем пути

```go
// BAD: reflect дорог (~10-100x медленнее)
func getField(v interface{}, name string) interface{} {
    rv := reflect.ValueOf(v)
    return rv.FieldByName(name).Interface()
}

// GOOD: кодогенерация (easyjson, msgp) или type assertion
type User struct { Name string }

func (u *User) Get(name string) interface{} {
    switch name {
    case "Name":
        return u.Name
    }
    return nil
}
```

### Предаллокация слайсов и мап

```go
// BAD: N реаллокаций
result := []int{}
for i := 0; i < N; i++ {
    result = append(result, i)
}

// GOOD: 0 реаллокаций
result := make([]int, 0, N)
for i := 0; i < N; i++ {
    result = append(result, i)
}

// GOOD для map:
m := make(map[string]int, expectedSize)
```

### Батчинг системных вызовов

```go
// BAD: каждый Write() — отдельный syscall
for _, item := range items {
    conn.Write(item)  // syscall каждый раз
}

// GOOD: bufio.Writer буферизует
w := bufio.NewWriterSize(conn, 64*1024)
for _, item := range items {
    w.Write(item)
}
w.Flush()  // один syscall

// GOOD для сети: writev (scatter-gather I/O)
// net.Buffers реализует WriteTo через writev
bufs := make(net.Buffers, len(items))
copy(bufs, items)
bufs.WriteTo(conn)
```

### Inlining — помочь компилятору

```go
// Компилятор инлайнит функции если они не слишком большие (~80 AST nodes)
// Проверить что инлайнится:
go build -gcflags="-m" 2>&1 | grep "inlining call"

// Форсировать инлайн (осторожно, увеличивает размер бинарника)
//go:nosplit  — для функций без split stack (unsafe!)
//go:noinline — запретить инлайн (для профилирования)

// Пример: маленькая функция инлайнится автоматически
func add(a, b int) int { return a + b }  // будет инлайнена

// Слишком большая — нет
func complexLogic(a, b int) int {
    // 100 строк...
}
```

### SIMD через assembly или компилятор

```go
// Компилятор Go генерирует SIMD автоматически в некоторых случаях
// bytes.Equal, strings.Compare, copy — используют SIMD

// Для ручного SIMD — assembly файлы (.s)
// или cgo с intrinsics (редко нужно)

// Проверить что генерируется:
go build -gcflags="-S" 2>&1 | grep VMOVDQU  // AVX инструкции
```

---

## 7. Измерение производительности

### Правила профилирования

1. **Измеряй, не угадывай** — bottleneck почти никогда там где думаешь
2. **Профилируй под реальной нагрузкой** — microbenchmarks обманывают
3. **Один эксперимент за раз** — иначе не поймёшь что помогло
4. **Следи за p99, не average** — outliers убивают UX

```go
// Хороший benchmark:
func BenchmarkSerialize(b *testing.B) {
    users := generateUsers(1000)  // реалистичные данные

    b.ResetTimer()
    b.ReportAllocs()
    b.SetBytes(int64(len(users) * avgUserSize))

    for i := 0; i < b.N; i++ {
        result, err := json.Marshal(users)
        if err != nil {
            b.Fatal(err)
        }
        _ = result
    }
}
```

### runtime.MemStats — метрики памяти

```go
var stats runtime.MemStats
runtime.ReadMemStats(&stats)

// Ключевые метрики:
stats.HeapAlloc    // живые объекты на heap сейчас
stats.HeapInuse    // heap страницы в использовании
stats.HeapIdle     // heap страницы не используются (вернуть ОС?)
stats.HeapReleased // страницы возвращённые ОС
stats.NumGC        // количество GC циклов
stats.PauseTotalNs // суммарное время Stop-the-World пауз
stats.LastGC       // время последнего GC (unix nano)

// Аллокации за время работы:
stats.TotalAlloc   // суммарно аллоцировано (не убывает)
stats.Mallocs      // количество аллокаций
stats.Frees        // количество освобождений
```

### GOGC и память

```go
// GOGC=100 (по умолчанию): GC запускается когда heap вырос на 100% от прошлого GC
// GOGC=off: отключить GC (только для batch jobs!)
// GOGC=200: GC реже → меньше CPU, больше памяти

// Go 1.19+: GOMEMLIMIT — лимит общей памяти
// runtime/debug.SetMemoryLimit(1 * 1024 * 1024 * 1024)  // 1 GB
// В контейнерах: ставь GOMEMLIMIT = 90% от container memory limit
```

---

## 8. Вопросы на собеседовании

### Q: Зачем нужен unsafe? Когда использовать?

**Ответ:** unsafe даёт прямой доступ к памяти в обход системы типов — zero-copy конвертации string↔[]byte, чтение бинарных форматов без копирования, атомарные операции над указателями произвольных типов. Использовать когда: профилирование показало, что конкретная аллокация — bottleneck, и безопасные способы не решают. Риски: GC может переместить объект (нельзя хранить uintptr между операциями), нарушение выравнивания приводит к панике на arm/sparc, код тяжело поддерживать. Всегда добавляй комментарий почему это безопасно именно здесь.

---

### Q: Как найти утечку памяти в Go приложении?

**Ответ:**
1. `go tool pprof http://localhost:6060/debug/pprof/heap` — смотрим топ аллокаторов
2. Сравниваем два heap профиля с интервалом: `go tool pprof -base heap1.prof heap2.prof`
3. `/debug/pprof/goroutine?debug=2` — проверяем нет ли goroutine leak (горутин накапливается)
4. Метрики: `HeapAlloc` растёт без выравнивания → утечка объектов. `NumGoroutine()` растёт → goroutine leak.

Типичные причины: goroutine не завершается (забытый context cancel), глобальный кеш без eviction, http.Response.Body не закрыт (держит соединение в пуле).

---

### Q: Что такое escape analysis? Как влияет на производительность?

**Ответ:** Escape analysis — анализ компилятора, определяющий куда выделить переменную: стек (дёшево, GC не нужен) или кучу (GC overhead). На кучу "убегает": указатель возвращённый из функции, переменная захваченная замыканием, любое значение переданное в `interface{}` (boxing). Смотреть через `go build -gcflags="-m"`. Оптимизации: использовать generics вместо interface{}, предаллоцировать slice с нужным capacity, возвращать значения а не указатели (если объект небольшой и не нужен снаружи). Benchmark + `-benchmem` показывает количество аллокаций на операцию — цель часто 0 allocs/op.

---

### Q: Как работает sync.Pool?

**Ответ:** sync.Pool — кеш временных объектов для переиспользования. Снижает давление на GC. Get() берёт объект из пула или создаёт через New(). Put() возвращает объект. Важно: GC может очистить пул в любой момент (на каждом GC цикле). Нельзя хранить объекты с состоянием между запросами — только транзиентные буферы. Паттерн: Get → использовать → Reset → Put. Используется в стандартной библиотеке: fmt, encoding/json, sync, http.
