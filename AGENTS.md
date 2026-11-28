# Sellgar Infrastructure

Начинай с `README.md`. Рабочая ветка — `development`, ветка выпуска — `master`; отдельные ветки без запроса
пользователя не создавать.

- Репозиторий владеет конфигурацией окружений, подготовкой узлов и GitOps bootstrap.
- Общий переиспользуемый chart находится в `charts/application/`.
- Приложения используют `.github/workflows/test-application.yml` для юнит-тестов до сборки,
  `.github/workflows/build-service.yml` для сборки/публикации
  и `.github/workflows/deliver-service.yml` для прямой Helm-доставки.
  Этапы migrate/deploy видны в GitHub; Argo не должен управлять теми же ресурсами.
  JavaScript actions должны объявлять Node 24 или новее в собственном manifest;
  принудительный запуск устаревшего action на новом runtime не заменяет обновление.
  Переход работающей среды выполняется отдельно, с сохранением ресурсов:
  `docs/product-delivery.md`. Все 14 dev-приложений работают под Helm CI;
  Argo CD управляет платформой. Компонентные jobs пока не включены.
  Первый rollout Order/Payment: `docs/checkout-services.md`; Payment пока test.
  Репозиторий приложения хранит `helm/values.app.yaml` и
  `helm/<dev|prod>/values.yaml`, включая содержимое ConfigMap своей среды.
  Не копировать эти values в инфраструктурный репозиторий.
- Общие компоненты среды находятся в `environments/<environment>/platform/`.
- Не хранить credentials, kubeconfig, Docker config, данные БД и бакетов в Git.
  Ссылки на Secrets допустимы; значения — только после настройки шифрования.
- Образы закреплять через `repository:tag@sha256:digest`. При продвижении в prod
  сохранять digest, проверенный в dev.
- CI приложения сохраняет digest и source SHA в release metadata и передаёт их
  Helm при доставке. Не коммитить эти значения в infrastructure на каждый выпуск.
  Старый Helm OCI packager удалён; контракт: `docs/application-delivery.md`.
- Изменение inventory не означает развёртывание. Установку, миграцию данных
  и удаление ресурсов выполнять только в разрешённом scope.
- Проверять YAML, доступность закреплённых образов и затронутый runtime-сценарий.
  Не останавливать приложения и не подключать их volumes к проверочным контейнерам.

- Внутренний HTTP-порт приложений и Kubernetes Services — 8080. Ingress снаружи
  обслуживает стандартные HTTP/HTTPS порты. PostgreSQL, RabbitMQ и MinIO
  сохраняют штатные протокольные порты.
