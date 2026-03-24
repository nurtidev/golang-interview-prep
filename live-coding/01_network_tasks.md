# Live Coding: Задачи по сетевому трафику (KazDream / DPI тематика)

Формат каждой задачи:
- **Задача** — что просят
- **Что проверяют** — на что смотрит интервьюер
- **Решение** — рабочий код с комментариями

---

## Задача 1: Топ-N IP по трафику (Sliding Window)

**Условие:**
Приходит поток событий: `(src_ip, bytes, timestamp)`. Реализуй функцию которая возвращает топ-N IP адресов по суммарному трафику за последние `window` секунд. Данные приходят в реальном времени.

**Что проверяют:** map, heap/sort, sliding window, time.

```go
package main

import (
    "container/heap"
    "fmt"
    "sort"
    "time"
)

type Event struct {
    SrcIP     string
    Bytes     int64
    Timestamp time.Time
}

type TrafficWindow struct {
    window time.Duration
    events []Event
    totals map[string]int64 // ip → суммарные байты в окне
}

func NewTrafficWindow(window time.Duration) *TrafficWindow {
    return &TrafficWindow{
        window: window,
        totals: make(map[string]int64),
    }
}

func (tw *TrafficWindow) Add(e Event) {
    tw.events = append(tw.events, e)
    tw.totals[e.SrcIP] += e.Bytes
    tw.evict(e.Timestamp)
}

// Удаляем устаревшие события из начала
func (tw *TrafficWindow) evict(now time.Time) {
    cutoff := now.Add(-tw.window)
    i := 0
    for i < len(tw.events) && tw.events[i].Timestamp.Before(cutoff) {
        tw.totals[tw.events[i].SrcIP] -= tw.events[i].Bytes
        if tw.totals[tw.events[i].SrcIP] <= 0 {
            delete(tw.totals, tw.events[i].SrcIP)
        }
        i++
    }
    tw.events = tw.events[i:]
}

// TopN возвращает N IP с наибольшим трафиком
func (tw *TrafficWindow) TopN(n int) []IPTraffic {
    result := make([]IPTraffic, 0, len(tw.totals))
    for ip, bytes := range tw.totals {
        result = append(result, IPTraffic{IP: ip, Bytes: bytes})
    }
    sort.Slice(result, func(i, j int) bool {
        return result[i].Bytes > result[j].Bytes
    })
    if n > len(result) {
        n = len(result)
    }
    return result[:n]
}

type IPTraffic struct {
    IP    string
    Bytes int64
}

func main() {
    tw := NewTrafficWindow(5 * time.Minute)
    now := time.Now()

    tw.Add(Event{"192.168.1.1", 1000, now})
    tw.Add(Event{"192.168.1.2", 5000, now})
    tw.Add(Event{"192.168.1.1", 2000, now.Add(1 * time.Minute)})
    // Это событие выйдет из окна через 5 минут
    tw.Add(Event{"192.168.1.3", 9000, now.Add(10 * time.Minute)})

    for _, t := range tw.TopN(3) {
        fmt.Printf("%s: %d bytes\n", t.IP, t.Bytes)
    }
}
```

**Усложнение от интервьюера:** "Как сделать это конкурентно-безопасным?"
```go
type SafeTrafficWindow struct {
    mu TrafficWindow
    sync.RWMutex
}
func (s *SafeTrafficWindow) Add(e Event) {
    s.Lock()
    defer s.Unlock()
    s.mu.Add(e)
}
func (s *SafeTrafficWindow) TopN(n int) []IPTraffic {
    s.RLock()
    defer s.RUnlock()
    return s.mu.TopN(n)
}
```

---

## Задача 2: Детектор Port Scan

**Условие:**
Есть поток соединений `(src_ip, dst_ip, dst_port, timestamp)`. Считай IP-адрес сканером портов если за последние 10 секунд он обратился к более чем `threshold` уникальным портам одного хоста.

