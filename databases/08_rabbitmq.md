# RabbitMQ: Архитектура и паттерны

---

## Лекция

### 1. Что такое RabbitMQ и зачем он нужен

RabbitMQ — брокер сообщений (message broker), реализующий протокол AMQP. Это посредник между сервисами: один сервис отправляет сообщение, другой — получает и обрабатывает.

**Проблемы, которые решает RabbitMQ:**
- Decoupling — отправитель не знает о получателе
- Буферизация при пиковых нагрузках
- Балансировка задач между воркерами
- Ретраи при ошибках обработки
- Роутинг сообщений по правилам

**Ключевое отличие от Kafka:**
> RabbitMQ — это **умный брокер** (smart broker, dumb consumer): логика роутинга в брокере, потребитель просто получает.
> Kafka — это **умный потребитель** (dumb broker, smart consumer): брокер просто хранит, потребитель сам управляет offset.

---

### 2. Архитектура RabbitMQ

```
Producer
   │
   ▼
[Exchange]  ←── routing rules (bindings)
   │
   ├──→ [Queue A] ──→ Consumer 1
   ├──→ [Queue B] ──→ Consumer 2
   └──→ [Queue C] ──→ Consumer 3
```

**Основные компоненты:**

**Producer** — отправляет сообщение в Exchange (не напрямую в очередь).

**Exchange** — получает сообщение и маршрутизирует в одну или несколько очередей по правилам (bindings).

**Binding** — правило связи Exchange → Queue. Может иметь routing key.

**Queue** — буфер, хранит сообщения до получения потребителем. Сообщение удаляется после ACK.

**Consumer** — читает сообщения из очереди и подтверждает обработку (ACK).

---

### 3. Типы Exchange

**Direct** — точное совпадение routing key:
```
Exchange (direct)
  binding: key="order.created" → Queue A
  binding: key="order.paid"    → Queue B

Producer отправляет key="order.created" → попадёт только в Queue A
```

**Fanout** — рассылка всем привязанным очередям (игнорирует routing key):
```
Exchange (fanout)
  → Queue A (notifications)
  → Queue B (analytics)
  → Queue C (audit-log)

Одно сообщение → все три очереди
```
Используется для pub/sub, broadcast событий.

**Topic** — паттерн с wildcards:
```
binding: "order.#"  → Queue A   # matches: order.created, order.paid, order.item.added
binding: "order.*"  → Queue B   # matches: order.created, order.paid (только один уровень)
binding: "#.error"  → Queue C   # matches: любой.error, order.payment.error

* — один уровень
# — ноль или более уровней
```

**Headers** — роутинг по заголовкам сообщения (редко используется):
```
binding: x-match=all, region=EU, type=order → Queue EU-orders
```

---

### 4. Гарантии доставки и ACK

**Consumer ACK** — потребитель подтверждает обработку:
```
Consumer получает сообщение
  ├── успешно обработал → ack()   → сообщение удаляется из очереди
  ├── ошибка, повторить → nack(requeue=true)  → сообщение возвращается в очередь
  └── яд (poison msg)  → nack(requeue=false) → уходит в Dead Letter Exchange
```

Если потребитель упал без ACK → RabbitMQ переотправит другому потребителю.

**Publisher Confirms** — подтверждение от брокера что сообщение принято:
```go
channel.Confirm(false)
// После публикации ждём confirm
confirm := <-channel.NotifyPublish(make(chan amqp.Confirmation, 1))
if !confirm.Ack {
    // сообщение не принято — повторить
}
```

**Persistent сообщения** — сохраняются на диск, переживают рестарт брокера:
```go
amqp.Publishing{
    DeliveryMode: amqp.Persistent, // 2
    Body:         body,
}
```
Очередь тоже должна быть `durable: true`.

---

### 5. Dead Letter Exchange (DLX)

Куда идут сообщения при:
- `nack(requeue=false)`
- Истёкшем TTL
- Переполненной очереди (если задан `x-max-length`)

```
Normal Queue ──(отказ/TTL)──→ Dead Letter Exchange ──→ DLQ (Dead Letter Queue)
```

Настройка:
```go
args := amqp.Table{
    "x-dead-letter-exchange": "dlx",
    "x-message-ttl":          30000, // 30 секунд
}
channel.QueueDeclare("orders", true, false, false, false, args)
```

Паттерн ретраев через DLX:
```
orders-queue → fail → DLX → retry-queue (TTL 5s) → orders-queue → fail → ...
                                                   (после N ретраев → dead-queue)
```

---

### 6. Prefetch (QoS)

Сколько сообщений брокер отдаёт потребителю без ACK:

```go
channel.Qos(
    prefetchCount, // максимум N сообщений без ACK
    0,             // prefetchSize (0 = не ограничиваем)
    false,         // global (false = per consumer)
)
```

