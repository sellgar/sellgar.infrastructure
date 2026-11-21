# Store service: dev deployment

Store владеет `code/`, корневыми Dockerfile/CI и `helm/` values. Infrastructure
владеет Argo CD Application, отдельной ролью/базой и общим chart application.
Рабочая ветка — `development`; одна реплика, внутренние container/Service 8080,
без Ingress. Доменные API остаются в существующих Admin/Client gateways.

## База и конфигурация

CloudNativePG создаёт database/login `store_srv` из Secret `store-database`,
без superuser. Database resource использует template0, UTF8 и C.UTF-8 locale;
retain reclaim policy сохраняет данные. Locale важна для русских lower/FTS и
уникальности соседних категорий без учёта регистра. Первая пустая база, созданная
с наследованной C-locale, пересоздана до запуска миграций и приложения.
Другие базы с данными и личный пользователь `sellgar` этим этапом не менялись.

Пять миграций запускает Sync Job до Deployment; synchronize отключён.
`.env` монтируется из ConfigMap `/app/.env`, пароли — Secret env refs.
Очереди `store_srv.command.queue`, `store_srv.event.queue`; exchange event.exchange,
каталог — product.commands. Outbox подключён как общая библиотека процесса Store.

`/health` проверяет PostgreSQL и оба RMQ-consumer. Общий chart задаёт readiness,
startup probe, RollingUpdate maxSurge 1/maxUnavailable 0, minReadySeconds и grace.
Полное отсутствие потерь всех in-flight RMQ-команд при rollout отдельно не доказано.

## Первичное восстановление dev-проекций

Store стартует с пустой базой, без переноса локальных таблиц. Его очередь появляется
после первых записей Shop/Product; уже опубликованные события туда не поступали.
`bootstrap/dev/initialize-store.py <kubectl> --kubeconfig <config>` адресно повторяет
сохранённые опубликованные shop/product события из dev-владельцев только в очередь
Store. Сохраняются исходные eventUuid, версии и порядок внутри агрегата. Отправка
использует publisher confirms, источник читается без изменения outbox. Inbox Store
сам применяет события; повторный запуск безопасен для уже обработанных событий.
Скрипт требует доступный Store, существующую очередь и непрерывную сохранённую
историю от версии 1; разрыв останавливает replay. Это начальный dev bootstrap,
а не универсальный backup/recovery после очистки outbox или изменения event schema.
Credentials читаются внутри pod, значения в Git/вывод не передаются.

Тем же скриптом через существующие RPC создаётся только отсутствующая RUB —
принятая валюта dev-магазинов; существующая запись сохраняется. Товары, цены,
остатки и категории для приёмки заводятся через интерфейс админки.

## Проверки

Docker verifier использует отдельную временную PostgreSQL 16 с UTF-8 locale:
22 unit-теста и 46 integration-тестов. Проверяются транзакции учёта, FK, категории,
идемпотентность, FTS, порядок outbox и восстановление Product snapshots. Бинарники
БД и dev dependencies отсутствуют в финальном Node 24 image.

Production-image smoke использует только временные PostgreSQL/RabbitMQ. Проверяет
пустую базу, пять миграций и повторный запуск, inbox Shop/Product, дерево категорий,
привязку листа, decimal цену, порядок двух учётных записей, приход, повтор команды,
409 на старую версию, FTS/фильтры, outbox и health 503 после отключения брокера.

После rollout проверить Argo Synced/Healthy, Deployment 1/1, пять migrations,
применённые проекции, создание Store Product через UI и реальное последующее
изменение Product → RabbitMQ → Store. File/Media/Cart и prod — следующие этапы.

## Известное ограничение среды

Проверка выявила C-locale у ранее созданных dev-баз. Они не пересоздавались,
поскольку уже содержат данные. Их перевод на Unicode locale с сохранением данных
требует отдельной миграции; исправление тестового initdb само по себе их не меняет.
