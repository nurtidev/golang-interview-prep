# ОС: Виртуальная Память, mmap, Syscalls в глубину

---

## 1. Виртуальная память в деталях

### Страничная организация памяти

Физическая память делится на **страницы** (обычно 4 КБ). Виртуальное пространство тоже делится на страницы. MMU транслирует виртуальную страницу → физическая страница через **page table**.

```
Виртуальный адрес (64-bit):
 63        48 47       39 38       30 29       21 20       12 11        0
┌───────────┬──────────┬──────────┬──────────┬──────────┬────────────┐
│  (не исп.) │  PGD idx │  PUD idx │  PMD idx │  PTE idx │  offset    │
│  16 bits   │  9 bits  │  9 bits  │  9 bits  │  9 bits  │  12 bits   │
└───────────┴──────────┴──────────┴──────────┴──────────┴────────────┘
     ↑ знаковое расширение (canonical addresses)      ↑ 4KB страница
```

**4-уровневая трансляция (x86_64):**
```
CR3 register → PGD (Page Global Directory)
                 │
                 ▼ [индекс из VA[47:39]]
               PUD (Page Upper Directory)
                 │
                 ▼ [индекс из VA[38:30]]
               PMD (Page Middle Directory)
                 │
                 ▼ [индекс из VA[29:21]]
               PTE (Page Table Entry)
                 │
                 ▼ [индекс из VA[20:12]]
               Physical Frame + offset → Physical Address
```

**Page Table Entry (PTE) флаги:**
```
Bit 0: Present    — страница в памяти (если 0 → page fault)
Bit 1: Writable   — можно писать
Bit 2: User       — доступна userspace (иначе только kernel)
Bit 5: Accessed   — MMU ставит при обращении (для LRU)
Bit 6: Dirty      — MMU ставит при записи (нужно сбрасывать на диск)
Bit 7: Huge Page  — страница 2MB или 1GB
Bits 12-51: Physical Frame Number
Bit 63: NX (No-Execute) — страница с данными, не исполнять
```

### TLB (Translation Lookaside Buffer)

TLB — ассоциативный кеш трансляций VA → PA. Обычно 64-1536 записей.

```
Без TLB: каждое обращение к памяти = 4 обращения к page table = 5 обращений к RAM!
С TLB: ~1 ns hit (vs ~100 ns RAM) — ускорение в 100x для cacheable доступов
```

**TLB miss** → hardware page table walk (на x86_64 — аппаратный, без участия ОС).

**TLB flush при context switch:**
- Смена process → смена CR3 → **полный TLB flush** (на старых CPU)
- Современные CPU: PCID (Process-Context Identifier) — TLB хранит записи для нескольких процессов одновременно → частичный flush
- ASID (Address Space ID) на ARM — аналог PCID

**KPTI (Kernel Page Table Isolation)** — защита от Meltdown: ядро и userspace имеют разные page tables. Стоит ~5-30% производительности на syscall-intensive workloads.

---

## 2. Page Fault — как работает

### Типы page fault

```
Page Fault
├── Minor (soft) fault — страница есть в RAM, просто не в page table
│   Примеры: copy-on-write, разделяемые библиотеки (первый доступ)
│   Стоимость: ~1 мкс (только обновить PTE)
│
└── Major (hard) fault — страница не в RAM (на диске)
    Примеры: swap, mmap файла (lazy loading), первый доступ после mmap
    Стоимость: ~10 мс (disk I/O) или ~0.1 мс (SSD)
```

### Stack growth через page fault

```
Стек процесса растёт сверху вниз. Ядро не выделяет всё сразу.

Initial:
┌──────────────────┐  ← stack top (высокий адрес)
│   Stack page 1   │  ← выделена
├──────────────────┤
│   guard page     │  ← PROT_NONE — ловит переполнение
├──────────────────┤
│ (не выделено)    │

После вызова функции с большими локальными переменными:
→ Page fault на guard page area
→ Ядро: "это расширение стека?" → да → выделить новую страницу
→ Stack вырос вниз
```

**Stack overflow:** когда стек достиг лимита (`ulimit -s`, обычно 8MB), page fault не обрабатывается ядром как расширение → SIGSEGV.

В Go: горутины начинают с 2-8 КБ стека, растут динамически до `GOMAXSTACK` (1 ГБ по умолчанию) через **stack copying** (не mmap!).

---

