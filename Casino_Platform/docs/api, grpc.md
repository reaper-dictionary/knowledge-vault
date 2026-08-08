
# Gateway ↔ Casino Service: контракты первой недели

## 1. Цель и границы недели

Цель первой недели — получить walking skeleton комнат:

```
авторизованный клиент
→ Gateway REST/WebSocket
→ Casino gRPC
→ Casino PostgreSQL
→ Redis snapshot/PubSub
→ подтверждённое состояние возвращается клиентам
```

К концу недели должны работать:

- создание, список, чтение и закрытие комнат;
    
- WebSocket-аутентификация;
    
- вход и явный выход из комнаты;
    
- потеря соединения, reconnect grace period и восстановление полного snapshot;
    
- broadcast подтверждённых изменений другим участникам;
    
- идемпотентность mutating-команд;
    
- обработка duplicate request, timeout и partial failure.
    

В эту границу **не входят** ставки, Wallet, раунды, игровые действия, fairness, Kafka-события, GraphQL, leaderboard и достижения. Их нельзя добавлять, пока сценарий комнат не проходит интеграционный тест целиком.

## 2. Зафиксированные архитектурные решения

1. Клиент обращается только к Gateway. Casino Service наружу не публикуется.
    
2. REST используется для lobby/query и lifecycle комнаты; realtime membership — через один WebSocket.
    
3. Gateway проверяет access token, rate limit и формат сообщения, но не решает, можно ли войти в комнату или закрыть её.
    
4. Casino является source of truth для `CasinoPlayer`, `Room` и `RoomParticipant`.
    
5. Gateway передаёт в Casino только `identity_user_id` из уже проверенного token context. `user_id` из client payload запрещён.
    
6. `create_public_room` и `create_private_room` — один `CreateRoom` с `visibility`.
    
7. Физического `remove_room` нет. Используется `CloseRoom`, позднее — архивирование.
    
8. `FULL` — не состояние Room, а вычисляемый признак `active_participants >= capacity`.
    
9. Casino — единственный writer `RoomStateCache` и publisher подтверждённых room events в Redis Pub/Sub. Gateway владеет только соединениями, subscriptions и presence.
    
10. PostgreSQL Casino — источник истины. Redis можно полностью очистить и восстановить.
    
11. Потеря WebSocket не равна явному выходу из комнаты. Участнику даётся 60 секунд на reconnect; окончательное решение принимает Casino.
    
12. В первой неделе пользователь создаёт комнаты Dice Duel. Общие комнаты Crash/Roulette позднее создаёт scheduler/seed внутри Casino, не публичная ручка.
    

## 3. Разделение работы

|Участник|Ответственность первой недели|
|---|---|
|Второй backend|Casino domain, миграции, gRPC server, все room use cases, state transitions, idempotency, Redis snapshot/PubSub, unit и integration tests|
|Team Lead|REST/WS Gateway, проверка токена, единый envelope, gRPC client, deadlines/retry, connection registry, Redis presence и доставка Pub/Sub событий в сокеты|
|Frontend|REST client lobby, один WebSocket client, authenticate/join/leave/resume, хранение `last_revision`, применение snapshot и reconnect UI|

Первый gRPC client и общую инфраструктуру proto создаёт Team Lead. Второй backend начинает интеграцию с этим примером только после merge и короткого разбора контракта.

## 4. Публичные ручки Gateway

Базовый prefix: `/api/v1/casino`.

|   |   |   |   |   |
|---|---|---|---|---|
|Transport|Публичная точка|Auth|Назначение|Внутренний вызов Casino|
|REST|`POST /rooms`|да|Создать player-owned Dice room|`CreateRoom`|
|REST|`GET /rooms`|да|Получить cursor-paginated lobby|`ListRooms`|
|REST|`GET /rooms/{room_id}`|да|Получить authoritative snapshot|`GetRoomSnapshot`|
|REST|`POST /rooms/{room_id}/close`|да|Закрыть комнату owner/admin-командой|`CloseRoom`|
|WebSocket|`GET /ws`|после upgrade|Открыть единственный realtime-канал|дальнейшие WS commands|

