# Имена Product

Текущая конфигурация использует:

| Назначение | Имя |
|---|---|
| PostgreSQL database / managed login | `product_srv` |
| Producer событий каталога | `product_srv` |
| Command queue | `product.commands` |
| File event queue Product | `product.events` |
| Environment keys Product | `PRODUCT_COMMAND_QUEUE`, `PRODUCT_EVENT_QUEUE` |
| Environment key Store / Cart | `PRODUCT_COMMAND_QUEUE` |

Gateway сохраняют ключ `AMQP_PRODUCT_SRV_COMMAND_QUEUE` со значением
`product.commands`. Exchange `event.exchange`, команды `catalog.v2.*`, HTTP API
и schemaVersion не переименовываются: это версии контрактов, а не имя старой базы.

## Переход работающего dev

Изменение исходников не означает переименование уже работающей базы или очередей.
Для этого выпуска нужен согласованный переход Product, Store, Cart, File и обоих
gateway, а не их независимая автораскатка.

1. Подготовить и проверить образы всех шести приложений. Не начинать миграции
   и deploy отдельных приложений до начала общего окна переключения.
2. Остановить входящие изменения через gateway. Дождаться публикации outbox и
   обработки старых событий Product, Store, Cart и File. Проверить ready и
   unacknowledged сообщения, ошибки inbox и незавершённые outbox; при наличии
   недоставленных сообщений сначала разобраться с ними, очереди не очищать.
3. Остановить приложения на время переименования, исключив параллельные CI jobs.
   Сохранить контрольные OID базы/роли и количества товаров, вариантов и событий.
4. Снять управление только с `Database/product-database` при
   `databaseReclaimPolicy: retain`; временно исключить старую роль из managed.roles.
   Не допустить Argo reconcile старых manifest во время перехода.
5. Из базы `postgres` выполнить `ALTER DATABASE product_srv_v2 RENAME TO product_srv`
   и `ALTER ROLE product_srv_v2 RENAME TO product_srv`. Перед этим проверить, что
   целевые имена свободны и активных подключений нет. Права и владельцы сохраняются
   по OID. При MD5-пароле PostgreSQL сбрасывает его при переименовании роли;
   восстановить пароль из существующего Secret, без ротации и записи в логи.
6. Обновить только `username` Secret `product-database`, сохранив пароль.
   Применить новые managed.roles и заново создать Database resource с новым
   `spec.name`/`owner`, сохранив `retain`. Проверить его статус и вход под ролью.
7. Запустить новые отдельные migration jobs Product, Store и Cart. Они меняют
   producer в outbox/inbox, source_producer снимков и ссылку source.producer в
   уведомлениях Cart. UUID, версии, статусы и payload товара не меняются.
   File не хранит этот producer в БД, ему нужна новая проверка DTO, без миграции.
8. Запустить приложения с новой конфигурацией и образами, затем открыть gateway.
   Проверить health, чтение каталога обоими gateway и доставку Product → Store/Cart,
   File → Product. Сверить данные с контрольными значениями и отсутствие старых
   producer-полей в изменённых таблицах.
9. Удалить старые очереди только после подтверждения отсутствия сообщений,
   publishers и consumers. Возобновить обычную автораскатку.

CloudNativePG запрещает изменение `Database.spec.name` существующего объекта:
нельзя заменить значение и ожидать переименования. При `retain` удаление CR
не удаляет физическую базу. [Документация CNPG](https://cloudnative-pg.io/docs/current/declarative_database_management/).
SQL и ограничения переименования: [ALTER DATABASE](https://www.postgresql.org/docs/16/sql-alterdatabase.html),
[ALTER ROLE](https://www.postgresql.org/docs/16/sql-alterrole.html).

## Локальный PostgreSQL

22 ноября 2026 старая локальная `product_srv` со схемой category/product_image
удалена по прямому разрешению пользователя. Рабочая `product_srv_v2` со схемой
product_type/type_product_field переименована в `product_srv`: OID сохранён,
13 товаров и 19 вариантов остались на месте. Локальный `.env` переключён после
переименования. Пользователь локальной БД `admin` не переименовывался.

Старое имя остаётся только во входных условиях миграций и исторических отчётах;
уже применённые миграции не переписываются.

## Проверка dev, 22 ноября 2026

Все шесть CI завершились успешно. Unit/build/publish выполнялись на GitHub,
migrate/deploy — на одном общем `sellgar-dev`. Gateway не имеют migration job.

| Приложение | Source | CI |
|---|---|---|
| Product | `e1c0f5f` | [44952203208](https://github.com/sellgar/sellgar.product.service/actions/runs/44952203208) |
| Store | `c036747` | [44952218138](https://github.com/sellgar/sellgar.store.service/actions/runs/44952218138) |
| Cart | `12045ed` | [44952230789](https://github.com/sellgar/sellgar.cart.service/actions/runs/44952230789) |
| File | `2b8eff7` | [44952244252](https://github.com/sellgar/sellgar.file.service/actions/runs/44952244252) |
| Admin gateway | `78ba1fb` | [44952257075](https://github.com/sellgar/sellgar.admin.gateway/actions/runs/44952257075) |
| Client gateway | `171c279` | [44952270691](https://github.com/sellgar/sellgar.client.gateway/actions/runs/44952270691) |

- Все шесть Deployment имеют по одной ready/updated/available реплике с этими SHA.
- База сохранила OID 16654, роль — OID 16653. Сохранены 11 товаров, 26 вариантов
  и 40 проекций изображений. Существующий личный пользователь `sellgar` читает базу.
- Product outbox: 72 опубликованных события с новым producer. Store: 11 снимков
  и 72 Product inbox-записи с новым producer. Cart: 71 Product inbox-запись с новым
  producer; старых source.producer в Cart outbox нет.
- Реальный RMQ getByUuid и Store catalog.synchronize через `product.commands`
  возвращают существующий iPhone, UUID `20d243ba-4176-49d9-a832-e49405af2c12`, version 38.
- Повторно доставлены обработанные Product и File события с сохранённой идентичностью.
  Очереди после повтора пусты, Cart DLQ пуста. Хеши всех строк product/variant/image
  совпали до и после: повтор не изменил данные.
- Старые очереди удалены в dev и локальном RabbitMQ только с if-empty/if-unused.
  Новые `product.commands` / `product.events` имеют по одному consumer.
- Оба HTTPS gateway health, существующий storefront offer, изображение CDN и
  главная страница витрины возвращают 200; изображение — image/webp.
- Gateway возвращены к одной реплике; общий runner активен. Автоматическая
  синхронизация Argo восстановлена: root, platform и SeaweedFS Synced/Healthy.
