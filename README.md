# 11-micros-03

## Задание 1

Лучшие решения из опыта: Github и Gitlab. Bitbucket был слаб в интеграции с CI/CD когда я его использовал.

| № | Требование | **GitHub + Actions** | **GitHub + Jenkins** | **GitLab CI/CD** | **AWS (CodeCommit/Build/Pipeline)** |
|---|---|---|---|---|---|
| 1 | **Облачная система** | SaaS, полностью облако | GitHub — облако, Jenkins — надо хостить самому (EC2, GKE) | SaaS (gitlab.com) или self-managed | Полностью managed AWS |
| 2 | **Git** | Git | Git | Git | Git |
| 3 | **Репозиторий на сервис** | Неограниченно (публичные бесплатно) | Неограниченно | Неограниченно (в рамках лимитов) | CodeCommit repos |
| 4 | **Сборка по событию из SCM** | `on: push`, `on: pull_request` | Webhook из GitHub → Jenkins job | `rules`, `only`, triggers | CodePipeline trigger из CodeCommit |
| 5 | **Сборка по кнопке с параметрами** | `workflow_dispatch` + `inputs` | «Build with Parameters» | Manual job + variables | Manual approval в CodePipeline, но запуск ограничен |
| 6 | **Настройки к каждой сборке** | YAML workflow + `inputs`, `env` | Параметры job + Jenkinsfile | `.gitlab-ci.yml` + variables | Через buildspec.yml + env variables |
| 7 | **Шаблоны для конфигураций** | Reusable workflows, composite actions | Shared Libraries, Job templates | `include`, `extends`, CI-компоненты | Через CodeBuild projects, слабо |
| 8 | **Секреты** | GitHub Secrets, OIDC, Vault-интеграция | Credentials Store (+ Vault plugin) | CI/CD Variables (masked, protected), Vault-интеграция | Secrets Manager, Parameter Store |
| 9 | **Несколько конфигураций из одного репо** | Много workflow-файлов, matrix | Много Jenkinsfile / параметры | Много job'ов, `parallel`, matrix | Один buildspec, ограниченно |
| 10 | **Кастомные шаги** | Любые скрипты, actions, Docker | Любые (Groovy, shell) | `script:`, любые команды | buildspec + custom commands |
| 11 | **Свои Docker-образы для сборки** | `container:` в job, свои actions | Docker agents, pods в K8s | `image:` в job, свои runner'ы | CodeBuild с custom image в ECR |
| 12 | **Свои агенты сборки** | Self-hosted runners | Свои агенты (основная модель) | Self-hosted runners | Только CodeBuild managed, свои — ограниченно |
| 13 | **Параллельные сборки** | Jobs в matrix, concurrency | Параллельные executor'ы | `parallel:`, `matrix` | Параллельные actions в CodePipeline |
| 14 | **Параллельные тесты** | Matrix strategy, `parallel` | Параллельные стадии | `parallel: matrix` | Через несколько CodeBuild projects |

### 🥇 Основной выбор: **GitLab CI/CD** (self-managed) - есть выделенный девопс (я), который сможет обслуживать всю эту инфраструктуру и будет максимальная гибкость в эксплуатации системы. Из опыта - если есть время на обслуживание это лучший вариант.
### 🥈 Альтернатива: **GitHub + GitHub Actions** - если уже гитхаб и я на парт тайме.
### ❌ Почему не GitHub + Jenkins - ощутимо сложнее обслуживать, только если экосистема уже связана с Jenkins (Java legacy).
### ❌ Почему не AWS - если уже в AWS и есть много лишних денег.


## Задание 2

