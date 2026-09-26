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

