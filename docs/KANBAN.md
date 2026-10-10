# KANBAN Board

## ЭПИК 1: Инфраструктура и конфигурация
Базовая настройка проекта, загрузка конфигурации, Docker-окружение

### Задачи

#### [ ] 1.1 Создать модель конфигурации
**Файлы:** `src/core/config.py`

**Описание:**
Реализовать класс `Settings` на основе `pydantic_settings.BaseSettings` для загрузки и валидации конфигурации из переменных окружения.

**Требования:**
- Объявить поля: `BOT_TOKEN: str`, `BOT_OWNER: int`, `BACKUP_PATH: str`, `TIMEZONE: str = "UTC"`, `LOG_LEVEL: str = "INFO"`, `TIMEOUT: int = 60`
- Добавить валидаторы Pydantic v2 (`@field_validator`):
  - `BOT_TOKEN` не пустой (длина > 0)
  - `BOT_OWNER` положительное целое (> 0)
  - `BACKUP_PATH` абсолютный путь (начинается с `/`)
  - `TIMEZONE` валидная временная зона (проверка через `zoneinfo.ZoneInfo`)
- Настроить загрузку из `.env` через `model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")`
- Создать глобальный singleton `settings: Settings = Settings()`

**DoD:**
- [ ] Класс `Settings` создан с валидаторами
- [ ] Создан singleton `settings`
- [ ] Написан unit-тест `tests/unit/core/test_config.py`:
  - Успешная загрузка валидной конфигурации
  - Проверка всех валидаторов (негативные кейсы)
- [ ] Тест проходит

---

#### [ ] 1.2 Настроить систему логирования
**Файлы:** `src/core/logging_config.py`

**Описание:**
Создать централизованную функцию настройки логирования для вывода структурированных логов в stdout.

**Требования:**
- Создать функцию `setup_logging() -> None`
- Формат логов: `%(asctime)s [%(levelname)s] %(name)s: %(message)s`
- Уровень логирования берется из `settings.LOG_LEVEL`
- Вывод в `sys.stdout` (для Docker)
- Использовать `logging.basicConfig()` с параметрами `level`, `format`, `stream`

**DoD:**
- [ ] Функция `setup_logging()` создана
- [ ] Написан unit-тест `tests/unit/core/test_logging_config.py`:
  - Проверка корректности формата логов
  - Проверка уровня логирования
- [ ] Тест проходит

---

#### [ ] 1.3 Инициализировать aiogram бота
**Файлы:** `src/core/bot.py`

**Описание:**
Создать функции для ленивой инициализации singleton-инстансов `Bot` и `Dispatcher`.

**Требования:**
- Создать функцию `get_bot() -> Bot` с глобальной переменной `_bot: Bot | None = None`
- Создать функцию `get_dispatcher() -> Dispatcher` с глобальной переменной `_dispatcher: Dispatcher | None = None`
- Настроить `Bot` с параметрами: `token=settings.BOT_TOKEN`, `default=DefaultBotProperties(parse_mode=ParseMode.HTML)`
- Использовать lazy loading: инициализация при первом вызове

**DoD:**
- [ ] Функции `get_bot()` и `get_dispatcher()` созданы
- [ ] Написан unit-тест `tests/unit/core/test_bot.py`:
  - Проверка singleton (повторный вызов возвращает тот же объект)
  - Проверка `parse_mode=HTML`
- [ ] Тест проходит

---

#### [ ] 1.4 Реализовать получение IP-адреса сервера
**Файлы:** `src/services/system_info.py`

**Описание:**
Создать асинхронную функцию для получения внешнего IP-адреса сервера с retry-политикой и fallback.

**Требования:**
- Создать функцию `async def get_server_ip() -> str`
- Основной метод: GET-запрос к `https://api.ipify.org?format=text` через `httpx.AsyncClient`
- Fallback: если запрос провалился, попытаться получить IP из локальных интерфейсов через `socket.gethostbyname(socket.gethostname())`
- Retry-политика через `tenacity`:
  - `@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=2, max=10))`
- При полном провале вернуть `"Unavailable"`
- Логировать ошибки на уровне `ERROR`

**DoD:**
- [ ] Функция `get_server_ip()` создана
- [ ] Написан unit-тест `tests/unit/services/test_system_info.py`:
  - Мок успешного запроса к API
  - Мок провала API с проверкой fallback
  - Проверка retry-политики (мок tenacity)
- [ ] Тест проходит

---

#### [ ] 1.5 Реализовать работу с временными зонами
**Файлы:** `src/core/datetime_utils.py`

**Описание:**
Создать утилиты для форматирования дат и времени с учетом временной зоны из конфигурации.

