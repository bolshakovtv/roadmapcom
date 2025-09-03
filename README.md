# Дорожная карта (2 месяца, 4 спринтов)

**Среды и кластера:**

* **Внутренние сервисы** — один кластер (gitlab, vpn etc.). **prod** — отдельный кластер с k8s.

## Спринт 1 — Фундамент IaC и репозитории (Ansible, GitLab, каркас k8s)

**Цели:** запустить базовую инфраструктуру dev/stage, репозитории и раннеры.

* Ansible: базовый образ ОС (DONE), users/SSH, firewalld/nftables baseline, journald/rsyslog, node-exporter, crond. (По минимому)
* GitLab: группа/проекты, **раздельная видимость** (developers не видят чужие проекты), protected branches/tags, артефакты.
* GitLab Runners в k8s (separate executor, изоляция по projects/tag). (вместе с пайплайнами)
* БД dev: PostgreSQL single + Redis (1 экземпляр), PV + бэкап-крон (pg\_dump + рестор-тест). (Уточнить у провайдера про сервисы, сравнить цены селфхостед и сервис)
* Док: конвенции Dockerfile (multi-stage, non-root, distroless), .env → values.yaml/Secrets.
* Оркестратор (протестировать, базавоя развертка)

**определение готовности:** GitLab доступен, Runners собирают образ и пушат в Nexus; dev k8s живой; один сервис деплоится вручную helm install; базовый бэкап БД работает.

---

## Спринт 2 — K8s-надстройки, GitOps, ingress для gRPC, витрина ML

**Цели:** полнофункциональная платформа dev/stage.

* Ingress NGINX c HTTP/2 и **gRPC**, **cert-manager** (Let’s Encrypt staging + внутренний CA), **external-dns**.
* **s3 как сервис** для ML Vosk файлов + presigned URLs; CI-джоб публикации моделей; (альтернатива — Nexus raw repo).
* Продовое исполнение k8s
* Эмуляторы: headless **Android Emulator** image в CI (Linux, KVM), smoke-тесты; опция внешнего Device Farm.
* **WireGuard** VPN для dev/stage; адресные пулы/ACL; onboarding скрипт.
* **IP-ban**: таблица в PostgreSQL → k8s CronJob (каждые N минут) генерит ipset/ConfigMap denylist для Ingress и хостов; Ansible таска для вне-k8s машин.
* Секреты: переход на External Secrets + Vault (опц.).

**определение готовности:** PR → build → авто-деплой в prod, k8s запущен и используется;

---

## Спринт 3 — Наблюдаемость и алертинг (логи, метрики)

**Цели:** найти/смотреть логи, видеть аномалии и «жизнь» gRPC.

* Prometheus + Alertmanager: правило ошибок/latency (P95/P99), saturations (CPU, память, диски, сеть), SLO SLI.
* **Loki + Promtail**: парсинг логов (JSON), индексы по `trace_id`/`uuid`; «show logs around» (range query).
* Grafana: дашборды по error/warn, среднее время выполнения (duration в логах), бизнес-SLA (регистрация/логин), а также k8s cluster/node/pod.

**определение готовности:** Мониторинг развернут, автоматизирован, уведомления приходят (проверить на тестовых уведомлениях), логи удобно искать и читать; приходят алерты (Slack/Telegram) по аномалиям. Дежурство знает, куда смотреть.

---

## Спринт 8 — Оптимизация, документация

**Цели:** снизить риски и стоимость, зафиксировать процессы.

* Полный **Runbook** (инциденты, алерты, отказы), **Onboarding** для dev/ops, диаграммы архитектуры.
* Финполитики: ретенции логов/метрик, удаление образов, защита секретов, аудиты.

**определение готовности:** задокументированные и повторяемые процедуры, ответственность передана.

---

## Спринт на будущее: надёжность и производительность

**Цели:** подготовить прод-уровень БД/Redis.

* PostgreSQL: HA (Patroni/StackGres) **или** single + реплика (зависит от SLA); **pgBackRest**, PITR, тест восстановления. (на будущее)
* Redis: Sentinel/Cluster (по нагрузке). Один общий или несколько инстансов для микросервисов — решение по метрикам. ()
* Пул соединений для app: pgbouncer; параметры (work\_mem, shared\_buffers) по профилю.
* Нагрузочное тестирование: pgbench/redis-benchmark + сценарии из приложения; таргеты (P95 latency, TPS).
* Terraform: VPC/подсети, NAT, брандмауэр, LB, dev/stage k8s (managed/cluster-api/kubespray), PV-класс (NVMe), DNS зоны.
* **Nexus**: hosted docker/helm/raw; роли/LDAP(опц.), ретенции, proxy к Docker Hub.
* Cilium + NetworkPolicy baseline; PodSecurity/PodDisruptionBudgets; liveness/readiness стартовый шаблон в Helm chart.
* **Argo CD** bootstrap: app-of-apps; окружения dev/stage; policy RBAC (читатели/деплойеры).
* Helm-чарты каркаса для микросервисов (общий chart lib/copybara).
* Секреты: SOPS + age (или Sealed Secrets) для values.
* GitLab CI: шаблоны `.gitlab-ci.yml` (build/test/lint/scan/push), кэш, SBOM (syft), Trivy scan, семантические версии.
* Разделение проектов: группы/сабгруппы, роли Maintainer/Developer, видимость **private** по умолчанию, доступ к общим shared libs по токенам.
* gRPC мTLS между сервисами (SPIFFE/SPIRE или mutual TLS в Envoy/Ingress); cert-manager Issuer (internal CA).
* **API Gateway xN**: NGINX/Envoy, шардирование маршрутов по спискам (host/path/header) → разные gateway deployments.
* Firewalld/nftables роли Ansible + закрыть БД-порты наружу, оставить **только docker/k8s сети и whitelisted IP**; для кросс-серверных микросервисов — IP allowlist.
* OpenTelemetry: экспорт трейсов/метрик; gRPC duration и error codes.
* Terraform prod VPC/кластера; Argo CD bootstrap prod.
* Hardening: PSP/PSS, imagePolicyWebhook (опц.), только подписанные образы (cosign), секреты — только через approved механизм.
* **Argo Rollouts**: canary/blue-green, прогревающие проверки.
* API Gateway в HA, rate limiting/quotas; WAF (модуль NGINX/ModSecurity, если требуется).
* **ML-раздача**: MinIO + presigned URLs + (опц.) CDN фронт; нагрузочное тестирование скачиваний.
* **Почтовый сервер**: Mailu/Mailcow — домены, DKIM/SPF/DMARC, SMTP auth для приложения; ящик поддержки support@; релей при необходимости.
