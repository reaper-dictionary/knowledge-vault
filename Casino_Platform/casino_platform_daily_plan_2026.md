# Согласованный подневный план Realtime Casino Platform

Период: **6 августа — 7 сентября 2026 года**  
Старт командной разработки: **8 августа**  
Желаемый product feature complete: **31 августа**  
Жёсткий дедлайн: **7 сентября**

## 1. Зафиксированные условия

- Команда: Team Lead / Backend / DevOps, второй backend-разработчик, frontend-разработчик.
- Team Lead работает 7–8 часов в день. В день новой технологии внутри этого времени: 3–5 часов изучения на Coffee House или Void Marketplace, 2–4 часа самостоятельного переноса в Casino Platform, до 1 часа на ревью и помощь команде.
- Второй backend и frontend работают по 5–6 часов в день.
- План охватывает каждый календарный день. В каждой семидневной итерации предусмотрен один flex-день на форс-мажор или накопившуюся интеграцию.
- До 8 августа Team Lead готовит skeleton, contracts, схемы, Compose и базовый CI. Готовой реализации gRPC к старту команды нет.
- Новую инфраструктурную технологию сначала изучает и реализует Team Lead. Только после merge эталона и короткого разбора второй backend использует её в своей зоне.
- Автотесты создаются вместе с функциональностью, в том числе с помощью ИИ. Отдельные дни на написание большого объёма тестового кода не выделяются. Разработчик обязан проверить сгенерированные тесты и реально их запустить.
- Итоговая среда: ноутбук с 16 ГБ RAM, Arch Linux и VM. K3s — демонстрационная технология. Публичный домен не обязателен; HTTPS локальный.
- Способ публикации Docker-образов заранее не выбран. Сначала изучаются варианты registry, затем принимается решение.
- PostgreSQL, Redis и Kafka разрешено оставить вне K3s, если перенос stateful-компонентов угрожает сроку. Обязательная Kubernetes-демонстрация охватывает application-сервисы, frontend, конфигурацию, probes, networking и observability.

## 2. Обязательный scope и допустимое упрощение

К 7 сентября должны работать:

- регистрация, login, access/refresh sessions и роли;
- Wallet, immutable ledger, reservation, settlement, idempotency и административная корректировка;
- комнаты, WebSocket, reconnect и актуальный snapshot;
- Crash, Roulette, Dice Duel, Coinflip и Slots;
- история игр и ставок;
- provably fair для Crash, Dice Duel и Coinflip;
- leaderboard, базовые achievements, promo codes и минимальный tournament;
- GraphQL aggregation;
- RTP simulator;
- admin panel: пользователи, блокировка, ledger, корректировка баланса, игры, промокоды и технические показатели;
- Prometheus и четыре группы Grafana-показателей: Platform, Realtime, Casino Business и Kafka;
- базовый CI/CD и демонстрационный K3s deployment с локальным HTTPS.

Если команда отстаёт, обязательные функции не удаляются, но уменьшается глубина реализации:

1. AI Assistant не выполняется.
2. Coinflip и Slots получают функциональный UI без сложной анимации.
3. Achievements ограничиваются 3–5 фиксированными правилами.
4. Promo codes ограничиваются одним типом бонуса и однократным применением.
5. Tournament — один time-boxed leaderboard без bracket-механики.
6. Admin panel остаётся утилитарной без сложного дизайна.
7. PostgreSQL, Redis и Kafka остаются в Compose/на VM вне K3s.
8. Публичный deployment, домен и production CA не добавляются.

## 3. Общие правила выполнения

### Definition of Done ежедневной задачи

Задача считается завершённой, только если:

- код объединён с актуальной веткой и запускается;
- contract не нарушен либо его изменение согласовано до реализации;
- happy path проверен вручную;
- критичные ошибки и повторные команды проверены;
- релевантные сгенерированные тесты просмотрены разработчиком и проходят;
- коротко записаны `Сделано / Проверено / Блокер / Следующий шаг`.

### Правило зависимостей

- Пока эталон новой технологии не готов, второй backend работает над domain layer через ports/fakes и не блокируется.
- Frontend сначала работает по зафиксированным contracts и mock events, затем заменяет mock на реальную интеграцию.
- Realtime идёт через WebSocket/Redis; Kafka используется только для значимых надёжных событий.
- GraphQL агрегирует профиль, баланс, каталог, историю и engagement-данные; realtime-команды через GraphQL не передаются.

### Flex-дни

**14, 21 и 28 августа**, а также **4 сентября** не содержат новых обязательных функций. Если один день недели потерян, незавершённая работа переносится туда. Если отставания нет, команда выполняет интеграцию, исправляет дефекты и улучшает качество уже готового результата.

## 4. Конкретные checkpoint’ы проекта

