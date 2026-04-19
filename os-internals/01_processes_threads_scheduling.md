# ОС: Процессы, Потоки и Планировщик

---

## Лекция

### 1. Процесс vs Поток

**Процесс** — изолированная единица выполнения с собственным адресным пространством (виртуальной памятью), файловыми дескрипторами, PID.

**Поток (Thread)** — единица выполнения внутри процесса. Потоки одного процесса разделяют: адресное пространство, файловые дескрипторы, глобальные переменные. Каждый поток имеет свой: стек, регистры, счётчик команд (PC).

```
Process:
├── Virtual Address Space (code, data, heap)
├── File Descriptors
└── Threads:
    ├── Thread 1: [stack] [registers] [PC]
    ├── Thread 2: [stack] [registers] [PC]
    └── Thread 3: [stack] [registers] [PC]
```

**Сравнение:**
| | Процесс | Поток |
|--|---------|-------|
| Изоляция | Полная (отдельная память) | Разделяют память процесса |
| Создание | Медленное (fork, новое адр. пространство) | Быстрое |
| Переключение | Медленное (смена page table) | Быстрое |
| Падение | Не влияет на другие процессы | Может убить весь процесс |
| IPC | Трудоёмкое (pipe, socket, shared memory) | Простое (общая память) |

---

### 2. Создание процессов в Linux

**fork()** — клонирует текущий процесс:
```c
pid_t pid = fork();
if (pid == 0) {
    // Дочерний процесс
    exec("new_program");
} else {
    // Родительский процесс, pid = PID дочернего
    wait(NULL);
}
```

**Copy-on-Write (CoW):** при fork() страницы памяти не копируются — дочерний процесс получает те же физические страницы, но помеченные как read-only. При первой записи в страницу — она копируется. Делает fork() дешёвым.

**exec()** — заменяет образ текущего процесса новой программой (загружает новый код в адресное пространство).

**vfork()** — оптимизированный fork для немедленного exec. Дочерний процесс разделяет адресное пространство родителя до exec().

---

### 3. Планировщик Linux (CFS)

**CFS (Completely Fair Scheduler)** — планировщик Linux с версии 2.6.23.

**Идея:** каждый процесс должен получить равную долю CPU. CFS использует "виртуальное время" (vruntime) — сколько CPU времени процесс уже использовал. Всегда выбирается процесс с наименьшим vruntime.

**Реализация:** красно-чёрное дерево, отсортированное по vruntime. Самый "голодный" процесс — всегда крайний левый узел. O(log n) для вставки, O(1) для выбора следующего.

**nice values:** от -20 (высокий приоритет) до +19 (низкий). Влияет на скорость роста vruntime. Nice=-20 → vruntime растёт медленно → процесс выбирается чаще.

**Preemption (вытеснение):**
- Таймер прерывает выполнение каждые `sched_min_granularity_ns` (по умолчанию ~0.75ms)
- Если vruntime текущего процесса > vruntime следующего в очереди → переключение

---

### 4. Context Switch (переключение контекста)

При переключении с процесса A на процесс B:
1. Сохранить регистры A в PCB (Process Control Block)
2. Сохранить PC, stack pointer A
3. Сменить page table (TLB flush — дорого!)
4. Загрузить регистры B из PCB
5. Загрузить PC, stack pointer B

**Стоимость:** переключение потоков дешевле (нет смены page table). Переключение процессов: ~1-10 микросекунд + TLB flush (аннулируется кеш адресных переводов).

**Горутины Go vs OS threads:** Go runtime управляет горутинами поверх небольшого количества OS потоков (M:N планирование). Переключение горутин дешевле OS thread switch — нет syscall, нет kernel mode, меньше данных для сохранения.

---

### 5. Виртуальная память

Каждый процесс видит собственное виртуальное адресное пространство (64-bit: 2^64 адресов), не зная о других процессах.

**MMU (Memory Management Unit)** + **Page Table:** транслирует виртуальные адреса в физические.

```
Виртуальный адрес → [MMU/Page Table] → Физический адрес в RAM
                                      или
                                      Page Fault (страница не в RAM)
```

**Page Fault:** страница не в RAM (на диске в swap/mmap файле). Ядро загружает страницу — процесс продолжает. Медленная операция (~100μs+).

