# Apache Kafka: Архитектура и паттерны

---

## Лекция

### 1. Что такое Kafka и зачем она нужна

Kafka — распределённая платформа для потоковой обработки данных. Это персистентная очередь сообщений с возможностью многократного чтения.

**Проблемы, которые решает Kafka:**
- Связь между микросервисами без прямых зависимостей (decoupling)
- Буферизация при пиковых нагрузках
- Доставка событий многим получателям
- Реплей событий (вернуться назад и переобработать)
- Event sourcing

**Kafka vs обычная очередь (RabbitMQ):**
| | Kafka | RabbitMQ |
|--|-------|---------|
| Хранение | Персистентное (дни/недели) | Удаляется после чтения |
| Реплей | Да, можно читать с любого offset | Нет |
| Потребители | Много независимых групп | Одно сообщение → один потребитель |
| Гарантии порядка | Внутри партиции | В рамках очереди |
| Throughput | Очень высокий (млн сообщений/сек) | Ниже |

---

### 2. Архитектура Kafka

```
Producers                 Kafka Cluster              Consumers
                     ┌──────────────────────┐
[Service A] ──────→  │  Topic: orders       │  ──→ [Consumer Group: billing]
[Service B] ──────→  │  ┌──────────────────┐│  ──→ [Consumer Group: notifications]
                     │  │ Partition 0      ││
                     │  │ [msg0][msg1][msg2]│|
                     │  │ Partition 1      ││
                     │  │ [msg0][msg1]     ││
                     │  └──────────────────┘│
                     └──────────────────────┘
```

**Topic** — именованный поток сообщений (как таблица в БД).

**Partition** — физическое разбиение топика. Каждая партиция — упорядоченный, неизменяемый лог. Партиции позволяют параллельную обработку.

**Broker** — сервер Kafka. Кластер = несколько брокеров.

**Offset** — порядковый номер сообщения в партиции. Потребитель сам управляет своим offset.

**Producer** — публикует сообщения в топик. Сам выбирает партицию (по ключу или round-robin).

**Consumer** — читает сообщения из партиций.

**Consumer Group** — группа потребителей, разделяющих нагрузку. Каждая партиция читается только одним потребителем из группы. Разные группы читают независимо.

---

### 3. Как работает запись сообщений

**Определение партиции:**
```
if key != nil:
    partition = hash(key) % numPartitions
else:
    partition = round_robin  // или sticky partition
```

**Важность ключа:** Сообщения с одинаковым ключом всегда попадают в одну партицию → гарантированный порядок для этого ключа.

```
key="user:42" → всегда Partition 0
key="user:99" → всегда Partition 1
key=nil       → Partition 0, 1, 2... по очереди
```

**Batch и compression:**
- Producer накапливает сообщения в batch (`linger.ms`, `batch.size`)
- Компрессия целого batch: snappy, lz4, gzip, zstd
- Один I/O вызов для batch → высокий throughput

---

### 4. Гарантии доставки

**Producer acks:**
| `acks` | Поведение | Надёжность |
|--------|-----------|-----------|
| `0` | Не ждать подтверждения | Возможна потеря |
| `1` | Ждать подтверждения лидер-брокера | Потеря при падении лидера до репликации |
| `all`/`-1` | Ждать подтверждения всех ISR реплик | Наиболее надёжно |

**ISR (In-Sync Replicas)** — реплики, синхронизированные с лидером.

**Semantics:**
- **At most once** — можно потерять сообщение, но не обработать дважды. Acks=0.
- **At least once** — можно обработать дважды, но не потерять. Acks=all + ретрай.
- **Exactly once** — строго один раз. Требует idempotent producer + транзакции.

```
# Idempotent producer (Kafka 0.11+)
enable.idempotence = true
# Producer ID + sequence number → брокер дедуплицирует ретраи
```

---

### 5. Consumer Group и балансировка

```
Topic: orders (3 партиции)
Consumer Group: billing (2 consumers)

Consumer 1: Partition 0, Partition 1
Consumer 2: Partition 2

Правило: потребителей <= партиций
         лишние потребители простаивают
```

**Rebalance** — перераспределение партиций при:
- Добавлении/удалении потребителя
- Падении потребителя (сессия истекла, нет heartbeat)

**Commit offset:**
```
auto.commit.enable = true  // автоматически каждые auto.commit.interval.ms
auto.commit.enable = false // ручной commit после обработки
```

Ручной commit (at-least-once):
```go
messages := consumer.Poll()
process(messages)
consumer.Commit() // commit только после успешной обработки
```

---

### 6. Retention и Compaction

