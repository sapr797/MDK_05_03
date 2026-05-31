# Практическое занятие 3.11: Написание модульных и интеграционных тестов для комплексного модуля

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться писать модульные и интеграционные тесты для комплексных модулей, комбинировать различные техники тестирования, обеспечивать высокое покрытие кода.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Разрабатывать модульные тесты для отдельных компонентов (ПК 3.2).
2. Разрабатывать интеграционные тесты для взаимодействующих компонентов (ПК 3.2, ПК 3.3).
3. Комбинировать мокирование и реальные зависимости в тестах (ПК 3.3).
4. Обеспечивать высокое покрытие кода для комплексного модуля (ОК 02, ОК 05).

---

## Теоретическая справка

### Модульное vs Интеграционное тестирование
┌─────────────────────────────────────────────────────────────────────────────┐
│ МОДУЛЬНОЕ vs ИНТЕГРАЦИОННОЕ ТЕСТИРОВАНИЕ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ МОДУЛЬНОЕ ТЕСТИРОВАНИЕ ИНТЕГРАЦИОННОЕ ТЕСТИРОВАНИЕ │
│ ┌─────────────────────────┐ ┌─────────────────────────┐ │
│ │ Тестируется один модуль │ │ Тестируются несколько │ │
│ │ в изоляции │ │ модулей вместе │ │
│ └─────────────────────────┘ └─────────────────────────┘ │
│ │
│ • Мокируются все зависимости • Некоторые зависимости реальные │
│ • Быстрые (1000+/сек) • Медленнее (10-100/сек) │
│ • Не требуют БД/API • Могут требовать БД/API │
│ • Легко локализовать ошибки • Сложнее локализовать ошибки │
│ • Разработчик пишет • QA + разработчик │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

---

## Объект тестирования

### Листинг 1. Комплексный модуль order_processor.py

