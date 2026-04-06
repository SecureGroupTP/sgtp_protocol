# Flutter SGTP Client: детальное описание

Этот документ описывает внутреннее устройство Flutter-клиента: архитектуру кода, state machine, управление ключами, обработку медиа, хранение истории и обработку ошибок.

---

## 1. Архитектура кода

### 1.1 Слои

```
┌────────────────────────────────────────────┐
│             UI Layer (Widgets)             │
│  ChatScreen, MessageBubble, MediaViewer    │
├────────────────────────────────────────────┤
│           Application Layer               │
│  SgtpBloc / ChangeNotifier                 │
│  (обёртка событий клиента для UI)          │
├────────────────────────────────────────────┤
│         SgtpClient (Core, ~3000 loc)       │
│  Управляет состоянием, peer-ами, ключами,  │
│  отправкой/приёмом, history sync           │
├────────────────────────────────────────────┤
│     Transport Layer (абстрактный)          │
│  TcpSgtpTransport / HttpSgtpTransport /    │
│  WebSocketSgtpTransport                    │
├────────────────────────────────────────────┤
│        Protocol Layer                      │
│  FrameBuilder / FrameParser / PacketTypes  │
├────────────────────────────────────────────┤
│       Crypto Layer                         │
│  ChaCha20Utils / Ed25519Utils / X25519Utils│
└────────────────────────────────────────────┘
```

### 1.2 Ключевые файлы

| Файл | Назначение |
|---|---|
| `core/protocol/frame_builder.dart` | Сборка SGTP-фреймов всех типов |
| `core/protocol/frame_parser.dart` | Парсинг входящего byte stream |
| `core/protocol/packet_types.dart` | Константы кодов пакетов (0x0000–0x0011) |
| `core/crypto/chacha20_utils.dart` | ChaCha20-Poly1305 encrypt/decrypt |
| `core/crypto/ed25519_utils.dart` | Подпись и верификация |
| `core/crypto/x25519_utils.dart` | DH-обмен |
| `features/messaging/data/services/sgtp_client.dart` | Основной клиент |
| `features/messaging/data/transport/tcp_sgtp_transport.dart` | TCP транспорт |
| `features/messaging/data/transport/http_sgtp_transport.dart` | HTTP session транспорт |
| `features/messaging/data/transport/websocket_sgtp_transport_io.dart` | WebSocket (native) |
| `features/messaging/data/transport/websocket_sgtp_transport_web.dart` | WebSocket (web) |
| `features/messaging/data/transport/server_discovery.dart` | HTTP discovery |
| `features/messaging/data/repositories/chat_history_repository.dart` | SQLite история |

---

## 2. Конфигурация клиента (SgtpConfig)

```dart
class SgtpConfig {
  // Сетевые параметры
  final String serverAddr;           // "example.com:7777"
  final SgtpTransportFamily transport; // tcp / http / websocket
  final bool useTls;

  // Идентификация
  final Uint8List roomUUID;          // 16 байт, разделяемый с другими участниками
  final SimpleKeyPairData identityKeyPair;  // Ed25519 ключевая пара
  final Uint8List myPublicKey;       // Ed25519 pubkey (32 байта)

  // Авторизация
  final Set<String> whitelist;       // Набор hex-строк доверенных ed25519 pubkey

  // Метаданные чата
  final String chatName;             // Имя пользователя в чате (≤255 UTF-8 байт)
  final Uint8List? chatAvatarBytes;  // Аватар (≤4 КиБ, PNG/JPEG)

  // Медиа
  final int mediaChunkSizeBytes;     // Размер чанка, по умолчанию 100 КиБ
                                     // На web авто-увеличивается до ≥512 КиБ
}
```

### 2.1 Как создаётся roomUUID

`roomUUID` — общий секрет для всех участников комнаты. Способы распределения:

