# SGTP Protocol Documentation

Это централизованная документация по SGTP-протоколу и его транспорту, собранная из `sgtp-go`, `server` и `flutter`.

## Навигация

1. [Глоссарий](./GLOSSARY.md)
2. [Базовый wire format](./WIRE_FORMAT.md)
3. [Типы пакетов](./PACKETS.md)
4. [Хендшейк и установление сессии](./HANDSHAKE.md)
5. [Сообщения, ключи и ротации](./MESSAGING_AND_KEYS.md)
6. [История, FIN и KICK](./HISTORY_FIN_KICK.md)
7. [Транспорты и discovery](./TRANSPORTS_AND_DISCOVERY.md)
8. [Совместимость и различия реализаций](./COMPATIBILITY.md)
9. [Карта исходников](./SOURCE_MAP.md)

## Что это за версия документации

- Актуальная на основе кода в текущем репозитории.
- Нормативной базой считается wire-структура из `sgtp-go/protocol` и рабочая логика клиентов/сервера.
- Исторические спеки сохранены в `protocol/reference`.

## Быстрый обзор

- SGTP кадр: `Header(64) + Payload(N) + Signature(64)`.
- Подпись: `ed25519` по всему кадру без последних 64 байт подписи.
- KEX: `x25519` (PING/PONG).
- Шифрование: `ChaCha20-Poly1305`.
- Роли: peer и master (минимальный UUID).
- Основные группы пакетов: handshake, key management, message delivery, history sync, lifecycle.
