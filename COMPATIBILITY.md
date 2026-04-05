# Совместимость и Различия Реализаций

## Версия протокола

- `sgtp-go` пакетный слой: базово `0x0001`.
- Flutter константа: `0x0002`.

## Ключевые отличия

1. `CHAT_KEY`:
- `v1`: nonce для расшифровки = `epoch`.
- `v2`: в payload добавлен явный `nonce`.

2. Shared-key пакеты (`STATUS`, `MESSAGE_FAILED` и др.):
- `v1`: nonce обычно выводится из `timestamp`.
- `v2` (Flutter): nonce может передаваться первым uint64 в payload.

3. `CHAT_REQUEST`:
- `v1`: только список UUID.
- Flutter extension: `chat_name` и `chat_avatar` в payload.

4. `INTENT`:
- В базовой спецификации это вход в комнату.
- Во Flutter дополнительно используется как keepalive при пустой комнате.

5. Discovery:
- Документ `flutter/SERVER_SPEC` описывает legacy TCP discovery.
- Текущая реализация сервера и клиента также использует HTTP JSON discovery.

## Практическое правило интеграции

1. Парсить `version` из header и поддерживать минимум `v1` и `v2` ветки.
2. Для `CHAT_KEY` и shared-key пакетов выбирать схему nonce по версии.
3. Не предполагать, что `CHAT_REQUEST` payload фиксирован только в старом формате.
4. Для discovery поддерживать и JSON (`/sgtp/discovery`), и 25-байтный бинарный формат.
