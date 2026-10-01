# Отладка- основные команды
1. Проверка адреса: `http://`: http://172. И так далее

Если внедрена политика авторизации, то доступ определяется политикой.
Например, при внедрении OP_Agent доступ к DAG решает не FAB-роль, а OpaFabAuthManager через OPA, и учётке каталога нужна кросс-тенантная сервисная роль. 

**1. Убедитесь, что причина в OPA** (на ВМ OpenMetadata):
```
docker exec -it \<ingestion_container\> curl -s -o /dev/null -w
'%{http_code}\n' \\
-u '\<user\>:\<password\>' http://172.21.101.83:8090/api/v1/dags
docker exec -it openmetadata_ingestion curl -s -o /dev/null -w
'%{http_code}\n' \\
-u 'openmetadata-svc:Password123!' http://172.21.101.83:8090/api/v1/dags
```
- Если вернулся 403, на ВМ ad-opa-demo посмотрите логи в момент запроса:
```
docker compose logs --tail 50 airflow-webserver \| grep -iE
'opa\|deny\|403\|openmetadata-svc'

docker compose logs --tail 50 opa \| grep -i '\<user\>'
```
**2. Проверьте три звена цепочки на ВМ ad-opa-demo.**

a\) **LDAP.** Учётка (openmetadata-svc или та, что вы завели в dev)
должна состоять в группе без тенантного префикса, например
Platform-MetadataIngestion:
```
docker compose exec \<ldap_container\> ldapsearch -x -H ldap://localhost
\\
-D "cn=admin,dc=demo,dc=local" -W -b "dc=demo,dc=local"
"(cn=Platform-MetadataIngestion)" member
```
Если группы нет, создайте её через ldif и добавьте в member DN учётки.

b\) **opa_auth_manager.py.** В ROLE_MAP должна быть строка:
```
ROLE_MAP = {"Analysts": "analyst", "Admins": "admin", "Viewers":
"viewer", "MetadataIngestion": "metadata_ingestion"}
```
c\) **authz.rego.** Роль metadata_ingestion должна быть в service_roles, чтобы обходить same_tenant. Кроме того, в role_permissions у неё должно быть чтение пайплайнов (DAG):
```
grep -nE 'metadata_ingestion\|service_roles' \<путь\>/authz.rego
```
**3. Перезапустите webserver.** Правки в .py без этого не применятся. OPA перечитывает .rego сам.

```
docker compose restart airflow-webserver
```

Затем повторите curl из шага 1: должен вернуться 200. После этого
нажмите **Retry Test** в OpenMetadata.

Если после этих шагов 403 останется, пришлите:

- вывод grep по authz.rego;

- строки ROLE_MAP из opa_auth_manager.py;

- логи webserver и OPA за момент запроса;

- **Show raw connection log** из окна теста.

Ошибка оказалась 404

404 говорит о том, что этот путь не существует, а не о запрете доступа.
Так бывает в двух случаях:

- **Airflow 3.x.** В нём убрали API v1, остался только /api/v2/....

- **Airflow работает под префиксом** (base_url), например
  http://…:8090/airflow/.

Проверьте, какой из случаев ваш (из контейнера ingestion):

for p in /api/v2/version /api/v1/version /api/v2/monitor/health /health;
do

printf '%-28s ' "\$p"

docker exec \<ingestion_container\> curl -s -o /dev/null -w
'%{http_code}\n' \\

-u '\<user\>:\<password\>' "http://172.21.101.83:8090\$p"

Done

for p in /api/v2/version /api/v1/version /api/v2/monitor/health /health;
do

printf '%-28s ' "\$p"

docker exec openmetadata_ingestion curl -s -o /dev/null -w
'%{http_code}\n' \\

-u 'openmetadata-svc:Password123!' "http://172.21.101.83:8090\$p"

Done

И на ВМ ad-opa-demo:

docker compose exec airflow-webserver airflow version

docker compose exec airflow-webserver airflow config get-value webserver
base_url

Если сервиса airflow-webserver нет, посмотрите имя через docker compose
ps. В Airflow 3 он обычно называется airflow-apiserver.

**Как интерпретировать результат:**

- **/api/v2/version отвечает 200, /api/v1/... отвечает 404.** Это
  Airflow 3. Тогда для диагностики дальше используем v2:

