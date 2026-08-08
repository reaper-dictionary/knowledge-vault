# Protobuf-контракты и генерация Python-кода

# 1. Краткие определения — выучить

## Protocol Buffers

> **Protobuf** — язык описания сообщений и бинарный формат сериализации. `.proto` является исходным контрактом, а не описанием таблицы БД.

## Message и field number

> **Message** — структура данных. **Field number** — постоянный номер поля в бинарном формате; после публикации его нельзя назначать другому смыслу.

## Service и RPC

> **Service** — набор удалённых операций. **RPC** — одна операция с конкретными request и response.

## Package

> **Package** — namespace и версия контракта, например `orders.v1`.

## Stub и Servicer

> **Stub** — сгенерированный клиент. **Servicer** — базовый серверный класс, методы которого реализует разработчик.

## Generated code

> **Generated code** — Python-модули, созданные из `.proto`. Их не редактируют вручную.

---

# 2. Структура `.proto`

~~~proto
syntax = "proto3";

package orders.v1;

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
}

message GetOrderRequest {
  string order_id = 1;
}

message GetOrderResponse {
  Order order = 1;
}

message Order {
  string order_id = 1;
  OrderStatus status = 2;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_NEW = 1;
  ORDER_STATUS_COMPLETED = 2;
}
~~~

Путь обычно повторяет package:

~~~text
contracts/orders/v1/order_service.proto
package orders.v1;
~~~

Правила типов:

- UUID — `string` с валидацией в сервисе;
- деньги — `int64 amount_minor` и currency, не `float`;
- время — `google.protobuf.Timestamp`;
- список — `repeated`;
- scalar с важным presence — `optional`;
- взаимоисключающие варианты — `oneof`;
- первый enum — `*_UNSPECIFIED = 0`.

Ошибки передаются gRPC status codes: `INVALID_ARGUMENT`, `NOT_FOUND`, `FAILED_PRECONDITION`, `UNAVAILABLE`. Stack trace в response не помещается.

---

# 3. Алгоритм написания контракта

1. Определить сервис-владелец операции.
2. Записать, кто вызывает RPC и какое решение ожидает.
3. Создать отдельные request и response messages.
4. Добавить только необходимые поля.
5. Назначить field numbers и больше не менять их смысл.
6. Для повторяемой command добавить `idempotency_key`.
7. Для конкурентного изменения при необходимости добавить `expected_revision`.
8. Определить gRPC status для каждого отказа.
9. Выполнить format, lint и генерацию.
10. Реализовать Servicer, Stub client и contract tests.

`request_id` нужен для логов и tracing. Он не заменяет `idempotency_key`.

---

# 4. Генерация Python-кода

~~~bash
python -m grpc_tools.protoc \
  -I contracts \
  --python_out=generated \
  --pyi_out=generated \
  --grpc_python_out=generated \
  contracts/orders/v1/order_service.proto
~~~

В репозитории команду лучше хранить в одном script или Make target, чтобы разработчики и CI генерировали код одинаково.

| Файл | Содержимое |
|---|---|
| `*_pb2.py` | Messages и enums |
| `*_pb2.pyi` | Type hints |
| `*_pb2_grpc.py` | Stub, Servicer и registration function |

После изменения:

~~~text
format → lint → generate → tests → commit
~~~

Если generated-файлы хранятся в Git, `.proto` и результат генерации коммитятся вместе. Альтернатива — всегда генерировать их во время build. Подход нельзя смешивать случайным образом.

---

# 5. Использование generated-кода

Сервер наследует Servicer и регистрирует implementation на gRPC server. Клиент создаёт channel и Stub:

~~~python
channel = grpc.aio.insecure_channel("order-service:50051")
stub = order_service_pb2_grpc.OrderServiceStub(channel)
response = await stub.GetOrder(request, timeout=2)
~~~

Deadline задаётся явно. В Compose и Kubernetes используется DNS-имя сервиса, а не `localhost`.

---

# 6. Совместимость

- не переиспользовать field number;
- удалённые номера и имена помещать в `reserved`;
- новые поля добавлять новыми номерами;
- не менять тип и смысл опубликованного поля;
- несовместимый контракт выпускать как новый package, например `orders.v2`;
- generated-код обновлять сразу после `.proto`.

---

# 7. Типовые ошибки

- редактировать `*_pb2.py` вместо `.proto`;
- хранить package не в соответствующей директории;
- считать успешную compilation заменой Buf lint;
- возвращать ORM model как сетевой контракт;
- переиспользовать старый field number;
- забывать повторную генерацию;
- вызывать другой контейнер через `localhost`.

---

# 8. Чек-лист

- владелец RPC определён;
- request и response имеют отдельные messages;
- field numbers стабильны;
- command учитывает duplicate request;
- ошибки сопоставлены с gRPC statuses;
- package совпадает с директорией;
- format, lint, generation и tests выполнены;
- generated-файлы не редактировались вручную.

## Связанные заметки

- [[02. Синхронное взаимодействие gRPC и устойчивость]]
- [[01. Границы сервисов и владение данными]]
