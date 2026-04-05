# Глоссарий

- `Frame` — единица передачи SGTP: заголовок + payload + подпись.
- `Header` — фиксированные 64 байта с адресацией, типом пакета, длиной payload и timestamp.
- `Payload` — тип-специфичная часть кадра.
- `Signature` — ed25519 подпись кадра (64 байта, всегда в конце).
- `Room UUID` — UUID комнаты, общий контекст чата.
- `Sender UUID` — UUID отправителя кадра.
- `Receiver UUID` — UUID адресата; `0000..0000` означает broadcast.
- `Broadcast UUID` — 16 нулевых байт (`BROADCAST_UUID`).
- `Master` — участник с наименьшим UUID в комнате.
- `Intent` — header-only кадр (`packet_type=0`, `payload_length=0`) для объявления о входе/keepalive.
- `Shared Secret` — результат x25519 DH между двумя peer.
- `Shared Key` — в Flutter v2 производный ключ из shared secret + room context.
- `Chat Key (CK)` — групповой симметричный ключ для `MESSAGE/FIN`.
- `Epoch` — идентификатор поколения Chat Key.
- `Nonce` — AEAD nonce (uint64) для шифрования полезной нагрузки.
- `HSIR/HSI/HSR/HSRA` — пакетная синхронизация истории.
- `CK Rotation Window` — период, когда мастер ждёт `CHAT_KEY_ACK` от всех и отклоняет `MESSAGE`.
