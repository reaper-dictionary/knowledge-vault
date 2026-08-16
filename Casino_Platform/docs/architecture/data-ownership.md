# Матрица владения данными Realtime Casino Platform


**Владелец / Source of Truth** — единственный сервис, чья версия данных считается правильной и который имеет право менять их по своим бизнес-правилам.

Другой сервис может хранить:

- внешний UUID;
- временный cache;
- read-only projection;
- результат ранее выполненной операции.

Но он не должен напрямую менять чужую таблицу или подключаться к чужой базе данных.

Колонка **Тип** специально показывает, что не каждое понятие должно стать отдельной таблицей:

- **entity/table** — самостоятельная сущность с идентификатором и жизненным циклом;
- **fields/value object** — поля внутри другой сущности;
- **projection** — производная read-only модель, которую можно восстановить;
- **cache/ephemeral** — временные данные; их потеря не должна повреждать бизнес-данные;
- **technical table** — запись, необходимая для надёжной работы инфраструктуры;
- **code/config** — интерфейс или конфигурация, а не доменная таблица.

---

## Identity Service

| Данные                                      | Тип                                              | Владелец         | Где хранятся                                  | Кто изменяет                                                                   | Кто использует и как получает                                                           | Зачем и важные правила                                                                                                                                                                      |
| ------------------------------------------- | ------------------------------------------------ | ---------------- | --------------------------------------------- | ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `User`                                      | entity/table, обязательно                        | Identity Service | Identity PostgreSQL                           | Только Identity Service; часть admin-операций требует permission               | Gateway получает через gRPC; другие сервисы обычно используют `user_id` и события Kafka | Глобальная учётная запись: `id`, email, nickname/profile, status, timestamps. Casino и Wallet не создают собственную версию User                                                            |
| `Credentials`                               | fields, обязательно                              | Identity Service | Identity PostgreSQL                           | Только Identity при регистрации/смене пароля                                   | Использует только Identity; наружу не передаётся                                        | Данные, доказывающие право пользователя войти: `password_hash`, password algorithm/version, `password_changed_at`. Пароль в открытом виде не хранится                                       |
| `RefreshSession`                            | entity/table, обязательно для текущего auth flow | Identity Service | Identity PostgreSQL                           | Identity создаёт при login/refresh и отзывает при logout/block/password change | Только Identity; клиент получает opaque refresh token, но не запись из БД               | Это не метод, а сохранённая сессия: кто вошёл, hash refresh token, срок жизни, revoked status, device/IP при необходимости. Позволяет иметь несколько устройств и отзывать отдельную сессию |
| `Role`                                      | enum, обязательно                                | Identity Service | исходный код и Identity PostgreSQL            | Только Identity admin use case                                                 | Identity помещает необходимые claims в access token; Gateway и сервисы проверяют claims | Например `user`, `admin`. Роль сама по себе не должна давать произвольный доступ без определённых permissions                                                                               |
| `Permission` и назначение permissions ролям |  code/config, обязательно для admin              | Identity Service | Identity PostgreSQL или versioned code config | Только Identity admin/configuration                                            | Передаются как claims или проверяются внутренним auth contract                          | Конкретные права: `users:block`, `ledger:read`, `balance:adjust`, `promo:manage`. Обычный пользователь их не имеет                                                                          |

### Identity: что не копировать

- `password_hash`, refresh tokens и credential data никогда не передаются в Gateway, Casino или Wallet.
- Casino может иметь только небольшую read-only копию публичного профиля (`user_id`, nickname, avatar), если она действительно нужна для realtime.
- Блокировка пользователя остаётся решением Identity; другие сервисы получают новый token status/claims или событие `user.blocked`.

---

## Wallet Service