**TLB (Translation Lookaside Buffer):** кеш трансляций виртуальных → физических адресов. При TLB miss → обращение к page table (дорого). Поэтому смена page table при context switch → TLB flush → все последующие обращения к памяти дороже.

**Структура виртуального адресного пространства:**
```
High address:
  Kernel space (недоступно из userspace)
  Stack (растёт вниз)
  ↓
  ...
  ↑
  Heap (растёт вверх)
  BSS (неинициализированные глобальные переменные)
  Data segment (инициализированные глобальные)
  Text segment (код программы)
Low address: 0x0000...
```

---

### 6. Системные вызовы (Syscalls)

Syscall — интерфейс между userspace и ядром. Приложение не может напрямую читать файлы или сеть — только через syscalls.

**Механизм:** `syscall` инструкция → переход в kernel mode → ядро выполняет → возврат в userspace.

**Стоимость:** ~100ns-1μs. Дорого из-за смены режима (user → kernel → user) и возможного context switch.

**Общие syscalls:**
```
Файлы: open, read, write, close, stat, mmap
Процессы: fork, exec, exit, wait, kill
Память: brk, mmap, munmap, mprotect
Сеть: socket, bind, listen, accept, connect, send, recv
Время: clock_gettime, nanosleep
```

**Go и syscalls:** Go по возможности использует non-blocking I/O + netpoller (epoll/kqueue). При блокирующем I/O поток "захватывается" до завершения, но горутина может быть перепланирована на другой поток.

---

### 7. IPC — межпроцессное взаимодействие

| Механизм | Описание | Скорость |
|----------|----------|----------|
| **Pipe** | Однонаправленный поток байтов | Fast (kernel buffer) |
| **Unix Socket** | Двунаправленный, full-duplex | Fast |
| **TCP Socket** | Сетевой, для разных хостов | Slower |
| **Shared Memory** | Общая область памяти | Fastest (нет копирования) |
| **mmap** | Отображение файла/памяти в адрес. пространство | Fast |
| **Signal** | Асинхронное уведомление | Minimal data |
| **Message Queue** | Структурированные сообщения | Medium |

---

## Практические примеры

### Пример 1: fork() в Go через os/exec

```go
// Go не предоставляет прямой доступ к fork() (кроме syscall пакета)
// Вместо этого — os/exec для запуска дочерних процессов

func runWorker(ctx context.Context, cmd string, args []string) error {
    c := exec.CommandContext(ctx, cmd, args...)
    c.Stdout = os.Stdout
    c.Stderr = os.Stderr

    if err := c.Start(); err != nil {
        return fmt.Errorf("start process: %w", err)
    }

    return c.Wait()
}

// Сигналы — взаимодействие через OS
func handleSignals() {
    sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, syscall.SIGTERM, syscall.SIGINT, syscall.SIGHUP)

    for sig := range sigs {
        switch sig {
        case syscall.SIGTERM, syscall.SIGINT:
            log.Println("graceful shutdown")
            os.Exit(0)
        case syscall.SIGHUP:
            log.Println("reloading config")
            reloadConfig()
        }
    }
}
```

### Пример 2: Мониторинг горутин и памяти

```go
import "runtime"

func printStats() {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)

    fmt.Printf("Goroutines: %d\n", runtime.NumGoroutine())
    fmt.Printf("Heap Alloc: %v MB\n", m.HeapAlloc/1024/1024)
    fmt.Printf("Heap Sys: %v MB\n", m.HeapSys/1024/1024)
    fmt.Printf("GC cycles: %d\n", m.NumGC)
    fmt.Printf("GC pause total: %v ms\n", m.PauseTotalNs/1000000)
    fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))
    fmt.Printf("NumCPU: %d\n", runtime.NumCPU())
}
```

### Пример 3: Работа с shared memory через mmap

```go
import (
    "golang.org/x/sys/unix"
    "os"
)

// Отображение файла в память — быстрый I/O
func mmapFile(path string) ([]byte, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer f.Close()

    fi, err := f.Stat()
    if err != nil {
        return nil, err
    }

    data, err := unix.Mmap(
        int(f.Fd()),
        0,
        int(fi.Size()),
        unix.PROT_READ,
        unix.MAP_SHARED,
    )
    if err != nil {
        return nil, err
    }
    // munmap когда данные больше не нужны:
    // defer unix.Munmap(data)
    return data, nil
}
```