## 3. mmap в деталях

### mmap системный вызов

```c
void* mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

**prot флаги:**
```
PROT_READ   = 1  — читать
PROT_WRITE  = 2  — писать
PROT_EXEC   = 4  — исполнять (JIT компиляторы!)
PROT_NONE   = 0  — нет доступа (guard pages)
```

**flags флаги:**
```
MAP_SHARED   — изменения видны другим процессам + записываются в файл
MAP_PRIVATE  — Copy-on-Write, изменения локальны для процесса
MAP_ANONYMOUS— не связан с файлом (для аллокации памяти)
MAP_FIXED    — именно по этому адресу (опасно!)
MAP_POPULATE — prefault все страницы сейчас (избегаем major faults позже)
MAP_HUGETLB  — использовать huge pages
MAP_LOCKED   — не свопировать (нужны права)
```

### Как malloc использует mmap

```
malloc реализация (glibc):
├── Маленькие аллокации (<128KB): sbrk() / brk() — расширяет heap сегмент
└── Большие аллокации (≥128KB): mmap(MAP_ANONYMOUS|MAP_PRIVATE)
    └── free() → munmap() — сразу возвращает память ОС

Go runtime:
└── mmap(MAP_ANONYMOUS) для всего — потом управляет своим allocator
```

### File-backed mmap — zero-copy file I/O

```go
import "golang.org/x/sys/unix"

func mmapRead(filename string) ([]byte, func(), error) {
    f, err := os.Open(filename)
    if err != nil {
        return nil, nil, err
    }

    fi, err := f.Stat()
    if err != nil {
        f.Close()
        return nil, nil, err
    }

    data, err := unix.Mmap(
        int(f.Fd()),
        0,
        int(fi.Size()),
        unix.PROT_READ,
        unix.MAP_SHARED,
    )
    f.Close()  // fd можно закрыть после mmap — маппинг остаётся!
    if err != nil {
        return nil, nil, err
    }

    cleanup := func() { unix.Munmap(data) }
    return data, cleanup, nil
}
```

**Преимущества над read():**
- Zero-copy: ядро передаёт страницы напрямую, без буфера userspace
- Lazy loading: страницы загружаются при первом обращении
- Shared между процессами: несколько процессов mmap одного файла → одна физическая копия
- Прямой random access без lseek

**Подводные камни:**
- SIGBUS при truncate файла после mmap — страница валидна в VA, но файл короче
- Major faults при первом доступе — для критичного кода использовать `madvise(MADV_WILLNEED)`
- На 32-bit системах — ограничен VA space

### madvise — подсказки ядру

```go
import "golang.org/x/sys/unix"

// Сообщить ядру о намерениях с памятью
unix.Madvise(data, unix.MADV_SEQUENTIAL)   // читаем последовательно → prefetch
unix.Madvise(data, unix.MADV_RANDOM)       // случайный доступ → не prefetch
unix.Madvise(data, unix.MADV_WILLNEED)     // скоро понадобится → загрузить в RAM сейчас
unix.Madvise(data, unix.MADV_DONTNEED)     // больше не нужно → выгрузить из RAM
unix.Madvise(data, unix.MADV_FREE)         // можно освободить при нехватке памяти
unix.Madvise(data, unix.MADV_HUGEPAGE)     // использовать huge pages для этого региона
unix.Madvise(data, unix.MADV_NOHUGEPAGE)   // не использовать huge pages
```

**Реальный пример:** RocksDB использует `MADV_RANDOM` для SST файлов (случайный доступ к индексам) и `MADV_SEQUENTIAL` при compaction.

### Shared memory между процессами

```go
// Process A: создаёт shared memory
fd, _ := unix.MemfdCreate("myshm", 0)
unix.Ftruncate(fd, 4096)
data, _ := unix.Mmap(fd, 0, 4096, unix.PROT_READ|unix.PROT_WRITE, unix.MAP_SHARED)

// Передаёт fd через Unix socket (sendmsg с SCM_RIGHTS)
// Process B: получает fd, делает mmap
// Теперь оба видят одну физическую страницу

