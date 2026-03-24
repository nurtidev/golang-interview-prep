# Сети: NAT, BGP, Мобильные операторы, Wireshark

---

## 1. Как работает Интернет: Big Picture

```
Твой ноутбук
    │
    │ (Wi-Fi / Ethernet)
    ▼
Home Router / CPE         ← NAT здесь! Твой 192.168.x.x → публичный IP
    │
    │ (DSL / Fiber / Cable)
    ▼
ISP (Провайдер)           ← Autonomous System (AS), BGP routing
    │
    │ (BGP peering / transit)
    ▼
Internet Exchange Point   ← MSK-IX, DE-CIX, Linx — физические точки обмена трафиком
    │
    ▼
Другой ISP / CDN / DC
    │
    ▼
Сервер назначения
```

**Ключевые уровни:**
- **Физический** — кабели, оптика, Wi-Fi радио
- **IP** — маршрутизация пакетов между сетями (BGP, OSPF)
- **TCP/UDP** — транспорт (надёжность, мультиплексирование портов)
- **Приложение** — HTTP, DNS, TLS

---

## 2. NAT (Network Address Translation)

### Зачем NAT существует

IPv4 адресов всего 4 миллиарда (~2^32). Реальных устройств в мире гораздо больше. Решение — приватные адреса (RFC 1918):
```
10.0.0.0/8       — 16 млн адресов
172.16.0.0/12    — 1 млн адресов
192.168.0.0/16   — 65k адресов
```
Эти адреса **не маршрутизируются** в публичном интернете.

### Как работает NAT (NAPT/PAT)

NAT-роутер поддерживает **таблицу трансляций**:

```
Приватный адрес:порт  ↔  Публичный адрес:порт
192.168.1.5:54321    ↔  203.0.113.1:1025
192.168.1.7:54321    ↔  203.0.113.1:1026   ← тот же порт клиента, другой внешний порт
192.168.1.5:54322    ↔  203.0.113.1:1027
```

**Исходящий пакет:**
```
[src: 192.168.1.5:54321 → dst: 8.8.8.8:53]
         │
         ▼ NAT router переписывает src
[src: 203.0.113.1:1025 → dst: 8.8.8.8:53]  ──► интернет
```

**Входящий ответ:**
```
[src: 8.8.8.8:53 → dst: 203.0.113.1:1025]  ◄── интернет
         │
         ▼ NAT lookup таблицы: 1025 → 192.168.1.5:54321
[src: 8.8.8.8:53 → dst: 192.168.1.5:54321]
```

### Типы NAT

| Тип | Поведение | Пробиваемость |
|-----|-----------|--------------|
| **Full Cone** | Внешний порт открыт для всех | Легко (любой может слать на внешний IP:порт) |
| **Restricted Cone** | Только если клиент ранее слал на тот IP | Средне |
| **Port Restricted** | Только если клиент слал на тот IP:порт | Сложнее |
| **Symmetric** | Новый внешний порт для каждого dst | Сложно (p2p невозможен) |

Большинство домашних роутеров — Port Restricted или Symmetric.

### NAT Traversal — как p2p работает за NAT

Проблема: два клиента за NAT не могут соединиться напрямую — у обоих нет публичных адресов.

**STUN (Session Traversal Utilities for NAT):**
```
Client A (192.168.1.5)            STUN Server (публичный)
      │                                   │
      │──── "какой мой внешний IP:port?" ─►│
      │◄─── "203.0.113.1:1025" ────────────│

Client B (10.0.0.7)               STUN Server
      │                                   │
      │──── "какой мой внешний IP:port?" ─►│
      │◄─── "198.51.100.2:2048" ───────────│
```

**Hole Punching (UDP):**
1. A и B через сигнальный сервер обменялись своими STUN-адресами
2. A шлёт пакет на адрес B → NAT A создаёт запись для B
3. B шлёт пакет на адрес A → NAT B создаёт запись для A
4. Пакеты A→B и B→A проходят через оба NAT