**Что проверяют:** map вложенных структур, set через map, логика детекции.

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Connection struct {
    SrcIP     string
    DstIP     string
    DstPort   uint16
    Timestamp time.Time
}

type PortScanDetector struct {
    mu        sync.Mutex
    threshold int
    window    time.Duration
    // src_ip → dst_ip → [(port, time)]
    conns map[string]map[string][]portTime
}

type portTime struct {
    port uint16
    t    time.Time
}

func NewPortScanDetector(threshold int, window time.Duration) *PortScanDetector {
    return &PortScanDetector{
        threshold: threshold,
        window:    window,
        conns:     make(map[string]map[string][]portTime),
    }
}

// Add возвращает true если обнаружено сканирование
func (d *PortScanDetector) Add(c Connection) bool {
    d.mu.Lock()
    defer d.mu.Unlock()

    if d.conns[c.SrcIP] == nil {
        d.conns[c.SrcIP] = make(map[string][]portTime)
    }

    // Добавляем соединение
    d.conns[c.SrcIP][c.DstIP] = append(
        d.conns[c.SrcIP][c.DstIP],
        portTime{c.DstPort, c.Timestamp},
    )

    // Evict старые записи и подсчитываем уникальные порты
    cutoff := c.Timestamp.Add(-d.window)
    var fresh []portTime
    uniquePorts := make(map[uint16]struct{})

    for _, pt := range d.conns[c.SrcIP][c.DstIP] {
        if pt.t.After(cutoff) {
            fresh = append(fresh, pt)
            uniquePorts[pt.port] = struct{}{}
        }
    }
    d.conns[c.SrcIP][c.DstIP] = fresh

    return len(uniquePorts) > d.threshold
}

func main() {
    detector := NewPortScanDetector(5, 10*time.Second)
    now := time.Now()
    scanner := "10.0.0.1"
    target := "192.168.1.100"

    // Сканируем 10 портов за 5 секунд
    for port := uint16(80); port < 90; port++ {
        c := Connection{scanner, target, port, now.Add(time.Duration(port) * time.Second / 10)}
        if detector.Add(c) {
            fmt.Printf("Port scan detected: %s → %s\n", scanner, target)
            break
        }
    }
}
```

---

## Задача 3: Worker Pool для обработки пакетов с graceful shutdown

**Условие:**
Реализуй систему обработки пакетов: N воркеров читают из канала, обрабатывают пакет (парсинг заголовка), результат пишут в output канал. Graceful shutdown по сигналу — дождаться завершения всех воркеров.

**Что проверяют:** goroutines, WaitGroup, context, каналы, graceful shutdown.

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"
)

type Packet struct {
    ID      int
    SrcIP   string
    DstIP   string
    Payload []byte
}

type ParsedPacket struct {
    PacketID int
    Protocol string
    Size     int
}

func parsePacket(p Packet) ParsedPacket {
    // Имитация парсинга
    time.Sleep(10 * time.Millisecond)
    return ParsedPacket{
        PacketID: p.ID,
        Protocol: detectProtocol(p.Payload),
        Size:     len(p.Payload),
    }
}

func detectProtocol(payload []byte) string {
    if len(payload) > 4 && string(payload[:4]) == "HTTP" {
        return "HTTP"
    }
    return "UNKNOWN"
}

func RunWorkerPool(
    ctx context.Context,
    numWorkers int,
    input <-chan Packet,
) <-chan ParsedPacket {
    output := make(chan ParsedPacket, numWorkers*2)
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for {
                select {
                case pkt, ok := <-input:
                    if !ok {
                        // Канал закрыт — воркер заканчивает работу
                        fmt.Printf("worker %d: input closed, exiting\n", workerID)
                        return
                    }
                    result := parsePacket(pkt)
                    select {
                    case output <- result:
                    case <-ctx.Done():
                        return
                    }
                case <-ctx.Done():
                    fmt.Printf("worker %d: context cancelled\n", workerID)
                    return
                }
            }
        }(i)
    }

    // Закрываем output когда все воркеры завершились
    go func() {
        wg.Wait()
        close(output)
    }()

    return output
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())

    // Graceful shutdown по сигналу
    sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, syscall.SIGTERM, syscall.SIGINT)
    go func() {
        <-sigs
        fmt.Println("shutdown signal received")
        cancel()
    }()

    // Генератор пакетов
    input := make(chan Packet, 100)
    go func() {
        defer close(input)
        for i := 0; i < 50; i++ {
            select {
            case <-ctx.Done():
                return
            case input <- Packet{ID: i, SrcIP: "10.0.0.1", Payload: []byte("HTTP GET /")}:
            }
        }
    }()

    output := RunWorkerPool(ctx, 4, input)

    processed := 0
    for result := range output {
        processed++
        fmt.Printf("processed packet %d: proto=%s size=%d\n",
            result.PacketID, result.Protocol, result.Size)
    }

    fmt.Printf("total processed: %d\n", processed)
}
```

