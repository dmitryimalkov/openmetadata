# OpenMetadata POC — Настройка подключений (Часть 1)

Sep 20, 2026 · @Someone

## Обзор и топология

OpenMetadata 2.0.2 развёрнут через официальный docker-compose на отдельной выделенной ВМ — намеренно вынесен от демо-стенда AD/LDAP + Keycloak + OPA + Postgres + ClickHouse + Airflow + MinIO, чтобы не рисковать многотенантной демонстрацией.

| ВМ | Роль | Внутренний IP | Публичный IP | Конфигурация |
| --- | --- | --- | --- | --- |
| openmetadata-vm | OpenMetadata сервер | 10.0.0.7 | 176.123.165.177 | 4 vCPU / 8GB RAM / 50GB SSD, Ubuntu 24.04 |
| demo-stand (192.144.13.138) | AD/LDAP, Keycloak, OPA, Postgres, ClickHouse, Airflow, MinIO | 10.0.0.5 | 192.144.13.138 | существующий стенд |

**Важно:** обе ВМ находятся в одной security group/подсети, поэтому все коннекторы OpenMetadata настроены на **внутренний IP** демо-стенда (`10.0.0.5`), а не на публичный — подключение по публичному IP стабильно давало таймаут на всех трёх сервисах (Postgres, Airflow, MinIO).

### Общий принцип безопасности

Главное ограничение всей работы: **ни одна политика изоляции тенантов не должна быть изменена или ослаблена** — ни RLS-политики в Postgres (`tenant_isolation_policy`, `tenant_a_direct_access`, `tenant_b_direct_access`), ни правила `authz.rego` (OPA), которые управляют доступом по ролям и тенантам.

Вместо этого для каждого источника создан отдельный технический (служебный) аккаунт с минимальными правами, а любые точечные обходы (например, `BYPASSRLS`) применены только к этому одному служебному аккаунту, не затрагивая существующие политики:

- **Postgres**: роль `openmetadata_ro` — read-only, с `BYPASSRLS` (подробности в разделе Postgres)
- **Airflow / MinIO**: LDAP-пользователь `openmetadata-svc` — единая учётная запись для обоих LDAP-based сервисов

## Postgres (salesdb)

### 1. Создание роли в Postgres

Выполняется на демо-стенде: `docker exec -it demo-postgres psql -U postgres`

```sql
CREATE USER openmetadata_ro WITH PASSWORD 'Ваш_пароль';
\c salesdb
GRANT CONNECT ON DATABASE salesdb TO openmetadata_ro;
GRANT USAGE ON SCHEMA public TO openmetadata_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO openmetadata_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO openmetadata_ro;

-- обязательно, см. пункт ниже про Row-Level Security
ALTER ROLE openmetadata_ro BYPASSRLS;
```

### 2. Настройка в OpenMetadata UI

Settings → Services → Databases → Add New Service → Postgres

| Поле | Значение |
| --- | --- |
| Host | `10.0.0.5` (внутренний IP, не публичный) |
| Port | `5432` |
| Database | `salesdb` |
| Username | `openmetadata_ro` |
| Password | Задан при создании роли |

После успешного Test Connection — мастер выбора объектов для сканирования (базы → схемы → таблицы/процедуры) → Create & Deploy.

### 3. Известные проблемы и решения

**Таймаут при подключении по публичному IP.** Решение: использовать внутренний IP `10.0.0.5`, так как обе ВМ в одной подсети.

**Profiler Agent показывает «Filtered: 1», 0 обработанных записей.** Причина: автосозданный Profiler Agent имел в разделе «Классификации» фильтр «Только определённые классификации» (паттерны `начинается с Tier1`/`Tier2`), исключающий не-Тирированные таблицы (`sales`). Исправлено переключением на «Сканировать все классификации».

**Sample Data пустой даже после успешного AutoClassification.** Причина — три PERMISSIVE Row-Level Security политики на таблице `sales` (часть демо-стенда мультитенантности), под которые `openmetadata_ro` не подходил. **Решение:** `ALTER ROLE openmetadata_ro BYPASSRLS;` — это точечно освобождает от RLS только эту одну служебную роль и никак не затрагивает и не изменяет сами политики изоляции тенантов — демо-кейс остаётся полностью рабочим.