Работает для Full/Restricted Cone. Для Symmetric NAT нужен **TURN** (relay сервер).

**TURN:** Traffic проходит через relay (дорого, но всегда работает).

**ICE (Interactive Connectivity Establishment)** — WebRTC протокол, пробует: direct → STUN hole punch → TURN.

### CGN (Carrier-Grade NAT)

Операторы сами делают NAT на уровне провайдера:
```
Твой роутер NAT: 192.168.1.5 → 100.64.0.1 (RFC 6598, shared address space)
Провайдерский CGN: 100.64.0.1 → 203.0.113.x (публичный)
```
Это "двойной NAT" — ещё сложнее для p2p. Вот почему WebRTC иногда не работает у части пользователей.

---

## 3. BGP (Border Gateway Protocol)

### Что такое BGP

BGP — протокол маршрутизации между **автономными системами (AS)**. Каждый ISP, крупная компания, CDN имеет свой AS номер (ASN).

```
AS15169 (Google)
AS13335 (Cloudflare)
AS16509 (Amazon/AWS)
AS32934 (Facebook/Meta)
AS8359  (МТС Россия)
```

Интернет = граф AS, связанных BGP сессиями.

### Как BGP работает

**Типы отношений:**
- **Transit** — платишь upstream-провайдеру за доступ к интернету
- **Peering** — бесплатный обмен трафиком между равными (на IXP)
- **Customer** — тебе платят за transit

```
Tier-1 ISPs (AT&T, NTT, Cogent) — peering между собой, no transit
    │
Tier-2 ISPs (региональные) — покупают transit у Tier-1
    │
Tier-3 ISPs (мелкие провайдеры, предприятия)
```

**BGP анонс (advertisement):**
```
AS15169 → AS13335: "Я могу доставить трафик в 8.8.8.0/24 через меня"
AS13335 → все соседи: "8.8.8.0/24 доступен через AS13335 → AS15169"
```

**Path vector:** BGP хранит полный путь AS, по которому прошёл анонс:
```
8.8.8.0/24  AS_PATH: [AS13335, AS1299, AS15169]
```
Это предотвращает петли (если видишь свой AS в пути — отбрасываешь маршрут).

### BGP атрибуты выбора маршрута

Приоритет (по убыванию):
1. **Weight** (Cisco-специфичный, локальный)
2. **LOCAL_PREF** — предпочтение для исходящего трафика (выше = лучше)
3. **Shortest AS_PATH** — меньше хопов
4. **MED (Multi-Exit Discriminator)** — подсказка соседу как входить в твою сеть
5. **eBGP > iBGP**
6. **IGP metric** до next-hop

### BGP инциденты (реальные)

**BGP Hijack (перехват маршрутов):**
2018: Pakistan Telecom анонсировал префиксы YouTube, трафик был перенаправлен в Pakistan и дропнут. 2 часа YouTube был недоступен.

Атака: злоумышленник анонсирует более специфичный префикс (например /24 вместо /16). BGP всегда выбирает наиболее специфичный маршрут → трафик идёт к атакующему.

**Защита:**
- **RPKI (Resource Public Key Infrastructure)** — криптографическая подпись: "ASN X уполномочен анонсировать префикс Y"
- **IRR (Internet Routing Registry)** — база данных авторизованных маршрутов
- **BGP Filtering** — не принимать /25 и более специфичные префиксы от клиентов

**BGP Route Leak:**
2019: Cloudflare упала на ~30 минут из-за утечки маршрутов через небольшого провайдера. Маршруты попали в Tier-2 и перегрузили Cloudflare.

---

## 4. Как работают мобильные операторы

### Радиосеть (RAN — Radio Access Network)

```
Телефон
  │ (LTE/5G радио, OFDM)
  ▼
eNodeB / gNodeB (базовая станция)
  │ (оптика / микроволновый backhaul)
  ▼
RNC / BBU (Radio Network Controller / Base Band Unit)
  │
  ▼
Core Network
```

### LTE Core (EPC — Evolved Packet Core)