| Данные          | Тип                                 | Владелец       | Где хранятся                                                     | Кто изменяет                                                                                | Кто использует и как получает                                                                                          | Зачем и важные правила                                                                                                                                                                              |
| --------------- | ----------------------------------- | -------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `WalletAccount` | entity/table, обязательно           | Wallet Service | Wallet PostgreSQL                                                | Только Wallet Service,<br>WalletAccount создаётся Wallet consumer после user.registered.v1. | Gateway запрашивает balance/summary через gRPC; Casino вызывает финансовые команды, но не меняет account               | Один account на `user_id`. Содержит currency, status, version и при выбранной модели текущий accounting balance                                                                                     |
| `Balance`       | значение                            | Wallet Service | Поле `WalletAccount` периодически сверяется с ledger calculation | Только Wallet через ledger operation                                                        | Gateway получает display value; Casino получает результат `ReserveFunds`, а не полагается на сохранённую копию balance | Casino не хранит authoritative balance. `available_balance = accounting_balance - active_reservations`. Деньги — integer minor units или `Decimal`, не `float`                                      |
| `LedgerEntry`   | immutable entity/table, обязательно | Wallet Service | Wallet PostgreSQL                                                | Только Wallet создаёт; после создания запись нельзя update/delete                           | Пользователь/admin читает через Gateway → Wallet; аналитика может получать события                                     | Неизменяемая запись каждой завершённой credit/debit операции: amount, type, account, operation/reference id, created_at. По ledger можно объяснить любое изменение денег и выполнить reconciliation |
| `Reservation`   | entity/table, обязательно           | Wallet Service | Wallet PostgreSQL                                                | Только Wallet через `ReserveFunds`, `SettleBet`, `ReleaseReservation`, expiration job       | Casino вызывает команды по gRPC и хранит только `reservation_id`; Kafka получает события результата                    | Временно удерживает сумму под ставку. Состояния минимум `ACTIVE → SETTLED/RELEASED/EXPIRED`. Одну reservation нельзя одновременно settle и release                                                  |


### Wallet: что не добавлять сейчас

- `Payment` не нужен, пока проект не работает с настоящими пополнениями/выводами денег.
- Общая сущность с названием `Transaction` слишком неоднозначна. Для проекта понятнее `LedgerEntry`, `Reservation` и idempotent financial operation.
- Casino не получает доступ к Wallet PostgreSQL и не создаёт LedgerEntry самостоятельно.

---

## Casino Service