---

## Задача 4: Rate Limiter (Token Bucket) с per-IP лимитами

**Условие:**
Реализуй rate limiter для входящих соединений. Каждый IP имеет свой bucket: максимум `burst` соединений, скорость пополнения `rate` соединений в секунду. Если bucket пуст — соединение дропается.

**Что проверяют:** алгоритм token bucket, time, sync, map cleanup.

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type TokenBucket struct {
    tokens   float64
    maxTokens float64
    rate     float64 // tokens per second
    lastTime time.Time
    mu       sync.Mutex
}

func NewTokenBucket(rate, burst float64) *TokenBucket {
    return &TokenBucket{
        tokens:    burst,
        maxTokens: burst,
        rate:      rate,
        lastTime:  time.Now(),
    }
}

func (b *TokenBucket) Allow() bool {
    b.mu.Lock()
    defer b.mu.Unlock()

    now := time.Now()
    elapsed := now.Sub(b.lastTime).Seconds()
    b.lastTime = now

    // Пополняем токены за прошедшее время
    b.tokens = min(b.maxTokens, b.tokens+elapsed*b.rate)

    if b.tokens >= 1 {
        b.tokens--
        return true
    }
    return false
}

// Per-IP rate limiter
type IPRateLimiter struct {
    mu       sync.Mutex
    buckets  map[string]*TokenBucket
    rate     float64
    burst    float64
    // Очистка неактивных записей
    lastSeen map[string]time.Time
}

func NewIPRateLimiter(rate, burst float64) *IPRateLimiter {
    rl := &IPRateLimiter{
        buckets:  make(map[string]*TokenBucket),
        lastSeen: make(map[string]time.Time),
        rate:     rate,
        burst:    burst,
    }
    go rl.cleanup()
    return rl
}

func (rl *IPRateLimiter) Allow(ip string) bool {
    rl.mu.Lock()
    bucket, ok := rl.buckets[ip]
    if !ok {
        bucket = NewTokenBucket(rl.rate, rl.burst)
        rl.buckets[ip] = bucket
    }
    rl.lastSeen[ip] = time.Now()
    rl.mu.Unlock()

    return bucket.Allow()
}

// Удаляем IP которые не было видно > 5 минут
func (rl *IPRateLimiter) cleanup() {
    ticker := time.NewTicker(1 * time.Minute)
    for range ticker.C {
        rl.mu.Lock()
        cutoff := time.Now().Add(-5 * time.Minute)
        for ip, t := range rl.lastSeen {
            if t.Before(cutoff) {
                delete(rl.buckets, ip)
                delete(rl.lastSeen, ip)
            }
        }
        rl.mu.Unlock()
    }
}

func min(a, b float64) float64 {
    if a < b {
        return a
    }
    return b
}