// POSIX shared memory (через /dev/shm)
fd, _ = unix.Open("/dev/shm/myshm", unix.O_CREAT|unix.O_RDWR, 0600)
// или shm_open() через cgo
```

---

## 4. Huge Pages

### Зачем Huge Pages

Стандартная страница 4KB. При большом heap (8 GB):
- Page table entries: 8 GB / 4 KB = 2 миллиона PTE
- TLB покрывает (1536 entries * 4KB) = 6 MB → для 8GB heap TLB hit rate ~0.07%!
- Huge page 2MB: 8 GB / 2 MB = 4096 PTE, TLB покрывает 3 GB → отлично

**Выигрыш:** снижение TLB misses → 10-30% ускорение для memory-intensive приложений (databases, JVM, Go с большим heap).

### Transparent Huge Pages (THP)

Ядро Linux автоматически объединяет 4KB страницы в 2MB huge pages.

```bash
# Настройки THP
cat /sys/kernel/mm/transparent_hugepage/enabled
# always  [madvise]  never
# always = агрессивно, madvise = только там где попросили, never = выключено

# Для latency-sensitive приложений (Redis, некоторые БД):
echo never > /sys/kernel/mm/transparent_hugepage/enabled
# THP может вызывать latency spikes при khugepaged работе

# Для throughput-sensitive (аналитика):
echo always > /sys/kernel/mm/transparent_hugepage/enabled
```

### Explicit Huge Pages в Go

```go
// Go 1.18+: GODEBUG=madvdontneed=0 оставляет страницы
// Go использует madvise(MADV_DONTNEED) по умолчанию на Linux

// Для huge pages: включить THP с madvise и в коде:
// madvise(ptr, size, MADV_HUGEPAGE)
// Но в Go нет прямого API — через cgo или //go:linkname
```

---

## 5. Swap и OOM Killer

### Как работает swap

```
RAM заполняется → ядро выбирает "жертву" через LRU + kswapd →
страница записывается в swap (disk) → PTE помечается как не-present →
при следующем доступе — major page fault → страница загружается обратно
```

**swappiness** (`/proc/sys/vm/swappiness`, 0-200):
- 0 = избегать swap до последнего
- 60 = default
- 100 = агрессивный swap

**Для серверов:** обычно swappiness=10 или swap вообще отключён. Databases ненавидят swap — задержки от 10 мс до секунд.

### OOM Killer

Когда памяти нет совсем (RAM + swap заполнены):

```
1. Ядро вызывает OOM Killer
2. Выбирает процесс с максимальным oom_score
3. oom_score = (память процесса / доступная память) * 1000 + oom_score_adj
4. SIGKILL → процесс убит

# Управление:
echo -17 > /proc/PID/oom_score_adj  # -1000 до 1000, -1000 = никогда не убивать
echo 1000 > /proc/PID/oom_score_adj # убить первым

# Просмотр:
cat /proc/PID/oom_score
```

**Защита критических процессов:**
```bash
# Ядро никогда не убьёт этот процесс:
echo -1000 > /proc/$(pgrep myservice)/oom_score_adj
```

---

## 6. Пространства имён и cgroups (основа контейнеров)

### Linux Namespaces

| Namespace | Что изолирует | Системный вызов |
|-----------|--------------|-----------------|
| **pid** | Дерево процессов | clone(CLONE_NEWPID) |
| **net** | Сетевой стек, интерфейсы, routing | clone(CLONE_NEWNET) |
| **mnt** | Точки монтирования, файловая система | clone(CLONE_NEWNS) |
| **uts** | hostname, domainname | clone(CLONE_NEWUTS) |
| **ipc** | Message queues, semaphores, shared memory | clone(CLONE_NEWIPC) |
| **user** | UID/GID маппинги | clone(CLONE_NEWUSER) |
| **cgroup** | cgroup root | clone(CLONE_NEWCGROUP) |
| **time** | Системное время (Linux 5.6) | clone(CLONE_NEWTIME) |

**Контейнер = процесс в наборе namespace'ов.**

### cgroups v2 — ограничение ресурсов

```bash
# Создать cgroup
mkdir /sys/fs/cgroup/myapp

# Ограничить CPU (0.5 CPU = 50000 мкс из 100000 мкс периода)
echo "50000 100000" > /sys/fs/cgroup/myapp/cpu.max

# Ограничить RAM
echo "512M" > /sys/fs/cgroup/myapp/memory.max

# Запустить процесс в cgroup
echo $PID > /sys/fs/cgroup/myapp/cgroup.procs