---

## Вопросы на собеседовании

### Q: В чём разница между процессом и потоком?

**Ответ:** Процесс — изолированная единица с собственным виртуальным адресным пространством, файловыми дескрипторами, PID. Поток — единица выполнения внутри процесса. Потоки разделяют память и файловые дескрипторы процесса, но имеют свой стек и регистры. Создание потока дешевле (нет нового адресного пространства). Переключение потоков одного процесса дешевле (нет TLB flush). Изоляция у процессов лучше — падение одного не убивает других.

---

### Q: Что такое Context Switch? Почему это дорого?

**Ответ:** Context switch — сохранение состояния текущего потока/процесса и загрузка состояния следующего. Стоимость: ~1-10μs. Для процессов дороже — нужно сменить page table → TLB flush (аннулируется кеш трансляций виртуальных адресов). После flush первые обращения к памяти дороже (TLB miss). Поэтому большое количество OS потоков с частым переключением плохо для производительности. Go решает это через горутины (M:N планирование) — переключение горутин без syscall и без TLB flush.

---

### Q: Как работает fork() и Copy-on-Write?

**Ответ:** fork() клонирует процесс: ребёнок получает копию PCB, файловых дескрипторов, page table. Без CoW это было бы очень дорого (скопировать всю память). CoW: при fork() страницы памяти помечаются read-only у обоих процессов, они указывают на те же физические страницы. При первой записи в страницу — Page Fault → ядро копирует страницу → запись продолжается. Поэтому fork() быстрый (копируем только метаданные), а реальные копии создаются лениво при необходимости.

---

### Q: Что такое виртуальная память? Зачем она нужна?

**Ответ:** Виртуальная память — каждый процесс видит собственное непрерывное 64-bit адресное пространство, независимо от реального объёма RAM. MMU через page table транслирует виртуальные адреса в физические. Зачем: 1) Изоляция — процессы не могут читать память друг друга. 2) Больше памяти чем RAM — часть страниц на диске (swap), загружаются по требованию. 3) Эффективное разделение — несколько процессов могут отображать один файл в память (shared library). 4) mmap — отображение файлов в адресное пространство для быстрого I/O.

---

### Q: Что такое системный вызов? Почему он дороже обычной функции?

**Ответ:** Syscall — переход из userspace в kernel mode для выполнения привилегированных операций (I/O, работа с процессами, сеть). Стоимость ~100ns-1μs. Дороже обычной функции: 1) smена режима процессора (user → kernel) с сохранением состояния. 2) Проверки безопасности в ядре. 3) Смена стека. 4) Возможный context switch если операция блокирующая. В Go netpoller (epoll/kqueue) позволяет делать async I/O — горутина блокируется, но OS поток может выполнять другие горутины.

---

## File Descriptors, ulimit и сетевые лимиты ОС

### 8. File Descriptors и ulimit

**File Descriptor (FD)** — целое число, которое ядро выдаёт процессу при открытии любого ресурса: файл, сокет, pipe, eventfd. Всё в Linux — файл.

```
Процесс → таблица FD процесса → таблица открытых файлов ядра → inode
FD 0 = stdin
FD 1 = stdout
FD 2 = stderr
FD 3+ = всё остальное (файлы, сокеты, ...)
```

**Лимиты на FD:**

```bash
# Мягкий лимит (soft) — текущий, можно поднять до hard
ulimit -n
# 1024 (дефолт в большинстве дистрибутивов)

# Жёсткий лимит (hard) — потолок для обычного пользователя
ulimit -Hn
# 1048576

# Поднять soft лимит в текущей сессии
ulimit -n 65536

# Посмотреть лимиты конкретного процесса
cat /proc/<pid>/limits | grep "open files"

# Посмотреть сколько FD сейчас открыто
ls /proc/<pid>/fd | wc -l
# или
cat /proc/sys/fs/file-nr  # открыто / свободно / максимум системой
```

**Системный лимит (все процессы):**
```bash
cat /proc/sys/fs/file-max  # максимум FD для всей системы
# Поднять:
sysctl -w fs.file-max=2097152
```