|Checkpoint|Дата|Что должно запускаться|Проверка прохождения|
|---|---:|---|---|
|CP0 — Foundation ready|7 августа|Skeleton всех сервисов, contracts, Compose, базовый CI|`docker compose up` поднимает зависимости и service skeletons; health endpoints отвечают; CI запускает lint/tests; команда получила инструкции запуска|
|CP1 — Walking skeleton|14 августа|Identity → Gateway WebSocket → Casino gRPC → room broadcast → reconnect snapshot|Два пользователя входят в одну комнату; после разрыва клиент получает актуальный snapshot; повторный `request_id` не выполняет команду дважды|
|CP2 — Crash + Wallet vertical|21 августа|Crash end-to-end с Wallet, ledger, Kafka/outbox, history и provably fair|Reserve, cash out/lose, settlement и ledger согласованы; повторная команда не меняет баланс второй раз; результат воспроизводится по seeds|
|CP3 — Roulette ready|24 августа|Roulette end-to-end и базовая observability в Compose|Несколько клиентов ставят до lock; сервер выдаёт единый результат; payout и новый баланс корректны; метрики доступны Prometheus/Grafana|
|CP4 — Three-game platform|28 августа|Crash, Roulette, Dice Duel, GraphQL и admin foundation|Dice Duel проходит от ready до settlement; GraphQL возвращает профиль, баланс, каталог и историю без N+1; admin permissions закрывают обычного пользователя|
|CP5 — Product feature complete|31 августа|Все пять игр и обязательные product-функции работают в Compose|Coinflip, Slots, leaderboard, achievements, promo, tournament, RTP command и admin операции проходят smoke-сценарий; новых product-фич после даты не добавляется|
|CP6 — Deployed candidate|3 сентября|Application stack работает в K3s через локальный HTTPS; CI собирает/доставляет images; dashboards заполнены|Регистрация, WebSocket и одна ставка проходят через Ingress; Pod restart не ломает данные; четыре группы dashboards получают реальные метрики|
|CP7 — Release candidate|5 сентября|Версия без известных P0 и с согласованным списком P1|Пройдены critical smoke, reconnect, duplicate command, Wallet consistency, Kafka retry и rollback проверки|
|CP8 — Final release|7 сентября|Финальная демонстрационная версия и воспроизводимый запуск|Чистый запуск по README, финальный tag, демонстрационный сценарий от регистрации до K3s/Grafana, нет известных P0|

## 5. План Team Lead / Backend / DevOps

