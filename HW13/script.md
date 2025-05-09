# Redis 2

## Подготовка к занятию

```sh
docker run \
  --name otus-redis-7.2 \
  -v "$(pwd)/redis.conf":/etc/redis/redis.conf \
  -p 6379:6379 \
  -d \
  redis:7.2.5 redis-server /etc/redis/redis.conf
```

```sh
docker start otus-redis-7.2
```

```sh
docker exec -it otus-redis-7.2 redis-cli
```

### IP-адреса серверов

Сервера для репликации — 1 Gb RAM
Сервер для кластера — 2 Gb RAM

- Redis-1: 31.184.253.246
- этот IP нужно использовать в т.ч. в конфиге Sentinel

### Настройка серверов перед занятием

Для сервера Redis 1:

```sh
vim /etc/redis/redis.conf
```

- закомментировать requirepass

```sh
systemctl restart redis
```

## Введение

```sh
redis-cli
```

```Redis
FLUSHDB
```

```Redis
SET user:1:name "Dmitry"
```

```Redis
GET user:1:name
```

```Redis
KEYS *
```

```Redis
SET cache:api "{a: 100, b: true}" EX 15
```

```Redis
TTL cache:api
```

```Redis
GET cache:api
```

## Конфигурация

### Безопасность

```sh
redis-cli -h 127.0.0.1 -p 6379 PING
```

```sh
docker exec \
  -it otus-redis-7.2 \
  redis-cli \
  -h 31.184.253.246 \
  -p 6379 \
  PING
```

```Redis
ACL WHOAMI
```

```Redis
ACL LIST
```

```Redis
ACL SETUSER dmitry on >123456 ~cached:* +get
```

```Redis
ACL LIST
```

```Redis
SET counter 10
```

```Redis
SET cached:counter 20
```

```Redis
AUTH dmitry 123456
```

```Redis
GET counter
```

```Redis
GET cached:counter
```

```Redis
SET cached:counter 25
```

```Redis
ACL SETUSER default ON >1q2w3e4r5t
```

```sh
docker exec \
  -it otus-redis-7.2 \
  redis-cli \
  -h 31.184.253.246 \
  -p 6379 \
  -a 1q2w3e4r5t \
  PING
```

```sh
export REDISCLI_AUTH=1q2w3e4r5t
```

```Redis
CONFIG REWRITE
```

```sh
tail /etc/redis/redis.conf
```

```sh
docker exec \
  -it otus-redis-7.2 \
  redis-cli \
  -h 31.184.253.246 \
  -p 6379 \
  -a 1q2w3e4r5t \
  PING
```

### Журналирование

```sh
vim /var/log/redis/redis-server.log
```

```Redis
MONITOR
```

```sh
docker exec \
  -e REDISCLI_AUTH=1q2w3e4r5t \
  -it otus-redis-7.2 \
  redis-cli \
  -h 31.184.253.246 \
  -p 6379
```

```Redis
SET test:1 10
```

```Redis
INCR test:1
```

```Redis
SLOWLOG GET
```

### Статистика

```Redis
INFO
```

```Redis
INFO STATS
```

```Redis
SET a 10
```

```Redis
GET a
```

```Redis
GET a
```

```Redis
GET abc
```

```Redis
INFO STATS
```

### Конфигурация

```Redis
CONFIG GET *
```

```Redis
CONFIG GET slowlog-log-slower-than
```

```Redis
CONFIG SET slowlog-log-slower-than 5000
```

```Redis
CONFIG REWRITE
```

```sh
vim /etc/redis/redis.conf
```

```
enable-debug-command local
```

### Производительность

```sh
redis-benchmark -a 1q2w3e4r5t -q
```

```sh
redis-cli FLUSHDB
```

### Управление памятью

```Redis
SET counter 10
```

```Redis
KEYS *
```

```Redis
DEBUG OBJECT counter
```

```Redis
GET counter
```

