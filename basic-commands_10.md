
# 📘 Конспект: Установка и настройка nginx + PHP-FPM + WordPress

> **Развёртывание полноценного веб-сайта на WordPress с использованием стека LEMP (Linux, nginx, MariaDB, PHP).**

## 📋 Содержание

- [1. Архитектура решения](#1-архитектура-решения)
- [2. Клиент-серверная модель и HTTP](#2-клиент-серверная-модель-и-http)
  - [Основные компоненты HTTP-запроса](#основные-компоненты-http-запроса)
  - [Основные коды HTTP-ответа](#основные-коды-http-ответа)
- [3. Установка и настройка nginx](#3-установка-и-настройка-nginx)
  - [Установка](#установка-nginx)
  - [Проверка](#проверка-nginx)
  - [Структура каталогов](#структура-каталогов-nginx)
  - [Настройка серверного блока для WordPress](#настройка-серверного-блока-для-wordpress)
  - [Активация сайта](#активация-сайта)
- [4. Установка PHP-FPM](#4-установка-php-fpm)
  - [Установка PHP и модулей](#установка-php-и-модулей)
  - [Проверка статуса](#проверка-статуса-php)
  - [Взаимодействие nginx и PHP-FPM](#взаимодействие-nginx-и-php-fpm)
  - [Тест PHP](#тест-php)
- [5. Установка и настройка MariaDB](#5-установка-и-настройка-mariadb)
  - [Установка и защита](#установка-и-защита-mariadb)
  - [Создание базы данных и пользователя](#создание-базы-данных-и-пользователя)
- [6. Установка и настройка WordPress](#6-установка-и-настройка-wordpress)
  - [Скачивание и распаковка](#скачивание-и-распаковка)
  - [Настройка прав доступа](#настройка-прав-доступа-wordpress)
  - [Настройка wp-config.php](#настройка-wp-configphp)
- [7. Справочник команд](#7-справочник-команд)
  - [nginx](#nginx-команды)
  - [PHP-FPM](#php-fpm-команды)
  - [MariaDB](#mariadb-команды)
  - [WordPress](#wordpress-команды)
- [8. Итоги](#8-итоги)

---

## 1. Архитектура решения

Все компоненты работают на одном сервере. Nginx выступает в роли «привратника», раздавая статику и проксируя динамические запросы к PHP-FPM.

```text
Браузер (Клиент)
    │ (HTTP-запрос)
    ▼
  nginx (Веб-сервер, порт 80)
    │
    ├── Статика (.html, .css, .js) → Отдаёт сам
    │
    └── Динамика (.php) → Проксирует в PHP-FPM
                              │
                              ▼
                          PHP-FPM (Обработчик)
                              │
                              ▼
                          WordPress (PHP-код)
                              │
                              ▼
                          MariaDB (База данных)
```

---

## 2. Клиент-серверная модель и HTTP

- **Клиент** (браузер) инициирует общение (запрос).
- **Сервер** (nginx) обрабатывает запрос и возвращает ответ.

### Основные компоненты HTTP-запроса

| Компонент | Пример | Назначение |
|-----------|--------|------------|
| Метод | `GET`, `POST` | Действие (получить / отправить) |
| URL | `/about`, `/wp-login.php` | Какой ресурс запрашивают |
| Заголовки | `Host`, `User-Agent` | Дополнительная информация |

### Основные коды HTTP-ответа

| Код | Значение | Когда возникает |
|-----|----------|-----------------|
| `200` | OK | Успешный запрос |
| `301` | Moved Permanently | Постоянный редирект |
| `403` | Forbidden | Доступ запрещён |
| `404` | Not Found | Страница не найдена |
| `500` | Internal Server Error | Ошибка на сервере (часто связана с PHP) |

---

## 3. Установка и настройка nginx

### Установка nginx

```bash
sudo apt update
sudo apt install nginx -y
```

### Проверка nginx

```bash
systemctl status nginx          # Статус сервиса
curl http://localhost           # Должна открыться welcome-страница
sudo ss -tlnp | grep :80        # Проверка прослушивания порта
```

### Структура каталогов nginx

| Каталог | Назначение |
|:---|:---|
| `/etc/nginx/nginx.conf` | Главный конфигурационный файл |
| `/etc/nginx/sites-available/` | Хранилище конфигураций сайтов |
| `/etc/nginx/sites-enabled/` | Символические ссылки на активные конфигурации |
| `/var/www/html/` | Стандартная корневая директория (по умолчанию) |
| `/var/log/nginx/` | Логи доступа и ошибок |

### Настройка серверного блока для WordPress

**Создаём директорию сайта:**

```bash
sudo mkdir -p /var/www/wordpress
sudo chown -R $USER:$USER /var/www/wordpress
```

**Создаём конфигурацию `/etc/nginx/sites-available/wordpress`:**

```nginx
server {
    listen 80;
    server_name _; # Замените на ваш домен или IP

    root /var/www/wordpress;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args; # Красивые URL для WP
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php-fpm.sock; # Связь с PHP-FPM
    }

    location ~ /\.ht {
        deny all; # Запрет доступа к .htaccess
    }
}
```

### Активация сайта

```bash
sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default  # Удаляем дефолтный сайт
sudo nginx -t                              # Проверка конфигурации
sudo systemctl reload nginx                # Применение изменений
```

---

## 4. Установка PHP-FPM

### Установка PHP и модулей

```bash
sudo apt install php-fpm php-mysql php-xml php-mbstring php-curl php-gd php-intl php-zip -y
```

### Проверка статуса PHP

```bash
systemctl status php*-fpm
```

### Взаимодействие nginx и PHP-FPM

Общение происходит через **Unix-сокет** `/run/php/php-fpm.sock`, что быстрее, чем через TCP-порт.

### Тест PHP

```bash
echo '<?php phpinfo(); ?>' | sudo tee /var/www/wordpress/info.php
curl http://localhost/info.php
sudo rm /var/www/wordpress/info.php # Обязательно удалите после проверки!
```

---

## 5. Установка и настройка MariaDB

### Установка и защита MariaDB

```bash
sudo apt install mariadb-server -y
sudo mysql_secure_installation
```

**Рекомендации при `mysql_secure_installation`:**
- Для root оставить аутентификацию через unix_socket
- Удалить анонимных пользователей
- Запретить удаленный root-доступ

### Создание базы данных и пользователя

```bash
sudo mariadb
```

```sql
CREATE DATABASE wordpress CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'SecurePassword123';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

> **Важно:** Замените `SecurePassword123` на свой надёжный пароль.

---

## 6. Установка и настройка WordPress

### Скачивание и распаковка

```bash
cd /tmp
curl -LO https://wordpress.org/latest.tar.gz
sudo tar -xzf latest.tar.gz -C /var/www/
```

### Настройка прав доступа WordPress

```bash
sudo chown -R www-data:www-data /var/www/wordpress
sudo find /var/www/wordpress -type d -exec chmod 755 {} \;
sudo find /var/www/wordpress -type f -exec chmod 644 {} \;
sudo chmod -R g+w /var/www/wordpress/wp-content
```

### Настройка wp-config.php

```bash
cd /var/www/wordpress
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
```

**Внесите данные о базе данных:**

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'SecurePassword123' );
define( 'DB_HOST', 'localhost' );
```

**Сгенерируйте и вставьте уникальные ключи (соли):**

```bash
curl -s https://api.wordpress.org/secret-key/1.1/salt/
```

Скопируйте вывод и замените соответствующие строки в `wp-config.php`.

---

## 7. Справочник команд

### nginx (команды)

| Команда | Описание |
|:---|:---|
| `sudo systemctl status nginx` | Статус сервиса |
| `sudo systemctl reload nginx` | Перезагрузка конфигурации без остановки |
| `sudo nginx -t` | Проверка синтаксиса конфигурации |
| `sudo ss -tlnp \| grep :80` | Проверка, слушает ли nginx порт 80 |

### PHP-FPM (команды)

| Команда | Описание |
|:---|:---|
| `sudo systemctl status php*-fpm` | Статус процесса PHP-FPM |
| `sudo systemctl restart php*-fpm` | Перезапуск обработчика PHP |

### MariaDB (команды)

| Команда | Описание |
|:---|:---|
| `sudo mariadb` | Подключение к БД от root (через сокет) |
| `mariadb -u wpuser -p wordpress` | Подключение к базе от пользователя |
| `sudo mysql_secure_installation` | Защита установки MariaDB |

### WordPress (команды)

| Команда | Описание |
|:---|:---|
| `curl -LO https://wordpress.org/latest.tar.gz` | Скачать последнюю версию WP |
| `curl -s https://api.wordpress.org/secret-key/1.1/salt/` | Генерация солей для wp-config.php |

---

## 8. Итоги

**Что нужно запомнить:**

1. **Архитектура LEMP:** nginx (веб-сервер) → PHP-FPM (обработчик PHP) → MariaDB (база данных)

2. **Последовательность установки:**
   ```bash
   nginx → PHP-FPM → MariaDB → WordPress
   ```

3. **Ключевые порты:**
   - `80` — HTTP (nginx)
   - `3306` — MariaDB (обычно закрыт для внешнего доступа)

4. **Важные пути:**
   - Конфиги nginx: `/etc/nginx/sites-available/` и `/etc/nginx/sites-enabled/`
   - Корень сайта: `/var/www/wordpress`
   - Сокет PHP-FPM: `/run/php/php-fpm.sock`

5. **Обязательные проверки после настройки:**
   ```bash
   sudo nginx -t                    # проверить конфигурацию nginx
   sudo systemctl reload nginx      # применить изменения
   curl -I http://localhost         # проверить HTTP-ответ
   ```

**📌 Ключевое правило администратора:** После каждого изменения конфигурации nginx сначала выполняйте `sudo nginx -t`, и только потом `sudo systemctl reload nginx`. Ошибка в конфигурации может положить весь веб-сервер!

---

**📌 Готово.** Вы настроили полноценный веб-сервер для WordPress с нуля.
