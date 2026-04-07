# Совместимость и Различия Реализаций

## Версия протокола

- `sgtp-go` пакетный слой: базово `0x0001`.
- Flutter константа: `0x0002` (актуальная рабочая ветка).

Важно: в текущем `SgtpClient` входящий кадр отбрасывается, если `frame.version != 0x0002`.
То есть фактическая межверсионная совместимость клиента сейчас строгая: `v2-only`.

## Ключевые отличия

1. `CHAT_KEY`:
- `v1`: nonce для расшифровки = `epoch`.
- `v2`: в payload добавлен явный `nonce`.

2. `CHAT_KEY_ACK`:
- `v1`: пустой payload.
- `v2`: `epoch(uint64)` в payload.

3. Shared-key пакеты (`STATUS`, `MESSAGE_FAILED` и др.):
- `v1`: nonce обычно выводится из `timestamp`.
- `v2` (Flutter): nonce может передаваться первым uint64 в payload.

4. `CHAT_REQUEST`:
- `v1`: только список UUID.
- Flutter extension: `chat_name` и `chat_avatar` в payload.

5. `INTENT`:
- В базовой спецификации это вход в комнату.
- Во Flutter дополнительно используется как keepalive при пустой комнате.
- На стороне мастера `INTENT` также используется как триггер `ensureChatKey(peer)` для досылки ключа текущей эпохи при рассинхроне ACK.

6. Discovery:
- Документ `flutter/SERVER_SPEC` описывает legacy TCP discovery.
- Текущая реализация сервера и клиента также использует HTTP JSON discovery.

## Практическое правило интеграции

1. Для Flutter-клиента используйте `version = 0x0002`.
2. Для `CHAT_KEY` и shared-key пакетов используйте `v2`-схему nonce (явный uint64 в payload).
3. Для `CHAT_KEY_ACK` отправляйте `epoch(uint64)`; при приёме поддерживайте и `v1` (пусто), и `v2`.
4. Не предполагайте, что `CHAT_REQUEST` payload фиксирован только в старом формате.
5. Для discovery поддерживайте JSON (`/sgtp/discovery`) и legacy 25-байтный бинарный формат.
