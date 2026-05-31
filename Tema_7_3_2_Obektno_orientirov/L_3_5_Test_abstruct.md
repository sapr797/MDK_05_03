# Лекция 3.5: Тестирование абстрактных классов и интерфейсов. Тестирование паттернов Factory и Strategy

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.2:** Объектно-ориентированное тестирование  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить методы тестирования абстрактных классов и интерфейсов, освоить подходы к тестированию паттернов Factory и Strategy, научиться проверять соблюдение контрактов.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Тестировать абстрактные классы через конкретные реализации-заглушки (ПК 3.2, ПК 3.3).
2. Проверять соблюдение контракта интерфейса (ОК 02, ОК 05).
3. Тестировать фабричный паттерн (Factory) для создания объектов (ПК 3.2).
4. Тестировать паттерн Strategy для проверки взаимозаменяемых алгоритмов (ПК 3.2).

---

## 1. Тестирование абстрактных классов и интерфейсов

### 1.1 Проблема тестирования абстрактных классов
┌─────────────────────────────────────────────────────────────────────────────┐
│ ПРОБЛЕМЫ ТЕСТИРОВАНИЯ АБСТРАКТНЫХ КЛАССОВ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ Проблема 1: Нельзя создать экземпляр │
│ └── TypeError: Can't instantiate abstract class │
│ │
│ Проблема 2: Нужно тестировать конкретные методы │
│ └── Абстрактный класс может иметь реализованные методы │
│ │
│ Проблема 3: Контракт должны соблюдать все наследники │
│ └── Нужно проверить, что наследники корректно реализуют интерфейс │
│ │
│ РЕШЕНИЯ: │
│ • Тестовый наследник (минимальная реализация) │
│ • Мокирование с create_autospec │
│ • Базовый тестовый класс для иерархии │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 1.2 Пример: абстрактный репозиторий

