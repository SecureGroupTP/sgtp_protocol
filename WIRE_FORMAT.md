# Базовый Wire Format

## Формат кадра

```text
[64-byte Header][N-byte Payload][64-byte Signature]
```

- Multi-byte числа: `big-endian`.
- Минимальный размер кадра: `128` байт (`64 + 64`).
- `payload_length` ограничен `MAX_PAYLOAD_LENGTH = 16 MiB`.

## Header (64 bytes)

1. `room_uuid` (16)
2. `receiver_uuid` (16)
3. `sender_uuid` (16)
4. `version` (uint16)
5. `packet_type` (uint16)
6. `payload_length` (uint32)
7. `timestamp` (uint64, Unix ms UTC)

## Валидации

1. `payload_length <= 16 MiB`.
2. Общая длина кадра должна совпадать: `64 + payload + 64`.
3. `timestamp` должен попадать в окно `TIMESTAMP_WINDOW` (30s).
4. Подпись должна проходить проверку публичным ключом отправителя.
5. Для Flutter-клиента дополнительно: `version` в header должен быть ровно `0x0002`, иначе кадр отбрасывается.

## Intent frame

Используется `packet_type=0x0000` и пустой payload.

- На сервере это триггер «новый peer подключился».
- На клиентах это триггер отправки `PING`.
- Во Flutter также используется как keepalive, когда peer-ов нет.
