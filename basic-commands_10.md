
---

# 📘 Конспект: Установка и настройка nginx + PHP-FPM + WordPress

Этот конспект является кратким руководством по развертыванию полноценного веб-сайта на WordPress с использованием стека LEMP (Linux, nginx, MariaDB, PHP). Материал основан на 11-м этапе сквозного проекта по администрированию Linux.

## 📖 Оглавление
1. [Архитектура решения](#архитектура-решения)
2. [Клиент-серверная модель и HTTP](#клиент-серверная-модель-и-http)
3. [Установка и настройка nginx](#установка-и-настройка-nginx)
4. [Установка PHP-FPM](#установка-php-fpm)
5. [Установка и настройка MariaDB](#установка-и-настройка-mariadb)
6. [Установка и настройка WordPress](#установка-и-настройка-wordpress)
7. [Справочник команд](#справочник-команд)
8. [Практические задания](#практические-задания)

---

## 🏗 Архитектура решения

Все компоненты работают на одном сервере. Nginx выступает в роли "привратника", раздавая статику и проксируя динамические запросы к PHP-FPM.

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

## 🌐 Клиент-серверная модель и HTTP

*   **Клиент** (браузер) инициирует общение (запрос).
*   **Сервер** (nginx) обрабатывает запрос и возвращает ответ.

### Основные компоненты HTTP-запроса
*   **Метод**: `GET` (получить), `POST` (отправить данные).
*   **URL**: `/about`, `/wp-login.php`.
*   **Заголовки**: `Host`, `User-Agent`.

### Основные коды HTTP-ответа
*   `200 OK`: Успех.
*   `301 Moved Permanently`: Редирект.
*   `403 Forbidden`: Доступ запрещен.
*   `404 Not Found`: Не найдено.
*   `500 Internal Server Error`: Ошибка на сервере (часто связана с PHP).

---

## ⚙️ Установка и настройка nginx

### 1. Установка
```bash
sudo apt update
sudo apt install nginx -y
```

### 2. Проверка
```bash
systemctl status nginx          # Статус сервиса
curl http://localhost           # Должна открыться welcome-страница
sudo ss -tlnp | grep :80        # Проверка прослушивания порта
```

### 3. Структура каталогов
| Каталог | Назначение |
| :--- | :--- |
| `/etc/nginx/nginx.conf` | Главный конфигурационный файл. |
| `/etc/nginx/sites-available/` | Хранилище конфигураций сайтов. |
| `/etc/nginx/sites-enabled/` | Символические ссылки на активные конфигурации. |
| `/var/www/html/` | Стандартная корневая директория (по умолчанию). |
| `/var/log/nginx/` | Логи доступа и ошибок. |

### 4. Настройка серверного блока (Virtual Host) для WordPress

Создаем директорию сайта:
```bash
sudo mkdir -p /var/www/wordpress
sudo chown -R webmaster:webdev /var/www/wordpress
```

Создаем конфигурацию `/etc/nginx/sites-available/wordpress`:
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

### 5. Активация сайта
```bash
sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default  # Удаляем дефолтный сайт
sudo nginx -t                              # Проверка конфигурации
sudo systemctl reload nginx                # Применение изменений
```

---

## 🐘 Установка PHP-FPM

### 1. Установка PHP и модулей
```bash
sudo apt install php-fpm php-mysql php-xml php-mbstring php-curl php-gd php-intl php-zip -y
```

### 2. Проверка статуса
```bash
systemctl status php*-fpm
```

### 3. Взаимодействие nginx и PHP-FPM
Общение происходит через Unix-сокет `/run/php/php-fpm.sock`, что быстрее, чем через TCP-порт.

### 4. Тест (после установки WordPress)
```bash
echo '<?php phpinfo(); ?>' | sudo tee /var/www/wordpress/info.php
curl http://localhost/info.php
sudo rm /var/www/wordpress/info.php # Обязательно удалите после проверки!
```

---

## 🗄️ Установка и настройка MariaDB

### 1. Установка и защита
```bash
sudo apt install mariadb-server -y
sudo mysql_secure_installation
```
*Рекомендации:* Для root оставить аутентификацию через unix_socket, удалить анонимных пользователей, запретить удаленный root-доступ.

### 2. Создание базы данных и пользователя
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
*Важно:* Замените `SecurePassword123` на свой надежный пароль.

---

## 📝 Установка и настройка WordPress

### 1. Скачивание и распаковка
```bash
cd /tmp
curl -LO https://wordpress.org/latest.tar.gz
sudo tar -xzf latest.tar.gz -C /var/www/
```

### 2. Настройка прав доступа
```bash
sudo chown -R webmaster:www-data /var/www/wordpress
sudo find /var/www/wordpress -type d -exec chmod 755 {} \;
sudo find /var/www/wordpress -type f -exec chmod 644 {} \;
sudo chmod -R g+w /var/www/wordpress/wp-content # Права на запись для загрузок и обновлений
```

### 3. Настройка wp-config.php
```bash
cd /var/www/wordpress
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
```

Внесите данные о базе данных:
```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'SecurePassword123' );
define( 'DB_HOST', 'localhost' );
```

Сгенерируйте и вставьте уникальные ключи (соли):
```bash
curl -s https://api.wordpress.org/secret-key/1.1/salt/
```
---

## 📚 Справочник команд

| Компонент | Команда | Описание |
| :--- | :--- | :--- |
| **nginx** | `sudo nginx -t` | Проверка синтаксиса конфигурации. |
| | `sudo systemctl reload nginx` | Перезагрузка конфигурации без остановки. |
| **PHP-FPM** | `sudo systemctl status php*-fpm` | Статус процесса. |
| | `sudo systemctl restart php*-fpm` | Перезапуск обработчика PHP. |
| **MariaDB** | `sudo mariadb` | Подключение к БД от root (через сокет). |
| | `mariadb -u wpuser -p wordpress` | Подключение к базе от пользователя. |
| **WordPress** | `curl -s https://api.wordpress.org/secret-key/1.1/salt/` | Генерация солей. |

---
