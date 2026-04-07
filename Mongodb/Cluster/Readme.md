# 📋 1. Подготовка серверов

# 📦 2. Установка MongoDB 7.0
````
sudo dnf install -y mongodb-org
````
# ⚙️ 3. Базовая конфигурация узлов
На всех трёх серверах отредактируйте /etc/mongod.conf:
````
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 0.0.0.0  # или конкретные IP: "127.0.0.1,<ip_узла>"

replication:
  replSetName: "rs0"  # Одинаковое имя на всех узлах!

````

Запустите сервисы на всех узлах:
````
sudo systemctl enable --now mongod
sudo systemctl status mongod
````

# 🚀 4. Инициализация Replica Set
одключитесь к будущему Primary с любого узла (удобнее с того, где будете запускать mongosh):
````
mongosh --host mongo-primary
````
Запустите инициализацию. Лучше указать все узлы сразу:
````
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-primary:27017", priority: 2 },
    { _id: 1, host: "mongo-secondary:27017", priority: 1 },
    { _id: 2, host: "mongo-arbiter:27017", arbiterOnly: true }
  ]
})
````

Проверьте статус:
````
rs.status()
````

# 🔐 5. Включение безопасности (Production)
## Шаг 1: Создайте администратора
На узле Primary:
````
use admin
db.createUser({
  user: "admin",
  pwd: "YourStrongPassword123!",
  roles: [ { role: "root", db: "admin" } ]
})
````

## Шаг 2: Сгенерируйте ключ внутренней аутентификации
````
openssl rand -base64 756 > /etc/mongodb-keyfile
sudo chown mongodb:mongodb /etc/mongodb-keyfile
sudo chmod 400 /etc/mongodb-keyfile
````

Скопируйте файл на все узлы с идентичными правами и владельцем:
````
sudo scp /etc/mongodb-keyfile user@mongo-secondary:/etc/mongodb-keyfile
sudo scp /etc/mongodb-keyfile user@mongo-arbiter:/etc/mongodb-keyfile
````
## Шаг 3: Включите авторизацию в конфиге
На всех узлах добавьте в /etc/mongod.conf:
````
security:
  authorization: "enabled"
  keyFile: "/etc/mongodb-keyfile"
````
## Шаг 4: Перезапустите все узлы
````
sudo systemctl restart mongod
````
````
⏱ Перезапускайте по очереди: сначала Arbiter, потом Secondary, в конце Primary. MongoDB обработает временную потерю кворума автоматически.
````

## Подключение с аутентификацией:
````
mongosh "mongodb://admin:YourStrongPassword123!@mongo-primary:27017/admin?authSource=admin"
````
# 🧪 6. Проверка и тестирование
## 1. Проверка состояния кластера
````
rs.status()
````

## 2. Тест репликации
На Primary:
````
use testdb
db.testcol.insertOne({ message: "Hello from Primary", ts: new Date() })
````
На Secondary (подключитесь с --host mongo-secondary):
````
// По умолчанию чтение с Secondary отключено
rs.slaveOk() // или в mongosh: db.getMongo().setReadPref("secondary")
db.testcol.find()
````

## 3. Тест Failover
На primary:
````
rs.stepDown(60) // Принудительно отдаёт роль Primary на 60 сек
````
Или просто остановите сервис:
````
Или просто остановите сервис:
````