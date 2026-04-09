Ниже — готовый текст для `README.md` по **нашему VPN-стеку**.
Все чувствительные данные заменены на заглушки.

# 🚀 Xray + VLESS + REALITY + Sing-Box Client

> Production-like VPN setup in Docker with:
>
> - **Xray-core**
> - **VLESS**
> - **REALITY**
> - **XTLS Vision**
> - **Nginx fallback**
> - **Sing-Box client**
>
> Конфигурация рассчитана на сервер с **Ubuntu** и клиент, например, на **Fedora/Linux**.

---

# 📌 Что реализовано

## ✅ Сервер
- Xray работает в Docker
- VLESS + REALITY на `443/tcp`
- fallback на `nginx:8444`
- обычный сайт на домене как fallback
- Let's Encrypt сертификаты для fallback-доменов

## ✅ Клиент
- Sing-Box
- локальный SOCKS5 прокси `127.0.0.1:1080`
- подключение к серверу через **VLESS + REALITY**
- `uTLS` с fingerprint `chrome`

---

# 🧠 Архитектура

```text
Интернет :443
   ↓
Xray (REALITY)
   ├─ REALITY-клиент → VPN туннель
   └─ обычный HTTPS → fallback → Nginx:8444
                                   ↓
                              статический сайт
````

---

# ⚠️ Важно

## 1. Не публикуйте секреты

Никогда не выкладывайте в репозиторий:

* UUID клиентов
* `privateKey`
* `publicKey`
* `shortId`
* IP сервера
* реальные домены, если не хотите их светить

## 2. Комментарии в JSON

`Xray` и `Sing-Box` **не понимают комментарии в JSON**.

Если вы храните шаблоны с комментариями для README, в реальные `config.json` вставляйте **чистый JSON**.

## 3. Внешний HTTPS fallback-сайт

В нашей схеме fallback используется как **живой HTTPS endpoint**, но основная маскировка VPN происходит за счёт **REALITY**, а не за счёт самого сайта.

---

# 🖥️ Сервер: Ubuntu + Docker

## 1. Подготовка сервера

Обновить пакеты:

```bash
sudo apt update && sudo apt upgrade -y
```

Установить Docker и Compose plugin:

```bash
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable --now docker
```

Проверка:

```bash
docker --version
docker compose version
```

---

## 2. Структура проекта

Рабочая директория:

```bash
/srv/matrix-infra
```

Минимальная структура для VPN-части:

```text
/srv/matrix-infra
├── docker-compose.yml
├── nginx
│   ├── certs
│   ├── conf.d
│   │   └── matrix.conf
│   └── www
│       └── html
│           └── index.html
└── xray
    └── config.json
```

Создание директорий:

```bash
mkdir -p /srv/matrix-infra/nginx/conf.d
mkdir -p /srv/matrix-infra/nginx/certs
mkdir -p /srv/matrix-infra/nginx/www/html
mkdir -p /srv/matrix-infra/xray
```

---

# 🔑 Генерация ключей и идентификаторов

## 1. UUID клиента

Один UUID на одно устройство — лучший вариант.

```bash
uuidgen
```

Пример:

```text
00000000-1111-2222-3333-444444444444
```

---

## 2. REALITY key pair

Генерируется **только Xray**, не `openssl`.

```bash
docker run --rm teddysun/xray xray x25519
```

Пример вывода:

```text
Private key: <XRAY_PRIVATE_KEY>
Public key:  <XRAY_PUBLIC_KEY>
```

* `Private key` идёт в **серверный** `xray/config.json`
* `Public key` идёт в **клиентский** `sing-box config`

---

## 3. Short ID

Сгенерировать hex-строку:

```bash
openssl rand -hex 8
```

Пример:

```text
8ec314b32376bceb
```

Можно сделать несколько `shortIds`.

---

# 🐳 docker-compose.yml

Файл:

```text
/srv/matrix-infra/docker-compose.yml
```

```yaml
services:
  nginx:
    image: nginx:latest
    container_name: nginx-proxy
    restart: always
    ports:
      - "80:80"
      - "127.0.0.1:8444:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/certs:/etc/letsencrypt
      - ./nginx/www:/var/www/certbot
    networks:
      - matrix-net

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - ./nginx/certs:/etc/letsencrypt
      - ./nginx/www:/var/www/certbot
    networks:
      - matrix-net

  xray:
    image: teddysun/xray
    container_name: xray-vpn
    restart: always
    ports:
      - "443:443"
    volumes:
      - ./xray/config.json:/etc/xray/config.json:ro
    networks:
      - matrix-net

networks:
  matrix-net:
    driver: bridge
