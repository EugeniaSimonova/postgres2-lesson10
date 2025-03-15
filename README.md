# Кластер Patroni on-premise
1. Создаём 3 VM для etcd:

````
for i in {1..3}; do yc compute instance create --name etcd$i --hostname etcd$i --cores 2 --memory 2 --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key ~/.ssh/id_rsa.pub & done;
````

2. Создаём 3 VM для PostgreSQL:

````
for i in {1..3}; do yc compute instance create --name pgsql$i --hostname pgsql$i --cores 2 --memory 4 --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key ~/.ssh/id_rsa.pub & done;
````

3. Создаём ВМ для HAProxy:
````
yc compute instance create \
    --name haproxy \
    --hostname haproxy \
    --cores 2 \
    --memory 2 \
    --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
    --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 \
    --ssh-key ~/.ssh/id_rsa.pub;

````

Проверяем, что получилось:

````
yc compute instance list

+----------------------+---------+---------------+---------+----------------+-------------+
|          ID          |  NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP |
+----------------------+---------+---------------+---------+----------------+-------------+
| fhm3c4r8dfv1m0pva3no | etcd3   | ru-central1-a | RUNNING | 51.250.7.197   | 10.0.0.12   |
| fhm40k90pkg097360tab | etcd2   | ru-central1-a | RUNNING | 51.250.4.181   | 10.0.0.6    |
| fhm4o73k7epam2795ip4 | etcd1   | ru-central1-a | RUNNING | 51.250.11.84   | 10.0.0.31   |
| fhmakj4gagt968obvkfh | pgsql1  | ru-central1-a | RUNNING | 51.250.93.67   | 10.0.0.24   |
| fhmc1gsbg49sc3stv7o2 | pgsql3  | ru-central1-a | RUNNING | 158.160.63.165 | 10.0.0.9    |
| fhmo45sot389p64h9htn | pgsql2  | ru-central1-a | RUNNING | 89.169.142.222 | 10.0.0.11   |
| fhmtarm26kcjh4dfm5t8 | haproxy | ru-central1-a | RUNNING | 89.169.132.240 | 10.0.0.39   |
+----------------------+---------+---------------+---------+----------------+-------------+

````

4. Прописываем хосты в каждой VM в файл _hosts_:

````
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address <<EOF
sudo bash -c 'cat >> /etc/hosts <<EOL
10.0.0.31 etcd1
10.0.0.6 etcd2
10.0.0.12 etcd3
10.0.0.24 pgsql1
10.0.0.11 pgsql2
10.0.0.9 pgsql3
10.0.0.39 haproxy
EOL
'
EOF
done;

