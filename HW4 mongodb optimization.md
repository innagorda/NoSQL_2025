# MongoDB

## Задание

Необходимо:

- построить шардированный кластер из 3 кластерных нод( по 3 инстанса с репликацией) и с кластером конфига(3 инстанса);
- добавить балансировку, нагрузить данными, выбрать хороший ключ шардирования, 
- посмотреть как данные перебалансируются между шардами;
- поронять разные инстансы, посмотреть, что будет происходить, поднять обратно. Описать что произошло.
- настроить аутентификацию и многоролевой доступ;


## Установка

### 1. Заполнение docker-compose.yml 
[Файл docker-compose.yml](./HW4/docker-compose.yml )

### 2. Запуск docker compose up -d

![run_container](pictures/4_1_run_rs.png)

В докере можно посмотреть состояние кластера
![rs_state](pictures/4_2_rs_state.png)

### 3. Инициализация Config Servers:
```bash

    docker exec -it cfg1 mongosh --port 27019 --eval "
rs.initiate({
  _id: 'cfgrs',
  configsvr: true,
  members: [
    { _id: 0, host: 'cfg1:27019' },
    { _id: 1, host: 'cfg2:27019' },
    { _id: 2, host: 'cfg3:27019' }
  ]
})"
    
```
### 4. Шарды (для каждого Replica Set):

Приоритеты:
    priority: 2 — главная нода (primary).
    priority: 1 — вторичная нода (может стать primary).
    priority: 0 — нода только для чтения (не может стать primary).


```bash

docker exec -it shard1-rs1 mongosh --port 27018 --eval "
rs.initiate({
  _id: 'shard1rs',
  members: [
    { _id: 0, host: 'shard1-rs1:27018', priority: 2 },
    { _id: 1, host: 'shard1-rs2:27018', priority: 1 },
    { _id: 2, host: 'shard1-rs3:27018', priority: 0 }
  ]
})"

docker exec -it shard2-rs1 mongosh --port 27018 --eval "
rs.initiate({
  _id: 'shard2rs',
  members: [
    { _id: 0, host: 'shard2-rs1:27018', priority: 2 },
    { _id: 1, host: 'shard2-rs2:27018', priority: 1 },
    { _id: 2, host: 'shard2-rs3:27018', priority: 0 }
  ]
})"

docker exec -it shard3-rs1 mongosh --port 27018 --eval "
rs.initiate({
  _id: 'shard3rs',
  members: [
    { _id: 0, host: 'shard3-rs1:27018', priority: 2 },
    { _id: 1, host: 'shard3-rs2:27018', priority: 1 },
    { _id: 2, host: 'shard3-rs3:27018', priority: 0 }
  ]
})"
```

### 5. Добавление шардов в кластер через Mongos:
```bash

    docker exec -it mongos mongosh --port 27017 --eval "
      sh.addShard('shard1rs/shard1-rs1:27018,shard1-rs2:27018,shard1-rs3:27018');
      sh.addShard('shard2rs/shard2-rs1:27018,shard2-rs2:27018,shard2-rs3:27018');
      sh.addShard('shard3rs/shard3-rs1:27018,shard3-rs2:27018,shard3-rs3:27018');
    "
```

Статус:
```bash
docker exec -it mongos mongosh --port 27017 --eval "sh.status()"
```
![shard_state](pictures/4_3_shard_state.png)

### 6. Балансировка

На уровне клиентского драйвера
```mongodb://localhost:27017/?readPreference=secondaryPreferred```

### 7. Загрузка данных
![import_data](pictures/4_4_import_data.png)

### 8. Ключ шардирования и сам процесс

Проверка шардирования коллекции. Вывод пустой, значит, шардирования нет.
```javascript
use config
db.collections.find({ _id: "sample_db.companies" })
```

![check_col](pictures/4_5_check_col.png)


Включение шардирования для базы

```sh.enableSharding("sample_db")```

Создание индекса по ключу шардирования

```db.companies.createIndex({ permalink: 1 })```
![create_index](pictures/4_6_index.png)


Проверка, что коллекция сейчас не шардирована, возвращает null, это ок

```javascript
use config
db.collections.findOne({ _id: "sample_db.companies" })
```

Проверка на каком шарде находятся данные

```javascript
use sample_db
db.companies.getShardDistribution()
```
![check](pictures/4_7_check.png)

Коллекция слишком маленькая, нужно сократить размер чанка
```javascript
use config
db.settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 10 } },  // Размер в MB
  { upsert: true }
)
```

Проверка балансировки - общий статус 
![check_status](pictures/4_8_companies.png)

Статистика по распределению данных
![distrib](pictures/4_9_distrib.png)