Отдельные operational endpoints Gateway, не относящиеся к Casino domain:

- `GET /health/live`;
    
- `GET /health/ready`.
    

### 4.1 `POST /api/v1/casino/rooms`

Headers:

```
Authorization: Bearer <access-token>
Idempotency-Key: <uuid-or-ulid>
```

Request:

```
{
  "name": "Room 17",
  "game_type": "DICE_DUEL",
  "visibility": "PUBLIC",
  "capacity": 2
}
```

Правила:

- в первой неделе разрешён только `DICE_DUEL`;
    
- `capacity` для Dice Duel равен `2` и дополнительно проверяется Casino;
    
- для `PRIVATE` Casino генерирует случайный `invite_token`, хранит только его hash и возвращает token только создателю в ответе;
    
- `owner_player_id` вычисляет Casino по проверенному `identity_user_id`;
    
- повтор с тем же `Idempotency-Key` и тем же payload возвращает первоначальный результат;
    
- тот же key с другим payload возвращает `IDEMPOTENCY_KEY_REUSED`.
    

Response `201 Created`:

```
{
  "room": {
    "id": "0194c9af-4ee8-7a31-b4de-1fe46f15a2ab",
    "name": "Room 17",
    "game_type": "DICE_DUEL",
    "visibility": "PUBLIC",
    "status": "OPEN",
    "capacity": 2,
    "participants_count": 0,
    "revision": 1,
    "owner_player_id": "0194c9ae-7ef2-7f5c-b390-f284254ef807",
    "created_at": "2026-08-08T10:00:00Z"
  },
  "invite_token": null
}
```

### 4.2 `GET /api/v1/casino/rooms`

Query parameters:

- `game_type=DICE_DUEL` — optional;
    
- `status=OPEN` — default `OPEN`;
    
- `visibility=PUBLIC` — default `PUBLIC`;
    
- `limit=20` — `1..100`;
    
- `cursor=<opaque>` — optional.
    

Response `200 OK`:

```
{
  "items": [
    {
      "id": "0194c9af-4ee8-7a31-b4de-1fe46f15a2ab",
      "name": "Room 17",
      "game_type": "DICE_DUEL",
      "visibility": "PUBLIC",
      "status": "OPEN",
      "capacity": 2,
      "participants_count": 1,
      "revision": 4
    }
  ],
  "next_cursor": null
}
```

Private rooms не попадают в общий lobby. В первой версии владелец или участник открывает такую комнату по `room_id`; token проверяется только при первом `room.join`.

### 4.3 `GET /api/v1/casino/rooms/{room_id}`

Возвращает `RoomSnapshot`, а не произвольную копию из памяти Gateway.

```
{
  "room": {
    "id": "0194c9af-4ee8-7a31-b4de-1fe46f15a2ab",
    "name": "Room 17",
    "game_type": "DICE_DUEL",
    "visibility": "PUBLIC",
    "status": "OPEN",
    "capacity": 2,
    "participants_count": 1,
    "revision": 4
  },
  "participants": [
    {
      "player_id": "0194c9ae-7ef2-7f5c-b390-f284254ef807",
      "nickname": "player-17",
      "avatar_url": null,
      "membership_status": "ACTIVE",
      "connection_status": "CONNECTED",
      "seat": 1,
      "joined_at": "2026-08-08T10:01:00Z"
    }
  ],
  "server_time": "2026-08-08T10:02:00Z"
}
```

Casino сам решает, читать snapshot из Redis или восстанавливать из PostgreSQL. Gateway не принимает authorization/business decisions по cached snapshot.

### 4.4 `POST /api/v1/casino/rooms/{room_id}/close`

Headers:

```
Authorization: Bearer <access-token>
Idempotency-Key: <uuid-or-ulid>
```

Body отсутствует. Закрыть player-owned room может владелец или пользователь с permission `casino:rooms:manage`. Повторное закрытие успешно возвращает уже закрытый snapshot. Участники получают `room.closed`.

### 4.5 Почему нет других REST-ручек

- Нет `DELETE /rooms/{id}`: история не удаляется физически.
    
