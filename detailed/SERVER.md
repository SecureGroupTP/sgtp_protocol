# SGTP Relay Server: детальное описание

Этот документ описывает внутреннее устройство Go-сервера: архитектуру, маршрутизацию, транспорты, HTTP-сессии, User Directory, лимиты и конфигурацию.

---

## 1. Философия сервера

SGTP relay — **stateless transparent forwarder**. Это сознательное архитектурное решение: сервер не знает о содержимом сообщений, не проверяет подписи, не хранит историю и не держит долгосрочное состояние.

Преимущества:
- Минимальная поверхность атаки (у сервера нет ключей)
- Горизонтальное масштабирование (любой узел может форвардировать любой фрейм)
- Простота реализации и аудита

Недостатки:
- Нет буферизации для оффлайн-получателей (UDP-like семантика)
- Нет защиты от UUID spoofing
- Нет rate limiting «из коробки»

---

## 2. Мульти-транспортная архитектура

Сервер запускает несколько listener-ов одновременно, каждый на своём порту. Все они форвардируют SGTP-фреймы в одну и ту же in-memory room map.

```
[TCP :7777]  ──────┐
[TLS :7778]  ──────┤
[HTTP :8000] ──────┤──→  Room Map (map + RWMutex)
[HTTPS :8001]──────┤         │
[WS :8080]   ──────┤    [room_uuid → room{clients}]
[WSS :8081]  ──────┘
```

### 2.1 Порты и переменные окружения

```bash
# Транспорты
TCP_PORT=7777
TCP_TLS_PORT=7778
HTTP_PORT=8000
HTTP_TLS_PORT=8001
WS_PORT=8080
WS_TLS_PORT=8081

# TLS
TLS_CERT_FILE=/etc/sgtp/cert.crt
TLS_KEY_FILE=/etc/sgtp/key.key

# Advertised ports (за NAT)
ADVERTISE_TCP_PORT=7777   # если внешний порт отличается от внутреннего

# HTTP session настройки
HTTP_RECV_TIMEOUT=60s
HTTP_SEND_MAX_BYTES=16777216      # 16 МиБ
HTTP_SESSION_BUFFER_BYTES=4194304 # 4 МиБ буфер на сессию
HTTP_SESSION_TTL=10m
HTTP_SESSION_CLEANUP=1m

# User Directory
PG_DSN=postgres://user:pass@host:5432/sgtp?sslmode=disable
AVATAR_MAX_BYTES=33554432   # 32 МиБ
SEARCH_MAX_RESULTS=20
SUBSCRIBE_MAX=500
```

---

## 3. Внутренние структуры данных

### 3.1 Room

```go
type room struct {
    mu      sync.RWMutex
    clients map[[16]byte]*conn  // UUID → соединение
}
```

- `clients` — map от 16-байтного UUID к указателю на соединение
- `mu` — RWMutex: broadcast читает под RLock, добавление/удаление peer-а под Lock
- При уходе последнего peer-а комната удаляется из Room Map

### 3.2 Connection

```go
type conn struct {
    uuid        [16]byte
    roomID      [16]byte
    netConn     net.Conn        // TCP / TLS / WebSocket underlying
    sendCh      chan []byte     // исходящая очередь, размер 64
    sendTimeout time.Duration   // 250ms по умолчанию
    closed      chan struct{}
}
```

- `sendCh`: буферизованный канал ёмкостью **64 фрейма**. Если канал полон, новый фрейм **сбрасывается** (slow consumer) и соединение закрывается через `sendTimeout` (250 мс)
- `closed`: сигнал для горутин, обслуживающих соединение

### 3.3 Room Map

```go
roomsMu sync.RWMutex
rooms   map[[16]byte]*room
```

Обычный `map` с явным `RWMutex`: broadcast читает под `RLock`, добавление/удаление комнат — под `Lock`. Отдельный мьютекс (не `sync.Map`) позволяет точнее контролировать блокировки при удалении пустых комнат.

---

## 4. Жизненный цикл TCP-соединения

```
[Client connect]
     │
     ├─ Accept() → новая горутина на соединение
     ├─ Немедленно отправляем 25-байтный discovery banner
     │
     ├─ readLoop():
     │   loop:
     │     read 64 байта (header)
     │     parse: roomID, receiverID, senderID, payloadLen
     │     read payloadLen байт
     │     read 64 байта (signature)
     │     → route(frame)
     │
     ├─ writeLoop():
     │   loop:
     │     select sendCh → write to netConn
     │     if write fails → close conn
     │
     └─ При первом фрейме:
         ├─ Регистрируем соединение в room.clients[senderID]
         └─ Если senderID уже занят → перезаписываем (UUID spoofing vulnerability)
```

