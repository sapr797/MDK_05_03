# Лекция 3.11: Интеграционное и модульное тестирование: продвинутые техники

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить продвинутые техники интеграционного и модульного тестирования, освоить методы тестирования взаимодействия между компонентами, научиться изолировать модули при интеграционном тестировании.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Различать модульное и интеграционное тестирование (ОК 01, ОК 02).
2. Применять продвинутые техники модульного тестирования (ПК 3.2).
3. Проектировать интеграционные тесты для взаимодействующих компонентов (ПК 3.2, ПК 3.3).
4. Использовать тестовые дублёры (mock, stub, fake) при интеграционном тестировании (ПК 3.2).

---

## 1. Модульное vs Интеграционное тестирование

### 1.1 Сравнение
┌─────────────────────────────────────────────────────────────────────────────┐
│ МОДУЛЬНОЕ vs ИНТЕГРАЦИОННОЕ ТЕСТИРОВАНИЕ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ МОДУЛЬНОЕ ТЕСТИРОВАНИЕ ИНТЕГРАЦИОННОЕ ТЕСТИРОВАНИЕ │
│ ┌─────────────────────────┐ ┌─────────────────────────┐ │
│ │ Тестируется один модуль │ │ Тестируется несколько │ │
│ │ в изоляции │ │ модулей вместе │ │
│ └─────────────────────────┘ └─────────────────────────┘ │
│ │
│ • Быстрые • Медленнее │
│ • Не требуют внешних зависимостей • Требуют БД, API, файлы │
│ • Легко локализовать ошибки • Сложнее локализовать ошибки │
│ • 70-80% всех тестов • 10-20% всех тестов │
│ • Пишут разработчики • Пишут QA и разработчики │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 1.2 Пирамида тестирования
/
/
/ \ UI-тесты (5%, медленные)
/__
/ \ Интеграционные тесты (15%)
/
/ \ Модульные тесты (80%, быстрые)
/____\

text

---

## 2. Продвинутые техники модульного тестирования

### 2.1 Тестирование с параметризацией

