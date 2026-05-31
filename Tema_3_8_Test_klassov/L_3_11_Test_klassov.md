# Лекция 3.11: Тестирование классов и объектов

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.8:** Тестирование классов и объектов  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить методы тестирования классов и объектов, освоить проверку состояния объекта после вызова методов, верификацию инвариантов класса, а также научиться изолировать тесты для классов с использованием фикстур-фабрик.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Тестировать состояние объекта после вызова методов (ПК 3.2, ПК 3.3).
2. Проверять инварианты класса (ОК 02, ОК 05).
3. Применять фикстуры-фабрики для создания тестовых объектов (ПК 3.2).
4. Изолировать тесты для классов с состоянием (ПК 3.3).

---

## 1. Тестирование состояния объекта

### 1.1 Что такое состояние объекта?

**Состояние объекта** — совокупность значений всех его атрибутов (полей) в определённый момент времени.

```python
class BankAccount:
    def __init__(self, owner: str, balance: float = 0):
        self.owner = owner
        self.balance = balance
        self.is_active = True
        self.transactions = []
    
    def deposit(self, amount: float):
        self.balance += amount
        self.transactions.append(("deposit", amount))
    
    def withdraw(self, amount: float):
        if amount > self.balance:
            raise ValueError("Insufficient funds")
        self.balance -= amount
        self.transactions.append(("withdraw", amount))
    
    def close(self):
        self.is_active = False
1.2 Принципы тестирования состояния
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                ТЕСТИРОВАНИЕ СОСТОЯНИЯ ОБЪЕКТА                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. ПРОВЕРКА НАЧАЛЬНОГО СОСТОЯНИЯ                                           │
│      ├── Атрибуты установлены правильно                                      │
│      └── Инварианты класса выполняются                                       │
│                                                                              │
│   2. ПРОВЕРКА СОСТОЯНИЯ ПОСЛЕ МУТИРУЮЩИХ МЕТОДОВ                              │
│      ├── Изменились ли нужные атрибуты?                                      │
│      ├── Не изменились ли "чужие" атрибуты?                                  │
│      └── Добавились ли записи в истории?                                     │
│                                                                              │
│   3. ПРОВЕРКА СОСТОЯНИЯ ПОСЛЕ НЕМУТИРУЮЩИХ МЕТОДОВ                            │
│      ├── Состояние не должно измениться                                      │
│      └── Возвращаемое значение корректно                                     │
│                                                                              │
│   4. ПРОВЕРКА СОСТОЯНИЯ ПРИ ИСКЛЮЧЕНИЯХ                                      │
│      ├── Состояние не должно измениться при ошибке                           │
│      └── Объект должен оставаться в валидном состоянии                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
1.3 Пример: тестирование состояния BankAccount
python
import pytest
from bank_account import BankAccount


class TestBankAccountState:
    """Тестирование состояния объекта BankAccount."""
    
    def test_initial_state(self):
        """Проверка начального состояния."""
        account = BankAccount("Alice", 1000)
        
        assert account.owner == "Alice"
        assert account.balance == 1000
        assert account.is_active is True
        assert account.transactions == []
    
    def test_state_after_deposit(self):
        """Проверка состояния после пополнения."""
        account = BankAccount("Bob", 500)
        account.deposit(200)
        
        assert account.balance == 700
        assert account.is_active is True
        assert len(account.transactions) == 1
        assert account.transactions[0] == ("deposit", 200)
    
    def test_state_after_withdraw(self):
        """Проверка состояния после снятия."""
        account = BankAccount("Charlie", 1000)
        account.withdraw(300)
        
        assert account.balance == 700
        assert len(account.transactions) == 1
        assert account.transactions[0] == ("withdraw", 300)
    
    def test_state_unchanged_after_getter(self):
        """Немутирующие методы не должны менять состояние."""
        account = BankAccount("Diana", 500)
        original_balance = account.balance
        original_transactions = account.transactions.copy()
        
        # Вызов метода, который не меняет состояние
        _ = account.balance  # просто чтение
        
        assert account.balance == original_balance
        assert account.transactions == original_transactions
    
    def test_state_after_multiple_operations(self):
        """Проверка состояния после нескольких операций."""
        account = BankAccount("Eve", 1000)
        
        account.deposit(500)
        account.withdraw(200)
        account.deposit(100)
        account.withdraw(50)
        
        assert account.balance == 1000 + 500 - 200 + 100 - 50
        assert len(account.transactions) == 4
    
    def test_state_unchanged_on_error(self):
        """При исключении состояние не должно меняться."""
        account = BankAccount("Frank", 100)
        original_balance = account.balance
        original_transactions = account.transactions.copy()
        
        with pytest.raises(ValueError, match="Insufficient funds"):
            account.withdraw(200)
        
        assert account.balance == original_balance
        assert account.transactions == original_transactions
2. Проверка инвариантов класса
2.1 Что такое инвариант класса?
Инвариант класса — условие, которое должно оставаться истинным для всех объектов класса на протяжении всего их существования (кроме моментов внутри методов).

python
class Rectangle:
    """Прямоугольник с инвариантом: ширина > 0, высота > 0."""
    
    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height
        self._validate()
    
    def _validate(self):
        """Проверка инварианта."""
        if self.width <= 0 or self.height <= 0:
            raise ValueError("Width and height must be positive")
    
    @property
    def area(self) -> float:
        return self.width * self.height
    
    def resize(self, factor: float):
        """Изменение размера с сохранением инварианта."""
        self.width *= factor
        self.height *= factor
        self._validate()
    
    def set_width(self, width: float):
        self.width = width
        self._validate()
    
    def set_height(self, height: float):
        self.height = height
        self._validate()
2.2 Тестирование инвариантов
python
class TestRectangleInvariants:
    """Тестирование инвариантов класса Rectangle."""
    
    def test_invariant_on_creation(self):
        """Инвариант соблюдается при создании."""
        rect = Rectangle(10, 20)
        assert rect.width > 0
        assert rect.height > 0
    
    def test_invariant_violation_on_creation(self):
        """Нарушение инварианта при создании."""
        with pytest.raises(ValueError, match="must be positive"):
            Rectangle(-10, 20)
        
        with pytest.raises(ValueError):
            Rectangle(10, -20)
        
        with pytest.raises(ValueError):
            Rectangle(0, 20)
    
    def test_invariant_after_resize(self):
        """Инвариант соблюдается после resize."""
        rect = Rectangle(10, 20)
        rect.resize(2)
        
        assert rect.width == 20
        assert rect.height == 40
        assert rect.width > 0 and rect.height > 0
    
    def test_invariant_violation_after_setter(self):
        """Нарушение инварианта через сеттер."""
        rect = Rectangle(10, 20)
        
        with pytest.raises(ValueError, match="must be positive"):
            rect.set_width(-5)
    
    def test_invariant_preserved_across_operations(self):
        """Инвариант сохраняется после цепочки операций."""
        rect = Rectangle(10, 20)
        
        rect.resize(0.5)
        assert rect.width > 0 and rect.height > 0
        
        rect.set_width(15)
        assert rect.width > 0 and rect.height > 0
        
        rect.set_height(30)
        assert rect.width > 0 and rect.height > 0
2.3 Функция для проверки инварианта
python
def check_invariant(account: BankAccount) -> bool:
    """Проверка инвариантов BankAccount."""
    # Инвариант: баланс не может быть отрицательным
    if account.balance < 0:
        return False
    
    # Инвариант: владелец не может быть пустой строкой
    if not account.owner or not isinstance(account.owner, str):
        return False
    
    # Инвариант: transactions — список кортежей
    for trans in account.transactions:
        if not isinstance(trans, tuple) or len(trans) != 2:
            return False
    
    return True


def test_invariant_helper():
    """Использование функции проверки инварианта."""
    account = BankAccount("Alice", 1000)
    assert check_invariant(account)
    
    account.deposit(500)
    assert check_invariant(account)
    
    account.withdraw(200)
    assert check_invariant(account)
3. Изоляция тестов для классов с состоянием
3.1 Проблема: разделяемое состояние между тестами
python
# ❌ НЕПРАВИЛЬНО: тесты влияют друг на друга
class TestBadIsolation:
    account = BankAccount("Shared", 1000)  # Разделяемый объект!
    
    def test_deposit(self):
        self.account.deposit(100)
        assert self.account.balance == 1100
    
    def test_withdraw(self):
        # Ожидает баланс 1000, но предыдущий тест изменил его!
        self.account.withdraw(100)
        assert self.account.balance == 900  # Упадёт!
3.2 Решение: свежий объект для каждого теста
python
# ✅ ПРАВИЛЬНО: каждый тест создаёт свой объект
class TestGoodIsolation:
    
    def test_deposit(self):
        account = BankAccount("Alice", 1000)
        account.deposit(100)
        assert account.balance == 1100
    
    def test_withdraw(self):
        account = BankAccount("Bob", 1000)
        account.withdraw(100)
        assert account.balance == 900
3.3 Фикстуры для создания объектов (фикстуры-фабрики)
python
import pytest


@pytest.fixture
def empty_account():
    """Фикстура, создающая пустой счёт."""
    return BankAccount("TestUser", 0)


@pytest.fixture
def funded_account():
    """Фикстура, создающая счёт с деньгами."""
    return BankAccount("TestUser", 1000)


@pytest.fixture
def account_with_history():
    """Фикстура, создающая счёт с историей операций."""
    account = BankAccount("TestUser", 1000)
    account.deposit(500)
    account.withdraw(200)
    account.deposit(100)
    return account


class TestWithFixtures:
    """Тесты с использованием фикстур-фабрик."""
    
    def test_empty_account_balance(self, empty_account):
        assert empty_account.balance == 0
        assert empty_account.transactions == []
    
    def test_funded_account_balance(self, funded_account):
        assert funded_account.balance == 1000
    
    def test_account_with_history(self, account_with_history):
        assert account_with_history.balance == 1000 + 500 - 200 + 100
        assert len(account_with_history.transactions) == 3
3.4 Фикстура-фабрика с параметрами
python
@pytest.fixture
def account_factory():
    """Фикстура-фабрика для создания счетов с разными параметрами."""
    def _create_account(owner: str = "Default", balance: float = 0):
        return BankAccount(owner, balance)
    return _create_account


class TestWithFactory:
    """Тесты с использованием фабрики."""
    
    def test_create_default_account(self, account_factory):
        account = account_factory()
        assert account.owner == "Default"
        assert account.balance == 0
    
    def test_create_custom_account(self, account_factory):
        account = account_factory("Alice", 500)
        assert account.owner == "Alice"
        assert account.balance == 500
    
    def test_multiple_accounts(self, account_factory):
        alice = account_factory("Alice", 1000)
        bob = account_factory("Bob", 500)
        
        assert alice.owner == "Alice"
        assert bob.owner == "Bob"
        assert alice.balance == 1000
        assert bob.balance == 500
3.5 Фикстура с teardown для очистки
python
@pytest.fixture
def temporary_account():
    """Фикстура, которая очищает внешние ресурсы после теста."""
    account = BankAccount("Temp", 1000)
    
    yield account
    
    # Очистка после теста (например, удаление из БД)
    account.transactions.clear()
    print(f"\n[TEARDOWN] Account {account.owner} cleaned")
4. Тестирование наследования и полиморфизма
4.1 Иерархия классов
python
class Animal:
    def __init__(self, name: str):
        self.name = name
    
    def speak(self) -> str:
        raise NotImplementedError
    
    def get_info(self) -> str:
        return f"Animal: {self.name}"


class Dog(Animal):
    def speak(self) -> str:
        return "Woof!"
    
    def wag_tail(self) -> str:
        return "Tail wagging"


class Cat(Animal):
    def speak(self) -> str:
        return "Meow!"
    
    def purr(self) -> str:
        return "Purrr..."
4.2 Тестирование иерархии
python
class TestAnimal:
    """Тесты базового класса."""
    
    def test_animal_creation(self):
        animal = Animal("Generic")
        assert animal.name == "Generic"
    
    def test_animal_speak_raises(self):
        animal = Animal("Generic")
        with pytest.raises(NotImplementedError):
            animal.speak()


class TestDog:
    """Тесты класса Dog."""
    
    def test_dog_inherits_from_animal(self):
        dog = Dog("Rex")
        assert isinstance(dog, Animal)
        assert dog.name == "Rex"
    
    def test_dog_speak(self):
        dog = Dog("Rex")
        assert dog.speak() == "Woof!"
    
    def test_dog_specific_method(self):
        dog = Dog("Rex")
        assert dog.wag_tail() == "Tail wagging"
    
    def test_dog_get_info(self):
        dog = Dog("Rex")
        assert dog.get_info() == "Animal: Rex"


class TestCat:
    """Тесты класса Cat."""
    
    def test_cat_speak(self):
        cat = Cat("Whiskers")
        assert cat.speak() == "Meow!"
    
    def test_cat_purr(self):
        cat = Cat("Whiskers")
        assert cat.purr() == "Purrr..."
    
    def test_polymorphism(self):
        """Полиморфное поведение."""
        animals = [Dog("Rex"), Cat("Whiskers")]
        sounds = [animal.speak() for animal in animals]
        assert sounds == ["Woof!", "Meow!"]
5. Фикстуры-фабрики для сложных объектов
5.1 Сложный класс
python
class Order:
    def __init__(self, customer_id: int):
        self.customer_id = customer_id
        self.items = []
        self.status = "created"
        self.created_at = datetime.now()
    
    def add_item(self, product_id: int, quantity: int, price: float):
        self.items.append({
            "product_id": product_id,
            "quantity": quantity,
            "price": price
        })
    
    def get_total(self) -> float:
        return sum(item["quantity"] * item["price"] for item in self.items)
    
    def confirm(self):
        if not self.items:
            raise ValueError("Cannot confirm empty order")
        self.status = "confirmed"
    
    def cancel(self):
        self.status = "cancelled"
5.2 Фикстура-фабрика для Order
python
@pytest.fixture
def order_factory():
    """Фабрика для создания заказов с предустановленными товарами."""
    
    def _create_order(customer_id: int = 1, with_items: bool = True):
        order = Order(customer_id)
        
        if with_items:
            order.add_item(product_id=100, quantity=2, price=50.0)
            order.add_item(product_id=101, quantity=1, price=30.0)
        
        return order
    
    return _create_order


@pytest.fixture
def confirmed_order(order_factory):
    """Фикстура с подтверждённым заказом."""
    order = order_factory(with_items=True)
    order.confirm()
    return order


class TestOrderWithFactory:
    """Тесты заказа с использованием фабрик."""
    
    def test_empty_order(self, order_factory):
        order = order_factory(with_items=False)
        assert order.status == "created"
        assert order.items == []
        assert order.get_total() == 0
    
    def test_order_with_items(self, order_factory):
        order = order_factory(with_items=True)
        assert len(order.items) == 2
        assert order.get_total() == 2 * 50 + 1 * 30
    
    def test_confirm_empty_order_fails(self, order_factory):
        order = order_factory(with_items=False)
        with pytest.raises(ValueError, match="Cannot confirm empty order"):
            order.confirm()
    
    def test_confirmed_order_state(self, confirmed_order):
        assert confirmed_order.status == "confirmed"
        assert len(confirmed_order.items) == 2
    
    def test_cancel_order(self, order_factory):
        order = order_factory()
        order.cancel()
        assert order.status == "cancelled"
6. Шпаргалка
python
# === ТЕСТИРОВАНИЕ СОСТОЯНИЯ ===
def test_state(account):
    assert account.balance == 1000
    account.deposit(100)
    assert account.balance == 1100

# === ПРОВЕРКА ИНВАРИАНТОВ ===
def test_invariant(account):
    assert account.balance >= 0
    assert account.owner != ""

# === ФИКСТУРА-ОБЪЕКТ ===
@pytest.fixture
def sample_account():
    return BankAccount("Test", 1000)

def test_with_fixture(sample_account):
    assert sample_account.balance == 1000

# === ФИКСТУРА-ФАБРИКА ===
@pytest.fixture
def account_factory():
    def _create(owner="User", balance=0):
        return BankAccount(owner, balance)
    return _create

def test_factory(account_factory):
    acc = account_factory("Alice", 500)
    assert acc.owner == "Alice"

# === ФИКСТУРА С TEARDOWN ===
@pytest.fixture
def temp_account():
    acc = BankAccount("Temp", 0)
    yield acc
    acc.transactions.clear()

# === ТЕСТИРОВАНИЕ НАСЛЕДОВАНИЯ ===
def test_inheritance():
    dog = Dog("Rex")
    assert isinstance(dog, Animal)
    assert dog.speak() == "Woof!"
Контрольные вопросы
Что такое состояние объекта и почему его важно тестировать?

Какие аспекты состояния нужно проверять после вызова метода?

Что такое инвариант класса? Приведите пример.

Как проверить, что инвариант сохраняется после всех операций?

Почему тесты для классов с состоянием должны быть изолированы?

Чем фикстура-объект отличается от фикстуры-фабрики?

Как создать фикстуру с teardown для очистки ресурсов?

Как тестировать полиморфное поведение?

Что нужно проверить при тестировании наследования?

Как убедиться, что при исключении состояние объекта не испортилось?

Следующее занятие: ПР 3.11 — Практическое тестирование классов и объектов.