**GetQueries — решено.** `pg_stat_statements` требует загрузки через `shared_preload_libraries`, что применяется только при пересоздании контейнера Postgres:

1. В `docker-compose.yml` демо-стенда, в сервисе `postgres`, добавлена строка:

```yaml
command: ["postgres", "-c", "shared_preload_libraries=pg_stat_statements", "-c", "pg_stat_statements.track=all"]
```

2. Применено через `docker compose up -d postgres` (пересоздаёт только контейнер Postgres, данные в volume `pgdata` не затронуты, RLS/OPA не трогали)
3. Расширение включено в каждой сканируемой базе:

```sql
\c salesdb
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
\c airflow
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

После пере-запуска Metadata Agent тест соединения показывает 9/9 — `GetQueries` больше не предупреждение.

## Airflow (Pipeline Service)

Айрфлоу на демо-стенде использует native LDAP-аутентификацию и кастомный OPA-авторизатор (`OpaFabAuthManager`), который проверяет каждый запрос через `authz.rego`, игнорируя стандартные роли Flask-AppBuilder (FAB).

### 1. Создание служебного LDAP-аккаунта

`/tmp/openmetadata-svc.ldif` (на демо-стенде, вне контейнера — можно удалить после импорта):

```
dn: uid=openmetadata-svc,ou=people,dc=demo,dc=local
objectClass: inetOrgPerson
uid: openmetadata-svc
cn: OpenMetadata Service
sn: Service
mail: openmetadata-svc@platform.internal
userPassword: {SSHA}CormEkr8mEb0PGRF+yhSDTHFNAYSJfvg
```

Импорт через `osixia/openldap` контейнер:

```bash
ldapadd -x -D "cn=admin,dc=demo,dc=local" -w AdminPass123! -f /tmp/openmetadata-svc.ldif
```
> [!NOTE]
> Остановились здесь

Это обычный текст, а <span style="color: blue;">этот фрагмент — синий</span>.

### 2. Роль FAB (необходима, но недостаточна)

```bash
airflow users add-role -u openmetadata-svc -r Admin
```

### 3. Проблема OPA-авторизации

Роль Admin в FAB **не снимала** 403 Forbidden на DAG-ах и переменных — `OpaFabAuthManager` переопределяет `is_authorized_dag`/`get_authorized_dag_ids`/`is_authorized_variable` и всё равно ходит в OPA. В `authz.rego` не было готовой кросс-тенантной группы — только `CompanyA-Admins`, `CompanyB-Viewers` и т.п. **Решение:** точечное правило в `authz.rego`, дающее сервисному аккаунту `openmetadata-svc` просмотр DAG-ов обоих тенантов без изменения ролевой модели для остальных пользователей.

### 4. Настройка в OpenMetadata UI

Settings → Services → Pipelines → Add New Service → Airflow

| Поле | Значение |
| --- | --- |
| Host and Port | `10.0.0.5:8090` |
| Connection | Airflow, Auth type LDAP |
| Username | `openmetadata-svc` |
| Password | Пароль из LDIF |

Test Connection — 2/2 проверки, далее — сканирование всех пайплайнов → Create & Deploy. Оба тенантских DAG-а (`company_a__load_dataset`, `company_b__load_dataset`) видны из одного кросс-тенантного аккаунта.

### 5. Известные ограничения

- Airflow 3.x использует FastAPI/uvicorn, классический `curl POST /login/` не работает (405) — для ручной проверки через браузер и SSH-туннель (порт 8090 закрыт снаружи).
- Lineage от Airflow-пайплайнов к таблице `sales` не появляется автоматически — потребовало бы `inlets`/`outlets` в коде DAG или OpenLineage provider. Отложено по решению пользователя.

## MinIO / S3 (Storage Service)

### 1. Проблема: MinIO в режиме LDAP identity provider

MinIO на демо-стенде настроен с `MINIO_IDENTITY_LDAP_*` переменными окружения — это отключает внутренний IAM. Команда `mc admin user add` даёт ошибку «Specified IAM action is not allowed» — нужен другой подход для выдачи статических ключей LDAP-пользователю.

**Решение:** переиспользован LDAP-аккаунт `openmetadata-svc`, уже созданный для Airflow.

### 2. Привязка политики

```bash
docker exec demo-minio mc idp ldap policy attach local readonly \
  --user "uid=openmetadata-svc,ou=people,dc=demo,dc=local"