- docker exec \<ingestion_container\> curl -s -o /dev/null -w
  '%{http_code}\n' \\ -u '\<user\>:\<password\>'
  http://172.21.101.83:8090/api/v2/dags

- 

- Учтите, что в Airflow 3 REST API работает с JWT-токеном, а Basic Auth
  по умолчанию не принимается. Если здесь будет 401 или 403, проверьте
  авторизацию через токен:

- docker exec \<ingestion_container\> curl -s -X POST
  http://172.21.101.83:8090/auth/token \\ -H 'Content-Type:
  application/json' -d
  '{"username":"\<user\>","password":"\<password\>"}'

- 

- **Все пути отвечают 404.** Скорее всего, есть префикс из base_url. Его
  же нужно будет добавить в Host and Port коннектора.

Пришлите вывод цикла и airflow version. От версии зависит, что дальше
чинить: права в OPA или способ аутентификации в коннекторе.

Команда зависает не из-за Airflow: из контейнера ingestion запрос вообще
не доходит до 172.21.101.83. Вероятных причин две. Сначала добавим
таймаут, чтобы ничего не висело, и посмотрим, где именно останавливается
запрос.

**1. Прокси в контейнере** (в закрытом контуре это частая история)

docker exec openmetadata_ingestion env \| grep -i proxy

Если там есть HTTP_PROXY или HTTPS_PROXY, curl отправляет запрос через
корпоративный прокси, а тот до внутреннего IP не достаёт. Повторите
запрос в обход прокси:

docker exec openmetadata_ingestion curl -sv --max-time 5 --noproxy '\*'
\\

-u 'openmetadata-svc:Password123!'
http://172.21.101.83:8090/api/v2/version

Сначала выясним, откуда в контейнере берётся прокси. От этого зависит,
где его исправлять.

**Шаг 1. Посмотреть текущие настройки**

cd ~/openmetadata

docker exec openmetadata_ingestion env \| grep -i proxy

cat ~/.docker/config.json 2\>/dev/null

grep -rni proxy .env docker-compose\*.yml 2\>/dev/null

docker compose config --services

Последняя команда покажет точные имена сервисов. В стандартном compose
OpenMetadata это openmetadata-server, ingestion, postgresql,
elasticsearch.

malkov-d@mcphub-dev-app-03:~/openmetadata\$ docker compose config
--services

