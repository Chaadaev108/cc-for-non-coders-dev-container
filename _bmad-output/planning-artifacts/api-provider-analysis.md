---
title: Предварительный анализ фичи переключения API-провайдеров
description: Анализ кода, затронутые файлы, варианты реализации, рекомендации
author: Чапаев
date: 2026-02-28
status: analysis-complete
depends_on: documentation-plan.md
next_step: Quick Spec (Barry)
---

# Предварительный анализ: переключение API-провайдеров

## Текущее состояние

Проект жёстко привязан к одному API-провайдеру: **Z.AI** (api.z.ai) с моделями GLM-5 и GLM-4.5-Air. Провайдер выступает как Anthropic-совместимый прокси — Claude Code думает, что общается с Anthropic API, но запросы идут через Z.AI.

### Существующий механизм переключения ключей

В проекте уже реализован `switch-api-key.sh` — переключает между primary и backup ключами **одного и того же провайдера**. Механизм: перезаписывает `ANTHROPIC_AUTH_TOKEN` в `.claude/.env`.

---

## Файлы с захардкоженным провайдером

### 1. Dockerfile (строки 156-161)

```dockerfile
ENV ANTHROPIC_BASE_URL="https://api.z.ai/api/anthropic"
ENV ANTHROPIC_DEFAULT_OPUS_MODEL="GLM-5"
ENV ANTHROPIC_DEFAULT_SONNET_MODEL="GLM-5"
ENV ANTHROPIC_DEFAULT_HAIKU_MODEL="GLM-4.5-Air"
ENV API_TIMEOUT_MS="3000000"
ENV ANTHROPIC_AUTH_TOKEN_BACKUP=""
```

**Роль:** дефолтные значения на уровне Docker-образа. Перекрываются docker-compose и entrypoint.

### 2. docker-compose.yml (строки 15-20)

```yaml
- ANTHROPIC_BASE_URL=https://api.z.ai/api/anthropic
- ANTHROPIC_DEFAULT_OPUS_MODEL=${GLM_OPUS_MODEL:-GLM-5}
- ANTHROPIC_DEFAULT_SONNET_MODEL=${GLM_SONNET_MODEL:-GLM-5}
- ANTHROPIC_DEFAULT_HAIKU_MODEL=${GLM_HAIKU_MODEL:-GLM-4.5-Air}
- API_TIMEOUT_MS=${API_TIMEOUT_MS:-3000000}
```

**Роль:** передаёт переменные в контейнер. `ANTHROPIC_BASE_URL` — захардкожен (не параметризован через `.env`). Имена моделей берутся из `.env`, но с дефолтами GLM.

### 3. claude-settings.json (строки 37-41)

```json
"env": {
    "ANTHROPIC_BASE_URL": "https://api.z.ai/api/anthropic",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "GLM-5",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "GLM-5",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "GLM-4.5-Air",
    "API_TIMEOUT_MS": "3000000"
}
```

**Роль:** статические настройки Claude Code CLI. Копируется в контейнер при сборке (Dockerfile строка 117). **Важно:** этот файл перезаписывает переменные окружения — Claude Code может использовать именно его значения, игнорируя `.claude/.env`.

### 4. entrypoint.sh (строки 23-30)

```bash
cat > /home/coder/.claude/.env << EOF
ANTHROPIC_AUTH_TOKEN=${ANTHROPIC_AUTH_TOKEN:-}
ANTHROPIC_BASE_URL=${ANTHROPIC_BASE_URL:-https://api.z.ai/api/anthropic}
ANTHROPIC_DEFAULT_OPUS_MODEL=${ANTHROPIC_DEFAULT_OPUS_MODEL:-GLM-5}
ANTHROPIC_DEFAULT_SONNET_MODEL=${ANTHROPIC_DEFAULT_SONNET_MODEL:-GLM-5}
ANTHROPIC_DEFAULT_HAIKU_MODEL=${ANTHROPIC_DEFAULT_HAIKU_MODEL:-GLM-4.5-Air}
API_TIMEOUT_MS=${API_TIMEOUT_MS:-3000000}
EOF
```

**Роль:** генерирует `.claude/.env` при старте контейнера. Берёт значения из переменных окружения, с fallback на Z.AI.

### 5. .env.example (строки 4-12)

```env
GLM_API_KEY=your-zai-api-key-here
GLM_OPUS_MODEL=GLM-5
GLM_SONNET_MODEL=GLM-5
GLM_HAIKU_MODEL=GLM-4.5-Air
```

**Роль:** шаблон для пользователя. Имена переменных (`GLM_*`) привязаны к конкретному провайдеру.

### 6. switch-api-key.sh (генерируется в entrypoint.sh, строки 33-52)

