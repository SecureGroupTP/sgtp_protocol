# Типы Пакетов

## Таблица кодов

| Type | Code |
|---|---|
| INTENT | `0x0000` |
| PING | `0x0001` |
| PONG | `0x0002` |
| INFO | `0x0003` |
| CHAT_REQUEST | `0x0004` |
| CHAT_KEY | `0x0005` |
| CHAT_KEY_ACK | `0x0006` |
| MESSAGE | `0x0007` |
| MESSAGE_FAILED | `0x0008` |
| MESSAGE_FAILED_ACK | `0x0009` |
| STATUS | `0x000A` |
| HSIR | `0x000B` |
| HSI | `0x000C` |
| HSR | `0x000D` |
| HSRA | `0x000E` |
| FIN | `0x000F` |
| KICK_REQUEST | `0x0010` |
| KICKED | `0x0011` |

## Кратко по payload

- `PING/PONG`: `x25519_pub(32) + ed25519_pub(32) + "Client Hello"`.
- `INFO(req)`: пусто.
- `INFO(resp)`: `count(uint64) + count*uuid(16)`.
- `CHAT_REQUEST`:
  - `v1`: `count + uuids`.
  - Flutter extension: дополнительно `chat_name_length(uint32) + chat_name(UTF-8) + avatar_length(uint32) + avatar_bytes`.
  - Ограничение, которое реально валидируется на клиенте: `chat_name <= 255` UTF-8 байт.
- `CHAT_KEY`:
  - `v1`: `epoch(8) + ciphertext(48)`.
  - `v2` (Flutter): `epoch(8) + nonce(8) + ciphertext(48)`.
- `CHAT_KEY_ACK`: пусто.
- `MESSAGE`: `message_uuid(16) + nonce(8) + ciphertext(...)`.
- `MESSAGE_FAILED`: зашифрованный UUID неуспешного сообщения.
- `MESSAGE_FAILED_ACK`: пусто.
- `STATUS`: зашифрованный статус (`code + text`).
- `HSIR`: пусто.
- `HSI`: `message_count(uint64)`.
- `HSR`: `offset(uint64) + limit(uint64)`.
- `HSRA`: `batch_number + message_count + offsets + blob(messages)`.
- `FIN`: `nonce(8) + tag(16)` (тег от пустого plaintext).
- `KICK_REQUEST/KICKED`: `target_uuid(16)`.
