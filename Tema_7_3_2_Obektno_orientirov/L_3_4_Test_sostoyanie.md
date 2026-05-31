# Лекция 3.4: Тестирование состояния объектов. Инварианты классов. Тестирование иерархий классов и полиморфизма

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.2:** Объектно-ориентированное тестирование  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить методы тестирования состояния объектов, освоить проверку инвариантов классов, научиться тестировать иерархии классов и полиморфное поведение.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Тестировать состояние объектов до и после вызова методов (ПК 3.2, ПК 3.3).
2. Проверять инварианты классов и соблюдение LSP (ОК 02, ОК 05).
3. Тестировать иерархии классов с использованием полиморфных тестов (ПК 3.2).
4. Применять параметризацию по классам для тестирования полиморфизма (ПК 3.2).

---

## 1. Тестирование состояния объектов

### 1.1 Что такое состояние объекта?

**Состояние объекта** — совокупность значений всех его атрибутов в определённый момент времени.
┌─────────────────────────────────────────────────────────────────────────────┐
│ ТЕСТИРОВАНИЕ СОСТОЯНИЯ ОБЪЕКТА │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ Состояние объекта = {атрибут1: значение1, атрибут2: значение2, ...} │
│ │
│ ПРИНЦИПЫ ТЕСТИРОВАНИЯ СОСТОЯНИЯ: │
│ │
│ 1. Начальное состояние (конструктор) │
│ └── Проверить, что все атрибуты инициализированы корректно │
│ │
│ 2. Состояние после мутирующих методов │
│ └── Проверить, какие атрибуты изменились, какие нет │
│ │
│ 3. Состояние после немутирующих методов │
│ └── Состояние не должно измениться │
│ │
│ 4. Состояние после исключений │
│ └── При ошибке состояние не должно быть испорчено │
│ │
│ 5. Инварианты класса │
│ └── Условия, которые всегда истинны │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 1.2 Пример: тестирование состояния BankAccount

