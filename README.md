# Golang Interview Prep

Материалы для подготовки к Go-собеседованиям в формате лекций.
Каждый файл: теория → примеры → вопросы с ответами.
Источник тем: реальные собесы (Авито, Касперский, Uzum и др.)

---

## Структура

### go-internals/
| Файл | Темы |
|------|------|
| ⭐ `01_slices.md` | Устройство слайса, append, copy, утечки памяти |
| `02_map.md` | hmap, sync.Map, конкурентность, set |
| ⭐ `03_interfaces.md` | iface/eface, nil trap, errors.Is/As |
| ⭐ `04_defer_panic_recover.md` | LIFO, замыкания, recover из горутин |
| `05_garbage_collector.md` | Tri-color mark-and-sweep, write barrier, escape analysis, GOGC, sync.Pool |
| `06_memory_stack_heap.md` | Стек горутины, growable stack, heap, escape analysis, TCMalloc, аллокатор |
| `07_unsafe_performance.md` | unsafe.Pointer, zero-copy string↔[]byte, struct padding/false sharing, escape analysis, sync.Pool, pprof, flamegraph, GOGC/GOMEMLIMIT |
| `08_testing.md` | Table-driven tests, testify, интерфейсы для тестируемости, моки, httptest, testcontainers, goleak, benchmarks |
| `09_design_patterns.md` | Functional Options, Singleton, Factory, Builder, Adapter, Decorator, Observer, Strategy, Repository, Circuit Breaker, Outbox |
| `10_oop_in_go.md` | Инкапсуляция, полиморфизм, абстракция, embedding vs наследование, утиная типизация |
| `11_go_vs_languages.md` | Плюсы/минусы Go, сравнение с Python, Java, Rust, Node.js, где применяется |

### concurrency/
| Файл | Темы |
|------|------|
| ⭐ `01_goroutines_scheduler.md` | GMP модель, work stealing, GOMAXPROCS |
| ⭐ `02_channels.md` | Аксиомы каналов, select, nil канал, утечки |
| `03_patterns.md` | Fan-In, Fan-Out, Pipeline, Worker Pool, Семафор |
| `04_sync_primitives.md` | Mutex, RWMutex, atomic, CAS, WaitGroup, deadlock |

### databases/
| Файл | Темы |
|------|------|
| ⭐ `01_indexes_transactions.md` | B-Tree, GIN, составные индексы, ACID, уровни изоляции, оконные функции |
| ⭐ `02_postgresql.md` | MVCC, WAL, JSONB, репликация, партиционирование, PgBouncer |
| `03_redis.md` | Структуры данных, персистентность, кластер, паттерны кеширования, distributed lock |
| `04_kafka.md` | Архитектура, партиции, consumer groups, гарантии доставки, Outbox pattern |
| `05_mysql.md` | InnoDB, кластерный индекс, репликация, MySQL vs PostgreSQL |
| `06_clickhouse.md` | Колоночное хранение, MergeTree, Materialized Views, OLAP |
| `07_elasticsearch.md` | Инвертированный индекс, PromQL, bool query, маппинг, агрегации |
| `08_rabbitmq.md` | Exchange типы, ACK/NACK, DLQ, prefetch, RabbitMQ vs Kafka |
| `09_functions_procedures.md` | Функции vs процедуры, триггеры, PL/pgSQL, вызов из Go |
| ⭐ `10_sql_practice.md` | DDL схема (rides/users/payments), JOIN, GROUP BY, оконные функции, типовые задачи (топ-N, дубликаты, LEFT JOIN IS NULL), EXPLAIN, PK vs UNIQUE |
| `11_normalization.md` | 1NF/2NF/3NF/BCNF — правила, примеры нарушений и исправлений, денормализация как трейдофф |

### infrastructure/
| Файл | Темы |
|------|------|
| `01_docker.md` | Namespaces, cgroups, OverlayFS, Dockerfile, multi-stage build, Compose |
| `02_kubernetes.md` | Архитектура, Pod, Deployment, Service, Probes, HPA, rolling update |

### observability/
| Файл | Темы |
|------|------|
| `01_prometheus_grafana.md` | Типы метрик, PromQL, алёрты, SLO/SLA, инструментация Go |

