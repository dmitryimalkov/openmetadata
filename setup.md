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
