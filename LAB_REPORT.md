# Лабораторная работа №5: CI/CD с GitHub Actions

## Выполненные задания

### 4.1 Подготовка

- [x] Создан репозиторий на GitHub
- [x] Создана и переключена ветка `dev`
- [x] Настроены GitHub Actions workflows

### 4.2 Покрытие тестами

В файле `tests/test_user.py` реализованы 4 теста для API пользователей.

#### Структура тестового файла

```python
from fastapi.testclient import TestClient
from src.main import app

client = TestClient(app)

# Тестовые данные — существующие пользователи
users = [
    {'id': 1, 'name': 'Ivan Ivanov', 'email': 'i.i.ivanov@mail.com'},
    {'id': 2, 'name': 'Petr Petrov', 'email': 'p.p.petrov@mail.com'}
]
```

---

#### Тест 1: `test_get_existed_user()`

**Назначение:** Проверка получения существующего пользователя по email.

```python
def test_get_existed_user():
    '''Получение существующего пользователя'''
    response = client.get("/api/v1/user", params={'email': users[0]['email']})
    assert response.status_code == 200
    assert response.json() == users[0]
```

**Проверяет:**
- Статус-код `200 OK`
- Возвращаемые данные совпадают с ожидаемыми

---

#### Тест 2: `test_get_unexisted_user()`

**Назначение:** Проверка получения несуществующего пользователя.

```python
def test_get_unexisted_user():
    '''Получение несуществующего пользователя'''
    response = client.get("/api/v1/user", params={'email': 'nonexistent@example.com'})
    assert response.status_code == 404
    assert response.json() == {"detail": "User not found"}
```

**Проверяет:**
- Статус-код `404 Not Found`
- Сообщение об ошибке `{"detail": "User not found"}`

---

#### Тест 3: `test_create_user_with_valid_email()`

**Назначение:** Проверка создания пользователя с уникальной почтой.

```python
def test_create_user_with_valid_email():
    '''Создание пользователя с уникальной почтой'''
    new_user = {
        'name': 'Sidor Sidorov',
        'email': 's.s.sidorov@mail.com'
    }
    response = client.post("/api/v1/user", json=new_user)
    assert response.status_code == 201
    assert isinstance(response.json(), int)
```

**Проверяет:**
- Статус-код `201 Created`
- Возвращается `int` (ID нового пользователя)

---

#### Тест 4: `test_create_user_with_invalid_email()`

**Назначение:** Проверка создания пользователя с дублирующимся email.

```python
def test_create_user_with_invalid_email():
    '''Создание пользователя с почтой, которую использует другой пользователь'''
    duplicate_user = {
        'name': 'Duplicate User',
        'email': users[0]['email']
    }
    response = client.post("/api/v1/user", json=duplicate_user)
    assert response.status_code == 409
    assert response.json() == {"detail": "User with this email already exists"}
```

**Проверяет:**
- Статус-код `409 Conflict`
- Сообщение об ошибке о существующем пользователе

---

#### Тест 5: `test_delete_user()`

**Назначение:** Проверка удаления пользователя.

```python
def test_delete_user():
    '''Удаление пользователя'''
    user_to_delete = {
        'name': 'Delete Me',
        'email': 'delete.me@example.com'
    }
    create_response = client.post("/api/v1/user", json=user_to_delete)
    assert create_response.status_code == 201
    
    delete_response = client.delete("/api/v1/user", params={'email': user_to_delete['email']})
    assert delete_response.status_code == 204
```

**Проверяет:**
- Статус-код `201 Created` при создании
- Статус-код `204 No Content` при удалении

---

### Таблица статус-кодов API

| Метод | Endpoint | Описание | Статус-коды |
|-------|----------|----------|-------------|
| GET | `/api/v1/user?email=...` | Получение пользователя | 200, 404 |
| POST | `/api/v1/user` | Создание пользователя | 201, 409 |
| DELETE | `/api/v1/user?email=...` | Удаление пользователя | 204 |

---

### 4.3 Создание пайплайна

Создана директория `.github/workflows` с двумя файлами.

---

#### Файл 1: `tests.yml`

**Назначение:** Запуск unit-тестов при любом коммите.

```yaml
name: Test Python App

on:
  push:  # Запуск при любом пуше

jobs:
  ci:
    runs-on: ubuntu-latest  # Тип машины
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest httpx

      - name: Run tests
        run: python -m pytest tests/
```

**Параметры:**
- Имя workflow: `Test Python App`
- Триггер: любой `push` в репозиторий
- Машина: `ubuntu-latest`
- Python версия: `3.12`

---

#### Файл 2: `build-and-delivery.yml`

**Назначение:** Сборка Docker-образа и доставка в Docker Hub при пуше в `main`.

```yaml
name: Build and Delivery

on:
  workflow_run:
    workflows:
      - Test Python App
    branches:
      - main
    types:
      - completed

jobs:
  cd:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: ${{ vars.DOCKERHUB_USERNAME }}/my-app:latest
```

**Параметры:**
- Имя workflow: `Build and Delivery`
- Триггер: завершение `Test Python App` в ветке `main`
- Условие запуска: `if: ${{ github.event.workflow_run.conclusion == 'success' }}`
- Машина: `ubuntu-latest`
- Задание: `cd`

**Шаги:**
1. `docker/login-action@v3` — логин в Docker Hub
2. `docker/build-push-action@v6` — сборка и пуш образа

---

### Необходимые переменные и секреты

Для работы пайплайна `build-and-delivery.yml` требуется настроить в GitHub:

| Тип | Имя | Описание |
|-----|-----|----------|
| **Variable** | `DOCKERHUB_USERNAME` | Логин Docker Hub |
| **Secret** | `DOCKERHUB_TOKEN` | Токен доступа Docker Hub |

**Настройка:**
1. Settings → Secrets and variables → Actions
2. Вкладка **Variables**: добавить `DOCKERHUB_USERNAME`
3. Вкладка **Secrets**: добавить `DOCKERHUB_TOKEN`

**Создание токена Docker Hub:**
1. https://hub.docker.com/settings/security
2. New Access Token
3. Права: **Read & Write**
4. Скопировать токен (показывается один раз)

---

### Структура проекта

```
devops-lab5/
├── .github/workflows/
│   ├── tests.yml              # CI пайплайн для тестов
│   └── build-and-delivery.yml # CD пайплайн для доставки
├── src/
│   ├── main.py                # Точка входа приложения
│   ├── settings.py            # Настройки
│   ├── routers/
│   │   └── user.py            # API пользователей
│   ├── schemas/
│   │   └── user.py            # Pydantic схемы
│   └── fake_db/
│       └── database.py        # Фейковая база данных
├── tests/
│   └── test_user.py           # Unit-тесты API
├── Dockerfile                 # Docker образ
├── requirements.txt           # Зависимости Python
└── LAB_REPORT.md              # Этот файл
```

---

### Результат

✅ Все 5 тестов проходят успешно  
✅ CI пайплайн запускается при любом пуше  
✅ CD пайплайн запускается после успешных тестов в `main`  
✅ Docker-образ собирается и публикуется в Docker Hub  

**Ссылка на Docker Hub:** https://hub.docker.com/r/starshulgin1/my-app