- Нет отдельных `create_public_room` и `create_private_room`: это один use case.
    
- Нет REST `join/leave`: иначе появятся два конкурирующих transport-контракта для одного realtime-сценария.
    
- Нет `start_game`, `create_bet`, `change_bet`, `remove_bet`: это не scope первой недели; изменение принятой ставки вообще не должно проектироваться как простой CRUD update.
    

## 5. WebSocket-протокол Gateway

Native browser WebSocket не позволяет надёжно передать произвольный `Authorization` header. Для первой версии:

1. клиент открывает `wss://host/api/v1/casino/ws`;
    
2. первым сообщением за 5 секунд отправляет `session.authenticate`;
    
3. Gateway проверяет token и никогда не пишет его в logs;
    
4. token запрещено передавать в query string;
    
5. неавторизованный socket получает close code `4401`, auth timeout — `4408`.
    

### 5.1 Command envelope клиента

```
{
  "type": "room.join",
  "protocol_version": 1,
  "request_id": "0194c9b2-b7f0-7cd1-adb3-5a18bd8ffb0a",
  "payload": {
    "room_id": "0194c9af-4ee8-7a31-b4de-1fe46f15a2ab",
    "invite_token": null
  }
}
```

### 5.2 Event envelope сервера

```
{
  "type": "room.snapshot",
  "protocol_version": 1,
  "event_id": "0194c9b3-30df-75c1-b29d-6ab8804d7e8f",
  "request_id": "0194c9b2-b7f0-7cd1-adb3-5a18bd8ffb0a",
  "room_id": "0194c9af-4ee8-7a31-b4de-1fe46f15a2ab",
  "revision": 5,
  "server_time": "2026-08-08T10:03:00Z",
  "payload": {}
}
```

`request_id` связывает ответ с командой. `event_id` устраняет повторную доставку. `revision` задаёт порядок room state. Если клиент ожидал revision `8`, а получил `10`, он не угадывает пропущенное состояние, а отправляет `session.resume` и заменяет local state полным snapshot.

### 5.3 Client commands

|   |   |   |   |
|---|---|---|---|
|`type`|Payload|Что делает Gateway|Casino RPC|
|`session.authenticate`|`access_token`|Проверяет token, связывает socket с user|нет|
|`room.join`|`room_id`, optional `invite_token`|Проверяет envelope/rate limit; после успеха добавляет subscription|`JoinRoom`|
|`room.leave`|`room_id`|После успеха удаляет subscription|`LeaveRoom`|
|`session.resume`|`room_id`, `last_revision`|Восстанавливает membership и всегда отдаёт полный snapshot|`ResumeRoomSession`|

### 5.4 Server events

|   |   |   |
|---|---|---|
|`type`|Получатель|Назначение|
|`session.authenticated`|конкретный socket|Аутентификация завершена|
|`room.joined`|инициатор|Join принят; содержит полный snapshot|
|`room.left`|инициатор|Явный leave принят|
|`room.snapshot`|инициатор|Полное authoritative состояние после resume/resync|
|`room.participant_joined`|room subscribers|Участник вошёл|
|`room.participant_disconnected`|room subscribers|Последний socket участника потерян, grace period начался|
|`room.participant_reconnected`|room subscribers|Участник восстановил membership|
|`room.participant_left`|room subscribers|Явный выход или истечение grace period|
|`room.closed`|room subscribers|Комната закрыта|
|`command.rejected`|инициатор|Валидная по протоколу команда отклонена|
|`protocol.error`|конкретный socket|JSON/envelope/type не соответствует протоколу|

## 6. Внутренний gRPC-контракт Gateway → Casino

