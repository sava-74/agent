# Hermes Agent — Аналитический отчёт проекта

**Дата анализа:** 2026-05-18  
**Версия проекта:** 0.14.0  
**Лицензия:** MIT  
**Автор:** Nous Research  

---

## 1. Общая характеристика

**Hermes Agent** — полнофункциональный AI-агент с замкнутым циклом обучения (self-improving loop). Это единственный агент со встроенной системой создания навыков из опыта, их автоулучшения при использовании, автономного управления памятью и кросс-платформенной доставки (CLI + мессенджеры).

### Ключевые возможности

| Возможность | Описание |
|---|---|
| Мультимодальный интерфейс | CLI (prompt_toolkit + Rich), TUI (Ink/React), мессенджеры (Telegram, Discord, Slack, WhatsApp, Signal, Matrix, и др.) |
| 40+ инструментов | web_search, terminal, file_ops, vision, browser, delegate_task, cronjob, kanban, MCP, и др. |
| 30+ провайдеров моделей | OpenAI, Anthropic, Gemini, Bedrock, OpenRouter, DeepSeek, Novita, NVIDIA, Xiaomi, Kimi, MiniMax, Ollama, и др. |
| Система навыков (Skills) | 25+ категорий встроенных + 17 категорий опциональных навыков |
| Кроны (Cron) | Встроенный планировщик задач с доставкой на любую платформу |
| Делегирование | Параллельные подагенты (delegate_task) с ролями leaf/orchestrator |
| Канбан (Kanban) | SQLite-backed мультиагентная доска задач с диспетчером |
| Профили | Изолированные инстансы с собственным HERMES_HOME |
| Плагины | Модель-провайдеры, memory-провайдеры, context-движки, image-gen, observability |
| Дашборд | FastAPI + uvicorn SPA + встроенный TUI через PTY |

---

## 2. Архитектура проекта

### 2.1 Структура директорий

```
hermes-agent/
├── run_agent.py              # AIAgent — ядро агента (~4100 строк)
├── cli.py                    # HermesCLI — интерактивный CLI (~14200 строк)
├── model_tools.py            # Оркестрация инструментов (~900 строк)
├── toolsets.py               # Определения тулсетов (~870 строк)
├── agent/                    # Внутренности агента (~85 файлов)
│   ├── conversation_loop.py  # Главный цикл диалога (~4019 строк)
│   ├── agent_init.py         # __init__ агента (~1470 строк)
│   ├── agent_runtime_helpers.py  # Помощники рантайма (~3800 строк)
│   ├── chat_completion_helpers.py # Помощники chat completion (~4200 строк)
│   ├── anthropic_adapter.py  # Адаптер Anthropic (~3800 строк)
│   ├── transports/           # Транспортный слой (11 файлов)
│   ├── lsp/                  # LSP интеграция (11 файлов)
│   └── ...                   # memory, caching, compression, curator, etc.
├── tools/                    # Реализации инструментов (~76 файлов)
│   ├── registry.py           # Центральный реестр инструментов
│   ├── browser_tool.py       # Браузерная автоматизация (~6700 строк)
│   ├── delegate_tool.py      # Делегирование подагентам (~5000 строк)
│   ├── terminal_tool.py      # Терминальные команды (~4200 строк)
│   ├── mcp_tool.py           # MCP интеграция (~6100 строк)
│   ├── skills_hub.py         # Skills Hub (~5000 строк)
│   ├── environments/         # Бэкенды терминалов (docker, ssh, modal, daytona, и др.)
│   └── ...
├── hermes_cli/               # CLI подкоманды (~90 файлов)
│   ├── main.py               # Точка входа CLI (~20000 строк)
│   ├── config.py             # Конфигурация (~9800 строк)
│   ├── models.py             # Каталог моделей (~6000 строк)
│   ├── gateway.py            # Управление шлюзом (~9400 строк)
│   └── ...
├── gateway/                  # Мессенджер-шлюз
│   ├── run.py                # Основной цикл шлюза
│   ├── session.py            # Управление сессиями
│   └── platforms/            # 33 адаптера платформ
│       ├── telegram.py       # Telegram (~8900 строк)
│       ├── discord.py        # Discord (~10500 строк)
│       ├── feishu.py         # Feishu/Lark (~8800 строк)
│       ├── slack.py          # Slack (~5400 строк)
│       ├── yuanbao.py        # Yuanbao (~8000 строк)
│       └── ...
├── plugins/                  # Плагины (16 директорий)
│   ├── model-providers/      # 30 провайдеров моделей
│   ├── memory/               # 8 memory-провайдеров
│   ├── context_engine/       # Контекстные движки
│   ├── kanban/               # Канбан плагин
│   └── ...
├── skills/                   # 25 категорий встроенных навыков
├── optional-skills/          # 17 категорий опциональных навыков
├── acp_adapter/              # ACP сервер (VS Code / Zed / JetBrains)
├── cron/                     # Планировщик
├── tui_gateway/              # JSON-RPC бэкенд для TUI
├── ui-tui/                   # React/Ink TUI фронтенд
├── web/                      # FastAPI дашборд
├── tests/                    # Тесты (~900+ файлов, ~17k тестов)
└── scripts/                  # Утилиты сборки и CI
```

