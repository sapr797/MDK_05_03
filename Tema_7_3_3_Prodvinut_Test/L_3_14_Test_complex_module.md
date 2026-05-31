# Лекция 3.14: Тестирование комплексных модулей. Стратегии

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить стратегии тестирования комплексных модулей, содержащих сложную бизнес-логику, множество зависимостей и состояний. Освоить методы декомпозиции, изоляции и интеграции при тестировании сложных компонентов.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Анализировать комплексные модули и выделять компоненты для тестирования (ПК 3.1).
2. Применять стратегии тестирования для сложной логики (ПК 3.2).
3. Использовать мокирование и стабы для изоляции зависимостей (ПК 3.2).
4. Проектировать тесты для комплексных сценариев использования (ПК 3.3).

---

## 1. Что такое комплексный модуль?

### 1.1 Признаки комплексного модуля
┌─────────────────────────────────────────────────────────────────────────────┐
│ ПРИЗНАКИ КОМПЛЕКСНОГО МОДУЛЯ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ 1. МНОГО ЗАВИСИМОСТЕЙ │
│ └── База данных, внешние API, кэш, очередь сообщений │
│ │
│ 2. РАЗВЕТВЛЁННАЯ ЛОГИКА │
│ └── Множество условных операторов, циклов, обработка ошибок │
│ │
│ 3. ИЗМЕНЕНИЕ СОСТОЯНИЯ │
│ └── Много атрибутов, сложные переходы между состояниями │
│ │
│ 4. БИЗНЕС-ПРАВИЛА │
│ └── Сложные правила валидации, расчёты, комиссии │
│ │
│ 5. ТРЕБОВАНИЯ К ПРОИЗВОДИТЕЛЬНОСТИ │
│ └── Кэширование, асинхронность, оптимизации │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 1.2 Пример комплексного модуля