- QR-код с параметрами (roomUUID + whitelist + serverAddr)
- Deep link (sgtp://join?room=hex&server=host:port&wl=hex1,hex2,...)
- Прямая передача out-of-band

SGTP не определяет протокол инвайтов — это ответственность приложения.

---

## 3. State Machine клиента

```
┌─────────────────┐
│  DISCONNECTED   │◄──────────────────────────────┐
└────────┬────────┘                               │
         │ connect()                              │
         ▼                                        │
┌─────────────────┐                               │
│   CONNECTING    │                               │
│ (TCP/WS/HTTP)   │──────────────────────────────►│ ошибка
└────────┬────────┘                               │
         │ соединение установлено                 │
         │ отправляем INTENT                      │
         ▼                                        │
┌─────────────────────┐                           │
│  WAITING_HANDSHAKE  │                           │
│                     │                           │
│ ← PING от peers     │──────────────────────────►│ PING timeout
│ → PONG              │   30 секунд без ответа     │
│ ← INFO_RESPONSE     │                           │
│ → CHAT_REQUEST      │                           │
│ ← CHAT_KEY          │                           │
│ → CHAT_KEY_ACK      │                           │
└────────┬────────────┘                           │
         │ CHAT_KEY получен (или мы мастер)       │
         ▼                                        │
┌─────────────────┐                               │
│      READY      │                               │
│                 │──────────────────────────────►│ disconnect() / FIN
│ Полное участие  │
└─────────────────┘
```

### 3.1 События клиента

```dart
// Генерируются клиентом и могут слушаться подписчиками
class SgtpReady        { bool isMaster; String roomUUIDHex; }
class SgtpDisconnected {}  // причина не передаётся
class SgtpMessageReceived {
  ChatMessage message;
}
class SgtpMediaProgress {
  String echoId;      // echo-идентификатор отправки
  String messageId;
  double progress;    // 0.0–1.0
}
class SgtpPeerJoined   { String peerUUID; String ed25519PubHex; }
class SgtpPeerLeft     { String peerUUID; }
```

---

## 4. PeerInfo: управление peer-ами

```dart
class PeerInfo {
  final String uuid;              // hex UUID (16 байт → 32 символа hex)
  final Uint8List uuidBytes;      // сырые 16 байт
  final Uint8List ed25519PubKey;  // 32 байта, из PING/PONG
  Uint8List? sharedKey;           // 32 байта, после DH обмена
  bool handshakeComplete;         // true после получения PONG
  int protocolVersion;            // в текущем runtime фактически 0x0002
  String? displayName;            // из CHAT_REQUEST или chat_meta
  Uint8List? avatar;              // из CHAT_REQUEST или chat_meta
  DateTime lastSeen;              // обновляется при каждом фрейме
}
```

### 4.1 Stale peer removal

Порог stale-таймаута зависит от `pingIntervalSeconds` (по умолчанию 30 с):
- Периодическая проверка: `pingIntervalSeconds × 3` мс (по умолчанию **90 с**)
- При получении INTENT от peer-а: `pingIntervalSeconds × 4` мс (по умолчанию **120 с**)

Peer считается stale и удаляется из `_peers`, если за это время не пришло ни одного фрейма. Мастер не отправляет `KICKED` для stale peer-ов автоматически; нормативный сценарий с `KICK_REQUEST` остаётся протокольной веткой, но в текущем клиентском runtime используется редко.

### 4.2 Keepalive

Flutter-клиент отправляет keepalive каждые `pingIntervalSeconds` (по умолчанию 30 с):
- Если `_peers` **не пуст** — шлёт **PING** каждому известному peer-у, чтобы обновить `lastSeen` и поддержать TCP-соединение через NAT.
- Если `_peers` **пуст** — шлёт **INTENT** (broadcast), чтобы поддержать idle-соединение с сервером.

---

## 5. Транспортный уровень

### 5.1 Абстракция транспорта

```dart
abstract class SgtpTransport {
  Stream<Uint8List> get inbound;   // поток входящих сырых байт
  Future<void> send(Uint8List frame);
  Future<void> connect();
  Future<void> close();
}
```

Все три реализации скрыты за этим интерфейсом. `SgtpClient` не знает, какой именно транспорт используется.

### 5.2 TCP транспорт

```dart
class TcpSgtpTransport implements SgtpTransport {
  // 1. dart:io SecureSocket.connect() при useTls=true
  // 2. Читаем 25 байт discovery banner, отбрасываем
  // 3. inbound: слушаем сокет и прокидываем байты в StreamController<Uint8List>
  // 4. send: socket.add(frame) + flush()
}
```

TCP — single-stream, поэтому важна сериализация через `_sendChain` (см. ниже).

### 5.3 WebSocket транспорт

Два файла: `_io.dart` для мобильных/desktop и `_web.dart` для браузера.
API идентичен, реализация разная — платформо-специфический код через conditional imports.

```dart
// native (io): raw socket + HTTP Upgrade + ручной WS frame encode/decode
// web: WebSocketChannel.connect(uri)

transport.send(frameBytes);
```

### 5.4 HTTP Session транспорт

```dart
class HttpSgtpTransport implements SgtpTransport {
  String? _sidHex;  // hex session ID

  Future<void> connect() async {
    // POST /sgtp/session -> sid (16 raw bytes; дополнительно поддерживается JSON {sid})
    _sidHex = await _createSession();
    _startRecvLoop();  // непрерывный GET /sgtp/recv long-poll loop
  }

  Future<void> send(Uint8List frame) async {
    await _client.post('/sgtp/send?sid=$_sidHex', body: frame);
  }
}
```

---

## 6. Frame parsing: накопительный буфер

Входящий byte stream не содержит маркеров разделения фреймов — длина следующего фрейма зависит от `payload_length` в заголовке. Реализован накопительный буфер:

```dart
final _buffer = BytesBuilder(copy: false);

void _onData(Uint8List chunk) {
  _buffer.add(chunk);
  // Пытаемся извлечь фреймы пока данных достаточно
  while (true) {
    final frame = _tryExtractFrame(_buffer.toBytes());
    if (frame == null) break;  // недостаточно данных
    _buffer.clear();
    // Добавляем оставшийся "хвост" обратно
    _dispatch(frame);
  }
}

Uint8List? _tryExtractFrame(Uint8List buf) {
  if (buf.length < 64) return null;  // нет даже заголовка
  final payloadLen = ByteData.view(buf.buffer).getUint32(52, Endian.big);
  final totalLen = 64 + payloadLen + 64;
  if (buf.length < totalLen) return null;
  return buf.sublist(0, totalLen);
}
```

---

## 7. Сериализованная очередь отправки (_sendChain)

TCP — stream-oriented, не поддерживает одновременную запись из нескольких асинхронных операций. Все отправки сериализуются через `_sendChain`:

```dart
Future<void> _sendFrame(Uint8List frame) async {
  _sendChain = _sendChain.then((_) async {
    await _transport.send(frame);
  });
  await _sendChain;
}
```

Это guarantees, что байты двух параллельных отправок не перемешиваются. Особенно важно при параллельных чанках медиа: даже если чанки готовятся параллельно (async), они записываются в сокет строго последовательно.

---

## 8. Обработка входящих MESSAGE

```dart
Future<void> _onMessage(ParsedFrame frame) async {
  // 1. Проверка ed25519 подписи
  final peer = _peers[frame.senderUUID];
  if (peer == null || !peer.handshakeComplete) return;
  final valid = await ed25519.verify(
    frame.headerAndPayload,
    signature: frame.signature,
    publicKey: peer.ed25519PubKey,
  );
  if (!valid) return;

  // 2. Извлечение message_uuid и nonce из payload
  final msgUUID = frame.payload.sublist(0, 16);
  final nonceBytes = frame.payload.sublist(16, 24);
  final ciphertext = frame.payload.sublist(24);

  // 3. Дешифровка с текущим Chat Key
  final nonce96 = Uint8List(12)..setRange(4, 12, nonceBytes);
  final plaintext = await chacha20Decrypt(ciphertext, _chatKey!, nonce96);
  if (plaintext == null) return;  // AEAD tag invalid

  // 4. Парсинг JSON
  final json = jsonDecode(utf8.decode(plaintext)) as Map<String, dynamic>;
  final type = json['type'] as String;

  // 5. Диспетчеризация по типу
  switch (type) {
    case 'text':       await _handleText(json, frame, msgUUID);
    case 'image':      await _handleMedia(json, 'image');
    case 'gif':        await _handleMedia(json, 'gif');
    case 'video':      await _handleMedia(json, 'video');
    case 'voice':      await _handleMedia(json, 'voice');
    case 'video_note': await _handleMedia(json, 'video_note');
    case 'file':       await _handleMedia(json, 'file');
    case 'chat_meta':  _handleChatMeta(json, frame.senderUUID);
    case 'message_read': _handleReadReceipt(json);
    case 'reaction':   _handleReaction(json);
  }
}
```

---

## 9. Сборка медиа (_PendingFile)

```dart
class _PendingFile {
  final String fileId;
  final int totalChunks;
  final String fileName;
  final String mimeType;
  final int totalSize;
  final String mediaType;   // "image", "video", etc.

  int receivedChunks = 0;
  int? chunkStride;          // размер chunk 0, фиксируется при первом приходе

  // In-memory (для изображений < порога)
  final List<Uint8List?> chunks;  // null = ещё не получен

  // On-disk (для видео, голоса, крупных файлов)
  RandomAccessFile? raf;
  String? tempPath;
}

Future<void> _handleMediaChunk(Map<String, dynamic> json) async {
  final fileId    = json['file_id'] as String;
  final chunkIdx  = json['chunk'] as int;
  final chunkData = base64Decode(json['data'] as String);

  _PendingFile pf = _pendingFiles[fileId] ??= _PendingFile.fromJson(json);

  if (pf.chunkStride == null && chunkIdx == 0) {
    pf.chunkStride = chunkData.length;
  }

  if (pf.raf != null) {
    // On-disk: seek к позиции и запись
    final offset = chunkIdx * pf.chunkStride!;
    await pf.raf!.setPosition(offset);
    await pf.raf!.writeFrom(chunkData);
  } else {
    // In-memory
    pf.chunks[chunkIdx] = chunkData;
  }

  pf.receivedChunks++;

  if (pf.receivedChunks == pf.totalChunks) {
    // Сборка завершена
    await _finalizeMedia(pf);
    _pendingFiles.remove(fileId);
  }
}

Future<void> _finalizeMedia(_PendingFile pf) async {
  Uint8List? bytes;
  String? localPath;

  if (pf.raf != null) {
    await pf.raf!.flush();
    await pf.raf!.close();
    localPath = pf.tempPath!;
  } else {
    // Склеиваем in-memory чанки
    final builder = BytesBuilder(copy: false);
    for (final chunk in pf.chunks) {
      builder.add(chunk!);
    }
    bytes = builder.toBytes();
  }

  final msg = ChatMessage(
    mediaType: pf.mediaType,
    imageBytes: (bytes != null && pf.mediaType == 'image') ? bytes : null,
    localMediaPath: localPath,
    // ...
  );
  emit(SgtpMessageReceived(message: msg));
}
```

---

## 10. Хранение истории (SQLite)

### 10.1 Схема

```sql
CREATE TABLE messages (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    message_uuid  BLOB UNIQUE NOT NULL,   -- 16 байт
    sender_uuid   BLOB NOT NULL,          -- 16 байт
    timestamp_ms  INTEGER NOT NULL,       -- Unix ms
    nonce         INTEGER NOT NULL,       -- uint64
    payload_json  TEXT NOT NULL,          -- расшифрованный JSON plaintext
    media_type    TEXT,                   -- "image", "video", etc. (nullable)
    local_path    TEXT                    -- путь к файлу на диске (nullable)
);

CREATE INDEX idx_messages_timestamp ON messages(timestamp_ms);
CREATE UNIQUE INDEX idx_messages_uuid ON messages(message_uuid);
```

Каждое сообщение хранится в расшифрованном виде (plaintext JSON). Это означает, что локальная история не защищена шифрованием уровня SGTP — безопасность на уровне SQLite обеспечивается только filesystem permissions / device encryption.

### 10.2 Дедупликация

При вставке используется `INSERT OR IGNORE` по `message_uuid`. Это защищает от:
- Двойной доставки при переподключении
- Совпадающих сообщений из History Sync и real-time потока

### 10.3 Загрузка истории при старте

```dart
Future<List<ChatMessage>> loadHistory({
  int limit = 50,
  int? beforeTimestampMs,
}) async {
  final rows = await _db.rawQuery('''
    SELECT * FROM messages
    WHERE timestamp_ms < ?
    ORDER BY timestamp_ms DESC
    LIMIT ?
  ''', [beforeTimestampMs ?? maxInt, limit]);
  return rows.map(ChatMessage.fromRow).toList().reversed.toList();
}
```

Сообщения загружаются постранично (по 50), от новых к старым.

---

## 11. Управление Chat Key на стороне клиента

### 11.1 Получение CHAT_KEY (не мастер)

```dart
Future<void> _onChatKey(ParsedFrame frame) async {
  // Парсим payload v2: epoch(8) + nonce(8) + ciphertext(48)
  final epoch     = readUint64(frame.payload, 0);
  final nonce64   = readUint64(frame.payload, 8);
  final ciphertext = frame.payload.sublist(16);

  // Получаем pairwise ключ с мастером
  final master = _peers[frame.senderUUID]!;
  final pairwiseKey = master.sharedKey!;

  // Дешифруем Chat Key
  final nonce96 = Uint8List(12)..setRange(4, 12, uint64ToBytes(nonce64));
  final chatKeyBytes = await chacha20Decrypt(ciphertext, pairwiseKey, nonce96);
  if (chatKeyBytes == null || chatKeyBytes.length != 32) return;

  // Применяем новый ключ
  _chatKey = chatKeyBytes;
  _currentEpoch = epoch;
  _myNonce = 0;  // ← сброс счётчика (источник проблемы nonce reuse)

  // Подтверждаем
  await _sendFrame(buildChatKeyAck(...));

  // Переходим в READY если ещё не были
  if (_state != ClientState.ready) {
    _state = ClientState.ready;
    emit(SgtpReady(isMaster: false));
  }
}
```

### 11.2 Ротация CHAT_KEY (мастер)

```dart
Timer? _ckRotationTimer;

void _startCkRotationTimer() {
  _ckRotationTimer = Timer.periodic(
    Duration(seconds: 180),
    (_) => _rotateChatKey(),
  );
}

Future<void> _rotateChatKey() async {
  _isRotating = true;  // Блокируем входящие MESSAGE
  _pendingAcks = Set.from(_peers.keys);

  // Генерируем новый Chat Key
  final newKey = Uint8List(32);
  fillRandom(newKey);  // SecureRandom

  final epoch = max(_currentEpoch + 1, DateTime.now().millisecondsSinceEpoch);

  // Отправляем каждому peer-у
  for (final peer in _peers.values) {
    final nonce64 = secureRandomUint64();
    final nonce96 = nonce64ToNonce96(nonce64);
    final ct = await chacha20Encrypt(newKey, peer.sharedKey!, nonce96);
    final payload = epochToBytes(epoch) + uint64ToBytes(nonce64) + ct;
    await _sendFrame(buildChatKey(peer.uuidBytes, payload));
  }

  // Ждём ACK-ов
  await _waitForAllAcks(timeout: Duration(seconds: 10));

  // Применяем новый ключ
  _chatKey = newKey;
  _currentEpoch = epoch;
  _myNonce = 0;
  _isRotating = false;
}
```

### 11.3 Обработка сообщений во время ротации

Если входящий MESSAGE приходит в момент `_isRotating == true`, мастер отправляет MESSAGE_FAILED:

```dart
if (_isMaster && _isRotating) {
  final failedPayload = await _encryptPairwise(
    peer.sharedKey!,
    msgUUID,  // UUID отклонённого сообщения
  );
  await _sendFrame(buildMessageFailed(peer.uuidBytes, failedPayload));
  return;
}
```

Отправитель, получив MESSAGE_FAILED, ждёт нового CHAT_KEY (который придёт в рамках текущей ротации) и повторяет отправку с новым ключом — до 3 раз.

---

## 12. Master election и переход ролей

```dart
String _computeMaster(List<String> allUUIDs) {
  // UUID в hex-строках, лексикографически минимальный
  return allUUIDs.reduce((a, b) => a.compareTo(b) < 0 ? a : b);
}

void _onPeerLeft(String departedUUID) {
  _peers.remove(departedUUID);
  final newMaster = _computeMaster([myUUID, ..._peers.keys]);
  if (newMaster == myUUID && !_isMaster) {
    _isMaster = true;
    _startCkRotationTimer();  // Берём на себя ротацию ключей
    _rotateChatKey();          // Немедленная ротация при смене мастера
    emit(SgtpReady(isMaster: true));
  }
}
```

При уходе текущего мастера новый мастер (следующий минимальный UUID) автоматически запускает ротацию Chat Key. Это гарантирует, что ушедший участник не сможет расшифровать будущие сообщения.

---

## 13. Kick механизм

В текущем Flutter runtime `KICK_REQUEST` не используется как активная клиентская ветка.
Клиент обрабатывает входящий `KICKED` (удаляет указанного peer из локального состояния).
Нормативный flow `KICK_REQUEST -> KICKED` остаётся частью протокола, но в приложении опора в основном на keepalive/stale-cleanup и `FIN`.

---

## 14. Конфигурационные константы

```dart
abstract class SgtpConstants {
  // Протокол
  static const int version            = 0x0002;
  static const int headerSize         = 64;
  static const int signatureSize      = 64;
  static const int maxPayloadSize     = 16 * 1024 * 1024; // 16 МиБ

  // Таймауты
  static const Duration pingTimeout       = Duration(seconds: 30);
  // peerStaleTimeout не фиксирован: вычисляется как pingIntervalSeconds × 3
  // (периодически) или × 4 (при получении INTENT); по умолчанию 90 с / 120 с
  static const Duration keepaliveInterval = Duration(seconds: 30);

  // Ключи
  static const Duration ckRotationInterval = Duration(seconds: 180);
  static const int      messageFailedRetries = 3;

  // Медиа
  static const int defaultMediaChunkSize = 100 * 1024;  // 100 КиБ
  static const int webMinChunkSize       = 512 * 1024;  // 512 КиБ
  static const int maxMediaChunks        = 9999;

  // Прогресс репортинг
  static const double progressDeltaThreshold = 0.03;    // 3%
  static const Duration progressMinInterval = Duration(milliseconds: 200);

  // HTTP
  static const int infoRequestDelay    = 1; // секунда после первого PING
}
```

---

## 15. Платформенные особенности

### 15.1 iOS / macOS

- `SecureSocket.connect()` по умолчанию использует системный trust store
- `badCertificateCallback` в HTTP discovery клиенте отключает проверку сертификата (**уязвимость**)
- Запись аудио: AVAudioRecorder через `record` пакет
- Крупные медиа-файлы кэшируются в `getTemporaryDirectory()`

### 15.2 Android

- Аналогично iOS для сети
- Запись аудио: MediaRecorder через `record` пакет
- `getExternalStorageDirectory()` / `getApplicationCacheDirectory()` для кэша

### 15.3 Web (браузер)

- Dart `dart:io` недоступен → используется `dart:html`
- WebSocket через `html.WebSocket`
- `dart:crypto` заменяется на WASM-скомпилированные реализации
- Минимальный чанк увеличен до 512 КиБ (накладные расходы на Uint8Array <-> JS)
- File API: `html.File` / `html.FileReader`
- Сохранение файлов: download через `html.AnchorElement`

### 15.4 Desktop (Windows / Linux / macOS)

- `dart:io` доступен, TCP transport
- Файловый выбор через `file_picker`
- Уведомления через `flutter_local_notifications`
