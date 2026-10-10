# Архитектура проекта

Проект реализует телеграм-бота для управления удаленным сервером (VPS). Впрочем, управлять он может любой машиной (хостом), на которой развернут. Управление реализуется при помощи Docker API для системных вызовов.

## Стек технологий

- **Python 3.12** — основной язык разработки
- **Aiogram 3.x** — фреймворк для работы с Telegram Bot API
- **Pydantic Settings** — управление конфигурацией с валидацией
- **Docker SDK for Python** — взаимодействие с Docker API
- **Tenacity** — retry-политики для внешних вызовов
- **psutil** — сбор системных метрик (CPU, RAM, процессы)
- **httpx** — асинхронные HTTP-запросы

## Структура проекта

```
src/
├── core/                      # Ядро приложения
│   ├── __init__.py           # Точка входа: main()
│   ├── config.py             # Модель конфигурации (pydantic-settings)
│   ├── logging_config.py     # Настройка системы логирования
│   ├── bot.py                # Инициализация aiogram Bot и Dispatcher
│   ├── datetime_utils.py     # Утилиты для работы с временными зонами
│   └── exceptions.py         # Кастомные исключения проекта
│
├── services/                  # Бизнес-логика и внешние интеграции
│   ├── system_info.py        # Получение IP-адреса и системных метрик
│   ├── docker_client.py      # Wrapper над Docker API
│   ├── backup_monitor.py     # Мониторинг бэкапов БД (эпик 7)
│   └── docker_monitor.py     # Мониторинг Docker-контейнеров (эпик 8)
│
├── handlers/                  # Telegram command handlers
│   ├── start.py              # Команда /start
│   ├── health.py             # Команда /health (метрики сервера)
│   ├── backups.py            # Команда /backups (мониторинг бэкапов)
│   └── docker_ps.py          # Команда /docker_ps (список контейнеров)
│
├── middlewares/               # Aiogram middlewares
│   └── auth.py               # Проверка BOT_OWNER (эпик 4)
│
└── keyboards/                 # Inline-клавиатуры (если потребуется)
    └── common.py
```

## Модули эпика 1: Инфраструктура

### core/config.py
**Назначение:** Загрузка и валидация конфигурации из переменных окружения.

**Компоненты:**
- `Settings` (pydantic BaseSettings) — модель конфигурации с полями:
  - `BOT_TOKEN: str` — токен Telegram Bot API (обязательно)
  - `BOT_OWNER: int` — Telegram user ID владельца (обязательно, > 0)
  - `BACKUP_PATH: str` — абсолютный путь к директории бэкапов (обязательно)
  - `TIMEZONE: str` — временная зона (по умолчанию "UTC", валидация через zoneinfo)
  - `LOG_LEVEL: str` — уровень логирования (по умолчанию "INFO")
  - `TIMEOUT: int` — таймаут операций в секундах (по умолчанию 60)
- `settings: Settings` — глобальный singleton для импорта в других модулях

**Валидаторы:**
- `BOT_TOKEN` не пустой
- `BOT_OWNER` положительное целое
- `BACKUP_PATH` абсолютный путь (начинается с `/`)
- `TIMEZONE` валидная временная зона (проверка через zoneinfo)

---

### core/logging_config.py
**Назначение:** Централизованная настройка системы логирования.

**Компоненты:**
- `setup_logging() -> None` — конфигурирует logging с форматом:
  - Timestamp (ISO 8601)
  - Level (INFO, ERROR, DEBUG)
  - Module name
  - Message
- Вывод логов в stdout (для Docker)
- Уровень логирования задается через `LOG_LEVEL` из конфигурации

---

### core/bot.py
**Назначение:** Инициализация aiogram бота и dispatcher.

**Компоненты:**
- `get_bot() -> Bot` — lazy singleton для экземпляра Bot
- `get_dispatcher() -> Dispatcher` — lazy singleton для Dispatcher
- По умолчанию `parse_mode='HTML'` для всех сообщений

**Принципы:**
- Бот инициализируется при первом обращении (lazy loading)
- Токен берется из `settings.BOT_TOKEN`

---

### core/datetime_utils.py
**Назначение:** Работа с временными зонами и форматирование дат.

**Компоненты:**
- `get_current_time_formatted() -> str` — текущее время в формате `DD.MM.YYYY, HH:MM (UTC+3)`
- `format_datetime(dt: datetime, fmt: str) -> str` — форматирование произвольных дат
- Использует `zoneinfo` (встроенная в Python 3.9+)
- Временная зона берется из `settings.TIMEZONE`

