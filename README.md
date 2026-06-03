# CI/CD Pipeline для фронтенда (React + Vite + Docker)

Этот документ — краткая шпаргалка по настройке CI/CD для фронтенд-проекта с автодеплоем на VPS (Selectel).

## 📦 Общая схема работы

```mermaid
graph LR
    A[git push] -->|триггер| B(GitHub Actions);
    B --> C[Запуск тестов];
    C --> D[Сборка приложения];
    D --> E[Сборка Docker-образа];
    E --> F[Пуш в ghcr.io];
    F --> G[Ручной деплой];
    G --> H[Сервер];
```


🧠 1. Что такое CI/CD?
CI (Continuous Integration): Автоматическая проверка кода после каждого пуша. Содержит:

Установку зависимостей (npm ci)

Запуск линтеров и тестов (npm run lint, npm run test)

Сборку приложения (npm run build).

Цель: убедиться, что новый код ничего не сломал.

CD (Continuous Deployment/Delivery): Автоматическая доставка приложения на сервер.

В моем случае — ручной (по кнопке workflow_dispatch).

Доставка через Docker-образ, который запускается на сервере.

🐳 2. Почему Docker?
Единое окружение: приложение работает в изолированном контейнере. На сервере больше не нужно ставить Node.js, npm или nginx. Все зависимости уже внутри образа.

Простота деплоя: на сервере нужен только Docker и команды docker pull + docker run.

Версионирование: Docker-образы хранятся в реестре (GitHub Container Registry). В любой момент можно откатиться к старому образу docker run ghcr.io/...:old_sha.

Основные сущности Docker
Dockerfile: рецепт сборки образа (многостадийная сборка: builder + nginx:alpine).

Docker-образ: готовый "слепок" приложения (хранится в ghcr.io).

Контейнер: запущенный образ (живет на сервере).

Volume: папка вне контейнера для сохранения данных (логов, загрузок пользователя). В чистом фронтенде обычно не нужен.

🔐 3. SSH-ключи (самое важное)
SSH нужен для того, чтобы GitHub Actions (раннер GitHub) мог безопасно подключиться к твоему серверу и выполнить команды (docker pull, docker run).

Как это работает (асимметричное шифрование)
Приватный ключ (Приватный): Никому не показывать. Лежит у тебя локально и в GitHub Secrets. Это "ключ от замка".

Публичный ключ (Публичный): Можно показывать всем. Лежит на сервере в файле ~/.ssh/authorized_keys. Это "замок".

Как настроить (краткая инструкция)
Создать новую пару ключей для GitHub Actions (чтобы не путать с личными):

```bash
ssh-keygen -t ed25519 -f ~/.ssh/github_actions_key -C "github-actions"
```

Положить публичный ключ на сервер:

```bash
ssh-copy-id -i ~/.ssh/github_actions_key.pub user@123.123.123.123
```

Если команда не сработала, вручную добавь содержимое .pub файла в ~/.ssh/authorized_keys на сервере.

Добавить приватный ключ в GitHub Secrets:

Скопируй содержимое приватного ключа: cat ~/.ssh/github_actions_key.

В GitHub: Settings → Secrets and variables → Actions → New repository secret.

Название: STAGING_SSH_KEY (или PRODUCTION_SSH_KEY).

Значение: вставить github_actions_key.

Важно: GitHub Actions будет использовать ключ из Secrets, чтобы зайти на сервер. Твой личный ключ (id_ed25519) используется только для ручного входа.

🔄 4. GitHub Actions (CI)
Файл: .github/workflows/ci.yml

Что происходит при пуше в main или develop:

Job test: Устанавливает зависимости и запускает тесты.

Job build: Собирает статику (папка dist).

Job docker (только при пуше, не для PR):

Логинится в GitHub Container Registry (ghcr.io).

Собирает Docker-образ.

Пушит образ в ghcr.io с тегами:

latest (для main)

develop (для develop)

sha-{hash} (уникальный тег для отката)

🚀 5. GitHub Actions (CD)
Файл: .github/workflows/deploy-staging.yml (и deploy-prod.yml)

Запуск: только вручную (workflow_dispatch).

Что происходит при нажатии кнопки:

Устанавливается SSH-ключ из Secrets.

По SSH подключаемся к серверу и выполняем команды:

```bash
docker login ghcr.io -u ${{ github.actor }} --password-stdin
docker pull ghcr.io/${{ github.repository }}:latest   # или :develop
docker stop my-app || true
docker rm my-app || true
docker run -d --name my-app -p 8080:80 --restart always ghcr.io/${{ github.repository }}:latest
```

📦 6. GitHub Container Registry (ghcr.io)
Это хранилище Docker-образов, привязанное к твоему GitHub-аккаунту.

Адрес: ghcr.io/ваш-логин/название-репозитория

Логин из CI: используется встроенный ${{ secrets.GITHUB_TOKEN }}.

Права: для записи нужно явно разрешить packages: write в permissions воркфлоу.

🛠️ 7. Что нужно на сервере (VPS)
Docker и Docker Compose (опционально).

Публичный SSH-ключ в ~/.ssh/authorized_keys.

Открытый порт 8080 (или другой, который указан в docker run -p).

🧩 8. Шпаргалка по файлам
Файл	Для чего
Dockerfile	Многостадийная сборка: builder (Node.js) → nginx:alpine.
nginx.conf	Отдача статики, обработка SPA-роутинга (try_files $uri /index.html), кеширование.
ci.yml	Тесты, сборка, публикация образа в ghcr.io.
deploy-staging.yml	Ручной деплой образа :develop на staging-сервер.
deploy-prod.yml	Ручной деплой образа :latest на production-сервер.
.gitignore	Должен содержать .env, node_modules, dist.
🔥 9. Частые ошибки и решения
Ошибка	Причина	Решение
npm ci fails	Рассинхрон package-lock.json и package.json	Выполнить npm install локально и запушить новый package-lock.json.
denied: installation not allowed	Нет прав на запись в ghcr.io	Добавить в workflow permissions: packages: write.
Host key verification failed	Сервер не добавлен в known_hosts раннера	Добавить шаг ssh-keyscan -H ${{ secrets.HOST }} >> ~/.ssh/known_hosts.
SSH permission denied	Не тот ключ в Secrets, или публичный ключ не добавлен на сервер.	Проверить ssh-copy-id, проверить содержимое секрета.
При перезагрузке страницы 404	Nginx не знает про роутинг SPA.	Проверить наличие try_files $uri $uri/ /index.html; в nginx.conf.
📌 10. Памятка
Установил новый пакет? → Запушти package-lock.json.

Хочешь обновить приложение? → Нажми Run workflow в Actions.

Сломал деплой? → Найди в ghcr.io старый тег (SHA) и запусти его вручную на сервере: docker run ... ghcr.io/...:старый_sha.

Забыл, что в секретах? → GitHub → Settings → Secrets and variables → Actions (но показать их нельзя, только перезаписать).