```

### 3. Генерация статического Access Key / Secret Key

Неудачные попытки: `mc admin user svcacct add` — ошибка «User DN not found», хотя `ldapsearch` подтверждает существование записи; `mc idp ldap login` — такой команды не существует в этой версии `mc`.

**Рабочее решение** — `mc idp ldap accesskey create-with-login`, логинится по LDAP-учётным данным и генерирует пару ключей. Требует реального TTY (не работает через heredoc/pipe в stdin):

```bash
docker exec -it demo-minio mc idp ldap accesskey create-with-login http://localhost:9000 \
  --name "openmetadata-connector" \
  --description "Read-only access key for OpenMetadata S3 connector"
# далее интерактивно вводится LDAP username и password
```

### 4. Настройка в OpenMetadata UI

Settings → Services → Storages → Add New Service → S3

| Поле | Значение |
| --- | --- |
| AWS Access Key ID | Из `create-with-login` |
| AWS Secret Access Key | Из `create-with-login` |
| AWS Region | `us-east-1` (обязательное поле SDK, MinIO его не проверяет) |
| Endpoint URL | `http://10.0.0.5:9000` |

### 5. GetMetrics warning

Test Connection показывает `s3:ListBuckets` — успешно, но `cloudwatch:ListMetrics` — ошибка (несовместимость версий STS API). Это ожидаемо: MinIO не реализует CloudWatch API вообще, а этот шаг необязателен (доп. статистика использования, не каталогизация). Можно смело продолжать деплой.

Результат: сервис `demo-stand-minio` создан, 1 контейнер (бакет) закаталогизирован. Бакет пуст, поэтому внутри нет объектов — это ожидаемо.

## ClickHouse (audit)

ClickHouse на демо-стенде хранит аудит-лог решений OPA в базе `audit` (таблицы `audit_log`, `request_log`). Порт 8123 (HTTP-интерфейс) уже проброшен наружу, родной протокол на 9000 — нет.

### 1. Проблема: SQL-driven access control выключен

`CREATE USER` через SQL даёт ошибку `ACCESS_DENIED` — у `default` нет права `CREATE USER ON *.*`. Чтобы его выдать, нужно включить `access_management` в конфиге и пересоздать контейнер (аналогично `shared_preload_libraries` в Postgres).

### 2. Включение access\_management

1. Файл `/home/user1/ad-opa-demo/clickhouse-config/zz-access-management.xml`:

```xml
<clickhouse>
  <users>
    <default>
      <access_management>1</access_management>
    </default>
  </users>
</clickhouse>
```

2. Volume в `docker-compose.yml` (сервис `clickhouse`):

```yaml
    volumes:
      - ./clickhouse-init:/docker-entrypoint-initdb.d
      - chdata:/var/lib/clickhouse
      - ./clickhouse-config/zz-access-management.xml:/etc/clickhouse-server/users.d/zz-access-management.xml:ro
```

3. `docker compose up -d clickhouse`

**Важный нюанс: порядок загрузки файлов в `users.d`.** Образ ClickHouse сам создаёт `default-user.xml` из `CLICKHOUSE_USER`/`CLICKHOUSE_PASSWORD`, где явно стоит `<access_management>0</access_management>`. Файлы в `users.d` грузятся по алфавиту, и `default-user.xml` перебивал настройку из файла, начинающегося на `a` (наш первоначальный `access-management.xml`). **Решение:** переименовали файл в `zz-access-management.xml`, чтобы он гарантированно грузился последним и побеждал.

### 3. Создание роли и прав

```sql
CREATE USER IF NOT EXISTS openmetadata_ro IDENTIFIED WITH plaintext_password BY 'ClickHouse_ro_2026!Xk9';
GRANT SELECT ON audit.* TO openmetadata_ro;
GRANT SELECT ON system.query_log TO openmetadata_ro;
```

