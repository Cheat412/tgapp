# Telegram bot MVP — readiness interview (telegram-architect)

## 1) Assumptions

- Продукт: **Telegram Bot API** (бот в чатах), не standalone **Telegram Client** / **TDLib**.
- **MVP**: long polling, один процесс, без отдельного **webhook**-хостинга и **SSL** на старте.
- Язык и рантайм: **Python 3.12+**, фреймворк **aiogram 3.x**, зависимости через **uv** (или **pip** + **venv**).
- Деплой позже: **VPS** / **Docker**; сейчас достаточно локального **first run**.
- Хранение состояния **MVP**: in-memory или по необходимости **SQLite** (один файл), без **Redis** до появления нагрузки.
- Секреты только в **`.env`**, не в репозитории; значения токенов в документации не дублируем.

## 2) Architecture / stack recommendation (MVP)

- **Entrypoint**: один модуль запуска (`main` / `__main__`), регистрация **router**-ов **aiogram**.
- **Handlers**: по доменам (например `handlers/start.py`, `handlers/common.py`).
- **Config**: загрузка из **environment** + валидация обязательных имён переменных (без логирования значений секретов).
- **Logging**: структурированный или plain **logging** с уровнем из **env**.
- **Error boundary**: глобальный **error handler** в **Dispatcher**, чтобы сбои в **handler** не роняли процесс без записи в лог.
- Масштабирование позже: переход на **webhook** + reverse proxy, вынос **FSM**/сессий в **Redis** при необходимости.

## 3) Bootstrap and first-run command checklist

- Установить **Python 3.12+** и **uv** (или использовать **venv** + **pip**).
- В корне проекта: `uv init` (или создать **pyproject.toml** вручную).
- `uv add aiogram python-dotenv` (или эквивалент в **pip**).
- Создать **`.env`** из **`.env.example`** (имена переменных без значений в репо).
- Реализовать минимальный **bot** + **Dispatcher** + **polling**.
- Первый запуск: `uv run python -m telegram.bot` (или согласованный путь к пакету после фиксации структуры).
- Проверка: в логе успешный **getMe** (или эквивалент после старта **long polling** без ошибок авторизации).

## 4) Proposed folder structure

```text
Date/MyTelegramProject/
├── .env                    # локально, в .gitignore
├── .env.example            # только имена переменных
├── pyproject.toml
├── telegram/
│   ├── README              # этот файл
│   └── bot/
│       ├── __init__.py
│       ├── __main__.py     # точка входа: uv run python -m telegram.bot
│       ├── config.py       # env, без секретов в коде
│       ├── main.py         # сборка bot, dispatcher, routers
│       ├── handlers/
│       │   └── start.py
│       └── middlewares/    # опционально, по мере роста
```

При необходимости позже: `tests/`, `docker/`, `scripts/`.

## 5) Required env variable NAMES only

- **`BOT_TOKEN`** — обязательно для **Bot API**.
- Опционально для **MVP**: **`LOG_LEVEL`**, **`BOT_USERNAME`** (если нужен для ссылок/текстов без лишних запросов).

Для будущего **webhook** (не MVP): **`WEBHOOK_URL`**, **`WEBHOOK_PATH`**, **`WEBHOOK_SECRET`**, **`PORT`**.

## 6) Smoke tests and minimal test strategy

- **Manual smoke**: в Telegram отправить **`/start`** — ожидаемый ответ и отсутствие traceback в логах.
- **API smoke** (один раз при настройке CI или локально): вызов **getMe** через **HTTP** к **Bot API** с тем же токеном (скрипт или curl) — проверка валидности токена без запуска бота.
- **Automated minimal**: один тест с **mock** входящего **Update** для **handler** **`/start`** (например **pytest**), без реальной сети.
- Регрессии **MVP**: только критичные пути (старт, необработанные ошибки → лог, не silent fail).

## 7) Key risks and fallback options

- **Утечка `BOT_TOKEN`**: строгий **`.gitignore`**, ротация токена в **BotFather**, никаких токенов в логах.
- **Long polling и перезапуски**: кратковременные дубликаты **update** — идемпотентные **handlers** или учёт **update_id** при введении побочных эффектов.
- **Рост нагрузки**: узкое место — один процесс; fallback — горизонтальное масштабирование только с **webhook** + общим хранилищем состояния.
- **Зависимость от сети Telegram**: ретраи на уровне **HTTP client** осторожно; при недоступности API — backoff и алерты по логам.

## 8) Verdict: ready / not-ready for coding

**Ready** — при принятии стека **Python + aiogram + long polling** и фиксации структуры каталога `telegram/bot/`. Блокеров для старта кодирования **MVP** нет.

## 9) Next minimal step

Инициализировать **pyproject.toml**, добавить зависимости (**aiogram**, **python-dotenv**), создать **`.env.example`** с именами переменных и пустой каркас **`telegram/bot/`** с **`__main__.py`** и **handler** для **`/start`**, затем первый успешный **local run** с **`BOT_TOKEN`** в **`.env`**.
