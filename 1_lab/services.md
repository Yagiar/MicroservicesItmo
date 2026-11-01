# Каталог микросервисов

Вариант 2226, логистическая платформа. Формат описания взят из задания, к нему добавлено поле `Data Owner`, потому что задание требует назвать владельца данных.

Общие правила для всех сервисов:

- Данные сервиса меняет только он сам. Другие сервисы получают их через API или события, общих БД нет.
- События публикуются через Transactional Outbox, потребители идемпотентны.
- Внешний трафик проходит через API Gateway. Это инфраструктурный компонент без бизнес-логики и своих данных, поэтому в каталог он не входит.

## Shipment Service

```text
Service: Shipment Service

Responsibility:
Приём заказов перевозки, описание груза, жизненный цикл заказа, хранение зафиксированной цены.

Owned Data:
Shipment
Cargo (часть агрегата Shipment)
Хранилище: PostgreSQL с read-репликами

Data Owner:
Команда Orders

API:
POST /shipments
GET /shipments/{id}
GET /shipments?customerId={id}
POST /shipments/{id}/cancel

Dependencies:
Pricing Service: проверка котировки; если он недоступен, заказ принимается в статусе PRICE_PENDING
Route Service: события назначения рейса
Tracking Service: вехи доставки

Communication:
REST + Kafka
Публикует: shipment.events
Потребляет: route.events, tracking.milestones

Scalability:
Горизонтальное масштабирование, минимум 3 экземпляра в разных зонах доступности.
Чтение идёт в реплики, при росте возможно шардирование по shipmentId.
```

## Pricing Service

```text
Service: Pricing Service

Responsibility:
Расчёт стоимости перевозки, управление версиями тарифов, выдача котировок с ограниченным сроком действия.

Owned Data:
Tariff (версии неизменяемы, изменение создаёт новую версию)
Quote
Хранилище: PostgreSQL, тарифы кэшируются в памяти экземпляра

Data Owner:
Команда Pricing & Comms

API:
POST /quotes
GET /quotes/{id}
GET /tariffs
POST /tariffs

Dependencies:
Route Service: оценка расстояния и времени; если он недоступен, используется приближённая оценка по прямому расстоянию

Communication:
REST

Scalability:
Горизонтальное масштабирование по CPU. Экземпляры не хранят состояния, тарифы читаются из кэша.
```

## Route Service

```text
Service: Route Service

Responsibility:
Построение маршрутов, подбор транспорта и водителя, назначение рейса и его перепланирование.

Owned Data:
Route
Проекция доступности транспорта (копия из fleet.events, источник истины остаётся в Fleet)
Хранилище: PostgreSQL + PostGIS

Data Owner:
Команда Routing

API:
POST /routes/estimate
GET /routes/{id}
GET /routes?shipmentId={id}
POST /routes/{id}/replan

Dependencies:
Fleet Service: бронирование транспорта и водителя
Shipment Service: события заказа
Картографический сервис: геокодинг и матрица расстояний (через ACL)

Communication:
REST + Kafka
Публикует: route.events
Потребляет: shipment.events, fleet.events

Scalability:
API и воркеры планирования масштабируются отдельно. Воркеры масштабируются по отставанию
consumer group и CPU, так как построение маршрута требует вычислений.
```

## Fleet Service

```text
Service: Fleet Service

Responsibility:
Учёт транспорта и водителей, их статусы и доступность, бронирование пары «транспорт и водитель».

Owned Data:
Vehicle
Driver
Reservation (уникальность брони защищает от двойного назначения)
Хранилище: PostgreSQL, схемы vehicle и driver без межсхемных JOIN

Data Owner:
Команда Fleet

API:
POST /vehicles
GET /vehicles/{id}
POST /drivers
GET /drivers/{id}
GET /availability?from={from}&to={to}&minCapacityKg={kg}
POST /reservations (идемпотентный, при конфликте возвращает 409)
DELETE /reservations/{id}

Dependencies:
Нет синхронных зависимостей от других сервисов

Communication:
REST + Kafka
Публикует: fleet.events

Scalability:
Горизонтальное масштабирование, нагрузка низкая, достаточно 2–3 экземпляров.
Write-операции бронирования упираются в primary PostgreSQL, чтение можно вынести в реплики.
```

## Tracking Service

```text
Service: Tracking Service

Responsibility:
Приём телеметрии и вех доставки, хранение истории, выдача текущей позиции и истории перемещений.

Owned Data:
TrackingEvent (неизменяемые записи)
Проекция «заказ — транспорт» (копия из route.events)
Хранилище: Cassandra (партиция vehicleId + день, TTL 90 дней), Redis для последней позиции

Data Owner:
Команда Tracking

API:
POST /tracking/events (пакетная отправка)
GET /tracking/shipments/{id}/latest
GET /tracking/shipments/{id}/events

Dependencies:
Route Service: привязка транспорта к заказу (только события)
Телематика: источник GPS-данных (через ACL)

Communication:
REST + Kafka
Публикует: tracking.positions (внутренний топик), tracking.milestones
Потребляет: tracking.positions, route.events

Scalability:
Три независимо масштабируемых развёртывания одного сервиса:
Ingest масштабируется по RPS, Writer по отставанию consumer group, Query по RPS.
Минимум 3 экземпляра Ingest в разных зонах доступности.
```

## Notification Service

```text
Service: Notification Service

Responsibility:
Формирование уведомлений из доменных событий, доставка по каналам, учёт статуса доставки и настроек получателей.

Owned Data:
Notification
NotificationPreference
Хранилище: PostgreSQL

Data Owner:
Команда Pricing & Comms

API:
GET /notifications?recipientId={id}
PUT /preferences/{recipientId}

Dependencies:
Shipment, Route, Tracking: источники событий
Fleet Service: контакт водителя
Identity Provider: контакт клиента (через ACL)
Провайдеры SMS, e-mail, push (через ACL)

Communication:
Kafka + REST
Потребляет: shipment.events, route.events, tracking.milestones

Scalability:
Горизонтальное масштабирование по отставанию consumer group.
При сбое уведомления задерживаются, но не теряются: непрочитанные события остаются в Kafka.
```
