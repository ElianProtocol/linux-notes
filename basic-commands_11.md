
---

# Логирование и мониторинг в Linux

> **Часть II завершающая:** Мониторинг веб-сервера nginx, системный журнал journalctl и анализ логов.

## 📋 Содержание

1. [Зачем следить за логами](#зачем-следить-за-логами)
2. [Системный журнал: journalctl](#системный-журнал-journalctl)
3. [Логи nginx](#логи-nginx)
4. [Анализ access.log с помощью coreutils](#анализ-accesslog-с-помощью-coreutils)
5. [Анализ error.log](#анализ-errorlog)
6. [Ротация логов (logrotate)](#ротация-логов-logrotate)
7. [Практика: анализируем логи WordPress](#практика-анализируем-логи-wordpress)
8. [Справочник команд](#справочник-команд)
9. [Итоги](#итоги)

---

## Зачем следить за логами

Сервер работает круглосуточно, и вы не можете наблюдать за ним каждую секунду. **Логи — это глаза и уши администратора.**

Без логов невозможно:

- понять, почему пользователь видит ошибку 500;
- найти, кто сканирует сервер в поисках уязвимостей;
- выяснить, когда и почему «упал» сайт;
- оценить нагрузку — сколько запросов в час получает сервер.

---

## Системный журнал: journalctl

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

### Основные команды

```bash
# Фильтрация по приоритету
journalctl -p err                    # только ошибки и критичнее
journalctl -p warning                # предупреждения и критичнее
journalctl -p err --since today      # ошибки за сегодня

# Логи текущей загрузки
journalctl -b                        # логи с момента загрузки
journalctl -b -p err                 # ошибки с момента загрузки

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

---

## Логи nginx

### Где лежат логи

| Файл | Содержание |
|------|-----------|
| `/var/log/nginx/access.log` | Каждый HTTP-запрос к серверу |
| `/var/log/nginx/error.log` | Ошибки nginx и PHP-FPM |

### Формат access.log (combined)

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

## Анализ access.log с помощью coreutils

### Количество запросов

```bash
sudo wc -l /var/log/nginx/access.log
```

### Топ IP-адресов (кто чаще всего обращается)

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
sudo awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
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

## Анализ error.log

### Типичные ошибки

| Ошибка | Причина |
|--------|---------|
| `No such file or directory` | Запрошенный файл не существует (404) |
| `Permission denied` | Нет прав на чтение файла |
| `upstream timed out` | PHP-FPM не ответил вовремя |
| `connect() failed` | PHP-FPM не запущен или сокет недоступен |

### Команды для анализа

```bash
# Последние ошибки
sudo tail -20 /var/log/nginx/error.log

# Поиск по типу
sudo grep 'Permission denied' /var/log/nginx/error.log
sudo grep 'upstream' /var/log/nginx/error.log
```

> **Совет:** Если сайт показывает ошибку 502 Bad Gateway, первое, что нужно проверить — работает ли PHP-FPM (`systemctl status php*-fpm`) и нет ли ошибок про `upstream` в error.log.

---

## Ротация логов (logrotate)

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
| `rotate 14` | Хранить 14 архивов (2 недели) |
| `compress` | Сжимать старые логи (gzip) |
| `delaycompress` | Не сжимать самый свежий архив |
| `notifempty` | Не ротировать пустые файлы |
| `create 0640 www-data adm` | Создать новый файл с указанными правами |
| `postrotate` | Команда после ротации (сигнал nginx перечитать файлы) |

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

## Практика: анализируем логи WordPress

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

## Справочник команд

### journalctl

| Команда | Описание |
|---------|----------|
| `journalctl -p err` | Только ошибки и критичнее |
| `journalctl -p warning` | Предупреждения и критичнее |
| `journalctl -b` | Логи с момента загрузки |
| `journalctl --disk-usage` | Размер журнала на диске |
| `sudo journalctl --vacuum-size=500M` | Ограничить размер 500 МБ |
| `sudo journalctl --vacuum-time=7d` | Удалить записи старше 7 дней |

### Анализ логов nginx

| Команда | Описание |
|---------|----------|
| `sudo wc -l /var/log/nginx/access.log` | Количество запросов |
| `sudo awk '{print $1}' access.log \| sort \| uniq -c \| sort -rn \| head` | Топ IP-адресов |
| `sudo awk '{print $9}' access.log \| sort \| uniq -c \| sort -rn` | Распределение по кодам |
| `sudo awk '$9 == 404 {print $7}' access.log` | Запросы с ошибкой 404 |
| `sudo awk '{print $7}' access.log \| sort \| uniq -c \| sort -rn \| head` | Популярные страницы |
| `sudo tail -20 /var/log/nginx/error.log` | Последние ошибки |

---

## Итоги

В этой главе вы:

- ✅ расширили знания о `journalctl`: приоритеты сообщений, фильтрация ошибок, управление размером журнала;
- ✅ разобрали формат логов nginx (access.log и error.log);
- ✅ научились анализировать access.log с помощью coreutils: находить самые активные IP, популярные страницы, коды ошибок;
- ✅ освоили диагностику по error.log: поиск ошибок прав доступа, отсутствующих файлов и проблем с PHP-FPM;
- ✅ познакомились с ротацией логов (logrotate).

---

**📌 Часть II завершена.** Вы прошли путь от установки пакетов до полноценного веб-сервера с мониторингом.

---

## 📁 Файлы для репозитория

Рекомендуемая структура для выгрузки на GitHub:

```
linux-logs-monitoring/
├── README.md                    # этот конспект
├── commands/
│   ├── journalctl-cheatsheet.md # шпаргалка по journalctl
│   ├── nginx-log-analysis.md    # команды для анализа логов
│   └── logrotate-example.conf   # пример конфигурации
├── scripts/
│   ├── top-ips.sh               # скрипт для топа IP
│   ├── error-counter.sh         # подсчет ошибок по типам
│   └── generate-traffic.sh      # генерация тестового трафика
└── examples/
    ├── access.log.sample        # пример лога для тренировки
    └── error.log.sample         # пример error.log
```

---