```python
from abc import ABC, abstractmethod
from typing import List, Optional, Dict, Any


class Repository(ABC):
    """Абстрактный репозиторий для работы с данными."""
    
    @abstractmethod
    def save(self, entity: Dict[str, Any]) -> int:
        """Сохраняет сущность, возвращает ID."""
        pass
    
    @abstractmethod
    def find_by_id(self, entity_id: int) -> Optional[Dict[str, Any]]:
        """Находит сущность по ID."""
        pass
    
    @abstractmethod
    def delete(self, entity_id: int) -> bool:
        """Удаляет сущность по ID."""
        pass
    
    @abstractmethod
    def get_all(self) -> List[Dict[str, Any]]:
        """Возвращает все сущности."""
        pass
    
    # Конкретный метод, общий для всех реализаций
    def exists(self, entity_id: int) -> bool:
        """Проверяет существование сущности."""
        return self.find_by_id(entity_id) is not None
    
    def save_all(self, entities: List[Dict[str, Any]]) -> List[int]:
        """Сохраняет несколько сущностей."""
        return [self.save(entity) for entity in entities]


# ==========================================================
# ТЕСТОВАЯ РЕАЛИЗАЦИЯ (для тестирования конкретных методов)
# ==========================================================

class TestableRepository(Repository):
    """
    Тестовая реализация репозитория с in-memory хранилищем.
    """
    
    def __init__(self):
        self._storage: Dict[int, Dict[str, Any]] = {}
        self._next_id = 1
    
    def save(self, entity: Dict[str, Any]) -> int:
        entity_id = self._next_id
        self._storage[entity_id] = entity.copy()
        self._next_id += 1
        return entity_id
    
    def find_by_id(self, entity_id: int) -> Optional[Dict[str, Any]]:
        return self._storage.get(entity_id)
    
    def delete(self, entity_id: int) -> bool:
        if entity_id in self._storage:
            del self._storage[entity_id]
            return True
        return False
    
    def get_all(self) -> List[Dict[str, Any]]:
        return list(self._storage.values())


# ==========================================================
# ТЕСТИРОВАНИЕ КОНКРЕТНЫХ МЕТОДОВ АБСТРАКТНОГО КЛАССА
# ==========================================================

class TestAbstractRepository:
    """Тесты для конкретных методов Repository."""
    
    def test_exists_returns_true_for_existing(self):
        repo = TestableRepository()
        entity_id = repo.save({"name": "test"})
        
        assert repo.exists(entity_id) is True
    
    def test_exists_returns_false_for_nonexistent(self):
        repo = TestableRepository()
        
        assert repo.exists(999) is False
    
    def test_save_all_returns_ids_list(self):
        repo = TestableRepository()
        entities = [
            {"name": "first"},
            {"name": "second"},
            {"name": "third"}
        ]
        
        ids = repo.save_all(entities)
        
        assert len(ids) == 3
        assert ids == [1, 2, 3]
        assert len(repo.get_all()) == 3


# ==========================================================
# ТЕСТИРОВАНИЕ КОНТРАКТА (С ПОМОЩЬЮ create_autospec)
# ==========================================================

from unittest.mock import create_autospec

def test_mock_complies_with_interface():
    """Мок, созданный с create_autospec, соблюдает контракт."""
    mock_repo = create_autospec(Repository)
    
    # Методы существуют
    assert hasattr(mock_repo, "save")
    assert hasattr(mock_repo, "find_by_id")
    assert hasattr(mock_repo, "delete")
    assert hasattr(mock_repo, "get_all")
    assert hasattr(mock_repo, "exists")
    
    # Настройка поведения
    mock_repo.find_by_id.return_value = {"id": 1, "name": "Mock"}
    
    result = mock_repo.find_by_id(1)
    assert result["name"] == "Mock"
    mock_repo.find_by_id.assert_called_once_with(1)
2. Тестирование паттерна Factory
2.1 Что такое паттерн Factory?
Фабрика (Factory) — порождающий паттерн, предоставляющий интерфейс для создания объектов без указания конкретных классов.

text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ПАТТЕРН FACTORY                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Клиент ──► NotificationFactory.create("email") ──► EmailNotification      │
│                │                                                             │
│                ├── create("sms")   ──► SMSNotification                       │
│                ├── create("push")  ──► PushNotification                      │
│                └── create("slack") ──► SlackNotification                     │
│                                                                              │
│   ЧТО ТЕСТИРОВАТЬ:                                                          │
│   1. Создание объектов всех типов                                           │
│   2. Обработку неизвестного типа (исключение)                               │
│   3. Передачу параметров конструкторам                                       │
│   4. Batch-создание (create_batch)                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
2.2 Пример: фабрика уведомлений
python
from abc import ABC, abstractmethod
from datetime import datetime


class Notification(ABC):
    """Абстрактное уведомление."""
    
    def __init__(self, recipient: str):
        self.recipient = recipient
        self.created_at = datetime.now()
        self._sent = False
    
    @abstractmethod
    def send(self) -> bool:
        pass
    
    @abstractmethod
    def get_type(self) -> str:
        pass


class EmailNotification(Notification):
    def __init__(self, recipient: str, subject: str, body: str, sender: str = "noreply@test.com"):
        super().__init__(recipient)
        self.subject = subject
        self.body = body
        self.sender = sender
    
    def send(self) -> bool:
        if "@" not in self.recipient or not self.body:
            return False
        self._sent = True
        return True
    
    def get_type(self) -> str:
        return "email"


class SMSNotification(Notification):
    def __init__(self, phone_number: str, message: str):
        super().__init__(phone_number)
        self.message = message
    
    def send(self) -> bool:
        if len(self.recipient) < 10 or not self.message:
            return False
        self._sent = True
        return True
    
    def get_type(self) -> str:
        return "sms"


class PushNotification(Notification):
    def __init__(self, device_token: str, title: str, body: str):
        super().__init__(device_token)
        self.title = title
        self.body = body
    
    def send(self) -> bool:
        if len(self.recipient) < 10 or not self.body:
            return False
        self._sent = True
        return True
    
    def get_type(self) -> str:
        return "push"


# ==========================================================
# ФАБРИКА УВЕДОМЛЕНИЙ
# ==========================================================

class NotificationFactory:
    """Фабрика для создания уведомлений."""
    
    @staticmethod
    def create(notification_type: str, **kwargs) -> Notification:
        if notification_type == "email":
            return EmailNotification(
                recipient=kwargs.get("recipient", "default@test.com"),
                subject=kwargs.get("subject", "Default Subject"),
                body=kwargs.get("body", "Default body"),
                sender=kwargs.get("sender", "noreply@test.com")
            )
        elif notification_type == "sms":
            return SMSNotification(
                phone_number=kwargs.get("phone_number", "+1234567890"),
                message=kwargs.get("message", "Default SMS")
            )
        elif notification_type == "push":
            return PushNotification(
                device_token=kwargs.get("device_token", "default_token_1234567890"),
                title=kwargs.get("title", "Default Title"),
                body=kwargs.get("body", "Default body")
            )
        else:
            raise ValueError(f"Unknown notification type: {notification_type}")
    
    @staticmethod
    def create_batch(configs: List[Dict]) -> List[Notification]:
        """Создаёт несколько уведомлений."""
        return [NotificationFactory.create(**config) for config in configs]
    
    @staticmethod
    def get_supported_types() -> List[str]:
        return ["email", "sms", "push"]


# ==========================================================
# ТЕСТЫ ДЛЯ ФАБРИКИ
# ==========================================================

class TestNotificationFactory:
    """Тесты фабрики уведомлений."""
    
    @pytest.mark.parametrize("ntype,expected_class", [
        ("email", EmailNotification),
        ("sms", SMSNotification),
        ("push", PushNotification),
    ])
    def test_create_returns_correct_type(self, ntype, expected_class):
        """Параметризованный тест создания уведомлений."""
        notification = NotificationFactory.create(ntype)
        assert isinstance(notification, expected_class)
        assert notification.get_type() == ntype
    
    def test_create_email_with_custom_params(self):
        """Создание email с пользовательскими параметрами."""
        email = NotificationFactory.create(
            "email",
            recipient="user@test.com",
            subject="Hello",
            body="Test body",
            sender="admin@test.com"
        )
        assert email.recipient == "user@test.com"
        assert email.subject == "Hello"
        assert email.body == "Test body"
        assert email.sender == "admin@test.com"
    
    def test_create_unknown_type_raises(self):
        """Неизвестный тип вызывает исключение."""
        with pytest.raises(ValueError, match="Unknown notification type: telegram"):
            NotificationFactory.create("telegram")
    
    def test_create_batch(self):
        """Пакетное создание уведомлений."""
        configs = [
            {"notification_type": "email", "recipient": "a@test.com"},
            {"notification_type": "sms", "phone_number": "+1234567890"},
            {"notification_type": "push", "device_token": "token123"},
        ]
        
        notifications = NotificationFactory.create_batch(configs)
        
        assert len(notifications) == 3
        assert notifications[0].get_type() == "email"
        assert notifications[1].get_type() == "sms"
        assert notifications[2].get_type() == "push"
3. Тестирование паттерна Strategy
3.1 Что такое паттерн Strategy?
Стратегия (Strategy) — поведенческий паттерн, определяющий семейство взаимозаменяемых алгоритмов и позволяющий выбирать их на лету.

text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ПАТТЕРН STRATEGY                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                    ┌─────────────────┐                                      │
│                    │   Context       │                                      │
│                    │ (PaymentContext)│                                      │
│                    └────────┬────────┘                                      │
│                             │                                                │
│                             │ strategy                                       │
│                             ▼                                                │
│              ┌─────────────────────────┐                                    │
│              │   PaymentStrategy       │                                    │
│              │   (interface)           │                                    │
│              └─────────────────────────┘                                    │
│                         △                                                   │
│           ┌─────────────┼─────────────┬──────────────────────┐             │
│           │             │             │                      │             │
│   ┌───────┴──────┐ ┌─────┴─────┐ ┌─────┴─────┐        ┌───────┴──────┐      │
│   │CreditCard   │ │ PayPal    │ │ Crypto    │        │  Cash        │      │
│   │Strategy     │ │ Strategy  │ │ Strategy  │        │  Strategy    │      │
│   └─────────────┘ └───────────┘ └───────────┘        └──────────────┘      │
│                                                                              │
│   ЧТО ТЕСТИРОВАТЬ:                                                          │
│   1. Каждая стратегия работает корректно                                    │
│   2. Стратегии взаимозаменяемы (LSP)                                        │
│   3. Контекст правильно делегирует стратегии                                │
│   4. Можно динамически менять стратегию                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
3.2 Пример: стратегии оплаты
python
from abc import ABC, abstractmethod


class PaymentStrategy(ABC):
    """Абстрактная стратегия оплаты."""
    
    @abstractmethod
    def pay(self, amount: float) -> dict:
        """Выполняет платёж."""
        pass
    
    @abstractmethod
    def get_fee(self, amount: float) -> float:
        """Возвращает комиссию."""
        pass


class CreditCardStrategy(PaymentStrategy):
    """Оплата кредитной картой."""
    
    def __init__(self, card_number: str, card_holder: str, expiry: str, cvv: str):
        self.card_number = card_number
        self.card_holder = card_holder
        self.expiry = expiry
        self.cvv = cvv
    
    def pay(self, amount: float) -> dict:
        return {
            "success": True,
            "method": "credit_card",
            "amount": amount,
            "fee": self.get_fee(amount),
            "card_last4": self.card_number[-4:]
        }
    
    def get_fee(self, amount: float) -> float:
        return amount * 0.025  # 2.5%


class PayPalStrategy(PaymentStrategy):
    """Оплата через PayPal."""
    
    def __init__(self, email: str):
        self.email = email
    
    def pay(self, amount: float) -> dict:
        if "@" not in self.email:
            return {"success": False, "error": "Invalid email"}
        
        return {
            "success": True,
            "method": "paypal",
            "amount": amount,
            "fee": self.get_fee(amount),
            "email": self.email
        }
    
    def get_fee(self, amount: float) -> float:
        return amount * 0.035  # 3.5%


class CryptoStrategy(PaymentStrategy):
    """Оплата криптовалютой."""
    
    def __init__(self, wallet_address: str):
        self.wallet_address = wallet_address
    
    def pay(self, amount: float) -> dict:
        if len(self.wallet_address) < 10:
            return {"success": False, "error": "Invalid wallet address"}
        
        return {
            "success": True,
            "method": "crypto",
            "amount": amount,
            "fee": self.get_fee(amount),
            "wallet": self.wallet_address[:6] + "..."
        }
    
    def get_fee(self, amount: float) -> float:
        return amount * 0.01  # 1%


class PaymentContext:
    """Контекст для выполнения платежей с выбранной стратегией."""
    
    def __init__(self, strategy: PaymentStrategy = None):
        self._strategy = strategy
    
    def set_strategy(self, strategy: PaymentStrategy):
        """Изменяет стратегию динамически."""
        self._strategy = strategy
    
    def execute_payment(self, amount: float) -> dict:
        """Выполняет платёж с текущей стратегией."""
        if not self._strategy:
            raise ValueError("No payment strategy set")
        return self._strategy.pay(amount)
    
    def get_fee(self, amount: float) -> float:
        if not self._strategy:
            raise ValueError("No payment strategy set")
        return self._strategy.get_fee(amount)


# ==========================================================
# ТЕСТЫ ДЛЯ СТРАТЕГИЙ
# ==========================================================

class TestPaymentStrategies:
    """Тестирование стратегий оплаты."""
    
    @pytest.fixture
    def amount(self):
        return 100.0
    
    # ==========================================================
    # ТЕСТЫ ОТДЕЛЬНЫХ СТРАТЕГИЙ
    # ==========================================================
    
    def test_credit_card_strategy(self, amount):
        strategy = CreditCardStrategy("4111111111111111", "John Doe", "12/25", "123")
        result = strategy.pay(amount)
        
        assert result["success"] is True
        assert result["method"] == "credit_card"
        assert result["amount"] == amount
        assert result["fee"] == 2.5
        assert result["card_last4"] == "1111"
    
    def test_paypal_strategy_success(self, amount):
        strategy = PayPalStrategy("user@paypal.com")
        result = strategy.pay(amount)
        
        assert result["success"] is True
        assert result["method"] == "paypal"
        assert result["fee"] == 3.5
    
    def test_paypal_strategy_invalid_email(self, amount):
        strategy = PayPalStrategy("invalid")
        result = strategy.pay(amount)
        
        assert result["success"] is False
        assert "Invalid email" in result["error"]
    
    def test_crypto_strategy(self, amount):
        strategy = CryptoStrategy("1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa")
        result = strategy.pay(amount)
        
        assert result["success"] is True
        assert result["method"] == "crypto"
        assert result["fee"] == 1.0
    
    # ==========================================================
    # ТЕСТЫ ПОЛИМОРФНОГО ПОВЕДЕНИЯ СТРАТЕГИЙ
    # ==========================================================
    
    @pytest.mark.parametrize("strategy,expected_fee", [
        (CreditCardStrategy("1111", "Name", "12/25", "123"), 2.5),
        (PayPalStrategy("user@test.com"), 3.5),
        (CryptoStrategy("1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa"), 1.0),
    ])
    def test_all_strategies_have_fee(self, strategy, expected_fee, amount):
        """Все стратегии возвращают комиссию."""
        assert strategy.get_fee(amount) == expected_fee
    
    @pytest.mark.parametrize("strategy", [
        CreditCardStrategy("1111", "Name", "12/25", "123"),
        PayPalStrategy("user@test.com"),
        CryptoStrategy("1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa"),
    ])
    def test_all_strategies_return_success_dict(self, strategy, amount):
        """Все стратегии возвращают словарь с success."""
        result = strategy.pay(amount)
        assert "success" in result
        assert "method" in result
        assert "amount" in result
    
    # ==========================================================
    # ТЕСТЫ КОНТЕКСТА
    # ==========================================================
    
    def test_context_executes_strategy(self, amount):
        strategy = CreditCardStrategy("1111", "Name", "12/25", "123")
        context = PaymentContext(strategy)
        
        result = context.execute_payment(amount)
        
        assert result["method"] == "credit_card"
    
    def test_context_can_switch_strategy(self, amount):
        context = PaymentContext()
        
        # Первая стратегия
        context.set_strategy(CreditCardStrategy("1111", "Name", "12/25", "123"))
        result1 = context.execute_payment(amount)
        assert result1["method"] == "credit_card"
        
        # Смена стратегии
        context.set_strategy(PayPalStrategy("user@test.com"))
        result2 = context.execute_payment(amount)
        assert result2["method"] == "paypal"
    
    def test_context_without_strategy_raises(self, amount):
        context = PaymentContext()
        
        with pytest.raises(ValueError, match="No payment strategy"):
            context.execute_payment(amount)
    
    def test_context_delegates_fee_calculation(self, amount):
        strategy = CryptoStrategy("1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa")
        context = PaymentContext(strategy)
        
        fee = context.get_fee(amount)
        assert fee == 1.0
4. Комбинирование Factory и Strategy
python
class PaymentStrategyFactory:
    """Фабрика для создания стратегий оплаты."""
    
    @staticmethod
    def create(strategy_type: str, **kwargs) -> PaymentStrategy:
        if strategy_type == "credit_card":
            return CreditCardStrategy(
                card_number=kwargs.get("card_number"),
                card_holder=kwargs.get("card_holder"),
                expiry=kwargs.get("expiry"),
                cvv=kwargs.get("cvv")
            )
        elif strategy_type == "paypal":
            return PayPalStrategy(email=kwargs.get("email"))
        elif strategy_type == "crypto":
            return CryptoStrategy(wallet_address=kwargs.get("wallet_address"))
        else:
            raise ValueError(f"Unknown strategy: {strategy_type}")


class TestPaymentStrategyFactory:
    """Тесты фабрики стратегий."""
    
    @pytest.mark.parametrize("strategy_type,expected_class", [
        ("credit_card", CreditCardStrategy),
        ("paypal", PayPalStrategy),
        ("crypto", CryptoStrategy),
    ])
    def test_factory_creates_correct_strategy(self, strategy_type, expected_class):
        strategy = PaymentStrategyFactory.create(strategy_type)
        assert isinstance(strategy, expected_class)
    
    def test_factory_with_credit_card_params(self):
        strategy = PaymentStrategyFactory.create(
            "credit_card",
            card_number="4111111111111111",
            card_holder="John Doe",
            expiry="12/25",
            cvv="123"
        )
        assert strategy.card_number == "4111111111111111"
    
    def test_factory_unknown_strategy_raises(self):
        with pytest.raises(ValueError, match="Unknown strategy: bitcoin"):
            PaymentStrategyFactory.create("bitcoin")
    
    def test_factory_integration_with_context(self, amount):
        # Создаём стратегию через фабрику
        strategy = PaymentStrategyFactory.create("paypal", email="user@test.com")
        
        # Используем в контексте
        context = PaymentContext(strategy)
        result = context.execute_payment(amount)
        
        assert result["success"] is True
        assert result["method"] == "paypal"
5. Шпаргалка
python
# === ТЕСТИРОВАНИЕ АБСТРАКТНОГО КЛАССА ===
class TestableAbstract(AbstractClass):
    def abstract_method(self):
        return "test"

def test_concrete_method():
    obj = TestableAbstract()
    assert obj.concrete_method() == expected

# === ТЕСТИРОВАНИЕ ФАБРИКИ ===
def test_factory():
    obj = Factory.create("type", param="value")
    assert isinstance(obj, ExpectedClass)

# === ТЕСТИРОВАНИЕ СТРАТЕГИИ ===
def test_strategy():
    strategy = ConcreteStrategy()
    result = strategy.execute(data)
    assert result == expected

def test_context():
    context = Context(ConcreteStrategy())
    assert context.do_something() == expected
    context.set_strategy(OtherStrategy())
    assert context.do_something() == other_expected
Контрольные вопросы
Почему абстрактные классы нельзя тестировать напрямую?

Как создать тестовую реализацию абстрактного класса?

Что такое create_autospec и зачем он нужен?

Какие аспекты нужно тестировать в фабрике?

Как проверить обработку неизвестного типа в фабрике?

Что такое паттерн Strategy и для чего он используется?

Как тестировать взаимозаменяемость стратегий?

Как проверить, что контекст правильно делегирует стратегии?

Как можно комбинировать Factory и Strategy?

Какие преимущества даёт использование паттернов с точки зрения тестирования?

Следующее занятие: ПР 3.5 — Практическое тестирование абстрактных классов, паттернов Factory и Strategy.
