# Конспект: Процессы и сервисы (systemd)

## 1. Что такое процесс

**Процесс** — запущенная программа. Каждый процесс имеет уникальный **PID** (Process ID).

Процессы образуют **дерево**:
- Корень — процесс с PID 1 (systemd)
- Каждый процесс (кроме первого) создан другим процессом — **родительским**

```
systemd (PID 1)
├── sshd
│   └── sshd (ваше подключение)
│       └── bash
│           └── ls
├── cron
└── nginx
    ├── nginx (worker)
    └── nginx (worker)
```

---

## 2. Просмотр процессов

### `ps` — снимок процессов

```bash
ps aux                 # все процессы (a=все, u=подробно, x=без терминала)
ps aux | grep sshd     # найти процесс по имени
```

Основные столбцы:
- `USER` — владелец
- `PID` — идентификатор
- `%CPU`, `%MEM` — использование ресурсов
- `COMMAND` — команда запуска

### `pstree` — дерево процессов

```bash
pstree                 # дерево процессов
pstree -p              # с PID
```

### `top` — интерактивный мониторинг

```bash
top                    # процессы в реальном времени
```

| Клавиша | Действие |
|---------|----------|
| `q` | Выйти |
| `k` | Завершить процесс |
| `M` | Сортировать по памяти |
| `P` | Сортировать по CPU |
| `1` | Показать все ядра |

> **htop** — улучшенная версия: `sudo apt install htop`

---

## 3. Управление процессами

### Сигналы

| Сигнал | Номер | Описание |
|--------|-------|----------|
| `SIGTERM` | 15 | Вежливое завершение (сохраняет данные) |
| `SIGKILL` | 9 | Принудительное завершение (нельзя игнорировать) |
| `SIGHUP` | 1 | Перечитать конфигурацию |

### `kill` — отправка сигнала

```bash
kill 1234              # SIGTERM (корректное завершение)
kill -9 1234           # SIGKILL (принудительное)
kill -HUP 1234         # SIGHUP (перечитать конфиг)
```

> **Правило:** сначала `kill` (SIGTERM), если не реагирует — `kill -9`

### Фоновые процессы

```bash
команда &              # запустить в фоне
Ctrl+C                 # прервать текущий процесс
Ctrl+Z                 # приостановить текущий процесс
jobs                   # список фоновых задач
fg                     # вернуть на передний план
bg                     # продолжить в фоне
```

---

## 4. systemd — система управления сервисами

**systemd** — процесс с PID 1, корень дерева процессов. Управляет:
- запуском сервисов при загрузке
- мониторингом и перезапуском упавших сервисов

### Типы юнитов (units)

| Тип | Расширение | Описание |
|-----|-----------|----------|
| Сервис | `.service` | Демон (nginx, sshd) |
| Таймер | `.timer` | Запуск по расписанию |
| Сокет | `.socket` | Активация при подключении |
| Цель | `.target` | Группа юнитов (уровни запуска) |

---

## 5. systemctl — управление сервисами

### Статус и управление

```bash
systemctl status ssh           # статус сервиса
sudo systemctl start ssh       # запустить
sudo systemctl stop ssh        # остановить
sudo systemctl restart ssh     # перезапустить (stop + start)
sudo systemctl reload ssh      # перечитать конфиг (без остановки)
```

> **restart vs reload:** `restart` — полная остановка, `reload` — без разрыва соединений (не все сервисы поддерживают)

### Автозапуск

```bash
sudo systemctl enable ssh          # включить автозапуск
sudo systemctl disable ssh         # отключить автозапуск
sudo systemctl enable --now ssh    # включить и запустить
```

### Список сервисов

```bash
systemctl list-units --type=service              # все загруженные
systemctl list-units --type=service --state=running   # только работающие
systemctl list-timers                            # все таймеры
```

---

## 6. journalctl — чтение логов

```bash
journalctl -u ssh                 # логи сервиса
journalctl -u ssh -n 20           # последние 20 строк
journalctl -u ssh -f              # следить в реальном времени (Ctrl+C выход)
journalctl -u ssh --since today   # с начала дня
journalctl -u ssh --since "1 hour ago"
```

