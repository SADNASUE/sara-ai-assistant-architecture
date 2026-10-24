# Sara — Автономный AI-ассистент (Архитектурный Case Study)

![Status](https://img.shields.io/badge/Status-Active_Development-blue)
![Python](https://img.shields.io/badge/Python-3.12-yellow)
![FastAPI](https://img.shields.io/badge/FastAPI-Async-green)
![LLM](https://img.shields.io/badge/LLM-Ollama_%7C_Cloud-orange)
![Memory](https://img.shields.io/badge/Memory-RAG_%7C_SQLite-purple)
![Architecture](https://img.shields.io/badge/Architecture-Agentic-red)

> ⚠️ **Внимание:** Исходный код проекта является **закрытым** (Private Repository) и представляет коммерческую ценность. Данный репозиторий — это подробный архитектурный обзор (Case Study). Здесь описан дизайн системы, стек технологий и инженерные решения, демонстрирующие подход к построению сложных AI-продуктов.

## 🌌 Общее описание

**Sara** — это локальный голосовой AI-ассистент для Windows с долговременной памятью, фоновой «жизнью» (сон, автономия, инициатива), управлением компьютером, программистской песочницей и визуальным аватаром. 

Работает как клиент-серверная система: FastAPI-сервер (порт настраивается через `SARA_PORT`), клиенты — браузерный UI, PySide6-оверлей с микрофоном, Electron-аватар и Electron dev-окно. Внешние зависимости — только OpenAI-совместимые LLM (облако или локальная Ollama); память, STT и TTS работают строго локально.

---

## 🛠 Технологический стек

*   **Backend:** Python 3.12, FastAPI, uvicorn, SSE, WebSocket, SQLite (WAL + FTS5), NumPy/SciPy.
*   **LLM:** OpenAI-совместимый слой. Роли: `chat` (qwen/qwen3.7-flash), `code` (openai/gpt-6-luna-pro, reasoning on), `sleep` (deepseek-v4-flash), `local` (Ollama, sara-qwen). Reasoning управляется `extra_body` (`enable_thinking` для routerai, `reasoning.enabled` для polza). Провайдер выбирается полем `provider` без смены кода.
*   **Память:** SQLite (`memory.db`, WAL, `busy_timeout=3000`) + FTS5 (`memory_fts`) + векторы в BLOB (`memory_vectors`). Эмбеддинги: локальный `intfloat/multilingual-e5-small` (384) по умолчанию, облачный `baai/bge-m3` (1024) как fallback.
*   **Речь:** faster-whisper (`medium`, ctranslate2, CUDA или CPU) — STT; Silero TTS (`v4_ru`, локально) и Inworld TTS (облако) — синтез; `sounddevice`/`soundfile` — аудио.
*   **Клиенты:** PySide6 (оверлей), `pynput` + Win32 `RegisterHotKey` (два независимых канала хоткея), Electron + three.js + `@pixiv/three-vrm` (VRM-аватар).
*   **DevOps:** `installer.py` (диагностика железа, GPU-aware torch, venv-only), `build_release.py` (чистая сборка без ключей и памяти).

---

## 🧠 Архитектурные блоки

```text
UI (браузер) ─┐
Оверлей (PySide6) ─┼─ HTTP / SSE ─→ Server (FastAPI, :8765)
Аватар (Electron) ─┘                     │
                                         ├─ LLM-роутер (chat / code / local / sleep)
                                         ├─ Prompt builder + бюджет контекста
                                         ├─ Tool kernel (25 инструментов)
                                         ├─ Память (SQLite/FTS5/векторы, recall, reconcile)
                                         ├─ Сон (micro / light / deep)
                                         ├─ Автономия (инициатива, проактивность)
                                         ├─ Профиль (16 флагов, пресеты)
                                         ├─ Dev-песочница (Workspace, git, share/import)
                                         └─ TTS / STT / зрение / скриншоты
```

**1. Server (FastAPI)**
Единая точка входа: `/chat` (JSON), `/chat_stream` (SSE: `delta`/`status`/`done`), `/health`, `/settings`, `/toggles`, `/api/workspace/*`, `/api/dev/*`, `/api/diag`. 
*Логика хода:* сборка промпта → цикл LLM с tool calls → финализация (запись в сессию, память, сон, датасет).

**2. Память (RAG)**
Хранит факты, сущности, связи, эпизоды, треды, паттерны, дневник, provenance и противоречия. 
*Гибридный recall:* FTS5 (BM25) + подстрочный поиск + векторный канал, с дедупликацией и порогами (z-score + абсолютный). 
*Консолидация:* происходит в light/deep сне; разрешение противоречий — через `pick_winner`/reconciliation.

**3. Сон (Фоновая обработка)**
Три уровня: 
*   `micro` (разбор хода в фоне);
*   `light` (сессия → факты);
*   `deep` (сжатие и пересборка памяти, reasoning off). 
Планировщик учитывает энергию/долг сна; запуски идемпотентны по `run_id`, без потери данных при паузах.

**4. Автономия**
Драйвы и настроение, импульсы, гейт инициативы, проактивные сообщения. Может полностью отключаться через профиль.

**5. Tools (Инструменты)**
25 схем: память (чтение/запись/дневник/треды), задачи, программы, `dev_workspace`, `screen_capture`, `research`, `avatar`, `set_mood` и др. 
Вызовы проходят через единый kernel; результаты нормализуются (потерянный `tool-result` заменяется явной пометкой).

**6. Dev Sandbox (Песочница)**
`Workspace` рядом с `brain`: действия `new/write/append/patch/read/list/run/exec/search/delete/open_window`. Жесткие лимиты файла/проекта без зависимостей, запрет выхода за пределы Workspace. 
*Code-профиль:* 32k токенов, 200k символов контекста, 12 раундов, 900 с, денежный лимит 5 ₽, авто-стоп по отсутствию прогресса. 
*Локальный git:* авто-коммиты после правок, revert/undo с бэкапом. 
*Share/Import:* приватный GitHub-репозиторий или ZIP с манифестом; доустановка только недостающих зависимостей (npm/pip).

**7. Профиль (Управление)**
16 флагов (память, фон, сон, инициатива, проактивность, задачи, установка/удаление/запуск программ, программирование, настроение, поиск, картинки, голос). Три пресета + пользовательские; применяются на лету без перезапуска.

---

## 🚀 Ключевые фичи

*   **Tool calling:** OpenAI function calling, многораундовый цикл, принудительный `tool_choice` для критичных операций (удаление/правки), проверка факта изменений (не «готово», пока инструмент не подтвердил).
*   **SSE-стриминг:** `delta`/`status`/`done`, остановка хода `/chat/stop`, ретраи клиентов при перезапуске сервера (3 попытки, 0.5/1/2 с) без дублей текста.
*   **Контекст:** бюджет в символах с приоритетами блоков и резервами на ответ/инструменты; стабильный префикс промпта для prompt caching.
*   **Векторная память (RAG):** e5/bge-m3, косинус с двухэшелонным отбором, прогревание индекса, предохранитель при сбоях (5 ошибок → пауза 300 с); карта проекта (PageRank по импортам, 9k символов) и hash-кэш чтения файлов.
*   **Мульти-модельность:** chat/code/local/sleep, выбор провайдера, reasoning off/on на уровне вызова.
*   **Надёжность:** retry на 429 (2/4 с) с человеческим ответом вместо 500, truth-сверка, очистка сирот и single-instance, WAL, retention логов и бэкапов.

---

## 🔥 Решённые инженерные челленджи

### Блокировка Event Loop (AsyncIO)
Все синхронные операции (SQLite, LLM, файлы) вынесены из корутин (`asyncio.to_thread`); SQLite в WAL с `busy_timeout=3000`; семантические эмбеддинги считаются до транзакции с жёстким лимитом 200 мс; watchdog `_MS_INFLIGHT` (900 с) снимает флаг только у мёртвого потока, не у живого запроса.

### Бюджет токенов и контекста
Бюджет считается в символах (~3 символа/токен) с приоритетами блоков (характер → собеседник → задача → состояние → память), резервами на ответ и инструменты; для dev — 200k символов и 32k токенов. Фактический остаток под `tool-result` считается отдельно, чтобы не обрезать результат в ноль. Prompt caching измеряется по `usage.prompt_tokens_details.cached_tokens` (живые замеры: 35–71% входа).

### Ошибки LLM и отказоустойчивость
429 обрабатывается retry (2/4 с) и превращается в человеческое сообщение, а не 500; обрывы `tool-pairs` нормализуются placeholder-ом; клиенты ретраят `ConnectionError` при перезапуске; reasoning-модели ограничены по `max_tokens`/`extra_body`.

### Консистентность памяти
Противоречия фиксируются `provenance`/`contradiction_edges`, разрешаются `pick_winner`; reconciliation идёт курсором, не блокируя recall; семантический канал имеет breaker; сон идемпотентен по `run_id`, энергия кредитуется с pending-retry.

### Долгие ходы и контроль денег
Стриминг + кнопка «Стоп»; авто-стопы: лимит стоимости хода (5 ₽), 8 раундов без прогресса, 12 раундов, 900 с; dev-промпт сжат с ~29k до ~2.9k символов без потери правил.

### Переносимость и сборка
`installer.py` определяет GPU и compute capability и выбирает сборку torch (cu128/cu126/cu124/cu121/cpu), ставит только в `.venv`, требует Python 3.12, предзагружает модели (Silero/whisper/e5), ставит portable Node при отсутствии. `Workspace` создаётся рядом с `brain`. `build_release.py` собирает бету по белому списку, вычищает ключи и сканирует утечки.

---

## ⚙️ Запуск (кратко)

```bash
install.bat   # диагностика, .venv, torch под GPU, модели, Node/npm, шаблоны
sara.bat      # сервер + оверлей + UI; первый запуск спросит имя и ключ
```

**Конфигурация:** 
`.env` (ключи, `SARA_PORT`, `SARA_STT_DEVICE`, `SARA_SEMANTIC_BACKEND`), `server/config.json` (LLM/TTS/профиль), `brain/config.json` (сон, имя), `overlay/config.json` (хоткеи).

---
*Автор: [Анатолий Михайлович](https://github.com/SADNASUE) — AI / Backend Developer*