### 4.1 25-байтный discovery banner

При каждом TCP accept сервер немедленно шлёт:

```
Byte 0:      flags (битовое поле: бит 0 = TCP активен, бит 1 = TCP+TLS, бит 2 = HTTP, бит 3 = HTTP+TLS, бит 4 = WS, бит 5 = WS+TLS)
Bytes 1-4:   TCP port (uint32 big-endian)
Bytes 5-8:   TCP+TLS port
Bytes 9-12:  HTTP port
Bytes 13-16: HTTP+TLS port
Bytes 17-20: WebSocket port
Bytes 21-24: WebSocket+TLS port
```

Порт = 0 означает, что транспорт не настроен (соответствующий бит в flags тоже = 0).

Flutter-клиент при TCP-подключении читает и игнорирует эти 25 байт (`readAndDiscard(25)`), затем начинает нормальный SGTP frame stream.

---

## 5. Routing: broadcast vs unicast

```go
func route(frame []byte) {
    header := parseHeader(frame[:64])
    r := getOrCreateRoom(header.roomID)

    if header.receiverID == BROADCAST_UUID {
        r.broadcast(header.senderID, frame)
    } else {
        r.unicast(header.receiverID, frame)
    }
}

func (r *room) broadcast(senderID [16]byte, frame []byte) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    for uuid, c := range r.clients {
        if uuid != senderID {
            select {
            case c.sendCh <- frame:
            default:
                // очередь полна → slow consumer, drop
            }
        }
    }
}

func (r *room) unicast(receiverID [16]byte, frame []byte) {
    r.mu.RLock()
    c, ok := r.clients[receiverID]
    r.mu.RUnlock()
    if !ok {
        return  // получатель не подключён → тихо дропаем
    }
    select {
    case c.sendCh <- frame:
    default:
        // slow consumer → drop
    }
}
```

**Важно:** сервер **не буферизует** фреймы для оффлайн-получателей. Unicast к отключённому peer-у молча теряется. Это означает, что CHAT_KEY, отправленный мастером, теряется, если получатель разорвал соединение между CHAT_REQUEST и получением CHAT_KEY. Клиент обрабатывает это через таймаут и повторный CHAT_REQUEST при переподключении.

---

## 6. WebSocket транспорт

WebSocket-соединения проходят стандартный HTTP Upgrade, после чего данные передаются как бинарные WS-сообщения.

**Особенность:** одно WS-сообщение может содержать **несколько конкатенированных SGTP-фреймов**. Клиент и сервер должны распарсивать поток из WS-payload побайтово, читая длину каждого фрейма из `payload_length` в заголовке.

```
WS binary message:
  [Header1(64)][Payload1(N1)][Sig1(64)][Header2(64)][Payload2(N2)][Sig2(64)]...
```

На сервере каждый WS binary message разбирается в цикле (extract frames until buffer empty), и каждый фрейм обрабатывается независимо.

---

## 7. HTTP Session транспорт

Для клиентов без возможности держать постоянное соединение реализован stateful HTTP-протокол поверх short-lived запросов.

### 7.1 Жизненный цикл HTTP-сессии

```
POST /sgtp/session
  Body: пусто или room_uuid (16 байт)
  Response: sid (16 байт raw или hex-строка)
  → Создаётся объект Session {
      sid, roomID,
      inBuf: ring buffer 4 МиБ,
      outBuf: ring buffer 4 МиБ,
      lastActivity: now,
      ttl: 10 минут
    }

POST /sgtp/send?sid=<hex32>
  Body: raw SGTP frame bytes (до 16 МиБ)
  → Парсим фреймы из body
  → Для каждого фрейма: route() → остальные peer-ы получают через sendCh

GET /sgtp/recv?sid=<hex32>
  → Long-poll: ждём данных в inBuf до 60 секунд (HTTP_RECV_TIMEOUT)
  → Когда данные появляются: отдаём как chunked transfer encoding
  → При timeout (нет данных) — 200 OK с пустым телом (поток завершается без данных)

DELETE /sgtp/session?sid=<hex32>
  → Удаляем сессию, высвобождаем буферы
```

### 7.2 Буферизация HTTP-сессии

Каждая HTTP-сессия имеет **4 МиБ in-memory буфер** (`HTTP_SESSION_BUFFER_BYTES`). Входящие фреймы от relay кладутся в буфер сессии. При GET /sgtp/recv данные вычитываются из буфера.

При переполнении буфера (4 МиБ) новые фреймы дропаются — нет back-pressure на relay. Это потенциальная потеря данных при медленном клиенте.

### 7.3 Cleanup

Каждые `HTTP_SESSION_CLEANUP` (1 минута) горутина очищает сессии старше `HTTP_SESSION_TTL` (10 минут). `lastActivity` обновляется при каждом send/recv запросе.