### 8. Поронять разные инстансы, посмотреть, что будет происходить, поднять обратно. Описать что произошло.

#### Тестирование отказоустойчивости отдельных нод в Replica Set

Определение текущего Primary для каждого шарда
```javascript
// Для shard1rs
mongosh --host shard1-rs1 --port 27018 --eval "rs.isMaster().primary"

// Для shard2rs
mongosh --host shard2-rs2 --port 27018 --eval "rs.isMaster().primary"

// Для shard3rs
mongosh --host shard3-rs1 --port 27018 --eval "rs.isMaster().primary"
```

![4_test_1_master](pictures/4_test_1_master.png)

Остановка Primary ноды ```docker stop shard2-rs1```

Primary перевыбралась
![4_test_2_primary](pictures/4_test_2_primary.png)


Подключаемся, чтобы проверить
```mongosh "mongodb://shard2-rs2:27018"```

```javascript
rs.status().members.forEach(m => {
  printjson({
    name: m.name,
    stateStr: m.stateStr,
    health: m.health,
    lastHeartbeat: new Date(m.lastHeartbeat)
  })
})
```
![4_test_3_not_reachable](pictures/4_test_3_not_reachable.png)

Включаем ноду обратно ```docker start shard2-rs1```

![4_test_4_node_primary](pictures/4_test_4_node_primary.png)

#### Тестирование отказа шарда 

Остановка всех нод shard2rs ```docker stop shard2-rs1 shard2-rs2 shard2-rs3```

Проверка состояния кластера ```h.status();``` выдает все шарды в состоянии 1

Проверка посчитать ```db.companies.countDocuments();```
MongoServerError[FailedToSatisfyReadPreference]: Could not find host matching read preference { mode: "secondaryPreferred" } for set shard2rs

Попытка записи прошла успешно - "Write succeeded"
![4_Write_succeeded](pictures/4_Write_succeeded.png)

```javascript
mongosh sample_db --eval "
  try {
    db.companies.insertOne({ name: 'Test Company', permalink: 'test-company-' + new Date().getTime() });
    print('Write succeeded');
  } catch(e) { print('Write failed:', e); }
"
```
Проверка распределения не проходит ```db.companies.getShardDistribution()```
MongoServerError[FailedToSatisfyReadPreference]: Could not find host matching read preference { mode: "secondaryPreferred" } for set shard2rs


Восстановление шарда ```docker start shard2-rs1 shard2-rs2 shard2-rs3```

Количество посчиталось - 9502 
```db.companies.countDocuments();```
Проверка распределения теперь прошла ```db.companies.getShardDistribution()```
{
  data: '34.78MiB',
  docs: 9502,
  chunks: 2,
  'Shard shard3rs': [
    '71.08 % data',
    '71.26 % docs in cluster',
    '3KiB avg obj size on shard'
  ],
  'Shard shard2rs': [
    '28.91 % data',
    '28.73 % docs in cluster',
    '3KiB avg obj size on shard'
  ]
}

При потере Primary ноды:
    Автоматический перевыбор нового Primary в течение 10-30 секунд
    Кратковременные ошибки записи во время выборов
    Чтение продолжает работать с Secondary (если настроено readPreference)

При полном отказе шарда:
    Кластер отмечает шард как недоступный
    Запросы к данным на этом шарде будут завершаться ошибкой
    Данные на других шардах остаются доступными

### 9. Настроить аутентификацию и многоролевой доступ

Создание ключа в директории с файлом docker-compose.yml
```openssl rand -base64 756 > mongo_keyfile && chmod 400 mongo_keyfile```

docker exec -it cfg1 mongosh --port 27019

# Инициализация реплика-сета
rs.initiate({
  _id: "cfgrs",
  configsvr: true,
  members: [
    { _id: 0, host: "cfg1:27019" },
    { _id: 1, host: "cfg2:27019" },
    { _id: 2, host: "cfg3:27019" }
  ]
})

# Создание администратора
use admin
db.createUser({
  user: "admin",
  pwd: "securepassword",
  roles: [ { role: "root", db: "admin" } ]
})

Инициализация replica set + создание пользователя
![4_auth_1](pictures/4_auth_1.png)