# Статистика
cat /sys/fs/cgroup/myapp/memory.current   # текущее использование
cat /sys/fs/cgroup/myapp/cpu.stat         # cpu.usage_usec, throttled_usec
```

**Go в контейнерах:** `runtime.GOMAXPROCS` нужно настраивать по cpu.max, а не NumCPU(). Пакет `go.uber.org/automaxprocs` делает это автоматически.

---

## 7. Системные вызовы — низкоуровневый взгляд

### Механизм syscall на x86_64

```
Userspace:
    MOV rax, 1       ; syscall number (1 = write)
    MOV rdi, 1       ; arg1: fd (stdout)
    MOV rsi, buf     ; arg2: buffer
    MOV rdx, len     ; arg3: length
    SYSCALL          ; переход в kernel mode

Процессор:
    1. Сохранить rip (instruction pointer) в ring3
    2. Сохранить rsp (stack pointer)
    3. Переключиться на kernel stack (из MSR_LSTAR)
    4. Сохранить регистры
    5. Вызвать sys_write() в ядре
    6. SYSRET → восстановить регистры, вернуться в userspace

Стоимость: ~100-300 ns (пустой syscall без I/O)
```

### vDSO — syscalls без перехода в ядро

Некоторые syscalls не требуют kernel mode. Ядро маппирует в адресное пространство каждого процесса страницу с быстрыми реализациями:

```bash
cat /proc/PID/maps | grep vdso
# 7fff9b7fe000-7fff9b800000 r-xp  [vdso]
```

Syscalls реализованные через vDSO (без перехода в ядро!):
- `clock_gettime(CLOCK_REALTIME)` — читает из shared memory
- `gettimeofday()`
- `getcpu()`
- `time()`

```go
// В Go time.Now() использует vDSO автоматически → ~10 ns вместо ~300 ns
start := time.Now()
// ...
elapsed := time.Since(start)
```

### epoll — эффективный I/O multiplexing

```
poll/select: O(n) — при каждом вызове ядро проверяет все n дескрипторов
epoll: O(1) — ядро сам уведомляет при событии

```

```go
// Принцип работы epoll (упрощённо):
// 1. epoll_create() — создать epoll instance
// 2. epoll_ctl(EPOLL_CTL_ADD, fd, EPOLLIN) — зарегистрировать fd
// 3. epoll_wait() — блокируемся ждать события, O(1) пробуждение
// 4. Ядро уведомляет только когда fd готов

// Go runtime использует epoll внутри (netpoller):
// - goroutine делает net.Read() → если данных нет → goroutine suspended
// - netpoller thread: epoll_wait() → данные пришли → resume goroutine
// Это позволяет N горутин с блокирующим API на M OS потоках
```

**Level-triggered vs Edge-triggered:**
- LT (default): epoll_wait вернёт пока данные есть (можно читать частями)
- ET (EPOLLET): epoll_wait вернёт только при новых данных → нужно читать до EAGAIN

### io_uring — async I/O нового поколения (Linux 5.1+)

```
Проблема epoll: всё равно есть syscall per operation
io_uring: очередь запросов в shared memory → батчинг syscalls → io_uring_enter() одним syscall
```

```
SQ (Submission Queue) ─── userspace пишет запросы (read, write, accept...)
                         │
                    ┌────▼────┐
                    │  Kernel  │  обрабатывает асинхронно
                    └────┬────┘
                         │
CQ (Completion Queue) ◄── ядро пишет результаты
```

Go ещё не использует io_uring нативно (обсуждается), но некоторые библиотеки (giouring) позволяют использовать из Go.

---

## 8. Память в Go: runtime internals

### Go Memory Allocator

Go использует tcmalloc-подобный аллокатор:

```
Размеры объектов разделены на классы (size classes):
8, 16, 24, 32, 48, 64, 80, 96, 112, 128 ... 32768 байт

mspan — непрерывный регион физических страниц, разделённый на объекты одного класса:
┌────────────────────────────────────────────┐
│ mspan (size class 32: объекты по 32 байта) │
│ [obj][obj][obj][obj][obj][free][free][obj] │
└────────────────────────────────────────────┘

mcache — per-P (per-logical-processor) кеш mspan'ов:
└── P0: mcache → hot mspan для каждого size class (без блокировок!)

mcentral — глобальный кеш mspan'ов (с блокировкой):
└── список mspan'ов с свободными объектами