```python
class OrderProcessor:
    """
    Комплексный модуль обработки заказов.
    
    Зависимости:
    - PaymentGateway (внешний API)
    - InventoryService (проверка наличия)
    - EmailService (уведомления)
    - Database (сохранение)
    - Cache (скидки и акции)
    """
    
    def __init__(
        self,
        payment_gateway: PaymentGateway,
        inventory: InventoryService,
        email_service: EmailService,
        db: Database,
        cache: Cache
    ):
        self.payment_gateway = payment_gateway
        self.inventory = inventory
        self.email_service = email_service
        self.db = db
        self.cache = cache
    
    def process_order(self, order: Order) -> ProcessResult:
        """
        Обработка заказа со сложной бизнес-логикой.
        """
        # 1. Валидация заказа
        if not order.items:
            return ProcessResult.error("Empty order")
        
        # 2. Проверка наличия товаров
        for item in order.items:
            if not self.inventory.check_availability(item.product_id, item.quantity):
                return ProcessResult.error(f"Product {item.product_id} out of stock")
        
        # 3. Расчёт скидки (сложная логика)
        discount = self._calculate_discount(order)
        
        # 4. Обработка платежа
        payment_result = self.payment_gateway.charge(order.total - discount)
        if not payment_result.success:
            return ProcessResult.error(f"Payment failed: {payment_result.error}")
        
        # 5. Резервирование товаров
        for item in order.items:
            self.inventory.reserve(item.product_id, item.quantity)
        
        # 6. Сохранение в БД
        order_id = self.db.save_order(order, discount, payment_result.transaction_id)
        
        # 7. Отправка уведомлений
        self.email_service.send_order_confirmation(order.customer_email, order_id)
        
        return ProcessResult.success(order_id)
2. Стратегии тестирования комплексных модулей
2.1 Стратегия декомпозиции
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    СТРАТЕГИЯ ДЕКОМПОЗИЦИИ                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Комплексный модуль                                                         │
│         │                                                                    │
│         ├── Шаг 1: Валидация                                                 │
│         ├── Шаг 2: Проверка наличия                                         │
│         ├── Шаг 3: Расчёт скидки                                            │
│         ├── Шаг 4: Платёж                                                   │
│         ├── Шаг 5: Резервирование                                           │
│         └── Шаг 6: Сохранение и уведомления                                 │
│                                                                              │
│   КАЖДЫЙ ШАГ ТЕСТИРУЕТСЯ ОТДЕЛЬНО                                            │
│                                                                              │
│   1. Модульные тесты для каждого шага                                       │
│   2. Интеграционные тесты для пар шагов                                     │
│   3. Сквозные тесты для всего модуля                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
2.2 Реализация стратегии декомпозиции
python
# 1. МОДУЛЬНЫЕ ТЕСТЫ ДЛЯ ОТДЕЛЬНЫХ ШАГОВ
class TestOrderProcessorUnit:
    """Модульные тесты (каждый шаг изолирован)."""
    
    def test_validation_empty_order(self):
        processor = OrderProcessor(mock_payment, mock_inventory, mock_email, mock_db, mock_cache)
        result = processor.process_order(Order(items=[]))
        assert result.is_error()
        assert "Empty order" in result.message
    
    def test_discount_calculation(self):
        # Тестируем только расчёт скидки (изолированно)
        processor = OrderProcessor(...)
        discount = processor._calculate_discount(test_order)
        assert discount == expected

# 2. ИНТЕГРАЦИОННЫЕ ТЕСТЫ ДЛЯ ПАР ШАГОВ
class TestOrderProcessorIntegration:
    """Интеграционные тесты (несколько шагов вместе)."""
    
    def test_validation_and_inventory_check(self):
        # Тестируем валидацию + проверку наличия
        processor = OrderProcessor(mock_payment, real_inventory, mock_email, mock_db, mock_cache)
        result = processor.process_order(order_with_out_of_stock_item)
        assert result.is_error()
        assert "out of stock" in result.message

# 3. СКВОЗНЫЕ ТЕСТЫ
class TestOrderProcessorE2E:
    """Сквозные тесты (весь модуль)."""
    
    def test_full_order_processing(self):
        processor = OrderProcessor(real_payment, real_inventory, real_email, real_db, real_cache)
        result = processor.process_order(valid_order)
        assert result.is_success()
        assert result.order_id is not None
3. Использование Test Doubles для изоляции
3.1 Типы Test Doubles для комплексных модулей
Тип	Назначение	Пример использования
Dummy	Заполнение параметров	DummyPaymentGateway()
Stub	Возврат предопределённых данных	InventoryStub.always_available()
Spy	Проверка вызовов	EmailSpy.get_sent_messages()
Mock	Проверка поведения	mock_payment.charge.assert_called_once()
Fake	Полноценная, но упрощённая реализация	FakeDatabase()
3.2 Пример: мокирование зависимостей
python
class TestOrderProcessorWithMocks:
    """Тестирование с моками всех зависимостей."""
    
    @pytest.fixture
    def mock_payment(self, mocker):
        mock = mocker.Mock()
        mock.charge.return_value = PaymentResult.success("tx_123")
        return mock
    
    @pytest.fixture
    def mock_inventory(self, mocker):
        mock = mocker.Mock()
        mock.check_availability.return_value = True
        return mock
    
    @pytest.fixture
    def mock_email(self, mocker):
        return mocker.Mock()
    
    @pytest.fixture
    def mock_db(self, mocker):
        mock = mocker.Mock()
        mock.save_order.return_value = 12345
        return mock
    
    @pytest.fixture
    def mock_cache(self, mocker):
        return mocker.Mock()
    
    @pytest.fixture
    def processor(self, mock_payment, mock_inventory, mock_email, mock_db, mock_cache):
        return OrderProcessor(mock_payment, mock_inventory, mock_email, mock_db, mock_cache)
    
    def test_successful_order_flow(self, processor, mock_payment, mock_inventory, mock_email, mock_db):
        order = create_valid_order()
        
        result = processor.process_order(order)
        
        assert result.is_success()
        assert result.order_id == 12345
        
        # Проверка вызовов всех зависимостей
        mock_inventory.check_availability.assert_called()
        mock_payment.charge.assert_called_once()
        mock_db.save_order.assert_called_once()
        mock_email.send_order_confirmation.assert_called_once()
    
    def test_payment_failure(self, processor, mock_payment):
        mock_payment.charge.return_value = PaymentResult.failure("Insufficient funds")
        
        order = create_valid_order()
        result = processor.process_order(order)
        
        assert result.is_error()
        assert "Payment failed" in result.message
        assert "Insufficient funds" in result.message
4. Стратегия тестирования состояний
4.1 Тестирование конечного автомата
python
class OrderStateMachine:
    """Конечный автомат для заказа."""
    
    STATES = ["CREATED", "CONFIRMED", "PAID", "SHIPPED", "DELIVERED", "CANCELLED"]
    
    def __init__(self):
        self.state = "CREATED"
    
    def confirm(self):
        if self.state != "CREATED":
            raise InvalidStateError(f"Cannot confirm from {self.state}")
        self.state = "CONFIRMED"
    
    def pay(self):
        if self.state not in ["CONFIRMED", "PAID"]:
            raise InvalidStateError(f"Cannot pay from {self.state}")
        self.state = "PAID"
    
    def ship(self):
        if self.state != "PAID":
            raise InvalidStateError(f"Cannot ship from {self.state}")
        self.state = "SHIPPED"
    
    def deliver(self):
        if self.state != "SHIPPED":
            raise InvalidStateError(f"Cannot deliver from {self.state}")
        self.state = "DELIVERED"
    
    def cancel(self):
        if self.state in ["DELIVERED", "SHIPPED"]:
            raise InvalidStateError(f"Cannot cancel from {self.state}")
        self.state = "CANCELLED"


class TestOrderStateMachine:
    """Тестирование переходов между состояниями."""
    
    @pytest.fixture
    def machine(self):
        return OrderStateMachine()
    
    @pytest.mark.parametrize("actions,expected_state", [
        (["confirm", "pay", "ship", "deliver"], "DELIVERED"),
        (["confirm", "pay", "ship"], "SHIPPED"),
        (["confirm", "pay"], "PAID"),
        (["confirm"], "CONFIRMED"),
        (["cancel"], "CANCELLED"),
    ])
    def test_happy_paths(self, machine, actions, expected_state):
        for action in actions:
            getattr(machine, action)()
        assert machine.state == expected_state
    
    @pytest.mark.parametrize("invalid_transition", [
        ("pay", "CREATED"),
        ("ship", "CONFIRMED"),
        ("deliver", "PAID"),
        ("confirm", "CANCELLED"),
    ])
    def test_invalid_transitions(self, machine, invalid_transition):
        action, start_state = invalid_transition
        
        # Устанавливаем начальное состояние
        machine.state = start_state
        
        with pytest.raises(InvalidStateError):
            getattr(machine, action)()
5. Стратегия тестирования бизнес-правил
5.1 Таблица решений для сложной логики
python
class PricingCalculator:
    """Расчёт цены со сложными правилами."""
    
    def calculate_price(
        self,
        base_price: float,
        quantity: int,
        is_vip: bool,
        promo_code: str = None,
        is_holiday: bool = False
    ) -> float:
        """Сложный расчёт цены с множеством правил."""
        price = base_price * quantity
        
        # Правило 1: скидка за количество
        if quantity >= 10:
            price *= 0.85  # 15% скидка
        elif quantity >= 5:
            price *= 0.90  # 10% скидка
        
        # Правило 2: VIP скидка
        if is_vip:
            price *= 0.95  # 5% скидка
        
        # Правило 3: промокод
        if promo_code == "SAVE20":
            price *= 0.80
        elif promo_code == "SAVE10":
            price *= 0.90
        
        # Правило 4: праздничная скидка
        if is_holiday:
            price *= 0.95
        
        return round(price, 2)


class TestPricingCalculator:
    """Тестирование бизнес-правил с помощью таблицы решений."""
    
    @pytest.fixture
    def calculator(self):
        return PricingCalculator()
    
    @pytest.mark.parametrize("base_price,quantity,is_vip,promo_code,is_holiday,expected", [
        # Базовые случаи
        (100, 1, False, None, False, 100.00),
        (100, 1, True, None, False, 95.00),   # VIP 5%
        
        # Скидка за количество
        (100, 5, False, None, False, 450.00),  # 10% = 500 * 0.9
        (100, 10, False, None, False, 850.00), # 15% = 1000 * 0.85
        
        # Комбинации
        (100, 10, True, None, False, 807.50),  # 15% + 5% = 1000 * 0.85 * 0.95
        (100, 10, True, "SAVE20", False, 646.00), # 15% + 5% + 20%
        (100, 1, True, "SAVE20", True, 76.00),  # 5% + 20% + 5%
    ])
    def test_price_calculation(self, calculator, base_price, quantity, is_vip, promo_code, is_holiday, expected):
        result = calculator.calculate_price(base_price, quantity, is_vip, promo_code, is_holiday)
        assert result == expected
6. Шпаргалка
python
# === СТРАТЕГИЯ ДЕКОМПОЗИЦИИ ===
# 1. Модульные тесты для каждого шага
def test_step_validation():
    pass

def test_step_inventory():
    pass

# 2. Интеграционные тесты для пар шагов
def test_validation_and_inventory():
    pass

# 3. Сквозные тесты
def test_full_flow():
    pass

# === ИЗОЛЯЦИЯ ЗАВИСИМОСТЕЙ ===
def test_with_mocks(mocker):
    mock_dependency = mocker.Mock()
    mock_dependency.method.return_value = "mocked"
    service = Service(mock_dependency)
    service.do_something()
    mock_dependency.method.assert_called_once()

# === ТЕСТИРОВАНИЕ СОСТОЯНИЙ ===
def test_state_transition():
    obj = StateMachine()
    obj.do_action()
    assert obj.state == "NEW_STATE"

# === ТАБЛИЦА РЕШЕНИЙ ===
@pytest.mark.parametrize("input,expected", [
    (case1, expected1),
    (case2, expected2),
])
def test_business_rule(input, expected):
    assert calculate(input) == expected
Контрольные вопросы
Какие признаки указывают на то, что модуль является комплексным?

В чём суть стратегии декомпозиции при тестировании?

Какие типы Test Doubles используются для изоляции зависимостей?

Как организовать тестирование конечного автомата?

Как таблица решений помогает в тестировании бизнес-правил?

Какие уровни тестирования нужны для комплексного модуля?

Как проверить, что все зависимости были вызваны корректно?

Почему важно тестировать не только успешные, но и ошибочные сценарии?

Как мокирование помогает при тестировании комплексных модулей?

Как обеспечить полное покрытие всех ветвлений сложной логики?

Следующее занятие: ПР 3.14 — Практическое тестирование комплексного модуля.

text

