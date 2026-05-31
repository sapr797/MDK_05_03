# Лекция 3.9: Тестирование функций и модулей. Организация тестов в проекте

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.6:** Покрытие кода и тестирование функций  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить принципы тестирования различных типов функций (чистые функции, функции с побочными эффектами), а также освоить эффективные способы организации тестов в проекте: разделение на тестовые модули и пакеты.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Различать чистые функции и функции с побочными эффектами (ОК 01, ОК 02).
2. Выбирать стратегию тестирования в зависимости от типа функции (ПК 3.2).
3. Организовывать тесты в проекте согласно лучшим практикам (ОК 05, ПК 3.1).
4. Проектировать тестовую архитектуру для масштабируемых проектов (ПК 3.3).

---

## 1. Чистые функции и функции с побочными эффектами

### 1.1 Определения

| Тип функции | Определение | Пример |
|:---|:---|:---|
| **Чистая функция (Pure function)** | Результат зависит только от входных аргументов; не изменяет внешнее состояние; одинаковые аргументы → одинаковый результат | `def add(a, b): return a + b` |
| **Функция с побочными эффектами (Impure function)** | Изменяет внешнее состояние (БД, файлы, глобальные переменные) или зависит от внешних данных (время, случайность) | `def save_to_db(data): db.insert(data)` |

### 1.2 Сравнение чистых и нечистых функций
┌─────────────────────────────────────────────────────────────────────────────┐
│ ЧИСТАЯ ФУНКЦИЯ (PURE) │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ Входные данные ──► ┌─────────┐ ──► Выходные данные │
│ │ Функция │ │
│ └─────────┘ │
│ │
│ • Нет доступа к БД, файлам, сети │
│ • Нет изменения глобальных переменных │
│ • Нет вызова input(), random(), datetime.now() │
│ • Легко тестировать: аргументы → результат │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ ФУНКЦИЯ С ПОБОЧНЫМИ ЭФФЕКТАМИ (IMPURE) │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ Входные данные ──► ┌─────────┐ ──► Выходные данные │
│ │ Функция │ │
│ └────┬────┘ │
│ ↓ │
│ ┌─────────────┐ │
│ │ БАЗА ДАННЫХ │ Изменение состояния │
│ │ ФАЙЛОВАЯ │ │
│ │ СИСТЕМА │ │
│ │ СЕТЬ │ │
│ └─────────────┘ │
│ │
│ • Требует мокирования для тестирования │
│ • Нужна очистка состояния после тестов │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 1.3 Примеры чистых функций