**Retention (по времени или размеру):**
```
retention.ms = 604800000   // 7 дней (по умолчанию)
retention.bytes = 1073741824 // 1GB
```
Старые сегменты удаляются. Подходит для событийных потоков.

**Log Compaction:**
```
cleanup.policy = compact
```
Kafka хранит только последнее сообщение для каждого ключа. Сообщение с `value=null` — tombstone (удаление). Подходит для хранения актуального состояния (как key-value store).

---

### 7. Kafka Streams и ksqlDB

**Kafka Streams** — библиотека для обработки потоков прямо в Kafka.
- Stateless операции: filter, map, flatMap
- Stateful: groupBy, aggregate, join (хранит state в RocksDB)
- Windowing: tumbling, hopping, session windows

**ksqlDB** — SQL интерфейс для потоковой обработки:
```sql
CREATE STREAM orders_stream AS
SELECT * FROM orders_raw
WHERE amount > 1000;

CREATE TABLE user_total_spend AS
SELECT user_id, SUM(amount) as total
FROM orders_stream
GROUP BY user_id;
```

---

## Практические примеры

### Пример 1: Producer на Go (sarama)

```go
import "github.com/IBM/sarama"

type OrderProducer struct {
    producer sarama.SyncProducer
}

func NewOrderProducer(brokers []string) (*OrderProducer, error) {
    config := sarama.NewConfig()
    config.Producer.RequiredAcks = sarama.WaitForAll // acks=all
    config.Producer.Retry.Max = 5
    config.Producer.Return.Successes = true
    config.Producer.Compression = sarama.CompressionSnappy
    config.Producer.Idempotent = true // exactly-once на уровне producer
    config.Net.MaxOpenRequests = 1    // требуется для idempotent

    producer, err := sarama.NewSyncProducer(brokers, config)
    if err != nil {
        return nil, err
    }
    return &OrderProducer{producer: producer}, nil
}

func (p *OrderProducer) PublishOrder(ctx context.Context, order *Order) error {
    value, err := json.Marshal(order)
    if err != nil {
        return err
    }

    msg := &sarama.ProducerMessage{
        Topic: "orders",
        Key:   sarama.StringEncoder(fmt.Sprintf("user:%d", order.UserID)), // ключ = порядок
        Value: sarama.ByteEncoder(value),
        Headers: []sarama.RecordHeader{
            {Key: []byte("source"), Value: []byte("order-service")},
        },
    }

    partition, offset, err := p.producer.SendMessage(msg)
    if err != nil {
        return fmt.Errorf("send message: %w", err)
    }
    log.Printf("order %d published: partition=%d offset=%d", order.ID, partition, offset)
    return nil
}
```

### Пример 2: Consumer Group на Go

```go
type OrderConsumer struct {
    consumer sarama.ConsumerGroup
    handler  OrderHandler
}

// ConsumerGroupHandler реализует sarama.ConsumerGroupHandler
type orderGroupHandler struct {
    handler OrderHandler
}

func (h *orderGroupHandler) Setup(sess sarama.ConsumerGroupSession) error {
    log.Printf("Consumer group setup: partitions=%v", sess.Claims())
    return nil
}

func (h *orderGroupHandler) Cleanup(sess sarama.ConsumerGroupSession) error {
    return nil
}

func (h *orderGroupHandler) ConsumeClaim(
    sess sarama.ConsumerGroupSession,
    claim sarama.ConsumerGroupClaim,
) error {
    for {
        select {
        case msg, ok := <-claim.Messages():
            if !ok {
                return nil
            }

            var order Order
            if err := json.Unmarshal(msg.Value, &order); err != nil {
                log.Printf("failed to unmarshal order: %v", err)
                // Не коммитим — сообщение не будет переобработано если это баг
                // Лучше: отправить в dead-letter queue
                sess.MarkMessage(msg, "") // пропустить
                continue
            }

            if err := h.handler.ProcessOrder(sess.Context(), &order); err != nil {
                log.Printf("failed to process order %d: %v", order.ID, err)
                // Не коммитим → Kafka повторит доставку
                return err // перезапустим consume loop
            }

            sess.MarkMessage(msg, "") // помечаем как обработанное
            // MarkMessage не делает commit сразу, commit происходит периодически

        case <-sess.Context().Done():
            return nil
        }
    }
}

func (c *OrderConsumer) Start(ctx context.Context) error {
    handler := &orderGroupHandler{handler: c.handler}
    for {
        if err := c.consumer.Consume(ctx, []string{"orders"}, handler); err != nil {
            return err
        }
        if ctx.Err() != nil {
            return ctx.Err()
        }
        // Rebalance → снова вызываем Consume
    }
}
```

