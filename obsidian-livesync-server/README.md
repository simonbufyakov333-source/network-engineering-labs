# Self-hosted Obsidian LiveSync

Самостоятельно размещённый сервер синхронизации Obsidian LiveSync на VPS с использованием:

- Docker
- CouchDB 3.4
- Nginx Reverse Proxy
- Let's Encrypt SSL
- Self-hosted LiveSync Plugin

## Архитектура

```
                Internet
                     │
             HTTPS (443)
                     │
              Nginx Reverse Proxy
                     │
             localhost:5984
                     │
              Docker Container
                CouchDB 3.4
                     │
        Self-hosted LiveSync Database
```

---

# Возможности

- Полностью собственный сервер синхронизации
- HTTPS
- Автоматическое продление сертификатов Let's Encrypt
- Работа с Windows
- Работа с Android
- End-to-End Encryption (через LiveSync)
- Не требует Obsidian Sync

---

# Используемый стек

| Компонент | Версия |
|-----------|---------|
| Ubuntu | 26.04 |
| Docker | Latest |
| Docker Compose | v2 |
| CouchDB | 3.4 |
| Nginx | 1.28 |
| Certbot | Latest |

---

# Структура проекта

```
/opt/services/couchdb/

├── compose.yaml
├── .env
├── data/
└── etc/
```

---

# Docker Compose

```yaml
services:
  couchdb:
    image: couchdb:3.4

    container_name: couchdb

    restart: unless-stopped

    environment:
      COUCHDB_USER: admin
      COUCHDB_PASSWORD: YOUR_PASSWORD

    ports:
      - "127.0.0.1:5984:5984"

    volumes:
      - ./data:/opt/couchdb/data
      - ./etc:/opt/couchdb/etc/local.d
```

---

# Переменные окружения

Файл `.env`

```env
COUCHDB_USER=admin
COUCHDB_PASSWORD=YOUR_LONG_PASSWORD
```

---

# Запуск

```bash
docker compose up -d
```

Проверка

```bash
docker ps
```

---

# Проверка работы CouchDB

```bash
curl http://127.0.0.1:5984
```

Ответ

```json
{
  "couchdb":"Welcome",
  "version":"3.4.3"
}
```

---

# Проверка списка баз

```bash
set -a
source .env
set +a

curl -u "$COUCHDB_USER:$COUCHDB_PASSWORD" \
http://127.0.0.1:5984/_all_dbs
```

---

# Reverse Proxy

Файл

```
/etc/nginx/sites-available/obsidian
```

```nginx
server {
    listen 80;
    server_name obsidian.example.com;

    return 301 https://$host$request_uri;
}

server {

    listen 443 ssl;
    http2 on;

    server_name obsidian.example.com;

    ssl_certificate /etc/letsencrypt/live/obsidian.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/obsidian.example.com/privkey.pem;

    client_max_body_size 100M;

    location / {

        proxy_pass http://127.0.0.1:5984;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_buffering off;
        proxy_request_buffering off;

        proxy_read_timeout 3600;
        proxy_send_timeout 3600;
    }
}
```

---

# Проверка CORS

```bash
curl -i -X OPTIONS \
-H "Origin: capacitor://localhost" \
-H "Access-Control-Request-Method: GET" \
-H "Access-Control-Request-Headers: authorization,content-type" \
https://obsidian.example.com/obsidian-livesync
```

Ожидаемый ответ

```
HTTP/2 204

Access-Control-Allow-Origin
Access-Control-Allow-Headers
Access-Control-Allow-Credentials
```

---

# Настройка Obsidian (Windows)

Установить плагин:

```
Self-hosted LiveSync
```

Настройки

```
URL
https://obsidian.example.com

Database
obsidian-livesync

Username
admin

Password
**********
```

Включить

```
Enable LiveSync
```

При первом запуске:

```
Overwrite server from this device
```

если сервер новый.

---

# Настройка Android

Импортировать

```
Setup URI
```

или

```
Manual setup
```

Заполнить:

```
URL

Username

Password

Database
```

При необходимости включить

```
Use Internal API
```

если Android сообщает

```
Native Fetch API failed
```

---

# Полезные команды

Просмотр контейнера

```bash
docker ps
```

Логи

```bash
docker logs couchdb
```

Перезапуск

```bash
docker restart couchdb
```

Остановка

```bash
docker stop couchdb
```

---

# Проверка документов

```bash
set -a
source .env
set +a

curl -u "$COUCHDB_USER:$COUCHDB_PASSWORD" \
http://127.0.0.1:5984/obsidian-livesync/_all_docs
```

---

# Резервное копирование

Рекомендуется регулярно сохранять каталог

```
/opt/services/couchdb/data
```

или делать резервную копию всего проекта

```
/opt/services/couchdb
```

---

# Возможные проблемы

## Account temporarily locked

Подождать несколько минут или перезапустить контейнер.

---

## Native Fetch API failed

Для Android включить

```
Use Internal API
```

---

## Changes do not sync

Проверить:

- подключение к серверу;
- наличие документа в CouchDB;
- логи LiveSync;
- включён ли LiveSync.

---

# TODO

- [ ] Автоматическое резервное копирование
- [ ] Мониторинг через Uptime Kuma
- [ ] Fail2Ban
- [ ] Cloudflare Proxy
- [ ] GitHub Actions
- [ ] Обновление Docker-образов через Watchtower

---

# Лицензия

MIT