### os-internals/
| Файл | Темы |
|------|------|
| `01_processes_threads_scheduling.md` | Процессы vs потоки, CFS планировщик, context switch, виртуальная память, syscalls |
| `02_memory_virtual.md` | Page table (4 уровня), TLB, page fault, mmap, huge pages, swap/OOM, epoll, io_uring, Go allocator internals |

### networks/
| Файл | Темы |
|------|------|
| `01_http_grpc_tcp.md` | TCP handshake, HTTP/1.1 vs 2, HTTPS, gRPC stream vs unary |
| `02_nat_bgp_mobile.md` | NAT/NAPT, NAT traversal (STUN/TURN/ICE), BGP, автономные системы, LTE/5G core (EPC/5GC), GTP туннели, Wireshark/tcpdump |

### security/
| Файл | Темы |
|------|------|
| `01_web_security.md` | OWASP Top 10, SQL injection, XSS, JWT уязвимости, bcrypt/argon2, rate limiting, SSRF, CORS, secure headers, secrets management |

### live-coding/
| Файл | Темы |
|------|------|
| `01_network_tasks.md` | 10 боевых задач: sliding window, port scan detector, worker pool, rate limiter, log parser, DDoS detector, TTL cache, pipeline fan-out/fan-in, ring buffer, graceful shutdown |
| ⭐ `02_algorithms_complexity.md` | Big O, правила подсчёта, 10 паттернов: Two Pointers, Sliding Window, Binary Search, BFS, DFS, Backtracking, DP, Heap, Prefix Sum, Monotonic Stack |

### system-design/
| Файл | Темы |
|------|------|
| ⭐ `01_fundamentals.md` | Scalability, Performance, back-of-the-envelope, CAP теорема |
| `02_caching_api.md` | Стратегии кэширования, REST vs gRPC, rate limiting, observability |
| `03_databases.md` | Выбор БД, SQL vs NoSQL, шардирование, денормализация |
| `04_distributed_storage.md` | Репликация, консистентность, distributed transactions |
| `05_patterns.md` | Монолит vs микросервисы, saga, CQRS, event sourcing |
| ⭐ `06_cases_feed_chat.md` | Лента друзей (ВКонтакте/Instagram), мессенджер |
| `07_cases_booking_drive.md` | Booking.com, Google Drive |
| `08_cases_taxi_judge.md` | Яндекс.Такси, Leetcode/Online Judge |
| `09_principles.md` | SOLID, DRY, KISS, YAGNI, TDD, Cohesion & Coupling |

### question-bank/
| Файл | Темы |
|------|------|
| `01_golang_top100.md` | Топ 100 вопросов по Go с ответами |
| `02_databases_top20.md` | Топ 20 вопросов по базам данных |
| `03_concurrency_top20.md` | Топ 20 вопросов по конкурентности |
| `04_networks_top20.md` | Топ 20 вопросов по сетям |
| `05_system_design_top20.md` | Топ 20 вопросов по System Design |

---

## Топ вопросов по частоте (из реальных собесов)

1. **Горутины и планировщик** — 4 раза
2. **Каналы** — 4 раза
3. **Индексы БД** — 4 раза
4. **Select** — 3 раза
5. **Транзакции / ACID** — 3 раза
6. **Слайсы** — 2 раза
7. **Defer** — 2 раза
8. **Nil interface trap** — 1 раз (но почти всегда спрашивают)

---

## Быстрая подготовка к скринингу (1-2 часа)

1. Аксиомы каналов — выучить наизусть
2. GMP модель — уметь объяснить за 2 минуты
3. Nil interface ловушка — знать код наизусть
4. B-Tree индекс + составные индексы
5. ACID + уровни изоляции (READ COMMITTED vs REPEATABLE READ)
6. Worker Pool — уметь написать за 5 минут

## Быстрая подготовка к full-loop (4-6 часов)

**Go internals:** все файлы go-internals/ + concurrency/

**System Design:** `01_fundamentals.md` → `06_cases_feed_chat.md` (разобрать хотя бы 2 кейса)

**Databases:** `01_indexes_transactions.md` + `02_postgresql.md` + `04_kafka.md`

**Вопросы для самопроверки перед собесом:**
- Что такое GMP? Что происходит при блокирующем syscall?
- Чем отличается буферизованный канал от небуферизованного?
- Как работает MVCC в PostgreSQL?
- В чём разница между REPEATABLE READ и SERIALIZABLE?
- Как спроектировать ленту новостей для 10M DAU?