---

### core/exceptions.py
**Назначение:** Кастомные исключения проекта.

**Компоненты:**
- `DockerUnavailableError` — поднимается при недоступности Docker daemon
- `BackupParsingError` — ошибка парсинга файлов бэкапов (эпик 7)
- `UnauthorizedError` — попытка доступа от неавторизованного пользователя (эпик 4)

---

### services/system_info.py
**Назначение:** Получение информации о системе и сервере.

**Компоненты:**
- `get_server_ip() -> str` — внешний IP-адрес сервера
  - Основной метод: запрос к `https://api.ipify.org`
  - Fallback: парсинг локальных сетевых интерфейсов
  - Retry-политика: 3 попытки, экспоненциальный backoff (tenacity)
  - При провале возвращает `"Unavailable"`

**Будущие функции (эпик 6):**
- `get_ram_metrics()` — метрики оперативной памяти
- `get_cpu_metrics()` — метрики процессора и load average
- `get_disk_metrics()` — метрики диска
- `get_uptime()` — uptime системы
- `get_top_processes()` — топ-5 процессов по RAM/CPU

---

### services/docker_client.py
**Назначение:** Wrapper над Docker API для безопасного доступа к Docker daemon.

**Компоненты:**
- `get_docker_client() -> docker.DockerClient` — singleton для Docker клиента
- Проверка доступности daemon при инициализации
- Поднимает `DockerUnavailableError` при недоступности Docker

**Будущие функции (эпик 8):**
- `list_containers()` — список всех контейнеров
- `get_container_logs()` — получение логов контейнера
- `subscribe_to_events()` — подписка на Docker Events API

---

## Паттерны и принципы

### Управление конфигурацией
- **Единственный источник правды:** `core/config.py` с моделью `Settings`
- Все переменные окружения загружаются через pydantic-settings
- Валидация на старте приложения (fail-fast)
- Глобальный singleton `settings` импортируется в модулях

### Обработка ошибок
- Кастомные исключения для бизнес-логики (`core/exceptions.py`)
- Логирование ошибок на уровне `ERROR`
- Уведомление пользователя о критических ошибках через Telegram
- Graceful degradation: если внешний сервис недоступен — fallback или пропуск

### Retry-политики
- Модуль `core/utils/retry.py` с настройками tenacity (эпик 8)
- Экспоненциальный backoff: 2, 4, 8, 10 секунд
- Максимум 3 попытки для внешних API
- Поддержка заголовка `Retry-After` при HTTP 429

### Логирование
- Структурированные логи с timestamp, level, module
- Уровень `INFO` для продакшена, `DEBUG` для разработки
- Вывод в stdout для интеграции с Docker logs
- Критические ошибки дублируются в Telegram (уведомление владельцу)

### Безопасность
- Middleware для проверки `BOT_OWNER` (эпик 4)
- Все команды доступны только авторизованному пользователю
- Docker доступ через монтирование `/var/run/docker.sock` (read-only в будущем)
- Доступ к файловой системе хоста через `/mnt/host` (минимизация прав)

---

## Deployment

### Docker-конфигурация
```yaml
bot:
  user: "0:0"  # Root для доступа к Docker socket
  volumes:
    - /:/mnt/host:ro  # Read-only доступ к ФС хоста
    - /var/run/docker.sock:/var/run/docker.sock  # Docker API
  environment:
    - BOT_TOKEN=${BOT_TOKEN}
    - BOT_OWNER=${BOT_OWNER}
    - BACKUP_PATH=${BACKUP_PATH}
    - TIMEZONE=${TIMEZONE}
    - LOG_LEVEL=${LOG_LEVEL}
```

### Graceful Shutdown (эпик 2)
- Обработка `SIGTERM` и `SIGINT`
- Закрытие aiogram polling
- Закрытие Docker API клиента
- Flush логов перед завершением

---

## Тестирование

### Unit-тесты
- Каждая функция модуля покрывается unit-тестом
- Внешние API мокаются (httpx, docker)
- Используются фикстуры для изоляции окружения

### Интеграционные тесты
- Проверка полного цикла инициализации приложения
- Проверка обработки команд с реальным dispatcher (без отправки в Telegram)
- Docker API мокается через `docker.from_env()`

### Конфигурация pytest
- `asyncio_mode = "auto"` в `pyproject.toml`
- Фикстуры в `tests/conftest.py`
- Разделение на `tests/unit/` и `tests/integration/`