func main() {
    limiter := NewIPRateLimiter(10, 20) // 10 req/s, burst 20

    ip := "192.168.1.1"
    allowed, denied := 0, 0

    for i := 0; i < 30; i++ {
        if limiter.Allow(ip) {
            allowed++
        } else {
            denied++
        }
    }

    fmt.Printf("allowed: %d, denied: %d\n", allowed, denied)
    // Burst = 20, значит первые 20 пройдут, остальные 10 — нет
}
```

---

## Задача 5: Парсер сетевого лога

**Условие:**
Дан лог файл в формате:
```
2024-01-15 10:23:45 SRC=192.168.1.1 DST=8.8.8.8 PROTO=TCP SPT=54321 DPT=443 LEN=60
2024-01-15 10:23:46 SRC=10.0.0.5 DST=1.1.1.1 PROTO=UDP SPT=1234 DPT=53 LEN=40
```
Распарси файл, посчитай: топ-5 destination IP, распределение по протоколам, суммарный трафик.

**Что проверяют:** разбор строк, regexp или strings, map, sort, обработка ошибок.

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "sort"
    "strconv"
    "strings"
)

type LogEntry struct {
    SrcIP    string
    DstIP    string
    Proto    string
    SrcPort  int
    DstPort  int
    Length   int
}

func parseLine(line string) (*LogEntry, error) {
    // Парсим поля вида KEY=VALUE
    fields := make(map[string]string)
    parts := strings.Fields(line)
    for _, part := range parts {
        if kv := strings.SplitN(part, "=", 2); len(kv) == 2 {
            fields[kv[0]] = kv[1]
        }
    }

    required := []string{"SRC", "DST", "PROTO", "SPT", "DPT", "LEN"}
    for _, k := range required {
        if _, ok := fields[k]; !ok {
            return nil, fmt.Errorf("missing field %s in line: %s", k, line)
        }
    }

    parseInt := func(s string) int {
        n, _ := strconv.Atoi(s)
        return n
    }

    return &LogEntry{
        SrcIP:   fields["SRC"],
        DstIP:   fields["DST"],
        Proto:   fields["PROTO"],
        SrcPort: parseInt(fields["SPT"]),
        DstPort: parseInt(fields["DPT"]),
        Length:  parseInt(fields["LEN"]),
    }, nil
}

type Stats struct {
    DstIPBytes   map[string]int64
    ProtoCount   map[string]int
    TotalBytes   int64
    TotalPackets int
    Errors       int
}

func analyzeLog(filename string) (*Stats, error) {
    f, err := os.Open(filename)
    if err != nil {
        return nil, err
    }
    defer f.Close()

    stats := &Stats{
        DstIPBytes: make(map[string]int64),
        ProtoCount: make(map[string]int),
    }

    scanner := bufio.NewScanner(f)
    for scanner.Scan() {
        line := scanner.Text()
        if line == "" {
            continue
        }

        entry, err := parseLine(line)
        if err != nil {
            stats.Errors++
            continue
        }

        stats.DstIPBytes[entry.DstIP] += int64(entry.Length)
        stats.ProtoCount[entry.Proto]++
        stats.TotalBytes += int64(entry.Length)
        stats.TotalPackets++
    }

    return stats, scanner.Err()
}

func topN(m map[string]int64, n int) []struct {
    Key   string
    Value int64
} {
    type kv struct {
        Key   string
        Value int64
    }
    var sorted []kv
    for k, v := range m {
        sorted = append(sorted, kv{k, v})
    }
    sort.Slice(sorted, func(i, j int) bool {
        return sorted[i].Value > sorted[j].Value
    })
    if n > len(sorted) {
        n = len(sorted)
    }
    return sorted[:n]
}

func main() {
    // Для теста — создаём временный файл
    content := `2024-01-15 10:23:45 SRC=192.168.1.1 DST=8.8.8.8 PROTO=TCP SPT=54321 DPT=443 LEN=1500
