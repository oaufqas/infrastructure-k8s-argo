<div align="center">

#### Проект представляет собой реализацию современного производственного (Production-ready) цикла для развертывания fullstack приложения в кластере kubernetes, управления инфраструктурой и доставки обновлений в нескольких инфраструктурных решениях: облаке Yandex Cloud, на виртуальнах машинах proxmox. Проект полностью автоматизирован с помощью ansible ролей.

  <div>
    <img src="https://img.shields.io/badge/Architecture-K8s-blue" height="20"/>
    <img width="12" />
    <img src="https://img.shields.io/badge/GitOps-ArgoCD-orange" height="20"/>
    <img width="12" />
    <img src="https://img.shields.io/badge/IaC-Terraform-purple" height="20"/>
    <img width="12" />
    <img src="https://img.shields.io/badge/Monitoring-VictoriaMetrics-brightgreen" height="20"/>
    <img width="12" />
  </div>
</div>

# Оглавление

- Архитектура проекта

- Технологический стек

- Компоненты и их взаимодействие

- CI/CD Pipeline

- Инфраструктура безопасности

- Мониторинг и Observability

- Структура репозиториев

- Быстрый старт

# Архитектура проекта

1. Infrastructure as Code (IaC)Terraform: Используется для управления жизненным циклом облачных ресурсов (Provisioning).
Создание сети (VPC, Subnets) и вычислительных ресурсов (Managed Service for Kubernetes).
Управление безопасностью через IAM (Service Accounts, Roles).
Использование Yandex KMS для криптографических операций (Auto-unseal Vault).
Terraform Helm Provider: Используется исключительно для инициализации "фундамента" — установки ArgoCD.

2. GitOps & Continuous Delivery (CD)
ArgoCD: Центральный контроллер доставки. Реализует паттерн "App of Apps".
Синхронизация состояния кластера с Git-репозиторием (Single Source of Truth).
Автоматический мониторинг дрейфа конфигурации (Self-healing).
Helm: Пакетный менеджер. Все компоненты (Ingress, Vault, App...) упакованы в чарты для управления зависимостями и параметризации через values.yaml.

3. Secret Management & Security
HashiCorp Vault (Raft HA): Хранилище секретов в режиме высокой доступности с интегрированным хранилищем Raft.
Интеграция с Yandex KMS для автоматического разпечатывания (Auto-unseal).
External Secrets Operator (ESO): Автоматизирует синхронизацию секретов из Vault в нативные Kubernetes Secrets.
Метод аутентификации: Kubernetes Auth Method (через ServiceAccounts).
Cert-Manager: Автоматизация выпуска TLS-сертификатов от Let's Encrypt через HTTP-01/DNS-01 челленджи.

4. Infrastructure Services
Ingress Nginx Controller: Точка входа трафика в кластер с поддержкой балансировки и терминации SSL.
NFS Server Provisioner: Обеспечение общего хранилища (Shared Storage) для распределенных компонентов приложения (ReadWriteMany).

5. Continuous Integration (CI)
GitLab CI:
Build Stage: Сборка Docker-образов с использованием Docker-in-Docker(DinD).
Registry: Хранение артефактов в Docker Container Registry.
Deploy Stage: Автоматическое обновление тегов образов в GitOps-репозитории.

6. Observability (Full Monitoring Stack)
VictoriaMetrics & Grafana: Сбор метрик с инфраструктуры и приложения, визуализация состояния через дашборды.
VictoriaLogs & Vector: Централизованный сбор и агрегация логов со всех подов кластера.

### Общая схема взаимодействия

![alt text](./images/image1.png)

### Детальная схема компонентов

![alt text](./images/image2.png)

## Технологический стек

### Infrastructure & Orchestration

|Технология | Версия | Назначение
|---|---|---
|Kubernetes | 1.28 | Оркестрация контейнеров (Yandex Managed)
|Terraform | 1.6+ | Infrastructure as Code
|Helm | 3.13+ | Package manager для K8s
|ArgoCD | 2.10+ | GitOps непрерывная доставка
|GitLab CI | Latest | CI/CD пайплайны


