# Восстановление и подключение (Часть 2) 
# Вход
```
- ad-opa-demo: ssh -i /Users/dmitry/Downloads/id_rsa user1@192.144.13.138 cd ad-opa-demo
./preflight-check.sh
    
- litellm: ssh -i /Users/dmitry/Downloads/CICD/id_rsa user1@176.109.108.197 cd /opt/litellm-stack

- openmetadata: ssh -i /Users/dmitry/Downloads/id_rsa_3 user1@176.123.165.177 cd openmetadata
    - http://176.123.165.177:8585/signin
    - admin@open-metadata.org : admin : ApostolPavel430m#74
    - ./backup.sh
```
## Проверка
```
docker exec -it openmetadata_postgresql psql -U openmetadata_user -d openmetadata_db -c "
select name, 'database' as kind from dbservice_entity
union all
select name, 'pipeline' as kind from pipeline_service_entity
union all
select name, 'storage' as kind from storage_service_entity;
"
```
Ответ должен быть такой
```
        name         |   kind   
---------------------+----------
 ClickHouseMeta-demo | database
 demo-stand-postgres | database
 demo-stand-airflow  | pipeline
 demo-stand-minio    | storage
(4 rows)


```

# Диагностика
Если сессии в терминале разрываются, то проверь нагрузку на ВМ:
SSH-сессия может подвисать/рваться из-за нехватки памяти, а заодно и OOM-killer мог убивать процесс Postgres прямо посреди записи — это отлично объяснило бы внезапную потерю данных без явного drop/recreate в логах.
```
free -h
uptime
dmesg -T | grep -i "killed process" | tail -20
sudo dmesg -T | grep -i "killed process" | tail -20
Если не сработало, то
sudo journalctl -k --since "2 hours ago" | grep -i "out of memory\|oom-killer\|killed process"
```
Хорошо, ВМ не перезагружалась (аптайм непрерывный, 1:43, совпадает с прошлыми замерами) — значит дело не в падении/ребуте самой машины. Значит остаются: CPU steal (шумный сосед по хосту) или сетевая нестабильность между тобой и cloud.ru, либо резкая нагрузка от самого стека (Elasticsearch/миграции/ingestion).
```
vmstat 1 5
last -20
//Все входы — с твоих же IP (109.252.219.91 и 185.115.94.134), ничего постороннего.
ss -tulpn
// Порты все ожидаемые — 22 (SSH), 8585/8586 (OpenMetadata), 9200/9300 (Elasticsearch), 5432 (Postgres), 8080 (ingestion/Airflow), плюс локальный DNS-резолвер (127.0.0.53/54). Ничего постороннего/неожиданного.
cat ~/.ssh/authorized_keys
```
Просто: на своём компьютере (не на ВМ), с которого ты подключаешься по SSH, выполни:

bash
cat ~/.ssh/id_rsa.pub

или, если ключ другого типа:

bash
cat ~/.ssh/id_ed25519.pub

(если не знаешь, где лежит — ls ~/.ssh/ покажет файлы; ищи *.pub).

Сравни начало строки — если совпадает с тем, что в authorized_keys на ВМ (ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDH8976sg...), то это твой ключ, всё чисто.