**Постоянная настройка в `/etc/security/limits.conf`:**
```
# username  type    item     value
*           soft    nofile   65536
*           hard    nofile   1048576
# или для конкретного сервиса через systemd:
# LimitNOFILE=1048576
```

**Почему это важно для бэкенда:**

Каждое TCP-соединение = 1 FD. При дефолтном лимите 1024:
- Nginx/Go сервер при >1024 соединениях бросает `too many open files`
- Решение: поднять `ulimit -n` до 65536-1048576

```go
// Проверить текущий лимит из Go
import "golang.org/x/sys/unix"

var rLimit unix.Rlimit
unix.Getrlimit(unix.RLIMIT_NOFILE, &rLimit)
fmt.Printf("soft: %d, hard: %d\n", rLimit.Cur, rLimit.Max)

// Поднять программно (нужны права)
rLimit.Cur = 65536
unix.Setrlimit(unix.RLIMIT_NOFILE, &rLimit)
```

---

### 9. TCP стек ОС и сетевые лимиты

**Путь TCP-соединения через ядро:**

```
Client SYN →
  → SYN queue (incomplete connections, ждут ACK)
  → [SYN-ACK отправлен]
Client ACK →
  → Accept queue (complete connections, ждут accept())
  → accept() забирает соединение → FD
```

**Ключевые параметры:**

```bash
# Размер accept queue (backlog) — сколько установленных соединений ждут accept()
# Передаётся в listen(fd, backlog). Ограничен системным параметром:
cat /proc/sys/net/core/somaxconn   # дефолт 128 (!), обычно поднимают до 65535
sysctl -w net.core.somaxconn=65535

# Размер SYN queue
cat /proc/sys/net/ipv4/tcp_max_syn_backlog  # дефолт 128-1024
sysctl -w net.ipv4.tcp_max_syn_backlog=65536

# TIME_WAIT — состояние после закрытия соединения (ждёт 2*MSL = ~60 сек)
# При высоком RPS заканчиваются локальные порты (65535 штук)
cat /proc/sys/net/ipv4/tcp_tw_reuse    # переиспользовать TIME_WAIT сокеты (рекомендуется)
sysctl -w net.ipv4.tcp_tw_reuse=1

# Диапазон локальных портов для исходящих соединений
cat /proc/sys/net/ipv4/ip_local_port_range  # дефолт 32768-60999 = ~28K портов
sysctl -w net.ipv4.ip_local_port_range="1024 65535"  # расширить до ~64K

# Буферы сокетов
cat /proc/sys/net/core/rmem_max   # максимальный receive buffer
cat /proc/sys/net/core/wmem_max   # максимальный send buffer
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
```

**SO_REUSEPORT — несколько процессов/потоков на одном порту:**

```go
import (
    "net"
    "syscall"
    "golang.org/x/sys/unix"
)

// Позволяет нескольким goroutine/процессам слушать один порт
// Ядро балансирует входящие соединения между ними (без общего мьютекса на accept)
lc := net.ListenConfig{
    Control: func(network, address string, c syscall.RawConn) error {
        return c.Control(func(fd uintptr) {
            unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
        })
    },
}
listener, err := lc.Listen(context.Background(), "tcp", ":8080")
```

**Зачем SO_REUSEPORT:** без него все горутины конкурируют за один `accept()` → lock contention. С SO_REUSEPORT ядро само балансирует → линейный рост производительности с числом CPU.

**Диагностика сетевых проблем:**

```bash
# Статистика TCP состояний
ss -s

# Все соединения с состоянием
ss -tan | grep TIME_WAIT | wc -l
ss -tan | grep ESTABLISHED | wc -l

# Количество открытых FD по процессам
lsof -n | awk '{print $2}' | sort | uniq -c | sort -rn | head -10

# Счётчики ядра — переполнение backlog, retransmits
netstat -s | grep -i "overflow\|retransmit\|drop"

# Современная альтернатива netstat
ss -ltnp  # слушающие сокеты с именами процессов
```

---

### 10. epoll — эффективный I/O мультиплексинг

**Проблема select/poll:** при каждом вызове передают весь массив FD в ядро. При 10K соединений — O(n) на каждой итерации.