```

---

# 🌐 Nginx fallback

Файл:

```text
/srv/matrix-infra/nginx/conf.d/matrix.conf
```

> Ниже показан fallback-сайт для доменов `ru-bird.ru` и `www.ru-bird.ru`.
> Используйте **свои домены**.

```nginx
server {
    listen 80;
    server_name <ROOT_DOMAIN> <WWW_DOMAIN>;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://<WWW_DOMAIN>$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name <ROOT_DOMAIN>;

    ssl_certificate /etc/letsencrypt/live/<ROOT_DOMAIN>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<ROOT_DOMAIN>/privkey.pem;

    return 301 https://<WWW_DOMAIN>$request_uri;
}

server {
    listen 443 ssl default_server;
    server_name <WWW_DOMAIN>;

    ssl_certificate /etc/letsencrypt/live/<ROOT_DOMAIN>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<ROOT_DOMAIN>/privkey.pem;

    root /var/www/certbot/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

# 📄 Статический fallback-сайт

Файл:

```text
/srv/matrix-infra/nginx/www/html/index.html
```

Пример:

```html
<!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8">
  <title>Сайт работает</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body>
  <h1>Сайт работает</h1>
  <p>Fallback endpoint поднят.</p>
</body>
</html>
```

---

# 🔒 Серверный Xray config

Файл:

```text
/srv/matrix-infra/xray/config.json
```

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 443,
      "listen": "0.0.0.0",
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "<CLIENT_UUID_1>",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none",
        "fallbacks": [
          {
            "dest": 8444
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.google.com:443",
          "xver": 0,
          "serverNames": [
            "www.google.com"
          ],
          "shortIds": [
            "<SHORT_ID_1>"
          ],
          "privateKey": "<XRAY_PRIVATE_KEY>"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom"
    }
  ]
}
```

---

# 🧾 Что значат параметры Xray

## `id`

UUID клиента.

## `flow`

Используем:

```text
xtls-rprx-vision
```

## `fallbacks`

Если входящее соединение не REALITY-клиент, Xray отправляет его на `8444`, где сидит nginx.

## `serverNames`

SNI-маска для REALITY.

## `dest`

Реальный удалённый TLS-ориентир для имитации handshake.
В текущей конфигурации:

```text
www.google.com:443
```

## `privateKey`

Секретный серверный ключ REALITY.

---

# ▶️ Запуск сервера

Из директории проекта:

```bash
cd /srv/matrix-infra
docker compose up -d
```

Проверка:

```bash
docker ps
docker logs xray-vpn
docker logs nginx-proxy
```

Проверка Xray-порта:

```bash
ss -ltnp | grep :443
```

---

# 📜 Получение сертификатов Let's Encrypt

## Для fallback-домена

Если `HTTP-01` challenge не проходит по сети, используйте `DNS challenge`.

Пример для одного домена:

```bash
docker run --rm -it \
  -v /srv/matrix-infra/nginx/certs:/etc/letsencrypt \
  certbot/certbot certonly \
  --manual \
  --preferred-challenges dns \
  -d <ROOT_DOMAIN> \
  -d <WWW_DOMAIN>
```

После запуска `certbot`:

1. он покажет TXT-запись
2. вы добавляете её у DNS-провайдера
3. проверяете через `dig`
4. нажимаете Enter

Проверка:

```bash
dig TXT _acme-challenge.<ROOT_DOMAIN> +short
```

---

# 💻 Клиент: Fedora + Sing-Box

## 1. Установка Sing-Box

На Fedora:

```bash
sudo dnf config-manager addrepo --from-repofile=https://sing-box.app/sing-box.repo
sudo dnf install sing-box
```

---

## 2. Клиентский config.json

Файл:

```text
/etc/sing-box/config.json
```

```json
{
  "log": {
    "level": "info"
  },
  "inbounds": [
    {
      "type": "socks",
      "listen": "127.0.0.1",
      "listen_port": 1080
    }
  ],
  "outbounds": [
    {
      "type": "vless",
      "server": "<SERVER_IP>",
      "server_port": 443,
      "uuid": "<CLIENT_UUID_1>",
      "flow": "xtls-rprx-vision",
      "tls": {
        "enabled": true,
        "server_name": "www.google.com",
        "utls": {
          "enabled": true,
          "fingerprint": "chrome"
        },
        "reality": {
          "enabled": true,
          "public_key": "<XRAY_PUBLIC_KEY>",
          "short_id": "<SHORT_ID_1>"
        }
      }
    }
  ]
}
```

---

# 🧠 Что значат параметры клиента

## `server`

Публичный IP сервера.

## `uuid`

UUID, который должен совпадать с UUID клиента в серверном `xray/config.json`.

## `server_name`

REALITY SNI. Должен совпадать с серверной маской.

## `utls.fingerprint`

Используем `chrome` как стабильный вариант.

## `public_key`

Публичный REALITY ключ сервера.

## `short_id`

Один из `shortIds`, указанных в серверном Xray config.

---

# ▶️ Запуск клиента вручную

```bash
sudo sing-box run -c /etc/sing-box/config.json
```

---

# 🔁 Systemd unit для клиента

Файл:

```text
/etc/systemd/system/sing-box.service
```

```ini
[Unit]
Description=Sing-box Service
After=network.target