| Критерий | **ELK** | **EFK** | **Grafana (Loki)** |
|---|---|---|---|
| **Компоненты** | Filebeat/Logstash → Elasticsearch → Kibana | Fluent Bit → Elasticsearch → Kibana | Promtail/Fluent Bit → Loki → Grafana |
| **Агент сбора** | Filebeat (Go) + Logstash (JVM) | Fluent Bit (C) | Promtail (Go) |
| **Агент: потребление ресурсов** | Filebeat лёгкий, Logstash тяжёлый (JVM) | Fluent Bit очень лёгкий | Promtail лёгкий |
| **Хранилище** | Elasticsearch (JVM, индексы Lucene) | Elasticsearch | Loki (индексирует только labels, чанки в S3/GCS) |
| **Что индексируется** | Полный текст + поля | Полный текст + поля | **Только labels** (метки), текст — в чанках |
| **1. Сбор со всех хостов** | DaemonSet / агент на VM | DaemonSet Fluent Bit | DaemonSet Promtail |
| **2. Сбор из stdout** | Filebeat читает `/var/log/containers/*` | Fluent Bit читает stdout K8s | Promtail читает stdout |
| **3. Гарантированная доставка** | Дисковый буфер + ack от ES | Дисковый буфер + ack от ES | Буфер есть, но Loki может отбрасывать при перегрузке |
| **4. Поиск и фильтрация** | Мощный полнотекстовый (Lucene), фильтры по полям, агрегации | То же | Сначала по labels, потом grep по чанкам. Полнотекст — медленно, ограниченно |
| **5. UI + доступ разработчикам** | Kibana: роли, Spaces, дашборды | Kibana: роли, Spaces, дашборды | Grafana: роли, teams, дашборды (единый UI для метрик + логов) |
| **6. Ссылка на сохранённый поиск** | Saved Searches → URL | Saved Searches → URL | Explore → Share link |
| **Стоимость хранения** | Дорого (ES требует RAM + SSD) | Дорого (ES) | Дёшево (S3/GCS + сжатые чанки) |
| **Ресурсоёмкость** | Высокая (JVM heap, RAM) | Высокая (ES) | Низкая |
| **Сложность эксплуатации** | Средняя/высокая (кластер ES) | Средняя (кластер ES) | Низкая (Loki проще) |
| **Масштабируемость** | Горизонтальная, но дорого | То же | Горизонтальная, дёшево |
| **Аналитика по логам** | Мощная (агрегации, визуализации) | То же | Ограниченная (нет агрегаций по тексту) |
| **Интеграция с метриками** | Через Elastic Stack (отдельно) | То же | Grafana — единый UI для метрик (Prometheus) и логов |
| **Формат запросов** | KQL / Lucene | KQL / Lucene | LogQL |
| **Порог входа** | Средний | Средний | Низкий |


### ELK — сложный парсинг, мощная аналитика (индексируют всё), много денег и времени на железо и обслуживание
### EFK — то же самое, но под k8s, меньше ресурсов потребляет
### Grafana Loki — мало денег, большие логи, уже Grafana + Prometheus, не нужен сложный поиск (индексирует только метки)

## Задание 3

| Критерий | **Prometheus + Grafana** | **Zabbix** | **TICK (InfluxDB)** | **Datadog** |
|---|---|---|---|---|
| **Модель сбора** | Pull (HTTP scrape) | Pull + active agent | Push (Telegraf) | Push (агент) |
| **1. Сбор со всех хостов** | ✅ node_exporter на каждой ноде | ✅ Zabbix Agent | ✅ Telegraf | ✅ Datadog Agent |
| **2. CPU, RAM, HDD, Network хоста** | ✅ node_exporter | ✅ Встроено | ✅ Telegraf | ✅ node_exporter | ✅ Встроено |
| **3. CPU, RAM, HDD, Network сервиса** | ✅ cAdvisor, kubelet, process-exporter | ⚠️ Ограниченно (через item'ы) | ✅ Telegraf + docker input | ✅ Встроено |
| **4. Специфичные метрики сервиса** | ✅ `/metrics` endpoint (клиентские библиотеки) | ✅ Через user parameters / trapper | ✅ Telegraf plugins | ✅ DogStatsD, APM |
| **5. UI с запросами и агрегацией** | ✅ Grafana + PromQL (мощный) | ⚠️ UI есть, но запросы слабее | ✅ Chronograf + InfluxQL/Flux | ✅ Мощный UI |
| **6. UI с панелями** | ✅ Grafana dashboards | ✅ Zabbix dashboards | ✅ Chronograf dashboards | ✅ Dashboards |
| **Хранение** | TSDB на диске | Реляционная БД (MySQL/Postgres) | InfluxDB (TSDB) | SaaS |
| **Ресурсоёмкость** | 🟡 Средняя | 🟡 Средняя | 🟡 Средняя | 🟢 SaaS |
| **Масштабируемость** | ✅ Federation, remote write | ✅ Прокси, кластер | ✅ Кластер (Enterprise) | ✅ SaaS |
| **Алертинг** | ✅ Alertmanager | ✅ Встроено | ✅ Kapacitor | ✅ Встроено |
| **Service Discovery** | ✅ K8s, Consul, EC2 | ✅ Auto-discovery | ⚠️ Ограниченно | ✅ Авто |
| **Vendor lock-in** | ❌ Нет | ❌ Нет | ❌ Нет | ✅ Да |
| **Стоимость** | 🟢 Бесплатно | 🟢 Бесплатно | 🟡 OSS + Enterprise | 🔴 Дорого |

### 🥇 Основной выбор: **Prometheus + Grafana** - наиболее опытное и широкое решение для микросервисов
### ❌ Почему не Zabbix - слабая агрегация, лучше для мониторинга железа или VM
### ❌ Почему не TICK - в Chronograf хуже функционал чем у Grafana, только если нужна push модель
### ❌ Почему не Datadog - слышал о нем, его тоже поэтому включил. Дорого и привязано к вендору.