# Шаг 1 Проверка текущего состояния
убедиться, что ничего не поменялось с пятницы (docker compose ps, вход admin/admin).
# Шаг 2 Бот для MCP-моста
1. В браузере: http://176.123.165.177:8585 → войди как admin/admin (пароль ещё дефолтный после сброса).
2. Settings → Bots → Add Bot.
3. Имя: mcp-bridge-bot, описание можно скопировать из документа (опционально).
4. Impersonation — оставь выключенным, как в прошлый раз.
5. Сохрани, скопируй сгенерированный JWT-токен.
```
eyJraWQiOiJHYjM4OWEtOWY3Ni1nZGpzLWE5MmotMDI0MmJrOTQzNTYiLCJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJvcGVuLW1ldGFkYXRhLm9yZyIsInN1YiI6Im1jcGFwcGxpY2F0aW9uYm90Iiwicm9sZXMiOltudWxsXSwiZW1haWwiOiJtY3BhcHBsaWNhdGlvbmJvdEBvcGVubWV0YWRhdGEub3JnIiwiaXNCb3QiOnRydWUsInRva2VuVHlwZSI6IkJPVCIsInVzZXJuYW1lIjoibWNwYXBwbGljYXRpb25ib3QiLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiJtY3BhcHBsaWNhdGlvbmJvdCIsImlhdCI6MTc5MDM5OTg4MywiZXhwIjpudWxsfQ.wY6WHpXGsoIDuuum0vlVZLdb-9l-Ud8_m2DxaFwDcCYPpAwWPm528LF0LOlqPu5ydr0oVb642azvhriium6l4N4rEd4F4r5i975WsAfKtyTtvrx4WAOYpTZEqqEk3XJPOQaChOpAtuEWGIbvCgPaKQQ_X7hFuqMt-_bAoRSmzK1LbouM9Q5XVqwo07MEtT-kKzHNS_K6iPlRP-382fvFz5mUwMeeYQ4pK3kBONONXaIdfd8CZ8lzLwDj1Hxp2GnQDVzBh16hlYHxNbyVj95-Fe6j1MYjjqcN-OqwNL6s25RhAFB4V9J8e-UnLQTYqCv3_s8DHhNwb0XwqbW14bItOA
```
6. Как скопируешь токен — сразу вставь его в .env на VM litellm-stack
```bash
cd /opt/litellm-stack
sed -i '/^OPENMETADATA_TOKEN=/d' .env
echo "OPENMETADATA_TOKEN=eyJraWQiOiJHYjM4OWEtOWY3Ni1nZGpzLWE5MmotMDI0MmJrOTQzNTYiLCJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJvcGVuLW1ldGFkYXRhLm9yZyIsInN1YiI6Im1jcGFwcGxpY2F0aW9uYm90Iiwicm9sZXMiOltudWxsXSwiZW1haWwiOiJtY3BhcHBsaWNhdGlvbmJvdEBvcGVubWV0YWRhdGEub3JnIiwiaXNCb3QiOnRydWUsInRva2VuVHlwZSI6IkJPVCIsInVzZXJuYW1lIjoibWNwYXBwbGljYXRpb25ib3QiLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiJtY3BhcHBsaWNhdGlvbmJvdCIsImlhdCI6MTc5MDM5OTg4MywiZXhwIjpudWxsfQ.wY6WHpXGsoIDuuum0vlVZLdb-9l-Ud8_m2DxaFwDcCYPpAwWPm528LF0LOlqPu5ydr0oVb642azvhriium6l4N4rEd4F4r5i975WsAfKtyTtvrx4WAOYpTZEqqEk3XJPOQaChOpAtuEWGIbvCgPaKQQ_X7hFuqMt-_bAoRSmzK1LbouM9Q5XVqwo07MEtT-kKzHNS_K6iPlRP-382fvFz5mUwMeeYQ4pK3kBONONXaIdfd8CZ8lzLwDj1Hxp2GnQDVzBh16hlYHxNbyVj95-Fe6j1MYjjqcN-OqwNL6s25RhAFB4V9J8e-UnLQTYqCv3_s8DHhNwb0XwqbW14bItOA" >> .env
grep -c OPENMETADATA_TOKEN .env   # должно быть 1
```
7. Настраиваем MCP-мост
```bash
cd /opt/litellm-stack
set -a; . ./.env; set +a
./mcp-bridge-venv/bin/pip install httpx mcp openai
python3 -m venv mcp-bridge-venv --clear
./mcp-bridge-venv/bin/pip install --upgrade pip
./mcp-bridge-venv/bin/pip install httpx mcp openai
```
8. Тест
```bash
./mcp-bridge-venv/bin/python3 mcp_bridge_test.py
```
- 15 инструментов подтянулось — значит новый токен бота рабочий, мост до OpenMetadata достучался.

