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

## Подробная документация

Детальные описания алгоритмов, пайплайнов и реализаций находятся в подпапке [`detailed/`](./detailed/):

- [**PIPELINE.md**](./detailed/PIPELINE.md) — Полный пайплайн передачи для каждого типа контента: текст, изображения, голосовые, видео, видеозаметки, документы. Сборка чанков, прогресс, read receipts, реакции, History Sync.
- [**CRYPTOGRAPHY.md**](./detailed/CRYPTOGRAPHY.md) — Детальная криптографическая модель: Ed25519, X25519, ChaCha20-Poly1305, DH-обмен, деривация ключей, nonce-схемы, жизненный цикл Chat Key, известные уязвимости.
- [**SERVER.md**](./detailed/SERVER.md) — Внутреннее устройство relay-сервера: мульти-транспорт, маршрутизация, HTTP-сессии, User Directory, PostgreSQL схема, все лимиты и конфигурация.
- [**CLIENT.md**](./detailed/CLIENT.md) — Flutter-клиент: архитектура, state machine, обработка медиа, SQLite история, управление Chat Key, мастер-элекшн, кик, платформенные особенности.

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