(последний GRANT — аналог `pg_stat_statements` для Postgres, нужен для query-based lineage; без него `GetQueries` падает с `ACCESS_DENIED` по `system.query_log`)

### 4. Настройка в OpenMetadata UI

Settings → Services → Databases → Add New Service → ClickHouse

| Поле | Значение |
| --- | --- |
| Username | `openmetadata_ro` |
| Password | `ClickHouse_ro_2026!Xk9` |
| Host and Port | `10.0.0.5:8123` |
| Database Schema | `audit` |
| Use HTTPS Protocol | выключено (у нас plain HTTP) |

Test Connection — 4/4 проверки с первого раза. Результат: сервис `ClickHouseMeta-demo`, 1 база, 1 схема, 2 таблицы (`audit_log`, `request_log`), 14 колонок.

## Единый граф lineage (demo)

Задача: собрать реальный, рабочий end-to-end граф `sales.orders (Postgres) → daily_sales_report (Airflow) → sales_by_region.json (S3) → метрика`, чтобы изучить, как OpenMetadata показывает единый граф метаданных, и где заканчивается автоматика и начинается ручная работа.

### 1. Новые технические аккаунты

Отдельные, специально заведённые под эту задачу — не переиспользовали `openmetadata_ro`/`openmetadata-svc`, у них другое назначение (read-only для каталогизации):

- **Postgres**: роль `reporting_ro` — `SELECT` на `sales` + `BYPASSRLS` (нужен для кросс-тенантной агрегации по всем регионам)
- **MinIO (LDAP)**: пользователь `airflow-writer` — тот же паттерн, что и `openmetadata-svc` (LDAP identity provider у MinIO отключает встроенный IAM), но с политикой на запись в бакет `datasets`

```sql
CREATE ROLE reporting_ro LOGIN PASSWORD 'Ваш_пароль' BYPASSRLS;
GRANT CONNECT ON DATABASE salesdb TO reporting_ro;
\c salesdb
GRANT USAGE ON SCHEMA public TO reporting_ro;
GRANT SELECT ON sales TO reporting_ro;
```

LDIF для `airflow-writer` — по образцу `openmetadata-svc.ldif` из раздела Airflow выше, только `uid=airflow-writer`. Политика записи в MinIO:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::datasets", "arn:aws:s3:::datasets/*"]
    }
  ]
}
```

```bash
docker exec demo-minio mc admin policy create local datasets-write /tmp/airflow-writer-policy.json
docker exec demo-minio mc idp ldap policy attach local datasets-write \
  --user "uid=airflow-writer,ou=people,dc=demo,dc=local"
docker exec -it demo-minio mc idp ldap accesskey create-with-login http://localhost:9000 \
  --name "airflow-writer-key" --description "Write access key for daily_sales_report DAG"
```

### 2. DAG daily\_sales\_report

Файл: `/home/user1/ad-opa-demo/airflow-dags/daily_sales_report.py` (том `./airflow-dags:/opt/airflow/dags`). Реально читает `sales` под `reporting_ro`, агрегирует `SUM(amount) GROUP BY region`, пишет `datasets/reports/sales_by_region.json` через `boto3` под `airflow-writer`. Пароли/ключи — не в коде, а в Airflow Variables:

```bash
docker exec demo-airflow airflow variables set REPORTING_PG_PASSWORD '...'
docker exec demo-airflow airflow variables set MINIO_WRITER_ACCESS_KEY '...'
docker exec demo-airflow airflow variables set MINIO_WRITER_SECRET_KEY '...'
```

TaskFlow API (`@dag`/`@task`), хосты `postgres`/`minio` — имена сервисов в docker-сети `demo-net` (та же схема, что у `AIRFLOW__DATABASE__SQL_ALCHEMY_CONN`).

Запуск: `airflow dags unpause daily_sales_report` → `airflow dags trigger daily_sales_report`. Оба таска — `success`, файл реально появился в MinIO (375 байт, JSON с суммами по регионам eu/uk/us).

### 3. Манифест для каталогизации файла в S3-коннекторе

По умолчанию S3/Storage-коннектор OpenMetadata каталогизирует только сам бакет как единый Container — не заглядывает внутрь без манифеста. Нужен файл `openmetadata.json` в корне бакета:

```json
{
  "entries": [
    {
      "dataPath": "reports",
      "structureFormat": "json",
      "isPartitioned": false
    }
  ]
}
```

```bash
docker exec demo-minio mc cp /tmp/openmetadata-manifest.json local/datasets/openmetadata.json
```

После этого коннектор попытался листить объекты по префиксу `reports/` и упал с ошибкой доступа — политика `readonly`, привязанная к `openmetadata-svc`, разрешает только `s3:GetBucketLocation` + `s3:GetObject`, без `s3:ListBucket`. Завели отдельную политику (не трогая общий `readonly`, вдруг используется где-то ещё):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetBucketLocation", "s3:ListBucket", "s3:GetObject"],
      "Resource": ["arn:aws:s3:::*", "arn:aws:s3:::*/*"]
    }
  ]
}
```

```bash
docker exec demo-minio mc admin policy create local openmetadata-storage-read /tmp/openmetadata-storage-read-policy.json
docker exec demo-minio mc idp ldap policy attach local openmetadata-storage-read \
  --user "uid=openmetadata-svc,ou=people,dc=demo,dc=local"
