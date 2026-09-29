# Параметры

- 4 vCPU/8GB/50GB, 
- порты 22+8585 без лишнего наружу.

# Начало
## key
ssh-keygen -t ed25519 -f C:\Users\malkov.d\.ssh\vm-omdkey -N "" -C "windows-dev-nopass"
```
icacls "C:\Users\malkov.d\.ssh\vm-omdkey" /inheritance:r /grant malkov.d:R
icacls "C:\Users\malkov.d\.ssh\vm-omdkey.pub" /inheritance:r /grant malkov.d:R
```
```
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```
На компьютере (не на ВМ) забрать ключ командой ниже и проставить на ВМ
```
 Get-Content C:\Users\malkov.d\.ssh\vm-omdkey.pub | Set-Clipboard
```
```
chmod 600 ~/.ssh/authorized_keys
chown -R $USER:$USER ~/.ssh
sudo LANG=C LC_ALL=C ufw enable
password for malkov-d:
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
sudo LANG=C LC_ALL=C ufw allow 22/tcp
sudo LANG=C LC_ALL=C ufw allow 8080/tcp
sudo LANG=C LC_ALL=C ufw allow 8585/tcp
ss -tlnp
```


```
cd /etc/apt
/etc/apt$ nano sources.list
убрать sberworks и поставить ubuntu...
После сделать apt update или что-то в этом роде и обновить все
```

# App
Дальше, когда VM поднимется и станет доступна по SSH, следующие шаги:

1. Подключиться и поставить Docker + Docker Compose (если ещё нет на образе Ubuntu 24.04) — стандартно через apt или get-docker.sh.
2. Добавить swap — мы договаривались, что на этой VM (в отличие от демо-стенда) swap стоит включить, раз JVM-стек чувствителен к скачкам памяти:
__________________________________________________________________
bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
___________________________________________________________________

3. Развернуть OpenMetadata через их официальный docker-compose (репозиторий open-metadata/OpenMetadata, ветка под нужную версию).
4. Открыть на демо-стенде доступ с IP 176.123.165.177 к портам Postgres (5432), ClickHouse (8123), Airflow (8090) — это то, что мы обсуждали как обязательный встречный шаг в security group демо-стенда.

5. Выполняй по порядку на новой VM (`ssh user1@176.123.165.177`).

## 1. Обновление системы и установка Docker

```bash
sudo apt update && sudo apt upgrade -y

# Установка Docker по официальному скрипту
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Добавить пользователя в группу docker, чтобы не писать sudo каждый раз
sudo usermod -aG docker $USER
newgrp docker

# Проверка
docker --version
docker compose version
```

## 2. Настройка swap (обязательно для этой VM — обсуждали из-за JVM-стека)

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Проверка
free -h
```

## 3. Разворачивание OpenMetadata

Официальный путь — через их docker-compose из релиза:

```bash
mkdir -p ~/openmetadata && cd ~/openmetadata

# Скачиваем docker-compose для последней стабильной версии (Postgres как backend, не MySQL)
curl -sL -o docker-compose.yml \
  https://raw.githubusercontent.com/open-metadata/OpenMetadata/main/docker/development/docker-compose-postgres.yml

docker compose -f docker-compose.yml up -d
```

Это поднимет: сам OpenMetadata server, Elasticsearch/OpenSearch, Postgres под метаданные и встроенный Airflow-ingestion.

Первый старт займёт несколько минут (Elasticsearch и Airflow долго инициализируются) — проверяй статус:
```bash
docker compose -f docker-compose.yml ps
docker compose -f docker-compose.yml logs -f openmetadata_server
```

Когда все контейнеры в статусе `healthy`/`running` — открывай `http://176.123.165.177:8585` (дефолтный логин: `admin@open-metadata.org` / `admin`).

Погнали. Дам всё по шагам — выполняй по порядку на новой VM (`ssh user1@176.123.165.177`).

## 1. Обновление системы и установка Docker

```bash
sudo apt update && sudo apt upgrade -y

# Установка Docker по официальному скрипту
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Добавить пользователя в группу docker, чтобы не писать sudo каждый раз
sudo usermod -aG docker $USER
newgrp docker

# Проверка
docker --version
docker compose version
```

## 2. Настройка swap (обязательно для этой VM — обсуждали из-за JVM-стека)

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Проверка
free -h
```

## 3. Разворачивание OpenMetadata

Официальный путь — через их docker-compose из релиза:

```bash
mkdir -p ~/openmetadata && cd ~/openmetadata

# Скачиваем docker-compose для последней стабильной версии (Postgres как backend, не MySQL)
curl -sL -o docker-compose.yml \
  https://raw.githubusercontent.com/open-metadata/OpenMetadata/main/docker/development/docker-compose-postgres.yml

docker compose -f docker-compose.yml up -d
```

Это поднимет: сам OpenMetadata server, Elasticsearch/OpenSearch, Postgres под метаданные и встроенный Airflow-ingestion.

Первый старт займёт несколько минут (Elasticsearch и Airflow долго инициализируются) — проверяй статус:
```bash
docker compose -f docker-compose.yml ps
docker compose -f docker-compose.yml logs -f openmetadata_server
```

Когда все контейнеры в статусе `healthy`/`running` — открывай `http://176.123.165.177:8585` (дефолтный логин: `admin@open-metadata.org` / `admin`).

Docker установился успешно (v29.8.1, Compose v5.5.1), несмотря на предупреждения — они не критичны.


**Дальше — добавляем пользователя в группу docker:**

```bash
sudo usermod -aG docker $USER
newgrp docker

# Проверка, что можно запускать без sudo
docker ps
```

Если `docker ps` отработает без ошибок прав — двигаемся к шагу 2 (swap):

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

free -h
```

Скинь вывод обеих команд (`docker ps` и `free -h`), и перейдём к разворачиванию самого OpenMetadata.
