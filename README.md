
# Remnawave + MWScdn: XHTTP + SNI Spoofing

Это руководство описывает настройку отказоустойчивой архитектуры для VPN-ноды на базе **Xray (Remnawave)** с использованием протокола **XHTTP** и маскировкой трафика через CDN (MWScdn).

## 📌 Архитектура и решение проблемы
Стандартное проксирование XHTTP-трафика через HTTP-модуль системного Nginx часто приводит к разрывам соединения на клиентах iOS из-за конфликта мультиплексированных HTTP/2 стримов. 

**Наше решение:** Использование Nginx в Docker в качестве «прозрачного» TCP-прокси (модуль `stream`). Nginx принимает трафик на 443 порту и перебрасывает сырые байты напрямую в ядро Xray, которое само занимается расшифровкой TLS и обработкой XHTTP-запросов.

### Схема прохождения трафика:
`Клиент ➔ MWScdn (Порт 443) ➔ Nginx TCP Stream (Порт 443) ➔ Xray Node (Порт 8443, TLS)`

---

## 🛠 Шаг 0: Подготовка сервера

Все команды выполняются на чистой ОС (Ubuntu / Debian). Нам нужно установить Docker и получить SSL-сертификаты для вашего домена.

**1. Обновление системы и установка базовых утилит:**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git certbot

```

**2. Установка Docker Compose:**

```bash
sudo curl -fsSL https://get.docker.com | sh

```

**3. Выпуск SSL-сертификата Let's Encrypt:**
Перед выполнением команды убедитесь, что ваш домен (например, `node.yourdomain.com`) уже направлен (А-запись) на IP вашего сервера.

```bash
sudo certbot certonly --standalone -d node.yourdomain.com

```

*Сертификаты будут сохранены в `/etc/letsencrypt/live/node.yourdomain.com/`. Мы пробросим их внутрь контейнеров на следующих шагах.*

**4. Создание рабочей директории для ноды:**

```bash
mkdir -p /opt/remnanode && cd /opt/remnanode

```

---

## 🚀 Шаг 1: Конфигурация ядра Xray (Remnawave)

Нода должна слушать кастомный внутренний порт (например, `8443`) на всех интерфейсах и самостоятельно обрабатывать TLS.

Откройте ваш профиль Xray и приведите конфигурацию входящего соединения (`inbounds`) к такому виду. Обязательно замените `node.yourdomain.com` на ваш реальный домен:

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "XHTTP_mws",
      "port": 8443,
      "listen": "0.0.0.0",
      "protocol": "vless",
      "settings": {
        "clients": [],
        "fallbacks": [],
        "decryption": "none"
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls",
          "quic"
        ]
      },
      "streamSettings": {
        "network": "xhttp",
        "security": "tls",
        "tlsSettings": {
          "minVersion": "1.2",
          "certificates": [
            {
              "keyFile": "/etc/letsencrypt/live/node.yourdomain.com/privkey.pem",
              "certificateFile": "/etc/letsencrypt/live/node.yourdomain.com/fullchain.pem"
            }
          ],
          "rejectUnknownSni": false
        },
        "xhttpSettings": {
          "mode": "auto",
          "path": "/xhttppath"
        }
      }
    }
  ],
  "outbounds": [
    {
      "tag": "DIRECT",
      "protocol": "freedom"
    },
    {
      "tag": "BLOCK",
      "protocol": "blackhole"
    }
  ],
  "routing": {
    "rules": []
  }
}

```

*Примечание: Директива `"rejectUnknownSni": false` критически важна для приема запросов с кастомным SNI, прошедших сквозь облако МТС.*

---

## 🚀 Шаг 2: Конфигурация Nginx (TCP Stream)

Создадим директорию и конфиг для нашего Docker-Nginx:

```bash
mkdir -p /opt/docker_nginx
nano /opt/docker_nginx/nginx.conf

```

Вставьте этот конфиг. Он использует только модуль `stream` для прозрачного проброса TCP-пакетов без вмешательства в HTTP-заголовки:

```nginx
events {
    worker_connections 1024;
}

stream {
    upstream xray_backend {
        server 127.0.0.1:8443;
    }

    server {
        listen 443;
        proxy_pass xray_backend;
        
        proxy_buffer_size 16k;
        proxy_timeout 52w;
        proxy_connect_timeout 5s;
    }
}

```

---

## 🚀 Шаг 3: Настройка Docker Compose

Оба сервиса (нода и Nginx) запускаются в режиме `network_mode: host` для прямой и быстрой маршрутизации внутри сервера.

Создайте файл `docker-compose.yml` в папке `/opt/remnanode`:

```bash
nano /opt/remnanode/docker-compose.yml

```

```yaml
services:
  remnanode:
    container_name: remnanode
    hostname: remnanode
    image: remnawave/node:latest
    network_mode: host
    restart: always
    volumes:
      - '/etc/letsencrypt:/etc/letsencrypt:ro'
    cap_add:
      - NET_ADMIN
    ulimits:
      nofile:
        soft: 1048576
        hard: 1048576
    environment:
      - NODE_PORT=ВАШ_ПОРТ
      - SECRET_KEY="ВАШ_СЕКРЕТНЫЙ_КЛЮЧ_ИЗ_ПАНЕЛИ"

  nginx-proxy:
    image: nginx:alpine
    container_name: nginx_stream_proxy
    network_mode: host
    restart: always
    volumes:
      - /opt/docker_nginx/nginx.conf:/etc/nginx/nginx.conf:ro

```

Запустите связку:

```bash
docker compose up -d

```

---

## 🚀 Шаг 4: Настройка MWScdn (Облако МТС)

Так как Nginx теперь прозрачно передает данные, CDN должен работать по полноценному защищенному контуру без модификации заголовков.

В панели управления MWScdn:

1. Перейдите во вкладку **«Основное»**:
* Источник: `https://node.yourdomain.com:443` *(ваш домен).*
* Изменить заголовок Host: **ВЫКЛЮЧЕНО**.


2. Перейдите во вкладку **«Оптимизация»**:
* WebSocket: **ВЫКЛЮЧЕНО** *(так как мы используем протокол XHTTP/HTTP2).*



---

## 🚀 Шаг 5: Настройка панели Remnawave

Создайте или отредактируйте Хост в панели администратора для генерации правильных клиентских подписок:

* **Вкладка «Основные»:**
* Адрес: `topxxxxxxxxxx.mwscdn.ru` *(Технический домен CDN для обхода белых списков ТСПУ).*
* Порт: `443`


* **Вкладка «Расширенные»:**
* SNI: `topxxxxxxxxxx.mwscdn.ru`
* Переопределить SNI из адреса: **ВЫКЛЮЧЕНО**
* Хост: *Оставить абсолютно пустым.*
* Путь: /xhttppath
* Security Layer: **TLS**
* ALPN: **h2** *(Обязательно для активации HTTP/2 потока в клиентах).*
