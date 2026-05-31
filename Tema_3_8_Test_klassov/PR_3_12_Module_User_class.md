# Практическое занятие 3.12: Тестирование классов и объектов

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.8:** Тестирование классов и объектов  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться тестировать классы и объекты: проверять конструкторы, методы, состояние объектов, использовать фикстуры для создания объектов и изоляции тестов.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Тестировать конструкторы классов и инициализацию состояния (ПК 3.2, ПК 3.3).
2. Проверять корректность работы методов, изменяющих состояние объекта (ПК 3.2).
3. Использовать фикстуры для создания объектов с разным состоянием (ПК 3.2).
4. Проверять инварианты класса после выполнения операций (ОК 02, ОК 05).

---

## Теоретическая справка

### Что нужно тестировать в классе

| Компонент | Что проверять |
|:---|:---|
| **Конструктор** | Правильная инициализация атрибутов, обработка невалидных параметров |
| **Геттеры** | Возвращают корректные значения |
| **Сеттеры/мутаторы** | Корректно изменяют состояние, проверяют входные данные |
| **Методы с логикой** | Возвращают ожидаемые значения, не нарушают инварианты |
| **Исключения** | Выбрасываются при неверных входных данных |

---

## Объект тестирования

### Листинг 1. Модуль user_class.py