```
eNodeB ──── MME (Mobility Management Entity)
              │  управление подключением, аутентификация
              │
            S-GW (Serving Gateway)
              │  маршрутизирует пакеты между eNodeB и PGW
              │
            P-GW (PDN Gateway)
              │  точка выхода в интернет
              │  выдаёт IP (обычно из приватного диапазона → CGN)
              │
            HSS (Home Subscriber Server)
              │  база данных абонентов (IMSI, профили, ключи)
              │
            PCRF (Policy and Charging Rules Function)
                 тарификация, QoS, throttling
```

### 5G Core (Service Based Architecture)

5G перешёл на микросервисную архитектуру:
```
AMF — Access and Mobility Management (был MME)
SMF — Session Management Function
UPF — User Plane Function (был S/P-GW) — маршрутизирует реальный трафик
UDM — Unified Data Management (был HSS)
PCF — Policy Control Function
AUSF — Authentication Server Function
NRF — Network Repository Function (service discovery)
```

**Ключевое отличие 5G:** User Plane (UPF) отделён от Control Plane (SMF). UPF можно разместить близко к базовым станциям (edge computing) → низкая задержка.

### Как телефон подключается (attach procedure)

```
1. Телефон включился → IMSI (SIM идентификатор) → MME/AMF
2. AMF → HSS/UDM: аутентифицировать абонента (AKA — Authentication and Key Agreement)
3. HSS проверяет IMSI, генерирует ключи шифрования → отправляет в AMF
4. AMF ↔ телефон: взаимная аутентификация
5. AMF → SMF: создать PDU session
6. SMF → UPF: создать туннель GTP-U
7. UPF выдаёт IP телефону (через SMF → AMF → базовая станция → телефон)
8. Телефон получил IP → может слать пакеты
```

### GTP туннели (GPRS Tunneling Protocol)

Весь трафик телефона инкапсулируется в GTP:
```
[Ethernet] [IP outer: eNodeB → SGW] [UDP:2152] [GTP header: TEID] [IP inner: телефон → интернет] [TCP] [данные]
```

TEID (Tunnel Endpoint Identifier) — идентификатор туннеля для конкретного bearer (соединения). Это как VXLAN в datacenter, но для мобильных сетей.

### Роуминг

```
Твой телефон в Германии, ты абонент МТС:

Немецкая сеть (T-Mobile DE, VPLMN):
  базовая станция → AMF немецкой сети

МТС (HPLMN, домашний оператор):
  UDM/HSS МТС → аутентифицирует твой IMSI
  PCF МТС → применяет тарифный план

Трафик при Home Routed:
  Телефон → немецкая базовая станция → UPF МТС (Moscow) → интернет

Трафик при Local Breakout:
  Телефон → немецкая базовая станция → UPF T-Mobile (Germany) → интернет
  (быстрее, 5G SA роуминг)
```

### Диапазоны частот и их свойства

| Диапазон | Частота | Дальность | Проникновение | Использование |
|---------|---------|-----------|---------------|---------------|
| Low band | 600-900 MHz | Большая (30+ км) | Отличное | Покрытие сельской местности |
| Mid band | 1.8-3.5 GHz | Средняя (1-10 км) | Хорошее | Городские сети, основной LTE/5G |
| mmWave | 24-100 GHz | Малая (200м) | Плохое | 5G hotspot в аэропортах, стадионах |

---

## 5. Wireshark и анализ трафика

### Ключевые фильтры Wireshark

```
# Фильтр по IP
ip.addr == 192.168.1.1
ip.src == 10.0.0.1
ip.dst == 8.8.8.8

# TCP
tcp.port == 443
tcp.flags.syn == 1                    # SYN пакеты
tcp.flags.syn == 1 && tcp.flags.ack == 0  # Только SYN (новые соединения)
tcp.analysis.retransmission           # Ретрансмиссии

# HTTP
http.request.method == "POST"
http contains "password"

# DNS
dns.qry.name contains "google"

# TLS
tls.handshake.type == 1               # ClientHello

# Комбинированные
(tcp.port == 80 || tcp.port == 443) && ip.addr == 1.2.3.4
```