### Security & Secrets

|Технология | Версия | Назначение
|---|---|---
|HashiCorp Vault | 1.15+ | Хранение и управление секретами
|External Secrets Operator | 0.9+ | Синхронизация Vault → K8s Secrets
|Cert-manager | 1.13+ | Автоматические SSL сертификаты
|Yandex KMS | - | Auto Unseal для Vault

### Observability

|Технология | Версия | Назначение
|---|---|---
|VictoriaMetrics | Latest | Сбор и хранение метрик
|VictoriaLogs | Latest | Сбор и хранение логов
|Grafana | 10+ | Визуализация метрик и логов
|Vector | Latest | Агент сбора логов

### Networking & Storage

|Технология | Версия | Назначение
|---|---|---
|Ingress NGINX | 1.9+ | Балансировка и SSL терминация
|NFS Server Provisioner | Latest | Общее хранилище RWX
|CoreDNS | 1.11+ | DNS внутри кластера

### Application Stack

|Компонент | Технологии
|---|---
|Backend | Node.js, Express, Sequelize, JWT, 2FA
|Frontend | React, Vite, CSS, MobX, Axios
|Database | MySQL 8.0 (StatefulSet + PVC)

## Компоненты и их взаимодействие

### CI/CD Pipeline Flow

![alt text](./images/image3.png)

### Security Flow (Vault + ESO)

![alt text](./images/image4.png)

### Observability Flow

![alt text](./images/image5.png)

## CI/CD Pipeline

```yaml
stages:
  - build
  - deploy

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD
    - echo "VITE_API_URL=${VITE_API_URL}" > ./client/.env
  script:
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker push $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
  only:
    - master

update_manifests:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache git yq
  script:
    - git config --global user.email "gitlab@runner.com"
    - git config --global user.name "CI runner"

    - git clone https://oauth2:${GITHUB_TOKEN}@${GITHUB_REPO} infra
    - cd infra/values/prod

    - yq -i '.server.container.tag = "'$CI_COMMIT_SHORT_SHA'"' app-values.yaml
    - git diff --quiet && exit 0

    - git add app-values.yaml
    - git commit -m "CD update app image tag to $CI_COMMIT_SHORT_SHA"
    - git push origin master
```

### Pipeline Steps:

Build — сборка Docker образа с многоступенчатой оптимизацией и публикация образа в Docker Container Registry.

Deploy — обновление values.yaml путем коммита в инфраструктурном репозитории (GitHub) с новым тегом.


## Инфраструктура безопасности: Vault + YC KMS + External Secrets Operator

```yaml
# Взаимодействие компонентов
Vault (Auto Unseal via Yandex KMS)
  ├── secrets engine: kv-v2 (path: secret/)
  ├── auth method: kubernetes
  ├── policy: app-policy (read secret/data/*)
  └── role: eso-role (bound SA: vault-sa)

External Secrets Operator
  ├── reads secrets from Vault
  ├── creates K8s Secret
  └── sync period: 1 hour

Application
  └── envFrom: secretRef (K8s Secret)
```

### Подробная схема: Как работает связка Vault и Yandex KMS.

В стандартном режиме работы Vault шифрует все ваши секреты одним главным ключом — Master Key.

1. При обычной инициализации Master Key разрезается на 5 частей (Unseal Keys), которые раздаются инженерам, а сам Master Key стирается из памяти.

2. При использовании Auto-Unseal применяется технология Envelope Encryption (конвертное шифрование):
    - Vault генерирует Master Key.
    - Он отправляет этот Master Key в Yandex KMS.
    - Yandex KMS шифрует его своим внутренним аппаратным ключом (который никто никогда не увидит) и возвращает Vault зашифрованный «конверт» (строку).
    - Vault сохраняет этот зашифрованный конверт у себя на диске (в папке `/vault/data`).

