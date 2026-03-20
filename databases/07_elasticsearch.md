# Elasticsearch: Поиск и аналитика

---

## Лекция

### 1. Что такое Elasticsearch

Elasticsearch — распределённый поисковый и аналитический движок на основе Apache Lucene. Предназначен для:
- Полнотекстового поиска
- Структурированного поиска (фильтры, агрегации)
- Логов и метрик (ELK Stack: Elasticsearch + Logstash + Kibana)
- Геопространственного поиска

**Ключевые отличия от реляционных БД:**
- Хранит документы в JSON (schema-free)
- Оптимизирован для полнотекстового поиска
- Near Real-Time (NRT) — изменения видны через ~1 секунду
- Горизонтально масштабируется из коробки

---

### 2. Архитектура: кластер, индекс, шарды

```
Cluster: my-cluster
    |
    ├── Node 1 (Master + Data)
    │     ├── Shard 0 (Primary)
    │     └── Shard 2 (Replica of 2)
    |
    ├── Node 2 (Data)
    │     ├── Shard 1 (Primary)
    │     └── Shard 0 (Replica of 0)
    |
    └── Node 3 (Data)
          ├── Shard 2 (Primary)
          └── Shard 1 (Replica of 1)
```

**Index** — логическое пространство для документов (аналог таблицы в SQL).

**Document** — единица хранения: JSON объект с уникальным `_id`.

**Shard** — физическая единица хранения: отдельный экземпляр Lucene. Primary shard + replica shards.

**Mapping** — схема документа (аналог CREATE TABLE): типы полей, анализаторы.

---

### 3. Как работает полнотекстовый поиск

**Инвертированный индекс:**
```
Текст: "Быстрая рыжая лиса"
       "Лиса перепрыгнула через забор"

Инвертированный индекс:
"быстр" → [doc1]
"рыж"   → [doc1]
"лис"   → [doc1, doc2]
"перепрыгн" → [doc2]
"забор" → [doc2]
```

**Анализ текста (Analysis):**
1. **Tokenization** — разбить текст на токены: "Быстрая рыжая лиса" → ["Быстрая", "рыжая", "лиса"]
2. **Normalization** — привести к нижнему регистру: ["быстрая", "рыжая", "лиса"]
3. **Stemming** — привести к основе: ["быстр", "рыж", "лис"]
4. **Stop words** — убрать стоп-слова: ["и", "в", "с", "на"]

**Relevance Scoring (BM25):**
- Чем чаще термин в документе (TF) — тем выше релевантность
- Чем реже термин встречается во всех документах (IDF) — тем важнее
- `score = sum(IDF * TF_norm)`

---

### 4. Типы запросов

**Term query** — точное совпадение (не анализируется):
```json
{"term": {"status": "active"}}
```

**Match query** — полнотекстовый поиск (анализируется):
```json
{"match": {"description": "быстрый поиск"}}
```

**Match phrase** — поиск фразы (слова рядом):
```json
{"match_phrase": {"description": "быстрый поиск"}}
```

**Bool query** — комбинация условий:
```json
{
  "bool": {
    "must": [{"match": {"title": "elasticsearch"}}],
    "filter": [{"term": {"status": "published"}}],
    "must_not": [{"term": {"draft": true}}],
    "should": [{"term": {"featured": true}}]
  }
}
```
- `must` — влияет на score + обязательно
- `filter` — НЕ влияет на score, кешируется → быстро
- `must_not` — исключить
- `should` — повышает score, не обязательно

**Range query:**
```json
{"range": {"created_at": {"gte": "2024-01-01", "lt": "2024-02-01"}}}
```

---

### 5. Маппинг и анализаторы

```json
PUT /products
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "russian",
        "fields": {
          "keyword": {"type": "keyword"}
        }
      },
      "price": {"type": "double"},
      "tags": {"type": "keyword"},
      "created_at": {"type": "date"},
      "location": {"type": "geo_point"}
    }
  }
}
```

**text vs keyword:**
- `text` — анализируется, используется для полнотекстового поиска
- `keyword` — не анализируется, для точных совпадений, агрегаций, сортировки

**Multi-field:** одно поле с двумя типами:
```json
"title": {
  "type": "text",
  "fields": {
    "keyword": {"type": "keyword", "ignore_above": 256}
  }
}
```
Поиск: `title`, Агрегация/сортировка: `title.keyword`

---

### 6. Агрегации

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_status": {
      "terms": {"field": "status"},
      "aggs": {
        "total_amount": {"sum": {"field": "amount"}},
        "avg_amount": {"avg": {"field": "amount"}}
      }
    },
    "orders_over_time": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "day"
      }
    }
  }
}
```

---

### 7. Персистентность и репликация

**Refresh** — создание нового сегмента Lucene из буфера, индекс становится searchable. По умолчанию каждую секунду. NRT (Near Real-Time).

**Flush** — запись сегментов на диск + очистка translog.

**Translog** — журнал операций (как WAL). Обеспечивает восстановление после сбоя.

**Репликация:** Primary shard получает запрос → выполняет → параллельно реплицирует на replica shards.

---

## Практические примеры

### Пример 1: Elasticsearch с Go (olivere/elastic)

```go
import "github.com/olivere/elastic/v7"