### Что видеть в TCP стриме

```
# SYN → SYN-ACK → ACK (3-way handshake)
Time   Source          Dest            Protocol  Info
0.000  192.168.1.5     93.184.216.34   TCP       54321 → 443 [SYN]
0.020  93.184.216.34   192.168.1.5     TCP       443 → 54321 [SYN, ACK]
0.020  192.168.1.5     93.184.216.34   TCP       54321 → 443 [ACK]

# TLS Handshake
0.021  192.168.1.5     93.184.216.34   TLSv1.3   Client Hello
0.045  93.184.216.34   192.168.1.5     TLSv1.3   Server Hello, Certificate, Finished
0.046  192.168.1.5     93.184.216.34   TLSv1.3   Finished
# ← с этого момента всё зашифровано

# Данные
0.047  192.168.1.5     93.184.216.34   TLSv1.3   Application Data (HTTP request внутри)
```

### TCP Retransmissions и что они значат

```
tcp.analysis.retransmission     — пакет отправлен повторно (нет ACK в RTO)
tcp.analysis.fast_retransmission — 3 дублированных ACK (получатель сообщает о пропуске)
tcp.analysis.duplicate_ack      — дублированный ACK (сигнал потери)
tcp.analysis.window_full        — receive window заполнен (получатель не успевает читать)
tcp.analysis.zero_window        — receive window = 0 (получатель перегружен, отправитель ждёт)
```

**Много ретрансмиссий → потери пакетов → плохой канал (Wi-Fi, мобильная сеть) или перегрузка.**

### tcpdump (CLI альтернатива)

```bash
# Захват на интерфейсе
tcpdump -i eth0 -w capture.pcap

# Фильтр host и port
tcpdump -i eth0 host 8.8.8.8 and port 53

# Только SYN пакеты (новые соединения)
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack == 0'

# HTTP трафик с данными
tcpdump -i eth0 -A port 80

# Захват без DNS разрешения (-n) и с максимальным размером пакета
tcpdump -i eth0 -n -s 0 -w /tmp/capture.pcap

# Анализ конкретного IP
tcpdump -i eth0 -n 'src 192.168.1.0/24'
```

### ss / netstat — состояние соединений

```bash
ss -tnp                  # TCP соединения с PID
ss -tlnp                 # listening TCP сокеты
ss -s                    # статистика (количество в каждом состоянии)

# Состояния TCP соединений
LISTEN       — ждёт входящих SYN
SYN_SENT     — отправил SYN, ждёт SYN-ACK
ESTABLISHED  — активное соединение
TIME_WAIT    — соединение закрыто, ждёт (2*MSL = 60-120с) для поздних пакетов
CLOSE_WAIT   — удалённая сторона закрыла, локальная ещё нет
FIN_WAIT_2   — наша FIN принята, ждём FIN от удалённого
```

**Много TIME_WAIT** — нормально при высоком трафике, но может исчерпать ephemeral порты. Решение: SO_REUSEADDR, keep-alive соединения, или увеличить `net.ipv4.ip_local_port_range`.

---

## 6. Сетевое программирование в Go

### Raw сокеты и pcap

```go
// Захват пакетов (нужны root права или CAP_NET_RAW)
import "github.com/google/gopacket/pcap"
import "github.com/google/gopacket"
import "github.com/google/gopacket/layers"

handle, err := pcap.OpenLive("eth0", 1600, true, pcap.BlockForever)
if err != nil {
    log.Fatal(err)
}
defer handle.Close()

// Фильтр как в tcpdump
handle.SetBPFFilter("tcp and port 80")

packetSource := gopacket.NewPacketSource(handle, handle.LinkType())
for packet := range packetSource.Packets() {
    // Достаём слои
    ipLayer := packet.Layer(layers.LayerTypeIPv4)
    if ipLayer != nil {
        ip := ipLayer.(*layers.IPv4)
        fmt.Printf("src: %s → dst: %s\n", ip.SrcIP, ip.DstIP)
    }

    tcpLayer := packet.Layer(layers.LayerTypeTCP)
    if tcpLayer != nil {
        tcp := tcpLayer.(*layers.TCP)
        fmt.Printf("port: %d → %d, SYN: %v\n", tcp.SrcPort, tcp.DstPort, tcp.SYN)
    }
}
```