9. Переиндексация, если нет коннекторов, но есть базы.
    переиндекс в свежих версиях OpenMetadata живёт как системное приложение. Попробуй так:
- Settings → Applications (в левом меню могут называться "Приложения" или через Settings → Marketplace → Installed Apps).
- Найди Search Indexing (или SearchIndexingApplication).
- Открой его → кнопка Run Now / «Запустить сейчас».
- Убедись, что в опциях выбрано «Reindex all» / «Все сущности» (не только delta).

# Шаг 3 проверка данных
```bash
cd /opt/litellm-stack
set -a; . ./.env; set +a
curl -s -H "Authorization: Bearer $OPENMETADATA_TOKEN" \
  "http://10.0.0.7:8585/api/v1/services/databaseServices?limit=20" | python3 -m json.tool
```
# Шаг 4 создаем backup и restore
~/openmetadata/backup.sh
```bash
#!/bin/bash
set -euo pipefail
cd ~/openmetadata
mkdir -p backups

TS=$(date +%Y%m%d_%H%M%S)
OUT="backups/openmetadata_db_${TS}.sql"

docker exec openmetadata_postgresql pg_dump -U openmetadata_user openmetadata_db > "$OUT"

echo "Бэкап сохранён: $OUT ($(du -h "$OUT" | cut -f1))"

ls -t backups/openmetadata_db_*.sql 2>/dev/null | tail -n +8 | xargs -r rm --
```
~/openmetadata/restore.sh
```bash
#!/bin/bash
set -euo pipefail
cd ~/openmetadata

LATEST=$(ls -t backups/openmetadata_db_*.sql 2>/dev/null | head -1)
if [ -z "$LATEST" ]; then
  echo "Бэкапов не найдено в ~/openmetadata/backups"
  exit 1
fi

echo "Восстанавливаю из: $LATEST"
read -p "Это перезапишет текущие данные в openmetadata_db. Продолжить? [y/N] " CONFIRM
if [ "$CONFIRM" != "y" ] && [ "$CONFIRM" != "Y" ]; then
  echo "Отменено."
  exit 0
fi

cat "$LATEST" | docker exec -i openmetadata_postgresql psql -U openmetadata_user -d openmetadata_db

echo "Восстановление завершено. Проверь: docker exec openmetadata_postgresql psql -U openmetadata_user -d openmetadata_db -c 'select count(*) from dbservice_entity;'"
```

Создай оба файла и сделай исполняемыми:
```bash
cd ~/openmetadata
nano backup.sh   # вставь содержимое, сохрани
nano restore.sh  # вставь содержимое, сохрани
chmod +x backup.sh restore.sh
```
Дальше — привычка: перед каждым выключением ВМ гоняй `./backup.sh.`

#### О бекап
У OpenMetadata есть официальная встроенная утилита именно для backup — openmetadata-ops.sh backup / restore (тот же образ, что мы использовали для migrate), которая под капотом делает примерно то же самое через pg_dump/pg_restore, но с доп. логикой (совместимость версий и т.д.). Мы написали свой скрипт, потому что он проще и достаточен для demo-стенда, но при желании можно переключиться на штатную утилиту — она рекомендована в их официальной документации именно потому, что by design бэкап не автоматический.

# Шаг 5 Создаем коннекторы
## Postgres-коннектор:

- Settings → Services → Databases → Add New Service.
- Тип: Postgres.
- Имя сервиса: demo-stand-postgres (как было).
- Connection:
- Username: openmetadata_ro
- Password: пароль этого read-only юзера (Volga430m74!)
- Host and Port: 10.0.0.5:5432
- Database: salesdb
- Test Connection → должно быть 8/9 (query-history/pg_stat_statements не пройдёт, это нормально, как было раньше).
- Save → затем добавь Metadata Ingestion pipeline (обычно предлагается сразу после Save) → Deploy → Run.