**Почему важно:**
```
prefetch=1:  воркер получает по одному → равномерная нагрузка, низкий throughput
prefetch=100: воркер получает пачку → высокий throughput, но один быстрый воркер
              может забрать всё и тормозить
```

Оптимальное значение: 10–50 для большинства задач.

---

### 7. Паттерны использования

**Work Queue (Task Queue)** — балансировка задач между воркерами:
```
Producer → Queue → [Worker1, Worker2, Worker3]
```
Каждое сообщение обрабатывается ровно одним воркером. Идеально для фоновых задач.

**Pub/Sub** — Fanout Exchange:
```
Event Bus → Fanout Exchange → [Queue1, Queue2, Queue3]
```
Каждый подписчик получает копию сообщения.

**RPC через RabbitMQ:**
```
Client → request_queue (reply_to="amq.gen-XXX", correlation_id="123")
Server → обрабатывает → reply_to queue
Client → ждёт ответа с correlation_id="123"
```

**Priority Queue:**
```go
args := amqp.Table{
    "x-max-priority": 10, // приоритеты 0-10
}
// Сообщения с высоким priority обрабатываются первыми
amqp.Publishing{Priority: 8}
```

---

## RabbitMQ vs Kafka — детальное сравнение

| | RabbitMQ | Kafka |
|--|----------|-------|
| **Модель** | Push (брокер толкает к потребителю) | Pull (потребитель сам читает) |
| **Хранение** | Удаляет после ACK | Хранит N дней независимо от чтения |
| **Реплей** | Нет | Да, с любого offset |
| **Роутинг** | Гибкий (Exchange, bindings, wildcard) | Только по топику/партиции |
| **Порядок** | В рамках очереди | Внутри партиции |
| **Throughput** | Сотни тысяч msg/s | Миллионы msg/s |
| **Latency** | Низкая (мс) | Чуть выше (батчинг) |
| **Consumer groups** | Конкурирующие потребители на очереди | Consumer Groups с offset |
| **Масштабирование** | Вертикально + federation | Горизонтально (партиции) |
| **Сложность** | Проще в настройке | Требует ZooKeeper/KRaft, мониторинг |

### Когда выбрать RabbitMQ:
- Нужна сложная логика роутинга
- Задачи с ретраями, DLQ, приоритетами
- RPC паттерн
- Небольшой/средний объём сообщений
- Важна низкая latency
- Команда не хочет управлять Kafka кластером

### Когда выбрать Kafka:
- Нужен реплей событий (Event Sourcing, аудит)
- Много независимых потребителей одного потока
- Огромный throughput (IoT, логи, аналитика)
- Нужно хранить историю событий
- Stream processing (Kafka Streams, ksqlDB)

---

## Практические примеры

### Пример 1: Producer на Go

```go
import "github.com/rabbitmq/amqp091-go"

type OrderPublisher struct {
    conn    *amqp.Connection
    channel *amqp.Channel
}

func NewOrderPublisher(url string) (*OrderPublisher, error) {
    conn, err := amqp.Dial(url)
    if err != nil {
        return nil, fmt.Errorf("connect: %w", err)
    }

    ch, err := conn.Channel()
    if err != nil {
        return nil, fmt.Errorf("channel: %w", err)
    }

    // Объявляем exchange (идемпотентно)
    err = ch.ExchangeDeclare(
        "orders",  // name
        "topic",   // type
        true,      // durable
        false,     // auto-deleted
        false,     // internal
        false,     // no-wait
        nil,
    )
    if err != nil {
        return nil, err
    }

    // Включаем подтверждения от брокера
    ch.Confirm(false)

    return &OrderPublisher{conn: conn, channel: ch}, nil
}

func (p *OrderPublisher) Publish(ctx context.Context, routingKey string, order *Order) error {
    body, err := json.Marshal(order)
    if err != nil {
        return err
    }

    confirms := p.channel.NotifyPublish(make(chan amqp.Confirmation, 1))

    err = p.channel.PublishWithContext(ctx,
        "orders",   // exchange
        routingKey, // routing key: "order.created", "order.paid", etc.
        true,       // mandatory (вернуть если не доставлено в очередь)
        false,      // immediate
        amqp.Publishing{
            ContentType:  "application/json",
            DeliveryMode: amqp.Persistent, // переживёт рестарт
            MessageId:    uuid.New().String(),
            Timestamp:    time.Now(),
            Body:         body,
        },
    )
    if err != nil {
        return fmt.Errorf("publish: %w", err)
    }

    // Ждём подтверждения от брокера
    confirm := <-confirms
    if !confirm.Ack {
        return fmt.Errorf("broker nack: message not accepted")
    }

    return nil
}
```

### Пример 2: Consumer с ретраями и DLQ