2024-01-15 10:23:46 SRC=10.0.0.5 DST=1.1.1.1 PROTO=UDP SPT=1234 DPT=53 LEN=40
2024-01-15 10:23:47 SRC=192.168.1.1 DST=8.8.8.8 PROTO=TCP SPT=54321 DPT=443 LEN=800`

    tmp, _ := os.CreateTemp("", "log*.txt")
    tmp.WriteString(content)
    tmp.Close()
    defer os.Remove(tmp.Name())

    stats, err := analyzeLog(tmp.Name())
    if err != nil {
        fmt.Println("error:", err)
        return
    }

    fmt.Printf("Total: %d packets, %d bytes, %d errors\n",
        stats.TotalPackets, stats.TotalBytes, stats.Errors)

    fmt.Println("\nTop-3 Destination IPs:")
    for _, t := range topN(stats.DstIPBytes, 3) {
        fmt.Printf("  %s: %d bytes\n", t.Key, t.Value)
    }

    fmt.Println("\nProtocol distribution:")
    for proto, count := range stats.ProtoCount {
        fmt.Printf("  %s: %d packets\n", proto, count)
    }
}
```

---

## Задача 6: DDoS детектор (аномальный трафик)

**Условие:**
Реализуй детектор DDoS: если количество уникальных источников для одного dst_ip превышает `threshold` за последние `window` секунд — генерируй алерт. Источники могут повторяться.

**Что проверяют:** set-семантика через map, sliding window, алерты через канал.

```go
package main

import (
    "fmt"
    "time"
)

type FlowEvent struct {
    SrcIP     string
    DstIP     string
    Timestamp time.Time
}

type DDoSAlert struct {
    DstIP       string
    UniqueSrcs  int
    DetectedAt  time.Time
}

type DDoSDetector struct {
    threshold int
    window    time.Duration
    // dst_ip → [(src_ip, time)]
    flows  map[string][]flowRecord
    Alerts chan DDoSAlert
}

type flowRecord struct {
    srcIP string
    t     time.Time
}

func NewDDoSDetector(threshold int, window time.Duration) *DDoSDetector {
    return &DDoSDetector{
        threshold: threshold,
        window:    window,
        flows:     make(map[string][]flowRecord),
        Alerts:    make(chan DDoSAlert, 100),
    }
}

func (d *DDoSDetector) Process(e FlowEvent) {
    d.flows[e.DstIP] = append(d.flows[e.DstIP], flowRecord{e.SrcIP, e.Timestamp})

    cutoff := e.Timestamp.Add(-d.window)
    var fresh []flowRecord
    uniqueSrcs := make(map[string]struct{})

    for _, r := range d.flows[e.DstIP] {
        if r.t.After(cutoff) {
            fresh = append(fresh, r)
            uniqueSrcs[r.srcIP] = struct{}{}
        }
    }
    d.flows[e.DstIP] = fresh

    if len(uniqueSrcs) > d.threshold {
        select {
        case d.Alerts <- DDoSAlert{
            DstIP:      e.DstIP,
            UniqueSrcs: len(uniqueSrcs),
            DetectedAt: e.Timestamp,
        }:
        default: // не блокируемся если алерты не читают
        }
    }
}

func main() {
    detector := NewDDoSDetector(3, 5*time.Second)

    // Читаем алерты в фоне
    go func() {
        for alert := range detector.Alerts {
            fmt.Printf("DDoS ALERT: %s targeted by %d unique IPs\n",
                alert.DstIP, alert.UniqueSrcs)
        }
    }()

    victim := "10.0.0.100"
    now := time.Now()

    // 5 разных источников атакуют жертву
    attackers := []string{"1.1.1.1", "2.2.2.2", "3.3.3.3", "4.4.4.4", "5.5.5.5"}
    for i, src := range attackers {
        detector.Process(FlowEvent{
            SrcIP:     src,
            DstIP:     victim,
            Timestamp: now.Add(time.Duration(i) * time.Second),
        })
    }

    time.Sleep(100 * time.Millisecond)
}
```

