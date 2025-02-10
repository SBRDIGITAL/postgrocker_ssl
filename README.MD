# Методичка по развёртыванию PostgreSQL в Docker Compose в режиме SSL и подключению к ней с помощью сертификатов.

## 1: Установите зависимости `Docker`, `Docker Compose` и `OpenSSL`

### 1.1 Установите Docker:
   - Перейдите на [официальный сайт Docker](https://www.docker.com/products/docker-desktop) и скачайте установщик для вашей операционной системы.
   - Установите Docker, следуя инструкциям на экране.

### 1.2 Установите OpenSSL:
   - Используйте команду для установки:
```bash
apt install openssl
```

### 1.3 Проверьте установку:
   - Для проверки установки `Docker` и `Docker Compose` в командной строке выполните команды:
```bash
docker --version
docker-compose --version
```
   - Для проверки установки `OpenSSL` в командной строке выполните команду:
```bash
openssl help
```

---
## 2: Клонирование проекта

### 2.1 Клонируйте репозиторий:
   - Используйте следующую команду для клонирования репозитория:
```bash
git clone https://github.com/SBRDIGITAL/postgrocker_ssl.git
```

## 3. Генерация SSL-сертификатов

### 3.1 Перейдите в директорию, где будут созданы сертификаты

```bash
cd ./certs
```

### 3.2 Создание корневого сертификата CA

```bash
# Создаём приватный ключ для CA
openssl genpkey -algorithm RSA -out ca.key -pkeyopt rsa_keygen_bits:4096

# Создаём самоподписанный сертификат CA
openssl req -new -x509 -days 3650 -key ca.key -out ca.pem -subj "/C=US/ST=State/L=City/O=Organization/OU=Unit/CN=MyCA"
```

### 3.3 Создание серверного сертификата
**Для `CN=localhost` укажите ip адрес сервера на котором будет работать БД или оставьте `localhost`** 
```bash
# Создаём приватный ключ сервера
openssl genpkey -algorithm RSA -out server.key -pkeyopt rsa_keygen_bits:2048

# Создаём запрос на подпись сертификата
openssl req -new -key server.key -out server.csr -subj "/C=US/ST=State/L=City/O=Organization/OU=Unit/CN=localhost"

# Подписываем CSR с помощью CA
openssl x509 -req -in server.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out server.pem -days 365 -sha256
```

### 3.4 Проверка сертификатов

```bash
# Проверяем подпись сертификата сервера
openssl x509 -in server.pem -noout -issuer -subject

# Проверяем верификацию сертификата
openssl verify -CAfile ca.pem server.pem
```

Должно получиться что-то на подобии:
```bash
issuer=C = US, ST = State, L = City, O = Organization, OU = Unit, CN = MyCA
subject=C = US, ST = State, L = City, O = Organization, OU = Unit, CN = 1.2.3.4
server.pem: OK
```
### 3.5 Выходим из директории с сертификатами

```bash
cd ..
```
## 4. Конфигурация PostgreSQL в Docker Compose

### 4.1 Настраиваем `docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:latest
    container_name: postgres_tls
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydatabase
    volumes:
      - ./data/db:/var/lib/postgresql/data
      - ./certs:/etc/ssl/postgres
      - ./pg_hba.conf:/etc/postgresql/pg_hba.conf
    networks:
      - postgres-cluster
    command: [
      "postgres",
      "-c", "ssl=on",
      "-c", "ssl_cert_file=/etc/ssl/postgres/server.pem",
      "-c", "ssl_key_file=/etc/ssl/postgres/server.key",
      "-c", "ssl_ca_file=/etc/ssl/postgres/ca.pem"
    ]

volumes:
  postgres_data:

networks:
  postgres-cluster:
```

### 4.2 Настройка `pg_hba.conf`

Создайте `pg_hba.conf` рядом с `docker-compose.yml` и добавьте:

```conf
# Разрешаем подключение только по SSL
hostssl all all 0.0.0.0/0 cert clientcert=verify-full
```
или можно использовать:
```bash
# Локальные соединения без пароля
local all all trust

# Соединения через SSL с проверкой сертификата клиента
hostssl all all 0.0.0.0/0 cert clientcert=verify-full

# Внешние соединения через SSL с проверкой пароля
hostssl all all 0.0.0.0/0 md5

# Внешние соединения через SSL с использованием более безопасного хэширования (SCRAM)
hostssl all all 0.0.0.0/0 scram-sha-256
```
## 5. Установка правильных прав на файлы сертификатов

```bash
chmod 600 ./certs/server.key
chmod 644 ./certs/server.pem ./certs/ca.pem
chown 999:999 ./certs/server.key ./certs/server.pem ./certs/ca.pem
```

## 6. Запуск контейнеров

```bash
docker-compose up -d
```

## 7. Подключение к PostgreSQL через pgAdmin 4

### 7.1 Скачайте сертификаты на локальный хост в удобную для вас директорию `не используйте в путях кириллицу!!!`
### 7.2 Конфигурация соединения в `pgAdmin4`
1. Создайте новый сервер (`New Server`).
2. Перейдите во вкладку **Connection**:
    - **Host name/address**: `217.114.13.80`
    - **Port**: `5432`
    - **Maintenance database**: `mydatabase`
    - **Username**: `admin`
    - **Password**: оставить пустым (если `clientcert=verify-full`)
    - **SSL Mode**: `verify-full`
3. Перейдите во вкладку **SSL** и укажите файлы:
    - **Root Certificate**: `ca.pem`
    - **Client Certificate**: `server.pem`
    - **Client Key**: `server.key`
    - **SSL Mode**: `verify-full`
4. Нажмите **Save** и попробуйте подключиться.
5. При подключении использует пароль который указан в `environment: POSTGRES_PASSWORD:` внутри `docker-compose.yml`

## 8. Возможные ошибки и их исправление
### 🔹 Ошибка `CN mismatch`
**Проблема**: PostgreSQL ожидает, что `CN` клиентского сертификата совпадает с именем пользователя (`admin`).
**Решение**: Пересоздать сертификат с `CN=admin` или отключить `clientcert=verify-full` в `pg_hba.conf`.

### 🔹 Ошибка `certificate signature failure`
**Проблема**: Сертификат `server.pem` подписан не тем CA.
**Решение**: Повторно подпишите `server.pem` с помощью `ca.pem`.

### 🔹 Ошибка `could not load private key`
**Проблема**: Неправильные права доступа у `server.key`.
**Решение**: Убедитесь, что:
```bash
chmod 600 ./certs/server.key
chown 999:999 ./certs/server.key
```

## 9. Дополнительно:
### 9.1 Файл `certs\__init__.py` нужен для отслеживания гитом директории `certs`. Он не используется в контейнере.
## Готово! `PostgreSQL` теперь работает в SSL-режиме 🚀