|Дата|Фокус и конкретные задачи|Результат дня|
|---|---|---|
|6 августа, чт|Зафиксировать scope и service boundaries. Создать monorepo/skeleton сервисов, базовые настройки, health endpoints, `.env.example`. Описать REST, proto, WebSocket envelope и Kafka event naming. Поднять PostgreSQL/Redis/Kafka в Compose. Начать CI: lint + unit tests.|Структура проекта и contracts доступны команде; Compose поднимает инфраструктуру; CI хотя бы запускается на push/PR.|
|7 августа, пт|Завершить proto-схемы, HTTP/GraphQL schema skeleton и генерацию клиентов. Добавить миграционные шаблоны, service Make/Task-команды, правила PR и README запуска. Прогнать чистый checkout → Compose → CI. Провести handoff команде.|CP0 пройден; 8 августа команда начинает работу без настройки архитектуры с нуля.|
|8 августа, сб|Провести kickoff и зафиксировать ownership. Адаптировать Identity из Void Marketplace: user, registration, login, password hashing, access/refresh session, роли. Проверить миграции и контракт Gateway ↔ Identity.|Регистрация и login работают через реальный Identity; frontend получает стабильный auth contract.|
|9 августа, вс|**Учёба:** gRPC server/client, generated stubs, deadlines, metadata, status codes и mapping ошибок на существующем проекте. **Проект:** реализовать первый эталонный Gateway → Casino unary call, interceptor/correlation ID и integration smoke. В конце дня заморозить шаблон и объяснить его второму backend.|Готов и проверен первый gRPC-эталон. Только со следующего дня второй backend начинает реализацию по нему.|
|10 августа, пн|Утром разобрать gRPC-эталон со вторым backend. **Учёба:** WebSocket lifecycle, auth handshake, disconnect и backpressure. **Проект:** создать Gateway WebSocket endpoint, проверку token, connection registry и versioned message envelope.|Авторизованный пользователь устанавливает WebSocket; Gateway умеет разобрать валидную команду и вернуть protocol error.|
|11 августа, вт|Реализовать routing WebSocket-команды в Casino через готовый gRPC client, request correlation и mapping gRPC errors в WebSocket response. Добавить структурированные логи. Провести интеграцию с frontend и Casino.|Один клиент проходит цепочку WebSocket → Gateway → Casino → response.|
|12 августа, ср|Использовать существующие знания Redis: presence, connection ownership, Pub/Sub channel convention и broadcast между соединениями. Добавить ограничение labels/keys и очистку presence после disconnect.|Два клиента видят подтверждённое сервером изменение комнаты; Redis не является источником постоянных данных.|
|13 августа, чт|Реализовать reconnect protocol: replacement старого соединения, last known revision, запрос snapshot, защита от устаревших сообщений. Провести end-to-end ревью первой вертикали и исправить только блокирующие проблемы.|Полный walking skeleton готов до flex-дня.|
|14 августа, пт — FLEX|Не добавлять новую технологию. Восстановить потерянную задачу недели. Если отставания нет — провести CP1 smoke двумя клиентами, исправить contract drift и документировать realtime protocol.|CP1 пройден либо сформирован конкретный список блокеров с владельцами и сроком не позднее 15 августа.|
|15 августа, сб|Спроектировать Wallet: account, balance projection, immutable ledger entry, reservation, operation/idempotency key, статусы и инварианты. Создать миграции, repositories/UoW и транзакционный сценарий пополнения demo-баланса.|Wallet хранит счёт и immutable ledger; прямое изменение balance вне доменной операции невозможно.|
|16 августа, вс|Реализовать Wallet gRPC: `GetBalance`, `ReserveFunds`, `ReleaseReservation`; row locking/optimistic checks, insufficient funds и повторный idempotency key. Передать второму backend готовый contract и примеры вызова.|Casino может безопасно зарезервировать и освободить средства; дубли не создают вторую операцию.|
|17 августа, пн|Реализовать `SettleBet`, credit/debit ledger entries, reconciliation query и безопасную административную корректировку как отдельный ledger operation. Зафиксировать failure codes для Casino.|Полный финансовый lifecycle доступен через Wallet API и проверяется по ledger.|
|18 августа, вт|Реализовать эталонный transactional outbox в Wallet, общий event contract, publisher в Kafka, retry policy и базовую DLQ convention. Объяснить второму backend способ подключения Casino events. Не отправлять realtime ticks в Kafka.|Wallet-события надёжно выходят из транзакции; готовый шаблон можно самостоятельно применить в Casino.|
|19 августа, ср|Собрать Gateway + Casino + Wallet + Redis + Kafka для полного Crash flow. Проверить границы транзакций, таймауты gRPC и обработку недоступного Wallet. Провести ревью cash out/settlement реализации второго backend.|Crash проходит end-to-end; финансовые ошибки возвращаются клиенту без повреждения ledger.|
|20 августа, чт|Помочь второму backend реализовать его первый Kafka consumer, не писать consumer за него. Проверить consumer idempotency и retry. Добавить correlation ID сквозь HTTP/gRPC/WebSocket/Kafka и минимальные operational counters в коде.|Второй backend самостоятельно владеет consumer; один trace/correlation key связывает критический сценарий в логах.|
|21 августа, пт — FLEX|Сначала восстановить потерянную задачу. Если график соблюдён — провести Wallet audit: reserve/settle/release, duplicate cash out, crash до/после команды, reconnect и replay outbox. Исправлять дефекты, а не писать новые тестовые наборы ради coverage.|CP2 пройден, Crash и Wallet готовы как эталон для остальных игр.|
|22 августа, сб|**Учёба:** Prometheus Counter/Gauge/Histogram, labels и `/metrics`. **Проект:** общий instrumentation middleware для HTTP/gRPC; метрики WebSocket, rooms, Wallet failures и outbox. Следить за cardinality.|Prometheus может собирать метрики всех application-сервисов в Compose.|
|23 августа, вс|**Учёба:** Prometheus scrape config и базовый PromQL. **Проект:** добавить Prometheus в Compose, recording/query examples и необходимые exporters только при реальной пользе. Проверить метрики Roulette вместе со вторым backend.|Request rate, errors, latency, connections, rooms, bets, settlement failures и outbox видимы запросами.|
|24 августа, пн|**Учёба:** Grafana datasource, variables и panels. **Проект:** минимальные Platform Overview и Realtime dashboards; добавить Casino Business panels для готовых игр. Провести checkpoint Roulette.|Метрики отображаются в Grafana; CP3 пройден.|
|25 августа, вт|**Учёба:** GraphQL schema/resolvers, auth context, N+1. **Проект:** Gateway schema для profile, balance, catalog и history; реализовать первые resolvers через service clients.|Frontend получает реальные агрегированные данные одним GraphQL query.|
|26 августа, ср|**Учёба:** DataLoader/batching, query depth/complexity и error model. **Проект:** устранить N+1, добавить ограничения и engagement/admin schema skeleton. Проверить, что realtime не переносится в GraphQL.|GraphQL aggregation стабильна, авторизована и защищена от тяжёлых запросов.|
|27 августа, чт|Реализовать admin authorization end-to-end: блокировка пользователя в Identity, просмотр ledger и корректировка баланса в Wallet, Gateway admin aggregation. Проверить запрет операций для обычной роли. Провести ревью Dice и event projections.|Backend-основа всех обязательных admin-операций готова; Dice не блокируется инфраструктурой.|
|28 августа, пт — FLEX|Восстановить потерянный день. Если отставания нет — checkpoint трёх игр, GraphQL и dashboards; исправить contract mismatches, P0/P1 и документацию API. Не начинать Coinflip раньше прохождения CP4.|CP4 пройден; общий game template стабилен для двух простых игр.|
|29 августа, сб|**Учёба:** GitHub Actions Docker build, image tags, cache и основы registry authentication. **Проект:** CI собирает все production images с commit tag, но публикация пока не считается обязательной. Доделать admin GraphQL aggregation при необходимости.|Воспроизводимые versioned images собираются CI; вопрос registry сформулирован на основе практики.|
|30 августа, вс|**Учёба:** Kubernetes Deployment, Service, ConfigMap, Secret и probes на Coffee House. **Проект:** установить/проверить K3s в VM, создать namespace и развернуть один stateless skeleton/image с readiness/liveness. Не переносить весь стек.|Один application-сервис стабильно работает в K3s; понятен способ доставки локального image.|
|31 августа, пн|Расширить manifests на Gateway, Identity, Wallet, Casino и frontend; зафиксировать ConfigMap/Secret strategy и resource requests/limits. Одновременно провести Compose product feature freeze и сверить обязательный scope.|CP5 пройден в Compose; application stack имеет рабочий K3s skeleton без требования переносить stateful dependencies.|
|1 сентября, вт|**Учёба и решение:** сравнить GHCR, локальный registry и build/import на VM; выбрать самый простой воспроизводимый путь. **Проект:** добавить push/import и ручной deploy job, version pinning и smoke after deploy. Решение записать ADR.|CI не только собирает, но и доставляет выбранную версию в K3s; registry выбран после практики.|
|2 сентября, ср|**Учёба:** Ingress и локальный TLS. **Проект:** настроить Ingress routes, self-signed/local CA certificate, `wss`, Services, Secrets и probes. Подключить внешние PostgreSQL/Redis/Kafka либо развернуть их только если это безопасно по сроку.|Auth, GraphQL и WebSocket доступны через единый локальный HTTPS endpoint.|
|3 сентября, чт|Перенести/подключить Prometheus и Grafana к развёрнутой версии. Завершить Platform, Realtime, Casino Business и Kafka dashboards. Провести полный deployed smoke вместе с командой.|CP6 пройден: K3s, HTTPS, CI image flow и dashboards демонстрируются на реальном пользовательском сценарии.|
|4 сентября, пт — FLEX / FREEZE|Не добавлять функции. Восстановить потерянный день либо исправлять P0/P1. Выполнить ограниченный smoke/load/fault прогон: Pod restart, Wallet timeout, reconnect, outbox retry, Kafka consumer restart. Зафиксировать freeze.|Полный feature freeze; все оставшиеся дефекты классифицированы, новых архитектурных изменений нет.|
|5 сентября, сб|Проверить воспроизводимый deploy выбранного tag, readiness, rollout/rollback и dashboards во время smoke. Организовать совместный release-candidate прогон и закрыть P0.|CP7 пройден; release candidate разворачивается повторяемо и не имеет известных критических дефектов.|
|6 сентября, вс|Сначала исправить оставшийся P0. Затем описать архитектуру, локальный Compose, CI/CD, K3s, HTTPS, observability и troubleshooting. Подготовить схему и команды демонстрации.|Другой разработчик может поднять проект по README; инфраструктурная часть презентации готова.|
|7 сентября, пн|Выпустить финальный tag, выполнить чистый deploy и smoke. Провести финальное архитектурное ревью, репетицию демонстрации и сохранить точный список известных некритичных ограничений.|CP8 пройден; финальная версия готова к показу.|