mheap — большой кеш, выделяет mspan'ы из ОС (mmap):
└── треап-дерево свободных регионов
```

**Путь аллокации:**
```
make([]byte, 32) →
  1. P.mcache → нашли mspan с free slots → atomic mark → done (fast path, нет блокировок)
  2. mcache miss → mcentral → забрать mspan → в mcache → done
  3. mcentral miss → mheap → sweepSpan или grow heap → mmap новую память
```

**Большие объекты (>32KB):** сразу из mheap, выделяется точное количество страниц.

### Stack allocator

```go
// Горутина стартует с 2KB-8KB стека (зависит от версии Go)
// При нехватке: stack growth через stack copying (не сегментированные стеки!)

// Проверка при каждом вызове функции:
// if sp < stackguard0 { runtime.morestack() }
// morestack():
//   1. Выделить новый стек (вдвое больше)
//   2. Скопировать старый стек
//   3. Обновить все указатели на стек (scan goroutine stack)
//   4. Продолжить выполнение

// Shrink стека: при GC если стек используется < 1/4 от размера → уменьшить
```

### Goroutine stack frames

```go
// Каждый вызов функции:
// - возвращаемые значения (reserved place)
// - аргументы (для старых версий; в Go 1.17+ передаются в регистрах)
// - saved frame pointer (BP register, для pprof)
// - local variables
// - saved return address

// Go 1.17+: register-based calling convention
// - до 9 integer arguments в registers (AX, BX, CX, DI, SI, R8, R9, R10, R11)
// - до 15 float arguments в registers (X0-X14)
// Оставшиеся — в стеке
// ~10-20% ускорение для функций с аргументами
```

---

## 9. Вопросы на собеседовании

### Q: Как работает виртуальная память? Что происходит при page fault?

**Ответ:** Каждый процесс имеет собственное виртуальное адресное пространство. MMU транслирует виртуальные адреса в физические через page table (4 уровня на x86_64). TLB кеширует трансляции (1-1536 записей, ~1ns hit). При обращении к странице которой нет в RAM — page fault: ядро прерывает процесс, загружает страницу (из swap или файла), обновляет PTE и resume. Minor fault (~1 мкс) — страница есть в RAM но не в page table (CoW, shared libs). Major fault (~10 мс HDD, ~0.1 мс SSD) — страница на диске. TLB flush при context switch — дорогая операция, поэтому горутины дешевле OS потоков.

---

### Q: Как работает mmap? Когда использовать вместо read()?

**Ответ:** mmap() отображает файл или анонимную память в виртуальное адресное пространство. Страницы загружаются лениво (major page fault при первом обращении). Преимущества: zero-copy (нет userspace буфера), random access без lseek, shared между процессами (MAP_SHARED), ядро сам управляет page cache. Когда использовать: большие файлы с random access (databases, log storage), IPC через shared memory, JIT компиляция (PROT_EXEC). Не стоит для: маленьких файлов или строго последовательного чтения (обычный read() может быть быстрее из-за меньшего overhead).

---

### Q: Чем cgroup отличается от namespace? Как реализованы контейнеры?

**Ответ:** Namespace — изоляция видимости: процесс видит только своё дерево процессов (pid ns), свои сетевые интерфейсы (net ns), свою файловую систему (mnt ns). Cgroup — ограничение ресурсов: CPU квота, лимит памяти, disk I/O bandwidth. Контейнер = процесс в наборе namespace'ов + cgroup для ограничений + chroot/pivot_root для файловой системы. Docker/containerd создают эти примитивы через clone(CLONE_NEWPID|CLONE_NEWNET|...) + cgroupfs. Всё это Linux kernel features, Docker — просто удобный фронтенд.

---

### Q: Что такое epoll? Почему Go может держать миллионы горутин на нескольких потоках?

**Ответ:** epoll — механизм уведомлений о готовности I/O дескрипторов за O(1). Регистрируешь N дескрипторов через epoll_ctl(), затем epoll_wait() спит пока хоть один не станет готов — ядро разбудит только при событии. Go runtime использует epoll (Linux) / kqueue (BSD) в netpoller: горячий loop (sysmon + netpoller goroutine) ждёт на epoll_wait(). Когда горутина делает net.Read() — если данных нет, горутина parkGoRoutine() (suspended), поток свободен. При приходе данных — netpoller делает readyG(g) → планировщик добавляет горутину в run queue. Один OS поток обслуживает тысячи горутин в I/O ожидании.