```

После повторного запуска Metadata Agent — контейнер `reports` появился со схемой из 3 колонок (`regions` массив с `region`/`order_count`/`total_amount`, `generated_at`, `source_table`).

### 4. Построение lineage-рёбер и Metric

Автоматического lineage между Postgres/Airflow/S3 в этой связке нет: query-based lineage (`pg_stat_statements`) работает только для SQL→SQL, а Airflow→S3 требует `inlets`/`outlets` в коде DAG или OpenLineage-провайдер — сознательно не подключали. Поэтому рёбра построены вручную, в режиме Edit Lineage (UI): узел добавляется drag-and-drop с левой панели типов (Tables/Pipelines/Containers/...), затем связь протягивается между хендлами узлов.

Создана Metric-сущность (Governance → Metrics):

- Name: `daily_sales_by_region`, Display Name: «Ежедневные продажи по регионам»
- Metric Type: `Sum`, Granularity: `Day`
- Связь с данными — тоже через вкладку Lineage самой метрики (поля Related Data Assets в форме создания нет в этой версии), ребро `reports → метрика` протянуто вручную так же, как остальные.

Итоговый граф (все узлы реальные, не декоративные):

`sales (Postgres) → daily_sales_report (Airflow) → reports/sales_by_region.json (MinIO) → «Ежедневные продажи по регионам» (Metric)`

## Отложенные задачи

- [x] **Выполнено.** Расширение `pg_stat_statements` включено на Postgres демо-стенда (см. шаги в разделе Postgres выше) — GetQueries теперь проходит (9/9 проверок), query-based lineage доступен.
- [x] **Выполнено.** ClickHouse подключён как четвёртый источник (см. раздел ClickHouse выше) — все четыре источника (Postgres, Airflow, MinIO, ClickHouse) теперь подключены к OpenMetadata.
- [x] **Выполнено (частично).** Собран реальный сквозной граф: `sales` (Postgres) → `daily_sales_report` (Airflow) → `reports/sales_by_region.json` (MinIO) → метрика «Ежедневные продажи по регионам» — см. раздел «Единый граф lineage (demo)» выше. Рёбра построены вручную через Edit Lineage в UI; автоматический lineage Airflow→sales (`inlets`/`outlets` или OpenLineage) по-прежнему не настроен — сознательно отложено.

## Справочник учётных данных

| Сервис | Аккаунт | Замечание |
| --- | --- | --- |
| Postgres (salesdb) | `openmetadata_ro` | Роль с `SELECT` + `BYPASSRLS` |
| Airflow (LDAP) | `openmetadata-svc` | uid=openmetadata-svc,ou=people,dc=demo,dc=local; роль Admin в FAB + доп. правило в authz.rego |
| MinIO (S3) | `openmetadata-svc` (тот же LDAP-пользователь) | Политика `readonly` + отдельный Access Key/Secret Key через `create-with-login` |

Пароли и ключи не дублируются здесь открытым текстом — храни их в своём менеджере секретов; пароли/ключи, сгенерированные в ходе POC, есть в истории команд на обеих ВМ.