## 6. План второго backend-разработчика

| Дата                           | Фокус и конкретные задачи                                                                                                                                                                                                                            | Результат дня                                                                                                 |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| 6 августа, чт                  | До старта реализации изучить границы микросервисов, структуру будущего Casino Service и gRPC concepts. По contracts составить список вопросов и черновик domain entities для rooms/rounds/bets.                                                      | Есть domain draft и список неясностей; код проекта не создаёт конфликтов со skeleton Team Lead.               |
| 7 августа, пт                  | Изучить state machines Crash/Roulette/Dice, основы Kafka/Redis/outbox на уровне назначения. Развернуть полученный skeleton, проверить Compose и миграционный шаблон.                                                                                 | Рабочее окружение готово; разработчик понимает, что хранится в PostgreSQL, Redis и Kafka.                     |
| 8 августа, сб                  | Создать структуру Casino Service: game registry, Room, Player, Round, Bet, состояния и базовые domain errors. Реализовать migrations, repositories/UoW и несколько полноценных use cases, а не только пустые handlers.                               | Casino сохраняет и читает комнаты/раунды/ставки локально; domain layer не зависит от transport.               |
| 9 августа, вс                  | Завершить create/list/get room, join/leave domain rules, round ownership и game-type validation. Подготовить gRPC adapter interfaces, но не использовать неготовый эталон. Работать через unit-level ports/fakes.                                    | Room domain функционален до transport; ничего не блокируется ожиданием gRPC.                                  |
| 10 августа, пн                 | После утреннего разбора готового эталона самостоятельно реализовать Casino gRPC server для room queries/commands, mapping domain errors и integration smoke. Не копировать внутреннюю структуру Gateway.                                             | Casino отвечает через gRPC; разработчик понимает server lifecycle и generated stubs.                          |
| 11 августа, вт                 | Реализовать Join/Leave/Create room, snapshot, `revision`, проверку stale command и идемпотентность по `request_id`. Подключить handlers к gRPC adapter.                                                                                              | Gateway получает подтверждённый room state; повтор команды безопасен.                                         |
| 12 августа, ср                 | Завершить room lifecycle и multi-player rules. Интегрировать server-side broadcast events с созданной Team Lead схемой. Добавить сценарий двух одновременных пользователей и корректные domain events.                                               | Два клиента видят одно authoritative состояние комнаты.                                                       |
| 13 августа, чт                 | После готовности Redis conventions самостоятельно реализовать room-state cache adapter и восстановление snapshot из постоянного состояния. Обработать disconnect/rejoin и устаревшую revision.                                                       | Выполнен обязательный практический результат по Redis; reconnect не зависит от сохранности Pub/Sub-сообщения. |
| 14 августа, пт — FLEX          | Закрыть незавершённую задачу недели. Если всё готово — исправить найденные на CP1 ошибки room lifecycle, duplicate request и snapshot. Не добавлять новую функцию.                                                                                   | Casino часть CP1 зелёная.                                                                                     |
| 15 августа, сб                 | Реализовать Crash как `GameDefinition`: состояния `BETTING → RUNNING → COMPLETED`, параметры комнаты, round scheduler и модели ставок. Использовать fake Wallet port.                                                                                | Crash state machine детерминированно проходит полный lifecycle без transport.                                 |
| 16 августа, вс                 | Реализовать `PlaceBet`, betting window, ограничения суммы/одной ставки, cancel до lock и domain events. Подготовить Wallet client adapter к уже зафиксированному proto.                                                                              | Правила ставки полностью работают через fake; adapter готов к реальному Wallet.                               |
| 17 августа, пн                 | Самостоятельно реализовать Casino → Wallet gRPC client по эталону Team Lead: deadlines, error mapping, reserve/release. Интегрировать его в betting flow.                                                                                            | Выполнен обязательный gRPC client; реальная ставка резервирует средства.                                      |
| 18 августа, вт                 | Реализовать multiplier progression, crash point, `CashOutCommand`, payout и вызов `SettleBet`. Обработать duplicate/late cash out и отказ Wallet.                                                                                                    | Crash корректно рассчитывает win/loss и не допускает двойной выплаты.                                         |
| 19 августа, ср                 | Реализовать provably fair для Crash, round/bet history и snapshot активного раунда. Добавить воспроизводимость seed/hash/nonce и восстановление после reconnect.                                                                                     | Crash результат проверяем, история сохраняется, reconnect возвращает текущий раунд.                           |
| 20 августа, чт                 | По предоставленным event/outbox conventions самостоятельно подключить Casino outbox producer и написать Kafka consumer `round.completed` для stats projection/leaderboard foundation, retry и идемпотентность. Исправить конкурентные дефекты Crash. | Выполнены Casino producer и обязательный Kafka consumer; повтор события не удваивает статистику.              |
| 21 августа, пт — FLEX          | Закрыть отставание. Если его нет — пройти CP2 вместе с Team Lead, исправить state machine, payout, repeated action и failure handling. Не расходовать день на увеличение coverage.                                                                   | Casino часть Crash готова как эталон.                                                                         |
| 22 августа, сб                 | Реализовать Roulette plugin: state machine `BETTING → LOCKED → SPINNING → COMPLETED`, bet types, validation и payout table. Переиспользовать rooms, Wallet port и events.                                                                            | Roulette domain проходит раунд на fake/детерминированном RNG.                                                 |
| 23 августа, вс                 | Реализовать server timer, приём нескольких ставок, lock, единый result, Wallet reservation/settlement и защиту от поздней ставки.                                                                                                                    | Несколько пользователей проходят общий Roulette round с реальным Wallet.                                      |
| 24 августа, пн                 | Добавить Roulette history, business metrics/events и leaderboard projection. Создать campaign/promo model и migration как подготовку следующей недели. Исправить интеграцию с frontend.                                                              | Roulette end-to-end готова; promo foundation создан без отвлечения Team Lead.                                 |
| 25 августа, вт                 | Реализовать Dice Duel room на двоих: join slots, ready state, одинаковая ставка, reserve обоих игроков, start/cancel rules.                                                                                                                          | Матч начинается только для двух готовых игроков с успешными reservations.                                     |
| 26 августа, ср                 | Реализовать несколько throws, server result, winner/payout, settlement/release и provably fair. Подключить history/events.                                                                                                                           | Dice Duel проходит полный игровой и финансовый lifecycle.                                                     |
| 27 августа, чт                 | Завершить edge cases Dice. На базе `round.completed` projection реализовать leaderboard queries и achievement engine с 3–5 правилами. Добавить admin read queries для games/rooms/rounds.                                                            | Dice готов; leaderboard и базовые achievements обновляются событиями.                                         |
| 28 августа, пт — FLEX          | Восстановить потерянную задачу. Если график соблюдён — пройти CP4, исправить дефекты трёх игр и стабилизировать общий game template. Coinflip не начинать до прохождения checkpoint.                                                                 | CP4 пройден; следующие игры добавляются без изменения архитектуры.                                            |
| 29 августа, сб                 | Реализовать Coinflip целиком на общем шаблоне: choice, deterministic result, provably fair, payout, Wallet, history и события. Подключить существующие stats/achievements.                                                                           | Coinflip функционально готов end-to-end.                                                                      |
| 30 августа, вс                 | Реализовать Slots в упрощённом обязательном объёме: reels/symbol table, deterministic RNG abstraction, paylines/payout table, Wallet settlement, history. Завершить promo claim с одним типом бонуса и idempotency.                                  | Slots и базовый promo code работают; повторное применение запрещено.                                          |
| 31 августа, пн                 | Реализовать минимальный tournament как time-boxed leaderboard по выбранной игре. Создать общий RTP simulator с theoretical/empirical RTP, wagered, paid, house edge и standard deviation. Экспортировать admin operations для games/promos.          | Все обязательные product-функции существуют в Compose; backend часть CP5 пройдена.                            |
| 1 сентября, вт                 | Прогнать RTP на пяти играх, проверить формулы и разумность результатов. Укрепить уже работающие admin API для rounds, games, promo, achievements и tournament; исправлять только выявленные пробелы CP5.                                             | RTP report воспроизводим; admin backend подтверждён на полном согласованном scope.                            |
| 2 сентября, ср                 | Проверить сервисы в K3s-среде: адреса dependencies, timeouts, migrations и secrets. Доработать outbox retry/DLQ и consumer recovery. Исправить проявившиеся deployment-интеграционные ошибки.                                                        | Casino/Wallet interaction переживает restart consumer/service без двойной финансовой операции.                |
| 3 сентября, чт                 | Провести deployed smoke всех пяти игр и product-функций. Подать реальные business/Kafka metrics в dashboards. Исправить P0/P1, найденные через K3s/HTTPS.                                                                                            | Backend полностью работает в deployed candidate; CP6 пройден.                                                 |
| 4 сентября, пт — FLEX / FREEZE | Закрыть потерянный день либо проверить duplicate commands, concurrent actions, Wallet unavailable, Kafka retry/DLQ и reconnect snapshot. Исправлять дефекты без добавления возможностей.                                                             | Backend feature freeze; нет известных P0.                                                                     |
| 5 сентября, сб                 | Пройти критический release-candidate сценарий всех игр, ledger invariants, promo/achievement/tournament и RTP. Проверить seed data и миграции на чистой БД. Исправить P0.                                                                            | Backend часть CP7 стабильна и воспроизводима.                                                                 |
| 6 сентября, вс                 | Сначала исправить P0. Затем документировать game state machines, Wallet interaction, Kafka/outbox, provably fair, RTP и известные ограничения. Подготовить данные для demo.                                                                          | Игровая и финансовая логика понятна из документации; demo data готова.                                        |
| 7 сентября, пн                 | Выполнить финальный backend smoke после чистого deploy, проверить миграции/seed и помочь с репетицией. Исправлять только release-blocker.                                                                                                            | Backend финального tag подтверждён.                                                                           |