```
service CasinoRoomService {
  rpc CreateRoom(CreateRoomRequest) returns (CreateRoomResponse);
  rpc ListRooms(ListRoomsRequest) returns (ListRoomsResponse);
  rpc GetRoomSnapshot(GetRoomSnapshotRequest) returns (GetRoomSnapshotResponse);
  rpc JoinRoom(JoinRoomRequest) returns (JoinRoomResponse);
  rpc LeaveRoom(LeaveRoomRequest) returns (LeaveRoomResponse);
  rpc CloseRoom(CloseRoomRequest) returns (CloseRoomResponse);
  rpc MarkParticipantDisconnected(MarkParticipantDisconnectedRequest)
      returns (MarkParticipantDisconnectedResponse);
  rpc ResumeRoomSession(ResumeRoomSessionRequest)
      returns (ResumeRoomSessionResponse);
}
```

|   |   |   |   |
|---|---|---|---|
|RPC|Кто вызывает|Изменяет PostgreSQL|Результат|
|`CreateRoom`|REST adapter|да|Room snapshot и optional one-time invite token|
|`ListRooms`|REST adapter|нет|Страница `RoomSummary`|
|`GetRoomSnapshot`|REST adapter/resync|нет|Полный authoritative snapshot|
|`JoinRoom`|WS command handler|да|Membership + snapshot + event|
|`LeaveRoom`|WS command handler|да|Обновлённый snapshot + event|
|`CloseRoom`|REST adapter|да|Closed snapshot + event|
|`MarkParticipantDisconnected`|Gateway при потере последнего socket user-room|да|Deadline + event; это факт транспорта, не решение об удалении участника|
|`ResumeRoomSession`|`session.resume`|да|CONNECTED membership + полный snapshot + event|

Каждый request содержит общий context:

```
message RequestContext {
  string request_id = 1;
  string actor_identity_user_id = 2;
  repeated string permissions = 3;
}
```

Правила context:

- `request_id` обязателен для mutating RPC;
    
- Gateway берёт actor и permissions только из проверенного access token;
    
- client payload не может переопределить actor;
    
- `trace_id`/`correlation_id` передаются через gRPC metadata;
    
- в production требуется service-to-service authentication; одной внутренней сети недостаточно.
    

Рекомендуемые deadlines для локального этапа:

- query RPC: `1 s`;
    
- command RPC: `2 s`;
    
- значения конфигурируются, а не вшиваются в handler.
    

Gateway может один раз retry `UNAVAILABLE`/`DEADLINE_EXCEEDED` только с тем же `request_id`. Timeout означает «результат неизвестен», а не «Casino точно ничего не изменил».

## 7. Sequence diagrams

### 7.1 Создание комнаты

```
sequenceDiagram
    actor Client
    participant Gateway
    participant Casino
    participant DB as Casino PostgreSQL
    participant Redis

    Client->>Gateway: POST /rooms + Idempotency-Key
    Gateway->>Gateway: Verify token, schema, rate limit
    Gateway->>Casino: CreateRoom(request_id, actor, data)
    Casino->>DB: TX: ensure CasinoPlayer, create Room,<br/>save ProcessedCommand + event_id
    DB-->>Casino: COMMIT revision=1
    Casino->>Redis: SET RoomStateCache(revision=1)
    Casino->>Redis: PUBLISH room.created(event_id, revision=1)
    Casino-->>Gateway: RoomSnapshot + optional invite_token
    Gateway-->>Client: 201 Created

    Note over Gateway,Casino: Duplicate с тем же key возвращает сохранённый результат.
    Note over Casino,DB: Если response потерян после COMMIT,<br/>retry не создаёт вторую Room.
    Note over Casino,Redis: Ошибка cache/PubSub не откатывает DB;<br/>Casino ставит retry и snapshot остаётся восстанавливаемым.
```

### 7.2 Список и snapshot комнаты

```
sequenceDiagram
    actor Client
    participant Gateway
    participant Casino
    participant Redis
    participant DB as Casino PostgreSQL

    alt GET /rooms
        Client->>Gateway: GET /rooms?cursor=...
        Gateway->>Casino: ListRooms(filters, cursor)
        Casino->>DB: Cursor query + access rules
        DB-->>Casino: RoomSummary page
        Casino-->>Gateway: items + next_cursor
        Gateway-->>Client: 200 OK
    else GET /rooms/{room_id}
        Client->>Gateway: GET /rooms/{room_id}
        Gateway->>Casino: GetRoomSnapshot(actor, room_id)
        Casino->>Casino: Authorize visibility/membership
        Casino->>Redis: GET cached snapshot
        alt Cache miss or stale
            Casino->>DB: Load Room + active participants
            DB-->>Casino: Authoritative state
            Casino->>Redis: SET rebuilt snapshot
        end
        Casino-->>Gateway: RoomSnapshot(revision)
        Gateway-->>Client: 200 OK
    end

    Note over Gateway,Casino: Query timeout можно безопасно retry.
    Note over Casino,Redis: Cache никогда не обходит Casino authorization.
```