Если не помнишь пароль openmetadata_ro — тогда сбросим его на demo-stand через ALTER USER.

## Click-house- коннектор
### Ищем данные на демо стенде
```bash
docker compose ps | grep -i clickhouse
docker compose config | grep -A 15 clickhouse
```
Это покажет имя контейнера, порты (обычно 8123 — HTTP-интерфейс, 9000 — native) и переменные окружения с юзером/паролем (CLICKHOUSE_USER, CLICKHOUSE_PASSWORD или похожие).
### Собираем коннектор
Есть все данные. Заодно вот и пароль от postgres пользователя (BLmLcV8xaKqiSTLiFM1HY7_d) и app_user — но нам нужен именно ClickHouse:
```
Host: 10.0.0.5 (внутренний IP demo-стенда, как и для Postgres)
Port: 8123 (HTTP-интерфейс, порт открыт наружу)
Username: default
Password: YqBucHdJFWRna8KvAm1JpHW3
Database: audit (это дефолтная БД, но коннектор ClickHouse в OpenMetadata обычно видит все базы, если у пользователя есть права)
```
В форме OpenMetadata:
```
Settings → Services → Databases → Add New Service → Clickhouse.
Имя: ClickHouseMeta-demo.
Заполни хост/порт/юзер/пароль выше.
Database schema (если попросит) — можно указать audit или оставить пустым, если позволяет сканировать все.
```

## minIO- коннектор
Через mc агента. Бинарники убрали. Нужно ставить GO и скачивать с Git. Цель получить mc --version

Test Connection → жду результат.
```
dmitry@Dmitrys-MacBook-Pro ~ % mc alias set demo-minio http://192.144.13.138:9000 admin 'Eaws-qymO03nZ5k6m_cq4l6WW8I'
Added `demo-minio` successfully.
dmitry@Dmitrys-MacBook-Pro ~ % mc admin info demo-minio
●  192.144.13.138:9000
   Uptime: 2 hours 
   Version: 2025-09-07T16:13:09Z
   Network: 1/1 OK 
   Drives: 1/1 OK 
   Pool: 1

┌──────┬───────────────────────┬─────────────────────┬──────────────┐
│ Pool │ Drives Usage          │ Erasure stripe size │ Erasure sets │
│ 1st  │ 60.3% (total: 28 GiB) │ 1                   │ 1            │
└──────┴───────────────────────┴─────────────────────┴──────────────┘

1.7 MiB Used, 1 Bucket, 14 Objects
1 drive online, 0 drives offline, EC:0
dmitry@Dmitrys-MacBook-Pro ~ % mc admin user svcacct add demo-minio openmetadata-svc
Access Key: 4AKEEZPBQNRBXIXRTBWH
Secret Key: YgaXlYjk1ehnbZmydWTqzUipAJ1RTdE9Tss0kNqu
Expiration: no-expiry
```
## Включаем Sample Data
1. Открой Настройки → Сервисы → demo-stand-postgres → Агенты → AutoClassification Agent →   "Store Sample Data". Включи этот тумблер, сохрани настройки агента и запусти AutoClassification Agent заново — теперь при следующем прогоне сэмплер должен реально сохранить строки в каталог, а не просто прогнать их через классификатор PII "в памяти", как было раньше.
Скинь, что видно в его настройках — особенно любой пункт, упоминающий "sample" или "store".

3. Почти — заголовки колонок теперь видны (id, tenant_id, region, product, amount, sale_date), значит структура точно подтянулась и таблица сейчас распознана как содержащая tenant_id — это прямое подтверждение той multi-tenant изоляции, которую отражают RLS-теги, которые мы видели раньше.

