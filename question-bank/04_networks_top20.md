# Топ 20 вопросов по сетям

---

**1. Что происходит при TCP handshake?**
> 3-way handshake: (1) Client → Server: SYN. (2) Server → Client: SYN-ACK. (3) Client → Server: ACK. После этого соединение установлено. Закрытие: 4-way (FIN, FIN-ACK, FIN, FIN-ACK).

**2. Чем TCP отличается от UDP?**
> TCP: гарантия доставки, порядок, flow control, congestion control, соединение-ориентированный. UDP: нет гарантий, нет порядка, быстрее, connectionless. UDP для: стриминга, игр, DNS, VoIP.

**3. Чем HTTP/1.1 отличается от HTTP/2?**
> HTTP/2: мультиплексирование (несколько запросов в одном TCP соединении), бинарный протокол, сжатие заголовков (HPACK), server push. HTTP/1.1: текстовый, один запрос за раз (или несколько соединений).

**4. Что такое gRPC и как он работает?**
> RPC фреймворк от Google. Транспорт — HTTP/2. Сериализация — Protocol Buffers (бинарный, компактный). Контракт в `.proto` файлах → кодогенерация. Поддерживает unary, server/client/bidirectional streaming.

**5. Как работает HTTPS?**
> TLS поверх TCP. TLS handshake: обмен сертификатами, согласование cipher suite, обмен ключами (ECDH), генерация session key. Весь HTTP трафик шифруется симметричным ключом.

**6. Что такое HTTP/3?**
> HTTP поверх QUIC (UDP-based). Устраняет head-of-line blocking TCP. Быстрее восстанавливается после потери пакетов. 0-RTT соединения. Используется в браузерах и облачных сервисах.

**7. Что такое REST и основные принципы?**
> Stateless, клиент-серверная архитектура, кешируемость, единый интерфейс (ресурсы + HTTP методы), слоистость. HTTP методы: GET (идемпотентный, безопасный), POST, PUT (идемпотентный), PATCH, DELETE (идемпотентный).

**8. Чем REST отличается от gRPC?**
> REST: текстовый (JSON), человекочитаемый, стандартный HTTP/1.1, слабая типизация. gRPC: бинарный (protobuf), строгая типизация, кодогенерация, HTTP/2, streaming из коробки. gRPC быстрее на ~7x по сериализации.

**9. Что такое WebSocket?**
> Полнодуплексное соединение поверх TCP. Начинается с HTTP upgrade запроса. После установки — двустороннее общение без overhead HTTP заголовков. Для realtime: чаты, live обновления.

**10. Что такое DNS и как работает резолвинг?**
> Distributed система преобразования доменов в IP. Запрос → local cache → OS cache → recursive resolver → root nameserver → TLD nameserver → authoritative nameserver. TTL определяет время кеширования.

**11. Что такое load balancer и типы балансировки?**
> Распределяет трафик между серверами. Алгоритмы: Round Robin, Weighted Round Robin, Least Connections, IP Hash (sticky sessions), Random. L4 (TCP) vs L7 (HTTP — может смотреть в заголовки).

**12. Что такое reverse proxy?**
> Сервер перед backend. Принимает запросы от клиентов, проксирует на backend. Функции: TLS termination, load balancing, кеширование, аутентификация. Примеры: Nginx, Envoy.

**13. Что такое keep-alive в HTTP?**
> HTTP/1.1: постоянное TCP соединение для нескольких запросов (Connection: keep-alive). Без него — новый TCP handshake на каждый запрос. HTTP/2: всегда одно соединение с мультиплексированием.

**14. Что такое CORS?**
> Cross-Origin Resource Sharing. Браузер запрещает JS делать запросы на другой домен. Сервер разрешает через заголовки: `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`. Preflight (OPTIONS) для non-simple запросов.

**15. Что такое rate limiting и как реализовать?**
> Ограничение количества запросов. Алгоритмы: Token Bucket (пополняется со временем), Sliding Window (счётчик за скользящий период), Fixed Window. В Go: `golang.org/x/time/rate`.

**16. Что такое circuit breaker паттерн?**
> Защита от каскадных сбоев. Состояния: Closed (работает), Open (не пропускает запросы), Half-Open (проверяет восстановление). При N ошибках → Open. Через timeout → Half-Open.

**17. В чём разница между HTTP 401 и 403?**
> 401 Unauthorized — не аутентифицирован (нет/невалидный токен). 403 Forbidden — аутентифицирован, но нет прав доступа.

**18. Что такое mTLS?**
> Mutual TLS — двустороннее TLS. Не только сервер предъявляет сертификат, но и клиент. Используется для service-to-service аутентификации в микросервисах (Istio, service mesh).

**19. Что такое long polling vs SSE vs WebSocket?**
> **Long Polling**: клиент делает запрос, сервер держит его открытым до события. **SSE (Server-Sent Events)**: однонаправленный поток от сервера (EventSource API). **WebSocket**: двунаправленный поток. SSE проще для push-уведомлений.

**20. Что такое HEAD-of-line blocking в HTTP/1.1?**
> В одном TCP соединении запросы обрабатываются последовательно. Медленный запрос блокирует все последующие. HTTP/2 решает мультиплексированием. HTTP/3 решает переходом на QUIC/UDP.