```python
import pytest


@pytest.mark.parametrize("a,b,expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
    (100, 200, 300),
])
def test_add(a, b, expected):
    assert add(a, b) == expected


@pytest.mark.parametrize("value,expected", [
    pytest.param("abc", True, id="valid_string"),
    pytest.param("", False, id="empty_string"),
    pytest.param(None, False, id="none_value"),
])
def test_validate(value, expected):
    assert validate(value) == expected
2.2 Тестирование свойств (property-based testing)
bash
pip install hypothesis
python
from hypothesis import given, strategies as st


@given(st.integers(), st.integers())
def test_commutative_property(a, b):
    """Тестирование коммутативности сложения."""
    assert add(a, b) == add(b, a)


@given(st.lists(st.integers()))
def test_reverse_property(numbers):
    """reversed(reversed(list)) == list."""
    assert reversed(reversed(numbers)) == numbers


@given(st.text())
def test_string_length_property(s):
    """len(s) == len(s[::-1])."""
    assert len(s) == len(s[::-1])
2.3 Тестирование исключений с параметризацией
python
@pytest.mark.parametrize("value,expected_exception", [
    ("abc", ValueError),
    (None, TypeError),
    ([1, 2, 3], TypeError),
])
def test_parse_int_with_exceptions(value, expected_exception):
    with pytest.raises(expected_exception):
        int(value)
2.4 Использование фикстур разного уровня
python
# conftest.py
import pytest


@pytest.fixture(scope="session")
def db_connection():
    """Один раз на всю сессию."""
    conn = create_connection()
    yield conn
    conn.close()


@pytest.fixture(scope="module")
def module_data():
    """Один раз на модуль."""
    return load_test_data()


@pytest.fixture
def fresh_state():
    """Для каждого теста."""
    return reset_state()
3. Продвинутые техники интеграционного тестирования
3.1 Тестирование с реальной БД
python
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker


@pytest.fixture(scope="function")
def db_session():
    """Фикстура с транзакционной изоляцией."""
    engine = create_engine("sqlite:///:memory:")
    connection = engine.connect()
    transaction = connection.begin()
    
    Session = sessionmaker(bind=connection)
    session = Session()
    
    # Создание таблиц
    Base.metadata.create_all(engine)
    
    yield session
    
    # Откат транзакции после теста
    transaction.rollback()
    connection.close()


class TestUserRepository:
    def test_save_user(self, db_session):
        user = User(name="Alice", email="alice@test.com")
        db_session.add(user)
        db_session.commit()
        
        saved = db_session.query(User).filter_by(name="Alice").first()
        assert saved is not None
        assert saved.email == "alice@test.com"
3.2 Тестирование REST API с реальным сервером
python
import pytest
from fastapi.testclient import TestClient
from main import app


@pytest.fixture
def client():
    return TestClient(app)


class TestAPI:
    def test_get_users(self, client):
        response = client.get("/users")
        assert response.status_code == 200
        assert isinstance(response.json(), list)
    
    def test_create_user(self, client):
        response = client.post("/users", json={
            "name": "Bob",
            "email": "bob@test.com"
        })
        assert response.status_code == 201
        assert response.json()["name"] == "Bob"
    
    def test_get_nonexistent_user(self, client):
        response = client.get("/users/999")
        assert response.status_code == 404
3.3 Тестирование с моком внешних сервисов
python
class TestPaymentIntegration:
    @pytest.fixture
    def mock_payment_gateway(self, mocker):
        mock = mocker.Mock()
        mock.charge.return_value = {"status": "success", "id": "tx_123"}
        return mock
    
    def test_process_order_with_mock_gateway(self, mock_payment_gateway):
        order_service = OrderService(payment_gateway=mock_payment_gateway)
        
        result = order_service.process_order(100)
        
        assert result["status"] == "completed"
        mock_payment_gateway.charge.assert_called_once_with(100)
    
    def test_payment_failure(self, mock_payment_gateway):
        mock_payment_gateway.charge.side_effect = PaymentError("Insufficient funds")
        
        order_service = OrderService(payment_gateway=mock_payment_gateway)
        
        with pytest.raises(PaymentError):
            order_service.process_order(100)
3.4 Тестирование с контейнерами (Docker)
bash
# test_database.py
import pytest
import testcontainers


@pytest.fixture(scope="module")
def postgres_container():
    from testcontainers.postgres import PostgresContainer
    
    with PostgresContainer("postgres:15") as postgres:
        yield postgres


def test_database_connection(postgres_container):
    connection_url = postgres_container.get_connection_url()
    engine = create_engine(connection_url)
    
    with engine.connect() as conn:
        result = conn.execute(text("SELECT 1"))
        assert result.scalar() == 1
4. Интеграционное тестирование с Test Doubles
4.1 Типы тестовых дублёров
Тип	Описание	Когда использовать
Dummy	Передаётся, но не используется	Для заполнения параметров
Stub	Возвращает предопределённые ответы	Когда нужны только возвращаемые значения
Spy	Запоминает вызовы для проверки	Когда нужно проверить, что метод был вызван
Mock	Предопределяет ожидания и проверяет их	Полноценная замена зависимости
Fake	Полноценная, но упрощённая реализация	Для тестов без мокирования
4.2 Примеры Test Doubles
python
# === STUB ===
class UserServiceStub:
    def get_user(self, user_id):
        return {"id": user_id, "name": "Test User"}

# === SPY ===
class EmailServiceSpy:
    def __init__(self):
        self.sent_emails = []
    
    def send(self, to, subject, body):
        self.sent_emails.append((to, subject, body))
        return True

# === FAKE ===
class FakeDatabase:
    def __init__(self):
        self._data = {}
    
    def save(self, key, value):
        self._data[key] = value
    
    def get(self, key):
        return self._data.get(key)


class TestIntegrationWithSpy:
    def test_notification_sent_on_user_registration(self):
        email_spy = EmailServiceSpy()
        user_service = UserService(email_service=email_spy)
        
        user_service.register("alice@test.com")
        
        assert len(email_spy.sent_emails) == 1
        assert email_spy.sent_emails[0][0] == "alice@test.com"
        assert "Welcome" in email_spy.sent_emails[0][1]
5. Стратегии интеграционного тестирования
5.1 Большой взрыв (Big Bang)
python
# Все модули интегрируются сразу
def test_big_bang_integration():
    # Запуск всех компонентов
    db = Database()
    api = APIServer()
    cache = RedisCache()
    
    # Тестирование полного сценария
    user_service = UserService(db, api, cache)
    result = user_service.process_user("alice")
    
    assert result["status"] == "success"
5.2 Инкрементная интеграция (сверху вниз)
python
# Тестирование с заменой нижних уровней на заглушки
class TestTopDownIntegration:
    def test_user_api_with_stub_repository(self):
        # Заглушка вместо реального репозитория
        stub_repo = StubUserRepository()
        user_api = UserAPI(repository=stub_repo)
        
        response = user_api.get_user(1)
        assert response["name"] == "Test User"
5.3 Инкрементная интеграция (снизу вверх)
python
# Тестирование с заглушками верхних уровней
class TestBottomUpIntegration:
    def test_repository_with_stub_notifier(self):
        stub_notifier = StubNotifier()
        repository = UserRepository(notifier=stub_notifier)
        
        user = repository.save({"name": "Alice"})
        assert user["id"] is not None
        # Проверка, что notifier был вызван
        assert stub_notifier.called
5.4 Санитарная интеграция (Sandwich)
python
# Комбинация top-down и bottom-up
def test_sandwich_integration():
    # Реальные модули среднего уровня
    # Заглушки для верхнего и нижнего уровней
    stub_api = StubAPIClient()
    stub_db = StubDatabase()
    
    service = BusinessService(stub_api, stub_db)
    result = service.process()
    
    assert result is not None
6. Шпаргалка
python
# === МОДУЛЬНОЕ ТЕСТИРОВАНИЕ ===
def test_unit():
    result = function(param)
    assert result == expected

# === ИНТЕГРАЦИОННОЕ ТЕСТИРОВАНИЕ ===
def test_integration(db_session):
    service = Service(db_session)
    result = service.do_something()
    assert result is not None

# === МОКИРОВАНИЕ ===
def test_with_mock(mocker):
    mock = mocker.patch("module.function")
    mock.return_value = "mocked"

# === FAKE ===
class FakeDB:
    def get(self, id):
        return {"id": id}

# === SPY ===
class Spy:
    def __init__(self):
        self.called = False
    
    def method(self):
        self.called = True
Контрольные вопросы
В чём основное различие между модульным и интеграционным тестированием?

Как параметризация помогает в модульном тестировании?

Что такое property-based testing и когда его использовать?

Как организовать тестирование с реальной базой данных?

Как тестировать API с помощью TestClient?

Какие типы тестовых дублёров существуют?

В чём разница между Mock и Stub?

Какие стратегии интеграционного тестирования вы знаете?

Когда стоит использовать Big Bang интеграцию?

Как тестировать взаимодействие с внешними сервисами без реального доступа?

Следующее занятие: ПР 3.11 — Практическое интеграционное и модульное тестирование.