```go
type OrderConsumer struct {
    channel *amqp.Channel
    handler func(order *Order) error
}

func (c *OrderConsumer) Setup() error {
    // Dead Letter Exchange
    c.channel.ExchangeDeclare("orders.dlx", "direct", true, false, false, false, nil)
    c.channel.QueueDeclare("orders.dead", true, false, false, false, nil)
    c.channel.QueueBind("orders.dead", "orders", "orders.dlx", false, nil)

    // Основная очередь с DLX
    args := amqp.Table{
        "x-dead-letter-exchange":    "orders.dlx",
        "x-dead-letter-routing-key": "orders",
        "x-message-ttl":             int32(60000), // 60 сек максимум в очереди
    }
    c.channel.QueueDeclare("orders.processing", true, false, false, false, args)
    c.channel.QueueBind("orders.processing", "order.*", "orders", false, nil)

    // Не более 10 сообщений без ACK
    c.channel.Qos(10, 0, false)

    return nil
}

func (c *OrderConsumer) Consume(ctx context.Context) error {
    msgs, err := c.channel.Consume(
        "orders.processing",
        "",    // consumer tag (auto)
        false, // auto-ack = false (ручной ACK)
        false, // exclusive
        false, false, nil,
    )
    if err != nil {
        return err
    }

    for {
        select {
        case msg, ok := <-msgs:
            if !ok {
                return fmt.Errorf("channel closed")
            }

            var order Order
            if err := json.Unmarshal(msg.Body, &order); err != nil {
                // Невалидное сообщение — в DLQ, не повторять
                msg.Nack(false, false)
                continue
            }

            if err := c.handler(&order); err != nil {
                log.Printf("process error: %v, retrying", err)
                // Вернуть в очередь для повтора
                msg.Nack(false, true)
                continue
            }

            msg.Ack(false) // успешно обработали

        case <-ctx.Done():
            return ctx.Err()
        }
    }
}
```

### Пример 3: Fanout для событий

```go
// Publisher: событие идёт всем подписчикам
ch.ExchangeDeclare("user.events", "fanout", true, false, false, false, nil)
ch.PublishWithContext(ctx, "user.events", "", false, false, amqp.Publishing{
    Body: mustMarshal(UserRegisteredEvent{UserID: 42}),
})

// Каждый сервис объявляет свою очередь и биндится к fanout exchange
// notifications-service:
ch.QueueDeclare("notifications.user-registered", true, false, false, false, nil)
ch.QueueBind("notifications.user-registered", "", "user.events", false, nil)

// analytics-service:
ch.QueueDeclare("analytics.user-registered", true, false, false, false, nil)
ch.QueueBind("analytics.user-registered", "", "user.events", false, nil)
```

---

## Вопросы на собеседовании

### Q: В чём разница между Exchange типов Direct, Fanout и Topic?

**Ответ:** Direct — точное совпадение routing key, сообщение идёт в конкретную очередь. Fanout — игнорирует routing key, рассылает копию всем привязанным очередям (pub/sub). Topic — паттерн с wildcards: `*` один уровень, `#` — ноль или более. Позволяет гибко маршрутизировать: `order.#` поймает все события заказа.

---

### Q: Что такое ACK и зачем он нужен?

**Ответ:** ACK (acknowledgment) — подтверждение потребителя что сообщение обработано. Без ACK RabbitMQ не удаляет сообщение из очереди. Если потребитель упал без ACK — сообщение переотправляется другому потребителю. NACK с `requeue=true` возвращает в очередь, `requeue=false` — отправляет в Dead Letter Exchange. Это основа гарантии at-least-once доставки.

---

### Q: Что такое Dead Letter Queue? Для чего используется?

**Ответ:** DLQ — очередь для "мёртвых" сообщений: тех, что не удалось обработать (nack без requeue), истёк TTL или очередь переполнена. Сообщения идут через Dead Letter Exchange в DLQ. Используется для: инспекции проблемных сообщений, ретраев с экспоненциальной задержкой (TTL + DLX цепочка), алертов при накоплении.

---

### Q: RabbitMQ или Kafka — что выбрать?

**Ответ:** RabbitMQ — когда нужен гибкий роутинг, ретраи, приоритеты, RPC, низкая latency и средний объём. Kafka — когда нужен реплей событий, огромный throughput, много независимых потребителей одного потока, хранение истории. Главное отличие: Kafka хранит сообщения независимо от потребления (можно перечитать), RabbitMQ удаляет после ACK.

---

### Q: Что такое prefetch и как его настроить?

**Ответ:** Prefetch (QoS) — количество сообщений, которые брокер отдаёт потребителю без получения ACK. `prefetch=1` — равномерная нагрузка, но низкий throughput. Большой prefetch — высокий throughput, но один медленный воркер может "застрять" с пачкой необработанных сообщений. Оптимум — 10–50 в зависимости от времени обработки одного сообщения.
