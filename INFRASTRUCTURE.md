# STM Infrastructure

## Сервисы

| Сервис | Репо | Описание |
|--------|------|----------|
| `bot` | [stm-bot](https://github.com/gladishevvasiliy/stm-bot) | Основной Telegram-бот симптотермального метода |
| `bot-test` | [stm-bot](https://github.com/gladishevvasiliy/stm-bot) | Тестовый стенд — тот же образ, другие env |
| `stm-admin-bot` | [stm-admin-bot](https://github.com/gladishevvasiliy/stm-admin-bot) | Admin-бот: перезапуск/остановка других ботов через Docker |
| `slack-to-telegram` | [slack-to-telegram](https://github.com/gladishevvasiliy/slack-to-telegram) | Форвардит алерты из Slack в Telegram + мониторит stm-bot |

## Деплой

Каждый репо имеет GitHub Actions workflow `.github/workflows/docker.yml`, который:
1. Собирает Docker-образ
2. Пушит в GitHub Container Registry (`ghcr.io/gladishevvasiliy/<name>`)
3. Подключается к серверу по SSH и запускает `docker compose up -d`

### Ветки stm-bot

| Ветка | Образ | Сервис |
|-------|-------|--------|
| `main` | `stm-bot:latest` | `bot` (продакшн) |
| `test` | `stm-bot:test` | `bot-test` (тестовый стенд) |

Остальные репо деплоятся только из `main`.

### Секреты GitHub Actions

В каждом репо (Settings → Secrets → Actions) должны быть:

| Секрет | Значение |
|--------|----------|
| `HOST` | IP сервера |
| `USERNAME` | Пользователь на сервере (`root`) |
| `PRIVATE_KEY` | Приватный SSH ключ (`~/.ssh/github_actions` на сервере) |

## Развёртывание на новом сервере

```bash
# 1. Установить Docker
curl -fsSL https://get.docker.com | sh

# 2. Создать SSH ключ для GitHub Actions
ssh-keygen -t ed25519 -f ~/.ssh/github_actions -N ""
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys

# 3. Клонировать infra
git clone git@github.com:gladishevvasiliy/stm-infra.git ~/stm-infra
cd ~/stm-infra

# 4. Создать env файлы из примеров
cp bot.env.example bot.env
cp bot-test.env.example bot-test.env
cp slack-to-telegram.env.example slack-to-telegram.env
cp stm-admin-bot.env.example stm-admin-bot.env
# Заполнить каждый файл реальными значениями

# 5. Авторизоваться в ghcr.io (нужен Personal Access Token с правом read:packages)
echo YOUR_TOKEN | docker login ghcr.io -u gladishevvasiliy --password-stdin

# 6. Запустить
docker compose pull
docker compose up -d
```

После этого обновить секрет `PRIVATE_KEY` во всех репо на содержимое `~/.ssh/github_actions`.

## Добавление нового сервиса

1. Создать `Dockerfile` в репо сервиса
2. Добавить `.github/workflows/docker.yml` (скопировать из любого существующего репо)
3. Добавить секреты `HOST`, `USERNAME`, `PRIVATE_KEY` в настройки репо
4. Добавить сервис в `docker-compose.yml` и `env.example` в этом репо
5. Запушить изменения в `stm-infra`
