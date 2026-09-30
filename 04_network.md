# Проверка соединения
Раз ping проходит, а соединение падает по timeout, до ВМ вы добираетесь, а до порта 5432 — нет. 

Ошибка пароля или прав выглядела бы по-другому: password authentication failed или no pg_hba.conf entry. Поэтому сейчас ищем сетевую причину.

*Test Connection в OpenMetadata выполняется из контейнера ingestion, а не с хоста. Проверять нужно на трёх уровнях*

### 1. С ВМ OpenMetadata, с хоста:
```
nc -zv -w 5 <IP_ad-opa-demo> 5432
```
- если nc нет:

`timeout 5 bash -c '</dev/tcp/<IP_ad-opa-demo>/5432' && echo OK || echo FAIL`

### 2. С ВМ OpenMetadata, из контейнера ingestion:
```
docker ps --format '{{.Names}}' | grep -i ingest
docker exec -it <ingestion_container> bash -c "timeout 5 bash -c '</dev/tcp/<IP_ad-opa-demo>/5432' && echo OK || echo FAIL"
```

### 3. На ВМ ad-opa-demo:
```
sudo ss -tlnp | grep 5432                              # кто и на каком адресе слушает
docker ps --format '{{.Names}}\t{{.Ports}}' | grep -i postgres   # нужен 0.0.0.0:5432->5432
sudo ufw status; sudo iptables -L INPUT -n | head -20; sudo iptables -L DOCKER-USER -n
```
#### Как читать результаты:
##### 1 = FAIL. 
Порт закрыт между ВМ. Варианты: 
- Postgres опубликован только на 127.0.0.1:5432 или не опубликован вовсе (видно в шаге 3);
- порт режет firewall на ВМ;
- в корпоративной сети ICMP разрешён, а TCP 5432 между сегментами закрыт. Это частый случай, тогда нужна заявка сетевикам. 

##### 1 = OK, 2 = FAIL.
Сеть в порядке, проблема в Docker на ВМ OpenMetadata. 

В закрытом контуре чаще всего подсеть Docker (172.17.x, 172.18.x и т. п.) пересекается с корпоративной подсетью. Тогда трафик из контейнера до <IP_ad-opa-demo> уходит в docker-bridge, а не в сеть. 
_Проверка🩹_ 
- ip routedocker network ls -q | xargs docker network inspect -f '{{.Name}} {{range .IPAM.Config}}{{.Subnet}}{{end}}'
Если IP ВМ ad-opa-demo попадает в одну из этих подсетей, причина найдена. Лечится сменой подсетей Docker через default-address-pools в /etc/docker/daemon.json. 

##### 1 и 2 = OK. 
Проверьте host и port в настройках коннектора: там должен быть внутренний IP, а не имя контейнера или localhost. 

##### Что прислать (IP можно замаскировать, но так, чтобы было видно, в какую подсеть они попадают):
1.	Вывод шагов 1, 2 и 3. 
2.	ip route и список подсетей Docker с ВМ OpenMetadata. 
3.	Текст ошибки из Test Connection (кнопка Copy log). 
4.	При необходимости: docker logs <ingestion_container> --tail 100. 
Пока ищем timeout, есть ещё одна мелочь, которая всплывёт следующей: в имени роли openmetadata-ro есть дефис. В SQL его нужно всегда брать в двойные кавычки (GRANT ... TO "openmetadata-ro"), иначе права выдадутся с ошибкой. В самом коннекторе OpenMetadata имя вводится как есть, без кавычек.
