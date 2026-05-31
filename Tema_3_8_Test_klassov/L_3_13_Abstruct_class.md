# Лекция 3.13: Абстрактные классы и интерфейсы. Тестирование через конкретные реализации

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.8:** Тестирование классов и объектов  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить методы тестирования абстрактных классов и интерфейсов, освоить подходы к тестированию через конкретные реализации, научиться проверять соблюдение контрактов и изолировать тесты абстрактной логики.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Тестировать абстрактные классы через конкретные реализации-заглушки (ПК 3.2, ПК 3.3).
2. Проверять соблюдение контракта интерфейса (ОК 02, ОК 05).
3. Использовать тестовые дублёры (Test Double) для абстракций (ПК 3.2).
4. Применять паттерны «Тестовый наследник» и «Фейковая реализация» (ПК 3.3).

---

## 1. Абстрактные классы и интерфейсы: особенности тестирования

### 1.1 Что такое абстрактный класс?

**Абстрактный класс** — класс, который содержит один или несколько абстрактных методов (методов без реализации). Его нельзя инстанцировать напрямую.

```python
from abc import ABC, abstractmethod
from typing import List, Optional


class DataRepository(ABC):
    """Абстрактный репозиторий для работы с данными."""
    
    @abstractmethod
    def save(self, data: dict) -> int:
        """Сохраняет данные, возвращает ID."""
        pass
    
    @abstractmethod
    def find_by_id(self, id: int) -> Optional[dict]:
        """Находит данные по ID."""
        pass
    
    @abstractmethod
    def delete(self, id: int) -> bool:
        """Удаляет данные по ID."""
        pass
    
    @abstractmethod
    def get_all(self) -> List[dict]:
        """Возвращает все записи."""
        pass
    
    # Конкретный метод, общий для всех реализаций
    def exists(self, id: int) -> bool:
        """Проверяет существование записи."""
        return self.find_by_id(id) is not None
1.2 Проблемы тестирования абстрактных классов
text
┌─────────────────────────────────────────────────────────────────────────────┐
│              ПРОБЛЕМЫ ТЕСТИРОВАНИЯ АБСТРАКТНЫХ КЛАССОВ                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. НЕЛЬЗЯ СОЗДАТЬ ЭКЗЕМПЛЯР                                               │
│      ├── Нельзя написать `obj = AbstractClass()`                            │
│      └── Нужна конкретная реализация                                         │
│                                                                              │
│   2. НУЖНО ТЕСТИРОВАТЬ КОНКРЕТНЫЕ МЕТОДЫ                                     │
│      ├── Абстрактный класс может иметь готовые методы                       │
│      └── Их нужно тестировать, но без конкретной реализации нельзя         │
│                                                                              │
│   3. КОНТРАКТ ДОЛЖЕН СОБЛЮДАТЬСЯ ВСЕМИ НАСЛЕДНИКАМИ                          │
│      ├── Каждый наследник должен следовать контракту                        │
│      └── Нужно тестировать контракт на всех реализациях                     │
│                                                                              │
│   4. ИЗОЛЯЦИЯ ЛОГИКИ АБСТРАКТНОГО КЛАССА                                     │
│      ├── Нужно отделить тестирование логики абстракции                      │
│      └── От тестирования конкретных реализаций                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
2. Паттерн «Тестовый наследник»
2.1 Идея паттерна
Создаётся специальный тестовый класс-наследник, который реализует абстрактные методы минимальным образом (заглушками) для возможности тестирования конкретных методов абстрактного класса.

python
class TestableDataRepository(DataRepository):
    """
    Тестовая реализация абстрактного репозитория.
    Использует in-memory хранилище для тестов.
    """
    
    def __init__(self):
        self._storage: dict[int, dict] = {}
        self._next_id = 1
    
    def save(self, data: dict) -> int:
        data_id = self._next_id
        self._storage[data_id] = data.copy()
        self._next_id += 1
        return data_id
    
    def find_by_id(self, id: int) -> Optional[dict]:
        return self._storage.get(id)
    
    def delete(self, id: int) -> bool:
        if id in self._storage:
            del self._storage[id]
            return True
        return False
    
    def get_all(self) -> List[dict]:
        return list(self._storage.values())
2.2 Тестирование конкретных методов абстрактного класса
python
import pytest
from data_repository import DataRepository, TestableDataRepository


class TestAbstractDataRepository:
    """Тесты для конкретных методов абстрактного класса DataRepository."""
    
    @pytest.fixture
    def repository(self):
        """Фикстура с тестовой реализацией."""
        return TestableDataRepository()
    
    def test_exists_returns_true_when_exists(self, repository):
        """exists() возвращает True для существующей записи."""
        # Сохраняем данные через тестовую реализацию
        record_id = repository.save({"name": "test"})
        
        # Проверяем метод exists() абстрактного класса
        assert repository.exists(record_id) is True
    
    def test_exists_returns_false_when_not_exists(self, repository):
        """exists() возвращает False для несуществующей записи."""
        assert repository.exists(999) is False
    
    def test_exists_with_multiple_records(self, repository):
        """exists() работает корректно с несколькими записями."""
        id1 = repository.save({"name": "first"})
        id2 = repository.save({"name": "second"})
        
        assert repository.exists(id1) is True
        assert repository.exists(id2) is True
        assert repository.exists(999) is False
    
    def test_exists_does_not_modify_storage(self, repository):
        """exists() не должен изменять хранилище."""
        initial_count = len(repository.get_all())
        
        repository.exists(1)
        repository.exists(100)
        repository.exists(500)
        
        assert len(repository.get_all()) == initial_count
3. Паттерн «Фейковая реализация» (Fake)
3.1 Идея паттерна
Создаётся полноценная фейковая (но рабочая) реализация абстракции для использования в тестах.

python
class FakePaymentGateway(ABC):
    """
    Фейковая реализация платёжного шлюза для тестирования.
    Не вызывает реальные API, но имитирует поведение.
    """
    
    def __init__(self):
        self._transactions = []
        self._should_fail = False
        self._fail_message = None
    
    def set_fail_mode(self, should_fail: bool, message: str = ""):
        """Устанавливает режим имитации ошибки."""
        self._should_fail = should_fail
        self._fail_message = message
    
    def process_payment(self, amount: float, card_number: str) -> dict:
        """Имитирует обработку платежа."""
        if self._should_fail:
            raise PaymentError(self._fail_message or "Payment failed")
        
        transaction = {
            "id": len(self._transactions) + 1,
            "amount": amount,
            "card_last4": card_number[-4:],
            "status": "success"
        }
        self._transactions.append(transaction)
        return transaction
    
    def get_transaction_count(self) -> int:
        return len(self._transactions)
    
    def get_last_transaction(self) -> Optional[dict]:
        return self._transactions[-1] if self._transactions else None


class PaymentError(Exception):
    pass
3.2 Тестирование с использованием фейковой реализации
python
class TestFakePaymentGateway:
    """Тесты с использованием фейковой реализации."""
    
    @pytest.fixture
    def gateway(self):
        return FakePaymentGateway()
    
    def test_successful_payment(self, gateway):
        """Успешный платёж."""
        result = gateway.process_payment(100.00, "4111111111111111")
        
        assert result["status"] == "success"
        assert result["amount"] == 100.00
        assert result["card_last4"] == "1111"
        assert gateway.get_transaction_count() == 1
    
    def test_failed_payment(self, gateway):
        """Платёж с ошибкой."""
        gateway.set_fail_mode(True, "Insufficient funds")
        
        with pytest.raises(PaymentError, match="Insufficient funds"):
            gateway.process_payment(50.00, "4111111111111111")
        
        assert gateway.get_transaction_count() == 0
    
    def test_multiple_payments(self, gateway):
        """Несколько платежей подряд."""
        gateway.process_payment(10, "1111")
        gateway.process_payment(20, "2222")
        gateway.process_payment(30, "3333")
        
        assert gateway.get_transaction_count() == 3
        assert gateway.get_last_transaction()["amount"] == 30
4. Тестирование контракта интерфейса
4.1 Что такое контракт интерфейса?
Контракт интерфейса — набор правил, которым должны следовать все реализации интерфейса:

Типы параметров и возвращаемых значений

Ожидаемое поведение (постусловия)

Исключения, которые могут быть выброшены

Инварианты

4.2 Тестирование контракта для всех реализаций
python
from abc import ABC, abstractmethod


class Cache(ABC):
    """Интерфейс кэша."""
    
    @abstractmethod
    def get(self, key: str) -> Optional[str]:
        """Возвращает значение по ключу или None."""
        pass
    
    @abstractmethod
    def set(self, key: str, value: str, ttl: int = 60) -> None:
        """Устанавливает значение с TTL в секундах."""
        pass
    
    @abstractmethod
    def delete(self, key: str) -> bool:
        """Удаляет ключ, возвращает True если существовал."""
        pass
    
    @abstractmethod
    def clear(self) -> None:
        """Очищает весь кэш."""
        pass


class InMemoryCache(Cache):
    """In-memory реализация кэша."""
    
    def __init__(self):
        self._data: dict[str, tuple[str, float]] = {}
    
    def get(self, key: str) -> Optional[str]:
        if key not in self._data:
            return None
        
        value, expires_at = self._data[key]
        if expires_at < time.time():
            del self._data[key]
            return None
        
        return value
    
    def set(self, key: str, value: str, ttl: int = 60) -> None:
        expires_at = time.time() + ttl
        self._data[key] = (value, expires_at)
    
    def delete(self, key: str) -> bool:
        if key in self._data:
            del self._data[key]
            return True
        return False
    
    def clear(self) -> None:
        self._data.clear()


class BaseCacheTest(ABC):
    """
    Абстрактный тест для всех реализаций Cache.
    Все наследники должны пройти эти тесты.
    """
    
    @abstractmethod
    def create_cache(self) -> Cache:
        """Создаёт экземпляр кэша для тестирования."""
        pass
    
    def test_set_and_get(self):
        """Установка и получение значения."""
        cache = self.create_cache()
        
        cache.set("key1", "value1")
        
        assert cache.get("key1") == "value1"
    
    def test_get_nonexistent_key(self):
        """Получение несуществующего ключа."""
        cache = self.create_cache()
        
        assert cache.get("nonexistent") is None
    
    def test_overwrite_key(self):
        """Перезапись существующего ключа."""
        cache = self.create_cache()
        
        cache.set("key", "first")
        cache.set("key", "second")
        
        assert cache.get("key") == "second"
    
    def test_delete_key(self):
        """Удаление ключа."""
        cache = self.create_cache()
        cache.set("key", "value")
        
        assert cache.delete("key") is True
        assert cache.get("key") is None
    
    def test_delete_nonexistent_key(self):
        """Удаление несуществующего ключа."""
        cache = self.create_cache()
        
        assert cache.delete("nonexistent") is False
    
    def test_clear_all(self):
        """Очистка всего кэша."""
        cache = self.create_cache()
        cache.set("a", "1")
        cache.set("b", "2")
        cache.set("c", "3")
        
        cache.clear()
        
        assert cache.get("a") is None
        assert cache.get("b") is None
        assert cache.get("c") is None
    
    def test_keys_are_independent(self):
        """Ключи не влияют друг на друга."""
        cache = self.create_cache()
        cache.set("a", "first")
        cache.set("b", "second")
        
        assert cache.get("a") == "first"
        assert cache.get("b") == "second"
4.3 Конкретные тесты для каждой реализации
python
class TestInMemoryCache(BaseCacheTest):
    """Тесты для InMemoryCache."""
    
    def create_cache(self) -> Cache:
        return InMemoryCache()
    
    def test_ttl_expiration(self):
        """Тест TTL (с мокированием времени)."""
        cache = InMemoryCache()
        
        # Устанавливаем значение с TTL 1 секунда
        cache.set("temp", "value", ttl=1)
        
        assert cache.get("temp") == "value"
        
        time.sleep(1.1)
        assert cache.get("temp") is None
    
    def test_custom_ttl_values(self):
        """Разные значения TTL."""
        cache = InMemoryCache()
        
        cache.set("short", "short", ttl=1)
        cache.set("long", "long", ttl=10)
        
        assert cache.get("short") == "short"
        assert cache.get("long") == "long"
        
        time.sleep(1.1)
        
        assert cache.get("short") is None
        assert cache.get("long") == "long"
5. Тестирование через заглушки (Stubs)
5.1 Когда нужны заглушки
Заглушки используются, когда нужно протестировать абстрактный класс, не создавая полную фейковую реализацию.

python
class EmailSender(ABC):
    """Абстрактный отправитель писем."""
    
    @abstractmethod
    def send(self, to: str, subject: str, body: str) -> bool:
        pass
    
    def send_welcome_email(self, to: str, name: str) -> bool:
        """Отправляет приветственное письмо (конкретный метод)."""
        body = f"Welcome, {name}!"
        return self.send(to, "Welcome!", body)


class StubEmailSender(EmailSender):
    """Заглушка для тестирования конкретных методов."""
    
    def __init__(self):
        self.sent_messages = []
        self.should_fail = False
    
    def send(self, to: str, subject: str, body: str) -> bool:
        if self.should_fail:
            return False
        self.sent_messages.append((to, subject, body))
        return True


def test_send_welcome_email():
    """Тестирование конкретного метода абстрактного класса."""
    stub = StubEmailSender()
    
    result = stub.send_welcome_email("user@test.com", "Alice")
    
    assert result is True
    assert len(stub.sent_messages) == 1
    to, subject, body = stub.sent_messages[0]
    assert to == "user@test.com"
    assert subject == "Welcome!"
    assert body == "Welcome, Alice!"


def test_send_welcome_email_failure():
    """Тестирование сценария ошибки."""
    stub = StubEmailSender()
    stub.should_fail = True
    
    result = stub.send_welcome_email("user@test.com", "Bob")
    
    assert result is False
    assert len(stub.sent_messages) == 0
6. Паттерн «Тестовый двойник» (Mock) для абстракций
6.1 Использование unittest.mock с абстрактными классами
python
from unittest.mock import Mock, create_autospec


def test_with_mock_autospec():
    """Создание мока с сигнатурой абстрактного класса."""
    
    # create_autospec создаёт мок, который проверяет соответствие интерфейсу
    mock_repo = create_autospec(DataRepository)
    
    # Настройка поведения мока
    mock_repo.find_by_id.return_value = {"id": 1, "name": "test"}
    mock_repo.save.return_value = 100
    
    # Использование в тесте
    def process_data(repo: DataRepository):
        repo.save({"data": "processed"})
        return repo.find_by_id(1)
    
    result = process_data(mock_repo)
    
    # Проверка вызовов
    mock_repo.save.assert_called_once_with({"data": "processed"})
    mock_repo.find_by_id.assert_called_once_with(1)
    
    assert result == {"id": 1, "name": "test"}


def test_mock_ensures_contract():
    """create_autospec проверяет соответствие интерфейсу."""
    
    # Правильно: методы соответствуют интерфейсу
    mock_repo = create_autospec(DataRepository)
    mock_repo.save(1)  # ❌ Неправильный тип аргумента? Нет проверки в рантайме
    
    # create_autospec не проверяет типы в рантайме,
    # но проверяет наличие методов
    assert hasattr(mock_repo, "save")
    assert hasattr(mock_repo, "find_by_id")
    assert hasattr(mock_repo, "delete")
    assert hasattr(mock_repo, "get_all")
7. Шпаргалка
python
# === ТЕСТОВЫЙ НАСЛЕДНИК ===
class TestableAbstract(AbstractClass):
    def __init__(self):
        self.storage = {}
    
    def abstract_method(self):
        return self.storage.get("key")

def test_concrete_method():
    obj = TestableAbstract()
    assert obj.concrete_method() == expected

# === ФЕЙКОВАЯ РЕАЛИЗАЦИЯ ===
class FakeService(AbstractService):
    def __init__(self):
        self.calls = []
    
    def do_something(self, arg):
        self.calls.append(arg)
        return "mocked"

# === ТЕСТИРОВАНИЕ КОНТРАКТА ===
class BaseContractTest(ABC):
    @abstractmethod
    def create_instance(self):
        pass
    
    def test_contract_method(self):
        instance = self.create_instance()
        assert instance.method() is not None

# === ЗАГЛУШКА ===
class Stub(AbstractClass):
    def __init__(self):
        self.flag = False
    
    def abstract_method(self):
        return self.flag

# === МОК С АВТОСПЕКОМ ===
from unittest.mock import create_autospec
mock = create_autospec(AbstractClass)
mock.method.return_value = "value"
Контрольные вопросы
Почему абстрактные классы нельзя тестировать напрямую?

В чём суть паттерна «Тестовый наследник»?

Чем отличается фейковая реализация от заглушки (stub)?

Как тестировать конкретные методы абстрактного класса?

Что такое контракт интерфейса и как его тестировать?

Как проверить, что все реализации интерфейса соблюдают контракт?

Когда стоит использовать create_autospec из unittest.mock?

Как протестировать, что наследник реализовал все абстрактные методы?

Как изолировать тесты абстрактной логики от конкретных реализаций?

Какие проблемы могут возникнуть при тестировании через заглушки?

Следующее занятие: ПР 3.13 — Практическое тестирование абстрактных классов и интерфейсов.