### 7.3 WebSocket authenticate и join

```
sequenceDiagram
    actor Client
    participant Gateway
    participant Casino
    participant DB as Casino PostgreSQL
    participant Redis

    Client->>Gateway: WebSocket upgrade /ws
    Client->>Gateway: session.authenticate(access_token)
    Gateway->>Gateway: Verify token; bind socket to user
    Gateway-->>Client: session.authenticated
    Client->>Gateway: room.join(request_id, room_id, invite_token?)
    Gateway->>Casino: JoinRoom(request_id, actor, room_id)
    Casino->>DB: TX: ensure CasinoPlayer,<br/>validate room/capacity/invite,<br/>upsert membership + ProcessedCommand
    DB-->>Casino: COMMIT revision=N
    Casino->>Redis: SET snapshot revision=N
    Casino->>Redis: PUBLISH participant_joined(event_id, N)
    Casino-->>Gateway: JoinRoomResponse + full snapshot
    Gateway->>Gateway: Add socket subscription
    Gateway-->>Client: room.joined(snapshot, revision=N)
    Redis-->>Gateway: Authoritative room event
    Gateway-->>Client: room.participant_joined(event_id, N)

    Note over Gateway,Casino: Повтор той же команды возвращает тот же результат.
    Note over Casino,DB: Unique active membership не допускает второго участника для того же player.
    Note over Gateway,Client: Event duplicate удаляется по event_id;<br/>revision gap вызывает full snapshot.
```

### 7.4 Явный выход

```
sequenceDiagram
    actor Client
    participant Gateway
    participant Casino
    participant DB as Casino PostgreSQL
    participant Redis

    Client->>Gateway: room.leave(request_id, room_id)
    Gateway->>Casino: LeaveRoom(request_id, actor, room_id)
    Casino->>DB: TX: ACTIVE → LEFT,<br/>left_at, revision++, ProcessedCommand
    DB-->>Casino: COMMIT
    Casino->>Redis: SET new snapshot
    Casino->>Redis: PUBLISH participant_left(event_id, revision)
    Casino-->>Gateway: LeaveRoomResponse
    Gateway->>Gateway: Remove socket subscription
    Gateway-->>Client: room.left

    Note over Gateway,Casino: Leave уже вышедшего player — успешная idempotent операция.
    Note over Casino,Redis: Если Pub/Sub потерян, другие клиенты восстановятся по revision/snapshot.
```

### 7.5 Disconnect, grace period и reconnect

```
sequenceDiagram
    actor Client
    participant Gateway
    participant Casino
    participant Redis
    participant DB as Casino PostgreSQL

    Client-xGateway: TCP/WebSocket disconnected
    Gateway->>Redis: Delete connection; presence TTL
    Gateway->>Casino: MarkParticipantDisconnected(request_id, room_id)
    Casino->>DB: TX: connection_status=DISCONNECTED,<br/>reconnect_deadline=now+60s, revision++
    DB-->>Casino: COMMIT
    Casino->>Redis: SET snapshot + PUBLISH disconnected

    Client->>Gateway: New WebSocket + session.authenticate
    Gateway-->>Client: session.authenticated
    Client->>Gateway: session.resume(request_id, room_id, last_revision)
    Gateway->>Casino: ResumeRoomSession(request_id, actor, room_id)
    Casino->>DB: TX: validate ACTIVE + deadline,<br/>connection_status=CONNECTED, revision++
    DB-->>Casino: COMMIT
    Casino->>Redis: SET snapshot + PUBLISH reconnected
    Casino-->>Gateway: Full RoomSnapshot
    Gateway->>Gateway: Restore subscription
    Gateway-->>Client: room.snapshot(full state)

    Note over Gateway,Casino: Disconnect не вызывает LeaveRoom.
    Note over Casino,DB: После deadline Casino job переводит membership в LEFT.
    Note over Gateway,Casino: Если Gateway умер до MarkDisconnected,<br/>Casino cleanup сверяет истёкшую Redis presence и закрывает stale membership.
```

