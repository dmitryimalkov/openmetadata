# Вход
```
- ad-opa-demo: ssh -i /Users/dmitry/Downloads/id_rsa user1@192.144.13.138 cd ad-opa-demo
- litellm: ssh -i /Users/dmitry/Downloads/CICD/id_rsa user1@176.109.108.197 cd /opt/litellm-stack
- openmetadata: ssh -i /Users/dmitry/Downloads/id_rsa_3 user1@176.123.165.177 cd openmetadata
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
eyJraWQiOiJHYjM4OWEtOWY3Ni1nZGpzLWE5MmotMDI0MmJrOTQzNTYiLCJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJvcGVuLW1ldGFkYXRhLm9yZyIsInN1YiI6ImkiLCJyb2xlcyI6W10sImVtYWlsIjoiaUBkbWFsa292LnJ1IiwiaXNCb3QiOnRydWUsInRva2VuVHlwZSI6IkJPVCIsInVzZXJuYW1lIjoiaSIsInByZWZlcnJlZF91c2VybmFtZSI6ImkiLCJpYXQiOjE3OTAzNjE1NTIsImV4cCI6bnVsbH0.sFUz8kJT1gxwXKR2yiljr6S5rLNyjag_Bui880AbN8LxTQmUcTBch4v-FRob6FpsrUaU7x-8-wAzVPx8OBFaTCIqYCZTIdVDQrpIkQwNw4P2XbK7Cbyd3SVx2w3R3C7-8TgYBTH4rYME4Cw6h0fz0I2f_2CoaWxuGlXW77iIylvwTBk9pMxS1_Gh8qGmiZ613i636eNqK0XyA-_m81fiHTCN9jgdKYgWWXHPB0z8wbLrinKii_AxhnRay5fnjwtT_W9Rr-Mm83JP8Bng4DB8CCAfAy_rqGM8gY46vyaYZVmAl6RsUSrXwFqIOOG1zcEaPe0IYHFgObKLHi_T70RrWQ
```
6. Как скопируешь токен — сразу вставь его в .env на VM litellm-stack
```bash
cd /opt/litellm-stack
sed -i '/^OPENMETADATA_TOKEN=/d' .env
echo "OPENMETADATA_TOKEN=eyJraWQiOiJHYjM4OWEtOWY3Ni1nZGpzLWE5MmotMDI0MmJrOTQzNTYiLCJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJvcGVuLW1ldGFkYXRhLm9yZyIsInN1YiI6ImkiLCJyb2xlcyI6W10sImVtYWlsIjoiaUBkbWFsa292LnJ1IiwiaXNCb3QiOnRydWUsInRva2VuVHlwZSI6IkJPVCIsInVzZXJuYW1lIjoiaSIsInByZWZlcnJlZF91c2VybmFtZSI6ImkiLCJpYXQiOjE3OTAzNjE1NTIsImV4cCI6bnVsbH0.sFUz8kJT1gxwXKR2yiljr6S5rLNyjag_Bui880AbN8LxTQmUcTBch4v-FRob6FpsrUaU7x-8-wAzVPx8OBFaTCIqYCZTIdVDQrpIkQwNw4P2XbK7Cbyd3SVx2w3R3C7-8TgYBTH4rYME4Cw6h0fz0I2f_2CoaWxuGlXW77iIylvwTBk9pMxS1_Gh8qGmiZ613i636eNqK0XyA-_m81fiHTCN9jgdKYgWWXHPB0z8wbLrinKii_AxhnRay5fnjwtT_W9Rr-Mm83JP8Bng4DB8CCAfAy_rqGM8gY46vyaYZVmAl6RsUSrXwFqIOOG1zcEaPe0IYHFgObKLHi_T70RrWQ" >> .env
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