## 7. План frontend-разработчика

| Дата                           | Фокус и конкретные задачи                                                                                                                                                                              | Результат дня                                                                                            |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| 6 августа, чт                  | Изучить access/refresh flow, WebSocket lifecycle/reconnect и GraphQL client concepts. Выбрать state-management подход. Подготовить визуальные референсы Crash/Roulette и технический animation spike.  | Есть список frontend-решений и рисков; выбран базовый подход к состоянию.                                |
| 7 августа, пт                  | Развернуть skeleton, изучить contracts и mock payloads. Составить route map, component hierarchy, design tokens и состояния loading/error/disconnected. Проверить возможность основных анимаций.       | Frontend готов начать реализацию без ожидания дизайнера.                                                 |
| 8 августа, сб                  | Создать application shell, routing, theme/design tokens и auth pages. Реализовать API client и формы registration/login по стабильному contract.                                                       | Пользователь регистрируется и входит через реальный Identity.                                            |
| 9 августа, вс                  | Реализовать token storage, refresh flow, protected routes, logout, восстановление сессии и обработку 401. Начать изолированный mock WebSocket client.                                                  | Auth переживает reload и корректно завершает истёкшую сессию.                                            |
| 10 августа, пн                 | Реализовать WebSocket manager: connect/disconnect, auth, message parsing, request correlation и connection status. Работать по зафиксированному envelope.                                              | Frontend подключается к Gateway и отображает состояние соединения.                                       |
| 11 августа, вт                 | Сделать lobby, list/create/join room и отображение server errors. Подключить реальные room responses через Gateway/WebSocket; внутренний gRPC остаётся скрытым от клиента.                             | Один пользователь создаёт комнату и входит в неё без mock.                                               |
| 12 августа, ср                 | Добавить participants list, live room state и обработку broadcast для двух вкладок/клиентов. Не рассчитывать состояние игры на клиенте.                                                                | Два клиента видят одинаковое подтверждённое состояние.                                                   |
| 13 августа, чт                 | Реализовать reconnect with backoff, повторную auth, last revision, замену локального состояния snapshot. Добавить loading/disconnected/stale/error states.                                             | После обрыва UI возвращается в корректную комнату без двойной команды.                                   |
| 14 августа, пт — FLEX          | Закрыть потерянную задачу. Если всё готово — провести CP1 в двух окнах, исправить auth/reconnect UX и документировать frontend event handling.                                                         | Frontend часть CP1 зелёная.                                                                              |
| 15 августа, сб                 | Создать Crash screen и UI state machine по mock events: betting/running/completed, ставка, список игроков. Начать качественную multiplier animation.                                                   | Полный визуальный Crash flow проигрывается на mock timeline.                                             |
| 16 августа, вс                 | Добавить balance display, bet amount controls, countdown и серверные ошибки: insufficient funds, betting closed, duplicate bet.                                                                        | Пользователь понимает статус ставки и финансовую ошибку.                                                 |
| 17 августа, пн                 | Заменить mock phases на реальные WebSocket events, синхронизировать countdown по server time и подключить создание ставки/обновление reservation.                                                      | Crash UI следует серверному времени и показывает принятую ставку.                                        |
| 18 августа, вт                 | Подключить live multiplier, cash out, win/loss, payout и обновление Wallet. Не вычислять crash point или выигрыш на клиенте.                                                                           | Полный игровой цикл отображается по authoritative events.                                                |
| 19 августа, ср                 | Добавить history, active bets/players, reconnect в середине раунда и экран проверки provably fair по hash/seed/nonce.                                                                                  | Crash восстанавливается после disconnect, результат можно проверить.                                     |
| 20 августа, чт                 | Провести совместный end-to-end прогон, исправить race UI, повторный click, медленную сеть, mobile layout и empty/loading states. Не выделять отдельный день на написание frontend tests.               | Crash визуально и функционально готов до flex-дня.                                                       |
| 21 августа, пт — FLEX          | Закрыть отставание либо пройти CP2 и исправить пользовательские дефекты. Если всё зелёное — переиспользуемые game layout/event hooks выносятся без изменения contracts.                                | Crash готов как frontend-эталон.                                                                         |
| 22 августа, сб                 | Создать Roulette wheel animation по mock server result, betting table, chips и визуальные состояния `BETTING/LOCKED/SPINNING/COMPLETED`.                                                               | Roulette полностью демонстрируется на mock events.                                                       |
| 23 августа, вс                 | Подключить реальные комнаты, countdown и bet commands; показать несколько ставок и ошибки закрытого периода/баланса.                                                                                   | Ставки проходят до lock и корректно блокируются после него.                                              |
| 24 августа, пн                 | Связать server result с анимацией, payout, history, новым балансом и reconnect. Исправить рассинхронизацию анимации и authoritative result.                                                            | Roulette end-to-end готова; frontend часть CP3 пройдена.                                                 |
| 25 августа, вт                 | Настроить GraphQL client, auth/error handling и cache policy. Подключить profile, balance, game catalog и history. Создать Dice Duel screen/ready mock.                                                | Продуктовые экраны читают реальные агрегированные данные; Dice UI подготовлен.                           |
| 26 августа, ср                 | Реализовать Dice room на двоих, player slots, ready state, bet и несколько throw animations. Подключить реальные события и результат.                                                                  | Dice match виден от ready до определения победителя.                                                     |
| 27 августа, чт                 | Завершить Dice payout/fairness/reconnect UI. Создать admin shell и страницы users/ledger. Добавить leaderboard и achievements screens по GraphQL.                                                      | Dice функционально готов; admin и engagement UI имеют рабочую основу.                                    |
| 28 августа, пт — FLEX          | Восстановить потерянный день. Если график соблюдён — пройти CP4, исправить общие game hooks, GraphQL cache и responsive состояния.                                                                     | Frontend часть CP4 зелёная.                                                                              |
| 29 августа, сб                 | Реализовать Coinflip UI по общему game layout, выбор стороны, result/fairness/history. Подключить admin users, blocking, ledger и balance adjustment.                                                  | Coinflip работает; ключевые пользовательские admin-операции доступны.                                    |
| 30 августа, вс                 | Реализовать упрощённый Slots UI, reels/result/payout/history. Добавить promo form и achievements states. Сложная slot-анимация не блокирует функциональность.                                          | Slots и применение promo code доступны пользователю.                                                     |
| 31 августа, пн                 | Добавить tournament leaderboard и завершить admin pages: games, rounds, promo management, технические показатели/переход к dashboards. Пройти Compose smoke всех обязательных экранов.                 | Frontend product feature complete; CP5 пройден.                                                          |
| 1 сентября, вт                 | Перепроверить полный admin flow и при необходимости добавить простой просмотр RTP report. Подготовить environment-based API/GraphQL/WebSocket URLs и проверить frontend image в CI/K3s.                | Один frontend image работает с конфигурацией окружения без rebuild URL; CP5 не зависит от нового экрана. |
| 2 сентября, ср                 | Проверить приложение через K3s Ingress и локальный HTTPS/WSS: auth refresh, cookies/storage, GraphQL base path, WebSocket reconnect. Исправить deployment-specific ошибки.                             | Основной пользовательский путь работает через единый HTTPS endpoint.                                     |
| 3 сентября, чт                 | Провести полный deployed smoke всех пяти игр, profile/history/engagement/admin. Проверить responsive layout, error/loading/empty states и отображение технических показателей.                         | Frontend deployed candidate готов; CP6 пройден.                                                          |
| 4 сентября, пт — FLEX / FREEZE | Закрыть потерянный день либо исправлять P0/P1. Проверить reconnect, slow network, API failure, повторный click и основные разрешения. Новых экранов не добавлять.                                      | Frontend feature freeze; нет известных P0.                                                               |
| 5 сентября, сб                 | Выполнить release-candidate E2E, визуальную полировку двух главных игр, проверить admin permissions и согласованность баланса/истории. Закрыть P0.                                                     | Frontend часть CP7 стабильна.                                                                            |
| 6 сентября, вс                 | Сначала исправить P0. Затем подготовить screenshots, пользовательский demo flow и краткое описание интерфейса. AI widget делать только после завершения общей документации и лишь если остаётся время. | Демонстрационный сценарий готов; AI не блокирует релиз.                                                  |
| 7 сентября, пн                 | Провести финальный smoke на чистом deploy, исправить только release-blocker и отрепетировать показ от регистрации до dashboards.                                                                       | Frontend финального tag подтверждён.                                                                     |