### 7.6 Закрытие комнаты

```
sequenceDiagram
    actor Client
    participant Gateway
    participant Casino
    participant DB as Casino PostgreSQL
    participant Redis

    Client->>Gateway: POST /rooms/{id}/close + Idempotency-Key
    Gateway->>Casino: CloseRoom(request_id, actor, room_id)
    Casino->>DB: TX: authorize owner/permission,<br/>OPEN → CLOSED, participants → LEFT,<br/>revision++, ProcessedCommand
    DB-->>Casino: COMMIT
    Casino->>Redis: SET closed snapshot
    Casino->>Redis: PUBLISH room.closed(event_id, revision)
    Casino-->>Gateway: Closed RoomSnapshot
    Gateway-->>Client: 200 OK
    Redis-->>Gateway: room.closed
    Gateway-->>Client: Broadcast room.closed

    Note over Gateway,Casino: Повторное CloseRoom возвращает CLOSED snapshot.
    Note over Casino,DB: CLOSED room больше не принимает JoinRoom.
```

## 8. State machines первой недели

### 8.1 Room

```
stateDiagram-v2
    [*] --> OPEN: CreateRoom committed
    OPEN --> CLOSED: CloseRoom
    CLOSED --> ARCHIVED: retention job (не в первой неделе)
    ARCHIVED --> [*]

    note right of OPEN
      FULL — derived flag,
      а не отдельное состояние
    end note
```

### 8.2 RoomParticipant

В таблице лучше хранить два независимых поля: `membership_status=ACTIVE|LEFT` и `connection_status=CONNECTED|DISCONNECTED`. Диаграмма ниже показывает допустимые комбинации как единый lifecycle.

```
stateDiagram-v2
    [*] --> CONNECTED: JoinRoom
    CONNECTED --> DISCONNECTED: Last socket disconnected
    DISCONNECTED --> CONNECTED: Resume within 60 seconds
    CONNECTED --> LEFT: Explicit LeaveRoom / Room closed
    DISCONNECTED --> LEFT: Grace expired / Room closed
    LEFT --> [*]
```

## 9. Ошибки и transport mapping

Единый REST error envelope:

```
{
  "error": {
    "code": "ROOM_FULL",
    "message": "Room has no free seats",
    "request_id": "0194c9b2-b7f0-7cd1-adb3-5a18bd8ffb0a",
    "retryable": false,
    "details": {}
  }
}
```

Тот же payload используется внутри WS event `command.rejected`.

|   |   |   |   |
|---|---|---|---|
|Domain/transport code|gRPC status|HTTP|WS поведение|
|`VALIDATION_ERROR`|`INVALID_ARGUMENT`|400|`command.rejected`|
|`UNAUTHENTICATED`|`UNAUTHENTICATED`|401|close `4401`|
|`FORBIDDEN`, `INVALID_INVITE`|`PERMISSION_DENIED`|403|`command.rejected`|
|`ROOM_NOT_FOUND`|`NOT_FOUND`|404|`command.rejected`|
|`ROOM_CLOSED`, `ROOM_FULL`, `REVISION_CONFLICT`|`FAILED_PRECONDITION`/`ABORTED`|409|`command.rejected`|
|`IDEMPOTENCY_KEY_REUSED`|`ALREADY_EXISTS`|409|`command.rejected`|
|`RATE_LIMITED`|`RESOURCE_EXHAUSTED`|429|`command.rejected`|
|`CASINO_UNAVAILABLE`|`UNAVAILABLE`|503|retryable `command.rejected`|
|`CASINO_TIMEOUT`|`DEADLINE_EXCEEDED`|504|retryable; результат команды неизвестен|