```python
import pytest
from datetime import datetime


class BankAccount:
    """Банковский счёт с отслеживанием состояния."""
    
    def __init__(self, owner: str, initial_balance: float = 0):
        self.owner = owner
        self.balance = initial_balance
        self._transactions = []
        self.is_active = True
        self.created_at = datetime.now()
        
        # Инвариант: баланс не может быть отрицательным
        if self.balance < 0:
            raise ValueError("Initial balance cannot be negative")
    
    def deposit(self, amount: float) -> None:
        """Пополнение счёта."""
        if amount <= 0:
            raise ValueError("Deposit amount must be positive")
        self.balance += amount
        self._transactions.append(("deposit", amount, datetime.now()))
    
    def withdraw(self, amount: float) -> None:
        """Снятие средств."""
        if amount <= 0:
            raise ValueError("Withdrawal amount must be positive")
        if amount > self.balance:
            raise ValueError("Insufficient funds")
        self.balance -= amount
        self._transactions.append(("withdraw", amount, datetime.now()))
    
    def get_balance(self) -> float:
        """Получение баланса (немутирующий)."""
        return self.balance
    
    def get_transaction_count(self) -> int:
        """Количество транзакций."""
        return len(self._transactions)
    
    def deactivate(self) -> None:
        """Деактивация счёта."""
        self.is_active = False
    
    def activate(self) -> None:
        """Активация счёта."""
        self.is_active = True


class TestBankAccountState:
    """Тестирование состояния BankAccount."""
    
    # ==========================================================
    # 1. ПРОВЕРКА НАЧАЛЬНОГО СОСТОЯНИЯ
    # ==========================================================
    
    def test_initial_state(self):
        """Проверка состояния после создания."""
        account = BankAccount("Alice", 1000)
        
        assert account.owner == "Alice"
        assert account.balance == 1000
        assert account.is_active is True
        assert account.get_transaction_count() == 0
        assert isinstance(account.created_at, datetime)
    
    def test_initial_state_with_zero_balance(self):
        """Счёт с нулевым балансом."""
        account = BankAccount("Bob", 0)
        assert account.balance == 0
    
    def test_initial_state_negative_balance_raises(self):
        """Отрицательный начальный баланс -> исключение."""
        with pytest.raises(ValueError, match="negative"):
            BankAccount("Charlie", -100)
    
    # ==========================================================
    # 2. СОСТОЯНИЕ ПОСЛЕ МУТИРУЮЩИХ МЕТОДОВ
    # ==========================================================
    
    def test_state_after_deposit(self):
        """Проверка состояния после пополнения."""
        account = BankAccount("Alice", 1000)
        original_transaction_count = account.get_transaction_count()
        
        account.deposit(500)
        
        assert account.balance == 1500
        assert account.get_transaction_count() == original_transaction_count + 1
        assert account.is_active is True  # Не изменилось
    
    def test_state_after_withdraw(self):
        """Проверка состояния после снятия."""
        account = BankAccount("Alice", 1000)
        
        account.withdraw(300)
        
        assert account.balance == 700
        assert account.get_transaction_count() == 1
    
    def test_state_after_multiple_operations(self):
        """Состояние после нескольких операций."""
        account = BankAccount("Alice", 1000)
        
        account.deposit(200)
        account.withdraw(100)
        account.deposit(50)
        account.withdraw(30)
        
        assert account.balance == 1120
        assert account.get_transaction_count() == 4
    
    # ==========================================================
    # 3. НЕМУТИРУЮЩИЕ МЕТОДЫ НЕ МЕНЯЮТ СОСТОЯНИЕ
    # ==========================================================
    
    def test_state_unchanged_after_get_balance(self):
        """get_balance не меняет состояние."""
        account = BankAccount("Alice", 1000)
        original_balance = account.balance
        original_count = account.get_transaction_count()
        
        balance = account.get_balance()
        
        assert balance == original_balance
        assert account.balance == original_balance
        assert account.get_transaction_count() == original_count
    
    # ==========================================================
    # 4. СОСТОЯНИЕ ПОСЛЕ ИСКЛЮЧЕНИЙ
    # ==========================================================
    
    def test_state_unchanged_after_invalid_deposit(self):
        """При ошибке пополнения состояние не меняется."""
        account = BankAccount("Alice", 1000)
        original_balance = account.balance
        original_count = account.get_transaction_count()
        
        with pytest.raises(ValueError, match="positive"):
            account.deposit(-100)
        
        assert account.balance == original_balance
        assert account.get_transaction_count() == original_count
    
    def test_state_unchanged_after_insufficient_funds(self):
        """При ошибке снятия состояние не меняется."""
        account = BankAccount("Alice", 100)
        original_balance = account.balance
        
        with pytest.raises(ValueError, match="Insufficient funds"):
            account.withdraw(200)
        
        assert account.balance == original_balance
    
    # ==========================================================
    # 5. ПРОВЕРКА ИЗМЕНЕНИЯ КОНКРЕТНЫХ АТРИБУТОВ
    # ==========================================================
    
    def test_deactivate_changes_is_active(self):
        """deactivate должен изменить is_active на False."""
        account = BankAccount("Alice", 1000)
        assert account.is_active is True
        
        account.deactivate()
        
        assert account.is_active is False
        # Другие атрибуты не должны измениться
        assert account.balance == 1000
    
    def test_activate_changes_is_active(self):
        """activate должен изменить is_active на True."""
        account = BankAccount("Alice", 1000)
        account.deactivate()
        assert account.is_active is False
        
        account.activate()
        
        assert account.is_active is True
2. Инварианты классов
2.1 Что такое инвариант класса?
Инвариант класса — условие, которое должно оставаться истинным для всех объектов класса на всём протяжении их существования (кроме моментов внутри методов).

text
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ИНВАРИАНТЫ КЛАССА                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Примеры инвариантов:                                                      │
│   • balance >= 0 (баланс не отрицательный)                                  │
│   • 0 <= age <= 150 (возраст в разумных пределах)                           │
│   • len(users) == len(emails) (согласованность данных)                      │
│   • is_active == False -> balance == 0 (неактивный счёт пуст)               │
│                                                                              │
│   ПРОВЕРКА ИНВАРИАНТОВ:                                                     │
│                                                                              │
│   def check_invariant(self):                                               │
│       assert self.balance >= 0                                             │
│       assert 0 <= self.age <= 150                                          │
│                                                                              │
│   В тестах:                                                                 │
│   def test_invariant_preserved(self):                                      │
│       account = BankAccount("Alice", 1000)                                 │
│       assert account.check_invariant()                                     │
│       account.deposit(500)                                                 │
│       assert account.check_invariant()                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
2.2 Пример: реализация и тестирование инвариантов
python
class Order:
    """Заказ в интернет-магазине."""
    
    def __init__(self, customer_id: int):
        self.customer_id = customer_id
        self._items = []
        self._total = 0.0
        self.status = "created"
    
    def add_item(self, product_name: str, price: float, quantity: int = 1):
        """Добавляет товар в заказ."""
        if price <= 0:
            raise ValueError("Price must be positive")
        if quantity <= 0:
            raise ValueError("Quantity must be positive")
        
        self._items.append({
            "name": product_name,
            "price": price,
            "quantity": quantity
        })
        self._total += price * quantity
    
    def remove_item(self, index: int):
        """Удаляет товар из заказа."""
        if 0 <= index < len(self._items):
            item = self._items.pop(index)
            self._total -= item["price"] * item["quantity"]
    
    def get_total(self) -> float:
        return self._total
    
    def get_items_count(self) -> int:
        return len(self._items)
    
    def confirm(self):
        """Подтверждает заказ."""
        if self._total == 0:
            raise ValueError("Cannot confirm empty order")
        self.status = "confirmed"
    
    # ==========================================================
    # ПРОВЕРКА ИНВАРИАНТОВ
    # ==========================================================
    
    def check_invariant(self) -> bool:
        """
        Проверяет инварианты класса:
        1. _total должен быть равен сумме price * quantity по всем товарам
        2. _total не может быть отрицательным
        """
        calculated_total = sum(item["price"] * item["quantity"] for item in self._items)
        
        if abs(self._total - calculated_total) > 0.001:
            return False
        if self._total < 0:
            return False
        return True


class TestOrderInvariants:
    """Тестирование инвариантов Order."""
    
    def test_invariant_after_creation(self):
        """Инвариант соблюдается после создания."""
        order = Order(customer_id=1)
        assert order.check_invariant() is True
    
    def test_invariant_after_add_item(self):
        """Инвариант соблюдается после добавления товара."""
        order = Order(customer_id=1)
        order.add_item("Laptop", 1000, 1)
        assert order.check_invariant() is True
    
    def test_invariant_after_multiple_adds(self):
        """Инвариант соблюдается после нескольких добавлений."""
        order = Order(customer_id=1)
        order.add_item("Mouse", 50, 2)
        order.add_item("Keyboard", 150, 1)
        assert order.check_invariant() is True
        assert order.get_total() == 250
    
    def test_invariant_after_remove_item(self):
        """Инвариант соблюдается после удаления товара."""
        order = Order(customer_id=1)
        order.add_item("Laptop", 1000, 1)
        order.add_item("Mouse", 50, 2)
        assert order.check_invariant() is True
        
        order.remove_item(1)  # Удаляем мышь
        assert order.check_invariant() is True
        assert order.get_total() == 1000
    
    def test_invariant_preserved_after_error(self):
        """При ошибке инвариант не нарушается."""
        order = Order(customer_id=1)
        
        with pytest.raises(ValueError):
            order.add_item("Bad", -100, 1)
        
        assert order.check_invariant() is True
    
    def test_invariant_violation_detection(self):
        """Проверка, что тест detect'ит нарушение инварианта."""
        order = Order(customer_id=1)
        order.add_item("Item", 100, 1)
        
        # Нарушаем инвариант вручную
        order._total = 999  # Не соответствует реальной сумме
        
        assert order.check_invariant() is False
3. Тестирование иерархий классов
3.1 Стратегии тестирования иерархий
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                СТРАТЕГИИ ТЕСТИРОВАНИЯ ИЕРАРХИЙ КЛАССОВ                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   СТРАТЕГИЯ 1: Базовый тестовый класс                                       │
│   ├── Содержит общие тесты для всех наследников                             │
│   └── Наследники переопределяют create_instance()                           │
│                                                                              │
│   СТРАТЕГИЯ 2: Параметризация по классам                                    │
│   ├── Один тест, который запускается для разных классов                     │
│   └── @pytest.mark.parametrize("cls", [ClassA, ClassB])                     │
│                                                                              │
│   СТРАТЕГИЯ 3: Фикстура-фабрика                                             │
│   ├── Фикстура создаёт объекты разных типов                                 │
│   └── Тест работает с любым объектом через общий интерфейс                  │
│                                                                              │
│   СТРАТЕГИЯ 4: Тестирование контракта (LSP)                                 │
│   ├── Проверяет, что наследники соблюдают принцип LSP                       │
│   └── Функции с базовым типом работают с любым наследником                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
3.2 Пример: иерархия фигур
python
from abc import ABC, abstractmethod
import math


class Shape(ABC):
    """Абстрактный базовый класс фигуры."""
    
    @abstractmethod
    def area(self) -> float:
        pass
    
    @abstractmethod
    def perimeter(self) -> float:
        pass
    
    @abstractmethod
    def get_name(self) -> str:
        pass


class Circle(Shape):
    def __init__(self, radius: float):
        if radius <= 0:
            raise ValueError("Radius must be positive")
        self.radius = radius
    
    def area(self) -> float:
        return math.pi * self.radius ** 2
    
    def perimeter(self) -> float:
        return 2 * math.pi * self.radius
    
    def get_name(self) -> str:
        return "Circle"


class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        if width <= 0 or height <= 0:
            raise ValueError("Width and height must be positive")
        self.width = width
        self.height = height
    
    def area(self) -> float:
        return self.width * self.height
    
    def perimeter(self) -> float:
        return 2 * (self.width + self.height)
    
    def get_name(self) -> str:
        return "Rectangle"


class Square(Rectangle):
    def __init__(self, side: float):
        super().__init__(side, side)
    
    def get_name(self) -> str:
        return "Square"
3.3 Тестирование иерархии
python
import pytest
from shapes import Shape, Circle, Rectangle, Square


# ==========================================================
# СТРАТЕГИЯ 1: Базовый тестовый класс
# ==========================================================

class BaseShapeTest:
    """Базовый класс для тестирования всех фигур."""
    
    def create_shape(self) -> Shape:
        raise NotImplementedError
    
    def test_area_positive(self):
        """Площадь всегда положительна."""
        shape = self.create_shape()
        assert shape.area() > 0
    
    def test_perimeter_positive(self):
        """Периметр всегда положителен."""
        shape = self.create_shape()
        assert shape.perimeter() > 0
    
    def test_get_name_returns_non_empty_string(self):
        """Название фигуры — непустая строка."""
        shape = self.create_shape()
        assert isinstance(shape.get_name(), str)
        assert len(shape.get_name()) > 0


class TestCircle(BaseShapeTest):
    def create_shape(self) -> Shape:
        return Circle(radius=5)
    
    def test_circle_area_formula(self):
        circle = Circle(5)
        assert circle.area() == math.pi * 25
    
    def test_circle_radius_property(self):
        circle = Circle(10)
        assert circle.radius == 10


class TestRectangle(BaseShapeTest):
    def create_shape(self) -> Shape:
        return Rectangle(width=4, height=6)
    
    def test_rectangle_area_formula(self):
        rect = Rectangle(4, 6)
        assert rect.area() == 24


class TestSquare(BaseShapeTest):
    def create_shape(self) -> Shape:
        return Square(side=5)
    
    def test_square_inherits_rectangle(self):
        square = Square(5)
        assert isinstance(square, Rectangle)
        assert square.width == 5
        assert square.height == 5
        assert square.area() == 25


# ==========================================================
# СТРАТЕГИЯ 2: Параметризация по классам
# ==========================================================

class TestShapeParametrized:
    """Параметризованные полиморфные тесты."""
    
    @pytest.mark.parametrize("shape_class,params,expected_area", [
        (Circle, {"radius": 5}, math.pi * 25),
        (Rectangle, {"width": 4, "height": 6}, 24),
        (Square, {"side": 5}, 25),
    ])
    def test_area_calculation(self, shape_class, params, expected_area):
        shape = shape_class(**params)
        assert shape.area() == expected_area
    
    @pytest.mark.parametrize("shape_class,params,expected_perimeter", [
        (Circle, {"radius": 5}, 2 * math.pi * 5),
        (Rectangle, {"width": 4, "height": 6}, 2 * (4 + 6)),
        (Square, {"side": 5}, 20),
    ])
    def test_perimeter_calculation(self, shape_class, params, expected_perimeter):
        shape = shape_class(**params)
        assert shape.perimeter() == expected_perimeter


# ==========================================================
# СТРАТЕГИЯ 3: Фикстура-фабрика
# ==========================================================

@pytest.fixture(params=[
    Circle(5),
    Rectangle(4, 6),
    Square(5),
])
def any_shape(request):
    """Фикстура, возвращающая экземпляры разных фигур."""
    return request.param


def test_area_positive_with_fixture(any_shape):
    """Тест работает с любым типом фигуры."""
    assert any_shape.area() > 0


def test_perimeter_positive_with_fixture(any_shape):
    assert any_shape.perimeter() > 0


# ==========================================================
# СТРАТЕГИЯ 4: Тестирование LSP (принципа подстановки)
# ==========================================================

def test_lsp_compliance():
    """
    Принцип подстановки Лисков:
    Объекты дочернего класса должны быть заменяемы объектами базового класса.
    """
    
    # Функция, работающая с базовым типом
    def describe_shape(shape: Shape) -> str:
        return f"{shape.get_name()}: area={shape.area():.2f}, perimeter={shape.perimeter():.2f}"
    
    # Все наследники должны работать с этой функцией
    shapes = [Circle(5), Rectangle(4, 6), Square(5)]
    
    for shape in shapes:
        result = describe_shape(shape)
        assert shape.get_name() in result
        assert str(shape.area()) in result
4. Тестирование полиморфного поведения
4.1 Проверка полиморфных вызовов
python
class Animal(ABC):
    @abstractmethod
    def sound(self) -> str:
        pass
    
    @abstractmethod
    def move(self) -> str:
        pass


class Dog(Animal):
    def sound(self) -> str:
        return "Woof!"
    
    def move(self) -> str:
        return "Running on four legs"


class Bird(Animal):
    def sound(self) -> str:
        return "Chirp!"
    
    def move(self) -> str:
        return "Flying in the sky"


class Fish(Animal):
    def sound(self) -> str:
        return "Blub!"
    
    def move(self) -> str:
        return "Swimming in water"


class TestPolymorphism:
    """Тестирование полиморфного поведения."""
    
    @pytest.mark.parametrize("animal,expected_sound,expected_move", [
        (Dog(), "Woof!", "Running"),
        (Bird(), "Chirp!", "Flying"),
        (Fish(), "Blub!", "Swimming"),
    ])
    def test_polymorphic_behavior(self, animal, expected_sound, expected_move):
        """Полиморфное поведение через единый интерфейс."""
        assert expected_sound in animal.sound()
        assert expected_move in animal.move()
    
    def test_polymorphic_collection(self):
        """Коллекция полиморфных объектов."""
        animals = [Dog(), Bird(), Fish()]
        
        sounds = [a.sound() for a in animals]
        moves = [a.move() for a in animals]
        
        assert sounds == ["Woof!", "Chirp!", "Blub!"]
        assert "Running" in moves[0]
        assert "Flying" in moves[1]
        assert "Swimming" in moves[2]
    
    def test_polymorphic_function(self):
        """Функция, принимающая базовый тип."""
        def animal_info(animal: Animal) -> dict:
            return {
                "sound": animal.sound(),
                "move": animal.move()
            }
        
        info = animal_info(Dog())
        assert info["sound"] == "Woof!"
        
        info = animal_info(Bird())
        assert info["sound"] == "Chirp!"
5. Комплексный пример: интернет-магазин
python
class DiscountStrategy(ABC):
    @abstractmethod
    def apply(self, price: float) -> float:
        pass


class NoDiscount(DiscountStrategy):
    def apply(self, price: float) -> float:
        return price


class PercentageDiscount(DiscountStrategy):
    def __init__(self, percent: float):
        self.percent = percent
    
    def apply(self, price: float) -> float:
        return price * (100 - self.percent) / 100


class FixedDiscount(DiscountStrategy):
    def __init__(self, amount: float):
        self.amount = amount
    
    def apply(self, price: float) -> float:
        return max(0, price - self.amount)


class Product:
    def __init__(self, name: str, price: float):
        self.name = name
        self.price = price
    
    def get_final_price(self, strategy: DiscountStrategy) -> float:
        return strategy.apply(self.price)


class TestDiscountStrategies:
    """Тестирование полиморфизма на примере скидок."""
    
    @pytest.fixture
    def product(self):
        return Product("Laptop", 1000)
    
    @pytest.mark.parametrize("strategy,expected", [
        (NoDiscount(), 1000),
        (PercentageDiscount(10), 900),
        (PercentageDiscount(25), 750),
        (FixedDiscount(100), 900),
        (FixedDiscount(1000), 0),
    ])
    def test_discount_polymorphism(self, product, strategy, expected):
        """Разные стратегии скидок через единый интерфейс."""
        assert product.get_final_price(strategy) == expected
    
    def test_strategies_are_interchangeable(self):
        """Все стратегии можно подставлять в одну функцию."""
        product = Product("Mouse", 50)
        
        strategies = [NoDiscount(), PercentageDiscount(20), FixedDiscount(10)]
        results = [product.get_final_price(s) for s in strategies]
        
        assert results == [50, 40, 40]
6. Шпаргалка
python
# === ПРОВЕРКА СОСТОЯНИЯ ===
def test_state():
    obj = MyClass()
    assert obj.attr == initial
    
    obj.method()
    assert obj.attr == new_value

# === ИНВАРИАНТЫ ===
def test_invariant():
    obj = MyClass()
    assert obj.check_invariant()
    obj.do_something()
    assert obj.check_invariant()

# === БАЗОВЫЙ ТЕСТ ДЛЯ ИЕРАРХИИ ===
class BaseTest:
    def create_instance(self):
        raise NotImplementedError
    
    def test_common(self):
        obj = self.create_instance()
        assert obj.common_method() == expected

# === ПАРАМЕТРИЗАЦИЯ ПО КЛАССАМ ===
@pytest.mark.parametrize("cls", [ClassA, ClassB])
def test_polymorphism(cls):
    obj = cls()
    assert obj.method() is not None

# === ФИКСТУРА-ФАБРИКА ===
@pytest.fixture(params=[ClassA(), ClassB()])
def obj(request):
    return request.param

def test_factory(obj):
    assert obj.method() is not None
Контрольные вопросы
Что такое состояние объекта и как его тестировать?

Почему важно проверять состояние после исключений?

Что такое инвариант класса? Приведите пример.

Как проверить, что инвариант соблюдается после всех операций?

Какие стратегии тестирования иерархий классов существуют?

Как параметризация по классам помогает в тестировании полиморфизма?

Что такое принцип подстановки Лисков (LSP) и как его тестировать?

Как фикстура-фабрика упрощает тестирование иерархий?

Как проверить, что полиморфная функция корректно работает со всеми наследниками?

Какие ошибки в ОО-дизайне можно выявить с помощью тестирования состояния?

Следующее занятие: ПР 3.4 — Практическое тестирование состояния объектов и иерархий классов.
