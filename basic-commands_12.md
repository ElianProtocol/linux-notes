# Конспект: Ошибки доступа к Git-репозиторию и их решение

## Ситуация

При попытке клонировать репозиторий возникает ошибка:

```
ERROR: The project you were looking for could not be found 
or you don't have permission to view it.
fatal: Не удалось прочитать из внешнего репозитория.
```

## Почему возникает эта ошибка

| Причина | Описание |
|---------|----------|
| **Репозиторий не существует** | Проект не был создан на GitLab/GitHub |
| **Неправильное имя** | Имя содержит недопустимые символы (`--`, `.git` в конце и т.д.) |
| **Нет прав доступа** | Вы не входите в группу или проект приватный |
| **SSH-ключ не добавлен** | GitLab не узнаёт ваш компьютер |
| **Неверный URL** | Опечатка в имени пользователя или группы |

---

## Диагностика

### 1. Проверить SSH-подключение

```bash
ssh -T git@gitlab.com
```

**Успешный ответ:**
```
Welcome to GitLab, @username!
```

**Ошибка:** значит ключ не добавлен или неверный.

### 2. Проверить существование репозитория

```bash
git ls-remote git@gitlab.com:elianprotocol/wordpress--project.git
```

**Если ошибка** — репозитория не существует.

### 3. Проверить текущий remote

```bash
cd ~/wordpress--project
git remote -v
```

---

## Решения

### Решение 1: Создать репозиторий в личном пространстве

```bash
# 1. Создать проект на GitLab через браузер:
#    - New project → Create blank project
#    - Project name: wordpress-project (без двойных дефисов)
#    - Не инициализировать с README

# 2. Изменить remote в локальном репозитории
git remote set-url origin git@gitlab.com:debiankv/wordpress-project.git

# 3. Отправить код
git push -u origin main
```

### Решение 2: Создать репозиторий в группе

```bash
# Требует прав на запись в группе elianprotocol
git remote set-url origin git@gitlab.com:elianprotocol/wordpress-project.git
git push -u origin main
```

### Решение 3: Добавить SSH-ключ

```bash
# Сгенерировать ключ (если нет)
ssh-keygen -t ed25519 -C "ваша-почта@example.com"

# Показать публичный ключ
cat ~/.ssh/id_ed25519.pub

# Скопировать и добавить в GitLab:
# Settings → SSH Keys → Add key
```

### Решение 4: Начать с чистого листа

```bash
# Удалить старые репозитории
rm -rf ~/wordpress--project
rm -rf ~/wordpress--project-clone

# Создать новый
mkdir -p ~/wordpress-project
cd ~/wordpress-project
git init
echo "# Project" > README.md
git add README.md
git commit -m "Initial commit"

# Добавить remote и запушить
git remote add origin git@gitlab.com:debiankv/wordpress-project.git
git push -u origin main

# Клонировать для проверки
git clone git@gitlab.com:debiankv/wordpress-project.git ~/wordpress-project-clone
```

---

## Правила именования репозиториев на GitLab

| Разрешено | Запрещено |
|-----------|-----------|
| `wordpress-project` | `wordpress--project` (два дефиса) |
| `wordpress_project` | `wordpress-project-` (дефис в конце) |
| `wordpress1` | `-wordpress` (дефис в начале) |
| `wp.project` | `wp.project.git` (оканчивается на .git) |
| `my-project-2024` | `my project` (пробелы) |

**Формула:** только буквы, цифры, `_`, `-`, `.` (не в начале и не в конце)

---

## Типичные ошибки и их решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `Permission denied (publickey)` | Нет SSH-ключа | Добавить ключ в GitLab |
| `repository not found` | Репозиторий не существует | Создать проект |
| `You don't have permission` | Нет прав на группу | Использовать личный namespace |
| `Project namespace path can only include...` | Недопустимые символы | Переименовать проект |

---

## Проверка после исправления

```bash
# 1. Проверить подключение
ssh -T git@gitlab.com

# 2. Проверить remote
git remote -v

# 3. Проверить ветку
git branch

# 4. Отправить код
git push -u origin main

# 5. Клонировать в другую папку
git clone URL ~/test-clone
```

---

## Шпаргалка

| Команда | Действие |
|---------|----------|
| `ssh -T git@gitlab.com` | Проверить SSH-подключение |
| `git remote -v` | Показать удалённые репозитории |
| `git remote set-url origin НОВЫЙ_URL` | Изменить URL |
| `git ls-remote URL` | Проверить существование репозитория |
| `cat ~/.ssh/id_ed25519.pub` | Показать SSH-ключ |

---

## Вывод

Ошибка `project could not be found` почти всегда означает одно из трёх:

1. **Репозиторий не создан** → создайте через веб-интерфейс
2. **Неправильное имя** → используйте `wordpress-project` вместо `wordpress--project`
3. **Нет прав** → используйте свой личный namespace (`debiankv/` вместо `elianprotocol/`)

**Быстрое решение:** создать новый проект на GitLab с именем `wordpress-project` в своём аккаунте и использовать `git remote set-url origin git@gitlab.com:debiankv/wordpress-project.git`
