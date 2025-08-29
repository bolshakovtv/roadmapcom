# Дорожная карта (4 месяца, 8 спринтов)

**Цель:** за 4 месяца «под ключ» развернуть инфраструктуру, CI/CD, безопасность, мониторинг/логирование и подготовить прод-выкатку. Далее — этап оптимизации и автоматизации.

**Инструменты и базовые решения:**

* IaC: **Terraform** (VPC/сети/ВМ/балансировщики/PV), **Ansible** (базовый образ ОС, firewall, users, hardening, post-provisioning).
* Репозитории: **GitLab (self-hosted)** для кода и CI/CD, **Nexus** для контейнерных образов/Helm/артефактов; **MinIO** (или S3) для ML-моделей Vosk (паблик-скачивание с presigned URL + опционально CDN).
* Оркестрация: **Kubernetes для всех сред**, GitOps — **Argo CD**. CNI — **Cilium** (NetworkPolicy/L7 obs), Ingress — **NGINX** (HTTP/2 + gRPC), **cert-manager** (ACME + внутренний CA).
* gRPC/API Gateway: **Envoy или NGINX** (шардинг маршрутов на несколько gateway-инстансов).
* Мониторинг: **Prometheus + Alertmanager + Grafana**; логи **Loki + Promtail**; трассировка **OpenTelemetry Collector**.
* Безопасность: **WireGuard VPN** для разработчиков, RBAC, NetworkPolicies, SOPS/Sealed Secrets или Vault+External Secrets; централизованный **бан-лист** из PostgreSQL (k8s CronJob + обновление ipset/ingress-банлистов + Ansible для вне-k8s).
* Почта: **Mailu** или **Mailcow** с DKIM/SPF/DMARC.

**Среды и кластера:**

* **dev+stage** — один кластер (namespaces/quotas/isolated runners). **prod** — отдельный кластер. (Вариант B: один кластер с строгой изоляцией, если ограничены по бюджету.)

## Спринт 1 — Фундамент IaC и репозитории (Terraform/Ansible, GitLab, Nexus, каркас k8s)

**Цели:** запустить базовую инфраструктуру dev/stage, репозитории и раннеры.

* Terraform: VPC/подсети, NAT, брандмауэр, LB, dev/stage k8s (managed/cluster-api/kubespray), PV-класс (NVMe), DNS зоны.
* Ansible: базовый образ ОС (CIS-hardening light), users/SSH, firewalld/nftables baseline, journald/rsyslog, node-exporter, crond.
* GitLab: группа/проекты, **раздельная видимость** (developers не видят чужие проекты), protected branches/tags, артефакты.
* GitLab Runners в k8s (separate executor, изоляция по projects/tag).
* **Nexus**: hosted docker/helm/raw; роли/LDAP(опц.), ретенции, proxy к Docker Hub.
* БД dev: PostgreSQL single + Redis (1 экземпляр), PV + бэкап-крон (pg\_dump + рестор-тест).
* Док: конвенции Dockerfile (multi-stage, non-root, distroless), .env → values.yaml/Secrets.

**определение готовности:** GitLab доступен, Runners собирают образ и пушат в Nexus; dev k8s живой; один сервис деплоится вручную helm install; базовый бэкап БД работает.

---

## Спринт 2 — K8s-надстройки, GitOps, ingress для gRPC, витрина ML

**Цели:** полнофункциональная платформа dev/stage.

* Cilium + NetworkPolicy baseline; PodSecurity/PodDisruptionBudgets; liveness/readiness стартовый шаблон в Helm chart.
* Ingress NGINX c HTTP/2 и **gRPC**, **cert-manager** (Let’s Encrypt staging + внутренний CA), **external-dns**.
* **Argo CD** bootstrap: app-of-apps; окружения dev/stage; policy RBAC (читатели/деплойеры).
* Helm-чарты каркаса для микросервисов (общий chart lib/copybara).
* **MinIO** для ML Vosk файлов + presigned URLs; CI-джоб публикации моделей; (альтернатива — Nexus raw repo).
* Секреты: SOPS + age (или Sealed Secrets) для values.

**определение готовности:** PR → build → образ в Nexus → Argo CD авто-деплой в dev; gRPC сервис доступен через Ingress с TLS; ML модель скачивается по presigned URL.

---

## Спринт 3 — CI/CD пайплайны и среда разработчика

**Цели:** автоматическая сборка/тест/деплой, изоляция разработчиков.

* GitLab CI: шаблоны `.gitlab-ci.yml` (build/test/lint/scan/push), кэш, SBOM (syft), Trivy scan, семантические версии.
* Разделение проектов: группы/сабгруппы, роли Maintainer/Developer, видимость **private** по умолчанию, доступ к общим shared libs по токенам.
* Эмуляторы: headless **Android Emulator** image в CI (Linux, KVM), smoke-тесты; опция внешнего Device Farm.
* Базовые тесты производительности (k6) в CI (sanity).

