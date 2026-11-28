# Доставка приложений

Рабочая ветка — `development`. Общий контракт `delivery-v0.7.0` ставит юнит-тесты
перед сборкой и проверен полным циклом всех 12 приложений через organization runner.
Ссылки на CI и результаты: [organization-transition.md](organization-transition.md#проверка-delivery-v070).
Работающий образец Product описан в [product-delivery.md](product-delivery.md).
Перед включением CI нового приложения сначала готовится runner и выполняется
передача управления от Argo CD к Helm. Публикация workflow не означает готовность доставки.

## Jobs

С БД: `unit → build → publish → migrate → deploy`.
Без БД: `unit → build → publish → deploy`.
Ошибка юнит-тестов блокирует сборку и все последующие этапы. Компонентной job сейчас нет.
В workflows нет `continue-on-error` или `always()` для обхода этого условия.

| Приложение | Release | Migration Job |
| --- | --- | --- |
| Identity | identity-service | да |
| Shop | shop-service | да |
| Product | product-service | да |
| Store | store-service | да |
| Cart | cart-service | да |
| Order | order-service | да |
| Payment (тестовая интеграция) | payment-service | да |
| File | file-service | да |
| Socket gateway | socket-gateway | да: события и cursors в PostgreSQL |
| Media | media-service | нет |
| Admin gateway | admin-gateway | нет |
| Client gateway | client-gateway | нет |
| Admin UI | admin-ui | нет |
| Storefront | storefront | нет |

Order (`order-service`) и Payment (`payment-service`) добавлены 28 ноября 2026:
полный CI успешен, отдельные миграции применены, обе реплики готовы в dev.
Теперь подтверждены 14 приложений. Доступы/БД и evidence первого rollout:
[checkout-services.md](checkout-services.md).

Build, tests и publication работают на GitHub-hosted runners. Каждый репозиторий
имеет Docker targets `application` и `tests`; только приложения с БД имеют `migration`.
Media использует `node:test` для проверки выдачи изображений и storage events.
Остальные приложения сохраняют существующие команды Jest/Vitest/Node tests;
Tests target устанавливает зависимости и проверяет исходники без сборки приложения.
При необходимости компилируется только зависимость `@sellgar/outbox` с entrypoint в dist.
Проверки PostgreSQL остаются отдельными тестами в исходниках и автоматически не запускаются.
Admin Vitest проверяет исходники приложения, исключая вложенные репозитории и integration-наборы;
у библиотек собственные тестовые контракты. Media использует установленный `ts-node/register` для исходников.

Собранные образы передаются из build в publish через штатные Actions artifacts;
publish проверяет image IDs и сохраняет registry digests. Никаких CI Python-обёрток нет.
Приложения вызывают маленькие общие reusable workflows из `.github/workflows/`:
test-application, build-service, deliver-service,
deploy-application, validate-release и back-merge.

## Runner и доступы

Один organization-level runner обслуживает доставку всех разрешённых приложений
dev. Число репозиториев не меняет число исполнителей. Общий предел — два runner:
dev и, после отдельного ввода production, prod. В dev inventory сейчас одна запись.
PR, сборки и тесты выполняются на GitHub-hosted runners; приватный исполнитель
принимает только доверенные development push и ручной master release.

Регистрация в personal repository не создаёт общий pool. Сначала создаётся
организация и согласованно переносятся репозитории; порядок и зависимости —
[organization-transition.md](organization-transition.md). Не регистрировать runner
на отдельные сервисы и не использовать динамическую перерегистрацию между ними.
Ansible отказывает до изменения узла, если имя организации не задано, runners
больше двух или среда повторяется. Установка использует официальный архив,
SHA-256 и upstream `config.sh`/`svc.sh`; своих установочных скриптов нет.

У общего dev runner один Linux-пользователь без sudo/Docker и одна systemd-служба
с автозапуском. Его доступ в GitHub ограничивается списком репозиториев организации.
Node actions объявляют Node 24 в manifest. Helm 4.3.2 и kubectl 1.37.2
используются через штатные setup actions; версии закреплены. Ansible заранее
загружает kubectl в стандартный cache с SHA-256, чтобы CI не зависел от первой загрузки.

Используется существующий namespace-scoped deployment principal
`product-delivery-dev`. Историческое имя не ограничивает его Product:
он доверен для всего `sellgar-dev`. В применённом RBAC добавлены Ingress/Middleware для gateway/UI,
но не даёт доступа к `sellgar-prod` или cluster administration.
Канонический namespace kubeconfig хранится в `/etc/sellgar/delivery/dev.kubeconfig`
(root, 0600) и копируется Ansible внутри VM в home общего исполнителя; административный K3s config
не выдаётся runner. Срок сертификата и обновление: [runner README](../bootstrap/runner/README.md).

Ansible подготовка:

```bash
cd ansible
.venv/bin/ansible-playbook -i inventory/dev.yml runners.yml --syntax-check
.venv/bin/ansible-playbook -i inventory/dev.yml runners.yml --tags prepare
```

Регистрация требует short-lived токен организации из Organization Settings →
Actions → Runners. Передавать его через игнорируемый Vault file с переменной
`runner_registration_token`. `delivery_organization: sellgar` задана после создания
организации владельцем. Пустое имя организации блокирует установку.
Токены не хранятся в inventory, command arguments, логах или Git.
Задача регистрации использует `no_log` и `ACTIONS_RUNNER_INPUT_TOKEN`.

```bash
.venv/bin/ansible-playbook -i inventory/dev.yml runners.yml --tags register \
  --ask-vault-pass -e @../.local/runner-registration.yml
```

## Передача управления

Для каждого release отдельно применяется процедура Product из
`bootstrap/runner/README.md`: зафиксировать readiness/образ/конфигурацию,
убрать Application из inventory, остановить reconciliation, удалить только
Argo Application без каскада. Затем принять существующие ресурсы в Helm с тем же
running digest, проверить spec и убрать только retired Argo managedFields,
если равные поля остались в совместном владении. Эти одноразовые действия
не входят в routine CI. Deployment, Service, credentials, базы и объекты не удаляются.

Далее обычный CI создаёт migration Job (только профиль с БД), ждёт её результата,
показывает логи и выполняет Helm upgrade с ожиданием готовности. Проверка после
deploy сравнивает digest и вызывает HTTP path существующей readiness probe:
`/health` у backend, `/config.js` у админки, `/healthz` у витрины.
Старый pending Helm release или действующая migration Job блокируют новый запуск.
Новый commit в ветке блокирует устаревшую раскатку.

Каждая раскатка создаёт отдельный immutable ConfigMap с runtime-файлом:
`.env` для backend/Next, `config.js` для Admin UI. Данные берутся из values того же
source SHA, секреты — из Secret refs. `environmentFile.filename/mountPath` сохраняют
реальное расположение файла. Старые ConfigMap сохраняются для истории Helm/старых pod.

## Production

Production не разворачивается в этой задаче. Необходимо подготовить отдельные values,
Secrets, runner `sellgar-prod` и защищённое GitHub environment prod.
Для backend prod values ещё пусты: render/deploy не пройдёт до их заполнения;
CI development проверяет dev values и не подставляет выдуманные production доступы.
Release candidate — immutable release с metadata и digest; продвигается без rebuild
после проверки успешного CI и совпадения дерева с master. Миграционный reverse
не реализуется. Автоматического DB rollback нет.

## Приёмка перехода

Для каждого приложения нужны: доступ к общему online runner, Helm ownership без Argo reconciliation,
терминальный успешный GitHub run, готовая реплика с новым digest и HTTP readiness.
Для gateway/UI дополнительно проверяются прежние домены/маршруты; для Socket —
соединение Socket.IO; для Media — чтение сохранённого изображения с CDN.
До выполнения этих проверок переход не считается законченным.
