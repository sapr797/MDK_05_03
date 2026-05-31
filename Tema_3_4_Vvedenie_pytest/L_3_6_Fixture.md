# Лекция 3.6: Фикстуры в pytest: базовый уровень

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.6:** Фикстуры в pytest  
**Тип занятия:** Лекция + Практика (4 часа по УП 2.1)

---

## Цель лекции

Изучить механизм фикстур в pytest для подготовки тестовых данных, настройки окружения и управления ресурсами.

## Планируемые результаты

После этой лекции вы сможете:
1. Создавать фикстуры с помощью `@pytest.fixture` (ПК 3.2).
2. Использовать фикстуры в тестовых функциях (ПК 3.2).
3. Настраивать область видимости фикстур (scope) (ПК 3.2, ОК 05).
4. Применять фикстуры для подготовки тестовых данных и очистки после тестов (ПК 3.3).

---

## Теоретическая справка

### Что такое фикстуры?

**Фикстура (fixture)** — это функция, которая выполняется перед тестом (или после него) для подготовки тестового окружения: создания данных, подключения к БД, настройки моков и т.д.
┌─────────────────────────────────────────────────────────────────┐
│ ЖИЗНЕННЫЙ ЦИКЛ ФИКСТУРЫ │
├─────────────────────────────────────────────────────────────────┤
│ │
│ def test_example(fixture): │
│ │ │
│ ├── 1. pytest вызывает фикстуру │
│ │ │ │
│ │ ├── setup (подготовка данных) │
│ │ │ │
│ │ └── yield (передача данных в тест) │
│ │ │ │
│ ├── 2. выполняется тело теста │
│ │ │ │
│ └── 3. после теста (teardown) — код после yield │
│ │
└─────────────────────────────────────────────────────────────────┘

text

### Основные возможности фикстур

| Возможность | Синтаксис | Описание |
|:---|:---|:---|
| **Создание фикстуры** | `@pytest.fixture` | Декоратор для функции-фикстуры |
| **Использование в тесте** | `def test_func(fixture_name)` | Фикстура передаётся как аргумент |
| **Область видимости** | `scope="function"` (по умолчанию) | Для каждой тестовой функции |
| | `scope="class"` | Один раз на класс |
| | `scope="module"` | Один раз на модуль |
| | `scope="session"` | Один раз на всю сессию |
| **Teardown (очистка)** | `yield` вместо `return` | Код после yield выполняется после теста |
| **Автоиспользование** | `autouse=True` | Фикстура применяется ко всем тестам автоматически |

### Пример базовой фикстуры