type ProductRepository struct {
    client *elastic.Client
    index  string
}

type Product struct {
    ID          string    `json:"id"`
    Title       string    `json:"title"`
    Description string    `json:"description"`
    Price       float64   `json:"price"`
    Tags        []string  `json:"tags"`
    CreatedAt   time.Time `json:"created_at"`
}

func (r *ProductRepository) Search(ctx context.Context, query string, filters map[string]string) ([]Product, error) {
    // Bool query: полнотекстовый поиск + фильтры
    boolQuery := elastic.NewBoolQuery().
        Must(elastic.NewMultiMatchQuery(query, "title^2", "description")). // title приоритетнее
        Filter(elastic.NewTermQuery("status", "active"))

    for field, value := range filters {
        boolQuery.Filter(elastic.NewTermQuery(field, value))
    }

    result, err := r.client.Search().
        Index(r.index).
        Query(boolQuery).
        Sort("_score", false).
        From(0).Size(20).
        Do(ctx)
    if err != nil {
        return nil, err
    }

    var products []Product
    for _, hit := range result.Hits.Hits {
        var p Product
        json.Unmarshal(hit.Source, &p)
        p.ID = hit.Id
        products = append(products, p)
    }
    return products, nil
}
```

### Пример 2: Индексация документа

```go
func (r *ProductRepository) Index(ctx context.Context, product *Product) error {
    _, err := r.client.Index().
        Index(r.index).
        Id(product.ID).
        BodyJson(product).
        Do(ctx)
    return err
}

// Batch индексация через Bulk API
func (r *ProductRepository) BulkIndex(ctx context.Context, products []Product) error {
    bulk := r.client.Bulk()
    for _, p := range products {
        req := elastic.NewBulkIndexRequest().
            Index(r.index).
            Id(p.ID).
            Doc(p)
        bulk.Add(req)
    }
    _, err := bulk.Do(ctx)
    return err
}
```

### Пример 3: Автодополнение (completion suggester)

```json
PUT /products
{
  "mappings": {
    "properties": {
      "title": {"type": "text"},
      "suggest": {
        "type": "completion",
        "analyzer": "simple"
      }
    }
  }
}

GET /products/_search
{
  "suggest": {
    "product_suggest": {
      "prefix": "iphon",
      "completion": {
        "field": "suggest",
        "size": 5,
        "fuzzy": {"fuzziness": 1}
      }
    }
  }
}
```

---

## Вопросы на собеседовании

### Q: Как работает полнотекстовый поиск в Elasticsearch?

**Ответ:** При индексации текст проходит через анализатор: токенизация → нормализация (нижний регистр) → стемминг (основа слова) → удаление стоп-слов. Результат записывается в инвертированный индекс: слово → список документов. При поиске запрос проходит тот же анализатор, затем ищутся совпадающие документы. Релевантность рассчитывается по BM25: учитывает частоту термина в документе (TF) и редкость термина во всей коллекции (IDF).

---

### Q: Чем filter отличается от must в bool query?

**Ответ:** `must` — условие влияет на score (relevance) документа. `filter` — условие не влияет на score (не учитывается при ранжировании) и результат кешируется битсетом. Для точных фильтров (status = "active", price < 1000, date range) всегда используй `filter` — это быстрее и кешируется. `must` используй только для полнотекстового поиска, где нужен relevance score.

---

### Q: Что такое шарды и реплики? Как выбрать количество?

**Ответ:** Шард — физическая единица хранения (Lucene индекс). Данные индекса разбиваются по шардам. Реплика — копия шарда для отказоустойчивости и масштабирования чтения. Количество primary shards фиксируется при создании индекса (нельзя изменить без reindex). Ориентир: shard размером 10-50GB. Если ожидаешь 100GB данных → 3-5 shards. Реплики: минимум 1 (для отказоустойчивости). Больше реплик → лучше чтение, хуже запись.

---

### Q: Что такое NRT (Near Real-Time)? Почему не Real-Time?

**Ответ:** При индексации документ сначала попадает в буфер памяти и translog (сразу доступен для восстановления). Раз в секунду (refresh interval) Elasticsearch создаёт новый сегмент Lucene из буфера — только тогда документ становится searchable. Это NRT: задержка ~1 секунда. Можно сделать `refresh_interval: -1` для bulk-загрузки (быстрее) и вручную делать `POST /index/_refresh`. Real-Time невозможен из-за архитектуры Lucene сегментов.

---

### Q: Когда использовать Elasticsearch вместо PostgreSQL для поиска?

**Ответ:** PostgreSQL достаточен для простого поиска (LIKE, tsvector) если данных < 1-10M документов, нужна консистентность, поиск — вспомогательная функция. Elasticsearch нужен если: сложный полнотекстовый поиск (релевантность, подсветка, fuzzy), автодополнение, фасетная навигация, > 10M документов с высокой нагрузкой на поиск, нужен анализ логов и метрик. Минус ES: eventual consistency, сложнее операционно, нет ACID транзакций.