docker exec -it cfg1 mongosh --port 27018 --eval "db.getSiblingDB('admin').getUsers()"
```bash
docker exec -it shard1-rs1 mongosh --port 27018 --eval "
rs.initiate({
  _id: 'shard1rs',
  members: [
    { _id: 0, host: 'shard1-rs1:27018', priority: 2 },
    { _id: 1, host: 'shard1-rs2:27018', priority: 1 },
    { _id: 2, host: 'shard1-rs3:27018', priority: 0 }
  ]
})"

docker exec -it shard1-rs1 mongosh --port 27018 --db admin --eval "
db.createUser({
  user: 'admin',
  pwd: 'securepassword',
  roles: [ { role: 'root', db: 'admin' } ]
})"


docker exec -it shard2-rs1 mongosh --port 27018 --eval "
rs.initiate({
  _id: 'shard2rs',
  members: [
    { _id: 0, host: 'shard2-rs1:27018', priority: 2 },
    { _id: 1, host: 'shard2-rs2:27018', priority: 1 },
    { _id: 2, host: 'shard2-rs3:27018', priority: 0 }
  ]
})"
docker exec -it shard1-rs1 mongosh --port 27018 --db admin --eval "
db.createUser({
  user: 'admin',
  pwd: 'securepassword',
  roles: [ { role: 'root', db: 'admin' } ]
})"

docker exec -it shard3-rs1 mongosh --port 27018 --eval "
rs.initiate({
  _id: 'shard3rs',
  members: [
    { _id: 0, host: 'shard3-rs1:27018', priority: 2 },
    { _id: 1, host: 'shard3-rs2:27018', priority: 1 },
    { _id: 2, host: 'shard3-rs3:27018', priority: 0 }
  ]
})"
docker exec -it shard1-rs1 mongosh --port 27018 --db admin --eval "
db.createUser({
  user: 'admin',
  pwd: 'securepassword',
  roles: [ { role: 'root', db: 'admin' } ]
})"
```

Хотелось сделать так, чтобы УЗ пробросились с роутера, но не получилось. Требуется больше автоматизации, т.к. ручное поднятие каждый раз - не эффективная история. Несмотря на то, что диски есть, инициализацию всего реплика-сета приходится запускать каждый раз.


## P.S. Полезные команды
Приостановка балансировки

```sh.stopBalancer()```

Запуск для конкретной коллекции

```sh.enableBalancing("mydb.mycollection")```


Для соединения с монго в докере
``` bash
echo "127.0.0.1 mongo1" | sudo tee -a /etc/hosts
echo "127.0.0.1 mongo2" | sudo tee -a /etc/hosts
echo "127.0.0.1 mongo3" | sudo tee -a /etc/hosts
```
echo -e "127.0.0.1 mongo-config-server-1 mongo-config-server-2 mongo-config-server-3
127.0.0.1 mongo-shard1-rs1 mongo-shard1-rs2 mongo-shard1-rs3
127.0.0.1 mongo-shard2-rs1 mongo-shard2-rs2 mongo-shard2-rs3
127.0.0.1 mongo-shard3-rs1 mongo-shard3-rs2 mongo-shard3-rs3
127.0.0.1 mongos-router" | sudo tee -a /etc/hosts

Для прямого соединения с бд указывать параметр directConnection

```mongosh "mongodb://localhost:27017/?readPreference=secondaryPreferred&directConnection=true"```

Команда для подключения должна быть в кавычках, внутри перечисление хостов

```mongosh "mongodb://localhost:30001,localhost:30002,localhost:30003/?replicaSet=myReplicaSet"```

Мониторинг процесса балансировки
```javascript

// Проверить активные миграции
use config
db.locks.find({ _id: "balancer" }).pretty()

// Логи балансировщика
db.adminCommand({ getLog: "balancer" })
```

Оптимизация 
```javascript

// Для больших коллекций можно настроить размер чанка
use config
db.settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 64 } },  // Размер в MB
  { upsert: true }
)

// Принудительное разделение чанков
sh.splitAt("sample_db.companies", { permalink: "m" }) 
```

Дополнительные команды для диагностики

```javascript
// Проверить статистику по шардам
db.adminCommand({ listShards: 1 })

// Детальная информация по конкретному чанку
use config
db.chunks.findOne({ ns: "sample_db.companies" })

// Принудительная балансировка конкретной коллекции
sh.moveChunk("sample_db.companies", { permalink: "twitter" }, "shard2")
```


## Распространение пользователей в кластере
### Для серверов конфигурации
Только на первичной (primary) ноде:
  1. Пользователи создаются один раз на первичном config сервере
  2. Автоматически реплицируются на все вторичные config серверы через механизм replica set
  3. Не нужно создавать вручную на каждом config сервере

### Для шардов
На первичной ноде каждого шарда:
  1. Локальные пользователи для каждого шарда создаются на его первичной ноде
  2. Реплицируются на вторичные ноды шарда автоматически
  3. Пользователи базы данных admin реплицируются на все шарды

### Для роутеров (mongos))

Не хранят пользователей:
  1. Mongos не хранит данные аутентификации
  2. Перенаправляет запросы аутентификации на config серверы