**Роль:** переключает только `ANTHROPIC_AUTH_TOKEN`. Не меняет `BASE_URL` и модели. Работает только в рамках одного провайдера.

### 7. README.md (строки 45-91)

Инструкции по настройке упоминают Z.AI, `GLM_API_KEY`, `switch-api-key.sh`.

---

## ИССЛЕДОВАНИЕ: Приоритет конфигурации Claude Code CLI

**Дата исследования:** 2026-02-28
**Статус:** завершено, критические находки

### Находка 1: `.claude/.env` НЕ является официальным механизмом

Claude Code CLI **не загружает** файлы `.claude/.env` автоматически. Это не dotenv, не документированная функция. Файл `.claude/.env`, генерируемый в `entrypoint.sh` (строки 22-30), **не читается Claude Code CLI напрямую**.

Это значит: текущий механизм `switch-api-key.sh` (который перезаписывает `.claude/.env`) **может не работать** для Claude Code. Он работает только если переменные из этого файла подхвачены bash-сессией через `source`, а не Claude Code напрямую.

### Находка 2: `settings.json` env секция — официальный механизм

Claude Code CLI читает секцию `env` из `settings.json` и устанавливает эти переменные как реальные переменные окружения процесса. Приоритет:

```
Managed settings (enterprise)          ← наивысший
         ↓
Аргументы командной строки (claude --env ...)
         ↓
.claude/settings.local.json (env)      ← проектный, персональный
         ↓
.claude/settings.json (env)            ← проектный, командный
         ↓
~/.claude/settings.json (env)          ← пользовательский
         ↓
Переменные окружения ОС (Docker ENV)  ← наименьший
```

### Находка 3: В текущем проекте двойная конфигурация

В проекте **два конфликтующих механизма:**

1. `claude-settings.json` → копируется в `~/.claude/settings.json` (Dockerfile строка 117) — содержит захардкоженные Z.AI значения в `env`
2. `entrypoint.sh` → генерирует `.claude/.env` — **не читается Claude Code**

Переменные из docker-compose.yml (`ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_BASE_URL`) устанавливаются как переменные окружения ОС контейнера. Но `settings.json` `env` секция может их перекрывать (приоритет settings.json выше ОС-переменных по документации).

### Вывод: что реально работает сейчас

- `ANTHROPIC_AUTH_TOKEN` — берётся из ОС (docker-compose), потому что его НЕТ в settings.json → **работает**
- `ANTHROPIC_BASE_URL` — захардкожен в settings.json env → **перекрывает ОС-переменную** → переключение невозможно
- `ANTHROPIC_DEFAULT_*_MODEL` — захардкожены в settings.json env → **перекрывает ОС-переменные** → переключение невозможно
- `switch-api-key.sh` — пишет в `.claude/.env` → **Claude Code не читает этот файл** → переключение ключей может не работать (работает только если bash `source`-ит файл)

### Влияние на фичу провайдеров

**КРИТИЧЕСКОЕ.** Переключение провайдера через переменные окружения (docker-compose, entrypoint) **не будет работать**, пока `claude-settings.json` содержит секцию `env` с захардкоженными значениями.

### Решение

Для фичи переключения провайдеров необходимо **генерировать `settings.json`** (или `settings.local.json`) динамически в `entrypoint.sh` с актуальными значениями провайдера. Это единственный официально поддерживаемый механизм.

Альтернатива: убрать секцию `env` из `settings.json` и полагаться на переменные окружения ОС (docker-compose). Тогда достаточно менять env vars контейнера.

---

## Варианты реализации

### Вариант A: Профили провайдеров через .env

**Суть:** набор пресетов (Z.AI, Anthropic, OpenRouter и др.) в `.env`, один активный.

**Изменения:**

- `.env.example` — добавить пресеты: `PROVIDER=zai|anthropic|openrouter`, с блоками переменных для каждого
- `docker-compose.yml` — параметризовать `ANTHROPIC_BASE_URL` через `.env`
- `entrypoint.sh` — логика выбора пресета, генерация `.claude/.env`
- `claude-settings.json` — убрать захардкоженные значения из `env` секции (или генерировать динамически)
- `switch-api-key.sh` — расширить до `switch-provider.sh` (меняет BASE_URL + модели + ключ)

**Плюсы:**

- Простая реализация
- Обратная совместимость (дефолт = Z.AI)
- Не требует пересборки образа

**Минусы:**

- При добавлении нового провайдера нужно редактировать `.env`
- Все пресеты захардкожены в entrypoint.sh

### Вариант B: Конфигурационный файл провайдеров (JSON/YAML)

**Суть:** отдельный файл `providers.json` с описанием провайдеров, entrypoint читает его.

**Изменения:**

