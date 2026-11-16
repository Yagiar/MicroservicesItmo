# Каталог микросервисов

Вариант 2226, логистическая платформа. Формат описания взят из задания, к нему добавлено поле `Data Owner`, потому что задание требует назвать владельца данных.

Общие правила для всех сервисов:

- Данные сервиса меняет только он сам. Другие сервисы получают их через API или события, общих БД нет.
- События публикуются через Transactional Outbox, потребители идемпотентны.
- Каждый сервис сам проверяет JWT и права доступа к своим данным, внутренние вызовы идут с токеном вызывающего сервиса ([README, раздел 4](README.md#4-границы-сервисов-ответственность-и-владение-данными)).
- Число подов каждого развёртывания меняет HPA по метрике из поля `Scalability`, минимум 2 пода в каждом ЦОДе ([раздел 10 отчёта](README.md#10-балансировка-и-автомасштабирование)).
- В поле `API` перечислены все бизнес-эндпоинты сервиса и кто их вызывает. Кроме них каждое развёртывание (включая воркеры) отдаёт служебные `GET /health/live`, `GET /health/ready` (для `livenessProbe` и `readinessProbe` Kubernetes) и `GET /metrics` (для Prometheus). Служебные эндпоинты доступны только внутри кластера, шлюзы их наружу не публикуют.
- Списки возвращаются постранично по курсору (`cursor`, `limit`), ошибки — в формате Problem Details (RFC 9457), создающие запросы принимают заголовок `Idempotency-Key`.
- Внешний трафик проходит через два шлюза: Ingest Gateway принимает телеметрию, Public Gateway обслуживает остальной API (README, раздел 4). Это инфраструктурные компоненты без бизнес-логики и своих данных, поэтому в каталог они не входят.

**Размещение по ЦОДам.** Все развёртывания и хранилища работают в каждом из трёх ЦОДов, схема репликации описана в [разделе 9 отчёта](README.md#9-размещение-и-репликация). Поэтому в описаниях сервисов это не повторяется. Что это гарантирует и чего не гарантирует:

| Гарантирует | Не гарантирует |
|---|---|
| Отказ пода, узла или целого ЦОДа не останавливает ни один сервис: два оставшихся ЦОДа рассчитаны на весь пик (запас 50 % в каждом) | Защиту от ошибки в коде или конфигурации: одна и та же версия работает во всех ЦОДах. Против этого выкладка идёт по одному ЦОДу с автоматическим откатом по росту доли ошибок |
| Подтверждённая запись в PostgreSQL, Cassandra и Kafka не теряется при отказе ЦОДа (RPO = 0) | Непрерывность записи в PostgreSQL: пока Patroni переключает лидера, запись в этот сервис недоступна до 1 мин |
| Чтение идёт из локальной реплики своего ЦОДа | Работу при отказе двух ЦОДов сразу: запись в PostgreSQL останавливается намеренно, переключение ручное |

## Shipment Service

```text
Service: Shipment Service

Responsibility:
Приём заказов перевозки и описание груза.
Жизненный цикл заказа: единственный источник истины о статусе заказа.
Фиксация цены: в заказ копируется снимок котировки, новый тариф принятый заказ не меняет.
Участие в саге назначения рейса: публикует подтверждение и отмену заказа, принимает результат назначения.
Отложенная фиксация цены: фоновая задача повторно проверяет котировку у заказов в PRICE_PENDING.

Owned Data:
Shipment (агрегат)
  shipmentId, customerId, quoteId, адреса и координаты забора и доставки,
  окно забора и окно доставки, контакт на месте,
  зафиксированная цена (сумма, валюта, версия тарифа), статус, версия агрегата
Cargo (часть агрегата Shipment)
  тип груза, вес, объём, число мест, признаки: опасный, хрупкий, температурный режим
ShipmentStatusHistory: переходы статусов с причиной и временем
Outbox: события, ожидающие публикации в Kafka
Хранилище: PostgreSQL, чтение списков и истории идёт в реплики

Жизненный цикл (ShipmentStatus):
PRICE_PENDING -> CONFIRMED          цена зафиксирована после восстановления Pricing
CONFIRMED -> ASSIGNED               RouteAssigned
CONFIRMED -> ASSIGNMENT_FAILED      RouteAssignmentFailed
ASSIGNMENT_FAILED -> CONFIRMED      повторное назначение по запросу диспетчера
ASSIGNED -> IN_TRANSIT              веха ShipmentPickedUp
IN_TRANSIT -> DELIVERED             веха ShipmentDelivered
PRICE_PENDING, CONFIRMED, ASSIGNED, ASSIGNMENT_FAILED -> CANCELLED

Инварианты:
Отменить можно только до забора груза (до IN_TRANSIT).
Изменить адрес, окно или груз можно только до назначения рейса (до ASSIGNED).
Цена после фиксации не меняется.
Переход проверяется по версии агрегата, повторное или запоздавшее событие не откатывает статус.

Data Owner:
Команда Orders

API:
Клиент (только свои заказы) и диспетчер (все заказы):
POST /shipments                     создать заказ по quoteId, заголовок Idempotency-Key защищает от дублей;
                                    201 и статус CONFIRMED или PRICE_PENDING, 422 при просроченной котировке
GET /shipments/{id}                 заказ с грузом, ценой и текущим статусом
GET /shipments?customerId={id}&status={status}&from={date}&to={date}&cursor={c}
                                    список заказов; клиенту customerId подставляется из токена,
                                    диспетчер может искать по всем клиентам
PATCH /shipments/{id}               изменить контакт, окна или груз до назначения рейса, 409 после ASSIGNED;
                                    заголовок If-Match с версией агрегата защищает от одновременной правки
POST /shipments/{id}/cancel         отмена с указанием причины, 409 после забора груза
GET /shipments/{id}/history         история статусов с причинами и временем

Только диспетчер:
POST /shipments/{id}/reassign       повторное назначение после ASSIGNMENT_FAILED

Dependencies:
Pricing Service: проверка котировки при создании (таймаут 300 мс, circuit breaker);
  если он недоступен, заказ принимается в статусе PRICE_PENDING
Route Service: события назначения рейса (только Kafka)
Tracking Service: вехи доставки (только Kafka)

Communication:
REST + Kafka
Синхронно: GET /quotes/{id} в Pricing
Публикует: shipment.events
  ShipmentConfirmed (цена, адреса, окна, параметры груза: всё, что нужно Route для подбора)
  ShipmentCancelled (причина)
  ShipmentStatusChanged (новый статус, версия агрегата)
Потребляет: route.events, tracking.milestones

Scalability:
Горизонтальное масштабирование по RPS, экземпляры не хранят состояния.
Чтение (около 97 % из 500 RPS) идёт в реплики, запись около 12 RPS в пик обслуживает лидер.
При росте возможно шардирование по shipmentId.
```

## Pricing Service

```text
Service: Pricing Service

Responsibility:
Расчёт стоимости перевозки, управление версиями тарифов, выдача котировок с ограниченным сроком действия.

Owned Data:
Tariff (версии неизменяемы, изменение создаёт новую версию с датой начала действия validFrom)
Quote (неизменяема, срок действия 30 минут)
Хранилище: PostgreSQL; кэш Redis

Data Owner:
Команда Pricing & Comms

API:
Клиент и диспетчер:
POST /quotes                        рассчитать котировку: точки забора и доставки, груз, тип ТС;
                                    ответ — цена, версия тарифа, срок действия, признак приближённой оценки
GET /quotes/{id}                    котировка; клиент видит только свои
                                    (этот же эндпоинт вызывает Shipment при создании заказа)

Только диспетчер:
GET /tariffs?activeAt={time}        тарифы, действующие на момент времени (по умолчанию — сейчас)
GET /tariffs/{id}                   тариф с текущей версией
GET /tariffs/{id}/versions          история версий тарифа
POST /tariffs                       создать тариф с первой версией
POST /tariffs/{id}/versions         опубликовать новую версию, validFrom не раньше чем через 5 минут
DELETE /tariffs/{id}/versions/{version}
                                    отменить версию, которая ещё не вступила в силу; 409, если уже действует

Dependencies:
Route Service: оценка расстояния и времени; если он недоступен, используется приближённая оценка по прямому расстоянию
Redis: кэш; если он недоступен, запросы идут напрямую в Route и PostgreSQL

Communication:
REST
Синхронно: POST /routes/estimate в Route (таймаут 500 мс, circuit breaker)

Кэш (cache-aside):
Оценки расстояния: Redis, ключ — геохеш точки отправления, геохеш точки назначения, тип ТС,
  TTL 24 часа. Повторные расчёты по одному направлению не доходят до Route,
  поэтому синхронная цепочка Pricing -> Route -> Tile Service срабатывает только при промахе.
Котировки: Redis, ключ quoteId, TTL равен сроку действия котировки.
  Котировка неизменяема, поэтому инвалидация не нужна; проверка котировки
  из Shipment обычно не доходит до PostgreSQL.
Активные тарифы: в памяти экземпляра, перечитываются каждые 60 секунд.
  Новая версия тарифа вступает в силу не раньше чем через 5 минут после публикации,
  поэтому все экземпляры начинают применять её одновременно.
Redis Pricing — отдельный экземпляр в каждом ЦОДе без репликации между ЦОДами:
  потеря кэша приводит только к промахам, а не к потере данных.

Scalability:
Горизонтальное масштабирование по CPU. Экземпляры не хранят состояния, тарифы читаются из кэша.
```

## Route Service (API)

Route Service — один bounded context и одна команда, но два развёртывания с разным профилем нагрузки: API отвечает на синхронные запросы, а [Route Planner](#route-planner-воркер-route-service) выполняет тяжёлое планирование асинхронно. Оба развёртывания собираются из одного репозитория и работают с одной БД Route, другие сервисы к ней доступа не имеют.

```text
Service: Route Service (API)

Responsibility:
Синхронные запросы к маршрутам: оценка расстояния и времени для Pricing,
просмотр маршрутов, приём запросов на перепланирование от диспетчера.
Сам маршруты не строит и ресурсы не бронирует, это делает Route Planner.

Owned Data:
Route (чтение; создаёт и меняет Route Planner)
PlanningRequest (запросы на перепланирование, передаются в Route Planner через outbox)
Хранилище: PostgreSQL + PostGIS, общее с Route Planner

Data Owner:
Команда Routing

API:
Внутренний (вызывает Pricing):
POST /routes/estimate               расстояние и время между двумя точками для типа ТС

Клиент (маршрут своего заказа) и диспетчер:
GET /routes/{id}                    маршрут: точки, плановое время, назначенные ТС и водитель, статус
GET /routes?shipmentId={id}         маршрут по заказу

Диспетчер и водитель (водитель видит только свои рейсы):
GET /routes?status={status}&vehicleId={id}&driverId={id}&date={date}&cursor={c}
                                    рейсы за день, по транспорту или водителю

Только диспетчер:
POST /routes/{id}/replan            ставит запрос на перепланирование с причиной, отвечает 202 Accepted
GET /routes/{id}/planning-requests  запросы на перепланирование и их результат

Dependencies:
Tile Service: матрица расстояний для /routes/estimate

Communication:
REST + Kafka
Синхронно: POST /matrix в Tile Service
Публикует: route.planning-requests (внутренний топик, читает только Route Planner)

Scalability:
Горизонтальное масштабирование по RPS, экземпляры не хранят состояния.
Нагрузка около 150 RPS вместе с Fleet, обычно хватает минимума HPA.
```

## Route Planner (воркер Route Service)

```text
Service: Route Planner (воркер Route Service)

Responsibility:
Построение маршрута по подтверждённому заказу.
Подбор кандидатов (ТС и водитель) по проекции доступности и бронирование пары в Fleet.
Публикация результата назначения: RouteAssigned или RouteAssignmentFailed.
Перепланирование по запросу диспетчера или при отказе назначенного ТС.
Компенсации саги: при отмене заказа отменяет рейс и снимает бронь,
при доставке завершает рейс и освобождает ресурсы.
Ведение проекции доступности из fleet.events.

Owned Data:
Route (создаёт и меняет)
Проекция доступности ТС и водителей (копия из fleet.events, источник истины остаётся в Fleet)
Outbox
Хранилище: PostgreSQL + PostGIS, общее с Route API

Data Owner:
Команда Routing

API:
Бизнес-эндпоинтов нет: входы — только топики Kafka (см. Communication),
есть лишь служебные /health/live, /health/ready и /metrics.
Состояние маршрутов, которые строит воркер, читается через Route API.

Dependencies:
Tile Service: геокодинг адресов и матрица расстояний
Fleet Service: POST /reservations, DELETE /reservations/{id};
  при ответе 409 берётся следующий кандидат;
  GET /availability только для начального наполнения проекции доступности
Shipment Service: события заказа (только Kafka)

Communication:
Kafka + REST
Потребляет: shipment.events, fleet.events, route.planning-requests
Публикует: route.events (RouteAssigned, RouteAssignmentFailed)
Синхронно: Tile Service, Fleet Service
Ключ партиции shipmentId: все события одного заказа обрабатывает один экземпляр по порядку.
Подбор ограничен таймаутом 2 минуты, после него публикуется RouteAssignmentFailed.

Scalability:
Масштабируется отдельно от Route API: по отставанию consumer group и CPU,
так как построение маршрута требует вычислений.
Максимум подов не больше числа партиций shipment.events.
Сбой воркера не влияет на Route API: события ждут в Kafka и обрабатываются после восстановления.
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
Диспетчер — транспорт:
POST /vehicles                      добавить ТС: номер, тип, грузоподъёмность, объём, документы
GET /vehicles?status={status}&type={type}&cursor={c}
GET /vehicles/{id}
PATCH /vehicles/{id}                изменить характеристики и документы
PUT /vehicles/{id}/status           ACTIVE, MAINTENANCE, DECOMMISSIONED; публикует VehicleStatusChanged

Диспетчер — водители:
POST /drivers                       добавить водителя: ФИО, категории прав, контакты
GET /drivers?status={status}&cursor={c}
GET /drivers/{id}                   профиль без контактов
PATCH /drivers/{id}                 изменить категории прав и данные

Диспетчер и водитель (водитель только про себя):
PUT /drivers/{id}/status            ON_SHIFT, OFF_SHIFT, SICK_LEAVE; публикует DriverStatusChanged

Внутренние (только с токеном сервиса):
GET /drivers/{id}/contact           телефон и push-адресат водителя, вызывает только Notification
GET /availability?from={from}&to={to}&minCapacityKg={kg}
                                    свободные пары ТС и водитель; Route Planner использует его
                                    для начального наполнения проекции доступности
POST /reservations                  забронировать пару на интервал; идемпотентный по Idempotency-Key,
                                    при конфликте 409; вызывает Route Planner
DELETE /reservations/{id}           снять бронь (отмена или завершение рейса)

Диспетчер:
GET /reservations?vehicleId={id}&driverId={id}&from={from}&to={to}
GET /reservations/{id}

Dependencies:
Нет синхронных зависимостей от других сервисов

Communication:
REST + Kafka
Публикует: fleet.events

Scalability:
Горизонтальное масштабирование по CPU, нагрузка низкая, обычно хватает минимума HPA: 2 пода в каждом ЦОДе.
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
Tracking Ingest (через Ingest Gateway; только трекер или водитель):
POST /tracking/events               пакет отметок: позиции и вехи (груз забран, груз доставлен);
                                    у каждой отметки eventId, повторная отправка не создаёт дублей;
                                    ответ 202 после записи в Kafka, 503 при недоступности Kafka

Tracking Query (через Public Gateway):
Клиент (только свои заказы) и диспетчер:
GET /tracking/shipments/{id}/latest              последняя позиция транспорта по заказу (Redis)
GET /tracking/shipments/{id}/events?from={t}&to={t}&cursor={c}
                                                 история перемещений по заказу (Cassandra)
GET /tracking/shipments/{id}/milestones          вехи доставки с временем

Только диспетчер:
GET /tracking/vehicles/{id}/latest               последняя позиция ТС
GET /tracking/vehicles/latest?ids={id1,id2,...}  последние позиции группы ТС для карты диспетчера
GET /tracking/vehicles/{id}/events?from={t}&to={t}&cursor={c}
                                                 история ТС за период, не больше суток за запрос

Tracking Writer: бизнес-эндпоинтов нет, вход — топики Kafka.

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
```

## Notification Service

```text
Service: Notification Service

Responsibility:
Формирование уведомлений из доменных событий по шаблонам, доставка по каналам (SMS, e-mail, push, в приложении),
повторные попытки и учёт статуса доставки, настройки получателей, регистрация устройств для push.

Owned Data:
Notification (получатель, канал, шаблон, статус доставки, число попыток)
NotificationPreference (каналы по типам событий, тихие часы)
NotificationTemplate (версии шаблонов по типу события, каналу и языку)
Device (push-токены приложений клиента и водителя)
Хранилище: PostgreSQL

Data Owner:
Команда Pricing & Comms

API:
Получатель (клиент, водитель, диспетчер; доступ только к своим данным):
GET /notifications?recipientId={id}&status={status}&channel={channel}&cursor={c}
                                        история уведомлений, постранично
GET /notifications/{id}                 одно уведомление со статусом доставки
GET /notifications/unread-count?recipientId={id}
POST /notifications/{id}/read           отметить уведомление в приложении прочитанным
POST /notifications/read-all?recipientId={id}
GET /preferences/{recipientId}
PUT /preferences/{recipientId}          каналы по типам событий, тихие часы
POST /devices                           зарегистрировать push-токен устройства
DELETE /devices/{id}                    отключить push на устройстве

Диспетчер:
POST /notifications                     ручное сообщение водителю или клиенту по заказу
POST /notifications/{id}/resend         повторная отправка, например по другому каналу
GET /templates                          список шаблонов по типам событий и каналам
GET /templates/{eventType}/{channel}    текущая версия шаблона
PUT /templates/{eventType}/{channel}    новая версия шаблона
POST /templates/{eventType}/{channel}/preview
                                        отрисовать шаблон на тестовых данных без отправки

Провайдеры связи (подпись запроса проверяется):
POST /webhooks/{provider}/delivery-status   подтверждение доставки или ошибка

Dependencies:
Shipment, Route, Tracking: источники событий
Fleet Service: контакт водителя
Identity Provider: контакт клиента (через ACL)
Провайдеры SMS, e-mail, push (через ACL)

Communication:
Kafka + REST
Потребляет: shipment.events, route.events, tracking.milestones
Синхронно: GET /drivers/{id}/contact в Fleet, контакт клиента в Identity Provider, отправка провайдерам
Неудачная отправка повторяется с экспоненциальной задержкой, затем пробуется следующий канал из настроек.

Scalability:
Горизонтальное масштабирование по отставанию consumer group.
При сбое уведомления задерживаются, но не теряются: непрочитанные события остаются в Kafka.
```

## Tile Service

```text
Service: Tile Service

Responsibility:
Типовой (generic) картографический сервис внутри платформы:
выдача тайлов карты для страниц отслеживания, кабинета диспетчера и приложения водителя,
геокодинг и обратный геокодинг адресов, построение пути и матрица расстояний и времени.
Бизнес-логики логистики не содержит, ничего не знает о заказах и рейсах.

Owned Data:
Набор данных карты одной версии, собранный из выгрузки OpenStreetMap по региону работы:
  векторные тайлы (MBTiles), граф дорог для расчёта пути (OSRM), индекс геокодинга
Хранилище: файлы тайлов и графа на томах подов, индекс геокодинга в PostgreSQL + PostGIS.
Данные только для чтения: каждый ЦОД держит свою полную копию, репликация не нужна.

Data Owner:
Команда Routing (основной потребитель)

API:
Клиент, диспетчер, водитель (через Public Gateway):
GET /tiles.json                         метаданные тайлов (TileJSON): текущая версия данных, зумы, границы региона
GET /tiles/{version}/{z}/{x}/{y}.pbf    векторный тайл; версия в пути, поэтому Cache-Control: immutable
GET /styles/{name}.json                 стиль отрисовки карты для веб-клиента и приложения
GET /geocode?q={address}&limit={n}      адрес в координаты, подсказки при вводе адреса
GET /reverse?lat={lat}&lon={lon}        координаты в адрес

Внутренние (Route API и Route Planner):
POST /directions                        путь между точками для типа ТС: геометрия, расстояние, время
POST /matrix                            матрица расстояний и времени, не больше 100 × 100 точек

Служебный (только команда Routing):
GET /dataset                            версии набора данных: активная, предыдущая, дата выгрузки OSM

Dependencies:
Нет синхронных зависимостей от других сервисов.
Выгрузка OpenStreetMap загружается раз в неделю пакетной задачей, во время работы не нужна.

Communication:
REST
Клиенты получают тайлы через Public Gateway, Route API и Route Planner вызывают /matrix, /directions и /geocode.
Обновление данных: новая версия собирается рядом со старой, проверяется и включается переключением версии,
откат — переключение обратно. После переключения /tiles.json отдаёт новую версию,
и клиенты начинают запрашивать тайлы по новым адресам, поэтому кэш не нужно сбрасывать.

Scalability:
Горизонтальное масштабирование по RPS и CPU, экземпляры не хранят изменяемого состояния.
Тайлы неизменяемы в пределах версии, поэтому кэшируются в браузере и на Public Gateway,
до сервиса доходит малая часть запросов.
```
