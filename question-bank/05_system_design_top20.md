# Топ 20 вопросов по System Design

---

**1. Как спроектировать URL shortener (bit.ly)?**
> Генерация короткого кода (base62 от ID или хэш). БД: id, short_code, original_url, created_at. Кеш (Redis) для hot links. 301 (кешируется браузером) vs 302 (всегда через сервер, для аналитики). При высокой нагрузке — шардирование по short_code.

**2. Что такое CAP теорема?**
> В распределённой системе при network partition нельзя одновременно гарантировать Consistency (все узлы видят одни данные) и Availability (каждый запрос получает ответ). Нужно выбирать: CP (PostgreSQL, HBase) или AP (Cassandra, DynamoDB).

**3. Что такое горизонтальное и вертикальное масштабирование?**
> **Вертикальное (scale up)**: более мощный сервер. Простое, но есть предел и downtime. **Горизонтальное (scale out)**: больше серверов. Сложнее (нужна stateless архитектура, балансировка), но без предела.

**4. Что такое шардирование БД?**
> Разделение данных между несколькими БД-инстансами по ключу (user_id % N, range by date, consistent hashing). Решает проблему роста данных. Сложность: cross-shard запросы, rebalancing при добавлении шарда.

**5. Что такое consistent hashing?**
> Алгоритм распределения данных по узлам. При добавлении/удалении узла перемещается минимум данных (1/N). Используется в Cassandra, Redis Cluster, CDN. Виртуальные узлы (vnodes) для равномерности.

**6. Как спроектировать систему кеширования?**
> Cache-aside (приложение управляет), Write-through (пишем в кеш и БД), Write-behind (пишем в кеш, асинхронно в БД). Инвалидация: TTL, event-based. Проблемы: cache stampede (singleflight), cold start, thundering herd.

**7. Что такое eventual consistency?**
> Данные в разных узлах расходятся на короткое время, но в итоге сходятся. Трейдофф: доступность выше, сложнее программировать. Примеры: DNS propagation, лайки в соцсети, корзина в e-commerce.

**8. Как обеспечить идемпотентность API?**
> Idempotency-Key в заголовке запроса. Сервер хранит результат по ключу (Redis, БД). Повторный запрос возвращает кешированный результат. Критично для платёжных систем.

**9. Что такое Outbox паттерн?**
> Запись событий в outbox таблицу в той же транзакции что и бизнес-логика. Отдельный publisher читает outbox и отправляет в Kafka/RabbitMQ. Гарантирует at-least-once delivery без distributed transactions.

**10. Как спроектировать систему уведомлений?**
> Producer → Kafka → Consumer workers → (Push/Email/SMS). Разные топики по типу уведомлений. Retry с backoff. DLQ для failed. User preferences в Redis. Batch отправка email для экономии.

**11. Что такое CQRS?**
> Command Query Responsibility Segregation. Разделение модели записи (Commands) и чтения (Queries). Read модель денормализована для быстрого чтения. Write модель нормализована. Синхронизация через события.

**12. Что такое Event Sourcing?**
> Хранение не текущего состояния, а последовательности событий. Текущее состояние = применение всех событий. Полный audit log. Возможность replay. Сложность: eventual consistency, snapshot-ы для производительности.

**13. Как защититься от DDoS?**
> Rate limiting (IP, user), CDN (Cloudflare поглощает трафик), geo-blocking, CAPTCHA, анализ поведения (аномалии), scrubbing centers. На уровне кода: circuit breaker, graceful degradation.

**14. Что такое Service Mesh?**
> Инфраструктурный слой для service-to-service communication. Istio/Linkerd. Функции: mTLS, load balancing, tracing, circuit breaking, retry — без изменения кода приложения. Sidecar proxy (Envoy) рядом с каждым pod.

**15. Как спроектировать distributed lock?**
> Redis: `SET key value NX PX 30000` (NX — только если не существует, PX — TTL). Redlock — консенсус большинства Redis инстансов. Альтернатива: ZooKeeper, etcd (lease-based).

**16. Что такое saga паттерн?**
> Управление распределёнными транзакциями через цепочку локальных транзакций с компенсирующими операциями. Choreography (события) vs Orchestration (координатор). Вместо 2PC (distributed transaction).

**17. Как масштабировать чат приложение?**
> WebSocket соединения на dedicated серверах (Chat Service). Message pub/sub через Redis/Kafka между серверами. История сообщений в Cassandra (partitioned by chat_id). Горизонтальное масштабирование Chat Service за балансировщиком с sticky sessions или без.

**18. Что такое back pressure?**
> Механизм когда consumer сигнализирует producer что не успевает обрабатывать. Предотвращает OOM. В Kafka: consumer lag мониторинг. В Go: буферизованные каналы, `select` с default.

**19. Как обеспечить zero-downtime deployment?**
> Blue-Green deployment (переключение между двумя окружениями). Rolling update (постепенная замена инстансов). Canary release (небольшой % трафика на новую версию). Feature flags. Database migrations — backward compatible.

**20. Что такое SLA, SLO, SLI?**
> **SLI** (Indicator) — метрика: latency p99, error rate. **SLO** (Objective) — цель: p99 < 200ms, error rate < 0.1%. **SLA** (Agreement) — юридически обязывающее соглашение с penalty при нарушении. SLO строже SLA.