### Диагностика сети из Go

```go
// Trace route (ICMP TTL probe)
// При каждом хопе TTL уменьшается на 1, при TTL=0 маршрутизатор отвечает ICMP Time Exceeded

// Измерение RTT к конкретному хосту
func measureRTT(host string) (time.Duration, error) {
    start := time.Now()
    conn, err := net.DialTimeout("tcp", host+":80", 5*time.Second)
    if err != nil {
        return 0, err
    }
    conn.Close()
    return time.Since(start), nil
}

// DNS lookup с таймаутом
r := &net.Resolver{
    PreferGo: true,
    Dial: func(ctx context.Context, network, address string) (net.Conn, error) {
        d := net.Dialer{Timeout: 2 * time.Second}
        return d.DialContext(ctx, "udp", "8.8.8.8:53")
    },
}
addrs, err := r.LookupHost(ctx, "google.com")
```

---

## 7. Вопросы на собеседовании

### Q: Объясни как работает NAT. Какие проблемы он создаёт?

**Ответ:** NAT (Network Address Translation) транслирует приватные IP адреса во внешний публичный IP. Роутер поддерживает таблицу трансляций: приватный IP:порт ↔ публичный IP:порт. При исходящем пакете роутер заменяет src IP:порт на публичный, при входящем — обратно. Проблемы: 1) p2p соединения невозможны напрямую (нужен STUN/TURN), 2) долгоживущие соединения могут потерять запись в таблице (NAT timeout), 3) некоторые протоколы не работают через NAT (FTP active mode, SIP), 4) CGN у провайдеров создаёт двойной NAT, усложняя troubleshooting.

---

### Q: Что такое BGP? Почему интернет работает?

**Ответ:** BGP — протокол обмена маршрутами между автономными системами. Каждый ISP — AS с уникальным ASN. BGP позволяет AS узнать о префиксах (IP подсетях) всех остальных AS. Маршрутизаторы BGP хранят полный путь AS (path vector) — это предотвращает петли. Интернет работает потому что все крупные игроки (ISP, CDN, облака) поддерживают BGP сессии на IXP и через прямые пиринговые соединения. Главная уязвимость — нет встроенной аутентификации анонсов (BGP hijack), решается через RPKI.

---

### Q: Как работает мобильная сеть? Что происходит когда звонишь?

**Ответ:** Телефон подключается к базовой станции (eNodeB в LTE, gNodeB в 5G) через радиоканал OFDM. Базовая станция соединяется с Core Network: в LTE это EPC (MME — управление, SGW/PGW — данные, HSS — абоненты). При подключении: телефон отправляет IMSI → Core аутентифицирует через HSS (AKA) → устанавливается GTP туннель → телефон получает IP. Весь трафик инкапсулирован в GTP туннель от базовой станции до шлюза PGW, который выходит в интернет (часто через CGN провайдера). В 5G архитектура Service-Based: User Plane (UPF) отделён от Control Plane, можно размещать UPF ближе к пользователю для edge computing.

---

### Q: Как работает Wireshark / что искать при отладке сетевых проблем?

**Ответ:** Первое — смотрю на ретрансмиссии (`tcp.analysis.retransmission`) — их много → потери пакетов. Второе — TCP Zero Window (`tcp.analysis.zero_window`) — получатель не успевает читать → backpressure проблема. Третье — время от SYN до SYN-ACK (RTT). Четвёртое — FIN/RST паттерны для неожиданных закрытий соединений. В продакшне использую tcpdump для захвата (pcap файл), анализирую в Wireshark. Для Go сервисов могу добавить pcap захват через gopacket.
