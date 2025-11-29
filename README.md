# nginx-ssl-proxy-guide

![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%20|%2024.04-E95420?style=flat&logo=ubuntu&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-1.24+-009639?style=flat&logo=nginx&logoColor=white)
![Let's Encrypt](https://img.shields.io/badge/SSL-Let's%20Encrypt-003A70?style=flat&logo=letsencrypt&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue?style=flat)
![Maintained](https://img.shields.io/badge/Maintained-Yes-green?style=flat)

Руководство по быстрой настройке Ubuntu сервера с Nginx, SSL-сертификатом и проксированием на приложение.

## Содержание

- [Требования](#требования)
- [Обновление системы](#1-обновление-системы)
- [Установка Nginx](#2-установка-nginx)
- [Установка Certbot](#3-установка-certbot)
- [Настройка Nginx](#4-настройка-nginx-для-домена)
- [Получение SSL](#5-получение-ssl-сертификата)
- [Полезные команды](#полезные-команды)
- [Troubleshooting](#troubleshooting)

## Требования

- Ubuntu 22.04 / 24.04 LTS
- Доменное имя, направленное на IP сервера
- Root или sudo доступ
- Открытые порты 80 и 443

## 1. Обновление системы

```bash
sudo apt update && sudo apt upgrade -y
```

## 2. Установка Nginx

```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Проверяем статус:
```bash
sudo systemctl status nginx
```

## 3. Установка Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
```

## 4. Настройка Nginx для домена

Создаём конфигурацию для сайта:

```bash
sudo nano /etc/nginx/sites-available/myapp
```

Вставляем базовую конфигурацию (замените `example.com` на ваш домен, `3000` на порт приложения):

```nginx
server {
    listen 80;
    server_name example.com www.example.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Активируем конфигурацию:

```bash
# Удаляем дефолтный конфиг (чтобы избежать конфликтов)
sudo rm /etc/nginx/sites-enabled/default

# Активируем наш конфиг
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/

# Проверяем и применяем
sudo nginx -t
sudo systemctl reload nginx
```

## 5. Получение SSL-сертификата

Убедитесь, что домен уже указывает на IP сервера, затем:

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Перезагружаем веб сервер nginx
```bash
sudo systemctl reload nginx
```

Certbot автоматически:
- Получит сертификат от Let's Encrypt
- Настроит редирект с HTTP на HTTPS
- Добавит автообновление сертификата

## 6. Проверка автообновления

```bash
sudo certbot renew --dry-run
```

## Полезные команды

| Команда | Описание |
|---------|----------|
| `sudo nginx -t` | Проверить конфигурацию |
| `sudo systemctl reload nginx` | Применить изменения |
| `sudo systemctl restart nginx` | Перезапустить Nginx |
| `sudo certbot certificates` | Список сертификатов |
| `sudo certbot renew` | Обновить сертификаты |

## Итоговая структура конфига (после Certbot)

После работы Certbot конфиг будет выглядеть примерно так:

```nginx
server {
    server_name example.com www.example.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    if ($host = www.example.com) {
        return 301 https://$host$request_uri;
    }
    if ($host = example.com) {
        return 301 https://$host$request_uri;
    }

    listen 80;
    server_name example.com www.example.com;
    return 404;
}
```

## Troubleshooting

**Ошибка "nginx: [emerg] could not build server_names_hash"**
```bash
sudo nano /etc/nginx/nginx.conf
# Добавить в блок http:
server_names_hash_bucket_size 64;
```

**Порт занят**
```bash
sudo lsof -i :80
sudo lsof -i :443
```

**Логи Nginx**
```bash
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log
```
