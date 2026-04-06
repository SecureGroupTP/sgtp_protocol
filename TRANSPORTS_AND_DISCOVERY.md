# Транспорты и Discovery

## SGTP payload одинаков, транспорт разный

Поддерживаются варианты:

1. `TCP`
2. `TCP+TLS`
3. `HTTP`
4. `HTTP+TLS`
5. `WebSocket`
6. `WebSocket+TLS`

## Discovery форматы

Сейчас в проекте одновременно есть два представления:

1. Legacy binary: 25-байтный баннер (`flags + 6 ports`).
2. HTTP JSON: `GET /sgtp/discovery`.

Сервер:

- На raw TCP соединении отправляет 25 байт сразу после accept.
- На HTTP/HTTPS отдаёт JSON discovery.
- `?format=raw|binary` отдаёт те же 25 байт как octet-stream.

Клиент Flutter:

- Для discover использует HTTP `GET /sgtp/discovery` с fallback-порядком портов:
  1. `https://<host>:443/sgtp/discovery`
  2. `http://<host>:80/sgtp/discovery`
  3. `http://<host>:77/sgtp/discovery`
- Для TCP транспорта при connect сначала читает и отбрасывает 25-байтный баннер, затем начинает SGTP frame stream.

## HTTP transport (session protocol)

1. `POST /sgtp/session` -> sid (16 raw bytes)
2. `POST /sgtp/send?sid=<hex32>` -> отправка frame bytes
3. `GET /sgtp/recv?sid=<hex32>` -> поток входящих bytes
4. `DELETE /sgtp/session?sid=<hex32>` -> закрытие сессии

## WebSocket transport

- Передаются бинарные сообщения с SGTP-байтами.
- Возможны несколько конкатенированных SGTP фреймов в одном WS сообщении.
- Поддерживаются `ws://` и `wss://`.
