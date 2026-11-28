# Подготовка Order и Payment к dev-доставке

2026-11-28: подготовлены локальные артефакты. Применение платформы, GitHub settings,
публикация и rollout ещё не выполнены. Эти сервисы не входят в подтверждённые
12 работающих dev-приложений до прохождения первого полного CI.

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
