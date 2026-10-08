# goit-pythonweb-hw-02

Домашнє завдання №2 з курсу **FullStack Web Development with Python**.

У цьому завданні готовий FastAPI-застосунок було налаштовано для запуску в Docker-контейнерах разом із PostgreSQL за допомогою Docker Compose.

## Автор

**Fedor Stoietskyi**

---

# Мета завдання

Основною метою було:

- клонувати готовий FastAPI-проєкт;
- створити `Dockerfile`;
- використати Python 3.10;
- створити `docker-compose.yaml`;
- налаштувати PostgreSQL;
- змінити підключення до бази даних для роботи всередині Docker network;
- запустити застосунок через Docker Compose;
- перевірити доступність FastAPI та PostgreSQL.

---

# Використані технології

- Python 3.10
- FastAPI
- Uvicorn
- SQLAlchemy
- PostgreSQL
- psycopg2
- Docker
- Docker Compose

---

# Структура проєкту

```text
goit-pythonweb-hw-02/
│
├── conf/
│   └── db.py
│
├── static/
├── templates/
│
├── .dockerignore
├── Dockerfile
├── docker-compose.yaml
├── main.py
├── readme.md
└── requirements.txt
```

---

# Dockerfile

Для створення Docker image застосунку використовується Python 3.10 відповідно до вимог домашнього завдання.

```dockerfile
FROM python:3.10-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "main.py"]
```

## Чому встановлюються `gcc` та `libpq-dev`

У `requirements.txt` використовується пакет:

```text
psycopg2==2.9.9
```

Для його встановлення всередині Linux Docker image необхідні системні компоненти для компіляції.

`libpq-dev` надає `pg_config`, необхідний для побудови `psycopg2`, а `gcc` використовується для компіляції пакета.

---

# Docker Compose

Для запуску застосунку та бази даних використовується `docker-compose.yaml`.

```yaml
services:
  db:
    image: postgres:15
    container_name: hw02-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: 567234
      POSTGRES_DB: hw02
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d hw02"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    build: .
    container_name: hw02-fastapi
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy

volumes:
  postgres_data:
```

---

# Сервіс PostgreSQL

Сервіс бази даних має ім'я:

```text
db
```

Для нього використовується образ:

```text
postgres:15
```

Основні параметри:

```text
POSTGRES_USER=postgres
POSTGRES_PASSWORD=567234
POSTGRES_DB=hw02
```

Для збереження даних створено Docker volume:

```text
postgres_data
```

Це дозволяє не втрачати дані після звичайного перезапуску контейнера.

---

# Healthcheck PostgreSQL

У конфігурацію PostgreSQL додано:

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres -d hw02"]
  interval: 5s
  timeout: 5s
  retries: 5
```

Це дозволяє Docker Compose визначити момент, коли PostgreSQL дійсно готовий приймати підключення.

FastAPI запускається після того, як база даних переходить у стан `healthy`.

---

# Підключення FastAPI до PostgreSQL

Початковий рядок підключення використовував:

```text
localhost
```

Однак у Docker Compose застосунок і PostgreSQL працюють у різних контейнерах.

Тому `localhost` всередині FastAPI-контейнера означав би сам FastAPI-контейнер, а не PostgreSQL.

Для взаємодії контейнерів використовується ім'я сервісу Docker Compose:

```text
db
```

У файлі:

```text
conf/db.py
```

рядок підключення було змінено на:

```python
SQLALCHEMY_DATABASE_URL = (
    "postgresql+psycopg2://postgres:567234@db:5432/hw02"
)
```

Таким чином:

```text
FastAPI container
        |
        v
       db
        |
        v
PostgreSQL container
```

Docker Compose автоматично забезпечує DNS-resolution для імен сервісів у внутрішній мережі.

---

# Побудова та запуск

Для побудови Docker image та запуску сервісів:

```bash
docker compose up --build
```

При успішному запуску створюються:

- FastAPI container;
- PostgreSQL container;
- Docker network;
- PostgreSQL volume.

---

# Перевірка контейнерів

Стан сервісів можна перевірити командою:

```bash
docker compose ps
```

Очікується, що FastAPI буде у стані `Up`, а PostgreSQL — `Up` та `healthy`.

---

# Перевірка FastAPI

Після запуску застосунок доступний за адресою:

```text
http://localhost:8000
```

FastAPI запускається на:

```text
0.0.0.0:8000
```

а порт Docker-контейнера проброшено на порт `8000` локального комп'ютера.

---

# Перевірка бази даних

У застосунку реалізовано спеціальну перевірку доступності PostgreSQL.

Після відкриття:

```text
http://localhost:8000
```

та натискання кнопки:

```text
Перевірити БД
```

застосунок виконує тестовий запит до PostgreSQL.

При успішному підключенні відображається:

```text
Welcome to FastAPI!
```

Це підтверджує, що:

- FastAPI працює;
- PostgreSQL працює;
- Docker network працює;
- рядок підключення налаштований правильно;
- застосунок успішно підключається до бази даних.

---

# Перевірка endpoint через командний рядок

Endpoint перевірки бази даних також можна викликати безпосередньо:

```bash
curl http://localhost:8000/healthchecker
```

Очікувана відповідь:

```json
{
  "message": "Welcome to FastAPI!"
}
```

---

# Зупинка середовища

Для зупинки Docker Compose:

```bash
docker compose down
```

Ця команда видаляє контейнери та Docker network, але залишає volume з даними PostgreSQL.

Для повного видалення середовища разом із volume можна використати:

```bash
docker compose down -v
```

---

# Основні результати

У межах домашнього завдання було виконано:

- клонування FastAPI-проєкту;
- створення Docker image на основі Python 3.10;
- встановлення Python та системних залежностей;
- конфігурацію PostgreSQL у Docker;
- конфігурацію FastAPI у Docker;
- створення спільної Docker network через Docker Compose;
- налаштування persistent volume для PostgreSQL;
- заміну `localhost` на ім'я сервісу `db`;
- налаштування healthcheck;
- запуск середовища через Docker Compose;
- перевірку FastAPI;
- успішну перевірку підключення до PostgreSQL.

---

# Результат

Застосунок успішно запускається у Docker-середовищі.

При переході на:

```text
http://localhost:8000
```

FastAPI доступний у браузері.

Після натискання кнопки **Перевірити БД** отримано:

```text
Welcome to FastAPI!
```

Це підтверджує успішне виконання завдання та коректну роботу зв'язки:

```text
FastAPI + Docker + Docker Compose + PostgreSQL
```