Но сами строки данных всё ещё пустые — "No data". Похоже, включение тумблера сохранилось, но агент ещё не перезапускался с новой настройкой. Зайди обратно в Агенты → AutoClassification Agent и нажми «Запустить» ещё раз — теперь, когда "Store Sample Data" включён, при этом прогоне строки должны реально сохраниться.
```
docker exec -it demo-postgres psql -U postgres -d salesdb
\dt public.*
SELECT schemaname, relname, n_live_tup FROM pg_stat_user_tables;

Проверим политики
SELECT * FROM pg_policies WHERE tablename = 'sales';


```
1. tenant_isolation_policy — применяется ко всем ({public}), требует tenant_id = current_setting('app.tenant_id'). Наш openmetadata_ro не устанавливает эту сессионную переменную — значит current_setting(..., true) возвращает NULL, и tenant_id = NULL никогда не равно TRUE.
2. Эта политика одна уже отсекает всё.
tenant_a_direct_access / tenant_b_direct_access — применяются только к ролям tenant_company_a_role / tenant_company_b_role, которых openmetadata_ro не имеет.

4. То есть данные не потерялись — они на месте, но openmetadata_ro их не видит из-за RLS. Это и объясняет весь эффект пустого Sample Data.

5. Так что openmetadata_ro не подходит ни под одну из трёх PERMISSIVE-политик — а раз это PERMISSIVE (не RESTRICTIVE), при отсутствии совпадения с любой из них результат — 0 строк. Это абсолютно корректная работа RLS, никакой поломки нет — просто наш read-only пользователь оказался "невидимым посетителем" с точки зрения политик изоляции. Хорошая новость: это значит демонстрация мультитенантности работает правильно — она не даёт доступа никому, кто явно не подтверждён.

Как аккуратно дать OpenMetadata доступ, не трогая существующие политики:

Самый чистый способ — сделать openmetadata_ro обходящим RLS ролью (bypass), а не пытаться подогнать его под tenant-политики (что было бы концептуально неверно — он не относится ни к какому тенанту, это read-only каталог для метаданных):

```sql
ALTER ROLE openmetadata_ro BYPASSRLS;
```
Это единственная точечная правка, ничего не меняет в самих политиках tenant_isolation_policy/tenant_a_direct_access/tenant_b_direct_access — они продолжат работать для реальных tenant-ролей ровно как задумано. BYPASSRLS даёт этой конкретной служебной роли полный обзор данных для целей каталогизации/профилирования, что логично для инструмента governance уровня "видит всё, чтобы описать всё".

Выполни эту команду от суперпользователя (postgres), затем повтори проверку:

```sql
SET ROLE openmetadata_ro;
SELECT * FROM sales LIMIT 5;
RESET ROLE;
```
Если увидишь все 5 строк — дело в шляпе, и после этого просто повтори "Запустить" на AutoClassification Agent в OpenMetadata.
```
salesdb=# SET ROLE openmetadata_ro;
SELECT * FROM sales LIMIT 5;
RESET ROLE;
Ответ:
SET
 id | tenant_id | region | product  | amount | sale_date  
----+-----------+--------+----------+--------+------------
  1 | company_a | eu     | Widget X |   1000 | 2026-07-01
  2 | company_a | eu     | Widget Y |   1500 | 2026-07-15
  3 | company_a | uk     | Widget X |    900 | 2026-08-01
  4 | company_b | eu     | Gadget Z |   5000 | 2026-07-05
  5 | company_b | us     | Gadget W |   7000 | 2026-08-10
(5 rows)

RESET
```
Отлично, всё сработало — 5 реальных строк видны, RLS-обход через BYPASSRLS сделал своё дело, при этом сами tenant-политики остались нетронутыми, так что демонстрация мультитенантности не пострадала.

Теперь возвращайся в OpenMetadata UI и запусти AutoClassification Agent ещё раз — на этот раз Sampler должен реально сохранить эти 5 строк, и на вкладке «Пример данных» у таблицы sales наконец появятся сами значения, а не только заголовки колонок.

Ура!!!

Ура, действительно! 🎉 Вся цепочка теперь работает целиком: discovery → структура → lineage → PII-классификация → sample data — от нуля до полностью рабочего каталога с реальными данными твоего multi-tenant demo stand.