```python
import pytest

@pytest.fixture
def sample_data():
    """Фикстура, возвращающая тестовые данные."""
    return {"name": "John", "age": 30}

def test_using_fixture(sample_data):
    assert sample_data["name"] == "John"
    assert sample_data["age"] == 30
Объект тестирования
Будем тестировать класс UserManager для работы с пользователями.

Листинг 1. Исходный код (user_manager.py)
python
# user_manager.py

from dataclasses import dataclass, field
from typing import List, Optional, Dict
from datetime import datetime


@dataclass
class User:
    """Модель пользователя."""
    id: int
    username: str
    email: str
    is_active: bool = True
    created_at: datetime = field(default_factory=datetime.now)


class UserManager:
    """Менеджер пользователей с тестовыми методами."""
    
    def __init__(self):
        self._users: Dict[int, User] = {}
        self._next_id = 1
    
    def add_user(self, username: str, email: str) -> User:
        """Добавляет нового пользователя."""
        user = User(id=self._next_id, username=username, email=email)
        self._users[user.id] = user
        self._next_id += 1
        return user
    
    def get_user(self, user_id: int) -> Optional[User]:
        """Возвращает пользователя по ID."""
        return self._users.get(user_id)
    
    def delete_user(self, user_id: int) -> bool:
        """Удаляет пользователя. Возвращает True, если пользователь существовал."""
        if user_id in self._users:
            del self._users[user_id]
            return True
        return False
    
    def get_active_users(self) -> List[User]:
        """Возвращает список активных пользователей."""
        return [user for user in self._users.values() if user.is_active]
    
    def deactivate_user(self, user_id: int) -> bool:
        """Деактивирует пользователя."""
        user = self.get_user(user_id)
        if user:
            user.is_active = False
            return True
        return False
    
    def count(self) -> int:
        """Возвращает количество пользователей."""
        return len(self._users)
Уровни сложности
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Создание простых фикстур, возвращающих данные
Средний (варианты 9-17)	Средние студенты	Фикстуры с yield, разная область видимости (scope)
Сложный (варианты 18-25)	Сильные студенты	Вложенные фикстуры, autouse, параметризация фикстур
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться создавать простые фикстуры с @pytest.fixture и использовать их в тестах.

Задания для базового уровня
Задание 1.1. Создание простой фикстуры (10 минут)
Создайте фикстуру, которая возвращает экземпляр UserManager.

python
# test_user_manager_base.py

import pytest
from user_manager import UserManager, User


# ============================================================
# ФИКСТУРЫ
# ============================================================

@pytest.fixture
def user_manager():
    """
    Фикстура, возвращающая новый экземпляр UserManager.
    
    Returns:
        UserManager: Пустой менеджер пользователей
    """
    return UserManager()


@pytest.fixture
def sample_user():
    """
    Фикстура, возвращающая тестового пользователя (не добавленного в менеджер).
    
    Returns:
        dict: Данные пользователя
    """
    return {
        "username": "testuser",
        "email": "test@example.com"
    }


@pytest.fixture
def user_with_id():
    """
    Фикстура, возвращающая пользователя с предопределённым ID.
    """
    return User(id=100, username="predefined", email="pre@example.com")


# ============================================================
# ТЕСТЫ
# ============================================================

class TestUserManagerBase:
    """Базовые тесты с использованием фикстур."""
    
    def test_add_user(self, user_manager, sample_user):
        """Тест добавления пользователя."""
        user = user_manager.add_user(**sample_user)
        
        assert user.id == 1
        assert user.username == sample_user["username"]
        assert user.email == sample_user["email"]
        assert user.is_active is True
    
    def test_get_user(self, user_manager, sample_user):
        """Тест получения пользователя по ID."""
        # Добавляем пользователя
        added = user_manager.add_user(**sample_user)
        
        # Получаем его по ID
        retrieved = user_manager.get_user(added.id)
        
        assert retrieved is not None
        assert retrieved.id == added.id
        assert retrieved.username == sample_user["username"]
    
    def test_get_nonexistent_user(self, user_manager):
        """Тест получения несуществующего пользователя."""
        result = user_manager.get_user(999)
        assert result is None
    
    def test_user_manager_is_empty(self, user_manager):
        """Проверка, что новый менеджер пуст."""
        assert user_manager.count() == 0
Задание 1.2. Использование нескольких фикстур в одном тесте (10 минут)
Напишите тесты, которые используют несколько фикстур одновременно.

python
class TestMultipleFixtures:
    """Тесты с использованием нескольких фикстур."""
    
    def test_add_multiple_users(self, user_manager):
        """Добавление нескольких пользователей."""
        user1 = user_manager.add_user("alice", "alice@example.com")
        user2 = user_manager.add_user("bob", "bob@example.com")
        
        assert user_manager.count() == 2
        assert user1.id == 1
        assert user2.id == 2
    
    def test_fixture_reuse(self, user_manager, sample_user):
        """Одна и та же фикстура может использоваться в нескольких тестах."""
        user_manager.add_user(**sample_user)
        assert user_manager.count() == 1
        
        # Фикстура sample_user не меняется между вызовами
        # Каждый тест получает свежую копию
    
    def test_user_data_fixture(self, user_with_id):
        """Использование фикстуры с объектом User."""
        assert user_with_id.id == 100
        assert user_with_id.username == "predefined"
Задание 1.3. Фикстуры для разных типов данных (10 минут)
Создайте фикстуры для различных типов тестовых данных.

python
@pytest.fixture
def users_list():
    """Фикстура, возвращающая список пользователей."""
    return [
        {"username": "user1", "email": "user1@test.com"},
        {"username": "user2", "email": "user2@test.com"},
        {"username": "user3", "email": "user3@test.com"},
    ]


@pytest.fixture
def user_manager_with_users(user_manager, users_list):
    """Фикстура, возвращающая UserManager с предзаполненными пользователями."""
    for user_data in users_list:
        user_manager.add_user(**user_data)
    return user_manager


class TestDataFixtures:
    """Тесты с фикстурами-данными."""
    
    def test_users_list_fixture(self, users_list):
        """Проверка фикстуры со списком."""
        assert len(users_list) == 3
        assert users_list[0]["username"] == "user1"
    
    def test_manager_with_users(self, user_manager_with_users):
        """Тест менеджера с предзаполненными данными."""
        assert user_manager_with_users.count() == 3
        
        users = user_manager_with_users.get_active_users()
        assert len(users) == 3
    
    def test_each_test_gets_fresh_fixture(self, user_manager_with_users):
        """Каждый тест получает свежую фикстуру (scope='function' по умолчанию)."""
        # Этот тест видит тех же 3 пользователей
        assert user_manager_with_users.count() == 3
        
        # Добавляем ещё одного
        user_manager_with_users.add_user("extra", "extra@test.com")
        assert user_manager_with_users.count() == 4


def test_direct_fixture_usage(user_manager):
    """Фикстуры можно использовать и вне класса."""
    assert user_manager.count() == 0
Задание 1.4. Запуск тестов (5 минут)
bash
# Запуск всех тестов базового уровня
pytest test_user_manager_base.py -v

# Запуск с подробным выводом
pytest test_user_manager_base.py -v --tb=short
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться использовать yield для teardown (очистки) и настраивать область видимости фикстур (scope).

Задания для среднего уровня
Задание 2.1. Фикстуры с teardown (yield) (15 минут)
Создайте фикстуры, которые выполняют очистку после тестов.

python
# test_user_manager_intermediate.py

import pytest
import tempfile
import os
import json
from user_manager import UserManager, User


# ============================================================
# ФИКСТУРЫ С YIELD (TEARDOWN)
# ============================================================

@pytest.fixture
def user_manager_with_cleanup():
    """
    Фикстура с автоматической очисткой после теста.
    """
    print("\n[SETUP] Создание UserManager")
    manager = UserManager()
    
    yield manager  # Передаём менеджер в тест
    
    print("\n[TEARDOWN] Очистка UserManager")
    # Здесь можно добавить дополнительную очистку
    manager._users.clear()


@pytest.fixture
def temp_file():
    """
    Фикстура, создающая временный файл и удаляющая его после теста.
    """
    print("\n[SETUP] Создание временного файла")
    fd, path = tempfile.mkstemp(suffix=".json")
    os.close(fd)
    
    yield path  # Передаём путь к файлу
    
    print(f"\n[TEARDOWN] Удаление временного файла: {path}")
    if os.path.exists(path):
        os.unlink(path)


@pytest.fixture
def user_manager_with_data_and_cleanup():
    """
    Фикстура, добавляющая тестовые данные и очищающая их после теста.
    """
    manager = UserManager()
    
    # Добавляем тестовых пользователей
    alice = manager.add_user("alice", "alice@test.com")
    bob = manager.add_user("bob", "bob@test.com")
    
    print(f"\n[SETUP] Добавлены пользователи: alice (id={alice.id}), bob (id={bob.id})")
    
    yield manager
    
    # Очистка: удаляем всех пользователей
    print(f"\n[TEARDOWN] Удаление всех пользователей")
    for user_id in list(manager._users.keys()):
        manager.delete_user(user_id)


# ============================================================
# ТЕСТЫ С ФИКСТУРАМИ, ИСПОЛЬЗУЮЩИМИ YIELD
# ============================================================

class TestTeardownFixtures:
    """Тесты с фикстурами, выполняющими teardown."""
    
    def test_manager_cleanup(self, user_manager_with_cleanup):
        """Тест с автоматической очисткой."""
        user_manager_with_cleanup.add_user("test", "test@test.com")
        assert user_manager_with_cleanup.count() == 1
        # После теста фикстура выполнит очистку
    
    def test_temp_file_creation(self, temp_file):
        """Тест с временным файлом."""
        # Записываем данные в файл
        with open(temp_file, 'w') as f:
            json.dump({"data": "test"}, f)
        
        # Проверяем, что файл существует
        assert os.path.exists(temp_file)
        # После теста файл будет удалён
    
    def test_manager_with_data_cleanup(self, user_manager_with_data_and_cleanup):
        """Тест менеджера с предзаполненными данными и очисткой."""
        assert user_manager_with_data_and_cleanup.count() == 2
        
        # Добавляем ещё одного пользователя
        user_manager_with_data_and_cleanup.add_user("charlie", "charlie@test.com")
        assert user_manager_with_data_and_cleanup.count() == 3
        # После теста все пользователи (включая charlie) будут удалены
Задание 2.2. Область видимости фикстур (scope) (15 минут)
Создайте фикстуры с разной областью видимости и пронаблюдайте их поведение.

python
# ============================================================
# ФИКСТУРЫ С РАЗНОЙ ОБЛАСТЬЮ ВИДИМОСТИ
# ============================================================

@pytest.fixture(scope="function")
def function_scoped_counter():
    """Фикстура с областью видимости function (по умолчанию)."""
    print("\n[SETUP] function_scoped_counter - создание")
    counter = {"value": 0}
    yield counter
    print("\n[TEARDOWN] function_scoped_counter - уничтожение")


@pytest.fixture(scope="class")
def class_scoped_counter():
    """Фикстура с областью видимости class."""
    print("\n[SETUP] class_scoped_counter - создание (ОДИН РАЗ НА КЛАСС)")
    counter = {"value": 0}
    yield counter
    print("\n[TEARDOWN] class_scoped_counter - уничтожение")


@pytest.fixture(scope="module")
def module_scoped_counter():
    """Фикстура с областью видимости module."""
    print("\n[SETUP] module_scoped_counter - создание (ОДИН РАЗ НА МОДУЛЬ)")
    counter = {"value": 0}
    yield counter
    print("\n[TEARDOWN] module_scoped_counter - уничтожение")


# ============================================================
# ТЕСТОВЫЕ КЛАССЫ ДЛЯ ПРОВЕРКИ SCOPE
# ============================================================

class TestFunctionScope:
    """Проверка фикстуры с scope='function'."""
    
    def test_first(self, function_scoped_counter):
        """Первый тест — получает свежую фикстуру."""
        function_scoped_counter["value"] += 1
        print(f"\nTest first: value = {function_scoped_counter['value']}")
        assert function_scoped_counter["value"] == 1
    
    def test_second(self, function_scoped_counter):
        """Второй тест — получает НОВУЮ фикстуру (счётчик сброшен)."""
        function_scoped_counter["value"] += 1
        print(f"\nTest second: value = {function_scoped_counter['value']}")
        assert function_scoped_counter["value"] == 1  # НЕ 2!


class TestClassScope:
    """Проверка фикстуры с scope='class'."""
    
    def test_first(self, class_scoped_counter):
        """Первый тест в классе."""
        class_scoped_counter["value"] += 1
        print(f"\nTest first: value = {class_scoped_counter['value']}")
        assert class_scoped_counter["value"] == 1
    
    def test_second(self, class_scoped_counter):
        """Второй тест в классе — та же фикстура (счётчик увеличился)."""
        class_scoped_counter["value"] += 1
        print(f"\nTest second: value = {class_scoped_counter['value']}")
        assert class_scoped_counter["value"] == 2  # ДА, 2!


class TestAnotherClassScope:
    """Другой класс — получает НОВУЮ фикстуру class_scoped_counter."""
    
    def test_in_another_class(self, class_scoped_counter):
        """Фикстура создаётся заново для этого класса."""
        class_scoped_counter["value"] += 1
        print(f"\nTest in another class: value = {class_scoped_counter['value']}")
        assert class_scoped_counter["value"] == 1  # Сброшена


def test_module_scope_first(module_scoped_counter):
    """Первый тест на уровне модуля."""
    module_scoped_counter["value"] += 1
    print(f"\nModule test first: value = {module_scoped_counter['value']}")
    assert module_scoped_counter["value"] == 1


def test_module_scope_second(module_scoped_counter):
    """Второй тест на уровне модуля — та же фикстура."""
    module_scoped_counter["value"] += 1
    print(f"\nModule test second: value = {module_scoped_counter['value']}")
    assert module_scoped_counter["value"] == 2  # Увеличилась!
Задание: Запустите тесты и проанализируйте вывод. Обратите внимание на сообщения [SETUP] и [TEARDOWN].

Задание 2.3. Фикстуры для подготовки БД (тестовые данные) (10 минут)
Создайте фикстуры для подготовки тестовых данных.

python
class TestDataPreparation:
    """Тесты с подготовкой данных через фикстуры."""
    
    @pytest.fixture
    def setup_users(self, user_manager):
        """Подготовка набора пользователей."""
        users = []
        users.append(user_manager.add_user("admin", "admin@test.com"))
        users.append(user_manager.add_user("editor", "editor@test.com"))
        users.append(user_manager.add_user("viewer", "viewer@test.com"))
        return users
    
    def test_user_roles_setup(self, setup_users):
        """Проверка, что фикстура setup_users создала 3 пользователей."""
        assert len(setup_users) == 3
        assert setup_users[0].username == "admin"
        assert setup_users[1].username == "editor"
        assert setup_users[2].username == "viewer"
    
    @pytest.fixture
    def complex_data_set(self, user_manager):
        """Сложная подготовка данных с несколькими этапами."""
        # Этап 1: создаём пользователей
        users = []
        for i in range(5):
            user = user_manager.add_user(f"user{i}", f"user{i}@test.com")
            users.append(user)
        
        # Этап 2: деактивируем некоторых
        user_manager.deactivate_user(users[1].id)
        user_manager.deactivate_user(users[3].id)
        
        return {
            "manager": user_manager,
            "users": users,
            "active_count": 3,
            "inactive_count": 2
        }
    
    def test_complex_data_set(self, complex_data_set):
        """Тест со сложными данными."""
        assert complex_data_set["active_count"] == 3
        assert complex_data_set["inactive_count"] == 2
        
        active_users = complex_data_set["manager"].get_active_users()
        assert len(active_users) == 3
Задание 2.4. Запуск тестов (5 минут)
bash
# Запуск с verbose выводом для наблюдения за setup/teardown
pytest test_user_manager_intermediate.py -v -s

# Параметр -s показывает print (позволяет увидеть сообщения SETUP/TEARDOWN)
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться создавать вложенные фикстуры, использовать autouse=True и параметризацию фикстур.

Задания для сложного уровня
Задание 3.1. Вложенные фикстуры (15 минут)
Фикстуры могут использовать другие фикстуры.

python
# test_user_manager_advanced.py

import pytest
from user_manager import UserManager, User


# ============================================================
# БАЗОВЫЕ ФИКСТУРЫ
# ============================================================

@pytest.fixture
def base_manager():
    """Базовая фикстура с пустым менеджером."""
    return UserManager()


@pytest.fixture
def user_data():
    """Базовые данные пользователя."""
    return {"username": "john_doe", "email": "john@example.com"}


# ============================================================
# ВЛОЖЕННЫЕ ФИКСТУРЫ (ИСПОЛЬЗУЮТ ДРУГИЕ ФИКСТУРЫ)
# ============================================================

@pytest.fixture
def manager_with_one_user(base_manager, user_data):
    """
    Вложенная фикстура: использует base_manager и user_data.
    """
    user = base_manager.add_user(**user_data)
    return {"manager": base_manager, "user": user}


@pytest.fixture
def manager_with_multiple_users(base_manager):
    """
    Вложенная фикстура: создаёт несколько пользователей.
    """
    users = []
    for i in range(3):
        user = base_manager.add_user(f"user{i}", f"user{i}@test.com")
        users.append(user)
    return {"manager": base_manager, "users": users}


@pytest.fixture
def complex_fixture_chain(manager_with_one_user, manager_with_multiple_users):
    """
    Фикстура, комбинирующая результаты других фикстур.
    """
    return {
        "single": manager_with_one_user,
        "multiple": manager_with_multiple_users,
        "total_users": 1 + 3
    }


# ============================================================
# ТЕСТЫ С ВЛОЖЕННЫМИ ФИКСТУРАМИ
# ============================================================

class TestNestedFixtures:
    """Тесты с вложенными фикстурами."""
    
    def test_nested_fixture_chain(self, complex_fixture_chain):
        """Проверка цепочки вложенных фикстур."""
        assert complex_fixture_chain["total_users"] == 4
        assert complex_fixture_chain["single"]["user"].username == "john_doe"
        assert len(complex_fixture_chain["multiple"]["users"]) == 3
    
    def test_manager_with_user(self, manager_with_one_user):
        """Тест с фикстурой, использующей другую фикстуру."""
        assert manager_with_one_user["manager"].count() == 1
        assert manager_with_one_user["user"].email == "john@example.com"
    
    def test_fixture_reuse_chain(self, base_manager, user_data, manager_with_one_user):
        """Демонстрация, что фикстуры не мешают друг другу."""
        # base_manager и manager_with_one_user используют разные экземпляры?
        # По умолчанию scope='function', поэтому да — разные
        assert base_manager.count() == 0
        assert manager_with_one_user["manager"].count() == 1
Задание 3.2. Автоиспользование фикстур (autouse=True) (10 минут)
Фикстуры с autouse=True выполняются автоматически для всех тестов.

python
# ============================================================
# ФИКСТУРЫ С AUTOSEUE=TRUE
# ============================================================

@pytest.fixture(autouse=True)
def global_setup():
    """Эта фикстура выполняется перед КАЖДЫМ тестом автоматически."""
    print("\n[GLOBAL SETUP] Выполняется перед каждым тестом")
    # Можно настроить окружение, логирование и т.д.
    yield
    print("\n[GLOBAL TEARDOWN] Выполняется после каждого теста")


@pytest.fixture(autouse=True, scope="class")
def class_setup():
    """Автоматическая фикстура на уровне класса."""
    print("\n[CLASS SETUP] Выполняется один раз перед тестами класса")
    yield
    print("\n[CLASS TEARDOWN] Выполняется один раз после тестов класса")


@pytest.fixture
def regular_fixture():
    """Обычная фикстура (не автоматическая)."""
    return {"data": "test"}


# ============================================================
# ТЕСТЫ С AUTOUSE ФИКСТУРАМИ
# ============================================================

class TestAutouseFixtures:
    """Тесты с автоматическими фикстурами."""
    
    def test_first(self, regular_fixture):
        """Первый тест — autouse фикстуры сработают автоматически."""
        print(f"\nВыполняется тест test_first, data={regular_fixture['data']}")
        assert regular_fixture["data"] == "test"
    
    def test_second(self):
        """Второй тест — даже без аргументов autouse фикстуры сработают."""
        print("\nВыполняется тест test_second (без аргументов)")
        assert True
    
    def test_third(self, regular_fixture):
        """Третий тест."""
        print(f"\nВыполняется тест test_third, data={regular_fixture['data']}")
        assert regular_fixture["data"] == "test"


@pytest.fixture(autouse=True)
def db_transaction():
    """Имитация транзакции БД: откат после каждого теста."""
    print("\n[DB] Начинаем транзакцию")
    # Здесь был бы реальный BEGIN
    yield
    print("[DB] Откатываем транзакцию (ROLLBACK)")
    # Здесь был бы реальный ROLLBACK


class TestDatabaseIsolation:
    """Тесты с изоляцией через автоподход."""
    
    def test_insert_data(self, user_manager):
        """Вставка данных."""
        user_manager.add_user("test", "test@test.com")
        assert user_manager.count() == 1
        # После теста транзакция откатится
    
    def test_database_is_empty(self, user_manager):
        """База данных пуста в новом тесте."""
        assert user_manager.count() == 0  # Данные из предыдущего теста не сохранились!
Задание 3.3. Параметризация фикстур (15 минут)
Фикстуры можно параметризовать с помощью @pytest.fixture(params=...).

python
# ============================================================
# ПАРАМЕТРИЗОВАННЫЕ ФИКСТУРЫ
# ============================================================

@pytest.fixture(params=[
    (1, "alice", "alice@test.com"),
    (2, "bob", "bob@test.com"),
    (3, "charlie", "charlie@test.com"),
])
def user_params(request):
    """
    Параметризованная фикстура с данными пользователей.
    Каждый параметр запускает тест отдельно.
    """
    return {
        "id": request.param[0],
        "username": request.param[1],
        "email": request.param[2]
    }


@pytest.fixture(params=["city", "suburb"])
def delivery_location(request):
    """Параметризованная фикстура для тестирования доставки."""
    return request.param


@pytest.fixture(params=["regular", "vip", "new"])
def user_status(request):
    """Параметризованная фикстура для статуса пользователя."""
    return request.param


@pytest.fixture(params=[
    {"name": "low", "total": 100},
    {"name": "medium", "total": 5000},
    {"name": "high", "total": 10000},
])
def order_params(request):
    """Параметризованная фикстура с параметрами заказа."""
    return request.param


# ============================================================
# ТЕСТЫ С ПАРАМЕТРИЗОВАННЫМИ ФИКСТУРАМИ
# ============================================================

class TestParametrizedFixtures:
    """Тесты с параметризованными фикстурами."""
    
    def test_user_params(self, user_params):
        """Тест будет запущен 3 раза (для каждого пользователя)."""
        print(f"\nТестируем пользователя: {user_params['username']}")
        assert user_params["id"] > 0
        assert "@" in user_params["email"]
    
    def test_delivery_location(self, delivery_location):
        """Тест будет запущен 2 раза (city и suburb)."""
        print(f"\nЛокация доставки: {delivery_location}")
        assert delivery_location in ["city", "suburb"]
    
    def test_user_status(self, user_status):
        """Тест будет запущен 3 раза (regular, vip, new)."""
        print(f"\nСтатус пользователя: {user_status}")
        assert user_status in ["regular", "vip", "new"]
    
    def test_combined_params(self, delivery_location, user_status):
        """Тест будет запущен 2×3 = 6 раз (комбинация параметров)."""
        print(f"\nКомбинация: {delivery_location} + {user_status}")
        assert isinstance(delivery_location, str)
        assert isinstance(user_status, str)
    
    def test_order_params(self, order_params):
        """Тест будет запущен 3 раза (low, medium, high)."""
        print(f"\nЗаказ: {order_params['name']}, сумма={order_params['total']}")
        assert order_params["total"] > 0


# ============================================================
# КОМБИНАЦИЯ ПАРАМЕТРИЗАЦИИ ФИКСТУР И ПАРАМЕТРИЗАЦИИ ТЕСТОВ
# ============================================================

class TestCombinedParametrization:
    """Комбинация параметризации фикстур и параметризации тестов."""
    
    @pytest.fixture(params=[10, 100, 1000])
    def base_amount(self, request):
        """Параметризованная фикстура с суммой."""
        return request.param
    
    @pytest.mark.parametrize("discount", [0, 10, 20])
    def test_discount_calculation(self, base_amount, discount):
        """
        Комбинация: 3 значения из фикстуры × 3 значения из parametrize = 9 тестов.
        """
        result = base_amount - (base_amount * discount / 100)
        print(f"\nСумма: {base_amount}, скидка: {discount}% → {result}")
        assert result >= 0
Задание 3.4. Полный пример: тестирование UserManager с фикстурами (15 минут)
Создайте полный набор тестов для UserManager, используя все изученные техники.

python
# test_user_manager_complete.py

import pytest
from user_manager import UserManager, User


# ============================================================
# ФИКСТУРЫ
# ============================================================

@pytest.fixture
def empty_manager():
    """Пустой менеджер пользователей."""
    return UserManager()


@pytest.fixture
def manager_with_users():
    """Менеджер с предзаполненными пользователями."""
    manager = UserManager()
    users = []
    for i in range(1, 4):
        user = manager.add_user(f"user{i}", f"user{i}@test.com")
        users.append(user)
    return {"manager": manager, "users": users}


@pytest.fixture(scope="function")
def clean_manager():
    """Менеджер, который очищается после каждого теста."""
    manager = UserManager()
    yield manager
    # Очистка после теста
    for user_id in list(manager._users.keys()):
        manager.delete_user(user_id)


@pytest.fixture(params=["active", "inactive"])
def user_state(request):
    """Параметризованная фикстура для состояния пользователя."""
    return request.param


@pytest.fixture
def state_manager(user_state):
    """Менеджер с пользователем в заданном состоянии."""
    manager = UserManager()
    user = manager.add_user("test", "test@test.com")
    if user_state == "inactive":
        manager.deactivate_user(user.id)
    return {"manager": manager, "user": user, "state": user_state}


# ============================================================
# ТЕСТЫ
# ============================================================

class TestUserManagerComplete:
    """Полный набор тестов UserManager."""
    
    def test_add_user(self, empty_manager):
        """Тест добавления пользователя."""
        user = empty_manager.add_user("newuser", "new@test.com")
        assert user.id == 1
        assert user.username == "newuser"
        assert empty_manager.count() == 1
    
    def test_get_user(self, manager_with_users):
        """Тест получения пользователя."""
        manager = manager_with_users["manager"]
        users = manager_with_users["users"]
        
        retrieved = manager.get_user(users[0].id)
        assert retrieved is not None
        assert retrieved.username == users[0].username
    
    def test_get_nonexistent_user(self, manager_with_users):
        """Тест получения несуществующего пользователя."""
        manager = manager_with_users["manager"]
        assert manager.get_user(999) is None
    
    def test_delete_user(self, manager_with_users):
        """Тест удаления пользователя."""
        manager = manager_with_users["manager"]
        users = manager_with_users["users"]
        
        assert manager.delete_user(users[0].id) is True
        assert manager.count() == 2
        assert manager.get_user(users[0].id) is None
    
    def test_delete_nonexistent_user(self, manager_with_users):
        """Тест удаления несуществующего пользователя."""
        manager = manager_with_users["manager"]
        assert manager.delete_user(999) is False
    
    def test_deactivate_user(self, manager_with_users):
        """Тест деактивации пользователя."""
        manager = manager_with_users["manager"]
        users = manager_with_users["users"]
        
        assert manager.deactivate_user(users[0].id) is True
        assert manager.get_user(users[0].id).is_active is False
    
    def test_active_users_filter(self, manager_with_users):
        """Тест фильтрации активных пользователей."""
        manager = manager_with_users["manager"]
        users = manager_with_users["users"]
        
        # Деактивируем одного
        manager.deactivate_user(users[1].id)
        
        active = manager.get_active_users()
        assert len(active) == 2
        assert all(u.is_active for u in active)
    
    def test_cleanup_fixture(self, clean_manager):
        """Тест с фикстурой, выполняющей очистку."""
        clean_manager.add_user("temp", "temp@test.com")
        assert clean_manager.count() == 1
        # После теста clean_manager будет очищен
    
    def test_parametrized_state(self, state_manager):
        """Параметризованный тест состояния пользователя."""
        if state_manager["state"] == "active":
            assert state_manager["user"].is_active is True
        else:
            assert state_manager["user"].is_active is False
    
    @pytest.mark.parametrize("username,email", [
        ("short", "a@b.c"),
        ("very_long_username_that_exceeds_typical_limits", "long@example.com"),
        ("user_with_numbers_123", "numbers@test.com"),
    ])
    def test_add_user_parametrized(self, empty_manager, username, email):
        """Параметризованный тест добавления пользователя."""
        user = empty_manager.add_user(username, email)
        assert user.username == username
        assert user.email == email
Задание 3.5. Запуск тестов и отчёт (5 минут)
bash
# Запуск всех тестов с отчётом о покрытии
pytest test_user_manager_complete.py -v --cov=user_manager --cov-report=html

# Запуск с подробным выводом (видно параметризацию)
pytest test_user_manager_complete.py -v -s

# Запуск только определённого класса
pytest test_user_manager_complete.py::TestUserManagerComplete -v
Карточка студента
text
ПР 3.6. ФИКСТУРЫ В PYTEST

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ Простые фикстуры (базовый)
□ Несколько фикстур в одном тесте (базовый)
□ Фикстуры с yield / teardown (средний)
□ Область видимости scope (средний)
□ Подготовка тестовых данных (средний)
□ Вложенные фикстуры (сложный)
□ Autouse фикстуры (сложный)
□ Параметризация фикстур (сложный)

=== ИСПОЛЬЗОВАННЫЕ МЕХАНИЗМЫ ===

□ @pytest.fixture
□ yield (teardown)
□ scope="function/class/module/session"
□ autouse=True
□ parametrized fixtures (params)
□ вложенные фикстуры

=== ВРЕМЯ ВЫПОЛНЕНИЯ ===

Подготовка: _____ мин
Выполнение: _____ мин
Оформление: _____ мин

=== ОТЧЁТ ===

Файлы:
- test_user_manager_base.py
- test_user_manager_intermediate.py
- test_user_manager_advanced.py
- test_user_manager_complete.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Тесты не работают или фикстуры не используются
3 (удовлетворительно)	Базовый	Простые фикстуры, 5+ тестов
4 (хорошо)	Средний	+ yield, teardown, scope, подготовка данных
5 (отлично)	Сложный	+ вложенные фикстуры, autouse, параметризация
Шпаргалка по фикстурам
python
# === ОСНОВНОЙ СИНТАКСИС ===

@pytest.fixture
def my_fixture():
    """Простая фикстура."""
    data = {"key": "value"}
    return data

def test_example(my_fixture):
    assert my_fixture["key"] == "value"


# === С TEARDOWN (YIELD) ===

@pytest.fixture
def resource():
    print("Setup")
    resource = create_resource()
    yield resource
    print("Teardown")
    resource.cleanup()


# === С ОБЛАСТЬЮ ВИДИМОСТИ ===

@pytest.fixture(scope="session")  # Один раз на все тесты
def session_fixture():
    return expensive_operation()


# === АВТОМАТИЧЕСКАЯ ===

@pytest.fixture(autouse=True)
def auto_fixture():
    """Выполняется для каждого теста автоматически."""
    setup_logging()
    yield
    cleanup_logging()


# === ПАРАМЕТРИЗОВАННАЯ ===

@pytest.fixture(params=[1, 2, 3])
def number_fixture(request):
    return request.param

def test_with_param(number_fixture):
    assert number_fixture in [1, 2, 3]  # Запустится 3 раза


# === ВЛОЖЕННЫЕ ===

@pytest.fixture
def parent_fixture():
    return {"value": 10}

@pytest.fixture
def child_fixture(parent_fixture):
    parent_fixture["value"] += 5
    return parent_fixture
Контрольные вопросы (для защиты)
Что такое фикстура в pytest и зачем она нужна?

В чём разница между return и yield в фикстуре?

Какие значения может принимать параметр scope? Когда какой использовать?

Как сделать фикстуру, которая выполняется автоматически для всех тестов?

Как параметризовать фикстуру?

Может ли фикстура использовать другую фикстуру? Приведите пример.

Как запустить тесты с verbose выводом, чтобы видеть сообщения setup/teardown?

В каком порядке выполняются фикстуры с разным scope?

Что произойдёт, если в фикстуре возникнет исключение до yield?

Как фикстуры помогают в тестировании баз данных?

Следующее занятие: ПР 3.7 — Мокирование в pytest (unittest.mock). Замена реальных зависимостей на моки.

text

