# Практическое занятие 3.3: Тестирование классов и объектов (проверка состояния, инвариантов)

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.2:** Объектно-ориентированное тестирование  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться тестировать классы и объекты: проверять состояние после вызова методов, реализовывать проверку инвариантов класса, выявлять нарушения консистентности данных.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Тестировать начальное состояние объектов (ПК 3.2, ПК 3.3).
2. Проверять изменение состояния после вызова методов (ПК 3.2).
3. Реализовывать и тестировать инварианты классов (ПК 3.2, ПК 3.3).
4. Выявлять нарушения инвариантов и ошибки в логике классов (ОК 02, ОК 05).

---

## Теоретическая справка

### Что нужно тестировать в классе

| Компонент | Что проверять |
|:---|:---|
| **Конструктор** | Правильная инициализация всех атрибутов |
| **Мутирующие методы** | Корректное изменение состояния |
| **Немутирующие методы** | Отсутствие побочных эффектов |
| **Исключения** | Состояние не меняется при ошибке |
| **Инварианты** | Условия, которые всегда истинны |

---

## Объект тестирования

### Листинг 1. Модуль shopping_cart.py

```python
# shopping_cart.py

from dataclasses import dataclass
from typing import List, Dict, Optional
from datetime import datetime
from decimal import Decimal, ROUND_HALF_UP


@dataclass
class CartItem:
    """Товар в корзине."""
    product_id: int
    name: str
    price: Decimal
    quantity: int
    
    def get_total(self) -> Decimal:
        """Общая стоимость товара."""
        return self.price * self.quantity


class ShoppingCart:
    """
    Корзина покупок с проверкой инвариантов.
    
    Инварианты:
    1. Количество каждого товара > 0
    2. Цена каждого товара > 0
    3. Общая сумма корзины >= 0
    4. Сумма товаров = сумме price * quantity
    5. Скидка не может быть больше общей суммы
    """
    
    def __init__(self, user_id: int):
        self.user_id = user_id
        self._items: List[CartItem] = []
        self._total: Decimal = Decimal('0')
        self._discount: Decimal = Decimal('0')
        self.created_at = datetime.now()
        self._check_invariant()
    
    def _check_invariant(self) -> None:
        """
        Проверяет инварианты класса.
        Выбрасывает AssertionError при нарушении.
        """
        # Инвариант 1: все цены положительные
        for item in self._items:
            assert item.price > 0, f"Price must be positive: {item.price}"
            assert item.quantity > 0, f"Quantity must be positive: {item.quantity}"
        
        # Инвариант 2: общая сумма не отрицательная
        assert self._total >= 0, f"Total cannot be negative: {self._total}"
        
        # Инвариант 3: сумма соответствует сумме товаров
        calculated_total = sum(item.get_total() for item in self._items)
        assert abs(self._total - calculated_total) < Decimal('0.01'), \
            f"Total mismatch: {self._total} vs {calculated_total}"
        
        # Инвариант 4: скидка не больше суммы
        assert self._discount <= self._total, \
            f"Discount {self._discount} cannot exceed total {self._total}"
    
    def add_item(self, product_id: int, name: str, price: Decimal, quantity: int = 1) -> None:
        """
        Добавляет товар в корзину.
        
        Raises:
            ValueError: При неверных параметрах
        """
        if price <= 0:
            raise ValueError("Price must be positive")
        if quantity <= 0:
            raise ValueError("Quantity must be positive")
        
        # Проверяем, есть ли уже такой товар
        for item in self._items:
            if item.product_id == product_id:
                item.quantity += quantity
                self._recalculate_total()
                self._check_invariant()
                return
        
        # Добавляем новый товар
        new_item = CartItem(product_id, name, price, quantity)
        self._items.append(new_item)
        self._recalculate_total()
        self._check_invariant()
    
    def remove_item(self, product_id: int, quantity: int = None) -> bool:
        """
        Удаляет товар из корзины (частично или полностью).
        
        Args:
            product_id: ID товара
            quantity: Количество для удаления (None = удалить полностью)
        
        Returns:
            True если товар был удалён
        """
        for i, item in enumerate(self._items):
            if item.product_id == product_id:
                if quantity is None or quantity >= item.quantity:
                    # Удаляем полностью
                    del self._items[i]
                else:
                    # Уменьшаем количество
                    item.quantity -= quantity
                self._recalculate_total()
                self._check_invariant()
                return True
        return False
    
    def update_quantity(self, product_id: int, new_quantity: int) -> bool:
        """Обновляет количество товара."""
        if new_quantity <= 0:
            return self.remove_item(product_id)
        
        for item in self._items:
            if item.product_id == product_id:
                item.quantity = new_quantity
                self._recalculate_total()
                self._check_invariant()
                return True
        return False
    
    def apply_discount(self, discount_percent: float) -> None:
        """
        Применяет скидку в процентах.
        
        Raises:
            ValueError: При неверном проценте скидки
        """
        if not 0 <= discount_percent <= 100:
            raise ValueError("Discount must be between 0 and 100")
        
        self._discount = self._total * Decimal(str(discount_percent)) / Decimal('100')
        self._check_invariant()
    
    def apply_fixed_discount(self, amount: Decimal) -> None:
        """Применяет фиксированную скидку."""
        if amount < 0:
            raise ValueError("Discount amount cannot be negative")
        if amount > self._total:
            raise ValueError("Discount cannot exceed total")
        
        self._discount = amount
        self._check_invariant()
    
    def get_total(self) -> Decimal:
        """Возвращает итоговую сумму с учётом скидки."""
        return self._total - self._discount
    
    def get_subtotal(self) -> Decimal:
        """Возвращает сумму без скидки."""
        return self._total
    
    def get_discount(self) -> Decimal:
        """Возвращает сумму скидки."""
        return self._discount
    
    def get_items(self) -> List[CartItem]:
        """Возвращает копию списка товаров."""
        return self._items.copy()
    
    def get_item_count(self) -> int:
        """Возвращает общее количество товаров."""
        return sum(item.quantity for item in self._items)
    
    def get_unique_items_count(self) -> int:
        """Возвращает количество уникальных товаров."""
        return len(self._items)
    
    def clear(self) -> None:
        """Очищает корзину."""
        self._items.clear()
        self._total = Decimal('0')
        self._discount = Decimal('0')
        self._check_invariant()
    
    def _recalculate_total(self) -> None:
        """Пересчитывает общую сумму."""
        self._total = sum(item.get_total() for item in self._items)
    
    def is_empty(self) -> bool:
        """Проверяет, пуста ли корзина."""
        return len(self._items) == 0
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Тестирование начального состояния и простых методов
Средний (варианты 9-17)	Средние студенты	+ тестирование инвариантов, сложные методы
Сложный (варианты 18-25)	Сильные студенты	+ полный набор тестов, проверка всех инвариантов
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться тестировать начальное состояние объектов и методы, изменяющие состояние.

Задания для базового уровня
Задание 1.1. Тестирование конструктора (10 минут)
python
# test_shopping_cart_base.py

import pytest
from decimal import Decimal
from datetime import datetime
from shopping_cart import ShoppingCart, CartItem


class TestShoppingCartConstructor:
    """Тесты конструктора ShoppingCart."""
    
    def test_initial_state(self):
        """Проверка начального состояния корзины."""
        cart = ShoppingCart(user_id=123)
        
        assert cart.user_id == 123
        assert cart.get_subtotal() == Decimal('0')
        assert cart.get_total() == Decimal('0')
        assert cart.get_discount() == Decimal('0')
        assert cart.is_empty() is True
        assert cart.get_item_count() == 0
        assert cart.get_unique_items_count() == 0
        assert cart.get_items() == []
        assert isinstance(cart.created_at, datetime)
    
    def test_initial_state_with_different_user(self):
        """Разные пользователи."""
        cart1 = ShoppingCart(user_id=1)
        cart2 = ShoppingCart(user_id=2)
        
        assert cart1.user_id == 1
        assert cart2.user_id == 2
    
    def test_initial_state_empty_items(self):
        """Проверка пустого списка товаров."""
        cart = ShoppingCart(user_id=1)
        
        assert len(cart.get_items()) == 0
        assert cart.get_item_count() == 0
Задание 1.2. Тестирование add_item (15 минут)
python
class TestAddItem:
    """Тесты метода add_item."""
    
    @pytest.fixture
    def cart(self):
        return ShoppingCart(user_id=1)
    
    def test_add_single_item(self, cart):
        """Добавление одного товара."""
        cart.add_item(product_id=1, name="Laptop", price=Decimal('1000'), quantity=1)
        
        assert cart.is_empty() is False
        assert cart.get_unique_items_count() == 1
        assert cart.get_item_count() == 1
        assert cart.get_subtotal() == Decimal('1000')
        
        items = cart.get_items()
        assert items[0].product_id == 1
        assert items[0].name == "Laptop"
        assert items[0].price == Decimal('1000')
        assert items[0].quantity == 1
    
    def test_add_item_with_quantity(self, cart):
        """Добавление товара с количеством > 1."""
        cart.add_item(product_id=1, name="Mouse", price=Decimal('50'), quantity=3)
        
        assert cart.get_item_count() == 3
        assert cart.get_subtotal() == Decimal('150')
        
        items = cart.get_items()
        assert items[0].quantity == 3
    
    def test_add_multiple_different_items(self, cart):
        """Добавление разных товаров."""
        cart.add_item(1, "Laptop", Decimal('1000'), 1)
        cart.add_item(2, "Mouse", Decimal('50'), 2)
        cart.add_item(3, "Keyboard", Decimal('150'), 1)
        
        assert cart.get_unique_items_count() == 3
        assert cart.get_item_count() == 4
        assert cart.get_subtotal() == Decimal('1000 + 100 + 150')  # 1000 + 50*2 + 150
    
    def test_add_same_item_twice_should_increase_quantity(self, cart):
        """Добавление одного и того же товара увеличивает количество."""
        cart.add_item(1, "Laptop", Decimal('1000'), 1)
        cart.add_item(1, "Laptop", Decimal('1000'), 2)
        
        assert cart.get_unique_items_count() == 1
        assert cart.get_item_count() == 3
        assert cart.get_subtotal() == Decimal('3000')
    
    def test_add_item_invalid_price_raises(self, cart):
        """Добавление товара с нулевой ценой -> исключение."""
        with pytest.raises(ValueError, match="Price must be positive"):
            cart.add_item(1, "Bad", Decimal('0'), 1)
    
    def test_add_item_negative_price_raises(self, cart):
        """Добавление товара с отрицательной ценой -> исключение."""
        with pytest.raises(ValueError, match="Price must be positive"):
            cart.add_item(1, "Bad", Decimal('-100'), 1)
    
    def test_add_item_zero_quantity_raises(self, cart):
        """Добавление товара с нулевым количеством -> исключение."""
        with pytest.raises(ValueError, match="Quantity must be positive"):
            cart.add_item(1, "Bad", Decimal('100'), 0)
Задание 1.3. Тестирование состояния после remove_item и clear (10 минут)
python
class TestRemoveAndClear:
    """Тесты remove_item и clear."""
    
    @pytest.fixture
    def cart_with_items(self):
        cart = ShoppingCart(user_id=1)
        cart.add_item(1, "Laptop", Decimal('1000'), 1)
        cart.add_item(2, "Mouse", Decimal('50'), 2)
        cart.add_item(3, "Keyboard", Decimal('150'), 1)
        return cart
    
    def test_remove_item_completely(self, cart_with_items):
        """Полное удаление товара."""
        result = cart_with_items.remove_item(product_id=2)
        
        assert result is True
        assert cart_with_items.get_unique_items_count() == 2
        assert cart_with_items.get_subtotal() == Decimal('1150')  # 1000 + 150
    
    def test_remove_item_partially(self, cart_with_items):
        """Частичное удаление товара."""
        result = cart_with_items.remove_item(product_id=2, quantity=1)
        
        assert result is True
        assert cart_with_items.get_unique_items_count() == 3
        assert cart_with_items.get_item_count() == 3
        assert cart_with_items.get_subtotal() == Decimal('1000 + 50 + 150')  # 1200
    
    def test_remove_nonexistent_item(self, cart_with_items):
        """Удаление несуществующего товара."""
        result = cart_with_items.remove_item(product_id=999)
        
        assert result is False
        assert cart_with_items.get_unique_items_count() == 3
    
    def test_clear_cart(self, cart_with_items):
        """Очистка корзины."""
        cart_with_items.clear()
        
        assert cart_with_items.is_empty() is True
        assert cart_with_items.get_subtotal() == Decimal('0')
        assert cart_with_items.get_discount() == Decimal('0')
        assert cart_with_items.get_items() == []
Задание 1.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Результаты тестирования

| Метод | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| __init__ | _____ | _____ | _____ |
| add_item | _____ | _____ | _____ |
| remove_item | _____ | _____ | _____ |
| clear | _____ | _____ | _____ |

### Проверка состояния

| Состояние | Начальное | После add_item | После remove_item |
|:---|:---|:---|:---|
| is_empty | ☐ | ☐ | ☐ |
| get_item_count | ☐ | ☐ | ☐ |
| get_subtotal | ☐ | ☐ | ☐ |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться тестировать инварианты класса и проверять состояние после сложных операций.

Задания для среднего уровня
Задание 2.1. Тестирование скидок (15 минут)
python
# test_shopping_cart_intermediate.py

import pytest
from decimal import Decimal
from shopping_cart import ShoppingCart


class TestDiscounts:
    """Тесты скидок."""
    
    @pytest.fixture
    def cart_with_items(self):
        cart = ShoppingCart(user_id=1)
        cart.add_item(1, "Laptop", Decimal('1000'), 1)
        cart.add_item(2, "Mouse", Decimal('50'), 2)
        return cart
    
    def test_apply_percentage_discount(self, cart_with_items):
        """Применение процентной скидки."""
        cart_with_items.apply_discount(10)
        
        assert cart_with_items.get_discount() == Decimal('110')  # 10% от 1100
        assert cart_with_items.get_total() == Decimal('990')
        assert cart_with_items.get_subtotal() == Decimal('1100')
    
    def test_apply_fixed_discount(self, cart_with_items):
        """Применение фиксированной скидки."""
        cart_with_items.apply_fixed_discount(Decimal('100'))
        
        assert cart_with_items.get_discount() == Decimal('100')
        assert cart_with_items.get_total() == Decimal('1000')
    
    def test_discount_cannot_exceed_total(self, cart_with_items):
        """Скидка не может превышать общую сумму."""
        with pytest.raises(ValueError, match="Discount cannot exceed total"):
            cart_with_items.apply_fixed_discount(Decimal('2000'))
    
    def test_discount_negative_raises(self, cart_with_items):
        """Отрицательная скидка -> исключение."""
        with pytest.raises(ValueError, match="cannot be negative"):
            cart_with_items.apply_fixed_discount(Decimal('-50'))
    
    def test_invalid_percentage_raises(self, cart_with_items):
        """Неверный процент скидки."""
        with pytest.raises(ValueError, match="Discount must be between 0 and 100"):
            cart_with_items.apply_discount(150)
Задание 2.2. Тестирование состояния после операций (15 минут)
python
class TestStateAfterOperations:
    """Тесты состояния после различных операций."""
    
    @pytest.fixture
    def cart(self):
        return ShoppingCart(user_id=1)
    
    def test_state_after_add_and_remove(self, cart):
        """Состояние после добавления и удаления."""
        cart.add_item(1, "Item", Decimal('100'), 1)
        assert cart.get_item_count() == 1
        
        cart.remove_item(1)
        assert cart.is_empty() is True
        assert cart.get_subtotal() == Decimal('0')
    
    def test_state_after_quantity_update(self, cart):
        """Состояние после обновления количества."""
        cart.add_item(1, "Item", Decimal('100'), 2)
        assert cart.get_subtotal() == Decimal('200')
        
        cart.update_quantity(1, 5)
        assert cart.get_item_count() == 5
        assert cart.get_subtotal() == Decimal('500')
    
    def test_update_quantity_to_zero_removes_item(self, cart):
        """Обновление количества до 0 удаляет товар."""
        cart.add_item(1, "Item", Decimal('100'), 3)
        cart.update_quantity(1, 0)
        
        assert cart.is_empty() is True
    
    def test_multiple_discounts_should_not_break_state(self, cart):
        """Несколько скидок подряд."""
        cart.add_item(1, "Item", Decimal('1000'), 1)
        
        cart.apply_discount(10)
        assert cart.get_discount() == Decimal('100')
        
        # Применяем другую скидку (заменяет предыдущую)
        cart.apply_fixed_discount(Decimal('50'))
        assert cart.get_discount() == Decimal('50')
        assert cart.get_total() == Decimal('950')
    
    def test_state_after_clear_with_discount(self, cart):
        """Очистка корзины со скидкой."""
        cart.add_item(1, "Item", Decimal('100'), 1)
        cart.apply_fixed_discount(Decimal('10'))
        
        cart.clear()
        
        assert cart.get_subtotal() == Decimal('0')
        assert cart.get_discount() == Decimal('0')
        assert cart.get_total() == Decimal('0')
Задание 2.3. Тестирование инвариантов (10 минут)
python
class TestInvariants:
    """Тесты инвариантов класса."""
    
    @pytest.fixture
    def cart(self):
        return ShoppingCart(user_id=1)
    
    def test_invariant_prices_positive(self, cart):
        """Инвариант: цены всегда положительные."""
        cart.add_item(1, "Item", Decimal('100'), 1)
        
        # Прямая проверка инварианта (метод _check_invariant вызывается внутри)
        # При нарушении будет AssertionError
        assert cart.get_items()[0].price > 0
    
    def test_invariant_total_matches_items(self, cart):
        """Инвариант: общая сумма равна сумме товаров."""
        cart.add_item(1, "A", Decimal('100'), 2)
        cart.add_item(2, "B", Decimal('50'), 3)
        
        expected_total = Decimal('100*2 + 50*3')  # 200 + 150 = 350
        assert cart.get_subtotal() == expected_total
    
    def test_invariant_discount_not_exceed_total(self, cart):
        """Инвариант: скидка не может превышать общую сумму."""
        cart.add_item(1, "Item", Decimal('100'), 1)
        
        with pytest.raises(ValueError, match="cannot exceed total"):
            cart.apply_fixed_discount(Decimal('200'))
    
    def test_invariant_maintained_after_operations(self, cart):
        """Инвариант сохраняется после всех операций."""
        cart.add_item(1, "A", Decimal('100'), 2)
        cart.add_item(2, "B", Decimal('50'), 3)
        cart.apply_discount(10)
        cart.remove_item(1, 1)
        cart.update_quantity(2, 5)
        
        # После всех операций инварианты должны быть соблюдены
        # _check_invariant вызывается в каждом методе
        assert cart.get_subtotal() > 0
Задание 2.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Результаты тестирования

| Метод | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| apply_discount | _____ | _____ | _____ |
| apply_fixed_discount | _____ | _____ | _____ |
| update_quantity | _____ | _____ | _____ |
| Инварианты | _____ | _____ | _____ |

### Проверенные инварианты

| Инвариант | Проверен |
|:---|:---|
| Цены положительные | ☐ |
| Количество > 0 | ☐ |
| Сумма = сумме товаров | ☐ |
| Скидка ≤ суммы | ☐ |

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться тестировать комплексные сценарии, проверять все инварианты и создавать полный тестовый набор.

Задания для сложного уровня
Задание 3.1. Комплексные сценарии (15 минут)
python
# test_shopping_cart_advanced.py

import pytest
from decimal import Decimal
from shopping_cart import ShoppingCart


class TestComplexScenarios:
    """Комплексные сценарии использования."""
    
    @pytest.fixture
    def cart(self):
        return ShoppingCart(user_id=1)
    
    def test_full_shopping_workflow(self, cart):
        """Полный сценарий покупки."""
        # 1. Добавление товаров
        cart.add_item(1, "Laptop", Decimal('1000'), 1)
        cart.add_item(2, "Mouse", Decimal('50'), 2)
        cart.add_item(3, "Keyboard", Decimal('150'), 1)
        
        assert cart.get_subtotal() == Decimal('1250')
        assert cart.get_item_count() == 4
        
        # 2. Частичное удаление
        cart.remove_item(2, 1)  # Убираем одну мышь
        assert cart.get_subtotal() == Decimal('1200')
        assert cart.get_item_count() == 3
        
        # 3. Применение скидки
        cart.apply_discount(10)  # 10% скидка
        assert cart.get_discount() == Decimal('120')
        assert cart.get_total() == Decimal('1080')
        
        # 4. Добавление ещё товара
        cart.add_item(4, "Monitor", Decimal('300'), 1)
        assert cart.get_subtotal() == Decimal('1500')
        assert cart.get_discount() == Decimal('120')  # Скидка не изменилась
        assert cart.get_total() == Decimal('1380')
        
        # 5. Очистка
        cart.clear()
        assert cart.is_empty() is True
        assert cart.get_total() == Decimal('0')
    
    def test_edge_case_multiple_discounts(self, cart):
        """Сценарий с несколькими скидками."""
        cart.add_item(1, "Item", Decimal('1000'), 1)
        
        cart.apply_discount(20)
        assert cart.get_discount() == Decimal('200')
        
        # Повторное применение скидки (заменяет)
        cart.apply_fixed_discount(Decimal('150'))
        assert cart.get_discount() == Decimal('150')
        assert cart.get_total() == Decimal('850')
        
        # Ещё одна процентная скидка
        cart.apply_discount(50)
        assert cart.get_discount() == Decimal('500')
        assert cart.get_total() == Decimal('500')
    
    def test_edge_case_quantity_updates(self, cart):
        """Сценарий с обновлением количества."""
        cart.add_item(1, "Item", Decimal('100'), 5)
        assert cart.get_subtotal() == Decimal('500')
        
        # Увеличение количества
        cart.update_quantity(1, 10)
        assert cart.get_subtotal() == Decimal('1000')
        
        # Добавление того же товара (должно объединиться)
        cart.add_item(1, "Item", Decimal('100'), 5)
        assert cart.get_unique_items_count() == 1
        assert cart.get_item_count() == 15
        assert cart.get_subtotal() == Decimal('1500')
Задание 3.2. Полная проверка инвариантов (15 минут)
python
class TestInvariantsComplete:
    """Полная проверка всех инвариантов."""
    
    def test_invariant_price_positive_after_add(self):
        """Инвариант: цена положительна после добавления."""
        cart = ShoppingCart(user_id=1)
        cart.add_item(1, "Item", Decimal('100'), 1)
        
        for item in cart.get_items():
            assert item.price > 0
    
    def test_invariant_quantity_positive_after_operations(self):
        """Инвариант: количество положительно после всех операций."""
        cart = ShoppingCart(user_id=1)
        cart.add_item(1, "Item", Decimal('100'), 3)
        cart.remove_item(1, 2)  # Остаётся 1
        
        for item in cart.get_items():
            assert item.quantity > 0
    
    def test_invariant_total_equals_sum_of_items_after_each_operation(self):
        """Инвариант: сумма = сумме товаров после каждой операции."""
        cart = ShoppingCart(user_id=1)
        
        # Проверка после добавления
        cart.add_item(1, "A", Decimal('100'), 2)
        assert cart.get_subtotal() == Decimal('200')
        
        # Проверка после добавления второго
        cart.add_item(2, "B", Decimal('50'), 3)
        assert cart.get_subtotal() == Decimal('350')
        
        # Проверка после удаления
        cart.remove_item(1, 1)
        assert cart.get_subtotal() == Decimal('250')
        
        # Проверка после скидки
        cart.apply_fixed_discount(Decimal('50'))
        assert cart.get_subtotal() == Decimal('250')
        assert cart.get_total() == Decimal('200')
    
    def test_invariant_discount_not_exceed_total_after_operations(self):
        """Инвариант: скидка не превышает сумму."""
        cart = ShoppingCart(user_id=1)
        cart.add_item(1, "Item", Decimal('100'), 1)
        cart.apply_fixed_discount(Decimal('50'))
        
        assert cart.get_discount() <= cart.get_subtotal()
        
        # Уменьшаем сумму
        cart.remove_item(1)
        assert cart.get_discount() <= cart.get_subtotal()
    
    def test_invariant_checks_raise_on_violation(self):
        """При нарушении инварианта метод _check_invariant выбрасывает ошибку."""
        cart = ShoppingCart(user_id=1)
        cart.add_item(1, "Item", Decimal('100'), 1)
        
        # Нарушаем инвариант вручную
        cart._items[0].quantity = -1
        
        with pytest.raises(AssertionError, match="Quantity must be positive"):
            cart.add_item(2, "Another", Decimal('50'), 1)  # Любая операция вызовет проверку
Задание 3.3. Тестирование граничных случаев (10 минут)
python
class TestEdgeCases:
    """Тесты граничных случаев."""
    
    def test_empty_cart_operations(self):
        """Операции с пустой корзиной."""
        cart = ShoppingCart(user_id=1)
        
        assert cart.is_empty() is True
        assert cart.get_total() == Decimal('0')
        assert cart.remove_item(1) is False
        assert cart.update_quantity(1, 5) is False
    
    def test_large_quantities(self):
        """Большие количества товаров."""
        cart = ShoppingCart(user_id=1)
        cart.add_item(1, "Item", Decimal('0.01'), 10000)
        
        assert cart.get_item_count() == 10000
        assert cart.get_subtotal() == Decimal('100')
    
    def test_precision_decimal(self):
        """Проверка точности Decimal."""
        cart = ShoppingCart(user_id=1)
        cart.add_item(1, "Item", Decimal('0.10'), 3)  # 0.30
        cart.add_item(2, "Item2", Decimal('0.20'), 3)  # 0.60
        
        assert cart.get_subtotal() == Decimal('0.90')
    
    def test_negative_scenarios(self):
        """Негативные сценарии."""
        cart = ShoppingCart(user_id=1)
        
        with pytest.raises(ValueError):
            cart.add_item(1, "Bad", Decimal('-100'), 1)
        
        with pytest.raises(ValueError):
            cart.add_item(1, "Bad", Decimal('100'), -5)
        
        with pytest.raises(ValueError):
            cart.apply_discount(-10)
        
        with pytest.raises(ValueError):
            cart.apply_discount(101)
Задание 3.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (сложный уровень)

**Студент:** _________________
**Вариант №:** ___ (18-25)

### Результаты тестирования

| Категория | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| Конструктор/начальное состояние | _____ | _____ | _____ |
| add_item | _____ | _____ | _____ |
| remove_item | _____ | _____ | _____ |
| update_quantity | _____ | _____ | _____ |
| Скидки | _____ | _____ | _____ |
| Инварианты | _____ | _____ | _____ |
| Комплексные сценарии | _____ | _____ | _____ |
| Граничные случаи | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Проверенные инварианты

| Инвариант | Проверен |
|:---|:---|
| price > 0 | ☐ |
| quantity > 0 | ☐ |
| total >= 0 | ☐ |
| total = sum(price * quantity) | ☐ |
| discount <= total | ☐ |

### Выводы

_______________________________________________________________

_______________________________________________________________
Карточка студента
text
ПР 3.3. ТЕСТИРОВАНИЕ КЛАССОВ И ОБЪЕКТОВ (СОСТОЯНИЕ, ИНВАРИАНТЫ)

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ Конструктор (базовый)
□ add_item (базовый)
□ remove_item / clear (базовый)
□ Скидки (средний)
□ Состояние после операций (средний)
□ Инварианты (средний)
□ Комплексные сценарии (сложный)
□ Граничные случаи (сложный)

=== ПРОВЕРЕННЫЕ АСПЕКТЫ ===

□ Начальное состояние
□ Состояние после мутирующих методов
□ Состояние после исключений
□ Инварианты класса
□ Граничные значения

=== РЕЗУЛЬТАТЫ ===

Всего тестов: _____
Пройдено: _____
Не пройдено: _____

=== ВРЕМЯ ВЫПОЛНЕНИЯ ===

Подготовка: _____ мин
Выполнение: _____ мин
Оформление: _____ мин

=== ОТЧЁТ ===

Файлы:
- test_shopping_cart_base.py
- test_shopping_cart_intermediate.py
- test_shopping_cart_advanced.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Тесты не работают или состояние не проверяется
3 (удовлетворительно)	Базовый	Конструктор + add_item + remove_item (10+ тестов)
4 (хорошо)	Средний	+ Скидки + инварианты (20+ тестов)
5 (отлично)	Сложный	+ Комплексные сценарии + граничные случаи (30+ тестов)
Контрольные вопросы (для защиты)
Какие аспекты состояния нужно проверять в тестах?

Почему важно проверять состояние после исключений?

Что такое инвариант класса? Как его тестировать?

Как проверить, что метод не меняет состояние, когда не должен?

Какие методы называются мутирующими и немутирующими?

Как протестировать, что инвариант сохраняется после цепочки операций?

Почему get_items() должен возвращать копию, а не оригинал?

Как проверить, что скидка правильно применяется?

Какие граничные случаи важны для тестирования корзины?

Как убедиться, что инварианты проверяются в каждом методе?

Следующее занятие: ПР 3.4 — Тестирование иерархий классов и полиморфизма.

text