```python
# order_processor.py

import json
import logging
import uuid
from dataclasses import dataclass, asdict
from datetime import datetime
from typing import List, Dict, Any, Optional
from enum import Enum


# ============================================================
# МОДЕЛИ ДАННЫХ
# ============================================================

class OrderStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"


@dataclass
class OrderItem:
    product_id: int
    name: str
    price: float
    quantity: int
    
    def get_total(self) -> float:
        return self.price * self.quantity


@dataclass
class Order:
    id: str
    customer_id: int
    items: List[OrderItem]
    status: OrderStatus
    total: float
    created_at: datetime
    updated_at: datetime


# ============================================================
# ИСКЛЮЧЕНИЯ
# ============================================================

class OrderProcessorError(Exception):
    pass


class InventoryError(OrderProcessorError):
    pass


class PaymentError(OrderProcessorError):
    pass


class NotificationError(OrderProcessorError):
    pass


# ============================================================
# КОМПОНЕНТ 1: ИНВЕНТАРИЗАЦИЯ
# ============================================================

class InventoryService:
    """Сервис проверки наличия товаров."""
    
    def __init__(self, db_client=None):
        self.db_client = db_client
        self._cache = {}
    
    def check_availability(self, product_id: int, quantity: int) -> bool:
        """
        Проверяет наличие товара.
        
        Returns:
            True если товар есть в нужном количестве
        """
        if product_id in self._cache:
            available = self._cache[product_id]
        else:
            available = self._get_available_quantity(product_id)
            self._cache[product_id] = available
        
        return available >= quantity
    
    def reserve(self, product_id: int, quantity: int) -> bool:
        """Резервирует товар."""
        if not self.check_availability(product_id, quantity):
            raise InventoryError(f"Insufficient stock for product {product_id}")
        
        # В реальном коде здесь было бы обновление БД
        self._cache[product_id] = self._cache.get(product_id, 0) - quantity
        return True
    
    def _get_available_quantity(self, product_id: int) -> int:
        """Получает доступное количество из БД."""
        if self.db_client:
            result = self.db_client.query(
                f"SELECT quantity FROM products WHERE id = {product_id}"
            )
            return result[0][0] if result else 0
        return 100  # Значение по умолчанию для тестов
    
    def clear_cache(self):
        self._cache.clear()


# ============================================================
# КОМПОНЕНТ 2: ПЛАТЁЖНЫЙ ШЛЮЗ
# ============================================================

class PaymentGateway:
    """Платёжный шлюз для обработки платежей."""
    
    def __init__(self, api_key: str = None):
        self.api_key = api_key
        self._transactions = []
    
    def process_payment(self, amount: float, customer_id: int) -> Dict:
        """
        Обрабатывает платёж.
        
        Returns:
            Словарь с информацией о транзакции
        """
        if amount <= 0:
            raise PaymentError("Amount must be positive")
        
        # Эмуляция вызова внешнего API
        transaction_id = str(uuid.uuid4())
        transaction = {
            "id": transaction_id,
            "amount": amount,
            "customer_id": customer_id,
            "status": "completed",
            "timestamp": datetime.now().isoformat()
        }
        self._transactions.append(transaction)
        
        return transaction
    
    def refund(self, transaction_id: str) -> bool:
        """Возвращает платёж."""
        for transaction in self._transactions:
            if transaction["id"] == transaction_id:
                transaction["status"] = "refunded"
                return True
        return False
    
    def get_transaction(self, transaction_id: str) -> Optional[Dict]:
        for t in self._transactions:
            if t["id"] == transaction_id:
                return t
        return None


# ============================================================
# КОМПОНЕНТ 3: НОТИФИКАЦИЯ
# ============================================================

class NotificationService:
    """Сервис отправки уведомлений."""
    
    def __init__(self, smtp_client=None):
        self.smtp_client = smtp_client
        self._sent_emails = []
    
    def send_order_confirmation(self, email: str, order_id: str) -> bool:
        """Отправляет подтверждение заказа."""
        subject = f"Order Confirmation #{order_id}"
        body = f"Your order {order_id} has been confirmed."
        
        return self._send_email(email, subject, body)
    
    def send_payment_confirmation(self, email: str, order_id: str, amount: float) -> bool:
        """Отправляет подтверждение оплаты."""
        subject = f"Payment Confirmation #{order_id}"
        body = f"Payment of ${amount} for order {order_id} has been processed."
        
        return self._send_email(email, subject, body)
    
    def _send_email(self, to: str, subject: str, body: str) -> bool:
        """Отправляет email."""
        if not to or "@" not in to:
            raise NotificationError(f"Invalid email address: {to}")
        
        email_record = {
            "to": to,
            "subject": subject,
            "body": body,
            "sent_at": datetime.now().isoformat()
        }
        self._sent_emails.append(email_record)
        
        if self.smtp_client:
            try:
                self.smtp_client.send(to, subject, body)
            except Exception as e:
                raise NotificationError(f"Failed to send email: {e}")
        
        return True
    
    def get_sent_emails(self) -> List[Dict]:
        return self._sent_emails.copy()
    
    def clear(self):
        self._sent_emails.clear()


# ============================================================
# КОМПОНЕНТ 4: ОСНОВНОЙ ПРОЦЕССОР ЗАКАЗОВ
# ============================================================

class OrderProcessor:
    """
    Комплексный модуль обработки заказов.
    Объединяет инвентаризацию, платежи и уведомления.
    """
    
    def __init__(
        self,
        inventory_service: InventoryService,
        payment_gateway: PaymentGateway,
        notification_service: NotificationService
    ):
        self.inventory = inventory_service
        self.payment = payment_gateway
        self.notification = notification_service
        self._orders: Dict[str, Order] = {}
        self.logger = logging.getLogger(__name__)
    
    def create_order(self, customer_id: int, items: List[Dict]) -> Order:
        """
        Создаёт заказ на основе корзины.
        
        Steps:
        1. Валидация входных данных
        2. Проверка наличия товаров
        3. Создание заказа
        4. Резервирование товаров
        5. Сохранение заказа
        """
        # Валидация
        if not items:
            raise ValueError("Order must contain at least one item")
        
        order_items = []
        total = 0.0
        
        # Проверка наличия и расчёт суммы
        for item_data in items:
            product_id = item_data["product_id"]
            quantity = item_data["quantity"]
            price = item_data.get("price", 0)
            
            if not self.inventory.check_availability(product_id, quantity):
                raise InventoryError(f"Product {product_id} is out of stock")
            
            order_item = OrderItem(
                product_id=product_id,
                name=item_data.get("name", f"Product {product_id}"),
                price=price,
                quantity=quantity
            )
            order_items.append(order_item)
            total += order_item.get_total()
        
        # Создание заказа
        order_id = str(uuid.uuid4())
        now = datetime.now()
        
        order = Order(
            id=order_id,
            customer_id=customer_id,
            items=order_items,
            status=OrderStatus.PENDING,
            total=total,
            created_at=now,
            updated_at=now
        )
        
        # Резервирование товаров
        for item in order_items:
            self.inventory.reserve(item.product_id, item.quantity)
        
        # Сохранение
        self._orders[order_id] = order
        
        self.logger.info(f"Order {order_id} created for customer {customer_id}")
        
        return order
    
    def confirm_order(self, order_id: str) -> bool:
        """Подтверждает заказ."""
        order = self._orders.get(order_id)
        if not order:
            raise ValueError(f"Order {order_id} not found")
        
        if order.status != OrderStatus.PENDING:
            raise OrderProcessorError(f"Cannot confirm order in status {order.status}")
        
        order.status = OrderStatus.CONFIRMED
        order.updated_at = datetime.now()
        
        self.logger.info(f"Order {order_id} confirmed")
        return True
    
    def process_payment(self, order_id: str, customer_email: str) -> Dict:
        """
        Обрабатывает оплату заказа.
        
        Steps:
        1. Получение заказа
        2. Проверка статуса
        3. Обработка платежа
        4. Обновление статуса
        5. Уведомление клиента
        """
        order = self._orders.get(order_id)
        if not order:
            raise ValueError(f"Order {order_id} not found")
        
        if order.status != OrderStatus.CONFIRMED:
            raise OrderProcessorError(f"Cannot process payment for order in status {order.status}")
        
        # Обработка платежа
        try:
            transaction = self.payment.process_payment(order.total, order.customer_id)
        except PaymentError as e:
            self.logger.error(f"Payment failed for order {order_id}: {e}")
            raise
        
        # Обновление статуса
        order.status = OrderStatus.PAID
        order.updated_at = datetime.now()
        
        # Уведомление
        try:
            self.notification.send_payment_confirmation(
                customer_email, order_id, order.total
            )
        except NotificationError as e:
            self.logger.warning(f"Failed to send payment confirmation: {e}")
        
        self.logger.info(f"Payment processed for order {order_id}")
        
        return transaction
    
    def cancel_order(self, order_id: str) -> bool:
        """Отменяет заказ."""
        order = self._orders.get(order_id)
        if not order:
            raise ValueError(f"Order {order_id} not found")
        
        if order.status in [OrderStatus.DELIVERED, OrderStatus.SHIPPED]:
            raise OrderProcessorError(f"Cannot cancel order in status {order.status}")
        
        order.status = OrderStatus.CANCELLED
        order.updated_at = datetime.now()
        
        self.logger.info(f"Order {order_id} cancelled")
        return True
    
    def get_order(self, order_id: str) -> Optional[Order]:
        return self._orders.get(order_id)
    
    def get_orders_by_customer(self, customer_id: int) -> List[Order]:
        return [o for o in self._orders.values() if o.customer_id == customer_id]
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Модульные тесты для отдельных компонентов
Средний (варианты 9-17)	Средние студенты	+ Интеграционные тесты + мокирование
Сложный (варианты 18-25)	Сильные студенты	+ Полное покрытие + E2E тесты + отчёт
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться писать модульные тесты для отдельных компонентов в изоляции.

Задания для базового уровня
Задание 1.1. Модульные тесты для InventoryService (15 минут)
python
# test_order_processor_base.py

import pytest
from order_processor import InventoryService, InventoryError


class TestInventoryServiceUnit:
    """Модульные тесты для InventoryService."""
    
    @pytest.fixture
    def inventory(self):
        return InventoryService()
    
    def test_check_availability_default(self, inventory):
        """Проверка наличия с значениями по умолчанию."""
        assert inventory.check_availability(1, 50) is True
        assert inventory.check_availability(1, 150) is False
    
    def test_reserve_success(self, inventory):
        """Успешное резервирование."""
        result = inventory.reserve(1, 30)
        
        assert result is True
        assert inventory._cache[1] == 70  # 100 - 30
    
    def test_reserve_insufficient_stock(self, inventory):
        """Резервирование при недостатке товара."""
        with pytest.raises(InventoryError, match="Insufficient stock"):
            inventory.reserve(1, 200)
    
    def test_reserve_reduces_cache(self, inventory):
        """Резервирование уменьшает кэш."""
        initial = inventory._cache.get(1, 100)
        inventory.reserve(1, 20)
        assert inventory._cache[1] == initial - 20
    
    def test_multiple_reservations(self, inventory):
        """Несколько резервирований подряд."""
        inventory.reserve(1, 30)
        inventory.reserve(1, 20)
        
        assert inventory._cache[1] == 50
    
    def test_clear_cache(self, inventory):
        """Очистка кэша."""
        inventory.reserve(1, 10)
        assert inventory._cache[1] == 90
        
        inventory.clear_cache()
        assert inventory._cache == {}
Задание 1.2. Модульные тесты для PaymentGateway (15 минут)
python
class TestPaymentGatewayUnit:
    """Модульные тесты для PaymentGateway."""
    
    @pytest.fixture
    def gateway(self):
        return PaymentGateway(api_key="test_key")
    
    def test_process_payment_success(self, gateway):
        """Успешная обработка платежа."""
        transaction = gateway.process_payment(100.0, 1)
        
        assert transaction["amount"] == 100.0
        assert transaction["customer_id"] == 1
        assert transaction["status"] == "completed"
        assert "id" in transaction
        assert "timestamp" in transaction
    
    def test_process_payment_negative_amount(self, gateway):
        """Отрицательная сумма платежа."""
        with pytest.raises(PaymentError, match="Amount must be positive"):
            gateway.process_payment(-50, 1)
    
    def test_process_payment_zero_amount(self, gateway):
        """Нулевая сумма платежа."""
        with pytest.raises(PaymentError, match="Amount must be positive"):
            gateway.process_payment(0, 1)
    
    def test_refund_success(self, gateway):
        """Успешный возврат платежа."""
        transaction = gateway.process_payment(100, 1)
        result = gateway.refund(transaction["id"])
        
        assert result is True
        assert gateway.get_transaction(transaction["id"])["status"] == "refunded"
    
    def test_refund_not_found(self, gateway):
        """Возврат несуществующей транзакции."""
        result = gateway.refund("non-existent-id")
        
        assert result is False
    
    def test_get_transaction(self, gateway):
        """Получение информации о транзакции."""
        transaction = gateway.process_payment(50, 2)
        retrieved = gateway.get_transaction(transaction["id"])
        
        assert retrieved is not None
        assert retrieved["amount"] == 50
Задание 1.3. Модульные тесты для NotificationService (10 минут)
python
class TestNotificationServiceUnit:
    """Модульные тесты для NotificationService."""
    
    @pytest.fixture
    def notification(self):
        return NotificationService()
    
    def test_send_order_confirmation_success(self, notification):
        """Успешная отправка подтверждения заказа."""
        result = notification.send_order_confirmation("test@test.com", "ORD-123")
        
        assert result is True
        assert len(notification.get_sent_emails()) == 1
        
        email = notification.get_sent_emails()[0]
        assert email["to"] == "test@test.com"
        assert "ORD-123" in email["subject"]
    
    def test_send_payment_confirmation(self, notification):
        """Отправка подтверждения оплаты."""
        result = notification.send_payment_confirmation("user@test.com", "ORD-456", 99.99)
        
        assert result is True
        email = notification.get_sent_emails()[0]
        assert "$99.99" in email["body"]
        assert "ORD-456" in email["subject"]
    
    def test_send_email_invalid_address(self, notification):
        """Неверный email адрес."""
        with pytest.raises(NotificationError, match="Invalid email"):
            notification.send_order_confirmation("invalid", "ORD-1")
    
    def test_multiple_emails(self, notification):
        """Отправка нескольких писем."""
        notification.send_order_confirmation("a@test.com", "1")
        notification.send_order_confirmation("b@test.com", "2")
        
        assert len(notification.get_sent_emails()) == 2
    
    def test_clear_emails(self, notification):
        """Очистка списка отправленных писем."""
        notification.send_order_confirmation("test@test.com", "1")
        assert len(notification.get_sent_emails()) == 1
        
        notification.clear()
        assert len(notification.get_sent_emails()) == 0
Задание 1.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Результаты модульных тестов

| Компонент | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| InventoryService | _____ | _____ | _____ |
| PaymentGateway | _____ | _____ | _____ |
| NotificationService | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться писать интеграционные тесты и комбинировать компоненты.

Задания для среднего уровня
Задание 2.1. Интеграционные тесты для OrderProcessor (20 минут)
python
# test_order_processor_intermediate.py

import pytest
from order_processor import (
    OrderProcessor, InventoryService, PaymentGateway,
    NotificationService, InventoryError, OrderProcessorError, OrderStatus
)


class TestOrderProcessorIntegration:
    """Интеграционные тесты для OrderProcessor."""
    
    @pytest.fixture
    def order_processor(self):
        inventory = InventoryService()
        payment = PaymentGateway()
        notification = NotificationService()
        return OrderProcessor(inventory, payment, notification)
    
    @pytest.fixture
    def sample_items(self):
        return [
            {"product_id": 1, "name": "Laptop", "price": 1000, "quantity": 1},
            {"product_id": 2, "name": "Mouse", "price": 50, "quantity": 2},
        ]
    
    def test_create_order_success(self, order_processor, sample_items):
        """Успешное создание заказа."""
        order = order_processor.create_order(customer_id=1, items=sample_items)
        
        assert order is not None
        assert order.customer_id == 1
        assert len(order.items) == 2
        assert order.total == 1100  # 1000 + 50*2
        assert order.status == OrderStatus.PENDING
        assert order.id is not None
    
    def test_create_order_empty_items(self, order_processor):
        """Создание заказа без товаров."""
        with pytest.raises(ValueError, match="at least one item"):
            order_processor.create_order(1, [])
    
    def test_create_order_out_of_stock(self, order_processor):
        """Создание заказа с товаром, которого нет в наличии."""
        items = [{"product_id": 999, "name": "Unknown", "price": 100, "quantity": 200}]
        
        with pytest.raises(InventoryError, match="out of stock"):
            order_processor.create_order(1, items)
    
    def test_confirm_order_success(self, order_processor, sample_items):
        """Успешное подтверждение заказа."""
        order = order_processor.create_order(1, sample_items)
        
        result = order_processor.confirm_order(order.id)
        
        assert result is True
        confirmed_order = order_processor.get_order(order.id)
        assert confirmed_order.status == OrderStatus.CONFIRMED
    
    def test_confirm_order_not_found(self, order_processor):
        """Подтверждение несуществующего заказа."""
        with pytest.raises(ValueError, match="not found"):
            order_processor.confirm_order("non-existent")
    
    def test_full_order_workflow(self, order_processor, sample_items):
        """Полный сценарий: создание → подтверждение → оплата."""
        # 1. Создание
        order = order_processor.create_order(1, sample_items)
        assert order.status == OrderStatus.PENDING
        
        # 2. Подтверждение
        order_processor.confirm_order(order.id)
        assert order_processor.get_order(order.id).status == OrderStatus.CONFIRMED
        
        # 3. Оплата
        transaction = order_processor.process_payment(order.id, "customer@test.com")
        assert transaction["amount"] == 1100
        assert order_processor.get_order(order.id).status == OrderStatus.PAID
Задание 2.2. Тестирование с моками зависимостей (15 минут)
python
class TestOrderProcessorWithMocks:
    """Тестирование OrderProcessor с моками зависимостей."""
    
    @pytest.fixture
    def mock_inventory(self, mocker):
        mock = mocker.Mock(spec=InventoryService)
        mock.check_availability.return_value = True
        mock.reserve.return_value = True
        return mock
    
    @pytest.fixture
    def mock_payment(self, mocker):
        mock = mocker.Mock(spec=PaymentGateway)
        mock.process_payment.return_value = {"id": "tx-123", "amount": 100}
        return mock
    
    @pytest.fixture
    def mock_notification(self, mocker):
        mock = mocker.Mock(spec=NotificationService)
        mock.send_order_confirmation.return_value = True
        mock.send_payment_confirmation.return_value = True
        return mock
    
    @pytest.fixture
    def processor(self, mock_inventory, mock_payment, mock_notification):
        return OrderProcessor(mock_inventory, mock_payment, mock_notification)
    
    @pytest.fixture
    def sample_items(self):
        return [{"product_id": 1, "price": 100, "quantity": 2}]
    
    def test_create_order_calls_inventory(self, processor, mock_inventory, sample_items):
        """При создании заказа вызывается check_availability."""
        processor.create_order(1, sample_items)
        
        mock_inventory.check_availability.assert_called_once_with(1, 2)
    
    def test_create_order_calls_reserve(self, processor, mock_inventory, sample_items):
        """При создании заказа вызывается reserve."""
        processor.create_order(1, sample_items)
        
        mock_inventory.reserve.assert_called_once_with(1, 2)
    
    def test_process_payment_calls_payment_gateway(self, processor, mock_payment, sample_items):
        """При оплате вызывается payment gateway."""
        order = processor.create_order(1, sample_items)
        processor.confirm_order(order.id)
        processor.process_payment(order.id, "test@test.com")
        
        mock_payment.process_payment.assert_called_once_with(200, 1)
    
    def test_process_payment_calls_notification(self, processor, mock_notification, sample_items):
        """При оплате отправляется уведомление."""
        order = processor.create_order(1, sample_items)
        processor.confirm_order(order.id)
        processor.process_payment(order.id, "customer@test.com")
        
        mock_notification.send_payment_confirmation.assert_called_once_with(
            "customer@test.com", order.id, 200
        )
    
    def test_verify_mock_calls(self, processor, mock_inventory, mock_payment, mock_notification, sample_items):
        """Проверка последовательности вызовов."""
        from unittest.mock import call
        
        order = processor.create_order(1, sample_items)
        processor.confirm_order(order.id)
        processor.process_payment(order.id, "test@test.com")
        
        expected_inventory_calls = [call(1, 2), call(1, 2)]
        assert mock_inventory.check_availability.call_count == 1
        assert mock_inventory.reserve.call_count == 1
Задание 2.3. Параметризованные тесты (10 минут)
python
class TestOrderProcessorParametrized:
    """Параметризованные тесты."""
    
    @pytest.fixture
    def processor(self):
        return OrderProcessor(InventoryService(), PaymentGateway(), NotificationService())
    
    @pytest.mark.parametrize("items,expected_total", [
        ([{"product_id": 1, "price": 100, "quantity": 1}], 100),
        ([{"product_id": 1, "price": 50, "quantity": 2}], 100),
        ([
            {"product_id": 1, "price": 100, "quantity": 1},
            {"product_id": 2, "price": 50, "quantity": 2},
        ], 200),
        ([
            {"product_id": 1, "price": 10, "quantity": 10},
            {"product_id": 2, "price": 5, "quantity": 5},
        ], 125),
    ])
    def test_order_total_calculation(self, processor, items, expected_total):
        """Проверка расчёта общей суммы заказа."""
        order = processor.create_order(1, items)
        assert order.total == expected_total
    
    @pytest.mark.parametrize("status,can_cancel", [
        (OrderStatus.PENDING, True),
        (OrderStatus.CONFIRMED, True),
        (OrderStatus.PAID, True),
        (OrderStatus.SHIPPED, False),
        (OrderStatus.DELIVERED, False),
        (OrderStatus.CANCELLED, True),
    ])
    def test_cancel_order_by_status(self, processor, status, can_cancel):
        """Отмена заказа в зависимости от статуса."""
        # Создаём заказ с нужным статусом
        order = processor.create_order(1, [{"product_id": 1, "price": 100, "quantity": 1}])
        order.status = status
        
        if can_cancel:
            result = processor.cancel_order(order.id)
            assert result is True
            assert processor.get_order(order.id).status == OrderStatus.CANCELLED
        else:
            with pytest.raises(OrderProcessorError, match="Cannot cancel"):
                processor.cancel_order(order.id)
Задание 2.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Результаты тестирования

| Тип тестов | Количество | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| Модульные (Inventory) | _____ | _____ | _____ |
| Модульные (Payment) | _____ | _____ | _____ |
| Модульные (Notification) | _____ | _____ | _____ |
| Интеграционные (OrderProcessor) | _____ | _____ | _____ |
| С моками | _____ | _____ | _____ |
| Параметризованные | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться писать комплексные тесты, E2E сценарии и анализировать покрытие.

Задания для сложного уровня
Задание 3.1. E2E тесты для полного сценария (15 минут)
python
# test_order_processor_advanced.py

import pytest
from order_processor import (
    OrderProcessor, InventoryService, PaymentGateway,
    NotificationService, OrderStatus
)


class TestEndToEnd:
    """Сквозные E2E тесты."""
    
    @pytest.fixture
    def processor(self):
        inventory = InventoryService()
        payment = PaymentGateway()
        notification = NotificationService()
        return OrderProcessor(inventory, payment, notification)
    
    @pytest.fixture
    def sample_items(self):
        return [
            {"product_id": 1, "name": "Laptop", "price": 1000, "quantity": 1},
            {"product_id": 2, "name": "Mouse", "price": 50, "quantity": 2},
        ]
    
    def test_complete_happy_path(self, processor, sample_items):
        """Полный счастливый путь: создание → подтверждение → оплата."""
        customer_email = "customer@example.com"
        
        # 1. Создание заказа
        order = processor.create_order(1, sample_items)
        assert order.status == OrderStatus.PENDING
        assert order.total == 1100
        
        # 2. Подтверждение заказа
        processor.confirm_order(order.id)
        assert processor.get_order(order.id).status == OrderStatus.CONFIRMED
        
        # 3. Обработка платежа
        transaction = processor.process_payment(order.id, customer_email)
        assert transaction["amount"] == 1100
        
        # 4. Проверка финального статуса
        final_order = processor.get_order(order.id)
        assert final_order.status == OrderStatus.PAID
        
        # 5. Проверка, что транзакция сохранена
        assert processor.payment.get_transaction(transaction["id"]) is not None
        
        # 6. Проверка, что уведомление отправлено
        emails = processor.notification.get_sent_emails()
        assert len(emails) == 1
        assert emails[0]["to"] == customer_email
    
    def test_cancel_before_payment(self, processor, sample_items):
        """Отмена заказа до оплаты."""
        order = processor.create_order(1, sample_items)
        processor.confirm_order(order.id)
        
        processor.cancel_order(order.id)
        
        assert processor.get_order(order.id).status == OrderStatus.CANCELLED
    
    def test_payment_failure_handling(self, processor, sample_items, mocker):
        """Обработка ошибки платежа."""
        # Мокируем ошибку платежа
        mocker.patch.object(processor.payment, 'process_payment',
                           side_effect=Exception("Payment gateway error"))
        
        order = processor.create_order(1, sample_items)
        processor.confirm_order(order.id)
        
        with pytest.raises(Exception, match="Payment gateway error"):
            processor.process_payment(order.id, "test@test.com")
        
        # Статус не должен измениться
        assert processor.get_order(order.id).status == OrderStatus.CONFIRMED
Задание 3.2. Тестирование граничных случаев и ошибок (10 минут)
python
class TestEdgeCases:
    """Тестирование граничных случаев."""
    
    @pytest.fixture
    def processor(self):
        return OrderProcessor(InventoryService(), PaymentGateway(), NotificationService())
    
    def test_multiple_orders_same_customer(self, processor):
        """Несколько заказов одного клиента."""
        items = [{"product_id": 1, "price": 100, "quantity": 1}]
        
        order1 = processor.create_order(1, items)
        order2 = processor.create_order(1, items)
        
        customer_orders = processor.get_orders_by_customer(1)
        assert len(customer_orders) == 2
        assert order1.id in [o.id for o in customer_orders]
        assert order2.id in [o.id for o in customer_orders]
    
    def test_get_nonexistent_order(self, processor):
        """Получение несуществующего заказа."""
        result = processor.get_order("non-existent")
        assert result is None
    
    def test_confirm_already_confirmed_order(self, processor):
        """Подтверждение уже подтверждённого заказа."""
        items = [{"product_id": 1, "price": 100, "quantity": 1}]
        order = processor.create_order(1, items)
        processor.confirm_order(order.id)
        
        with pytest.raises(OrderProcessorError, match="Cannot confirm"):
            processor.confirm_order(order.id)
    
    def test_payment_without_confirmation(self, processor):
        """Оплата без предварительного подтверждения."""
        items = [{"product_id": 1, "price": 100, "quantity": 1}]
        order = processor.create_order(1, items)
        
        with pytest.raises(OrderProcessorError, match="Cannot process payment"):
            processor.process_payment(order.id, "test@test.com")
Задание 3.3. Полный набор тестов с покрытием (10 минут)
python
class TestFullCoverage:
    """Тесты для достижения полного покрытия."""
    
    @pytest.fixture
    def processor(self):
        return OrderProcessor(InventoryService(), PaymentGateway(), NotificationService())
    
    def test_inventory_service_with_db_client(self):
        """InventoryService с реальным клиентом БД (мок)."""
        class MockDB:
            def query(self, sql):
                return [(50,)]
        
        inventory = InventoryService(db_client=MockDB())
        assert inventory.check_availability(1, 30) is True
        assert inventory.check_availability(1, 60) is False
    
    def test_notification_service_with_smtp(self, mocker):
        """NotificationService с SMTP клиентом."""
        mock_smtp = mocker.Mock()
        mock_smtp.send.return_value = True
        
        notification = NotificationService(smtp_client=mock_smtp)
        result = notification.send_order_confirmation("test@test.com", "ORD-1")
        
        assert result is True
        mock_smtp.send.assert_called_once()
    
    def test_order_processor_error_handling(self, processor):
        """Обработка ошибок в OrderProcessor."""
        items = [{"product_id": 1, "price": 100, "quantity": 1}]
        order = processor.create_order(1, items)
        
        # Повторное создание с тем же ID невозможно
        # (ID генерируется автоматически, так что это не проблема)
        
        # Отмена после оплаты
        processor.confirm_order(order.id)
        processor.process_payment(order.id, "test@test.com")
        
        # Нельзя отменить оплаченный заказ (если не доставлен)
        # В нашей реализации можно, так как PAID не запрещает отмену
        result = processor.cancel_order(order.id)
        assert result is True
Задание 3.4. Анализ покрытия и итоговый отчёт (5 минут)
bash
# Запуск тестов с измерением покрытия
pytest test_order_processor_advanced.py --cov=order_processor --cov-branch --cov-report=term --cov-report=html
markdown
## Итоговый отчёт по тестированию (сложный уровень)

**Студент:** _________________
**Вариант №:** ___ (18-25)

### Результаты тестирования

| Категория | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| Модульные тесты | _____ | _____ | _____ |
| Интеграционные тесты | _____ | _____ | _____ |
| E2E тесты | _____ | _____ | _____ |
| Тесты ошибок | _____ | _____ | _____ |
| Параметризованные | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Покрытие кода

| Компонент | Покрытие строк | Покрытие ветвей |
|:---|:---|:---|
| InventoryService | _____% | _____% |
| PaymentGateway | _____% | _____% |
| NotificationService | _____% | _____% |
| OrderProcessor | _____% | _____% |

### Выводы

_______________________________________________________________

_______________________________________________________________
Карточка студента
text
ПР 3.11. МОДУЛЬНЫЕ И ИНТЕГРАЦИОННЫЕ ТЕСТЫ

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ Модульные тесты InventoryService (базовый)
□ Модульные тесты PaymentGateway (базовый)
□ Модульные тесты NotificationService (базовый)
□ Интеграционные тесты (средний)
□ Тесты с моками (средний)
□ Параметризованные тесты (средний)
□ E2E тесты (сложный)
□ Граничные случаи (сложный)
□ Полное покрытие (сложный)

=== РЕЗУЛЬТАТЫ ===

Всего тестов: _____
Пройдено: _____
Не пройдено: _____

Покрытие кода: _____%

=== ВРЕМЯ ВЫПОЛНЕНИЯ ===

Подготовка: _____ мин
Выполнение: _____ мин
Оформление: _____ мин

=== ОТЧЁТ ===

Файлы:
- test_order_processor_base.py
- test_order_processor_intermediate.py
- test_order_processor_advanced.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Тесты не работают
3 (удовлетворительно)	Базовый	Модульные тесты для 3 компонентов (15+ тестов)
4 (хорошо)	Средний	+ Интеграционные + моки + параметризация (25+ тестов)
5 (отлично)	Сложный	+ E2E + граничные случаи + покрытие (35+ тестов)
Контрольные вопросы (для защиты)
Чем отличается модульное тестирование от интеграционного?

Какие зависимости нужно мокировать при модульном тестировании?

Как проверить, что при создании заказа вызывается метод резервирования?

Какие сценарии нужно покрыть интеграционными тестами?

Как параметризация помогает в тестировании комплексных модулей?

Какие граничные случаи важны для тестирования заказов?

Как проверить обработку ошибок платежа?

Как обеспечить высокое покрытие кода для комплексного модуля?

Какие E2E сценарии нужно тестировать?

Как организовать тесты для больших проектов?

Следующее занятие: Контрольная работа 3.4 по темам 3.9–3.15.