### 2.2 Цепочка зависимостей

```
tools/registry.py  (нет зависимостей — импортируется всеми tool-файлами)
       ↑
tools/*.py  (каждый вызывает registry.register() при импорте)
       ↑
model_tools.py  (импортирует tools/registry + запускает обнаружение инструментов)
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

### 2.3 Основной цикл агента

Цикл находится в `conversation_loop.py` (извлечён из `run_agent.py`):

```python
while (api_call_count < max_iterations and iteration_budget.remaining > 0) \
        or budget_grace_call:
    if interrupt_requested: break
    response = client.chat.completions.create(model=model, messages=messages, tools=tool_schemas)
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args, task_id)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        return response.content
```

Ключевые свойства:
- **Синхронный** цикл с проверками прерываний и бюджетом итераций
- **Grace call** — один дополнительный вызов после исчерпания бюджета
- Формат сообщений — OpenAI (role: system/user/assistant/tool)
- Reasoning-контент хранится в `assistant_msg["reasoning"]`

### 2.4 Транспортный слой

`agent/transports/` поддерживает несколько API-режимов:

| Транспорт | Назначение |
|---|---|
| `chat_completions.py` | OpenAI-совместимый API (основной) |
| `anthropic.py` | Нативный Anthropic Messages API |
| `bedrock.py` | AWS Bedrock |
| `codex.py` | Codex Responses API |
| `codex_app_server.py` | Codex App Server режим |
| `hermes_tools_mcp_server.py` | MCP сервер для инструментов |

---

## 3. Система инструментов

### 3.1 Реестр инструментов

Центральный реестр (`tools/registry.py`) использует паттерн саморегистрации:
- Каждый tool-файл вызывает `registry.register()` на уровне модуля
- `discover_builtin_tools()` автоматически импортирует все файлы с `registry.register()`
- AST-анализ для определения файлов с регистрацией (без полного импорта)

### 3.2 Тулсеты

Определены в `toolsets.py` как `TOOLSETS` dict. Ключевые тулсеты:

| Тулсет | Инструменты |
|---|---|
| `_HERMES_CORE_TOOLS` | Базовый набор: web, terminal, file, vision, browser, todo, memory, delegate, cronjob, kanban, computer_use |
| `messaging` | Для мессенджер-платформ (наследует core) |
| `browser` | browser_navigate, browser_snapshot, browser_click, и др. |
| `terminal` | terminal, process |
| `delegation` | delegate_task |
| `vision` | vision_analyze |
| `image_gen` | image_generate |
| `tts` | text_to_speech |

### 3.3 Полный список инструментов (40+)

**Web:** web_search, web_extract, x_search  
**Terminal:** terminal, process  
**File:** read_file, write_file, patch, search_files  
**Browser:** browser_navigate, browser_snapshot, browser_click, browser_type, browser_scroll, browser_back, browser_press, browser_get_images, browser_vision, browser_console, browser_cdp, browser_dialog  
**Vision/Image:** vision_analyze, image_generate, video_generate  
**Skills:** skills_list, skill_view, skill_manage  
**Memory/Planning:** todo, memory, session_search  
**Communication:** clarify, send_message  
**Execution:** execute_code, delegate_task  
**Automation:** cronjob  
**Home Assistant:** ha_list_entities, ha_get_state, ha_list_services, ha_call_service  
**Kanban:** kanban_show, kanban_list, kanban_complete, kanban_block, kanban_heartbeat, kanban_comment, kanban_create, kanban_link, kanban_unblock  
**Other:** computer_use, text_to_speech  

---

## 4. Система навыков (Skills)

### 4.1 Структура

Две параллельные поверхности:
- **`skills/`** — встроенные навыки (25 категорий), активны по умолчанию
- **`optional-skills/`** — опциональные (17 категорий), устанавливаются явно через `hermes skills install`

### 4.2 Категории встроенных навыков

apple, autonomous-ai-agents, creative, data-science, devops, diagramming, dogfood, domain, email, gaming, gifs, github, index-cache, inference-sh, mcp, media, mlops, note-taking, productivity, red-teaming, research, smart-home, social-media, software-development, yuanbao

### 4.3 Категории опциональных навыков

autonomous-ai-agents, blockchain, communication, creative, devops, dogfood, email, finance, health, mcp, migration, mlops, productivity, research, security, software-development, web-development

### 4.4 Curator (жизненный цикл навыков)

Фоновая система обслуживания навыков:
- Отслеживает использование агент-созданных навыков
- Авто-архивирует устаревшие (в `~/.hermes/skills/.archive/`)
- LLM-ревью навыков
- PIN-защита от авто-архивации
- Никогда не удаляет — максимум архивация

---

## 5. Система мессенджер-шлюза

### 5.1 Поддерживаемые платформы (20+)

| Платформа | Файл | Размер |
|---|---|---|
| Telegram | telegram.py | ~8900 строк |
| Discord | discord.py | ~10500 строк |
| Feishu/Lark | feishu.py + feishu_comment.py | ~8800 + 2100 строк |
| Yuanbao | yuanbao.py + yuanbao_media + yuanbao_proto + yuanbao_sticker | ~8000 строк |
| Slack | slack.py | ~5400 строк |
| WhatsApp | whatsapp.py | ~2300 строк |
| Signal | signal.py | ~2500 строк |
| Matrix | matrix.py | ~4700 строк |
| Email | email.py | ~1200 строк |
| SMS | sms.py | ~600 строк |
| DingTalk | dingtalk.py | ~2500 строк |
| WeCom | wecom.py + wecom_callback + wecom_crypto | ~3500 строк |
| Weixin | weixin.py | ~3500 строк |
| Mattermost | mattermost.py | ~1400 строк |
| HomeAssistant | homeassistant.py | ~700 строк |
| QQ Bot | qqbot/ (8 файлов) | — |
| BlueBubbles | bluebubbles.py | ~1400 строк |
| API Server | api_server.py | ~6400 строк |
| Webhook | webhook.py | ~1300 строк |
| Microsoft Graph | msgraph_webhook.py | ~600 строк |

### 5.2 Архитектура шлюза

```
gateway/
├── run.py          # Основной цикл шлюза
├── session.py      # Управление сессиями
├── config.py       # Конфигурация
├── delivery.py     # Доставка сообщений
├── pairing.py      # Привязка пользователей
├── hooks.py        # Хуки расширения
├── mirror.py       # Зеркалирование сообщений
├── status.py       # Статус и блокировки
├── platforms/      # Адаптеры платформ
└── builtin_hooks/  # Встроенные хуки
```

Два message guard в шлюзе:
1. **Base adapter** — очередь сообщений при активной сессии
2. **Gateway runner** — перехват служебных команд (/stop, /new, /approve, и др.)

---

## 6. Плагины

### 6.1 Плагин-система

`PluginManager` обнаруживает плагины из:
- `~/.hermes/plugins/`
- `./.hermes/plugins/`
- pip entry points

Хуки жизненного цикла: `pre_tool_call`, `post_tool_call`, `pre_llm_call`, `post_llm_call`, `on_session_start`, `on_session_end`

### 6.2 Провайдеры моделей (30 плагинов)

ai-gateway, alibaba, alibaba-coding-plan, anthropic, arcee, azure-foundry, bedrock, copilot, copilot-acp, custom, deepseek, gemini, gmi, huggingface, kilocode, kimi-coding, minimax, nous, novita, nvidia, ollama-cloud, openai-codex, opencode-zen, openrouter, qwen-oauth, stepfun, xai, xiaomi, zai

### 6.3 Memory-провайдеры (8 плагинов)

byterover, hindsight, holographic, honcho, mem0, openviking, retaindb, supermemory

### 6.4 Другие плагины

browser, context_engine, disk-cleanup, example-dashboard, google_meet, hermes-achievements, image_gen, kanban, observability, platforms, spotify, teams_pipeline, video_gen, web

---

## 7. CLI и точки входа

### 7.1 Точки входа (pyproject.toml)

```
hermes       → hermes_cli.main:main
hermes-agent → run_agent:main
hermes-acp   → acp_adapter.entry:main
```

### 7.2 Ключевые подкоманды

```
hermes              # Интерактивный CLI
hermes model        # Выбор провайдера и модели
hermes tools        # Настройка инструментов
hermes config set   # Установка параметров конфигурации
hermes gateway      # Запуск мессенджер-шлюза
hermes setup        # Полный мастер настройки
hermes doctor       # Диагностика проблем
hermes update       # Обновление версии
hermes cron         # Управление крон-задачами
hermes kanban       # Управление канбан-досками
hermes curator      # Управление жизненным циклом навыков
hermes memory       # Настройка памяти
hermes dashboard    # Веб-дашборд
```

### 7.3 Система Skin/Theme

4 встроенных скина: `default` (золотой/kawaii), `ares` (багровый/бронзовый), `mono` (монохром), `slate` (голубой). Пользовательские скины через YAML в `~/.hermes/skins/`.

---

## 8. Тестирование и CI/CD

### 8.1 Тесты

- **Объём:** ~900+ файлов, ~17,000 тестов
- **Фреймворк:** pytest + pytest-xdist (4 workers)
- **Обязательный запуск:** через `scripts/run_tests.sh` (обеспечивает паритет с CI)
- **Изоляция:** `tests/conftest.py` перенаправляет HERMES_HOME в temp dir

Структура тестов:
```
tests/
├── agent/          (92 файла)
├── cli/            (61 файл)
├── gateway/        (251 файл)
├── hermes_cli/     (222 файла)
├── tools/          (214 файлов)
├── run_agent/      (91 файл)
├── cron/           (14 файлов)
├── skills/         (11 файлов)
├── plugins/        (17 файлов)
├── acp/            (12 файлов)
├── e2e/            (5 файлов)
├── integration/    (8 файлов)
├── stress/         (11 файлов)
└── ...
```

### 8.2 CI/CD (14 workflows)

| Workflow | Назначение |
|---|---|
| `tests.yml` | Основной тестовый пайплайн |
| `lint.yml` | Линтинг (ruff, PLW1514) |
| `docker-publish.yml` | Сборка и публикация Docker-образов |
| `upload_to_pypi.yml` | Публикация в PyPI |
| `supply-chain-audit.yml` | Аудит цепочки поставок |
| `osv-scanner.yml` | Сканирование уязвимостей OSV |
| `uv-lockfile-check.yml` | Проверка консистентности uv.lock |
| `nix.yml` + `nix-lockfile-fix.yml` | Nix сборка |
| `deploy-site.yml` | Деплой документации |
| `skills-index.yml` | Индексация навыков |
| `contributor-check.yml` | Проверка контрибьюторов |
| `history-check.yml` | Проверка истории коммитов |

---

## 9. Зависимости

### 9.1 Основные (core)

Отличительная политика — **точные пины (==X.Y.Z)** для всех зависимостей, без диапазонов. Обоснование: защита от атак на цепочку поставок (Mini Shai-Hulud worm, May 2026).

| Пакет | Версия | Назначение |
|---|---|---|
| openai | 2.24.0 | OpenAI SDK |
| python-dotenv | 1.2.1 | Загрузка .env |
| fire | 0.7.1 | CLI фреймворк |
| httpx[socks] | 0.28.1 | HTTP-клиент |
| rich | 14.3.3 | Терминальное форматирование |
| tenacity | 9.1.4 | Повторные попытки |
| pyyaml | 6.0.3 | YAML парсинг |
| ruamel.yaml | 0.18.17 | YAML с сохранением формата |
| requests | 2.33.0 | HTTP-запросы |
| jinja2 | 3.1.6 | Шаблонизация |
| pydantic | 2.12.5 | Валидация данных |
| prompt_toolkit | 3.0.52 | Интерактивный ввод |
| croniter | 6.0.0 | Cron-выражения |
| PyJWT[crypto] | 2.12.1 | JWT аутентификация |
| psutil | 7.2.2 | Управление процессами |

### 9.2 Опциональные зависимости (20+ extras)

anthropic, exa, firecrawl, parallel-web, fal, edge-tts, modal, daytona, vercel, hindsight, dev, messaging, slack, matrix, cli, tts-premium, voice, pty, honcho, mcp, homeassistant, sms, computer-use, acp, bedrock, termux, dingtalk, feishu, google, youtube, web, all

### 9.3 Ленивая установка зависимостей

Провайдер-специфичные зависимости (anthropic, firecrawl, exa, fal, edge-tts, и др.) устанавливаются **лениво** при первом использовании через `tools/lazy_deps.py`. Это уменьшает поверхность атаки и размер базовой установки.

---

## 10. Развёртывание

### 10.1 Поддерживаемые платформы

- Linux, macOS, WSL2 (полная поддержка)
- Windows native (early beta)
- Android/Termux (кураторский набор extras)
- Docker (официальный Dockerfile на базе Debian trixie + Python 3.13)

### 10.2 Установка

```bash
# Linux/macOS/WSL2
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# Windows (PowerShell)
irm https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.ps1 | iex
```

### 10.3 Терминальные бэкенды (7)

local, Docker, SSH, Singularity, Modal, Daytona, Vercel Sandbox

---

## 11. Метрики проекта

| Метрика | Значение |
|---|---|
| Версия | 0.14.0 |
| Python | >=3.11 |
| Основные файлы Python | ~300+ |
| Файлы инструментов | 76 |
| Адаптеры платформ | 20+ |
| Провайдеры моделей | 30 |
| Memory-провайдеры | 8 |
| Категории навыков | 25 + 17 optional |
| Тестовые файлы | 900+ |
| CI/CD workflows | 14 |
| Основной файл агента (run_agent.py) | ~4100 строк |
| Основной файл CLI (cli.py) | ~14200 строк |
| Основной файл CLI main (hermes_cli/main.py) | ~20000 строк |

---

## 12. Сильные стороны

1. **Модульная архитектура** — чёткое разделение: agent core, tools, gateway, CLI, plugins, skills
2. **Саморегистрация инструментов** — через `registry.register()`, автоматическое обнаружение через AST
3. **Плагинная расширяемость** — 3 типа плагинов (model-providers, memory, general) + ленивая установка зависимостей
4. **Мультиплатформенность** — 20+ мессенджер-платформ, 30+ провайдеров моделей, 7 терминальных бэкендов
5. **Безопасность цепочки поставок** — точные пины зависимостей, OSV-сканирование, supply-chain audit
6. **Профили** — полная изоляция инстансов через HERMES_HOME
7. **Кроны и канбан** — встроенная автоматизация и мультиагентная координация
8. **Качественная документация** — AGENTS.md (1100+ строк), подробный README, Docusaurus-сайт

---

## 13. Потенциальные проблемы и риски

1. **Размер кодовой базы** — отдельные файлы превышают 10,000-20,000 строк (cli.py ~14,200, hermes_cli/main.py ~20,000). Это затрудняет навигацию и поддержку.

2. **Монолитные файлы** — `agent/conversation_loop.py` (4019 строк), `agent/agent_init.py` (1470 строк) извлечены из `run_agent.py`, но цикломатическая сложность остаётся высокой.

3. **Синхронный основной цикл** — `run_conversation()` полностью синхронный, что может ограничивать масштабируемость при множественных параллельных сессиях.

4. **Тесная связанность** — тесты патчат внутренние символы `run_agent` напрямую (OpenAI, handle_function_call, _set_interrupt), что создаёт хрупкость.

5. **Windows-поддержка** — early beta; PTY-зависимые функции (дашборд) требуют WSL2; некоторые POSIX-специфичные конструкции сохраняются.

6. **Удалённый mistralai** — PyPI-пакет quarantined после атаки Mini Shai-Hulud; все mistral-зависимости отключены до снятия карантина.

7. **Сложность конфигурации** — 3 загрузчика конфигурации (CLI, hermes_cli/config.py, gateway), что может вызывать рассинхронизацию.

8. **Глобальное состояние** — `_last_resolved_tool_names` в `model_tools.py` — глобальная переменная процесса, требующая аккуратного сохранения/восстановления при делегировании.

---

## 14. Рекомендации

1. **Продолжить декомпозицию** — дальнейшее извлечение методов из cli.py и hermes_cli/main.py в отдельные модули по доменной области.

2. **Асинхронный цикл** — рассмотреть миграцию основного цикла на async/await для лучшей масштабируемости при параллельных сессиях.

3. **Унификация загрузчиков конфигурации** — свести к единому механизму с чётким контрактом для CLI и gateway.

4. **Улучшить Windows-поддержку** — заменить POSIX-специфичные конструкции на кросс-платформенные аналоги (psutil, pathlib, tempfile).

5. **Интеграционные тесты** — расширить e2e-покрытие (сейчас 5 файлов) для критических путей: шлюз, делегирование, кроны.

6. **Документация API** — формализовать контракты инструментов через TypedDict/Protocol вместо dict-параметров.

7. **Мониторинг** — добавить метрики производительности агента (latency, token usage, tool call success rate) через observability-плагин.

---

## 15. Карта взаимодействий

```
┌──────────────────────────────────────────────────────────────┐
│                      Пользователь                             │
│         CLI / TUI / Telegram / Discord / Slack / ...         │
└──────────┬──────────────────────┬───────────────────────────┘
           │                      │
           ▼                      ▼
    ┌─────────────┐      ┌────────────────┐
    │  cli.py /   │      │   gateway/     │
    │  HermesCLI  │      │   run.py       │
    └──────┬──────┘      └───────┬────────┘
           │                      │
           ▼                      ▼
    ┌─────────────────────────────────────┐
    │         run_agent.py / AIAgent       │
    │   ┌─────────────────────────────┐   │
    │   │  conversation_loop.py       │   │
    │   │  (цикл модель → инструмент)  │   │
    │   └──────────┬──────────────────┘   │
    │              │                       │
    │   ┌──────────▼──────────────────┐   │
    │   │    model_tools.py            │   │
    │   │    (оркестрация инструментов)│   │
    │   └──────────┬──────────────────┘   │
    │              │                       │
    │   ┌──────────▼──────────────────┐   │
    │   │    tools/registry.py         │   │
    │   │    (центральный реестр)      │   │
    │   └──────────┬──────────────────┘   │
    │              │                       │
    │   ┌──────────▼──────────────────┐   │
    │   │  tools/*.py (40+ инструментов)│  │
    │   │  + plugins/ (расширения)      │  │
    │   └─────────────────────────────┘   │
    └─────────────────────────────────────┘
```

---

*Отчёт сгенерирован автоматически на основе анализа кодовой базы.*