**Требования:**
- Создать функцию `get_current_time_formatted() -> str`
  - Получает текущее время через `datetime.now(tz=ZoneInfo(settings.TIMEZONE))`
  - Возвращает строку в формате `DD.MM.YYYY, HH:MM (UTC+3)` (смещение вычисляется автоматически)
- Создать функцию `format_datetime(dt: datetime, include_tz: bool = True) -> str`
  - Конвертирует `dt` в зону `settings.TIMEZONE`
  - Возвращает строку в том же формате

**DoD:**
- [ ] Функции `get_current_time_formatted()` и `format_datetime()` созданы
- [ ] Написан unit-тест `tests/unit/core/test_datetime_utils.py`:
  - Проверка форматирования для разных временных зон (UTC, Europe/Moscow)
  - Проверка корректности смещения
- [ ] Тест проходит

---

#### [ ] 1.6 Подключить Docker API клиент
**Файлы:** `src/services/docker_client.py`

**Описание:**
Создать wrapper над Docker SDK с проверкой доступности daemon и обработкой ошибок.

**Требования:**
- Создать функцию `get_docker_client() -> docker.DockerClient` с глобальной переменной `_client: docker.DockerClient | None = None`
- Инициализация через `docker.from_env(timeout=settings.TIMEOUT)`
- При инициализации выполнить `client.ping()` для проверки доступности
- При ошибке:
  - Логировать критическую ошибку
  - Поднять `DockerUnavailableError("Docker daemon is not available")`

**DoD:**
- [ ] Функция `get_docker_client()` создана
- [ ] Написан unit-тест `tests/unit/services/test_docker_client.py`:
  - Мок успешного подключения
  - Мок провала `client.ping()` с проверкой исключения
  - Проверка singleton
- [ ] Тест проходит

---

#### [ ] 1.7 Обновить зависимости проекта
**Файлы:** `pyproject.toml`

**Описание:**
Добавить все необходимые зависимости для эпика 1 и обновить lock-файл.

**Требования:**
- Зависимости уже добавлены в `pyproject.toml`
- Выполнить `uv sync` для обновления `uv.lock`
- Проверить, что все импорты резолвятся

**DoD:**
- [ ] Выполнена команда `uv sync`
- [ ] Файл `uv.lock` обновлен
- [ ] Проверен запуск `python -c "import aiogram; import pydantic_settings; import docker; import tenacity; import httpx"`

---

#### [ ] 1.8 Написать интеграционный тест инициализации
**Файлы:** `tests/integration/test_initialization.py`

**Описание:**
Создать интеграционный тест, проверяющий полный цикл инициализации всех компонентов эпика 1.

**Требования:**
- Тест `test_full_initialization()`:
  - Загрузить конфигурацию (используя фикстуру с `.env.test`)
  - Вызвать `setup_logging()`
  - Инициализировать бота через `get_bot()`
  - Инициализировать dispatcher через `get_dispatcher()`
  - Инициализировать Docker клиент через `get_docker_client()` (с моком)
  - Проверить, что все объекты созданы корректно
- Использовать фикстуру `tests/conftest.py` для изоляции окружения

**DoD:**
- [ ] Тест `test_full_initialization()` создан
- [ ] Фикстура в `tests/conftest.py` настроена
- [ ] Тест проходит

---

## ЭПИК 2: Graceful Shutdown
Корректное завершение работы при получении SIGTERM/SIGINT: закрытие соединений, flush логов

---

## ЭПИК 3: Healthcheck бота
HTTP-эндпоинт или команда /healthcheck для проверки работоспособности самого бота

---

## ЭПИК 4: Авторизация и безопасность
Middleware для проверки BOT_OWNER, защита всех команд от неавторизованных пользователей

---

## ЭПИК 5: Базовые команды бота
Команда /start с приветствием и отображением IP/времени

---

## ЭПИК 6: Мониторинг состояния сервера
Команда /health: метрики RAM, CPU, диска, uptime, топ-процессы

---

## ЭПИК 7: Мониторинг бэкапов БД
Команда /backups: парсинг файлов дампов, ротация, валидация размера, проверка синхронности, алерты

---

## ЭПИК 8: Мониторинг Docker-контейнеров
Команда /docker_ps, подписка на Docker Events API, обработка падений/стартов контейнеров, батчинг событий, троттлинг уведомлений

---

## Легенда статусов
- `[ ]` TODO — задача не начата
- `[~]` IN PROGRESS — задача в работе
- `[T]` TESTING — задача на тестировании
- `[R]` REVIEW — задача на код-ревью
- `[x]` DONE — задача завершена