3. Процесс авто-распечатывания при старте: Под Vault просыпается ➔ читает зашифрованный конверт с диска ➔ отправляет его в API Yandex KMS ➔ KMS расшифровывает конверт и возвращает чистый Master Key в оперативную память Vault ➔ Vault готов к работе.

### SSL/TLS (cert-manager)

```yaml
Issuer: ClusterIssuer (Let's Encrypt)
  ├── server: https://acme-v02.api.letsencrypt.org/directory
  ├── solver: http01 (ingress)
  └── email: admin@email.com

Certificate
  ├── secretName: secret-tls
  ├── dnsNames: [any.domain.com]
  └── issuerRef: letsencrypt-issuer
```

Процесс получения SSL-сертификатов в Kubernetes с помощью cert-manager полностью автоматизирован:
1. В кластере больше не нужно вручную генерировать ключи, cкачивать файлы или следить за датой окончания действия.
2. (Let's Encrypt выдает сертификаты на 90 дней, а cert-manager сам обновляет их за 30 дней до конца)

- Архитектурная схема ACME (Automated Certificate Management Environment):

```text
   [Кластер K8s]                                       [Let's Encrypt API]
       |                                                         |
       |cert-manager  --- (1. Просит сертификат для домена) ---> |
       |                                                         |
       |<-- (2. Возвращает секретный токен проверки/Challenge) --|
       |                                                         |
      (3. Создает временный под и путь /.well-known/acme-challenge/)
       |                                                         |
       |<-- (4. Робот Let's Encrypt заходит по HTTP на сайт) --- |
       |                                                         |
       |--- (5. Проверка прошла -> Выпускает SSL сертификат) --> |

```

## Мониторинг и Observability

Метрики (VictoriaMetrics)

```yaml
k8s-metrics:
  - node_exporter (CPU, RAM, Disk, Network)
  - kube-state-metrics (pods, deployments, services)
  - kubelet (cadvisor; container metrics)
  - api-server, controller-manager, scheduler
  - core-dns, etcd

app-metrics:
  - http_requests_total (rate, errors, latency)
  - db_connection_pool (active, idle, wait)
  - business_metrics (users, orders, etc.)
```

Логи (VictoriaLogs)

```yaml
logs:
  - vector (Собирает логи со всех подов, серверов)
  - victoria-logs-single-server
```

### Grafana Dashboards

|Dashboard | Источник | Описание
|---|---|---
|K8s Cluster Monitoring | VictoriaMetrics | Состояние кластера, ресурсы нод
|Pods Dashboard | VictoriaMetrics | CPU/Memory per pod, restarts
|App Metrics | VictoriaMetrics | RPS, ошибки, latency
|Logs Explorer | VictoriaLogs | Поиск и фильтрация логов

# Структура репозиториев

### Application Repository (GitLab)

```text
app/
├── client/                 # React + Vite фронтенд
│   ├── src/
│   ├── public/
│   └── package.json
├── server/                 # Node.js + Express бэкенд
│   ├── config/
│   │   ├── database.js
│   │   └── models.js
│   ├── controllers/
│   ├── dtos/
│   ├── middleware/
│   ├── routes/
│   ├── services/
│   ├── utils/
│   ├── index.js   # main file
│   └── package.json
├── uploads/                # Директория загруженных на сервер файлов
├── kubernetes              # Helm chart приложения
├── nginx/                  # Конфиги nginx
├── node_modules/
├── ansible_deploy_rent    # Вариант с деплоем через ansible & docker compose
├── ssl                    # Docker compose для сертификатов
├── .gitlab-ci.yml         # CI/CD пайплайн
├── .dockeringnore
├── .gitignore
├── docker-compose.yaml    # Docker compose файл
├── Dockerfile             # Dockerfile для сборки образов
└── README.md
```

### Infrastructure Repository (GitHub)

```text
argocd-infrastructure-repo/
├── terraform/              # IaC для Yandex Cloud
│   ├── main.tf            # кластер, сети, диски, KMS
│   └── variables.tf
├── apps/                 # GitOps манифесты applications
│   ├── cert-manager.yaml
│   ├── victoria-logs.yaml
│   ├── victoria-metrics.yaml
│   ├── ingress-nginx.yaml
│   ├── nfs-server.yaml
│   ├── vault.yaml
│   ├── external-secrets.yaml
│   └── app.yaml
├── charts/                 # Helm чарты
│   ├── vault/values.prod.yaml
│   ├── victoria-logs-single/
│   ├── victoria-metrics-k8s-stack/
│   ├── argo-cd/
│   ├── cert-manager/
│   ├── external-secrets/
│   ├── ingress-nginx/
│   ├── nfs-server-provisioner/
│   └── app-chart/
├── values/                 # Переменные для чартов
│   ├── prod/
│   │   ├── nfs-values.yaml
│   │   ├── vault-values.yaml
│   │   ├── victoria-logs-values.yaml
│   │   ├── victoria-metrics-values.yaml
│   │   └── app-values.yaml
│   └── local/
│       ├── nfs-values.yaml
│       ├── vault-values.yaml
│       ├── victoria-logs-values.yaml
│       ├── victoria-metrics-values.yaml
│       └── app-values.yaml
├── .gitlab-ci.yml          # CI-CD gitlab
├── .gitignore         
├── images/                 # Графики для readme
├── README.md
└── root-app.yaml           # Точка входа (App of Apps)
```

## Быстрый старт

Prerequisites

- Yandex Cloud аккаунт
- GitLab проект с кодом приложения
- GitHub репозиторий для инфраструктуры
- Установленные: terraform, helm, kubectl

Step 1: Установка yc cli и получение токена для terraform

```bash
curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash

yc init --username=<email_addr>

yc iam create-token # --> ./terraform/terraform.tfvars, также нужно указать в переменных cloud_id и folder_id из яндекса.
```


Step 2: Развернуть инфраструктуру

```bash
git clone https://github.com/oaufqas/argocd-infrastructure-repo

cd ./terraform

terraform init

terraform apply -auto-approve
```


Step 3: Создать в YC KMS синхронный ключ и положить его ID в values valut по пути `./values/prod/vault-values.yaml` строка `kms_key_id = "KEY_ID"` для автоматического unseal. Указать домен в `./values/prod/app-values.yaml` либо выключить параметр ssl.


Step 4: Применить главный файл argo (App of Apps)

```bash
kubectl apply -f argocd/root-app.yaml

# Настроить A-DNS запись домена в интерфейсе провайдера на IP Ingress контроллера кластера, после того, как ip адрес появится, для прохождения HTTP челленджа (TLS).
```


Step 5: Один раз настроить vault и заполнить секретами (unseal автоматический):

```bash
kubectl exec -it vault-0 -n vault -- sh

vault operator init

vault login

vault secrets enable -path=secret kv-v2

vault auth enable kubernetes

vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://${KUBERNETES_PORT_443_TCP_ADDR}:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

vault policy write app-policy - <<EOF
path "secret/data/*" {
  capabilities = ["read"]
}
EOF

vault write auth/kubernetes/role/eso-role \
    bound_service_account_names=vault-sa \
    bound_service_account_namespaces=default \
    policies=app-policy \
    ttl=1h

# Наполнение секретами
vault put secret/app...
```

Step 6: Настроить GitLab CI для полноценного CI-CD

```bash
# Создать репозиторий для приложения в gitlab, скопировать с https://github.com/oaufqas/rent исходный код.

# Добавить переменные в GitLab CI/CD Settings:
- CI_REGISTRY_USER
- CI_REGISTRY_PASSWORD
- GITHUB_TOKEN
- GITHUB_REPO
- IMAGE_NAME
- VITE_API_URL=/api
```