| Данные                       | Тип                                                              | Владелец       | Где хранятся                                               | Кто изменяет                                                          | Кто использует и как получает                                                               | Зачем и важные правила                                                                                                                                               |
| ---------------------------- | ---------------------------------------------------------------- | -------------- | ---------------------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CasinoPlayer`               | entity/table, обязательно                                        | Casino Service | Casino PostgreSQL                                          | Только Casino; создание может быть вызвано событием `user.registered` | Gateway получает casino-specific данные через gRPC/GraphQL                                  | Ссылка на пользователя в контексте казино: собственный `id`, `identity_user_id`, status, timestamps. Это не копия Identity User и не место для password/roles        |
| `GameDefinition`             | code, обязательно                                                | Casino Service | Исходный код                                               | Разработчики через versioned code                                     | Casino runtime                                                                              | Общий интерфейс `validate_bet/create_round/accept_action/finish_round/calculate_payout`. Это не таблица                                                              |
| `GameConfiguration`          | code/config                                                      | Casino Service | исходный код                                               | разработчик                                                           | Casino runtime; admin может читать                                                          | Payout table, limits и игровые параметры. Конфигурация, повлиявшая на завершённый раунд, должна быть воспроизводимой по версии                                       |
| `Room`                       | entity/table, обязательно                                        | Casino Service | Casino PostgreSQL                                          | Только Casino                                                         | Gateway запрашивает через gRPC; клиенты получают snapshot через WebSocket/GraphQL           | Комната конкретной игры: type, visibility, status, capacity, owner/system marker, revision. Комнату лучше close/archive, а не физически удалять вместе с историей    |
| `RoomParticipant`            | entity/table, обязательно                                        | Casino Service | Casino PostgreSQL                                          | Только Casino по Join/Leave/Ready use cases                           | Gateway получает snapshot через gRPC/WebSocket                                              | Связь игрока с конкретной комнатой: room, player/user, joined/left timestamps, status, seat, ready при необходимости. Не заменяет `CasinoPlayer`                     |
| `Round`                      | entity/table, обязательно                                        | Casino Service | Casino PostgreSQL                                          | Только Casino state machine/scheduler                                 | Gateway получает queries/snapshot; Kafka consumers получают domain events                   | Серверный раунд игры: game type, room, state, phase timestamps, result, revision, configuration version. Сервер authoritative                                        |
| `RoundAction`                | entity/table                                                     | Casino Service | Casino PostgreSQL                                          | Только Casino после проверки команды                                  | History/admin; Round state machine                                                          | Записывает ready, throw, cash out и другие подтверждённые действия, если их нельзя полностью восстановить из Bet/Round. Не хранит неподтверждённые намерения клиента |
| `Bet`                        | entity/table, обязательно                                        | Casino Service | Casino PostgreSQL                                          | Только Casino state machine                                           | Gateway получает history/snapshot; Wallet получает только `bet_id` внутри financial command | Ставка: round, player, amount, selection, reservation id, status, payout, timestamps, idempotency key. Финансовые поля и история не удаляются физически              |
| `FairnessData`               | fields/value object в Round, обязательно для Crash/Dice/Coinflip | Casino Service | Casino PostgreSQL; secret хранится защищённо до раскрытия  | Только Casino до/после раунда по строгому lifecycle                   | Клиент получает hash до раунда и revealed seed после; RTP tool использует тот же алгоритм   | `server_seed`, `server_seed_hash`, `client_seed`, `nonce`, algorithm/version. Результат должен воспроизводиться; seed нельзя менять после commitment                 |
| `PlayerStatisticsProjection` | projection, обязательно как основа engagement                    | Casino Service | Casino PostgreSQL                                          | Casino Kafka consumer обрабатывает `round.completed` идемпотентно     | GraphQL/admin/leaderboard                                                                   | Производные totals: rounds, wins, wagered, payout. Источником истины остаются Round/Bet; projection можно пересчитать                                                |
| `LeaderboardEntry`           | projection/cache, обязательно                                    | Casino Service | Casino PostgreSQL; Redis может быть cache                  | Casino projection handler                                             | Gateway получает через GraphQL; frontend читает                                             | Быстрый ранжированный результат по выбранной metric/period. Redis не должен быть единственным местом хранения невосстановимых данных                                 |
| `AchievementDefinition`      | versioned config, обязательно                                    | Casino Service | code config                                                | разработчик                                                           | Achievement engine, admin, frontend                                                         | Описание достижения и условие его получения. Definition не является фактом получения достижения игроком                                                              |
| `UserAchievement`            | entity/table, обязательно                                        | Casino Service | Casino PostgreSQL                                          | Только Casino achievement handler                                     | Gateway/GraphQL, admin                                                                      | Факт получения: player/user, achievement, progress/status, unlocked_at. Нужен unique constraint, чтобы событие не выдало достижение дважды                           |
| `Campaign`                   | entity/table, обязательно в минимальном виде                     | Casino Service | Casino PostgreSQL                                          | Только Casino admin use case                                          | Promo engine/admin                                                                          | Контейнер правил бонусной кампании: период действия, status, budget/limits при необходимости                                                                         |
| `PromoCode`                  | entity/table, обязательно                                        | Casino Service | Casino PostgreSQL                                          | Только Casino admin use case                                          | Пользователь отправляет code через Gateway; Casino проверяет                                | Код, campaign, reward rule, validity, usage limits. Сам PromoCode не меняет balance напрямую                                                                         |
| `PromoRedemption`            | entity/table, обязательно                                        | Casino Service | Casino PostgreSQL                                          | Только Casino при успешном claim                                      | Admin/history; Wallet получает idempotent bonus-credit command                              | Фиксирует, что конкретный пользователь применил код. Unique constraint защищает от повторного применения                                                             |
| `Tournament`                 | entity/table, обязательно по согласованному scope                | Casino Service | Casino PostgreSQL                                          | Только Casino admin/scheduler                                         | Gateway/GraphQL, tournament scoring handler                                                 | Минимально: выбранная игра, start/end, status, scoring rule. Не требуется сложная bracket-механика                                                                   |
| `TournamentParticipant`      | entity/table с производным score                                 | Casino Service | Casino PostgreSQL                                          | Casino scoring handler                                                | Gateway/GraphQL/admin                                                                       | Участие и score пользователя. Обновление по завершённым раундам должно быть идемпотентным                                                                            |
| `ProcessedCommand`           | technical table, обязательно для финансовых/realtime команд      | Casino Service | Casino PostgreSQL; Redis допустим только как быстрый cache | Только Casino command handling                                        | Внутреннее использование                                                                    | Хранит `request_id/idempotency_key` и результат. Повтор PlaceBet/CashOut не создаёт вторую ставку или выплату после restart                                          |

### Casino: внешние ссылки

- `identity_user_id` — обычный UUID, а не foreign key в Identity PostgreSQL.
- `wallet_reservation_id` в Bet — UUID Reservation, а не foreign key в Wallet PostgreSQL.
- Wallet не запрашивает Bet у Casino: Casino передаёт `bet_id` и необходимые financial parameters в команду Wallet.

---

## Gateway и Redis

Gateway не владеет User, Balance, Room, Round или Bet. Он отвечает за transport, authentication edge, routing, aggregation и realtime connections.

| Данные                | Тип              | Владелец                                                     | Где хранятся                                             | Кто изменяет | Кто использует и как получает                      | Зачем и важные правила                                                                                                     |
| --------------------- | ---------------- | ------------------------------------------------------------ | -------------------------------------------------------- | ------------ | -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `WebSocketConnection` | ephemeral        | Gateway                                                      | Память процесса; connection ownership может быть в Redis | Gateway      | Gateway                                            | Текущее соединение пользователя, connection id и subscriptions. После disconnect запись удаляется; это не бизнес-история   |
| `Presence`            | cache/ephemeral  | Gateway                                                      | Redis с TTL                                              | Gateway      | Другие Gateway instances, Casino при необходимости | Показывает, кто сейчас online/в комнате. Потеря Redis не должна удалять RoomParticipant, Bet или Round                     |
| `RoomStateCache`      | cache/projection | Source of Truth — Casino; техническую копию обновляет Casino | Redis                                                    | casino       | Gateway                                            | Cache содержит revision. При сомнении клиент/ Gateway получает свежий snapshot из Casino PostgreSQL                        |
| `RateLimitState`      | cache/ephemeral  | Gateway                                                      | Redis с TTL                                              | Gateway      | Gateway                                            | Счётчики ограничения HTTP/WebSocket команд. Потеря счётчиков временно ослабляет rate limit, но не повреждает бизнес-данные |

---

## Общие технические данные сервисов

Эти записи не являются бизнес-сущностями пользователя, но нужны для профессиональной надёжности микросервисов.

| Данные                      | Тип                                                 | Владелец                                    | Где хранятся                                                     | Кто изменяет                                                                      | Кто использует и как получает       | Зачем и важные правила                                                                                                              |
| --------------------------- | --------------------------------------------------- | ------------------------------------------- | ---------------------------------------------------------------- | --------------------------------------------------------------------------------- | ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `OutboxMessage`             | technical table, обязательно у producer-сервисов    | Сервис, создавший business event            | Та же PostgreSQL и та же transaction, что business data          | Identity/Wallet/Casino записывают только в своей БД; publisher помечает published | Outbox publisher отправляет в Kafka | Не позволяет потерять event между commit БД и отправкой Kafka. Payload/version/event id фиксируются при business transaction        |
| `ProcessedEvent`            | technical table, обязательно у важных consumers     | Каждый consumer service                     | Его собственная PostgreSQL                                       | Consumer при обработке Kafka event                                                | Внутреннее использование consumer   | Хранит уже обработанные `event_id`; повтор Kafka event не удваивает статистику, bonus или achievement                               |
| `DomainEvent`               | immutable message, не основная business table       | Producer и его versioned contract           | Kafka с retention; исходное business state остаётся в PostgreSQL | Producer публикует через Outbox                                                   | Подписанные consumers               | Содержит `event_id`, name/version, occurred_at, aggregate_id, correlation_id и payload. Kafka event не заменяет источник истины     |
| `AuditRecord`               | audit-поля существующих сущностей + structured logs | Сервис, выполняющий чувствительную операцию | Его PostgreSQL и/или централизованные structured logs            | Identity/Wallet/Casino автоматически                                              | Admin/Team Lead                     | Кто, когда и почему блокировал пользователя, корректировал баланс или менял promo. Audit нельзя редактировать обычной командой      |
| `Metrics` и structured logs | observability data                                  | Каждый сервис генерирует свои данные        | Prometheus/log storage; Grafana только визуализирует             | Автоматически instrumentation/logging                                             | Team Lead/Grafana/alerts            | Это не доменные таблицы. Labels не должны содержать `user_id`, `bet_id` и другие значения с высокой cardinality                     |
| `RTPReport`                 | generated artifact, не online entity                | Casino simulation tool                      | stdout/file/CI artifact                                          | Команда simulation tool                                                           | Разработчики/admin/demo             | Рассчитывается из versioned game algorithm/config. Не требуется таблица в основной БД, если отчёты не нужно хранить в admin history |

---
## Общее правило идемпотентности

Все команды с побочными эффектами должны безопасно обрабатывать повтор:

- создание WalletAccount;
- PlaceBet и CashOut;
- ReserveFunds, SettleBet и ReleaseReservation;
- административную корректировку;
- применение PromoCode;
- выдачу Achievement;
- обновление Tournament score;
- обработку Kafka events.

Конкретные idempotency keys, поля и database constraints
описываются в документах сущностей и контрактов, а не в этой матрице.