- Новый файл `providers.json` — реестр провайдеров (URL, модели, дефолты)
- `entrypoint.sh` — парсить JSON (через `jq`), генерировать `.claude/.env`
- `.env` — только `PROVIDER=zai` и `API_KEY=...`
- `switch-provider.sh` — выбор из реестра
- `claude-settings.json` — убрать или генерировать из реестра

**Плюсы:**

- Легко добавлять провайдеров (редактировать один файл)
- Чистое разделение данных и логики
- Удобно для upstream (контрибьюторы добавляют провайдеров через PR)

**Минусы:**

- Сложнее реализация (парсинг JSON в bash)
- Новый файл в проекте
- `jq` уже установлен в контейнере (не проблема)

### Вариант C: Генерация claude-settings.json в entrypoint

**Суть:** `claude-settings.json` не статический, а генерируется при старте.

**Изменения:**

- `claude-settings.json` — переименовать в `claude-settings.template.json`
- `entrypoint.sh` — генерировать финальный `claude-settings.json` из шаблона + переменных провайдера
- Остальное — как в варианте A или B

**Плюсы:**

- Решает проблему приоритетов (settings.json всегда актуален)
- Один источник правды

**Минусы:**

- Меняет существующую механику (claude-settings.json больше не статический)
- Нужен шаблонизатор (envsubst или sed)

---

## Рекомендуемый подход: B + C (гибрид)

**Вариант B (конфигурационный файл) + элемент C (генерация settings):**

1. Создать `providers.json` — реестр провайдеров
2. `.env` — минимум: `PROVIDER=zai`, `API_KEY=...`, `API_KEY_BACKUP=...`
3. `entrypoint.sh` — читает `providers.json` через `jq`, генерирует `~/.claude/settings.json` (env секция) с актуальными значениями провайдера
4. `switch-provider.sh` — заменяет `switch-api-key.sh`, перегенерирует `settings.json` + показывает инструкцию перезапустить Claude Code
5. `Dockerfile` — убрать захардкоженные ENV, `claude-settings.json` становится шаблоном без секции `env` (permissions остаются)
6. **Убрать генерацию `.claude/.env`** — это нерабочий механизм (Claude Code не читает этот файл)

**Почему этот подход:**

- Использует **официальный механизм** Claude Code CLI (`settings.json` `env` секция)
- Новые провайдеры добавляются одним PR в `providers.json`
- Не ломает существующую механику (дефолт = Z.AI)
- Решает проблему приоритетов (settings.json генерируется с правильными значениями)
- `jq` уже установлен
- Минимальный blast radius при ошибке
- **Исправляет существующий баг:** текущий `switch-api-key.sh` пишет в `.claude/.env`, который Claude Code не читает

---

## Обратная совместимость

### Что не должно сломаться

- **Существующие `.env` файлы** с `GLM_API_KEY` должны работать (миграция через fallback в entrypoint.sh)
- **Дефолтное поведение** без `.env` — Z.AI как и раньше
- **`switch-api-key.sh`** — сохранить для обратной совместимости (deprecated, ссылается на новый скрипт)
- **Healthcheck** в docker-compose — не затронут (auth-gateway не меняется)

### Что изменится

- `.env.example` — новый формат (PROVIDER вместо GLM_*)
- `claude-settings.json` — станет генерируемым (или шаблоном)
- `entrypoint.sh` — основные изменения здесь
- `README.md` — новая секция по провайдерам

### Миграция для существующих пользователей

```bash
# Старый формат (.env):
GLM_API_KEY=xxx
GLM_OPUS_MODEL=GLM-5

# Новый формат (.env):
PROVIDER=zai
API_KEY=xxx

# entrypoint.sh должен поддерживать ОБА формата
# Если видит GLM_API_KEY — маппит на PROVIDER=zai + API_KEY=GLM_API_KEY
```

---

## Оценка затронутых файлов

| Файл | Тип изменения | Критичность |
| --- | --- | --- |
| `providers.json` | Новый файл | Ядро фичи |
| `entrypoint.sh` | Значительная переработка | Высокая (запуск контейнера) |
| `.env.example` | Переработка формата | Средняя |
| `docker-compose.yml` | Параметризация BASE_URL | Средняя |
| `Dockerfile` | Убрать захардкоженные ENV | Средняя |
| `claude-settings.json` | Шаблонизация или генерация | Высокая (приоритет конфигов) |
| `switch-provider.sh` | Новый файл (замена switch-api-key.sh) | Средняя |
| `switch-api-key.sh` | Deprecated wrapper | Низкая |
| `README.md` | Обновление документации | Низкая |
| `login.html` | Без изменений | Не затронут |
| `auth-gateway.py` | Без изменений | Не затронут |
| `run.sh` | Без изменений | Не затронут |
