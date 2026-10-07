# STM Infrastructure

## Сервисы

| Сервис | Репо | Описание |
|--------|------|----------|
| `bot` | [stm-bot](https://github.com/gladishevvasiliy/stm-bot) | Основной Telegram-бот симптотермального метода |
| `bot-test` | [stm-bot](https://github.com/gladishevvasiliy/stm-bot) | Тестовый стенд — тот же образ, другие env |
| `stm-admin-bot` | [stm-bot](https://github.com/gladishevvasiliy/stm-bot) | Админ-бот кампаний и управления основным/тестовым контейнерами |
| `slack-to-telegram` | [slack-to-telegram](https://github.com/gladishevvasiliy/slack-to-telegram) | Форвардит алерты из Slack в Telegram + мониторит stm-bot |

## Деплой

Репозитории приложений имеют GitHub Actions workflow `.github/workflows/docker.yml`, который:
1. Собирает Docker-образ
2. Пушит в GitHub Container Registry (`ghcr.io/gladishevvasiliy/<name>`)
3. Подключается к серверу по SSH и обновляет свой сервис через `docker compose up -d`

`stm-bot` собирает один образ для `bot` и `stm-admin-bot`; второй сервис запускается командой `node admin/index.js`. Отдельный образ админ-бота больше не используется. Команды `/restart`, `/restart_test`, `/stop_test`, `/start_test` управляют только фиксированными контейнерами через Docker API и смонтированный `/var/run/docker.sock`; Docker CLI внутри нового образа не требуется. Доступ к сокету даёт контейнеру высокие права на сервере: не передавать токен админ-бота и администраторские ID посторонним. Сам `stm-infra` хранит конфигурацию Compose и не перезапускает сервисы при обновлении файлов в Git — изменения нужно применить на сервере.

### Ветки stm-bot

| Ветка | Образ | Сервис |
|-------|-------|--------|
| `main` | `stm-bot:latest` | `bot` и `stm-admin-bot` (продакшн) |
| `test` | `stm-bot:test` | `bot-test` (тестовый стенд) |

Остальные приложения деплоятся только из `main`.

## Первое переключение админ-бота

1. Дождаться сборки `ghcr.io/gladishevvasiliy/stm-bot:latest` с каталогом `admin/`. До этого не применять новую конфигурацию Compose на сервере.
2. Отдельно применить новую схему MongoDB для кампании: production-деплой `stm-bot` не запускает `prisma db push`. Перед изменением рабочей БД сделать резервную копию и проверить результат команды; не использовать `--force-reset` или `--accept-data-loss` без отдельного разбора предупреждений.
3. На сервере обновить `~/stm-infra/stm-admin-bot.env` по новому примеру. Старое значение `BOT_TOKEN` (токен админ-бота) перенести в `ADMIN_BOT_TOKEN`; в `BOT_TOKEN` указать токен основного бота, который отправляет сообщения пользователям. Также задать `ADMIN_TG_USER_IDS`, `MONGODB_URL` и `REACTIVATION_CAMPAIGN_ID`. Если отдельный алерт-аккаунт отправляет `/restart`, добавить его Telegram ID в `ADMIN_TG_USER_IDS`: он получит доступ также к рассылке и управлению тестовым контейнером. Секреты не коммитить.
4. Обновить `~/stm-infra` из Git и выполнить:

   ```bash
   cd ~/stm-infra
   docker compose config --quiet
   docker compose pull stm-admin-bot
   docker compose up -d --no-deps --force-recreate stm-admin-bot
   docker compose ps stm-admin-bot
   docker compose logs --tail=100 stm-admin-bot
   ```

Compose пересоздаст **существующий** сервис `stm-admin-bot`: старый контейнер будет остановлен до запуска нового. Не добавлять второй сервис с тем же Telegram-токеном и не выполнять `docker compose down` для всего проекта. После переключения проверить `/start`, предпросмотр кампании, команды управления тестовым контейнером и доставку `/restart` от алерт-аккаунта. Перезапуск основного контейнера проверять в согласованное окно; рассылку запускать отдельно после проверки.

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
# Заполнить каждый файл реальными значениями: BOT_TOKEN в stm-admin-bot.env — токен основного бота, ADMIN_BOT_TOKEN — токен админ-бота

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