**epoll:** регистрируем FD один раз, ядро уведомляет только о готовых — O(1) на событие.

```
epoll_create() → epoll_fd
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, socket_fd, event)  → регистрируем FD
epoll_wait(epoll_fd, events, maxevents, timeout)        → блокируемся, получаем готовые
```

**Level-Triggered vs Edge-Triggered:**

| Режим | Когда уведомляет | Риск |
|-------|-----------------|------|
| **LT (Level-Triggered)** | Пока данные есть в буфере (дефолт) | Частые уведомления |
| **ET (Edge-Triggered)** | Только при появлении новых данных | Нужно читать до EAGAIN, иначе пропустишь данные |

**Go netpoller — как Go использует epoll:**

```
Горутина вызывает net.Read() →
  → данных нет → горутина паркуется (не блокирует OS поток) →
  → Go runtime добавляет FD в epoll →
  → данные пришли → epoll_wait() вернул событие →
  → runtime будит горутину → горутина продолжает выполнение
```

Это объясняет почему Go может держать миллион горутин при небольшом числе OS потоков — все сетевые ожидания через epoll, OS потоков нужно ровно столько сколько горутин реально выполняется.

```go
// В Go всё это скрыто — просто пишем синхронный код:
conn, _ := net.Dial("tcp", "example.com:80")
conn.Read(buf)  // горутина паркуется, OS поток свободен
// ^^ под капотом epoll
```

---

### 11. Сигналы — управление процессом

```
SIGTERM (15) — вежливый запрос завершиться. Можно поймать → graceful shutdown
SIGKILL  (9) — немедленное убийство ядром. Поймать НЕЛЬЗЯ
SIGINT   (2) — Ctrl+C в терминале. Можно поймать
SIGHUP   (1) — перезагрузка конфига (традиционно для демонов)
SIGPIPE      — запись в закрытый сокет/pipe. Дефолт: завершить процесс
```

**Graceful shutdown в Go:**

```go
func main() {
    server := &http.Server{Addr: ":8080"}

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)

    go func() {
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    sig := <-quit
    log.Printf("received signal: %v, shutting down...", sig)

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // Прекращаем принимать новые запросы, ждём завершения текущих
    if err := server.Shutdown(ctx); err != nil {
        log.Fatal("forced shutdown:", err)
    }
}
```

**Kubernetes и сигналы:**
```
kubectl delete pod → K8s отправляет SIGTERM → preStop hook (если есть) →
  → ждёт terminationGracePeriodSeconds (дефолт 30s) →
  → если не завершился → SIGKILL
```

Поэтому `terminationGracePeriodSeconds` должен быть больше времени обработки самого долгого запроса.

---

### 12. cgroups и namespaces — основа контейнеров

**Namespaces** — изоляция: каждый контейнер видит свою "вселенную".

| Namespace | Что изолирует |
|-----------|-------------|
| `pid` | Дерево процессов (PID 1 в контейнере ≠ PID 1 в хосте) |
| `net` | Сетевые интерфейсы, порты, маршруты |
| `mnt` | Файловая система (точки монтирования) |
| `uts` | Hostname |
| `ipc` | Shared memory, message queues |
| `user` | UID/GID маппинг |

**cgroups (Control Groups)** — ограничение ресурсов.

```bash
# Лимиты CPU и памяти для контейнера — это cgroups под капотом Docker/K8s
docker run --cpus="0.5" --memory="256m" myapp

# Посмотреть cgroup текущего процесса
cat /proc/self/cgroup

# Лимит памяти контейнера (внутри контейнера)
cat /sys/fs/cgroup/memory/memory.limit_in_bytes
```

**Что это значит для Go-сервиса в K8s:**

```yaml
resources:
  requests:
    memory: "128Mi"   # cgroup soft limit — гарантировано
    cpu: "100m"       # cgroup cpu.shares
  limits:
    memory: "256Mi"   # cgroup hard limit — при превышении → OOM Kill
    cpu: "500m"       # cgroup cpu.quota — при превышении → throttling (не kill!)
```

**OOM Kill vs CPU Throttling:**
- Превысил memory limit → ядро убивает процесс (OOM Kill) → Pod перезапускается
- Превысил CPU limit → ядро замедляет процесс (throttling) → latency растёт, процесс жив