**определение готовности:** push → пайплайн собирает, тестит, сканирует, пушит; Argo CD выкатывает в dev; MR создаёт эфемерную среду; разработчики изолированы по проектам.

---

## Спринт 4 — Сеть и безопасность (TLS, firewall, VPN, IP-ban)

**Цели:** mTLS внутри, защищённые периметры, управляемые бан-листы.

* gRPC мTLS между сервисами (SPIFFE/SPIRE или mutual TLS в Envoy/Ingress); cert-manager Issuer (internal CA).
* **API Gateway xN**: NGINX/Envoy, шардирование маршрутов по спискам (host/path/header) → разные gateway deployments.
* Firewalld/nftables роли Ansible + закрыть БД-порты наружу, оставить **только docker/k8s сети и whitelisted IP**; для кросс-серверных микросервисов — IP allowlist.
* **WireGuard** VPN для dev/stage; адресные пулы/ACL; onboarding скрипт.
* **IP-ban**: таблица в PostgreSQL → k8s CronJob (каждые N минут) генерит ipset/ConfigMap denylist для Ingress и хостов; Ansible таска для вне-k8s машин.
* Секреты: переход на External Secrets + Vault (опц.).

**определение готовности:** скан из внешней сети не видит БД/Redis; IP из ban-таблицы блокируется в течение X минут; гейтвеи маршрутизируют на разные инстансы; VPN работает.

---

## Спринт 5 — БД и кеш: надёжность и производительность

**Цели:** подготовить прод-уровень БД/Redis.

* PostgreSQL: HA (Patroni/StackGres) **или** single + реплика (зависит от SLA); **pgBackRest**, PITR, тест восстановления.
* Redis: Sentinel/Cluster (по нагрузке). Один общий или несколько инстансов для микросервисов — решение по метрикам.
* Пул соединений для app: pgbouncer; параметры (work\_mem, shared\_buffers) по профилю.
* Нагрузочное тестирование: pgbench/redis-benchmark + сценарии из приложения; таргеты (P95 latency, TPS).

**определение готовности:** бэкапы проходят и проверены восстановлением; базовые SLO (latency, TPS) подтверждены; дашборды БД/Redis в Grafana; алерты на репликацию/отставание/исчерпание памяти.

---

## Спринт 6 — Наблюдаемость и алертинг (логи, метрики, трассировка)

**Цели:** найти/смотреть логи, видеть аномалии и «жизнь» gRPC.

* Prometheus + Alertmanager: правило ошибок/latency (P95/P99), saturations (CPU, память, диски, сеть), SLO SLI.
* **Loki + Promtail**: парсинг логов (JSON), индексы по `trace_id`/`uuid`; «show logs around» (range query).
* Grafana: дашборды по error/warn, среднее время выполнения (duration в логах), бизнес-SLA (регистрация/логин), а также k8s cluster/node/pod.
* OpenTelemetry: экспорт трейсов/метрик; gRPC duration и error codes.

**определение готовности:** поиск по UUID находит события и «соседние» логи; приходят алерты (Slack/Telegram) по аномалиям; трассы видны; дежурство знает, куда смотреть.

---

## Спринт 7 — Прод-среда, выкаты и почта

**Цели:** подготовка и мягкая выкатка в прод.

* Terraform prod VPC/кластера; Argo CD bootstrap prod.
* Hardening: PSP/PSS, imagePolicyWebhook (опц.), только подписанные образы (cosign), секреты — только через approved механизм.
* **Argo Rollouts**: canary/blue-green, прогревающие проверки.
* API Gateway в HA, rate limiting/quotas; WAF (модуль NGINX/ModSecurity, если требуется).
* **ML-раздача**: MinIO + presigned URLs + (опц.) CDN фронт; нагрузочное тестирование скачиваний.
* **Почтовый сервер**: Mailu/Mailcow — домены, DKIM/SPF/DMARC, SMTP auth для приложения; ящик поддержки support@; релей при необходимости.

**определение готовности:** прод-кластер готов и запущен; тестовая выкатка через Argo Rollouts; письма доставляются с валидными DMARC; скачивания моделей держат целевой RPS.

---

## Спринт 8 — Оптимизация, DR, документация/хэндовер

**Цели:** снизить риски и стоимость, зафиксировать процессы.

* Полный **Runbook** (инциденты, алерты, отказы), **Onboarding** для dev/ops, диаграммы архитектуры.
* Финполитики: ретенции логов/метрик, удаление образов, защита секретов, аудиты.

**определение готовности:** задокументированные и повторяемые процедуры, ответственность передана.