```python
# Чистые функции

def add(a: int, b: int) -> int:
    """Результат зависит только от a и b."""
    return a + b

def factorial(n: int) -> int:
    """Не изменяет ничего вне функции."""
    if n <= 1:
        return 1
    return n * factorial(n - 1)

def format_price(price: float, currency: str = "₽") -> str:
    """Форматирует цену без внешних зависимостей."""
    return f"{price:.2f} {currency}"

def find_max(numbers: list) -> int:
    """Работает только с переданными данными."""
    return max(numbers) if numbers else 0

# Тестирование чистых функций — простое
def test_add():
    assert add(2, 3) == 5
    assert add(-1, 1) == 0

def test_factorial():
    assert factorial(0) == 1
    assert factorial(5) == 120
1.4 Примеры функций с побочными эффектами
python
import datetime
import random
import json

# Функции с побочными эффектами

def save_to_file(filename: str, data: dict) -> None:
    """Записывает данные в файл (побочный эффект)."""
    with open(filename, 'w') as f:
        json.dump(data, f)

def get_current_time() -> str:
    """Зависит от внешнего времени."""
    return datetime.datetime.now().isoformat()

def get_random_number() -> int:
    """Зависит от состояния генератора случайных чисел."""
    return random.randint(1, 100)

def update_user_balance(user_id: int, amount: float) -> None:
    """Изменяет состояние базы данных."""
    # UPDATE users SET balance = balance + amount WHERE id = user_id
    pass

# Тестирование нечистых функций требует подготовки окружения
def test_save_to_file(tmp_path):
    """Используем tmp_path для изоляции."""
    file_path = tmp_path / "data.json"
    save_to_file(file_path, {"key": "value"})
    assert file_path.read_text() == '{"key": "value"}'
1.5 Почему чистые функции легче тестировать?
Аспект	Чистая функция	Функция с побочными эффектами
Изоляция тестов	Не нужна	Нужна (БД, файлы, сеть)
Порядок тестов	Не важен	Важен (состояние может меняться)
Параллельный запуск	Безопасен	Может конфликтовать
Повторяемость	100%	Зависит от окружения
Мокирование	Не требуется	Часто необходимо
2. Принципы тестирования чистых функций
2.1 Основные принципы
python
# Принцип 1: Один тест — одна проверка
def test_add_positive_numbers():
    assert add(2, 3) == 5

def test_add_negative_numbers():
    assert add(-2, -3) == -5

def test_add_zero():
    assert add(5, 0) == 5

# Принцип 2: Тестирование граничных значений
def test_factorial_edge_cases():
    assert factorial(0) == 1
    assert factorial(1) == 1

# Принцип 3: Параметризация для множества случаев
@pytest.mark.parametrize("a,b,expected", [
    (2, 3, 5),
    (-1, 1, 0),
    (0, 0, 0),
    (100, 200, 300),
])
def test_add_parametrized(a, b, expected):
    assert add(a, b) == expected
2.2 Стратегия тестирования чистых функций
text
┌─────────────────────────────────────────────────────────────────────────────┐
│              СТРАТЕГИЯ ТЕСТИРОВАНИЯ ЧИСТЫХ ФУНКЦИЙ                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. ПОКРЫТИЕ ВХОДНЫХ ДАННЫХ                                                  │
│     ├── Все возможные ветвления (if/else)                                   │
│     ├── Граничные значения                                                   │
│     └── Классы эквивалентности                                               │
│                                                                              │
│  2. ПРОВЕРКА ВЫХОДНЫХ ДАННЫХ                                                 │
│     ├── Тип возвращаемого значения                                           │
│     ├── Корректность вычислений                                              │
│     └── Форматирование (для строк)                                           │
│                                                                              │
│  3. ПРОВЕРКА ИСКЛЮЧЕНИЙ                                                      │
│     ├── Некорректные входные данные                                          │
│     └── Ошибки в логике                                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
3. Принципы тестирования функций с побочными эффектами
3.1 Основные подходы
python
# Подход 1: Изоляция с помощью временных ресурсов
def test_save_to_file_with_tmp_path(tmp_path):
    """Использование временной директории."""
    file_path = tmp_path / "output.json"
    save_to_file(file_path, {"data": 42})
    assert json.loads(file_path.read_text())["data"] == 42

# Подход 2: Мокирование внешних вызовов
def test_send_email(monkeypatch):
    """Подмена функции отправки письма."""
    sent_messages = []
    
    def mock_send(to, subject, body):
        sent_messages.append((to, subject, body))
        return True
    
    monkeypatch.setattr("email_service.send", mock_send)
    
    result = send_welcome_email("user@test.com")
    assert result is True
    assert len(sent_messages) == 1
    assert sent_messages[0][0] == "user@test.com"

# Подход 3: Очистка после тестов (teardown)
@pytest.fixture
def database():
    """Создание тестовой БД и очистка после теста."""
    db = create_test_database()
    yield db
    db.drop()  # Очистка
3.2 Стратегия тестирования нечистых функций
text
┌─────────────────────────────────────────────────────────────────────────────┐
│           СТРАТЕГИЯ ТЕСТИРОВАНИЯ ФУНКЦИЙ С ПОБОЧНЫМИ ЭФФЕКТАМИ               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. ИЗОЛЯЦИЯ                                                                │
│     ├── Временные файлы (tmp_path)                                          │
│     ├── Тестовые базы данных                                                 │
│     └── Подменённые сетевые вызовы                                          │
│                                                                              │
│  2. ФИКСТУРЫ ДЛЯ ПОДГОТОВКИ ОКРУЖЕНИЯ                                        │
│     ├── Создание ресурсов перед тестом                                       │
│     └── Очистка ресурсов после теста (yield)                                │
│                                                                              │
│  3. МОКИРОВАНИЕ                                                              │
│     ├── monkeypatch для подмены функций                                     │
│     └── Создание заглушек для внешних сервисов                              │
│                                                                              │
│  4. ПРОВЕРКА СОСТОЯНИЯ                                                       │
│     ├── Что изменилось в БД/файлах?                                         │
│     └── Были ли вызваны внешние сервисы?                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
4. Организация тестов в проекте
4.1 Базовая структура тестов
text
project/
│
├── src/                          # Исходный код
│   ├── __init__.py
│   ├── calculator.py             # Модуль с калькулятором
│   ├── user_manager.py           # Модуль с пользователями
│   └── file_processor.py         # Модуль с файлами
│
├── tests/                        # Тесты
│   ├── __init__.py
│   ├── test_calculator.py        # Тесты для calculator.py
│   ├── test_user_manager.py      # Тесты для user_manager.py
│   └── test_file_processor.py    # Тесты для file_processor.py
│
└── conftest.py                   # Общие фикстуры
4.2 Структура для больших проектов
text
project/
│
├── src/
│   ├── __init__.py
│   ├── auth/
│   │   ├── __init__.py
│   │   ├── models.py
│   │   └── services.py
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   ├── calculator.py
│   │   └── exceptions.py
│   │
│   └── utils/
│       ├── __init__.py
│       ├── file_utils.py
│       └── validators.py
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py               # Общие фикстуры
│   │
│   ├── auth/                     # Тесты модуля auth
│   │   ├── __init__.py
│   │   ├── conftest.py           # Фикстуры для auth
│   │   ├── test_models.py
│   │   └── test_services.py
│   │
│   ├── core/                     # Тесты модуля core
│   │   ├── __init__.py
│   │   ├── test_calculator.py
│   │   └── test_exceptions.py
│   │
│   └── utils/                    # Тесты модуля utils
│       ├── __init__.py
│       ├── test_file_utils.py
│       └── test_validators.py
│
├── pytest.ini                    # Конфигурация pytest
└── conftest.py                   # Глобальные фикстуры
4.3 Файл pytest.ini
ini
# pytest.ini
[pytest]
# Минимальная версия Python
minversion = 7.0

# Директория с тестами
testpaths = tests

# Паттерны для поиска тестов
python_files = test_*.py
python_classes = Test*
python_functions = test_*

# Добавить корневую директорию в PYTHONPATH
pythonpath = src

# Опции командной строки по умолчанию
addopts = -v --strict-markers --tb=short

# Маркеры
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    unit: unit tests
    integration: integration tests
    regression: regression tests

# Исключения для покрытия
norecursedirs = .git .venv .pytest_cache __pycache__

# Таймаут для тестов (секунды)
timeout = 300
4.4 Конфигурация для покрытия кода (pyproject.toml)
toml
# pyproject.toml
[tool.coverage.run]
source = ["src"]
omit = ["tests/*", "src/__init__.py"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "if self.debug:",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
]

fail_under = 80
show_missing = True
4.5 Пример организации тестов
python
# tests/conftest.py — глобальные фикстуры

import pytest
from src.user_manager import UserManager


@pytest.fixture(scope="session")
def global_config():
    """Глобальная конфигурация для всех тестов."""
    return {"debug": True, "test_mode": True}


@pytest.fixture
def user_manager():
    """Свежий экземпляр UserManager для каждого теста."""
    return UserManager()


@pytest.fixture
def user_manager_with_users(user_manager):
    """UserManager с предзаполненными пользователями."""
    users = []
    for i in range(3):
        user = user_manager.add_user(f"user{i}", f"user{i}@test.com")
        users.append(user)
    return {"manager": user_manager, "users": users}
python
# tests/auth/test_services.py

import pytest
from src.auth.services import AuthService


@pytest.fixture
def auth_service(user_manager):
    """Фикстура для тестирования AuthService."""
    return AuthService(user_manager)


class TestAuthService:
    """Тесты для AuthService."""
    
    def test_login_success(self, auth_service, user_manager):
        """Успешный вход."""
        user_manager.add_user("test", "test@test.com")
        result = auth_service.login("test", "password")
        assert result is not None
    
    def test_login_fail(self, auth_service):
        """Неудачный вход."""
        result = auth_service.login("unknown", "pass")
        assert result is None
    
    @pytest.mark.parametrize("username,password", [
        ("", "pass"),
        ("user", ""),
        ("", ""),
    ])
    def test_login_empty_credentials(self, auth_service, username, password):
        """Пустые учётные данные."""
        result = auth_service.login(username, password)
        assert result is None
5. Разделение тестов по типам
5.1 Маркировка тестов
python
# tests/auth/test_services.py

import pytest


@pytest.mark.unit
class TestAuthServiceUnit:
    """Модульные тесты (без внешних зависимостей)."""
    
    def test_hash_password(self):
        result = hash_password("secret")
        assert len(result) > 0


@pytest.mark.integration
class TestAuthServiceIntegration:
    """Интеграционные тесты (с БД)."""
    
    def test_save_user_to_db(self, db_session):
        user = User(username="test")
        db_session.add(user)
        db_session.commit()
        assert user.id is not None


@pytest.mark.regression
def test_regression_bug_123():
    """Регрессионный тест на баг #123."""
    # Проверяем, что исправление работает
    pass


@pytest.mark.slow
def test_heavy_computation():
    """Медленный тест, запускаемый отдельно."""
    result = expensive_calculation(10000)
    assert result > 0
5.2 Запуск тестов по маркерам
bash
# Запуск только модульных тестов
pytest -m unit

# Запуск всех тестов, кроме медленных
pytest -m "not slow"

# Запуск модульных и интеграционных
pytest -m "unit or integration"

# Запуск регрессионных тестов
pytest -m regression
6. Сравнение чистых и нечистых функций в тестировании
Критерий	Чистая функция	Нечистая функция
Пример	add(a, b), factorial(n)	save_to_db(data), send_email()
Сложность тестирования	Низкая	Высокая
Нужен ли мок	Нет	Да
Нужна ли фикстура	Редко	Часто
Параллельный запуск	Безопасно	Рискованно
Техники	ЭП, ГЗ, таблицы решений	Мокирование, изоляция
Кто пишет	Разработчик	Разработчик + QA
7. Практические рекомендации
7.1 Рекомендации по организации тестов
Практика	Описание
Один модуль — один тестовый файл	calculator.py → test_calculator.py
Использование подпапок	Тесты повторяют структуру src
Общие фикстуры в conftest.py	Не дублировать код
Маркировка тестов	@pytest.mark.unit, @pytest.mark.integration
Читаемые имена тестов	test_add_positive_numbers, а не test_1
Изоляция тестов	Каждый тест независим
7.2 Что не следует делать
python
# ❌ НЕПРАВИЛЬНО: зависимость от порядка тестов
class TestBad:
    def test_create(self):
        self.user = create_user()  # Создаём
        
    def test_delete(self):
        delete_user(self.user)     # Используем из другого теста

# ✅ ПРАВИЛЬНО: каждый тест независим
class TestGood:
    def test_create(self):
        user = create_user()
        assert user is not None
    
    def test_delete(self):
        user = create_user()
        result = delete_user(user)
        assert result is True
python
# ❌ НЕПРАВИЛЬНО: один тест проверяет слишком много
def test_user_lifecycle():
    user = create_user()
    assert user.id == 1
    user.email = "new@test.com"
    update_user(user)
    assert get_user(user.id).email == "new@test.com"
    delete_user(user)
    assert get_user(user.id) is None

# ✅ ПРАВИЛЬНО: разделить на отдельные тесты
def test_create_user():
    user = create_user()
    assert user is not None

def test_update_user():
    user = create_user()
    user.email = "new@test.com"
    update_user(user)
    assert get_user(user.id).email == "new@test.com"

def test_delete_user():
    user = create_user()
    delete_user(user)
    assert get_user(user.id) is None
8. Шпаргалка
python
# === ЧИСТАЯ ФУНКЦИЯ ===
def pure_add(a, b):
    return a + b

def test_pure_add():
    assert pure_add(2, 3) == 5
    assert pure_add(-1, 1) == 0
    assert pure_add(0, 0) == 0

# === НЕЧИСТАЯ ФУНКЦИЯ (С ПОБОЧНЫМ ЭФФЕКТОМ) ===
def save_to_file(path, content):
    with open(path, 'w') as f:
        f.write(content)

def test_save_to_file(tmp_path):
    path = tmp_path / "test.txt"
    save_to_file(path, "hello")
    assert path.read_text() == "hello"

# === ОРГАНИЗАЦИЯ ТЕСТОВ ===
# tests/test_calculator.py
# tests/test_user_manager.py
# tests/auth/test_services.py
# tests/utils/test_validators.py

# === МАРКИРОВКА ===
@pytest.mark.unit
def test_unit():
    pass

@pytest.mark.integration
def test_integration():
    pass

# === ЗАПУСК ===
# pytest -m unit
# pytest -m "not slow"
Контрольные вопросы
Что такое чистая функция? Приведите пример.

Чем функция с побочными эффектами отличается от чистой функции?

Почему чистые функции легче тестировать?

Какие инструменты pytest помогают тестировать нечистые функции?

Как организовать тесты для большого проекта?

Зачем нужны маркеры в pytest?

Что должно лежать в conftest.py?

Как протестировать функцию, которая пишет в файл, без модификации реальной файловой системы?

Почему тесты не должны зависеть от порядка выполнения?

Какие техники тест-дизайна лучше всего подходят для чистых функций?

Следующее занятие: ПР 3.8 — Практическое тестирование чистых функций и функций с побочными эффектами. Организация тестовой структуры проекта.