Не следует возвращать `ALREADY_MEMBER` как ошибку. Повторный `JoinRoom` для уже активного участника должен вернуть актуальный snapshot без повторного membership и без второго domain event.

## 10. Обязательные ограничения БД

Это часть реализации сущностей, а не матрицы владения:

- `casino_players.identity_user_id UNIQUE`;
    
- одна активная membership для пары `(room_id, player_id)`;
    
- `processed_commands.request_id UNIQUE`;
    
- `processed_commands` хранит request hash, result/error, event id и completion timestamp;
    
- optimistic version/revision на Room;
    
- `capacity > 0` и допустимые значения `visibility/status`;
    
- private invite token хранится только как криптографический hash;
    
- Room и RoomParticipant не имеют foreign key в Identity PostgreSQL.
    

Проверка capacity должна быть защищена transaction/row lock или optimistic concurrency. Простого `SELECT count → INSERT` без блокировки недостаточно: два параллельных JoinRoom могут занять последнее место одновременно.

## 11. Failure policy

### Duplicate request

- Gateway сохраняет исходный `request_id` при retry.
    
- Casino сначала проверяет `ProcessedCommand`.
    
- Повтор возвращает тот же business result и тот же `event_id`.
    
- Клиент удаляет повторные события по `event_id` и не применяет revision дважды.
    

### Timeout

- Query можно повторить.
    
- Command можно повторить только с тем же idempotency key.
    
- Gateway не сообщает «операция точно не выполнена» после deadline.
    
- UI показывает retryable состояние и повторяет ту же команду с тем же `request_id`.
    

### Partial failure

- DB commit успешен, Redis write упал: операция успешна; Casino логирует/метрирует ошибку и восстанавливает cache.
    
- DB commit успешен, response потерян: retry возвращает сохранённый результат.
    
- Pub/Sub event потерян: данные не повреждены; revision gap/reconnect приводит к полному snapshot.
    
- Redis очищен: Casino строит snapshot из PostgreSQL.
    
- Gateway instance упал: новый socket подключается к другому instance и выполняет `session.resume`.
    

## 12. Definition of Done для второго backend

Casino-часть считается готовой, когда выполнены все пункты:

1. Миграции создают `casino_players`, `rooms`, `room_participants`, `processed_commands`.
    
2. Все восемь gRPC methods реализованы и соответствуют proto.
    
3. Для каждой mutating-команды есть идемпотентность и проверка повторного key с другим payload.
    
4. Параллельные joins не превышают capacity.
    
5. Casino единолично обновляет RoomStateCache с revision.
    
6. После каждого committed room change публикуется event с `event_id` и revision.
    
7. Cache/PubSub failure не откатывает committed PostgreSQL state.
    
8. Есть cleanup job для истёкших disconnected memberships.
    
9. Unit tests покрывают transitions и authorization.
    
10. Integration tests с PostgreSQL и Redis покрывают create/list/get/join/leave/close/reconnect.
    
11. Отдельные tests воспроизводят duplicate request, response loss после commit и concurrent join на последнее место.
    
12. Gateway может пройти полный сценарий без прямого доступа к Casino PostgreSQL.
    

Сквозной acceptance test:

```
User A создаёт Dice room
→ A подключает WebSocket и входит
→ User B входит и оба получают revision N
→ B теряет соединение
→ A видит DISCONNECTED
→ B reconnect в течение 60 секунд и получает full snapshot
→ A видит RECONNECTED
→ A закрывает комнату
→ оба получают room.closed
→ повтор каждой команды не создаёт новых сущностей или событий
```

## 13. Рабочие допущения, которые нужно подтвердить

Документ сейчас использует три значения, чтобы разработка могла начаться без ожидания:

1. reconnect grace period — **60 секунд**;
    
2. player-created room первой недели — только **Dice Duel**;
    
3. private room использует **одноразово показанный invite token**, в БД хранится только hash.
    

Если одно из этих решений меняется, публичные route names и общая архитектура сохраняются; меняются только правила Casino use cases и часть payload.