---

## Задача 7: Конкурентный кэш с TTL

**Условие:**
Реализуй in-memory кеш для хранения результатов DNS резолвинга (IP → hostname). Записи должны автоматически истекать через TTL. Безопасен для конкурентного доступа.

**Что проверяют:** sync.RWMutex, goroutine для очистки, time, generics (бонус).

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type entry[V any] struct {
    value     V
    expiresAt time.Time
}

type TTLCache[K comparable, V any] struct {
    mu      sync.RWMutex
    items   map[K]entry[V]
    stopCh  chan struct{}
}

func NewTTLCache[K comparable, V any](cleanupInterval time.Duration) *TTLCache[K, V] {
    c := &TTLCache[K, V]{
        items:  make(map[K]entry[V]),
        stopCh: make(chan struct{}),
    }
    go c.cleanup(cleanupInterval)
    return c
}

func (c *TTLCache[K, V]) Set(key K, value V, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = entry[V]{
        value:     value,
        expiresAt: time.Now().Add(ttl),
    }
}

func (c *TTLCache[K, V]) Get(key K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    e, ok := c.items[key]
    if !ok || time.Now().After(e.expiresAt) {
        var zero V
        return zero, false
    }
    return e.value, true
}

func (c *TTLCache[K, V]) Delete(key K) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.items, key)
}

func (c *TTLCache[K, V]) cleanup(interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    for {
        select {
        case <-ticker.C:
            now := time.Now()
            c.mu.Lock()
            for k, e := range c.items {
                if now.After(e.expiresAt) {
                    delete(c.items, k)
                }
            }
            c.mu.Unlock()
        case <-c.stopCh:
            return
        }
    }
}

func (c *TTLCache[K, V]) Stop() {
    close(c.stopCh)
}

func main() {
    // DNS кэш: IP → hostname, TTL 30 секунд
    cache := NewTTLCache[string, string](10 * time.Second)
    defer cache.Stop()

    cache.Set("8.8.8.8", "dns.google", 30*time.Second)
    cache.Set("1.1.1.1", "one.one.one.one", 30*time.Second)

    if hostname, ok := cache.Get("8.8.8.8"); ok {
        fmt.Println("8.8.8.8 →", hostname)
    }

    // Короткий TTL — быстро истечёт
    cache.Set("192.168.1.1", "local-host", 100*time.Millisecond)
    time.Sleep(200 * time.Millisecond)

    if _, ok := cache.Get("192.168.1.1"); !ok {
        fmt.Println("192.168.1.1: expired (as expected)")
    }
}
```

---

## Задача 8: Pipeline обработки пакетов (Fan-Out → Fan-In)

**Условие:**
Реализуй pipeline: читаем пакеты → параллельно классифицируем (HTTP / DNS / TLS / OTHER) → агрегируем статистику. Каждый классификатор — отдельная горутина.

**Что проверяют:** pipeline паттерн, fan-out, fan-in, merge каналов.

```go
package main

import (
    "fmt"
    "strings"
    "sync"
)

type RawPacket struct {
    ID      int
    DstPort uint16
    Payload string
}

type ClassifiedPacket struct {
    ID       int
    Protocol string
}

// Генератор пакетов
func generate(packets []RawPacket) <-chan RawPacket {
    out := make(chan RawPacket, len(packets))
    go func() {
        defer close(out)
        for _, p := range packets {
            out <- p
        }
    }()
    return out
}

// Один классификатор
func classify(in <-chan RawPacket) <-chan ClassifiedPacket {
    out := make(chan ClassifiedPacket, 10)
    go func() {
        defer close(out)
        for pkt := range in {
            out <- ClassifiedPacket{
                ID:       pkt.ID,
                Protocol: detectProto(pkt),
            }
        }
    }()
    return out
}

