# Карта Исходников

## Основные спецификации

- `sgtp-go/docs/spec.md`
- `flutter/SERVER_SPEC`

## Wire и packet model

- `sgtp-go/protocol/packet.go`
- `sgtp-go/protocol/parser.go`
- `server/protocol/packet.go`
- `flutter/lib/core/protocol/packet_types.dart`
- `flutter/lib/core/protocol/frame_builder.dart`
- `flutter/lib/core/protocol/frame_parser.dart`

## Клиентская логика протокола

- `sgtp-go/client/client.go`
- `sgtp-go/client/handshake.go`
- `sgtp-go/client/send.go`
- `sgtp-go/client/session.go`
- `sgtp-go/client/master.go`
- `flutter/lib/features/messaging/data/services/sgtp_client.dart`

## Сервер и транспорт

- `sgtp-go/server/server.go`
- `server/server/server.go`
- `server/internal/mtransport/ports.go`
- `server/internal/mtransport/multi.go`
- `server/internal/mtransport/http_sessions.go`
- `server/internal/mtransport/ws.go`
- `flutter/lib/features/messaging/data/transport/server_discovery.dart`
- `flutter/lib/features/messaging/data/transport/tcp_sgtp_transport.dart`
- `flutter/lib/features/messaging/data/transport/http_sgtp_transport.dart`
- `flutter/lib/features/messaging/data/transport/websocket_sgtp_transport_io.dart`
- `flutter/lib/features/messaging/data/transport/websocket_sgtp_transport_web.dart`