[Service]
ExecStart=/usr/bin/sing-box run -c /etc/sing-box/config.json
Restart=always

[Install]
WantedBy=multi-user.target
```

Активировать:

```bash
sudo systemctl enable --now sing-box
```

Проверить:

```bash
sudo systemctl status sing-box
journalctl -u sing-box -f
```

---

# 🌍 Использование VPN через SOCKS5

После запуска клиента локально работает:

```text
127.0.0.1:1080
```

Пример проверки:

```bash
curl --proxy socks5h://127.0.0.1:1080 https://ipinfo.io/ip
```

Если всё настроено правильно, будет показан IP сервера.

Ещё проверка:

```bash
curl --proxy socks5h://127.0.0.1:1080 https://www.google.com -I
```

---

# 🧪 Проверки сервера

## Проверка fallback-сайта локально

```bash
curl -kI https://127.0.0.1:8444 -H 'Host: <WWW_DOMAIN>'
```

## Проверка содержимого fallback

```bash
curl -k https://127.0.0.1:8444 -H 'Host: <WWW_DOMAIN>'
```

## Проверка сертификата fallback

```bash
openssl s_client -connect 127.0.0.1:8444 -servername <WWW_DOMAIN> </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

---

# ⚠️ Важные замечания

## 1. Внешний HTTPS может подменяться вне сервера

В нашей реальной схеме серверный fallback локально работает корректно, но внешний HTTPS к домену может подменяться не на сервере, а по дороге. Это **не ломает VPN**, потому что основная маскировка идёт через REALITY.

## 2. Сертификаты `--manual` не продлеваются автоматически

За продлением нужно следить вручную.

## 3. Один UUID на одно устройство

Это лучший operational practice.

## 4. Не используйте свой домен как REALITY-маску

REALITY должен маскироваться под внешний TLS endpoint, например:

```text
www.google.com
```

---

# 🔐 Hardening checklist

## Минимум

* [x] Xray только на `443`
* [x] nginx fallback только на `127.0.0.1:8444`
* [x] `show: false` в `realitySettings`
* [x] `xtls-rprx-vision`
* [x] `uTLS` fingerprint `chrome`
* [x] отдельный `UUID` на клиента

## Желательно позже

* [ ] убрать лишние наружные порты у дополнительных сервисов
* [ ] автоматизировать продление сертификатов через DNS API
* [ ] вести учёт UUID/shortId по устройствам
* [ ] добавить системный бэкап конфигов и сертификатов

---

# 📂 Быстрый список важных файлов

## Сервер

* `/srv/matrix-infra/docker-compose.yml`
* `/srv/matrix-infra/xray/config.json`
* `/srv/matrix-infra/nginx/conf.d/matrix.conf`
* `/srv/matrix-infra/nginx/www/html/index.html`

## Клиент

* `/etc/sing-box/config.json`
* `/etc/systemd/system/sing-box.service`

---

# ✅ Статус готовности

Если у вас выполняются эти проверки:

```bash
curl --proxy socks5h://127.0.0.1:1080 https://ipinfo.io/ip
curl --proxy socks5h://127.0.0.1:1080 https://www.google.com -I
docker logs xray-vpn
```

и трафик идёт через IP сервера — VPN поднят успешно.

---

# 🛠️ Troubleshooting

## `curl --proxy socks5h://127.0.0.1:1080 ...` не работает

Проверь:

```bash
sudo systemctl status sing-box
ss -ltnp | grep 1080
journalctl -u sing-box -n 100 --no-pager
```

## Xray не стартует

Проверь JSON и логи:

```bash
docker logs xray-vpn
```

## nginx не стартует

Чаще всего:

* неправильный путь к сертификату
* в конфиге есть домен без выпущенного сертификата

Проверить:

```bash
docker logs nginx-proxy
```

## REALITY не поднимается

Проверь:

* `privateKey` на сервере
* `public_key` на клиенте
* `short_id`
* `server_name`
* `uuid`

---

# 📎 Лицензии и ответственность

Этот репозиторий — пример конфигурации и документации для личного администрирования собственного сервера и защищённого клиентского доступа. Используйте его в рамках законных и допустимых для вашей юрисдикции сценариев.

```
