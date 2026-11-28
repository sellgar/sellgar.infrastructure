# Подготовка Order и Payment к dev-доставке

2026-11-28: первый dev rollout завершён и проверен. Order и Payment прошли полный
CI, отдельные миграции и проверку runtime. В dev теперь 14 приложений под Helm CI;
Payment использует только тестовый провайдер. Результаты приведены ниже.

| Сервис | Helm release | База / роль | Secret | Images |
| --- | --- | --- | --- | --- |
| Order | order-service | order_srv | order-database | ghcr.io/sellgar/sellgar.order.service |
| Payment | payment-service | payment_srv | payment-database | ghcr.io/sellgar/sellgar.payment.service |

Оба используют общий chart и reusable workflows `delivery-v0.7.0`:
unit → build → publish → migrate → deploy. Application values и ConfigMap
хранятся в соответствующем сервисе. Infrastructure добавляет только ресурсы
CNPG и генерацию credentials. Приложениями управляет Helm CI; новых Argo
Applications, runners, публичных ingress или банковских credentials нет.
Payment на этом этапе — явная тестовая интеграция.

## Перед первым push с CI

1. Через `ansible/secrets.yml` создать только отсутствующие `order-database`
   и `payment-database`. Playbook сохраняет существующие значения, не ротирует
   credentials. Сначала syntax/check, затем разрешённое применение.
2. Опубликовать platform-конфигурацию: управляемые login-роли CNPG без superuser
   и две Database с `databaseReclaimPolicy: retain`. Argo CD управляет платформой.
   Дождаться, что роли и базы созданы; Kubernetes Secret должен существовать
   до reconciliation роли. Миграции приложений ещё не запускать.
3. В GitHub добавить только два новых репозитория к разрешённым репозиториям
   существующей runner group с online `sellgar-dev`. Сохранить текущий список,
   одобрение внешних PR workflows и защиту development/master. Runner не
   регистрировать повторно; workflow-доставка идёт только по trusted push/manual
   release. Политика Selected repositories описана в [GitHub Docs](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/manage-access).
4. Проверить доступ Actions к reusable workflows infrastructure и recursive
   submodules. Если требуется, выдать существующий `SUBMODULES_READ_TOKEN` через
   текущий механизм Secrets, без копирования значения в Git/логи. Проверить
   environment dev и права публикации GHCR / чтения через `ghcr-pull`.
5. Сначала доставить совместимые Store и Cart: резервирование, checkout preview/get,
   поддержка bigint envelope completion. Только после этого публиковать новые
   сервисы с CI. Payment можно поднять перед Order; без принятых заказов Order
   не требует доступного Payment для своего readiness.
6. Дождаться успешных unit/build/publish/migrate/deploy, проверить миграции,
   digest обеих реплик, их `/health` и доставку событий. Наличие workflows или
   успешная локальная сборка не означает, что dev уже развёрнут.

Команды Ansible из `ansible/`:

```bash
.venv/bin/ansible-playbook -i inventory/dev.yml secrets.yml --syntax-check
.venv/bin/ansible-playbook -i inventory/dev.yml secrets.yml --check
```

Применение playbook/платформы и push сервисов — отдельная стадия после подготовки.
Prod остаётся незаполненным, автоматической подстановки dev доступов нет.

## Первый dev rollout, 2026-11-28

- Ansible check/apply создал только два отсутствующих DB Secrets: changed=2,
  failed=0. Существующие credentials не ротировались. CNPG согласовал обе роли
  без superuser и базы; `dev-platform` после публикации — Synced/Healthy.
- С подтверждения владельца к существующей runner group добавлены только два
  новых репозитория: 14 selected repositories, один `sellgar-dev`. Проверено
  обязательное одобрение workflows всех внешних contributors. В обоих новых
  репозиториях сохранена защита development/master: PR, без force push и удаления;
  owner bypass соответствует текущему процессу публикации. Unit job блокирует build.
- Сначала опубликованы и доставлены зависимости:
  [Store CI 46165290057](https://github.com/sellgar/sellgar.store.service/actions/runs/46165290057),
  source `88e04eee09d9be4455b59dea30ce240f06a13413`;
  [Cart CI 46165291246](https://github.com/sellgar/sellgar.cart.service/actions/runs/46165291246),
  source `af87da51ecb7a7f11e7532a573a4b979f4359ee7`. Миграции завершены, реплики 1/1.
- [Order CI 46165931713](https://github.com/sellgar/sellgar.order.service/actions/runs/46165931713)
  завершился success с первой попытки. Source
  `fde2481357e0d3fb5d0348815073b76cc7b9b689`, application digest
  `sha256:e2da06f94b715f45d0dfb815a006f22225bea09c5d0834f9cf5fc19533974f44`.
- [Payment CI 46165813098](https://github.com/sellgar/sellgar.payment.service/actions/runs/46165813098)
  завершился success на attempt 2. Первая публикация получила GHCR `denied` после
  загрузки слоёв. Проверены правильный source repository, inherited access и Admin
  Actions access своего репозитория к package. Повторены только failed jobs, без
  изменения source, прав или пересборки успешных этапов. Повтор публикации успешен;
  причина первоначального отказа registry не установлена. Source
  `44f71ac4cdcf03853fe93323c87093a6f431185a`, application digest
  `sha256:18bef9fd2b204c577c3676fac11cde410d22cadebde9d85f4e45ac9ff0a9678c`.
- Оба Deployment 1/1, реальные source annotations и image digest совпадают с CI.
  `/health` обоих сервисов возвращает 200. Из runtime проверены собственные
  database/current_user (`order_srv`, `payment_srv`) и записи миграций
  `CreateOrders1795850000000`, `CreatePayments1795851000000`.
- Через действующий RabbitMQ выполнены read-only `order.get` и `payment.get`
  случайных UUID: оба вернули ожидаемый 404, без создания заказов/платежей/резервов.
  Проверены bindings `payment.succeeded` и `payment.failed` в Order, а также
  `order.checkout.completed` в Cart через `event.exchange`.

Публичного checkout API и экранов ещё нет. Банк, реальные оплаты, письма и prod
не подключались; успешный dev rollout не означает готовность всего оформления.

## Проверки до rollout

`actionlint` проверяет workflows обоих приложений. Helm lint/render используют
тот же chart, что закреплён в `delivery-v0.7.0`. Kustomize собирает platform
без обращения к кластеру; Ansible проходит syntax-check.

В workspace сохранён `agent/scripts/smoke-checkout-images.cjs`: миграции дважды
с отдельными non-superuser DB-ролями, запуск production images с non-root/read-only
и secret env, HTTP readiness, потеря readiness при отключении RabbitMQ. Только
приватная временная Docker-сеть и новые volumes, с очисткой после проверки.
Это локальная ручная проверка deployment artifacts, не новая CI component job.

Проверено 2026-11-28: 45 unit в Docker test targets, все четыре deploy images,
actionlint 1.7.12, Helm 4.3.2 strict lint/render, platform kustomize и Ansible
syntax-check. Render сохраняет Secret refs, security context, порт/readiness и
отсутствие Ingress; роли БД без superuser, базы retain. Image smoke успешно
применил миграции дважды, запустил оба production images и подтвердил readiness
200 → 503 после отключения брокера. Временные containers/volumes/network удалены.