for i in {1..3}; do vm_ip_address=$(yc compute instance show --name etcd$i | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address <<EOF
sudo bash -c 'cat >> /etc/hosts <<EOL
10.0.0.31 etcd1
10.0.0.6 etcd2
10.0.0.12 etcd3
10.0.0.24 pgsql1
10.0.0.11 pgsql2
10.0.0.9 pgsql3
10.0.0.39 haproxy
EOL
'
EOF
done;

````
На хосте haproxy редактируем файл, добавляем те же хосты:
````
vm_ip_address=$(yc compute instance show --name haproxy | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address

sudo nano /etc/hosts
````

5. Установка etcd на 3 VM:

````
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name etcd$i | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address 'sudo apt update && sudo apt upgrade -y && sudo apt install -y etcd' & done;

````
Заходим на первую ноду `etcd1` и проверяем версию:

````

vm_ip_address=$(yc compute instance show --name etcd1 | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address

yc-user@etcd1:~$ etcd --version
etcd Version: 3.2.26
Git SHA: Not provided (use ./build instead of go build)
Go Version: go1.13.8
Go OS/Arch: linux/amd64

````

6. Конфигурируем и запускаем etcd:

````
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name etcd$i | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address <<EOF
sudo bash -c 'cat >> /etc/default/etcd <<EOL
ETCD_NAME="etcd$i"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://etcd$i:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://etcd$i:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd_claster"
ETCD_INITIAL_CLUSTER="etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ENABLE_V2="true"
EOL
'
EOF
done;
;


for i in {1..3}; do vm_ip_address=$(yc compute instance show --name etcd$i | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address 'ETCDCRL_API=2 && echo $ETCDCRL_API' & done;

for i in {1..3}; do vm_ip_address=$(yc compute instance show --name etcd$i | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address 'sudo systemctl start etcd' & done;


````

7. Перейдём на _etcd1_ и посмотрим member list и состояние кластера:

````
vm_ip_address=$(yc compute instance show --name etcd1 | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address

yc-user@etcd1:~$ etcdctl member list

25d0bfa2dcffa2d7: name=etcd1 peerURLs=http://etcd1:2380 clientURLs=http://etcd1:2379 isLeader=false
875a9230d9ea0259: name=etcd2 peerURLs=http://etcd2:2380 clientURLs=http://etcd2:2379 isLeader=false
df0d680875ee2e00: name=etcd3 peerURLs=http://etcd3:2380 clientURLs=http://etcd3:2379 isLeader=true

yc-user@etcd1:~$ etcdctl cluster-health
member 25d0bfa2dcffa2d7 is healthy: got healthy result from http://etcd1:2379
member 875a9230d9ea0259 is healthy: got healthy result from http://etcd2:2379
member df0d680875ee2e00 is healthy: got healthy result from http://etcd3:2379
cluster is healthy

````


8. Ставим PostgreSQL на 3 VM и проверяем, что он стартовал в выводе второй команды `status` всех нод должен быть `online`:


````
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address 'sudo apt update && sudo apt upgrade -y -q && echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | sudo tee -a /etc/apt/sources.list.d/pgdg.list && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-14' & done;

for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address 'hostname; pg_lsclusters' & done;

````

9. Ставим Patroni:

````
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address 'sudo apt install -y python3 python3-pip git mc && sudo pip3 install psycopg2-binary && sudo systemctl stop postgresql@14-main && sudo -u postgres pg_dropcluster 14 main && sudo pip3 install patroni[etcd] && sudo ln -s /usr/local/bin/patroni /bin/patroni' & done;

````

Проверяем - переходим на первую ноду patroni и проверяем версию:


````
vm_ip_address=$(yc compute instance show --name pgsql1 | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address

yc-user@pgsql1:~$ /usr/local/bin/patroni --version
patroni 4.0.5

```` 

10. Конфигурируем patroni (_/etc/systemd/system/patroni.service_):


````
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address 'cat > temp.cfg << EOF 
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target
[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no
[Install]
WantedBy=multi-user.target
EOF
cat temp.cfg | sudo tee -a /etc/systemd/system/patroni.service
' & done;

````

11. На всех нодах pgsql создем папку _/mnt/patroni_ для хранения файлов и назначаем ей нужные права:


````
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address 'sudo systemctl daemon-reload && sudo apt install etcd-client && sudo mkdir /mnt/patroni && sudo chown postgres:postgres /mnt/patroni && sudo chmod 700 /mnt/patroni' & done;

````


12. Настраиваем _patroni.yml_:

````
for i in {1..3}; do vm_ip_address=$(yc compute instance show --name pgsql$i | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address 'cat > temp2.cfg << EOF 
scope: patroni
name: $(hostname)
restapi:
  listen: $(hostname -I | tr -d " "):8008
  connect_address: $(hostname -I | tr -d " "):8008
etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
  initdb: 
  - encoding: UTF8
  - data-checksums
  pg_hba: 
  - host replication replicator 10.0.0.0/24 md5
  - host all all 10.0.0.0/24 md5
  users:
    admin:
      password: admin_321
      options:
        - createrole
        - createdb
postgresql:
  listen: 127.0.0.1, $(hostname -I | tr -d " "):5432
  connect_address: $(hostname -I | tr -d " "):5432
  data_dir: /var/lib/postgresql/14/main
  bin_dir: /usr/lib/postgresql/14/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: rep-pass_321
    superuser:
      username: postgres
      password: postgres
    rewind:  
      username: rewind_user
      password: rewind_password_321
  parameters:
    unix_socket_directories: '.'
tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
EOF
cat temp2.cfg | sudo tee -a /etc/patroni.yml
' & done;

````

Проверяем содержимое файла _patroni.yml_ на ноде psql1:

````
vm_ip_address=$(yc compute instance show --name pgsql1 | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address

sudo nano /etc/patroni.yml

````

Стартуем кластер patroni по одной ноде и проверяем результат:


````
vm_ip_address=$(yc compute instance show --name pgsql1 | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address

yc-user@pgsql1:~$ sudo systemctl enable patroni && sudo systemctl start patroni 
Created symlink /etc/systemd/system/multi-user.target.wants/patroni.service → /etc/systemd/system/patroni.service.

yc-user@pgsql1:~$ sudo patronictl -c /etc/patroni.yml list
+ Cluster: patroni (7482143785088512907) ----+-----------+
| Member | Host      | Role   | State   | TL | Lag in MB |
+--------+-----------+--------+---------+----+-----------+
| pgsql1 | 10.0.0.24 | Leader | running |  1 |           |
+--------+-----------+--------+---------+----+-----------+


vm_ip_address=$(yc compute instance show --name pgsql2 | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address

yc-user@pgsql2:~$ sudo systemctl enable patroni && sudo systemctl start patroni
Created symlink /etc/systemd/system/multi-user.target.wants/patroni.service → /etc/systemd/system/patroni.service.

yc-user@pgsql2:~$ sudo patronictl -c /etc/patroni.yml list 
+ Cluster: patroni (7482143785088512907) --+----+-----------+
| Member | Host      | Role    | State     | TL | Lag in MB |
+--------+-----------+---------+-----------+----+-----------+
| pgsql1 | 10.0.0.24 | Leader  | running   |  1 |           |
| pgsql2 | 10.0.0.11 | Replica | streaming |  1 |         0 |
+--------+-----------+---------+-----------+----+-----------+

vm_ip_address=$(yc compute instance show --name pgsql3 | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address

yc-user@pgsql3:~$ sudo systemctl enable patroni && sudo systemctl start patroni 
Created symlink /etc/systemd/system/multi-user.target.wants/patroni.service → /etc/systemd/system/patroni.service.

yc-user@pgsql3:~$ sudo patronictl -c /etc/patroni.yml list
+ Cluster: patroni (7482143785088512907) --+----+-----------+
| Member | Host      | Role    | State     | TL | Lag in MB |
+--------+-----------+---------+-----------+----+-----------+
| pgsql1 | 10.0.0.24 | Leader  | running   |  1 |           |
| pgsql2 | 10.0.0.11 | Replica | streaming |  1 |         0 |
| pgsql3 | 10.0.0.9  | Replica | running   |  1 |         0 |
+--------+-----------+---------+-----------+----+-----------+
````
Убедимся, что все реплики имеют верный статус:

````
yc-user@pgsql3:~$ sudo patronictl -c /etc/patroni.yml list
+ Cluster: patroni (7482143785088512907) --+----+-----------+
| Member | Host      | Role    | State     | TL | Lag in MB |
+--------+-----------+---------+-----------+----+-----------+
| pgsql1 | 10.0.0.24 | Leader  | running   |  1 |           |
| pgsql2 | 10.0.0.11 | Replica | streaming |  1 |         0 |
| pgsql3 | 10.0.0.9  | Replica | streaming |  1 |         0 |
+--------+-----------+---------+-----------+----+-----------+

````

13. Устанавливаем HAproxy на ноду `haproxy`:

````
vm_ip_address=$(yc compute instance show --name haproxy | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address

yc-user@haproxy:~$ sudo apt-get update && sudo apt-get install -y haproxy

````

14. Правим _haproxy.cfg_:

````
sudo nano /etc/haproxy/haproxy.cfg

global
        log /dev/log local0
        log /dev/log local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

        # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE->
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
        log global
        option tcplog
        option dontlognull
        timeout connect 5000
        timeout client 50000
        timeout server 50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http

        frontend postgres_frontend
        bind *:5432
        mode tcp
        default_backend postgres_backend

        backend postgres_backend
        mode tcp
        balance roundrobin
        option tcp-check
        default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
        server pgsql1 10.0.0.24:5432 check
        server pgsql2 10.0.0.11:5432 check
        server pgsql3 10.0.0.9:5432 check

        listen stats
        bind *:8404
        mode http
        stats enable
        stats uri /stats
        stats refresh 10s
        stats auth admin:admin_pass
````

15. рестартуем HAProxy и проверяем его статус:


````
yc-user@haproxy:~$ sudo systemctl restart haproxy
yc-user@haproxy:~$ sudo systemctl status haproxy
● haproxy.service - HAProxy Load Balancer
     Loaded: loaded (/lib/systemd/system/haproxy.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2025-03-15 21:47:44 UTC; 12s ago
       Docs: man:haproxy(1)
             file:/usr/share/doc/haproxy/configuration.txt.gz
    Process: 2354 ExecStartPre=/usr/sbin/haproxy -Ws -f $CONFIG -c -q $EXTRAOPTS (code=exited, status=0/SUCCESS)
   Main PID: 2366 (haproxy)
      Tasks: 3 (limit: 2290)
     Memory: 34.6M
     CGroup: /system.slice/haproxy.service
             ├─2366 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock
             └─2369 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock

Mar 15 21:47:43 haproxy systemd[1]: Starting HAProxy Load Balancer...
Mar 15 21:47:44 haproxy haproxy[2366]: Proxy postgres_frontend started.
Mar 15 21:47:44 haproxy haproxy[2366]: Proxy postgres_frontend started.
Mar 15 21:47:44 haproxy haproxy[2366]: Proxy postgres_backend started.
Mar 15 21:47:44 haproxy haproxy[2366]: Proxy postgres_backend started.
Mar 15 21:47:44 haproxy haproxy[2366]: Proxy stats started.
Mar 15 21:47:44 haproxy haproxy[2366]: Proxy stats started.
Mar 15 21:47:44 haproxy haproxy[2366]: [NOTICE] 073/214744 (2366) : New worker #1 (2369) forked
Mar 15 21:47:44 haproxy systemd[1]: Started HAProxy Load Balancer.

````

17. Заходим на другую ноду и пытаемся приконектится на хосте haproxy к PostgreSQL:


````
yc-user@pgsql3:~$ psql -h 10.0.0.39 -p 5432 -U postgres -W
Password: 
psql (14.17 (Ubuntu 14.17-1.pgdg20.04+1))
Type "help" for help.

postgres=# 

````

У нас получилось.


18. Прверяем отказоустойчивость. 

Останавливаем лидер и одну из реплик:

````
yc-user@pgsql3:~$ sudo patronictl -c /etc/patroni.yml list 

+ Cluster: patroni (7482143785088512907) --+----+-----------+
| Member | Host      | Role    | State     | TL | Lag in MB |
+--------+-----------+---------+-----------+----+-----------+
| pgsql1 | 10.0.0.24 | Leader  | running   |  1 |           |
| pgsql2 | 10.0.0.11 | Replica | streaming |  1 |         0 |
| pgsql3 | 10.0.0.9  | Replica | streaming |  1 |         0 |
+--------+-----------+---------+-----------+----+-----------+

yc-user@pgsql3:~$ sudo systemctl stop patroni


vm_ip_address=$(yc compute instance show --name pgsql1 | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address


yc-user@pgsql1:~$ sudo systemctl stop patroni
yc-user@pgsql1:~$ sudo patronictl -c /etc/patroni.yml list
+ Cluster: patroni (7482143785088512907) +----+-----------+
| Member | Host      | Role    | State   | TL | Lag in MB |
+--------+-----------+---------+---------+----+-----------+
| pgsql1 | 10.0.0.24 | Replica | stopped |    |   unknown |
| pgsql2 | 10.0.0.11 | Leader  | running |  2 |           |
+--------+-----------+---------+---------+----+-----------+

````
Лидером стал pgsql2, последняя оставшаяся нода.

Подключаемся к хосту haproxy:


````
psql -h 10.0.0.39 -p 5432 -U postgres -W
postgres=# 

````

Корректная настройка Pgbouncer не была показана во время вебинара, также я не нашла её в чате группы, также в ДЗ ничего не говорилось про PgBouncer, поэтому я не стала его устанавливать.