```Redis
DEBUG OBJECT counter
```

```Redis
CONFIG GET hz
```

```Redis
INFO STATS
```

## Персистентность

### RDB (Redis Database)

```Redis
FLUSHDB
```

```Redis
SET twenty 20
```

```Redis
SET twenty-one 021
```

```Redis
SAVE
```

```Redis
BGSAVE
```

```sh
cd /var/lib/redis
```

```sh
vim -b dump.rdb
```

```Redis
INFO STATS
```

```Redis
INFO PERSISTENCE
```

### AOF (Append Only File)

```Redis
SET counter 10
```

```Redis
INCR counter
```

```sh
redis-cli BGREWRITEAOF
```

## Масштабирование

### Репликация

```sh
export REDISCLI_AUTH=1q2w3e4r5t
```

```sh
redis-cli INFO REPLICATION
```

```sh
redis-cli GET counter
```

```sh
vim /etc/redis/redis.conf
```

```
replicaof 31.184.253.246 6379
masterauth 1q2w3e4r5t
```

```sh
redis-cli INFO REPLICATION
```

```sh
redis-cli GET counter
```

```sh
redis-cli MONITOR
```

```Redis
SET c 100
```

```Redis
INCR counter
```

```Redis
SET cache "value" EX 5
```

### Redis Sentinel

```Redis
CONFIG GET min-replicas-to-write
```

```Redis
CONFIG SET min-replicas-to-write 3
```

```Redis
SET abc 100
```

```Redis
CONFIG SET min-replicas-to-write 2
```

```Redis
SET abc 100
```

```Redis
CONFIG SET min-replicas-to-write 0
```

Настройка Sentinel

```sh
vim /etc/redis/sentinel.conf
```

```
daemonize yes
logfile "/var/log/redis/sentinel.log"
sentinel monitor mymaster 31.184.253.246 6379 2
sentinel auth-pass mymaster 1q2w3e4r5t
sentinel down-after-milliseconds mymaster 5000
```

```sh
chown redis:redis /etc/redis/sentinel.conf \
  && touch /var/log/redis/sentinel.log \
  && chown redis:redis /var/log/redis/sentinel.log \
  && chmod 640 /etc/redis/sentinel.conf \
  && chmod 660 /var/log/redis/sentinel.log
```

```sh
ps aux | grep redis
```

```sh
/usr/bin/redis-server /etc/redis/sentinel.conf --sentinel
```

```sh
ps aux | grep redis
```

```sh
tail /var/log/redis/sentinel.log
```

```sh
redis-cli -p 26379
```

```Redis
SENTINEL MASTERS
```

```Redis
SENTINEL SENTINELS mymaster
```

Сценарий отказа: недоступность master

```sh
tail -f /var/log/redis/sentinel.log
```

```sh
systemctl stop redis
```

```sh
redis-cli INFO REPLICATION
```

### Redis Cluster

```sh
docker-compose up -d
```

```sh
docker exec -it root_redis-node-0_1 bash
```

```sh
redis-cli -a bitnami
```

```Redis
SET a 12
```

```Redis
SET b 14
```

```sh
redis-cli -a bitnami -c
```

```Redis
SET a 12
```

```Redis
GET a
```

```Redis
GET b
```

```Redis
SET {user:1}:token "qwerty"
```

```Redis
SET {user:1}:session "wasd"
```

```Redis
SET {user:1}:somekey "otus-rulez"
```

```Redis
KEYS *
```

```sh
redis-cli -a bitnami --cluster call localhost:6379 KEYS "*"
```

```sh
redis-cli -a bitnami -c
```

```Redis
CLUSTER KEYSLOT a
```

```Redis
CLUSTER KEYSLOT b
```

```Redis
CLUSTER KEYSLOT a{test}
```

```Redis
CLUSTER KEYSLOT b{test}
```

## Ссылка на опрос

```
https://otus.ru/polls/119988/
```