## 8. Ежедневная координация

Начиная с 11 августа:

- Team Lead тратит на плановое ревью 40–60 минут в день.
- Каждый участник оставляет асинхронный отчёт `Сделано / Проверено / Блокер / Следующий шаг`.
- Созвон проводится только при блокирующей зависимости, contract change или интеграционном дефекте, который нельзя локализовать асинхронно.
- Изменения общих contracts принимаются до начала зависимой реализации, а не после готового frontend/backend кода.

В конце flex-дня фиксируется один статус:

- **Green:** checkpoint пройден, следующая фаза начинается по плану.
- **Yellow:** отставание не больше одного дня; используется упрощение UI/глубины необязательных деталей.
- **Red:** checkpoint не работает end-to-end; следующая крупная функция не начинается, пока вертикаль не восстановлена.

## 9. Финальный демонстрационный сценарий

1. Регистрация и login.
2. Получение demo-баланса через разрешённую операцию.
3. Вход двух клиентов в комнату, disconnect и reconnect snapshot.
4. Crash: ставка, cash out/проигрыш, ledger и provably fair.
5. Roulette: несколько ставок и единый server result.
6. Dice Duel: два готовых игрока, несколько бросков и settlement.
7. Coinflip и Slots.
8. History, leaderboard, achievements, promo и tournament.
9. Admin: блокировка, ledger, корректировка, games и promo management.
10. RTP report.
11. Grafana: Platform, Realtime, Casino Business и Kafka.
12. CI image flow, K3s resources, локальный HTTPS и rollback выбранной версии.