```python
# user_class.py

from datetime import datetime
from typing import List, Dict, Optional
from enum import Enum


class UserStatus(Enum):
    """Статус пользователя."""
    ACTIVE = "active"
    INACTIVE = "inactive"
    BLOCKED = "blocked"


class Order:
    """Класс заказа."""
    
    def __init__(self, order_id: int, product: str, amount: float):
        self.order_id = order_id
        self.product = product
        self.amount = amount
        self.created_at = datetime.now()
        self.status = "new"
    
    def confirm(self) -> bool:
        """Подтверждение заказа."""
        if self.status != "new":
            return False
        self.status = "confirmed"
        return True
    
    def cancel(self) -> bool:
        """Отмена заказа."""
        if self.status == "cancelled":
            return False
        self.status = "cancelled"
        return True


class User:
    """Класс пользователя."""
    
    def __init__(self, user_id: int, name: str, email: str):
        """
        Конструктор пользователя.
        
        Args:
            user_id: Уникальный идентификатор (положительное целое)
            name: Имя пользователя (непустая строка)
            email: Email пользователя (валидный формат)
        
        Raises:
            ValueError: При неверных параметрах
        """
        if not isinstance(user_id, int) or user_id <= 0:
            raise ValueError("user_id must be a positive integer")
        
        if not name or not isinstance(name, str):
            raise ValueError("name must be a non-empty string")
        
        if not email or "@" not in email:
            raise ValueError("email must be a valid email address")
        
        self.user_id = user_id
        self.name = name
        self.email = email
        self.status = UserStatus.ACTIVE
        self._orders: List[Order] = []
        self.created_at = datetime.now()
    
    def activate(self) -> bool:
        """
        Активирует пользователя.
        
        Returns:
            True если статус изменён, False если уже активен
        """
        if self.status == UserStatus.ACTIVE:
            return False
        self.status = UserStatus.ACTIVE
        return True
    
    def deactivate(self) -> bool:
        """
        Деактивирует пользователя.
        
        Returns:
            True если статус изменён, False если уже неактивен
        """
        if self.status == UserStatus.INACTIVE:
            return False
        self.status = UserStatus.INACTIVE
        return True
    
    def block(self) -> bool:
        """
        Блокирует пользователя.
        
        Returns:
            True если статус изменён
        """
        self.status = UserStatus.BLOCKED
        return True
    
    def is_active(self) -> bool:
        """Проверяет, активен ли пользователь."""
        return self.status == UserStatus.ACTIVE
    
    def add_order(self, product: str, amount: float) -> Order:
        """
        Добавляет новый заказ пользователю.
        
        Args:
            product: Название товара
            amount: Сумма заказа (положительная)
        
        Returns:
            Созданный заказ
        
        Raises:
            ValueError: Если сумма не положительная
            PermissionError: Если пользователь не активен
        """
        if not self.is_active():
            raise PermissionError("Cannot add order for inactive user")
        
        if amount <= 0:
            raise ValueError("Order amount must be positive")
        
        order_id = len(self._orders) + 1
        order = Order(order_id, product, amount)
        self._orders.append(order)
        return order
    
    def get_orders(self) -> List[Order]:
        """Возвращает список заказов."""
        return self._orders.copy()
    
    def get_total_spent(self) -> float:
        """Возвращает общую сумму заказов."""
        return sum(order.amount for order in self._orders)
    
    def get_order_count(self) -> int:
        """Возвращает количество заказов."""
        return len(self._orders)
    
    def cancel_last_order(self) -> bool:
        """Отменяет последний заказ."""
        if not self._orders:
            return False
        last_order = self._orders[-1]
        return last_order.cancel()
    
    def get_info(self) -> Dict:
        """Возвращает информацию о пользователе."""
        return {
            "user_id": self.user_id,
            "name": self.name,
            "email": self.email,
            "status": self.status.value,
            "order_count": self.get_order_count(),
            "total_spent": self.get_total_spent(),
            "created_at": self.created_at.isoformat()
        }
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Тестирование конструктора и простых методов
Средний (варианты 9-17)	Средние студенты	+ тестирование add_order, get_orders, состояние
Сложный (варианты 18-25)	Сильные студенты	+ фикстуры-фабрики, комплексные сценарии
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться тестировать конструктор класса и методы, изменяющие состояние.

Задания для базового уровня
Задание 1.1. Тестирование конструктора (15 минут)
python
# test_user_class_base.py

import pytest
from datetime import datetime
from user_class import User, UserStatus


class TestUserConstructor:
    """Тесты конструктора класса User."""
    
    # ==========================================================
    # ПОЗИТИВНЫЕ СЦЕНАРИИ
    # ==========================================================
    
    def test_valid_constructor(self):
        """Создание пользователя с валидными данными."""
        user = User(user_id=1, name="Alice", email="alice@example.com")
        
        assert user.user_id == 1
        assert user.name == "Alice"
        assert user.email == "alice@example.com"
        assert user.status == UserStatus.ACTIVE
        assert user.get_order_count() == 0
        assert isinstance(user.created_at, datetime)
    
    def test_constructor_sets_correct_status(self):
        """Статус пользователя должен быть ACTIVE."""
        user = User(1, "Bob", "bob@test.com")
        assert user.is_active() is True
    
    def test_constructor_initializes_empty_orders(self):
        """Список заказов должен быть пустым."""
        user = User(1, "Charlie", "charlie@test.com")
        assert user.get_orders() == []
        assert user.get_total_spent() == 0.0
    
    # ==========================================================
    # НЕГАТИВНЫЕ СЦЕНАРИИ
    # ==========================================================
    
    def test_invalid_user_id_zero(self):
        """user_id = 0 -> ValueError."""
        with pytest.raises(ValueError, match="user_id must be a positive integer"):
            User(user_id=0, name="Test", email="test@test.com")
    
    def test_invalid_user_id_negative(self):
        """Отрицательный user_id -> ValueError."""
        with pytest.raises(ValueError, match="user_id must be a positive integer"):
            User(user_id=-5, name="Test", email="test@test.com")
    
    def test_invalid_user_id_type(self):
        """user_id не int -> ValueError."""
        with pytest.raises(ValueError, match="user_id must be a positive integer"):
            User(user_id="1", name="Test", email="test@test.com")
    
    def test_empty_name(self):
        """Пустое имя -> ValueError."""
        with pytest.raises(ValueError, match="name must be a non-empty string"):
            User(user_id=1, name="", email="test@test.com")
    
    def test_none_name(self):
        """None вместо имени -> ValueError."""
        with pytest.raises(ValueError, match="name must be a non-empty string"):
            User(user_id=1, name=None, email="test@test.com")
    
    def test_invalid_email_no_at_symbol(self):
        """Email без @ -> ValueError."""
        with pytest.raises(ValueError, match="email must be a valid email address"):
            User(user_id=1, name="Test", email="invalid-email")
    
    def test_empty_email(self):
        """Пустой email -> ValueError."""
        with pytest.raises(ValueError, match="email must be a valid email address"):
            User(user_id=1, name="Test", email="")
    
    def test_none_email(self):
        """None вместо email -> ValueError."""
        with pytest.raises(ValueError, match="email must be a valid email address"):
            User(user_id=1, name="Test", email=None)
Задание 1.2. Тестирование методов activate и deactivate (10 минут)
python
class TestUserActivation:
    """Тесты для activate/deactivate."""
    
    def test_new_user_is_active(self):
        """Новый пользователь активен по умолчанию."""
        user = User(1, "Alice", "alice@test.com")
        assert user.is_active() is True
    
    def test_deactivate_active_user(self):
        """Деактивация активного пользователя."""
        user = User(1, "Bob", "bob@test.com")
        
        result = user.deactivate()
        
        assert result is True
        assert user.is_active() is False
        assert user.status == UserStatus.INACTIVE
    
    def test_deactivate_inactive_user_returns_false(self):
        """Повторная деактивация возвращает False."""
        user = User(1, "Charlie", "charlie@test.com")
        user.deactivate()
        
        result = user.deactivate()
        
        assert result is False
        assert user.is_active() is False
    
    def test_activate_inactive_user(self):
        """Активация неактивного пользователя."""
        user = User(1, "Diana", "diana@test.com")
        user.deactivate()
        
        result = user.activate()
        
        assert result is True
        assert user.is_active() is True
        assert user.status == UserStatus.ACTIVE
    
    def test_activate_active_user_returns_false(self):
        """Активация уже активного пользователя возвращает False."""
        user = User(1, "Eve", "eve@test.com")
        
        result = user.activate()
        
        assert result is False
        assert user.is_active() is True
    
    def test_block_user(self):
        """Блокировка пользователя."""
        user = User(1, "Frank", "frank@test.com")
        
        result = user.block()
        
        assert result is True
        assert user.status == UserStatus.BLOCKED
        assert user.is_active() is False  # blocked не считается active
Задание 1.3. Тестирование get_info (5 минут)
python
class TestUserInfo:
    """Тесты для get_info."""
    
    def test_get_info_returns_correct_data(self):
        """get_info возвращает корректные данные."""
        user = User(42, "Grace", "grace@test.com")
        
        info = user.get_info()
        
        assert info["user_id"] == 42
        assert info["name"] == "Grace"
        assert info["email"] == "grace@test.com"
        assert info["status"] == "active"
        assert info["order_count"] == 0
        assert info["total_spent"] == 0.0
        assert "created_at" in info
    
    def test_get_info_after_deactivation(self):
        """get_info после деактивации."""
        user = User(1, "Henry", "henry@test.com")
        user.deactivate()
        
        info = user.get_info()
        
        assert info["status"] == "inactive"
Задание 1.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Результаты тестирования

| Компонент | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| Конструктор | _____ | _____ | _____ |
| activate/deactivate | _____ | _____ | _____ |
| get_info | _____ | _____ | _____ |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться тестировать методы add_order, get_orders, get_total_spent и проверять состояние объекта после операций.

Задания для среднего уровня
Задание 2.1. Тестирование add_order (15 минут)
python
# test_user_class_intermediate.py

import pytest
from user_class import User, Order, UserStatus


class TestUserOrders:
    """Тесты для методов работы с заказами."""
    
    @pytest.fixture
    def active_user(self):
        """Фикстура с активным пользователем."""
        return User(1, "Alice", "alice@test.com")
    
    @pytest.fixture
    def inactive_user(self):
        """Фикстура с неактивным пользователем."""
        user = User(2, "Bob", "bob@test.com")
        user.deactivate()
        return user
    
    # ==========================================================
    # ТЕСТЫ add_order
    # ==========================================================
    
    def test_add_order_to_active_user(self, active_user):
        """Добавление заказа активному пользователю."""
        order = active_user.add_order("Laptop", 1000.0)
        
        assert order is not None
        assert order.product == "Laptop"
        assert order.amount == 1000.0
        assert order.status == "new"
        assert active_user.get_order_count() == 1
        assert active_user.get_total_spent() == 1000.0
    
    def test_add_multiple_orders(self, active_user):
        """Добавление нескольких заказов."""
        order1 = active_user.add_order("Mouse", 25.0)
        order2 = active_user.add_order("Keyboard", 75.0)
        order3 = active_user.add_order("Monitor", 300.0)
        
        assert active_user.get_order_count() == 3
        assert active_user.get_total_spent() == 400.0
        
        assert order1.order_id == 1
        assert order2.order_id == 2
        assert order3.order_id == 3
    
    def test_add_order_to_inactive_user(self, inactive_user):
        """Добавление заказа неактивному пользователю -> PermissionError."""
        with pytest.raises(PermissionError, match="Cannot add order for inactive user"):
            inactive_user.add_order("Phone", 500.0)
        
        assert inactive_user.get_order_count() == 0
    
    def test_add_order_zero_amount(self, active_user):
        """Добавление заказа с нулевой суммой -> ValueError."""
        with pytest.raises(ValueError, match="Order amount must be positive"):
            active_user.add_order("Free", 0.0)
    
    def test_add_order_negative_amount(self, active_user):
        """Добавление заказа с отрицательной суммой -> ValueError."""
        with pytest.raises(ValueError, match="Order amount must be positive"):
            active_user.add_order("Negative", -100.0)
    
    # ==========================================================
    # ТЕСТЫ get_orders
    # ==========================================================
    
    def test_get_orders_returns_copy(self, active_user):
        """get_orders возвращает копию списка, а не оригинал."""
        active_user.add_order("Book", 15.0)
        
        orders = active_user.get_orders()
        orders.append(None)  # Изменение копии
        
        # Оригинальный список не изменился
        assert active_user.get_order_count() == 1
        assert len(active_user.get_orders()) == 1
    
    def test_get_orders_empty(self, active_user):
        """get_orders для пользователя без заказов."""
        assert active_user.get_orders() == []
    
    # ==========================================================
    # ТЕСТЫ get_total_spent
    # ==========================================================
    
    def test_get_total_spent_empty(self, active_user):
        """Общая сумма для пользователя без заказов."""
        assert active_user.get_total_spent() == 0.0
    
    def test_get_total_spent_after_orders(self, active_user):
        """Общая сумма после нескольких заказов."""
        active_user.add_order("Item1", 100.0)
        active_user.add_order("Item2", 200.0)
        active_user.add_order("Item3", 50.0)
        
        assert active_user.get_total_spent() == 350.0
    
    def test_get_total_spent_float_precision(self, active_user):
        """Проверка точности с плавающей точкой."""
        active_user.add_order("Precise", 10.10)
        active_user.add_order("Precise2", 20.20)
        
        assert active_user.get_total_spent() == 30.30
Задание 2.2. Тестирование get_order_count и cancel_last_order (10 минут)
python
class TestOrderManagement:
    """Тесты для управления заказами."""
    
    @pytest.fixture
    def user_with_orders(self):
        """Пользователь с несколькими заказами."""
        user = User(1, "Charlie", "charlie@test.com")
        user.add_order("Item1", 100.0)
        user.add_order("Item2", 200.0)
        user.add_order("Item3", 300.0)
        return user
    
    def test_get_order_count(self, user_with_orders):
        """get_order_count возвращает правильное количество."""
        assert user_with_orders.get_order_count() == 3
    
    def test_cancel_last_order_success(self, user_with_orders):
        """Отмена последнего заказа."""
        result = user_with_orders.cancel_last_order()
        
        assert result is True
        assert user_with_orders.get_order_count() == 3  # Заказ не удаляется
        # Проверяем статус последнего заказа
        orders = user_with_orders.get_orders()
        assert orders[-1].status == "cancelled"
    
    def test_cancel_last_order_when_no_orders(self, active_user):
        """Отмена заказа при пустом списке."""
        result = active_user.cancel_last_order()
        
        assert result is False
    
    def test_order_id_auto_increment(self, active_user):
        """ID заказов автоматически увеличиваются."""
        order1 = active_user.add_order("First", 10.0)
        order2 = active_user.add_order("Second", 20.0)
        order3 = active_user.add_order("Third", 30.0)
        
        assert order1.order_id == 1
        assert order2.order_id == 2
        assert order3.order_id == 3
Задание 2.3. Тестирование состояния после операций (10 минут)
python
class TestStateAfterOperations:
    """Тесты состояния объекта после операций."""
    
    @pytest.fixture
    def user(self):
        return User(1, "Diana", "diana@test.com")
    
    def test_state_after_add_order(self, user):
        """Проверка состояния после добавления заказа."""
        assert user.get_order_count() == 0
        
        user.add_order("Product", 99.99)
        
        assert user.get_order_count() == 1
        assert user.get_total_spent() == 99.99
        assert len(user.get_orders()) == 1
    
    def test_state_after_deactivate(self, user):
        """Проверка состояния после деактивации."""
        assert user.is_active() is True
        
        user.deactivate()
        
        assert user.is_active() is False
        assert user.status == UserStatus.INACTIVE
    
    def test_state_after_reactivate(self, user):
        """Проверка состояния после реактивации."""
        user.deactivate()
        assert user.is_active() is False
        
        user.activate()
        
        assert user.is_active() is True
        assert user.status == UserStatus.ACTIVE
    
    def test_state_invariant_maintained(self, user):
        """Инварианты класса должны сохраняться."""
        # Инвариант: количество заказов должно равняться длине списка
        user.add_order("A", 10)
        user.add_order("B", 20)
        
        assert user.get_order_count() == len(user.get_orders())
        
        # Инвариант: total_spent должно равняться сумме amount заказов
        total = sum(o.amount for o in user.get_orders())
        assert user.get_total_spent() == total
Задание 2.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Результаты тестирования

| Метод | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| add_order | _____ | _____ | _____ |
| get_orders | _____ | _____ | _____ |
| get_total_spent | _____ | _____ | _____ |
| cancel_last_order | _____ | _____ | _____ |
| Состояние | _____ | _____ | _____ |

### Проверенные сценарии

| Сценарий | Статус |
|:---|:---|
| Добавление заказа активному пользователю | ☐ |
| Добавление заказа неактивному пользователю | ☐ |
| Добавление заказа с отрицательной суммой | ☐ |
| Отмена последнего заказа | ☐ |

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться создавать фикстуры-фабрики, тестировать сложные сценарии и проверять интеграцию методов.

Задания для сложного уровня
Задание 3.1. Фикстуры-фабрики (15 минут)
python
# test_user_class_advanced.py

import pytest
from user_class import User, Order, UserStatus


class TestUserFixtures:
    """Тесты с использованием фикстур-фабрик."""
    
    @pytest.fixture
    def user_factory(self):
        """Фабрика для создания пользователей с разными параметрами."""
        def _create_user(user_id=None, name=None, email=None, status="active"):
            if user_id is None:
                user_id = 1
            if name is None:
                name = "Default User"
            if email is None:
                email = "default@test.com"
            
            user = User(user_id, name, email)
            
            if status == "inactive":
                user.deactivate()
            elif status == "blocked":
                user.block()
            
            return user
        
        return _create_user
    
    @pytest.fixture
    def user_with_orders_factory(self, user_factory):
        """Фабрика для создания пользователей с заказами."""
        def _create_user_with_orders(order_counts=0, order_amounts=None):
            user = user_factory()
            
            if order_amounts:
                for amount in order_amounts:
                    user.add_order(f"Product_{amount}", amount)
            elif order_counts > 0:
                for i in range(order_counts):
                    user.add_order(f"Product_{i}", (i + 1) * 10.0)
            
            return user
        
        return _create_user_with_orders
    
    def test_factory_default_user(self, user_factory):
        """Создание пользователя с параметрами по умолчанию."""
        user = user_factory()
        
        assert user.user_id == 1
        assert user.name == "Default User"
        assert user.email == "default@test.com"
        assert user.is_active() is True
    
    def test_factory_custom_user(self, user_factory):
        """Создание пользователя с пользовательскими параметрами."""
        user = user_factory(user_id=100, name="Custom", email="custom@test.com")
        
        assert user.user_id == 100
        assert user.name == "Custom"
        assert user.email == "custom@test.com"
    
    def test_factory_inactive_user(self, user_factory):
        """Создание неактивного пользователя."""
        user = user_factory(status="inactive")
        
        assert user.is_active() is False
        assert user.status == UserStatus.INACTIVE
    
    def test_factory_blocked_user(self, user_factory):
        """Создание заблокированного пользователя."""
        user = user_factory(status="blocked")
        
        assert user.status == UserStatus.BLOCKED
        assert user.is_active() is False
    
    def test_factory_with_orders(self, user_with_orders_factory):
        """Создание пользователя с заказами."""
        user = user_with_orders_factory(order_counts=5)
        
        assert user.get_order_count() == 5
        assert user.get_total_spent() == 150.0  # 10+20+30+40+50
    
    def test_factory_with_custom_amounts(self, user_with_orders_factory):
        """Создание пользователя с заказами по списку сумм."""
        user = user_with_orders_factory(order_amounts=[10.0, 20.0, 30.0])
        
        assert user.get_order_count() == 3
        assert user.get_total_spent() == 60.0
Задание 3.2. Комплексные сценарии (15 минут)
python
class TestComplexScenarios:
    """Комплексные сценарии использования."""
    
    @pytest.fixture
    def user(self):
        return User(1, "Eve", "eve@test.com")
    
    def test_user_lifecycle(self, user):
        """Полный жизненный цикл пользователя."""
        # 1. Создание пользователя
        assert user.is_active() is True
        assert user.get_order_count() == 0
        
        # 2. Добавление заказов
        user.add_order("Laptop", 1000.0)
        user.add_order("Mouse", 25.0)
        assert user.get_order_count() == 2
        assert user.get_total_spent() == 1025.0
        
        # 3. Деактивация
        user.deactivate()
        assert user.is_active() is False
        
        # 4. Попытка добавить заказ (должна быть ошибка)
        with pytest.raises(PermissionError):
            user.add_order("Keyboard", 50.0)
        
        # 5. Реактивация
        user.activate()
        assert user.is_active() is True
        
        # 6. Добавление заказа после реактивации
        user.add_order("Monitor", 300.0)
        assert user.get_order_count() == 3
        assert user.get_total_spent() == 1325.0
    
    def test_order_status_transitions(self, user):
        """Переходы статусов заказа."""
        order = user.add_order("Book", 30.0)
        
        assert order.status == "new"
        
        order.confirm()
        assert order.status == "confirmed"
        
        # Повторное подтверждение не меняет статус
        result = order.confirm()
        assert result is False
        assert order.status == "confirmed"
        
        order.cancel()
        assert order.status == "cancelled"
    
    def test_cancel_last_order_and_add_new(self, user):
        """Отмена последнего заказа и добавление нового."""
        user.add_order("First", 100.0)
        user.add_order("Second", 200.0)
        user.add_order("Third", 300.0)
        
        assert user.get_total_spent() == 600.0
        
        # Отменяем последний заказ
        user.cancel_last_order()
        
        # Сумма не должна измениться (заказ отменён, но не удалён)
        assert user.get_total_spent() == 600.0
        
        # Добавляем новый заказ
        user.add_order("Fourth", 400.0)
        
        assert user.get_order_count() == 4
        assert user.get_total_spent() == 1000.0
    
    def test_multiple_users_isolation(self, user_factory):
        """Проверка изоляции между разными пользователями."""
        alice = user_factory(user_id=1, name="Alice", email="alice@test.com")
        bob = user_factory(user_id=2, name="Bob", email="bob@test.com")
        
        alice.add_order("Alice's order", 100.0)
        bob.add_order("Bob's order", 200.0)
        
        # Заказы разных пользователей не смешиваются
        assert alice.get_total_spent() == 100.0
        assert bob.get_total_spent() == 200.0
        
        assert alice.get_order_count() == 1
        assert bob.get_order_count() == 1
        
        # Операции с одним пользователем не влияют на другого
        alice.deactivate()
        assert alice.is_active() is False
        assert bob.is_active() is True
Задание 3.3. Тестирование с параметризацией (10 минут)
python
class TestParametrized:
    """Параметризованные тесты."""
    
    @pytest.mark.parametrize("name,email,expected_error", [
        ("", "test@test.com", "name must be a non-empty string"),
        ("   ", "test@test.com", "name must be a non-empty string"),
        ("Valid", "invalid", "email must be a valid email address"),
        ("Valid", "", "email must be a valid email address"),
    ])
    def test_invalid_user_creation(self, name, email, expected_error):
        """Параметризованный тест неверных параметров конструктора."""
        with pytest.raises(ValueError, match=expected_error):
            User(1, name, email)
    
    @pytest.mark.parametrize("amounts,total", [
        ([], 0.0),
        ([10.0], 10.0),
        ([10.0, 20.0], 30.0),
        ([10.0, 20.0, 30.0, 40.0], 100.0),
    ])
    def test_total_spent_calculation(self, amounts, total):
        """Параметризованный тест расчёта общей суммы."""
        user = User(1, "Test", "test@test.com")
        
        for amount in amounts:
            user.add_order(f"Product_{amount}", amount)
        
        assert user.get_total_spent() == total
    
    @pytest.mark.parametrize("status,before_active,after_active", [
        ("active", True, False),
        ("inactive", False, True),
        ("blocked", False, False),
    ])
    def test_activate_scenarios(self, user_factory, status, before_active, after_active):
        """Параметризованный тест активации."""
        user = user_factory(status=status)
        
        assert user.is_active() == before_active
        
        result = user.activate()
        
        if before_active:
            assert result is False
        else:
            assert result is True
        
        assert user.is_active() == (after_active or status == "active")
Задание 3.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (сложный уровень)

**Студент:** _________________
**Вариант №:** ___ (18-25)

### Результаты тестирования

| Компонент | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| Конструктор | _____ | _____ | _____ |
| activate/deactivate/block | _____ | _____ | _____ |
| add_order | _____ | _____ | _____ |
| get_orders/get_total_spent | _____ | _____ | _____ |
| cancel_last_order | _____ | _____ | _____ |
| get_info | _____ | _____ | _____ |
| Фикстуры-фабрики | _____ | _____ | _____ |
| Комплексные сценарии | _____ | _____ | _____ |
| Параметризация | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Покрытие сценариев

| Сценарий | Покрыт |
|:---|:---|
| Конструктор с валидными данными | ☐ |
| Конструктор с невалидными данными | ☐ |
| Активация/деактивация | ☐ |
| Добавление заказа активному пользователю | ☐ |
| Добавление заказа неактивному пользователю | ☐ |
| Добавление заказа с неверной суммой | ☐ |
| Отмена заказа | ☐ |
| Полный жизненный цикл пользователя | ☐ |
| Изоляция между пользователями | ☐ |

### Найденные дефекты

| ID | Описание | Серьёзность | Статус |
|:---|:---|:---|:---|
| | | | |

### Вывод

_______________________________________________________________

_______________________________________________________________
Карточка студента
text
ПР 3.12. ТЕСТИРОВАНИЕ КЛАССОВ И ОБЪЕКТОВ

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ Тестирование конструктора (базовый)
□ Тестирование activate/deactivate (базовый)
□ Тестирование get_info (базовый)
□ Тестирование add_order (средний)
□ Тестирование get_orders/get_total_spent (средний)
□ Тестирование состояния (средний)
□ Фикстуры-фабрики (сложный)
□ Комплексные сценарии (сложный)
□ Параметризация тестов (сложный)

=== РЕЗУЛЬТАТЫ ===

Всего тестов: _____
Пройдено: _____
Не пройдено: _____

Найдено дефектов: _____

=== ВРЕМЯ ВЫПОЛНЕНИЯ ===

Подготовка: _____ мин
Выполнение: _____ мин
Оформление: _____ мин

=== ОТЧЁТ ===

Файлы:
- test_user_class_base.py
- test_user_class_intermediate.py
- test_user_class_advanced.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Тесты не работают или отсутствуют
3 (удовлетворительно)	Базовый	Конструктор + activate/deactivate (10+ тестов)
4 (хорошо)	Средний	+ add_order + get_orders + состояние (20+ тестов)
5 (отлично)	Сложный	+ фикстуры-фабрики + комплексные сценарии (30+ тестов)
Контрольные вопросы (для защиты)
Какие аспекты конструктора нужно тестировать?

Как проверить, что метод не меняет состояние при ошибке?

Зачем нужны фикстуры-фабрики для тестирования классов?

Как проверить, что get_orders() возвращает копию, а не оригинальный список?

Как протестировать, что несколько пользователей не влияют друг на друга?

Какие инварианты нужно проверять у класса User?

Как протестировать переходы состояний пользователя (active→inactive→active)?

Почему важно тестировать не только позитивные, но и негативные сценарии?

Как параметризация помогает в тестировании классов?

Как фикстуры помогают изолировать тесты?

Следующее занятие: Лекция 3.14 — Паттерны тестирования: Test Double, Mock, Stub, Fake.

text