---

## 8. User Directory (Userdir)

Userdir — встроенный сервис регистрации и поиска пользователей по ed25519 публичному ключу. Работает на том же TCP-порту, что и relay, через мультиплексирование по magic prefix.

### 8.1 Демультиплексирование на уровне соединения

```
TCP connection → читаем первые 32 байта
  if all 32 bytes == 0x00:
    → Userdir protocol
  else:
    → SGTP relay protocol (эти 32 байта = начало первого SGTP header)
```

Поскольку SGTP header начинается с `room_uuid` (16 байт) и `receiver_uuid` (16 байт), валидный SGTP фрейм никогда не начинается с 32 нулей (UUID не бывают all-zero в нормальной работе).

### 8.2 Протокол Userdir

Бинарный протокол с length-prefixed сообщениями:

```
[u8   type]
[u32  length big-endian]
[N    payload]
```

**Типы сообщений:**

| Code | Направление | Назначение |
|---|---|---|
| 0x01 | C→S | REGISTER / UPDATE профиля |
| 0x02 | C→S | SEARCH по username или pubkey |
| 0x03 | C→S | GET_PROFILE (полный профиль) |
| 0x04 | C→S | GET_META (лёгкий метаданные) |
| 0x05 | C→S | SUBSCRIBE на обновления профиля |
| 0x06 | C→S | UNSUBSCRIBE |
| 0x81 | S→C | OK (подтверждение) |
| 0x82 | S→C | ERROR (код + текст) |
| 0x83 | S→C | SEARCH_RESULTS |
| 0x84 | S→C | PROFILE (полные данные) |
| 0x85 | S→C | META |
| 0x86 | S→C | NOTIFY (push при обновлении подписанного профиля) |

### 8.3 REGISTER payload

```
u8     version = 1
u16    username_len
u8[]   username (UTF-8, паттерн @[A-Za-z0-9_]*, до 32 символов, опционально)
u16    fullname_len
u8[]   fullname (UTF-8)
u8[32] ed25519 pubkey
u32    avatar_len
u8[]   avatar bytes (до AVATAR_MAX_BYTES = 32 МиБ)
u8     sig_alg = 1 (Ed25519)
u8[64] signature (Ed25519 подпись над всем payload без подписи)
```

Сервер верифицирует подпись перед записью. Это единственное место, где relay проверяет подпись — Userdir отдельный контекст.

### 8.4 Database схема (PostgreSQL)

```sql
CREATE TABLE profiles (
    pubkey          bytea PRIMARY KEY,         -- Ed25519 pubkey, 32 байта
    username        varchar(255) UNIQUE,       -- NULL если не задан
    fullname        text NOT NULL DEFAULT '',
    avatar          bytea,                     -- до 32 МиБ
    avatar_sha256   bytea(32),
    updated_at      timestamptz NOT NULL DEFAULT now(),
    created_at      timestamptz NOT NULL DEFAULT now()
);

-- Индекс для поиска по username (case-insensitive)
CREATE UNIQUE INDEX idx_profiles_username_lower
    ON profiles (lower(username))
    WHERE username IS NOT NULL;
```

Retention: профили хранятся бессрочно. Удаление возможно только напрямую через SQL (нет DELETE API).

### 8.5 SUBSCRIBE / NOTIFY

Клиент может подписаться на обновления до `SUBSCRIBE_MAX = 500` профилей за одно соединение. При обновлении профиля сервер проверяет список активных подписок и шлёт NOTIFY всем подписчикам.

Подписки хранятся только in-memory и теряются при разрыве соединения. При переподключении клиент должен повторно подписаться.

---

## 9. Лимиты и квоты

### 9.1 Протокольные лимиты

| Параметр | Значение | Где задан |
|---|---|---|
| Макс. payload фрейма | 16 МиБ (16 777 216 байт) | Парсер заголовка |
| Макс. размер фрейма | 16 МиБ + 128 байт | Суммарно |
| Окно timestamp | ±30 секунд | Только на клиенте |
| Очередь отправки | 64 фрейма | `sendCh` capacity |
| Таймаут slow consumer | 250 мс | `sendTimeout` |

### 9.2 HTTP Session лимиты

| Параметр | Значение | ENV переменная |
|---|---|---|
| Буфер сессии | 4 МиБ | HTTP_SESSION_BUFFER_BYTES |
| TTL сессии | 10 минут | HTTP_SESSION_TTL |
| Период очистки | 1 минута | HTTP_SESSION_CLEANUP |
| Таймаут recv | 60 секунд | HTTP_RECV_TIMEOUT |
| Макс. body /send | 16 МиБ | HTTP_SEND_MAX_BYTES |

