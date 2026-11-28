# Sellgar Infrastructure

Репозиторий хранит подготовку узлов, bootstrap Kubernetes и желаемое состояние
платформы. Рабочая ветка — `development`, выпуск — `master`.
Dev работает в локальной K3s VM; prod на VPS пока не развёрнут.

## Владельцы конфигурации

```text
ansible/                         upstream K3s collection и bootstrap playbooks
bootstrap/                       закреплённые manifests операторов и Argo CD
charts/application/             общий chart приложений
environments/dev/
  platform/                      PostgreSQL, RabbitMQ, сертификаты, S3 identity refs
  storage/seaweedfs/              официальный SeaweedFS chart и dev values
  services/<service>/             Argo CD Applications
environments/prod/               подготовка будущей prod-среды
docs/                            эксплуатация и результаты переходов
```

Приложение хранит `helm/values.app.yaml`, `helm/dev/values.yaml` и
`helm/prod/values.yaml` в своём репозитории. ConfigMap принадлежит приложению;
пароли поступают через Kubernetes Secret refs. В инфраструктурном Git нет
копий values приложений, kubeconfig, Docker config или plaintext credentials.

## Подготовка инфраструктуры

Используется [Ansible](ansible/README.md): готовая коллекция `k3s.orchestration`
и модули `kubernetes.core`. Подготовка сервера отделена от установки операторов
и GitOps. Повторный bootstrap сохраняет существующие пароли.

CloudNativePG управляет отдельными базами и ролями сервисов; RabbitMQ Cluster
Operator управляет брокером и его первоначальной учётной записью. Общие компоненты
среды находятся в `platform/`. Они используют постоянные local-path PVC;
`dev-platform` не удаляет их автоматически при изменении inventory.

Dev-хранилище — [SeaweedFS](docs/seaweedfs.md) из официального Helm chart.
Оно доступно только внутри Kubernetes. Media имеет доступ к бакету `sellgar`;
браузер получает изображения через публичный read endpoint Media.
Историческое имя административного Secret `minio-root` сохранено при переходе,
чтобы не ротировать credentials. Это не означает использование MinIO.

Бизнес-сиды и воспроизведение проекций не входят в bootstrap и будут разобраны
отдельно. `bootstrap/dev/initialize-store.py` — отложенный legacy-инструмент;
Ansible и CI его не вызывают.

## Доставка приложений

Все 14 dev-приложений используют [общий CI-контракт](docs/application-delivery.md):
unit → build → publish → migrate → deploy. Для приложений без БД migrate отсутствует.
Юнит-тесты обязательны до сборки; компонентная job пока не включена.
Unit/build/publish работают на GitHub, migrate/deploy — на одном общем runner `sellgar-dev`.
Приложениями управляет Helm CI, платформой — Argo CD. Переход и доказательства:
[organization-transition.md](docs/organization-transition.md).
Коммит в infrastructure на каждый выпуск приложения не нужен.
Prod promotion выполняется вручную с проверенным digest, без пересборки.

Order/Payment развёрнуты с отдельными БД/ролями и проверены в dev; Payment использует
тестовый адаптер. Порядок включения, CI и runtime evidence —
[checkout-services.md](docs/checkout-services.md).

## Доступ

- Админка: `https://admin.dev.sellgar.ru`.
- Витрина: `https://dev.sellgar.ru`.
- Изображения: `https://cdn.dev.sellgar.ru/images/<fileUuid>`.
- Будущий production CDN: `https://cdn.sellgar.ru`; prod не подключается автоматически.

HTTPS: [cert-manager / Let's Encrypt](docs/dev-https.md).
Постоянные туннели PostgreSQL, RabbitMQ UI, Admin и Client gateway описаны
в [bootstrap/README.md](bootstrap/README.md#постоянные-туннели-dev-среды).
Туннели независимы от watcher и агентской сессии.

HTTP приложений и их внутренних Services — 8080. PostgreSQL, RabbitMQ и S3
используют штатные протокольные порты; SeaweedFS S3 — 8333.
Базы, брокер, S3 и административные консоли не публикуются через Ingress.

Операционные документы: [Identity](docs/identity-service.md),
[Shop](docs/shop-service.md), [Product](docs/product-service.md),
[Store](docs/store-service.md), [Cart/File/Media](docs/commerce-services.md),
[Admin UI](docs/admin-ui.md), [Storefront](docs/storefront.md).
