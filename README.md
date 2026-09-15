# Tafel UI Tests with Allure Report

Автотесты UI для проекта **Tafel** на Python + Selenium + Pytest с формированием
отчёта Allure и публикацией на GitHub Pages / GitLab Pages.

[![UI Tests](https://github.com/KiNo1703/Tafel_Ui/actions/workflows/tests.yml/badge.svg)](https://github.com/KiNo1703/Tafel_Ui/actions/workflows/tests.yml)
[![Allure Report](https://img.shields.io/badge/Allure-Report-brightgreen)](https://kino1703.github.io/Tafel_Ui/)

---

## 📋 Содержание

- [О проекте](#о-проекте)
- [Стек](#стек)
- [Структура проекта](#структура-проекта)
- [Установка](#установка)
- [🐳 Запуск через Docker](#-запуск-через-docker)
- [Запуск тестов локально](#запуск-тестов-локально)
- [Allure Report](#allure-report)
- [CI/CD](#cicd)
- [Переменные окружения](#переменные-окружения)
- [Уведомления в Slack](#уведомления-в-slack)
- [Полезные ссылки](#полезные-ссылки)

---

## О проекте

Проект содержит набор UI-автотестов для проверки релизных типов (release types).
Тесты запускаются:

- локально — вручную (в том числе через Docker);
- в CI — по расписанию (каждые 2 часа), при пуше в `main`/`master` и вручную.

Результаты прогонов публикуются в виде Allure-отчёта на GitHub Pages,
а также отправляются уведомлением в Slack.

---

## Стек

| Компонент | Версия / инструмент |
|---|---|
| Язык | Python 3.12 |
| Тестовый фреймворк | Pytest |
| UI-автоматизация | Selenium + Chromium |
| Отчётность | Allure 2.32.0 |
| CI/CD | GitHub Actions |
| Хостинг отчётов | GitHub Pages |
| Уведомления | Slack Webhook |
| Контейнеризация | Docker |

---

## Структура проекта

```
Tafel_Ui/
├── .github/
│   └── workflows/
│       └── tests.yml              # CI workflow
├── tests/
│   └── test_release_types.py      # UI-тесты
├── allure-results/                # сырые результаты (генерируется)
├── Dockerfile                     # образ с Python + Chromium + Allure
├── requirements.txt               # зависимости
├── send_allure_to_slack.py        # скрипт уведомления в Slack
└── README.md
```

---

## Установка

### 1. Клонировать репозиторий

```bash
git clone https://github.com/KiNo1703/Tafel_Ui.git
cd Tafel_Ui
```

### 2. Создать виртуальное окружение

```bash
python -m venv .venv
source .venv/bin/activate        # Linux / macOS
# .venv\Scripts\activate         # Windows
```

### 3. Установить зависимости

```bash
pip install -r requirements.txt
```

### 4. Установить Chrome / Chromium и Allure

**Ubuntu / Debian:**

```bash
sudo apt-get update
sudo apt-get install -y chromium chromium-driver default-jre wget unzip

wget https://github.com/allure-framework/allure2/releases/download/2.32.0/allure-2.32.0.zip
unzip allure-2.32.0.zip -d /opt/
sudo ln -s /opt/allure-2.32.0/bin/allure /usr/local/bin/allure
```

**macOS (через Homebrew):**

```bash
brew install --cask chromium
brew install allure
```

---

## 🐳 Запуск через Docker

Самый простой способ запустить тесты — использовать Docker. Образ уже содержит
Python 3.12, Chromium, Chromedriver, Allure и все системные библиотеки,
необходимые для работы headless-браузера. На хост-машине ничего доустанавливать
не нужно.

### Dockerfile

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y \
    wget unzip curl gnupg chromium chromium-driver default-jre \
    libnss3 libxss1 libasound2 fonts-liberation libgbm1 xdg-utils \
    && rm -rf /var/lib/apt/lists/*

# Симлинки для Chrome (chromedriver уже в /usr/bin)
RUN ln -sf /usr/bin/chromium /usr/bin/google-chrome \
    && ln -sf /usr/bin/chromium /usr/bin/google-chrome-stable

ENV CHROME_BIN=/usr/bin/chromium
ENV CHROMEDRIVER_PATH=/usr/bin/chromedriver

RUN wget https://github.com/allure-framework/allure2/releases/download/2.32.0/allure-2.32.0.zip \
    && unzip allure-2.32.0.zip -d /opt/ \
    && ln -s /opt/allure-2.32.0/bin/allure /usr/local/bin/allure \
    && rm allure-2.32.0.zip

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PYTHONUNBUFFERED=1
ENV PYTHONIOENCODING=utf-8

CMD ["pytest", "tests/test_release_types.py", "-v", "--headless", "--alluredir=allure-results", "--capture=no"]
```

### 1. Собрать образ

```bash
docker build -t tafel-ui-tests .
```

### 2. Запустить тесты

**Вариант A — с сохранением результатов на хост-машину:**

```bash
docker run --rm \
  -v "$(pwd)/allure-results:/app/allure-results" \
  tafel-ui-tests
```

**Windows (PowerShell):**

```powershell
docker run --rm `
  -v "${PWD}/allure-results:/app/allure-results" `
  tafel-ui-tests
```

После прогона результаты окажутся в папке `allure-results/` на вашем компьютере —
их можно открыть через `allure serve allure-results`.

### 3. Открыть Allure-отчёт

```bash
allure serve allure-results
```

### 4. Полезные команды

| Задача | Команда |
|---|---|
| Собрать образ | `docker build -t tafel-ui-tests .` |
| Запустить тесты | `docker run --rm tafel-ui-tests` |
| Запустить с сохранением результатов | `docker run --rm -v "$(pwd)/allure-results:/app/allure-results" tafel-ui-tests` |
| Зайти внутрь контейнера | `docker run --rm -it tafel-ui-tests bash` |
| Запустить конкретный тест | `docker run --rm tafel-ui-tests pytest tests/test_release_types.py -k "название" -v` |
| Удалить образ | `docker rmi tafel-ui-tests` |

### Альтернатива: docker-compose

Если удобнее через `docker-compose.yml`:

```yaml
version: "3.9"

services:
  tests:
    build: .
    volumes:
      - ./allure-results:/app/allure-results
    command: >
      pytest tests/test_release_types.py -v --headless
      --alluredir=allure-results --capture=no
```

Запуск:

```bash
docker compose up --abort-on-container-exit
```

---

## Запуск тестов локально

> Если не хотите использовать Docker — установите Chrome и Allure вручную
> (см. [Установка](#установка)).

### Запуск всех тестов

```bash
pytest tests/test_release_types.py -v
```

### Запуск в headless-режиме (без открытия браузера)

```bash
pytest tests/test_release_types.py -v --headless
```

### Запуск с сохранением результатов для Allure

```bash
pytest tests/test_release_types.py -v --alluredir=allure-results
```

---

## Allure Report

### Сгенерировать и открыть отчёт локально

```bash
allure serve allure-results
```

Отчёт откроется в браузере автоматически.

### Сгенерировать статичный HTML-отчёт

```bash
allure generate allure-results -o allure-report --clean
```

Готовый отчёт будет в папке `allure-report/`.

### Онлайн-отчёт (GitHub Pages)

После каждого прогона в CI отчёт публикуется по адресу:

🔗 **https://kino1703.github.io/Tafel_Ui/**

Отчёты хранятся по пути `builds/tests/<YYYYMMDDHHMM>/`, т.е. каждый прогон —
в отдельной папке с меткой времени.

---

## CI/CD

### GitHub Actions

Workflow: [`.github/workflows/tests.yml`](.github/workflows/tests.yml)

**Триггеры:**

| Событие | Условие |
|---|---|
| `push` | в ветки `main` или `master` |
| `schedule` | каждые 2 часа (`0 */2 * * *`, UTC) |
| `workflow_dispatch` | вручную из вкладки Actions |

**Что делает pipeline:**

1. Забирает код из репозитория.
2. Подтягивает ветку `gh-pages` с историей отчётов.
3. Устанавливает Python 3.12, Chromium, Allure.
4. Ставит зависимости из `requirements.txt`.
5. Формирует уникальный путь отчёта по дате/времени.
6. Запускает Pytest с сохранением результатов в `allure-results`.
7. Генерирует Allure-отчёт.
8. Публикует отчёт в `gh-pages` (GitHub Pages).
9. Отправляет уведомление в Slack.

> 💡 В CI сейчас используется установка Chromium/Allure напрямую через `apt-get`
> на раннере `ubuntu-latest`. При желании можно перейти на запуск внутри
> Docker-образа (см. раздел [🐳 Запуск через Docker](#-запуск-через-docker)) —
> это сделает окружение CI и локальной разработки идентичным.

### GitLab CI (если используется)

Для GitLab создайте `.gitlab-ci.yml` в корне проекта:

```yaml
stages:
  - test
  - report

default:
  image: python:3.12

variables:
  ALLURE_VERSION: "2.32.0"

ui-tests:
  stage: test
  before_script:
    - apt-get update && apt-get install -y chromium chromium-driver default-jre wget unzip
    - wget https://github.com/allure-framework/allure2/releases/download/${ALLURE_VERSION}/allure-${ALLURE_VERSION}.zip
    - unzip allure-${ALLURE_VERSION}.zip -d /opt/
    - ln -s /opt/allure-${ALLURE_VERSION}/bin/allure /usr/local/bin/allure
    - pip install -r requirements.txt
  script:
    - pytest tests/test_release_types.py -v --headless --alluredir=allure-results || true
    - allure generate allure-results -o public --clean
  artifacts:
    when: always
    paths:
      - public
    expire_in: 30 days
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
    - if: $CI_PIPELINE_SOURCE == "push"
    - if: $CI_PIPELINE_SOURCE == "web"

pages:
  stage: report
  dependencies:
    - ui-tests
  script:
    - echo "Publishing Allure report to GitLab Pages"
  artifacts:
    paths:
      - public
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

Либо, если хотите использовать тот же Docker-образ, что и локально:

```yaml
ui-tests:
  stage: test
  image: tafel-ui-tests:latest   # или собранный в предыдущем stage
  script:
    - pytest tests/test_release_types.py -v --headless --alluredir=allure-results || true
    - allure generate allure-results -o public --clean
  artifacts:
    when: always
    paths:
      - public
```

После этого отчёт будет доступен по адресу GitLab Pages:
`https://<username>.gitlab.io/<project>/`

**Расписание в GitLab:** задаётся не в YAML, а в UI —
**CI/CD → Schedules → New schedule** (cron-формат такой же: `0 */2 * * *`).

---

## Переменные окружения

В CI используются следующие секреты (Settings → Secrets and variables → Actions):

| Переменная | Назначение |
|---|---|
| `SLACK_WEBHOOK_URL` | Webhook для отправки уведомлений в Slack |
| `GITHUB_TOKEN` | Автоматически предоставляется GitHub Actions |

Локально переменные можно задать в `.env` или экспортом:

```bash
export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/..."
export CHROME_BIN="/usr/bin/chromium"
export CHROMEDRIVER_PATH="/usr/bin/chromedriver"
```

При запуске через Docker эти переменные уже заданы внутри образа
(`CHROME_BIN`, `CHROMEDRIVER_PATH`, `PYTHONUNBUFFERED`, `PYTHONIOENCODING`).

---

## Уведомления в Slack

После каждого прогона тестов скрипт [`send_allure_to_slack.py`](send_allure_to_slack.py)
отправляет в канал `#general` сообщение со ссылкой на свежий Allure-отчёт.

Пример сообщения:

```
✅ UI Tests finished
Report: https://kino1703.github.io/Tafel_Ui/builds/tests/202509151700/
```

Для настройки:

1. Создайте Incoming Webhook в Slack.
2. Добавьте его в секреты репозитория как `SLACK_WEBHOOK_URL`.

---

## Полезные ссылки

- [Allure Framework](https://docs.qameta.io/allure/)
- [Pytest documentation](https://docs.pytest.org/)
- [Selenium documentation](https://www.selenium.dev/documentation/)
- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [GitLab CI documentation](https://docs.gitlab.com/ee/ci/)
- [Docker documentation](https://docs.docker.com/)

---

## Лицензия

Проект распространяется под лицензией MIT.
