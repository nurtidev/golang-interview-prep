# Профилирование Go приложений (pprof)

> Вакансия Rocket Tech явно требует: "Опыт профилирования и тюнинга Go приложений"
> Это значит — тебя могут спросить как теорию, так и попросить объяснить реальный кейс.

---

## Что такое pprof

`pprof` — стандартный профайлер Go. Показывает где программа тратит время (CPU) и память (heap). Встроен в стандартную библиотеку, не нужно подключать сторонние инструменты.

---

## Типы профилей

| Профиль | Что показывает | Когда использовать |
|---|---|---|
| `cpu` | Где CPU тратит время (sampling 100Hz) | Сервис медленный, высокий CPU |
| `heap` | Объекты в памяти, где аллоцируются | Утечка памяти, OOM |
| `goroutine` | Все живые горутины и их стек | Утечка горутин, deadlock |
| `allocs` | Все аллокации (включая уже собранные GC) | Много аллокаций → давление на GC |
| `block` | Где горутины блокируются | Contention на mutex/channel |
| `mutex` | Contention на mutex | mutex слишком долго занят |
| `trace` | Полная трассировка рантайма | Latency spikes, GC паузы |

---

## Как подключить (HTTP endpoint)

```go
import _ "net/http/pprof"  // side-effect import

func main() {
    // Отдельный порт для pprof, НИКОГДА не на публичный
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    
    // ... основной сервер
}
```

После этого доступны:
- `http://localhost:6060/debug/pprof/` — список профилей
- `http://localhost:6060/debug/pprof/goroutine?debug=2` — горутины с полным стеком

---

## Как снять и проанализировать профиль

### CPU профиль (30 секунд под нагрузкой)
```bash
# Снять
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Или сохранить файл
curl -o cpu.prof http://localhost:6060/debug/pprof/profile?seconds=30
go tool pprof cpu.prof
```

### Heap профиль
```bash
go tool pprof http://localhost:6060/debug/pprof/heap
```

### В интерактивном режиме pprof
```
(pprof) top10          # топ 10 функций по времени/памяти
(pprof) top10 -cum     # топ по cumulative (включая дочерние вызовы)
(pprof) list funcName  # построчный анализ функции
(pprof) web            # открыть граф в браузере (нужен graphviz)
(pprof) svg            # сохранить граф как SVG
```

---

## Программный профиль (в тестах)

```go
import (
    "os"
    "runtime/pprof"
    "testing"
)

func BenchmarkPaymentProcess(b *testing.B) {
    f, _ := os.Create("cpu.prof")
    defer f.Close()
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()

    for i := 0; i < b.N; i++ {
        processPayment(amount)
    }
}
```

```bash
go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof ./...
go tool pprof cpu.prof
```

---

## Реальный кейс: утечка горутин (eMoney-style)

**Симптом:** сервис медленно растёт по памяти. Через 12 часов — OOM.

**Диагностика:**
```bash
# Смотрим количество горутин
curl http://localhost:6060/debug/pprof/goroutine?debug=1

# Видим: 50000 горутин, все висят в одном месте:
# goroutine ... [chan receive, 2 hours]:
# main.processWebhook(...)
```

**Причина:**
```go
// ПЛОХО — горутина висит навсегда если никто не пишет в канал
func processWebhook(payload []byte) {
    resultCh := make(chan Result)
    go func() {
        resultCh <- callExternalProvider(payload) // внешний провайдер упал
    }()
    result := <-resultCh // блокируемся навсегда
}
```

**Фикс:**
```go
func processWebhook(ctx context.Context, payload []byte) error {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    resultCh := make(chan Result, 1) // буферизованный!
    go func() {
        resultCh <- callExternalProvider(ctx, payload)
    }()

    select {
    case result := <-resultCh:
        return handleResult(result)
    case <-ctx.Done():
        return fmt.Errorf("provider timeout: %w", ctx.Err())
    }
}
```

---

## Реальный кейс: неожиданный CPU spike (Sergek-style)

**Симптом:** при ~1000 rps — CPU 95%, хотя логика простая.

**Диагностика:**
```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
(pprof) top10
```

**Видим:**
```
  flat  flat%   sum%        cum   cum%
 2.30s 38.33% 38.33%      2.30s 38.33%  runtime.mallocgc
 1.20s 20.00% 58.33%      1.20s 20.00%  encoding/json.Marshal
```

