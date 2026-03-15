# Отчёт по отладке приложения

**ФИО:** Галеев Тамирлан Рустамович

---

## Шаг 1: Исправление конфигурации базы данных

Не получалось поднять docker, проверил файл конфигурации `config.py` и обнаружил ошибку в переменной окружения для строки подключения к базе данных.

**Файл:** `config.py`

**Код до:**

```py
database_url: str = Field(
    "postgresql+asyncpg://postgres:postgres@db:5432/postgres_typo",
    validation_alias="DATABSE_URL",
)
```

**Код после:**

```py
database_url: str = Field(
    "postgresql+asyncpg://postgres:postgres@db:5432/postgres",
    validation_alias="DATABASE_URL",
)
```

---

## Шаг 2: Исправление обработки пустых значений в парсере

При обработке данных из внешнего API некоторые поля могли отсутствовать (`None`), что приводило к ошибкам обращения к атрибутам.

**Файл:** `parser.py`

**Код до:**

```py
"timetable_mode_name": item.timetable_mode.name,
"tag_name": item.tag.name,
"city_name": item.city.name.strip(),
```

**Код после:**

```py
"timetable_mode_name": item.timetable_mode.name if item.timetable_mode else "Не указан",
"tag_name": item.tag.name if item.tag else "Не указан",
"city_name": item.city.name.strip() if item.city else "Не указан",
```

---

## Шаг 3: Исправление интервала планировщика

**Описание проблемы:**

Планировщик запускал задачу с интервалом в 5 секунд, хотя в задании было сказано 5 минут.

**Файл:** `scheduler.py`

**Код до:**

```py
minutes=settings.parse_schedule_minutes
```

**Код после:**

```py
seconds=settings.parse_schedule_minutes
```

---

## Шаг 4: Исправление работы HTTP-клиента

**Описание проблемы:**

HTTP-клиент создавался без использования контекстного менеджера, что могло привести к утечке соединений. Контекстный менеджер автоматически закрывает соединения после завершения работы клиента и предотвращает утечки соединений.

**Файл:** `parser.py`

**Код до:**

```py
client = httpx.AsyncClient(timeout=timeout)
```

**Код после:**

```py
async with httpx.AsyncClient(timeout=timeout) as client:
```

---

## Шаг 5: Исправление структуры данных для хранения ID

**Описание проблемы:**

Для хранения идентификаторов использовался словарь, хотя фактически требовалась структура для хранения уникальных значений.

**Файл:** `vacancy.py`

**Код до:**

```py
existing_ids = {}
```

**Код после:**

```py
existing_ids = set()
```

---

## Шаг 6: Исправление обработки дубликатов вакансий

**Описание проблемы:**

При попытке создать вакансию с уже существующим `external_id` сервер возвращал `status.HTTP_200_OK`.

**Файл:** `vacancies.py`

**Код до:**

```py
return JSONResponse(
    status_code=status.HTTP_200_OK,
    content={"detail": "Vacancy with external_id already exists"},
)
```

**Код после:**

```py
raise HTTPException(
    status_code=status.HTTP_409_CONFLICT,
    detail="Vacancy with external_id already exists",
)
```

---

## Итог

- Найдены и исправлены ошибки конфигурации приложения.
- Исправлены потенциальные ошибки обработки данных из внешнего API.
- Улучшена работа HTTP-клиента и предотвращены возможные утечки соединений.
- Исправлена структура данных для хранения идентификаторов.
- Реализована корректная обработка дубликатов вакансий.

В результате приложение работает стабильнее и корректно обрабатывает входящие запросы и данные.

---

# Selectel Vacancies API

FastAPI-приложение для парсинга публичных вакансий Selectel, хранения в PostgreSQL и предоставления CRUD API.

## Быстрый старт

1. Клонируйте репозиторий (или распакуйте проект из архива):
   `git clone --branch with-bugs https://github.com/selectel/be-test.git`
2. Создайте `.env` на основе примера:
   `cp .env.example .env`
3. Примените переменные окружения `.env`:
   `source .env`
4. Запуск через Docker Desktop:
   `docker compose up --build`
5. Проверка работоспособности:
   откройте `http://localhost:8000/docs`
6. Остановка и очистка:
   `docker-compose down -v`

## Переменные окружения

- `DATABASE_URL` — строка подключения к PostgreSQL.
- `LOG_LEVEL` — уровень логирования (`INFO`, `DEBUG`).
- `PARSE_SCHEDULE_MINUTES` — интервал фонового парсинга в минутах.

## Основные эндпоинты

- `GET /api/v1/vacancies/` — список вакансий
- `GET /api/v1/vacancies/{id}` — детали вакансии.
- `POST /api/v1/vacancies/` — создание вакансии.
- `PUT /api/v1/vacancies/{id}` — обновление вакансии.
- `DELETE /api/v1/vacancies/{id}` — удаление вакансии.
- `POST /api/v1/parse/` — ручной запуск парсинга.

## Примечания

- При старте приложения выполняется первичный парсинг.
- Фоновый парсинг запускается планировщиком APScheduler (в рамках заданного интервала).
