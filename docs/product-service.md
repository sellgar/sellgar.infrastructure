# Product service: dev deployment

Product владеет `code/`, корневыми Dockerfile/CI и `helm/` values. Infrastructure
владеет Argo CD Application, общей Helm-схемой, ролью и базой PostgreSQL.
Рабочая ветка — `development`. Одна реплика, внутренний HTTP/Service 8080,
без внешнего Ingress. Клиент команд — существующий Admin gateway.

## Данные и конфигурация

Целевое имя базы и владельца — `product_srv`. При первом развёртывании база
называлась `product_srv_v2`; переименование сохраняет её данные и права.
Порядок согласованного перехода: [Product naming](product-naming.md).
CloudNativePG создаёт одноимённую роль без superuser из Secret `product-database`.
Database resource сохраняет данные через reclaim policy retain.
Личный пользователь PostgreSQL и его права не изменяются.

Все семь миграций запускаются Sync Job перед Deployment; повторный запуск не
изменяет схему. TypeORM synchronize отключён. ConfigMap приложения монтируется
как `/app/.env`; Secret env refs содержат credentials, отсутствующие в Git/образе.
`PRODUCT_AMQP_URL` собирается Kubernetes env expansion из ранее определённых
`AMQP_USERNAME`/`AMQP_PASSWORD` Secret refs. Нынешние dev credentials URI-safe.
При замене на содержащие URI-разделители потребуется кодирование credentials
для существующего URL-контракта, без помещения пароля в ConfigMap.

Имена очередей в конфигурации — `product.commands` и `product.events`, topic exchange —
`event.exchange`. File events поступают в существующий inbox, исходящие полные
снимки Product — через общую `@sellgar/outbox` в том же процессе.

`PRODUCT_COMMAND_QUEUE` и `PRODUCT_EVENT_QUEUE` не содержат исторического `V2`.
Store и Cart используют `PRODUCT_COMMAND_QUEUE`; оба gateway сохраняют свой ключ
`AMQP_PRODUCT_SRV_COMMAND_QUEUE` со значением `product.commands`.
При переходе работающей среды нужно согласованно переключить Product и клиентов.
RabbitMQ не переименовывает очереди: старые очереди удаляют только после проверки
отсутствия ready/unacknowledged сообщений и старых publishers/consumers. До этого
File events из старой очереди должны быть обработаны, чтобы не потерять проекции.

## Проверки и обновление

CI собирает Node 24 image. Отдельный build stage с PostgreSQL 16 выполняет
104 теста, включая SQL-проекции изображений; семь FTS-тестов запускаются отдельно
по env-флагу. Бинарники PostgreSQL отсутствуют в runtime-образе.

Production-image smoke создаёт только временные PostgreSQL/RabbitMQ, проверяет
пустую базу, семь миграций и повторный запуск, RMQ catalog create/read/update,
поиск, варианты и изображения, получение File events, ответы 400/409 и четыре
упорядоченных полных снимка из outbox. Runtime работает non-root/read-only с
монтируемым `.env`; отключение проверочного RabbitMQ переводит health в 503.

`/health` проверяет PostgreSQL и оба RMQ consumer; liveness проверяет TCP.
Общий chart: RollingUpdate maxSurge 1/maxUnavailable 0, readiness и grace period.
Это не доказательство отсутствия потерь всех активных RMQ-команд при rollout.

После CI закрепить digest/source/CI run в `environments/dev/services/product-service/`.
Проверить Synced/Healthy, Deployment 1/1, таблицу migrations и сценарий создания
товара через dev-админку с последующим повторным чтением. Store/File/Media
разворачиваются отдельно; их готовность этим этапом не подтверждается.


## Результат проверки 2026-11-20

CI `44614717883` прошёл build/smoke/publish; source
`a76bd00eaedd8cfa84752c21771789b20fcb7d4c`, digest
`sha256:f31bb58d3850341588dc39e3cfe0a21eab9b6f9d759cdc4744308b48e2d163a7`.
Argo CD Synced/Healthy, Product 1/1; в `migrations` семь записей.

В dev-админке созданы бренд `dev-brand`, шаблон
`1035968a-c159-44f0-b62c-ebc91ac2a5db` и товар
`6b898082-fcdf-4a35-94ee-b248c23ac6b6` с вариантом. После перезагрузки страницы
товар присутствует в каталоге и повторно открывается с сохранёнными данными;
его outbox-событие опубликовано. Список сразу после создания остаётся старым —
отдельный frontend-дефект синхронизации, как у Shop. Импорт/загрузка изображений
через File/Media и применение событий в Store пока не проверяются в dev,
поскольку эти сервисы ещё не развёрнуты. Изображения и доставка событий проверены
в изолированном production-image smoke. Prod не развёрнут.