func detectProto(p RawPacket) string {
    switch {
    case p.DstPort == 53:
        return "DNS"
    case p.DstPort == 443:
        return "TLS"
    case p.DstPort == 80 || strings.HasPrefix(p.Payload, "HTTP"):
        return "HTTP"
    default:
        return "OTHER"
    }
}

// Fan-out: один источник → N обработчиков
func fanOut(in <-chan RawPacket, n int) []<-chan RawPacket {
    channels := make([]chan RawPacket, n)
    for i := range channels {
        channels[i] = make(chan RawPacket, 10)
    }

    go func() {
        defer func() {
            for _, ch := range channels {
                close(ch)
            }
        }()
        i := 0
        for pkt := range in {
            channels[i%n] <- pkt
            i++
        }
    }()

    result := make([]<-chan RawPacket, n)
    for i, ch := range channels {
        result[i] = ch
    }
    return result
}

// Fan-in: N каналов → один
func fanIn(channels ...<-chan ClassifiedPacket) <-chan ClassifiedPacket {
    out := make(chan ClassifiedPacket, 10)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan ClassifiedPacket) {
            defer wg.Done()
            for pkt := range c {
                out <- pkt
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}

func main() {
    packets := []RawPacket{
        {1, 80, "HTTP GET /"},
        {2, 53, "DNS query"},
        {3, 443, "TLS ClientHello"},
        {4, 8080, "custom proto"},
        {5, 80, "HTTP POST /api"},
        {6, 53, "DNS response"},
    }

    // Pipeline: generate → fanOut(3 workers) → classify → fanIn → aggregate
    source := generate(packets)
    shards := fanOut(source, 3)

    classified := make([]<-chan ClassifiedPacket, len(shards))
    for i, shard := range shards {
        classified[i] = classify(shard)
    }

    merged := fanIn(classified...)

    stats := make(map[string]int)
    for pkt := range merged {
        stats[pkt.Protocol]++
    }

    fmt.Println("Protocol statistics:")
    for proto, count := range stats {
        fmt.Printf("  %s: %d\n", proto, count)
    }
}
```

---

## Задача 9: Circular Buffer (Ring Buffer) для хранения последних N пакетов

**Условие:**
Реализуй кольцевой буфер фиксированного размера для хранения последних N пакетов. Используется для "захвата последних событий перед алертом". Потокобезопасен.

**Что проверяют:** кольцевой буфер, модульная арифметика, структуры данных.

```go
package main

import (
    "fmt"
    "sync"
)

type RingBuffer[T any] struct {
    mu     sync.Mutex
    buf    []T
    head   int // куда пишем следующий
    count  int // сколько элементов заполнено
    size   int
}

func NewRingBuffer[T any](size int) *RingBuffer[T] {
    return &RingBuffer[T]{
        buf:  make([]T, size),
        size: size,
    }
}

func (r *RingBuffer[T]) Push(v T) {
    r.mu.Lock()
    defer r.mu.Unlock()

    r.buf[r.head] = v
    r.head = (r.head + 1) % r.size
    if r.count < r.size {
        r.count++
    }
}

// Snapshot возвращает все элементы в порядке от старого к новому
func (r *RingBuffer[T]) Snapshot() []T {
    r.mu.Lock()
    defer r.mu.Unlock()

    if r.count == 0 {
        return nil
    }

    result := make([]T, r.count)
    start := (r.head - r.count + r.size) % r.size
    for i := 0; i < r.count; i++ {
        result[i] = r.buf[(start+i)%r.size]
    }
    return result
}

func (r *RingBuffer[T]) Len() int {
    r.mu.Lock()
    defer r.mu.Unlock()
    return r.count
}

type Packet struct {
    ID   int
    Info string
}

func main() {
    buf := NewRingBuffer[Packet](5) // храним последние 5 пакетов

    for i := 1; i <= 8; i++ {
        buf.Push(Packet{i, fmt.Sprintf("packet-%d", i)})
    }

    fmt.Println("Last 5 packets (oldest to newest):")
    for _, p := range buf.Snapshot() {
        fmt.Printf("  ID=%d %s\n", p.ID, p.Info)
    }
    // Ожидаем: 4, 5, 6, 7, 8
}
```

---

## Задача 10: Graceful Shutdown HTTP сервера с in-flight запросами

**Условие:**
Реализуй HTTP сервер с эндпоинтом `/analyze` который принимает пакет, обрабатывает его (имитация ~100ms). При получении SIGTERM — дождаться завершения всех текущих запросов, затем завершить работу. Новые запросы после сигнала не принимать.

**Что проверяют:** http.Server.Shutdown, context, signal handling, реальный production-код.

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

type AnalyzeRequest struct {
    SrcIP   string `json:"src_ip"`
    DstIP   string `json:"dst_ip"`
    Payload string `json:"payload"`
}

type AnalyzeResponse struct {
    Protocol string `json:"protocol"`
    Risk     string `json:"risk"`
}

func analyzeHandler(w http.ResponseWriter, r *http.Request) {
    var req AnalyzeRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "bad request", http.StatusBadRequest)
        return
    }

    // Имитация тяжёлой обработки
    select {
    case <-time.After(100 * time.Millisecond):
    case <-r.Context().Done():
        // Клиент отвалился или сервер шатается — прерываемся
        return
    }

    resp := AnalyzeResponse{
        Protocol: detectProto(req.Payload),
        Risk:     assessRisk(req.SrcIP),
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(resp)
}

func detectProto(payload string) string {
    if len(payload) > 4 && payload[:4] == "HTTP" {
        return "HTTP"
    }
    return "UNKNOWN"
}

func assessRisk(ip string) string {
    // В реальности — проверка по блок-листам, geo, reputation
    return "low"
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/analyze", analyzeHandler)
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })

    server := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Запускаем сервер в горутине
    go func() {
        log.Println("server started on :8080")
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    // Ждём сигнал завершения
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
    <-quit

    log.Println("shutdown signal received, waiting for in-flight requests...")

    // Даём 30 секунд на завершение текущих запросов
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Printf("forced shutdown: %v", err)
    }

    log.Println("server stopped gracefully")
    fmt.Println("done")
}
```

---

## Советы для live coding

### Перед тем как писать код:
1. **Уточни условие** — "Данные приходят отсортированные по времени?", "Сколько горутин?", "Нужен ли graceful shutdown?"
2. **Проговори подход** — "Буду использовать sliding window с map для уникальных IP"
3. **Оцени сложность** — "O(n) на Add, O(k log k) на TopN"

### Что упомянуть проактивно:
- **Конкурентность** — "В проде это будет вызываться параллельно, добавлю mutex/RWMutex"
- **Memory** — "При большом трафике map будет расти, нужна периодическая очистка"
- **Graceful shutdown** — "Воркеры слушают context.Done() для чистого завершения"
- **Ошибки** — "Логирую но не паникую на плохой записи — продолжаем обработку"

### Типичные ловушки:
```go
// 1. Горутина утекает если никто не читает канал
go func() {
    result <- compute()  // зависнет если result никто не читает!
}()
// Фикс: буферизованный канал или select с ctx.Done()

// 2. Closure захватывает переменную цикла
for i := 0; i < 10; i++ {
    go func() { fmt.Println(i) }()  // все напечатают одно и то же!
}
// Фикс: передать как аргумент
for i := 0; i < 10; i++ {
    go func(i int) { fmt.Println(i) }(i)
}

// 3. map concurrent read/write — race condition
// Фикс: sync.RWMutex или sync.Map
```
