# 📘 Конспект: Логирование и мониторинг в Linux

> **Часть II завершающая:** Мониторинг веб-сервера nginx, системный журнал journalctl и анализ логов.

## 📋 Содержание

- [1. Зачем следить за логами](#1-зачем-следить-за-логами)
- [2. Системный журнал: journalctl](#2-системный-журнал-journalctl)
  - [Приоритеты сообщений](#приоритеты-сообщений)
  - [Основные команды](#основные-команды-journalctl)
  - [Постоянное ограничение размера](#постоянное-ограничение-размера)
- [3. Логи nginx](#3-логи-nginx)
  - [Где лежат логи](#где-лежат-логи)
  - [Формат access.log](#формат-accesslog)
  - [Формат error.log](#формат-errorlog)
- [4. Анализ access.log с помощью coreutils](#4-анализ-accesslog-с-помощью-coreutils)
  - [Количество запросов](#количество-запросов)
  - [Топ IP-адресов](#топ-ip-адресов)
  - [Распределение по кодам ответа](#распределение-по-кодам-ответа)
  - [Найти все ошибки 404](#найти-все-ошибки-404)
  - [Найти серверные ошибки (500)](#найти-серверные-ошибки-500)
  - [Самые популярные страницы](#самые-популярные-страницы)
  - [Запросы за последний час](#запросы-за-последний-час)
  - [Найти ботов](#найти-ботов)
- [5. Анализ error.log](#5-анализ-errorlog)
  - [Типичные ошибки](#типичные-ошибки)
  - [Команды для анализа](#команды-для-анализа-errorlog)
- [6. Ротация логов (logrotate)](#6-ротация-логов-logrotate)
  - [Зачем нужна ротация](#зачем-нужна-ротация)
  - [Конфигурация для nginx](#конфигурация-для-nginx)
  - [Основные директивы](#основные-директивы)
  - [Структура после ротации](#структура-после-ротации)
- [7. Практика: анализируем логи WordPress](#7-практика-анализируем-логи-wordpress)
  - [Шаг 1. Смотрим системный журнал](#шаг-1-смотрим-системный-журнал)
  - [Шаг 2. Генерируем трафик для анализа](#шаг-2-генерируем-трафик-для-анализа)
  - [Шаг 3. Анализируем access.log](#шаг-3-анализируем-accesslog)
  - [Шаг 4. Проверяем error.log](#шаг-4-проверяем-errorlog)
  - [Шаг 5. Проверяем ротацию](#шаг-5-проверяем-ротацию)
- [8. Справочник команд](#8-справочник-команд)
  - [journalctl](#journalctl-команды)
  - [Анализ логов nginx](#анализ-логов-nginx-команды)
- [9. Итоги](#9-итоги)

---

## 1. Зачем следить за логами

Сервер работает круглосуточно, и вы не можете наблюдать за ним каждую секунду. **Логи — это глаза и уши администратора.**

Без логов невозможно:

- понять, почему пользователь видит ошибку 500;
- найти, кто сканирует сервер в поисках уязвимостей;
- выяснить, когда и почему «упал» сайт;
- оценить нагрузку — сколько запросов в час получает сервер.

---

## 2. Системный журнал: journalctl

### Приоритеты сообщений

Каждое сообщение в журнале имеет **приоритет** — уровень важности.

| Приоритет | Номер | Описание |
|-----------|-------|----------|
| `emerg` | 0 | Система неработоспособна |
| `alert` | 1 | Требуется немедленное вмешательство |
| `crit` | 2 | Критическая ошибка |
| `err` | 3 | Ошибка |
| `warning` | 4 | Предупреждение |
| `notice` | 5 | Важное информационное сообщение |
| `info` | 6 | Информационное сообщение |
| `debug` | 7 | Отладочная информация |

### Основные команды journalctl

```bash
# Фильтрация по приоритету
journalctl -p err                    # только ошибки и критичнее
journalctl -p warning                # предупреждения и критичнее
journalctl -p err --since today      # ошибки за сегодня

# Логи текущей загрузки
journalctl -b                        # логи с момента загрузки
journalctl -b -p err                 # ошибки с момента загрузки

# Логи конкретного сервиса
journalctl -u nginx                  # только логи nginx
journalctl -u php*-fpm               # только логи PHP-FPM

# Размер журнала и управление
journalctl --disk-usage              # сколько места занимает
sudo journalctl --vacuum-size=500M   # оставить не более 500 МБ
sudo journalctl --vacuum-time=7d     # удалить записи старше 7 дней
```

### Постоянное ограничение размера

Отредактируйте файл `/etc/systemd/journald.conf`:

```ini
SystemMaxUse=500M
```

После изменений перезапустите службу:

```bash
sudo systemctl restart systemd-journald
```

---

## 3. Логи nginx

### Где лежат логи

| Файл | Содержание |
|------|-----------|
| `/var/log/nginx/access.log` | Каждый HTTP-запрос к серверу |
| `/var/log/nginx/error.log` | Ошибки nginx и PHP-FPM |

### Формат access.log

```
192.168.1.50 - - [17/Mar/2026:10:15:32 +0300] "GET /wp-login.php HTTP/1.1" 200 4523 "-" "Mozilla/5.0"
```

| # | Поле | Пример | Значение |
|---|------|--------|----------|
| 1 | IP-адрес клиента | `192.168.1.50` | Кто обратился |
| 2 | Идентификатор | `-` | Обычно пустой |
| 3 | Пользователь | `-` | Имя (при HTTP-авторизации) |
| 4 | Дата и время | `[17/Mar/2026:10:15:32 +0300]` | Когда |
| 5 | Запрос | `"GET /wp-login.php HTTP/1.1"` | Метод, URL, протокол |
| 6 | Код ответа | `200` | Результат |
| 7 | Размер ответа | `4523` | В байтах |
| 8 | Referrer | `"-"` | Откуда пришёл пользователь |
| 9 | User-Agent | `"Mozilla/5.0 ..."` | Браузер или бот |

### Формат error.log

```
2026/03/17 10:20:45 [error] 1234#1234: *5 open() "/var/www/wordpress/favicon.ico" failed (2: No such file or directory), client: 192.168.1.50, server: _, request: "GET /favicon.ico HTTP/1.1"
```

---

## 4. Анализ access.log с помощью coreutils

### Количество запросов

```bash
sudo wc -l /var/log/nginx/access.log
```

### Топ IP-адресов

Кто чаще всего обращается к серверу:

```bash
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
```

**Результат:**
```
    342 192.168.1.50
    128 10.0.0.5
     45 192.168.1.1
```

### Распределение по кодам ответа

```bash
sudo awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
```

**Результат:**
```
    412 200
     23 301
     15 404
      3 500
```

### Найти все ошибки 404

```bash
# Какие страницы вызывают 404
sudo awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Полные строки с 404
sudo grep '" 404 ' /var/log/nginx/access.log
```

### Найти серверные ошибки (500)

```bash
sudo grep '" 500 ' /var/log/nginx/access.log
```

### Самые популярные страницы

```bash
sudo awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
```

### Запросы за последний час

```bash
sudo awk -v d="$(date -d '1 hour ago' '+%d/%b/%Y:%H')" '$4 ~ d' /var/log/nginx/access.log | wc -l
```

### Найти ботов

```bash
sudo grep -i 'bot\|crawl\|spider' /var/log/nginx/access.log | awk '{print $1}' | sort -u
```

---

## 5. Анализ error.log

### Типичные ошибки

| Ошибка | Причина |
|--------|---------|
| `No such file or directory` | Запрошенный файл не существует (404) |
| `Permission denied` | Нет прав на чтение файла |
| `upstream timed out` | PHP-FPM не ответил вовремя |
| `connect() failed` | PHP-FPM не запущен или сокет недоступен |

### Команды для анализа error.log

```bash
# Последние ошибки
sudo tail -20 /var/log/nginx/error.log

# Поиск по типу
sudo grep 'Permission denied' /var/log/nginx/error.log
sudo grep 'upstream' /var/log/nginx/error.log

# Ошибки за сегодня
sudo grep "$(date '+%Y/%m/%d')" /var/log/nginx/error.log
```

> **Совет:** Если сайт показывает ошибку 502 Bad Gateway, первое, что нужно проверить — работает ли PHP-FPM (`systemctl status php*-fpm`) и нет ли ошибок про `upstream` в error.log.

---

## 6. Ротация логов (logrotate)

### Зачем нужна ротация

Логи растут непрерывно. **Ротация логов** решает эту проблему: старый лог архивируется (сжимается), создаётся новый пустой файл, а самые старые архивы удаляются.

### Конфигурация для nginx

```bash
cat /etc/logrotate.d/nginx
```

```
/var/log/nginx/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        if [ -d /etc/nginx ]; then
            invoke-rc.d nginx rotate >/dev/null 2>&1
        fi
    endscript
}
```

### Основные директивы

| Директива | Значение |
|-----------|----------|
| `daily` | Ротация раз в день |
| `weekly` | Ротация раз в неделю |
| `rotate 14` | Хранить 14 архивов (2 недели) |
| `compress` | Сжимать старые логи (gzip) |
| `delaycompress` | Не сжимать самый свежий архив |
| `notifempty` | Не ротировать пустые файлы |
| `missingok` | Не выдавать ошибку, если файла нет |
| `create 0640 www-data adm` | Создать новый файл с указанными правами |
| `postrotate` ... `endscript` | Команды после ротации |

### Структура после ротации

```
/var/log/nginx/
├── access.log            # текущий лог
├── access.log.1          # вчерашний (ещё не сжат)
├── access.log.2.gz       # позавчерашний (сжат)
├── access.log.3.gz       # три дня назад
└── ...
```

---

## 7. Практика: анализируем логи WordPress

### Шаг 1. Смотрим системный журнал

```bash
journalctl -u nginx -p err --since today
journalctl -u php*-fpm -p err --since today
journalctl -u mariadb -p err --since today
```

### Шаг 2. Генерируем трафик для анализа

```bash
curl -s http://localhost > /dev/null
curl -s http://localhost/wp-login.php > /dev/null
curl -s http://localhost/not-found-page > /dev/null
curl -s http://localhost/readme.html > /dev/null
```

### Шаг 3. Анализируем access.log

```bash
# Сколько запросов
sudo wc -l /var/log/nginx/access.log

# Топ запрашиваемых URL
sudo awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# Коды ответов
sudo awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Есть ли 404?
sudo awk '$9 == 404 {print $7}' /var/log/nginx/access.log
```

### Шаг 4. Проверяем error.log

```bash
sudo tail -10 /var/log/nginx/error.log
```

### Шаг 5. Проверяем ротацию

```bash
ls -la /var/log/nginx/
cat /etc/logrotate.d/nginx
```

---

## 8. Справочник команд

### journalctl (команды)

| Команда | Описание |
|---------|----------|
| `journalctl -p err` | Только ошибки и критичнее |
| `journalctl -p warning` | Предупреждения и критичнее |
| `journalctl -b` | Логи с момента загрузки |
| `journalctl -u nginx` | Логи конкретного сервиса |
| `journalctl --disk-usage` | Размер журнала на диске |
| `sudo journalctl --vacuum-size=500M` | Ограничить размер 500 МБ |
| `sudo journalctl --vacuum-time=7d` | Удалить записи старше 7 дней |

### Анализ логов nginx (команды)

| Команда | Описание |
|---------|----------|
| `sudo wc -l /var/log/nginx/access.log` | Количество запросов |
| `sudo awk '{print $1}' access.log \| sort \| uniq -c \| sort -rn \| head` | Топ IP-адресов |
| `sudo awk '{print $9}' access.log \| sort \| uniq -c \| sort -rn` | Распределение по кодам |
| `sudo awk '$9 == 404 {print $7}' access.log` | Запросы с ошибкой 404 |
| `sudo awk '{print $7}' access.log \| sort \| uniq -c \| sort -rn \| head` | Популярные страницы |
| `sudo tail -20 /var/log/nginx/error.log` | Последние ошибки |
| `sudo grep -i 'bot' access.log` | Поиск ботов |

---

## 9. Итоги

**Что нужно запомнить:**

| № | Что узнали | Ключевые команды |
|---|------------|------------------|
| 1 | Приоритеты journalctl | `journalctl -p err`, `-p warning` |
| 2 | Формат логов nginx | `access.log` (9 полей), `error.log` |
| 3 | Анализ access.log | `awk '{print $1}'`, `sort`, `uniq -c` |
| 4 | Поиск ошибок | `awk '$9 == 404'`, `grep '" 500 '` |
| 5 | Ротация логов | `/etc/logrotate.d/nginx` |

**📌 Ключевые команды для запоминания:**

```bash
# Смотреть ошибки в реальном времени
sudo journalctl -u nginx -f

# Найти самых активных посетителей
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# Проверить последние ошибки
sudo tail -20 /var/log/nginx/error.log

# Узнать размер логов
sudo journalctl --disk-usage && du -sh /var/log/nginx/
```

**📌 Ключевое правило администратора:** При возникновении проблем с сайтом всегда сначала проверяйте три вещи:

```bash
# 1. Системный журнал
journalctl -u nginx -p err --since "5 minutes ago"

# 2. Лог ошибок nginx
sudo tail -50 /var/log/nginx/error.log

# 3. Лог доступа (для анализа запросов)
sudo tail -50 /var/log/nginx/access.log
```

---

**📌 Часть II завершена.** Вы прошли путь от установки пакетов до полноценного веб-сервера с мониторингом.