### 9.3 User Directory лимиты

| Параметр | Значение | ENV переменная |
|---|---|---|
| Макс. размер аватара | 32 МиБ | AVATAR_MAX_BYTES |
| Макс. результатов поиска | 20 | SEARCH_MAX_RESULTS |
| Макс. подписок на соединение | 500 | SUBSCRIBE_MAX |
| Макс. длина username | 32 символа | hardcoded |

### 9.4 Отсутствующие лимиты (известные пробелы)

Следующие лимиты **не реализованы** в текущей версии:

- **Per-IP rate limiting**: нет ограничений на количество подключений с одного IP
- **Per-room limits**: нет ограничений на количество peer-ов в комнате
- **Global connection limit**: нет ограничений на общее число соединений
- **Bandwidth throttling**: нет управления полосой пропускания
- **Room count limit**: нет ограничений на количество уникальных комнат
- **HTTP session count**: нет ограничений на количество HTTP-сессий

**Последствие:** сервер уязвим к resource exhaustion attacks (создание огромного количества комнат, подключений, сессий).

---

## 10. Конфигурация и деплой

### 10.1 Минимальный запуск (только TCP)

```bash
TCP_PORT=7777 ./sgtp-server
```

Сервер слушает TCP на порту 7777 и возвращает discovery banner с заполненным только `TCP_PORT`.

### 10.2 Полная конфигурация

```bash
TCP_PORT=7777 \
TCP_TLS_PORT=7778 \
HTTP_PORT=8000 \
HTTP_TLS_PORT=8001 \
WS_PORT=8080 \
WS_TLS_PORT=8081 \
TLS_CERT_FILE=/etc/letsencrypt/live/example.com/fullchain.pem \
TLS_KEY_FILE=/etc/letsencrypt/live/example.com/privkey.pem \
PG_DSN="postgres://sgtp:password@localhost:5432/sgtp?sslmode=disable" \
./sgtp-server
```

### 10.3 Discovery endpoint

Клиенты, использующие HTTP discovery, делают:

```
GET http://example.com:8000/sgtp/discovery
```

Ответ:

```json
{
  "transports": [
    {"type": "tcp",       "host": "example.com", "port": 7777, "tls": false},
    {"type": "tcp",       "host": "example.com", "port": 7778, "tls": true},
    {"type": "http",      "host": "example.com", "port": 8000, "tls": false},
    {"type": "http",      "host": "example.com", "port": 8001, "tls": true},
    {"type": "websocket", "host": "example.com", "port": 8080, "tls": false},
    {"type": "websocket", "host": "example.com", "port": 8081, "tls": true}
  ]
}
```

Если порт не настроен (значение `0`), он не включается в ответ.

### 10.4 За reverse-proxy (nginx)

Если сервер стоит за nginx, нужно настроить `ADVERTISE_TCP_PORT` на внешний порт:

```nginx
stream {
    server {
        listen 443;
        proxy_pass 127.0.0.1:7777;
    }
}
```

```bash
TCP_PORT=7777 ADVERTISE_TCP_PORT=443 ./sgtp-server
```

---

## 11. Метрики и наблюдаемость

На данный момент сервер **не экспортирует метрики** (нет Prometheus endpoint, нет structured logging). Логирование через стандартный `log` пакет Go:

```
2024/01/15 12:00:00 new connection from 1.2.3.4:51234
2024/01/15 12:00:00 peer registered: roomID=abc, uuid=def
2024/01/15 12:00:01 connection closed: uuid=def (reason: EOF)
```

Для production мониторинга рекомендуется:
- Оберть `net.Listen` в middleware, считающий активные соединения
- Добавить Prometheus gauge для `rooms.count` и `connections.total`
- Включить `SO_REUSEPORT` для multi-process деплоя

---

## 12. Userdir: SEARCH API

### 12.1 SEARCH request

```
u8     type = 0x02
u32    len
  u8   search_type  // 1=by_username, 2=by_pubkey, 3=full_text
  u16  query_len
  u8[] query
```

### 12.2 SEARCH_RESULTS response

```
u8     type = 0x83
u32    len
  u32  count
  for each result:
    u8[32] pubkey
    u16    username_len
    u8[]   username (может быть пустым)
    u16    fullname_len
    u8[]   fullname
    u32    avatar_sha256_len  // 0 если нет аватара, иначе 32
    u8[]   avatar_sha256      // SHA-256 аватара для кэширования
```

Обратите внимание: в результатах поиска аватар не передаётся напрямую — только его SHA-256. Полный аватар запрашивается отдельно через GET_PROFILE / GET_META, что позволяет клиенту кэшировать аватары по хэшу и не перекачивать неизменившиеся.