WARN\[0000\] /home/malkov-d/openmetadata/docker-compose.yml: the
attribute \`version\` is obsolete, it will be ignored, please remove it
to avoid potential confusion

elasticsearch

postgresql

execute-migrate-all

openmetadata-server

ingestion

Имена сервисов стандартные, а файл называется docker-compose.yml.
Значит, Docker Compose подхватит docker-compose.override.yml
автоматически, указывать -f не нужно. Вариант с override сработает,
откуда бы ни брался прокси. Переменные, заданные явно в compose, имеют
приоритет над теми, что Docker подставляет из ~/.docker/config.json.

**1. Создайте override-файл** (в ~/openmetadata):

cat \> docker-compose.override.yml \<\<'EOF'

x-noproxy: &noproxy

NO_PROXY:
"localhost,127.0.0.1,172.21.0.0/16,postgresql,elasticsearch,openmetadata-server,ingestion,execute-migrate-all"

no_proxy:
"localhost,127.0.0.1,172.21.0.0/16,postgresql,elasticsearch,openmetadata-server,ingestion,execute-migrate-all"

services:

openmetadata-server:

environment: \*noproxy

ingestion:

environment: \*noproxy

EOF

**2. Проверьте, что Compose видит переменные:**

docker compose config \| grep -iA0 no_proxy

Каждая переменная должна появиться по два раза: у openmetadata-server и
у ingestion.

<img src="media/image3.png" style="width:6.5in;height:1.03125in" />

**3. Сделайте бэкап и пересоздайте два контейнера:**

docker compose exec -T -e PGPASSWORD="\$(grep ^DB_USER_PASSWORD .env \|
cut -d= -f2)" ingestion \\

pg_dump -U openmetadata_user -h postgresql -d openmetadata_db \>
backup\_\$(date +%Y%m%d%H%M).sql

Это пароль пользователя openmetadata_user во внутренней PostgreSQL
самого OpenMetadata. Он не относится ни к Airflow, ни к ad-opa-demo.
Файла .env у вас нет, поэтому скрипт не нашёл пароль и pg_dump спросил
его вручную.

**1. Прервите команду:** Ctrl+C. Затем удалите пустой файл бэкапа,
который успел создаться:

rm -f backup\_\*.sql

**2. Найдите пароль в compose-файле:**

grep -nE 'DB_USER_PASSWORD\|DB_USER:\|POSTGRES_PASSWORD'
docker-compose.yml

В стандартном compose OpenMetadata строка обычно выглядит так:
DB_USER_PASSWORD: \${DB_USER_PASSWORD:-openmetadata_password}. Значение
после :- — пароль по умолчанию, то есть openmetadata_password.

<img src="media/image4.png" style="width:6.5in;height:0.94792in" />

Всё стандартно: пользователь openmetadata_user, пароль
openmetadata_password. Запускайте бэкап:

docker compose exec -T -e PGPASSWORD='openmetadata_password' ingestion
\\

pg_dump -U openmetadata_user -h postgresql -d openmetadata_db \>
backup\_\$(date +%Y%m%d%H%M).sql

ls -lh backup\_\*.sql

<img src="media/image5.png" style="width:6.5in;height:0.40625in" />

Если файл получился ненулевого размера, пересоздайте контейнеры и
проверьте:

docker compose up -d openmetadata-server ingestion

docker exec openmetadata_ingestion env \| grep -i no_proxy

docker exec openmetadata_ingestion curl -s --max-time 5 -o /dev/null -w
'%{http_code}\n' \\

-u 'openmetadata-svc:Password123!' http://172.21.101.83:8090/api/v2/dags

Пришлите размер бэкапа и код ответа последней команды.

<img src="media/image6.png" style="width:6.5in;height:1.27083in" />

Ошибка 401

401 с настоящим паролем означает одно из двух: либо Airflow 3 не
принимает Basic Auth на этом endpoint, либо LDAP отклоняет сам пароль.
Разделим эти случаи. Запрос к /api/v2/version ранее вернул 200, но этот
endpoint, скорее всего, открыт без авторизации, так что он ничего не
доказывает.

**1. Проверьте логин через токен** (в Airflow 3 это основной способ
входа в API):

docker exec openmetadata_ingestion curl -s --max-time 5 -X POST \\

http://172.21.101.83:8090/auth/token \\

-H 'Content-Type: application/json' \\

-d '{"username":"openmetadata-svc","password":"Password123!"}'

- **Вернулся access_token:** логин и пароль верны, проблема только в
  Basic Auth. Проверьте доступ к DAG с токеном:

- TOKEN='\<access_token из ответа\>'docker exec openmetadata_ingestion
  curl -s -o /dev/null -w '%{http_code}\n' \\ -H "Authorization: Bearer
  \$TOKEN" http://172.21.101.83:8090/api/v2/dags

TOKEN='eyJhbGciOiJIUzUxMiIsImtpZCI6Im5vdC11c2VkIiwidHlwIjoiSldUIn0.eyJzdWIiOiIyIiwianRpIjoiYWE4ZGM0OWFhZTg0NDQzYmEyZTBjYzcyNTAzMjU3NWIiLCJhdWQiOiJhcGFjaGUtYWlyZmxvdyIsIm5iZiI6MTc5MDc2ODA5MiwiZXhwIjoxNzkwODU0NDkyLCJpYXQiOjE3OTA3NjgwOTJ9.DzYK3auwUkQzsOX4hBF_T0xgVxX1H-7kLu9aYBtwaRrv5j0Zm8HpGRszvxxdvRzJp3_9tTfqhHivon48kCc8WQ'docker
exec openmetadata_ingestion curl -s -o /dev/null -w '%{http_code}\n' \\
-H "Authorization: Bearer \$TOKEN" http://172.21.101.83:8090/api/v2/dags

- 

<img src="media/image7.png" style="width:6.5in;height:0.96875in" />

Склеились команды:

TOKEN=\$(docker exec openmetadata_ingestion curl -s --max-time 5 -X POST
\\

http://172.21.101.83:8090/auth/token -H 'Content-Type: application/json'
\\

-d '{"username":"openmetadata-svc","password":"Password123!"}' \\

\| python3 -c 'import sys,json;
print(json.load(sys.stdin)\["access_token"\])')

docker exec openmetadata_ingestion curl -s -o /dev/null -w
'%{http_code}\n' \\

-H "Authorization: Bearer \$TOKEN" http://172.21.101.83:8090/api/v2/dags

<img src="media/image8.png" style="width:6.5in;height:0.91667in" />

Да, 403 — ожидаемый результат, и он полезнее 401: токен принят, то есть
Airflow знает, кто такой openmetadata-svc. Но OpaFabAuthManager
спрашивает OPA, и OPA отказывает в доступе к DAG. Это та же ситуация,
что и на облачном стенде. Чинится на **ВМ ad-opa-demo** в три шага.

**1. Посмотрите, что сейчас есть**

cd \<каталог ad-opa-demo\>

docker compose ps --services \| grep -iE 'airflow\|opa\|ldap'

grep -n 'ROLE_MAP' -A3 \$(find . -name opa_auth_manager.py)

grep -nE 'metadata_ingestion\|service_roles' \$(find . -name authz.rego)

<img src="media/image9.png" style="width:6.5in;height:1.53125in" />

Картина ясна. Dev-копия ad-opa-demo собрана из более ранней версии, где
нет ни роли metadata_ingestion, ни service_roles в authz.rego. Поэтому
для OPA openmetadata-svc — пользователь без тенанта и без ролей, и он
получает отказ. Добавим все три звена.

**1. ROLE_MAP** (можно делать сразу):

F=\$(find . -name opa_auth_manager.py)

cp "\$F" "\$F.bak"

sed -i 's/"Viewers": "viewer"}/"Viewers": "viewer", "MetadataIngestion":
"metadata_ingestion"}/' "\$F"

grep -n 'ROLE_MAP =' "\$F"

В строке 38 должен появиться "MetadataIngestion": "metadata_ingestion".

<img src="media/image10.png" style="width:6.5in;height:0.90625in" />

Вставка из нескольких строк склеилась, и последний grep остался ждать
закрывающую кавычку (\>), поэтому пришлось нажать Ctrl+C. Непонятно,
успел ли выполниться sed. Давайте проверим, выполняя команды **по
одной**.

F=\$(find . -name opa_auth_manager.py); echo "\$F"

ls -l "\$F" "\$F.bak"

grep -n 'ROLE_MAP =' "\$F"

**В строке 38 уже есть "MetadataIngestion": "metadata_ingestion"** —
правка применилась, ничего повторять не нужно.

**Строка прежняя** — выполните отдельно:

sed -i 's/"Viewers": "viewer"}/"Viewers": "viewer", "MetadataIngestion":
"metadata_ingestion"}/' "\$F"

После этого снова запустите grep.

**Нет файла .bak** — сначала сделайте cp "\$F" "\$F.bak".

Повторный запуск sed не навредит: после первой замены шаблон "viewer"} в
файле больше не встречается.

Затем, тоже по одной команде, покажите authz.rego и группы LDAP:

R=\$(find . -name authz.rego); wc -l "\$R"

cat "\$R"

docker compose exec ldap ldapsearch -x -H ldap://localhost -D
"cn=admin,dc=demo,dc=local" -W -b "dc=demo,dc=local"
"(objectClass=groupOfNames)" dn member

Пришлите вывод grep ROLE_MAP, authz.rego и ldapsearch.

<img src="media/image11.png" style="width:6.5in;height:0.1875in" />

<img src="media/image12.png" style="width:6.5in;height:7.5in" />

<img src="media/image13.png" style="width:6.5in;height:6.46875in" />

grep -nE
'LDAP_ADMIN_PASSWORD\|LDAP_ADMIN\|LDAP_ROOT\|LDAP_DOMAIN\|LDAP_BASE'
docker-compose.yml .env 2\>/dev/null

Всё понятно, у нас есть всё нужное. Роль в ROLE_MAP уже добавлена.
Осталось два шага: сделать группу в LDAP и прописать правило в OPA.
Потом перезапустим сервисы и проверим.

Почему сейчас 403: в authz.rego allow срабатывает только при двух
условиях сразу — rbac_allow и same_tenant. У openmetadata-svc нет ни
ролей, ни тенанта. Добавим отдельное правило для сервисных ролей: оно
даёт только просмотр DAG и не требует совпадения тенанта. Существующие
правила для тенантов остаются как есть.

**1. OPA: добавьте правило в конец authz.rego**

R=\$(find . -name authz.rego)

cp "\$R" "\$R.bak"

cat \>\> "\$R" \<\<'EOF'

\# Сервисные (платформенные) роли: кросс-тенантный доступ только на
чтение.

\# Нужны для каталога OpenMetadata, изоляцию тенантов для людей не
затрагивают.

service_roles := {"metadata_ingestion"}

service_permissions := {

"metadata_ingestion": {

"airflow_dag": {"airflow_dag_view"},

},

}

allow {

some role in input.user.roles

role in service_roles

input.action in service_permissions\[role\]\[input.resource.type\]

}

EOF

tail -20 "\$R"

Несколько правил allow в Rego объединяются по «ИЛИ», поэтому старая
логика для людей не меняется. Синтаксис взят тот же, что в файле (без
if), чтобы подошёл под вашу версию OPA.

Перезапустите OPA и проверьте, что политика загрузилась без ошибок:

docker compose restart opa

docker compose logs --tail 20 opa

В логах не должно быть rego_parse_error или compile error.

**2. LDAP: создайте группу**

Пользователи у вас хранятся как uid=...,ou=people, группы — в ou=groups.
Сначала проверьте, что учётка существует:

docker compose exec ldap ldapsearch -x -H ldap://localhost -D
"cn=admin,dc=demo,dc=local" -W \\

-b "ou=people,dc=demo,dc=local" "(uid=openmetadata-svc)" dn

Если нашлась запись uid=openmetadata-svc,ou=people,dc=demo,dc=local,
создайте группу:

cat \> /tmp/mi-group.ldif \<\<'EOF'

dn: cn=Platform-MetadataIngestion,ou=groups,dc=demo,dc=local

objectClass: groupOfNames

cn: Platform-MetadataIngestion

member: uid=openmetadata-svc,ou=people,dc=demo,dc=local

EOF

docker compose cp /tmp/mi-group.ldif ldap:/tmp/mi-group.ldif

docker compose exec ldap ldapadd -x -H ldap://localhost -D
"cn=admin,dc=demo,dc=local" -W -f /tmp/mi-group.ldif

Как opa_auth_manager.py разберёт имя группы: Platform не входит в
TENANT_MAP, значит, тенанта не будет. MetadataIngestion → роль
metadata_ingestion. Так и задумано.

**3. Пересоздайте Airflow, чтобы подхватилась правка ROLE_MAP**

Сначала проверьте, подключён ли файл томом:

grep -n 'opa_auth_manager' docker-compose.yml

- Если файл есть в volumes: docker compose up -d --force-recreate
  airflow

- Если нет (он вшит в образ): docker compose up -d --build airflow

**4. Проверьте на ВМ OpenMetadata**

Токен нужно получить **заново**:

TOKEN=\$(docker exec openmetadata_ingestion curl -s --max-time 5 -X POST
\\

http://172.21.101.83:8090/auth/token -H 'Content-Type: application/json'
\\

-d '{"username":"openmetadata-svc","password":"Password123!"}' \\

\| python3 -c 'import sys,json;
print(json.load(sys.stdin)\["access_token"\])')

docker exec openmetadata_ingestion curl -s -o /dev/null -w
'%{http_code}\n' \\

-H "Authorization: Bearer \$TOKEN" http://172.21.101.83:8090/api/v2/dags

Должен вернуться **200**. Затем нажмите **Retry Test** в OpenMetadata.

Если снова будет 403, пришлите логи за момент запроса:

docker compose logs --tail 30 airflow \| grep -iE
'\_extract_tenant_and_roles\|opa\|openmetadata-svc'

<img src="media/image14.png" style="width:6.5in;height:0.26042in" />

Там будет видно, какие roles и tenant_id Airflow передаёт в OPA.

Проверка 403:

Пустой вывод grep тоже информативен. Либо лог-строки \_debug не попадают
в docker compose logs, либо до \_extract_tenant_and_roles дело вообще не
доходит. Проверим цепочку по звеньям. Выполняйте команды по одной в
~/atlas.

**1. OPA загрузил новое правило и пропускает сервисную роль?** Спросим
OPA напрямую из контейнера Airflow:

docker compose exec airflow curl -s -X POST
http://opa:8181/v1/data/platform/authz/allow \\

-H 'Content-Type: application/json' \\

-d
'{"input":{"user":{"roles":\["metadata_ingestion"\],"tenant_id":null},"action":"airflow_dag_view","resource":{"type":"airflow_dag","tenant_id":"company_a"}}}'

Ожидаем {"result":true}. Если придёт false или ошибка, дело в политике
или она не перечиталась. Если в контейнере нет curl или у OPA другое имя
хоста или порт, пришлите ошибку.

**2. Группа в LDAP создалась?**

docker compose exec ldap ldapsearch -x -H ldap://localhost -D
"cn=admin,dc=demo,dc=local" -W \\

-b "ou=groups,dc=demo,dc=local" "(cn=Platform-MetadataIngestion)" member

**3. Контейнер Airflow видит правку ROLE_MAP?** Проверяем именно внутри
контейнера, а не на хосте:

docker compose exec airflow sh -c 'grep -rn "ROLE_MAP =" \$(python -c
"import
opa_auth_manager,os;print(os.path.dirname(opa_auth_manager.\_\_file\_\_))"
2\>/dev/null \|\| echo /opt/airflow)'

Если в выводе нет MetadataIngestion, файл вшит в образ, и нужен docker
compose up -d --build airflow.

<img src="media/image15.png" style="width:6.5in;height:2.95833in" />

Нет необходимой роли

На скриншоте главное — строка внизу:

/opt/airflow/custom/opa_auth_manager.py:38:ROLE_MAP = {"Analysts":
"analyst", "Admins": "admin", "Viewers": "viewer"}

Внутри контейнера по-прежнему **старый** ROLE_MAP, без
MetadataIngestion. Поэтому и 403.

Почему так вышло:

- Файл попадает в контейнер через bind-mount, а не через образ. В
  Dockerfile только три шага (FROM, apt, pip), --build взял их из кэша,
  и контейнер вообще не пересоздался: статус остался Running.

- sed -i не редактирует файл на месте. Он пишет новый файл и подменяет
  им старый (новый inode). Если смонтирован **отдельный файл**,
  работающий контейнер продолжает видеть старую версию.

Лечится пересозданием контейнера:

docker compose up -d --force-recreate airflow

Затем проверьте, что в контейнере новый код:

docker compose exec airflow grep -n "ROLE_MAP ="
/opt/airflow/custom/opa_auth_manager.py

В строке должен быть "MetadataIngestion": "metadata_ingestion".

После этого на ВМ OpenMetadata заново получите TOKEN и повторите запрос
к /api/v2/dags.

**4. Откуда opa_auth_manager.py берёт группы?** Это самое важное: если
группы берутся не напрямую из LDAP, а из ролей FAB, то новой LDAP-группы
недостаточно.

sed -n '60,110p' \$(find . -name opa_auth_manager.py)

**5. Последние логи Airflow без фильтра** (сразу после повторного
запроса к /api/v2/dags):

docker compose logs --tail 40 airflow

Пришлите результаты пунктов 1–4. По ним будет видно, какое звено не
срабатывает: OPA, LDAP, код в контейнере или способ получения групп.

Чиним агента postgres

<img src="media/image16.png" style="width:6.5in;height:6.73958in" />

На POC мы это не чинили, а отложили: 8 из 9 проверок проходили, и
предупреждение про query history было ожидаемым. Причина в том, что на
Postgres не включено расширение pg_stat_statements. Метаданные
собираются и без него, но usage и lineage по запросам не строятся.
Включается оно так, на ВМ ad-opa-demo в ~/atlas.

**1. Подключить библиотеку при старте Postgres**

Проверьте текущее значение:

docker compose exec postgres psql -U postgres -c 'show
shared_preload_libraries;'

Если в выводе нет pg_stat_statements, откройте docker-compose.yml и
добавьте сервису postgres строку command. Если command уже есть,
допишите флаги к нему:

postgres:

...

command: postgres -c shared_preload_libraries=pg_stat_statements -c
pg_stat_statements.track=all

Если в shared_preload_libraries уже что-то было, перечислите значения
через запятую.

**2. Перезапустить Postgres**

docker compose up -d --force-recreate postgres

docker compose exec postgres psql -U postgres -c 'show
shared_preload_libraries;'

Airflow, Superset и остальные сервисы, которые ходят в эту БД, на
несколько секунд потеряют соединение. Для dev-стенда это нормально.

**3. Создать расширение и выдать права**

docker compose exec postgres psql -U postgres -d salesdb -c 'CREATE
EXTENSION IF NOT EXISTS pg_stat_statements;'

docker compose exec postgres psql -U postgres -c 'GRANT
pg_read_all_stats TO "openmetadata-ro";'

Расширение создаётся в каждой БД, которую подключаете к каталогу. Сейчас
это salesdb. У роли дефис в имени, поэтому в SQL она в двойных кавычках.

Если суперпользователь у вас называется не postgres, посмотрите имя в
POSTGRES_USER в docker-compose.yml.

**4. Проверить**

docker compose exec -e PGPASSWORD='ВАШ_ПАРОЛЬ' postgres \\

psql -h localhost -U openmetadata-ro -d salesdb -c 'select count(\*)
from pg_stat_statements;'

Команда должна вернуть число, а не ошибку прав. После этого в
OpenMetadata нажмите **Test Connection**: проверок должно стать 9/9.
Затем запустите **Usage** и **Lineage** агенты.

Запросы в pg_stat_statements накапливаются с момента включения, поэтому
lineage появится после того, как DAG хотя бы раз отработают. Для демо
запустите вручную company_a\_\_load_dataset и company_b\_\_load_dataset,
а затем Usage и Lineage агенты в OpenMetadata.

Включаем minIO

Давайте. Делаем по той же схеме, что на POC. MinIO работает через LDAP,
поэтому обычного пользователя в нём завести нельзя. Вместо этого создаём
service account со статическими ключами поверх LDAP-учётки
openmetadata-svc.

Все команды выполняются на ВМ ad-opa-demo в ~/atlas, по одной.

**1. Разведка**

docker compose ps --services \| grep -i minio

docker compose exec minio env \| grep -E
'MINIO_ROOT_USER\|MINIO_IDENTITY_LDAP_SERVER_ADDR'

docker compose exec minio which mc

<img src="media/image17.png" style="width:6.5in;height:0.4375in" />

docker compose exec minio mc --version

<img src="media/image18.png" style="width:6.5in;height:0.54167in" />

Скачать mc в закрытом контуре не получится, поэтому надеемся, что он уже
есть внутри образа MinIO. Если which mc ничего не вывел, напишите —
найдём обходной путь.

**2. Alias под root** (внутри контейнера):

docker compose exec minio sh -c 'mc alias set local
http://localhost:9000 "\$MINIO_ROOT_USER" "\$MINIO_ROOT_PASSWORD"'

**3. Политика на чтение для LDAP-учётки**

docker compose exec minio mc idp ldap policy attach local readonly \\

--user 'uid=openmetadata-svc,ou=people,dc=demo,dc=local'

**4. Service account со статическими ключами**

docker compose exec minio mc admin user svcacct add local \\

'uid=openmetadata-svc,ou=people,dc=demo,dc=local'

<img src="media/image19.png" style="width:6.5in;height:0.29167in" />

MinIO не смог найти эту учётку в LDAP. Дело не в правах, а в том, как
MinIO ищет пользователей: по uid и в указанной ему ветке каталога. Есть
две частые причины.

**1. Попробуйте короткое имя вместо DN.** По короткому имени MinIO сам
ищет DN через свой фильтр:

docker compose exec minio mc admin user svcacct add local
openmetadata-svc

Если сработало, повторите по той же схеме привязку политики:

docker compose exec minio mc idp ldap policy attach local readonly
--user openmetadata-svc

<img src="media/image20.png" style="width:6.5in;height:0.25in" />

docker compose exec minio mc idp ldap policy entities local --user
openmetadata-svc

Отлично, ключи есть. Осталось проверить сеть, собрать манифест и создать
сервис.

**1. Структура бакета** (на ВМ ad-opa-demo):

docker compose exec minio mc ls --recursive local/datasets \| head -30

**2. Доступ к MinIO с ВМ OpenMetadata:**

docker exec openmetadata_ingestion curl -s -o /dev/null -w
'%{http_code}\n' --max-time 5 \\

http://172.21.101.83:9000/minio/health/live

<img src="media/image21.png" style="width:6.5in;height:0.3125in" />

Ожидаем 200.

mc ls local

<img src="media/image22.png" style="width:6.5in;height:1.14583in" />

**3. Манифест openmetadata.json.** Состав зависит от вывода шага 1. Если
структура такая же, как на POC, подойдёт этот вариант:

docker compose exec minio sh -c 'cat \> /tmp/openmetadata.json \<\<EOF

{"entries":\[

{"dataPath":"company_a/raw","structureFormat":"csv","isPartitioned":false},

{"dataPath":"company_b/raw","structureFormat":"csv","isPartitioned":false}

\]}

EOF

mc cp /tmp/openmetadata.json local/datasets/openmetadata.json'

**4. Сервис в OpenMetadata:** Settings → Services → **Storage** → Add →
**S3**.

- **AWS Access Key ID / Secret Access Key** — сохранённые ключи.

- **AWS Region** — us-east-1.

- **Endpoint URL** — http://172.21.101.83:9000.

- **Test Connection**. Предупреждение про CloudWatch ожидаемо, его можно
  игнорировать.

- Сохраните сервис и запустите **Metadata Agent**.

Пришлите вывод шагов 1 и 2. Если структура бакета отличается от POC,
поправлю манифест под неё.

**2. Если не помогло, проверьте, где MinIO ищет пользователей:**

docker compose exec minio env \| grep -E
'MINIO_IDENTITY_LDAP\_(SERVER_ADDR\|USER_DN_SEARCH\|LOOKUP_BIND_DN)'

Смотрим на USER_DN_SEARCH_BASE_DN и USER_DN_SEARCH_FILTER. Для вашего
LDAP должно быть примерно так:

- base DN: ou=people,dc=demo,dc=local (или dc=demo,dc=local);

- фильтр: (uid=%s).

Если base DN указывает на другую ветку или фильтр ищет по cn=%s,
openmetadata-svc там не найдётся.

Заодно проверьте, что учётка действительно лежит там, где мы думаем:

docker compose exec ldap ldapsearch -x -H ldap://localhost -D
"cn=admin,dc=demo,dc=local" -W \\

-b "dc=demo,dc=local" "(uid=openmetadata-svc)" dn

Пришлите результат шага 1. Если он не сработает, пришлите ещё вывод env
из шага 2 и ldapsearch.

Сохраните **Access Key** и **Secret Key** в менеджер паролей. Secret Key
показывается только один раз. В чат ключи не присылайте.

**5. Доступность с ВМ OpenMetadata**

docker exec openmetadata_ingestion curl -s -o /dev/null -w
'%{http_code}\n' --max-time 5 \\

http://172.21.101.83:9000/minio/health/live

Ожидаем 200. Подсеть 172.21.0.0/16 уже есть в NO_PROXY, так что прокси
мешать не должен. Если запрос зависнет, проверьте, что порт 9000
опубликован: docker compose ps minio.

**6. Манифест для бакета**

Урок POC: OpenMetadata каталогизирует только пути, перечисленные в
openmetadata.json. Сначала посмотрите структуру бакета:

docker compose exec minio mc ls --recursive local/datasets \| head -30

Затем создайте манифест. Пути подставьте свои по выводу ls:

docker compose exec minio sh -c 'cat \> /tmp/openmetadata.json \<\<EOF

{"entries":\[

{"dataPath":"company_a/raw","structureFormat":"csv","isPartitioned":false},

{"dataPath":"company_b/raw","structureFormat":"csv","isPartitioned":false}

\]}

EOF

mc cp /tmp/openmetadata.json local/datasets/openmetadata.json'

**7. Сервис в OpenMetadata**

Settings → Services → **Storage** → Add → **S3**:

- **AWS Access Key ID / Secret Access Key** — ключи из шага 4;

- **AWS Region** — us-east-1;

- **Endpoint URL** — http://172.21.101.83:9000;

- **Test Connection**. Предупреждение про CloudWatch GetMetrics
  ожидаемо, потому что MinIO этот API не поддерживает;

- сохраните, затем запустите **Metadata Agent**.

Если на шаге 7 не пройдёт проверка **ListBuckets**, значит, встроенной
политики readonly в этой версии MinIO не хватает. Тогда сделаем свою
политику с s3:ListAllMyBuckets и s3:ListBucket.

Пришлите вывод шагов 1 и 5. По ним будет видно, есть ли mc в контейнере
и открыт ли порт 9000.
