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

На ноутбуке `product_srv` занят прежней схемой с category/product_image,
а рабочая схема product_type/type_product_field находится в `product_srv_v2`.
Сначала сохранить legacy-базу под отдельным архивным именем, затем переименовать
рабочую в `product_srv`. Не подменять текущую схему legacy-базой и не удалять данные.
Пользователь локальной БД `admin` не переименовывается. Локальный `.env` переключать
после переименования, отдельно от `.env` сохранённого `code/_service`.

Старое имя остаётся только во входных условиях миграций и исторических отчётах;
уже применённые миграции не переписываются.