**Причина:** в каждом запросе создаём новый `json.Marshal` и много мелких аллокаций.

**Фикс:**
```go
// ПЛОХО — каждый раз новый encoder
func handleEvent(event Event) []byte {
    data, _ := json.Marshal(event)
    return data
}

// ХОРОШО — переиспользуем буфер через sync.Pool
var bufPool = sync.Pool{
    New: func() interface{} { return new(bytes.Buffer) },
}

func handleEvent(event Event) []byte {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufPool.Put(buf)
    }()
    
    json.NewEncoder(buf).Encode(event)
    return buf.Bytes()
}
```

---

## Реальный кейс: memory leak через map (классика)

**Симптом:** heap растёт, но объекты "удаляются" из map.

**Диагностика:**
```bash
go tool pprof http://localhost:6060/debug/pprof/heap
(pprof) top10
# видим огромный map с маленькими значениями
```

**Причина:** Go не уменьшает внутренний массив map после удаления ключей.

```go
// Map с 1M ключей, потом удалили все — память НЕ освобождается
cache := make(map[string][]byte)
for i := 0; i < 1_000_000; i++ {
    cache[fmt.Sprintf("key-%d", i)] = data
}
for k := range cache {
    delete(cache, k) // bucket'ы остались в памяти!
}
```

**Фикс:** пересоздать map или использовать `sync.Map` / кастомный LRU.

---

## Trace — когда pprof не хватает

`trace` нужен когда нужно увидеть **latency спайки** и **GC паузы**:

```go
import "runtime/trace"

f, _ := os.Create("trace.out")
trace.Start(f)
defer trace.Stop()
```

```bash
go tool trace trace.out
# Открывает браузер с timeline: горутины, GC, сеть
```

**Что смотреть в trace:**
- GC паузы > 1ms — проблема для real-time
- Горутины ждут scheduler слишком долго — GOMAXPROCS
- `syscall` блоки — возможно сетевые вызовы без таймаутов

---

## Escape analysis — аллокации в heap

```bash
go build -gcflags="-m" ./...
# или подробнее:
go build -gcflags="-m=2" ./...
```

**Читать вывод:**
```
./payment.go:42:6: moved to heap: req    ← аллокация в heap (дорого)
./payment.go:55:14: inlining call to...  ← функция заинлайнена (хорошо)
```

**Частые причины escape в heap:**
- Возврат указателя на локальную переменную
- Интерфейс с конкретным типом (`interface{}`)
- Замыкание захватывает переменную
- Слишком большая переменная (> 64KB)

---

## Benchmark как инструмент оптимизации

```go
func BenchmarkProcessPayment(b *testing.B) {
    b.ReportAllocs() // показывает аллокации
    b.ResetTimer()
    
    for i := 0; i < b.N; i++ {
        processPayment(testPayload)
    }
}
```

```bash
go test -bench=BenchmarkProcessPayment -benchmem -count=5 ./...
# -benchmem показывает: allocs/op и B/op
```

**Сравниваем до/после:**
```bash
go test -bench=. -count=5 > before.txt
# ... вносим изменения
go test -bench=. -count=5 > after.txt
benchstat before.txt after.txt
```

---

## Вопросы которые могут задать на интервью

**Q: Как ты находил bottleneck в своих сервисах?**
> Подключал pprof через side-effect import на отдельный порт. Снимал CPU профиль под нагрузкой (30 секунд нагрузочного теста). В `top10 -cum` смотрел что занимает больше всего cumulative времени. Потом `list funcName` для построчного анализа.

**Q: Что такое escape analysis и зачем он нужен?**
> Компилятор Go решает — хранить переменную на стеке (дёшево, GC не нужен) или в heap (GC must track). `go build -gcflags="-m"` показывает эти решения. Если критичный hot path аллоцирует в heap — это давление на GC и latency спайки.

**Q: Как обнаружить утечку горутин?**
> `/debug/pprof/goroutine?debug=2` — показывает все горутины со стеком. Если их количество монотонно растёт при одинаковой нагрузке — утечка. Ищу горутины с состоянием `[chan receive]` или `[select]` — часто это горутины без timeout/context.

**Q: Как использовал sync.Pool?**
> В Sergek — переиспользовал буферы для сериализации event'ов. При ~200k событий/сутки это значительно снижало давление на GC: меньше аллокаций → меньше GC пауз → стабильнее latency.