**GOMAXPROCS и cgroups:**

```go
// По умолчанию Go читает количество CPU хоста, а не контейнера
// На ноде с 32 CPU в контейнере с лимитом 0.5 CPU → GOMAXPROCS=32
// Результат: 32 OS потока конкурируют за 0.5 CPU → огромный контекст свитч

// Решение — библиотека uber-go/automaxprocs
import _ "go.uber.org/automaxprocs"
// Автоматически читает cgroup лимит и выставляет правильный GOMAXPROCS
```

---

## Вопросы с интервью (OS internals)

**Q: Почему Go-сервис падает с "too many open files" при росте нагрузки?**

**Ответ:** Каждое TCP-соединение потребляет один file descriptor. Дефолтный лимит `ulimit -n = 1024` исчерпывается при ~1000 одновременных соединений. Диагностика: `cat /proc/<pid>/limits`, `ls /proc/<pid>/fd | wc -l`. Фикс: поднять `ulimit -n 65536` в системе и `LimitNOFILE=65536` в systemd unit файле сервиса. В K8s — через `securityContext` или системные настройки ноды.

**Q: Что такое TIME_WAIT и почему он проблема при высоком RPS?**

**Ответ:** После закрытия TCP-соединения инициатор закрытия входит в TIME_WAIT на 2*MSL (~60 сек) — чтобы поймать запоздавшие пакеты. Проблема: исходящие соединения используют локальные порты (32768-60999, ~28K). При 500 RPS за 60 сек накапливается 30K TIME_WAIT — порты заканчиваются, новые соединения не открываются. Решение: `tcp_tw_reuse=1` (переиспользовать TIME_WAIT сокеты для исходящих), расширить диапазон портов, или использовать keepalive (не закрывать соединения).

**Q: Как Go использует epoll? Почему горутины эффективнее OS потоков для I/O?**

**Ответ:** Go runtime при старте создаёт epoll instance. Когда горутина делает net.Read() и данных нет — runtime паркует горутину (снимает с OS потока) и регистрирует FD в epoll. OS поток свободен и выполняет другие горутины. Когда данные приходят — epoll_wait() возвращает событие, runtime будит горутину и ставит в очередь планировщика. Итог: тысячи I/O горутин выполняются на десятке OS потоков. OS поток с blocking I/O всё время заблокирован — нужен отдельный поток на каждое соединение.

**Q: Почему важно выставлять GOMAXPROCS правильно в K8s?**

**Ответ:** GOMAXPROCS определяет количество P в GMP модели и создаваемых OS потоков. Дефолтно Go читает количество CPU хоста (например 32). Контейнеру выделено 0.5 CPU через cgroup. 32 OS потока конкурируют за 0.5 CPU → ядро постоянно делает context switch → деградация производительности и рост latency. `uber-go/automaxprocs` читает cgroup лимит и выставляет GOMAXPROCS соответственно (при 0.5 CPU → GOMAXPROCS=1).

**Q: Чем SIGTERM отличается от SIGKILL? Как правильно делать graceful shutdown?**

**Ответ:** SIGTERM — запрос на завершение, процесс может поймать и завершить работу корректно (дочитать запросы, закрыть соединения, сбросить буферы). SIGKILL — немедленное убийство ядром, поймать невозможно. K8s при удалении пода сначала отправляет SIGTERM, ждёт terminationGracePeriodSeconds (30с), потом SIGKILL. Правильный graceful shutdown: signal.Notify(SIGTERM) → server.Shutdown(ctx с таймаутом) → дождаться завершения активных запросов → выйти. terminationGracePeriodSeconds должен быть больше таймаута Shutdown.

**Q: Что произойдёт если превысить memory limit в K8s?**

**Ответ:** cgroup memory hard limit — при превышении ядро OOM Kill-ит процесс. Pod перезапускается. В отличие от CPU throttling (замедление без смерти). Поэтому memory limit нужно выставлять с запасом (2x от typical usage). Диагностика: `kubectl describe pod` → Events → OOMKilled. В Go: контролировать через `GOMEMLIMIT` (мягкий лимит для GC) чуть ниже K8s memory limit — GC начнёт агрессивнее собирать мусор до того как ядро убьёт процесс.