---

## 7. Устройство unit-файла

### Где хранятся

| Каталог | Назначение |
|---------|-----------|
| `/lib/systemd/system/` | Файлы от пакетов (nginx, ssh, cron) |
| `/etc/systemd/system/` | Файлы администратора (имеют приоритет) |

### Структура (на примере ssh.service)

```ini
[Unit]
Description=OpenBSD Secure Shell server
After=network.target          # запускать после настройки сети

[Service]
Type=notify                   # тип сервиса
ExecStart=/usr/sbin/sshd -D   # основная команда
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target    # цель для автозапуска
```

### Типы `Type`

| Type | Описание |
|------|----------|
| `simple` | Процесс работает постоянно |
| `oneshot` | Выполняется и завершается |
| `notify` | Как simple, но уведомляет systemd о готовности |
| `forking` | Создаёт дочерний процесс и завершается |

---

## 8. Практика: сервис и таймер для бэкапа

### Шаг 1. Service-файл
```bash
sudo nano /etc/systemd/system/wordpress-backup.service
```

```ini
[Unit]
Description=Backup wordpress-project
After=local-fs.target

[Service]
Type=oneshot
User=webmaster
ExecStart=/home/student/wordpress-project/backup.sh
```

### Шаг 2. Timer-файл
```bash
sudo nano /etc/systemd/system/wordpress-backup.timer
```

```ini
[Unit]
Description=Run wordpress-backup daily

[Timer]
OnCalendar=daily          # ежедневно в полночь
Persistent=true           # выполнить, если был пропущен

[Install]
WantedBy=timers.target
```

**Варианты `OnCalendar`:**
- `hourly` — каждый час
- `daily` — раз в сутки
- `weekly` — раз в неделю
- `*-*-* 03:00:00` — каждый день в 3:00
- `Mon *-*-* 09:00:00` — понедельник в 9:00

### Шаг 3. Активация
```bash
sudo systemctl daemon-reload                      # перечитать unit-файлы
sudo systemctl enable --now wordpress-backup.timer  # включить и запустить таймер
sudo systemctl start wordpress-backup.service     # запустить вручную для проверки
```

### Шаг 4. Проверка
```bash
systemctl status wordpress-backup.service
journalctl -u wordpress-backup.service
systemctl list-timers
```

---

## 9. Шпаргалка

### Процессы
| Команда | Описание |
|---------|----------|
| `ps aux` | Все процессы |
| `ps aux \| grep имя` | Найти процесс |
| `pstree -p` | Дерево процессов с PID |
| `top` | Интерактивный мониторинг |
| `kill PID` | Завершить процесс (SIGTERM) |
| `kill -9 PID` | Принудительно завершить |

### systemctl
| Команда | Описание |
|---------|----------|
| `systemctl status имя` | Статус |
| `systemctl start/stop/restart имя` | Управление |
| `systemctl enable/disable имя` | Автозапуск |
| `systemctl list-timers` | Список таймеров |
| `systemctl daemon-reload` | Перечитать unit-файлы |

### journalctl
| Команда | Описание |
|---------|----------|
| `journalctl -u имя` | Логи сервиса |
| `journalctl -u имя -f` | Следить в реальном времени |
| `journalctl -u имя --since today` | Логи с начала дня |

---

## 10. Итог: что нужно запомнить

1. **Процесс** — запущенная программа с уникальным **PID**
2. **Дерево процессов** — корень `systemd` (PID 1)
3. **Сигналы:** `SIGTERM` (15) — вежливое завершение, `SIGKILL` (9) — принудительное
4. **systemd** — система инициализации и управления сервисами
5. **systemctl** — главная команда для управления сервисами и таймерами
6. **journalctl** — просмотр логов всех сервисов
7. **Unit-файлы** лежат в `/etc/systemd/system/` (свои) и `/lib/systemd/system/` (от пакетов)
8. **Таймеры** — современная замена cron для расписания задач