### Пример 3: Outbox Pattern (гарантированная доставка)

```go
// Проблема: как атомарно сохранить заказ в БД И отправить событие в Kafka?
// Решение: Outbox Pattern

// 1. В одной транзакции: создать заказ + записать событие в таблицу outbox
func (s *OrderService) CreateOrder(ctx context.Context, order *Order) error {
    return s.db.WithTransaction(ctx, func(tx *sql.Tx) error {
        // Создаём заказ
        if err := insertOrder(tx, order); err != nil {
            return err
        }

        // Записываем событие в outbox (в той же транзакции)
        event := OutboxEvent{
            Topic:   "orders",
            Key:     fmt.Sprintf("user:%d", order.UserID),
            Payload: mustMarshal(order),
        }
        return insertOutboxEvent(tx, event)
    })
}

// 2. Отдельный relay процесс: читает outbox → публикует в Kafka → помечает как отправленное
func (r *OutboxRelay) Run(ctx context.Context) {
    ticker := time.NewTicker(100 * time.Millisecond)
    for {
        select {
        case <-ticker.C:
            events, _ := r.db.GetPendingEvents(ctx, 100)
            for _, event := range events {
                r.kafka.Publish(event)
                r.db.MarkSent(ctx, event.ID)
            }
        case <-ctx.Done():
            return
        }
    }
}
```

---

## Вопросы на собеседовании

### Q: Объясни архитектуру Kafka. Что такое топик, партиция, offset?

**Ответ:** Topic — именованный поток сообщений (логическая единица). Partition — физическое разбиение топика: упорядоченный неизменяемый лог сообщений. Партиции позволяют параллельную обработку и хранятся на разных брокерах. Offset — порядковый номер сообщения в партиции (начиная с 0). Потребители хранят свой текущий offset — могут читать с любого места, перечитывать сообщения.

---

### Q: Как Kafka гарантирует порядок сообщений?

**Ответ:** Порядок гарантирован только внутри одной партиции. Если нужен порядок для связанных событий (все заказы одного пользователя) — использовать ключ партиции: `key = "user:42"`. Все сообщения с одинаковым ключом всегда попадают в одну партицию через `hash(key) % numPartitions`. Порядок между разными партициями не гарантирован.

---

### Q: At-least-once vs exactly-once. Как достичь exactly-once?

**Ответ:** At-least-once: `acks=all` + ретраи при ошибке. Гарантирует доставку, но при ретрае возможна дублирование. Exactly-once: 1) Idempotent producer (`enable.idempotence=true`) — дедупликация ретраев на уровне брокера. 2) Transactional API — атомарная запись в несколько партиций + commit offset потребителя. 3) На уровне потребителя — идемпотентная обработка (проверка ID перед обработкой). Чаще всего используют at-least-once + идемпотентность в бизнес-логике.

---

### Q: Что такое Consumer Group? Как работает rebalance?

**Ответ:** Consumer Group — группа потребителей, совместно читающих топик. Каждая партиция читается только одним потребителем из группы. Разные группы читают независимо (свой offset). Rebalance — перераспределение партиций при добавлении/удалении потребителя или при отсутствии heartbeat. Во время rebalance всё чтение останавливается (stop-the-world). Cooperative Rebalance (Kafka 2.4+) позволяет переназначать только нужные партиции.

---

### Q: Что такое Outbox Pattern? Зачем нужен?

**Ответ:** Outbox Pattern решает проблему двойной записи: нужно атомарно и сохранить данные в БД, и отправить событие в Kafka. Наивное решение: сначала БД, потом Kafka — если Kafka упадёт, событие потеряется. Outbox: событие записывается в таблицу outbox в одной транзакции с основными данными. Отдельный relay-процесс (или Debezium CDC) читает outbox и публикует в Kafka, потом помечает как отправленное. Гарантирует exactly-once с точки зрения бизнес-логики.

---

### Q: Как Kafka обеспечивает высокую производительность?

**Ответ:** 1) Sequential disk writes — данные дописываются в конец файла, а не модифицируются. SSD/HDD оптимизированы для последовательного I/O. 2) Zero-copy — при отправке данных клиенту использует `sendfile()` syscall: данные из файлового дескриптора напрямую в сетевой буфер, минуя userspace. 3) Batching — producer и consumer работают с пачками сообщений. 4) Compression — сжатие целых batch. 5) Page cache — ОС кеширует файлы в RAM, горячие данные читаются из памяти.
