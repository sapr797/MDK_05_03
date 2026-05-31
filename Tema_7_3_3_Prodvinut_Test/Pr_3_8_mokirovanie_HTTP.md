# Практическое занятие 3.7: Тестирование пользовательских исключений и иерархии

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться тестировать пользовательские исключения и их иерархии, проверять обработку ошибок, атрибуты исключений и корректность работы механизмов обработки ошибок.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Создавать и тестировать иерархии пользовательских исключений (ПК 3.2, ПК 3.3).
2. Проверять атрибуты исключений и сообщения об ошибках (ПК 3.2).
3. Тестировать обработку ошибок в различных сценариях (ПК 3.3).
4. Использовать `pytest.raises` для проверки типов и атрибутов исключений (ОК 02, ОК 05).

---

## Теоретическая справка

### Иерархия исключений
┌─────────────────────────────────────────────────────────────────────────────┐
│ ИЕРАРХИЯ ПОЛЬЗОВАТЕЛЬСКИХ ИСКЛЮЧЕНИЙ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ BaseAppException │
│ △ │
│ │ │
│ ┌────────────────┼────────────────┐ │
│ │ │ │ │
│ ValidationError BusinessError SystemError │
│ △ △ △ │
│ │ │ │ │
│ ┌──────┼──────┐ ┌─────┼─────┐ ┌─────┼─────┐ │
│ │ │ │ │ │ │ │ │ │ │
│ Required Format │ Insufficient Product│ DB │ API │ │
│ Field Error │ Funds OutOfStock │Error │ Error│ │
│ │ │ │ │ │ │
│ FieldError │ │ │ │ │
│ │ │ │ │ │
│ BusinessRuleError │ │ │ │
│ │ │ │ │
│ ConnectionError │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

---

## Объект тестирования

### Листинг 1. Модуль exceptions.py

```python
# exceptions.py

from datetime import datetime
from typing import Optional, Dict, Any


# ============================================================
# БАЗОВЫЕ ИСКЛЮЧЕНИЯ
# ============================================================

class BaseAppException(Exception):
    """Базовое исключение приложения."""
    
    def __init__(
        self,
        message: str,
        code: Optional[int] = None,
        details: Optional[Dict[str, Any]] = None,
        cause: Optional[Exception] = None
    ):
        self.message = message
        self.code = code
        self.details = details or {}
        self.cause = cause
        self.timestamp = datetime.now()
        
        super().__init__(self.message)
    
    def to_dict(self) -> Dict[str, Any]:
        """Преобразует исключение в словарь."""
        return {
            "type": self.__class__.__name__,
            "message": self.message,
            "code": self.code,
            "details": self.details,
            "timestamp": self.timestamp.isoformat()
        }
    
    def __str__(self) -> str:
        parts = [self.message]
        if self.code:
            parts.append(f"[code: {self.code}]")
        if self.details:
            parts.append(f"details: {self.details}")
        return " ".join(parts)


# ============================================================
# ИСКЛЮЧЕНИЯ ВАЛИДАЦИИ
# ============================================================

class ValidationError(BaseAppException):
    """Ошибка валидации."""
    pass


class RequiredFieldError(ValidationError):
    """Отсутствует обязательное поле."""
    
    def __init__(self, field_name: str):
        self.field_name = field_name
        super().__init__(
            message=f"Required field '{field_name}' is missing",
            code=400,
            details={"field": field_name}
        )


class InvalidFormatError(ValidationError):
    """Неверный формат поля."""
    
    def __init__(self, field_name: str, expected_format: str, actual_value: Any):
        self.field_name = field_name
        self.expected_format = expected_format
        self.actual_value = actual_value
        super().__init__(
            message=f"Invalid format for '{field_name}': expected {expected_format}, got '{actual_value}'",
            code=400,
            details={
                "field": field_name,
                "expected_format": expected_format,
                "actual_value": str(actual_value)
            }
        )


class RangeError(ValidationError):
    """Значение вне допустимого диапазона."""
    
    def __init__(self, field_name: str, min_value: Any, max_value: Any, actual_value: Any):
        self.field_name = field_name
        self.min_value = min_value
        self.max_value = max_value
        self.actual_value = actual_value
        super().__init__(
            message=f"Value for '{field_name}' must be between {min_value} and {max_value}, got {actual_value}",
            code=400,
            details={
                "field": field_name,
                "min": min_value,
                "max": max_value,
                "actual": actual_value
            }
        )


# ============================================================
# БИЗНЕС-ИСКЛЮЧЕНИЯ
# ============================================================

class BusinessError(BaseAppException):
    """Ошибка бизнес-логики."""
    pass


class InsufficientFundsError(BusinessError):
    """Недостаточно средств."""
    
    def __init__(self, balance: float, requested: float, currency: str = "RUB"):
        self.balance = balance
        self.requested = requested
        self.currency = currency
        super().__init__(
            message=f"Insufficient funds: requested {requested} {currency}, available {balance} {currency}",
            code=422,
            details={
                "balance": balance,
                "requested": requested,
                "currency": currency,
                "shortage": requested - balance
            }
        )


class ProductOutOfStockError(BusinessError):
    """Товар отсутствует на складе."""
    
    def __init__(self, product_id: int, requested: int, available: int):
        self.product_id = product_id
        self.requested = requested
        self.available = available
        super().__init__(
            message=f"Product {product_id} out of stock: requested {requested}, available {available}",
            code=422,
            details={
                "product_id": product_id,
                "requested": requested,
                "available": available
            }
        )


class OrderStateError(BusinessError):
    """Недопустимая операция для текущего состояния заказа."""
    
    def __init__(self, order_id: int, current_state: str, attempted_operation: str):
        self.order_id = order_id
        self.current_state = current_state
        self.attempted_operation = attempted_operation
        super().__init__(
            message=f"Cannot {attempted_operation} order {order_id} in state '{current_state}'",
            code=422,
            details={
                "order_id": order_id,
                "current_state": current_state,
                "attempted_operation": attempted_operation
            }
        )


# ============================================================
# СИСТЕМНЫЕ ИСКЛЮЧЕНИЯ
# ============================================================

class SystemError(BaseAppException):
    """Системная ошибка."""
    pass


class DatabaseError(SystemError):
    """Ошибка базы данных."""
    
    def __init__(self, operation: str, message: str, cause: Optional[Exception] = None):
        self.operation = operation
        super().__init__(
            message=f"Database error during {operation}: {message}",
            code=500,
            details={"operation": operation},
            cause=cause
        )


class APIClientError(SystemError):
    """Ошибка вызова внешнего API."""
    
    def __init__(self, endpoint: str, status_code: int, response: str):
        self.endpoint = endpoint
        self.status_code = status_code
        self.response = response
        super().__init__(
            message=f"API call to {endpoint} failed with status {status_code}",
            code=502,
            details={
                "endpoint": endpoint,
                "status_code": status_code,
                "response": response[:200]
            }
        )


class ConnectionError(SystemError):
    """Ошибка соединения."""
    
    def __init__(self, service: str, reason: str):
        self.service = service
        self.reason = reason
        super().__init__(
            message=f"Connection to {service} failed: {reason}",
            code=503,
            details={"service": service, "reason": reason}
        )
Листинг 2. Модуль user_service.py (с обработкой ошибок)
python
# user_service.py

from typing import Dict, Any, Optional, List
from exceptions import (
    RequiredFieldError, InvalidFormatError, RangeError,
    InsufficientFundsError, ProductOutOfStockError,
    DatabaseError, APIClientError, ConnectionError,
    ValidationError
)


class UserService:
    """Сервис для работы с пользователями с обработкой ошибок."""
    
    def __init__(self, db_client=None, payment_gateway=None, email_service=None):
        self.db_client = db_client
        self.payment_gateway = payment_gateway
        self.email_service = email_service
    
    def validate_user_data(self, user_data: Dict[str, Any]) -> None:
        """
        Валидирует данные пользователя.
        
        Raises:
            RequiredFieldError: При отсутствии обязательных полей
            InvalidFormatError: При неверном формате
            RangeError: При неверном возрасте
        """
        # Проверка обязательных полей
        required_fields = ["name", "email", "age"]
        for field in required_fields:
            if field not in user_data or not user_data[field]:
                raise RequiredFieldError(field)
        
        # Проверка формата email
        email = user_data["email"]
        if "@" not in email or "." not in email:
            raise InvalidFormatError("email", "email@domain.com", email)
        
        # Проверка возраста
        age = user_data["age"]
        if not isinstance(age, int):
            raise InvalidFormatError("age", "integer", age)
        if age < 18 or age > 120:
            raise RangeError("age", 18, 120, age)
    
    def create_user(self, user_data: Dict[str, Any]) -> int:
        """
        Создаёт нового пользователя.
        
        Returns:
            ID созданного пользователя
        
        Raises:
            ValidationError: При неверных данных
            DatabaseError: При ошибке БД
        """
        # Валидация
        self.validate_user_data(user_data)
        
        # Сохранение в БД
        try:
            user_id = self.db_client.insert("users", user_data)
            return user_id
        except Exception as e:
            raise DatabaseError("insert user", str(e), cause=e)
    
    def process_payment(self, user_id: int, amount: float) -> Dict[str, Any]:
        """
        Обрабатывает платёж пользователя.
        
        Raises:
            InsufficientFundsError: При недостатке средств
            APIClientError: При ошибке API
            ConnectionError: При проблемах соединения
        """
        # Получаем баланс пользователя
        user = self.db_client.get_user(user_id)
        if not user:
            raise ValidationError(f"User {user_id} not found", code=404)
        
        if user["balance"] < amount:
            raise InsufficientFundsError(user["balance"], amount)
        
        try:
            result = self.payment_gateway.charge(user["card_token"], amount)
            return {"success": True, "transaction_id": result["id"]}
        except ConnectionError:
            raise
        except Exception as e:
            raise APIClientError("/charge", 500, str(e))
    
    def get_user_balance(self, user_id: int) -> float:
        """Возвращает баланс пользователя."""
        try:
            user = self.db_client.get_user(user_id)
            return user["balance"] if user else 0.0
        except Exception as e:
            raise DatabaseError("get user balance", str(e), cause=e)
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Тестирование базовых исключений и атрибутов
Средний (варианты 9-17)	Средние студенты	+ иерархия исключений + наследование
Сложный (варианты 18-25)	Сильные студенты	+ обработка ошибок + контекст + причина
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться тестировать отдельные исключения и проверять их атрибуты.

Задания для базового уровня
Задание 1.1. Тестирование базовых исключений (15 минут)
python
# test_exceptions_base.py

import pytest
from datetime import datetime
from exceptions import (
    BaseAppException, RequiredFieldError, InvalidFormatError, RangeError
)


class TestBaseExceptions:
    """Тестирование базовых исключений."""
    
    # ==========================================================
    # ТЕСТЫ RequiredFieldError
    # ==========================================================
    
    def test_required_field_error_attributes(self):
        """Проверка атрибутов RequiredFieldError."""
        error = RequiredFieldError("email")
        
        assert error.field_name == "email"
        assert error.code == 400
        assert "Required field 'email' is missing" in error.message
        assert error.details["field"] == "email"
        assert isinstance(error.timestamp, datetime)
    
    def test_required_field_error_str(self):
        """Проверка строкового представления."""
        error = RequiredFieldError("password")
        
        error_str = str(error)
        assert "Required field 'password' is missing" in error_str
        assert "[code: 400]" in error_str
    
    def test_required_field_error_to_dict(self):
        """Проверка преобразования в словарь."""
        error = RequiredFieldError("name")
        error_dict = error.to_dict()
        
        assert error_dict["type"] == "RequiredFieldError"
        assert error_dict["code"] == 400
        assert error_dict["details"]["field"] == "name"
        assert "timestamp" in error_dict
    
    # ==========================================================
    # ТЕСТЫ InvalidFormatError
    # ==========================================================
    
    def test_invalid_format_error_attributes(self):
        """Проверка атрибутов InvalidFormatError."""
        error = InvalidFormatError("email", "email@domain.com", "invalid")
        
        assert error.field_name == "email"
        assert error.expected_format == "email@domain.com"
        assert error.actual_value == "invalid"
        assert error.code == 400
        assert "Invalid format for 'email'" in error.message
    
    def test_invalid_format_error_str(self):
        """Проверка строкового представления."""
        error = InvalidFormatError("age", "integer", "twenty")
        
        error_str = str(error)
        assert "Invalid format for 'age'" in error_str
        assert "expected integer" in error_str
        assert "got 'twenty'" in error_str
    
    # ==========================================================
    # ТЕСТЫ RangeError
    # ==========================================================
    
    def test_range_error_attributes(self):
        """Проверка атрибутов RangeError."""
        error = RangeError("age", 18, 120, 150)
        
        assert error.field_name == "age"
        assert error.min_value == 18
        assert error.max_value == 120
        assert error.actual_value == 150
        assert error.code == 400
        assert "between 18 and 120" in error.message
    
    def test_range_error_with_string_values(self):
        """RangeError со строковыми значениями."""
        error = RangeError("status", "active", "inactive", "pending")
        
        assert error.message == "Value for 'status' must be between active and inactive, got pending"
    
    # ==========================================================
    # ТЕСТЫ С pytest.raises
    # ==========================================================
    
    def test_raise_required_field_error(self):
        """Проверка, что исключение выбрасывается."""
        with pytest.raises(RequiredFieldError) as exc_info:
            raise RequiredFieldError("username")
        
        assert exc_info.value.field_name == "username"
        assert exc_info.value.code == 400
    
    def test_raise_invalid_format_error(self):
        """Проверка InvalidFormatError."""
        with pytest.raises(InvalidFormatError) as exc_info:
            raise InvalidFormatError("phone", "+7XXXXXXXXXX", "123")
        
        assert exc_info.value.field_name == "phone"
        assert "123" in str(exc_info.value)
Задание 1.2. Тестирование бизнес-исключений (15 минут)
python
class TestBusinessExceptions:
    """Тестирование бизнес-исключений."""
    
    # ==========================================================
    # ТЕСТЫ InsufficientFundsError
    # ==========================================================
    
    def test_insufficient_funds_error_attributes(self):
        """Проверка атрибутов InsufficientFundsError."""
        error = InsufficientFundsError(balance=100.0, requested=150.0)
        
        assert error.balance == 100.0
        assert error.requested == 150.0
        assert error.currency == "RUB"
        assert error.code == 422
        assert "insufficient funds" in error.message.lower()
        assert error.details["shortage"] == 50.0
    
    def test_insufficient_funds_error_custom_currency(self):
        """InsufficientFundsError с другой валютой."""
        error = InsufficientFundsError(balance=50.0, requested=100.0, currency="USD")
        
        assert error.currency == "USD"
        assert "50 USD" in error.message
        assert "100 USD" in error.message
    
    def test_insufficient_funds_error_float_precision(self):
        """Проверка точности чисел."""
        error = InsufficientFundsError(balance=100.10, requested=100.20)
        
        assert error.details["shortage"] == 0.10
    
    # ==========================================================
    # ТЕСТЫ ProductOutOfStockError
    # ==========================================================
    
    def test_product_out_of_stock_error(self):
        """Проверка ProductOutOfStockError."""
        error = ProductOutOfStockError(product_id=123, requested=5, available=2)
        
        assert error.product_id == 123
        assert error.requested == 5
        assert error.available == 2
        assert error.code == 422
        assert "Product 123 out of stock" in error.message
        assert error.details["product_id"] == 123
    
    # ==========================================================
    # ТЕСТЫ OrderStateError
    # ==========================================================
    
    def test_order_state_error(self):
        """Проверка OrderStateError."""
        error = OrderStateError(order_id=456, current_state="shipped", attempted_operation="cancel")
        
        assert error.order_id == 456
        assert error.current_state == "shipped"
        assert error.attempted_operation == "cancel"
        assert "Cannot cancel order 456 in state 'shipped'" in error.message
Задание 1.3. Тестирование системных исключений (10 минут)
python
class TestSystemExceptions:
    """Тестирование системных исключений."""
    
    def test_database_error_with_cause(self):
        """DatabaseError с причиной."""
        original_error = ValueError("Connection refused")
        error = DatabaseError("insert user", "Failed to connect", cause=original_error)
        
        assert error.operation == "insert user"
        assert error.code == 500
        assert "Database error during insert user" in error.message
        assert error.cause is original_error
    
    def test_api_client_error(self):
        """APIClientError."""
        error = APIClientError("/users", 404, "User not found")
        
        assert error.endpoint == "/users"
        assert error.status_code == 404
        assert error.response == "User not found"
        assert error.code == 502
        assert "API call to /users failed with status 404" in error.message
    
    def test_connection_error(self):
        """ConnectionError."""
        error = ConnectionError("database", "Timeout after 30s")
        
        assert error.service == "database"
        assert error.reason == "Timeout after 30s"
        assert error.code == 503
        assert "Connection to database failed" in error.message
    
    def test_to_dict_includes_cause_info(self):
        """to_dict должен включать информацию о причине."""
        cause = ValueError("Original error")
        error = DatabaseError("select", "Failed", cause=cause)
        error_dict = error.to_dict()
        
        assert error_dict["type"] == "DatabaseError"
        assert error_dict["code"] == 500
        # Причина не включается в to_dict (только в атрибут)
Задание 1.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Результаты тестирования исключений

| Исключение | Атрибуты | Сообщение | to_dict |
|:---|:---|:---|:---|
| RequiredFieldError | ☐ | ☐ | ☐ |
| InvalidFormatError | ☐ | ☐ | ☐ |
| RangeError | ☐ | ☐ | ☐ |
| InsufficientFundsError | ☐ | ☐ | ☐ |
| ProductOutOfStockError | ☐ | ☐ | ☐ |
| OrderStateError | ☐ | ☐ | ☐ |
| DatabaseError | ☐ | ☐ | ☐ |
| APIClientError | ☐ | ☐ | ☐ |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться тестировать иерархию исключений и полиморфную обработку.

Задания для среднего уровня
Задание 2.1. Тестирование иерархии исключений (15 минут)
python
# test_exceptions_intermediate.py

import pytest
from exceptions import (
    BaseAppException, ValidationError, BusinessError, SystemError,
    RequiredFieldError, InvalidFormatError, RangeError,
    InsufficientFundsError, ProductOutOfStockError,
    DatabaseError, APIClientError
)


class TestExceptionHierarchy:
    """Тестирование иерархии исключений."""
    
    def test_validation_error_hierarchy(self):
        """Проверка, что ValidationError является BaseAppException."""
        error = ValidationError("Validation failed")
        
        assert isinstance(error, BaseAppException)
        assert isinstance(error, ValidationError)
        assert not isinstance(error, BusinessError)
    
    def test_required_field_error_in_hierarchy(self):
        """RequiredFieldError должен быть ValidationError."""
        error = RequiredFieldError("email")
        
        assert isinstance(error, ValidationError)
        assert isinstance(error, BaseAppException)
    
    def test_insufficient_funds_hierarchy(self):
        """InsufficientFundsError должен быть BusinessError."""
        error = InsufficientFundsError(100, 200)
        
        assert isinstance(error, BusinessError)
        assert isinstance(error, BaseAppException)
        assert not isinstance(error, ValidationError)
    
    def test_database_error_hierarchy(self):
        """DatabaseError должен быть SystemError."""
        error = DatabaseError("query", "failed")
        
        assert isinstance(error, SystemError)
        assert isinstance(error, BaseAppException)
        assert not isinstance(error, BusinessError)
    
    def test_polymorphic_exception_handling(self):
        """Полиморфная обработка исключений."""
        
        def handle_error(error: BaseAppException) -> str:
            if isinstance(error, ValidationError):
                return "validation"
            elif isinstance(error, BusinessError):
                return "business"
            elif isinstance(error, SystemError):
                return "system"
            return "unknown"
        
        assert handle_error(RequiredFieldError("name")) == "validation"
        assert handle_error(InsufficientFundsError(100, 200)) == "business"
        assert handle_error(DatabaseError("select", "fail")) == "system"
Задание 2.2. Тестирование полиморфной обработки (15 минут)
python
class TestPolymorphicHandling:
    """Тестирование полиморфной обработки ошибок."""
    
    @pytest.fixture
    def error_handler(self):
        class ErrorHandler:
            def handle(self, error: Exception) -> dict:
                if isinstance(error, RequiredFieldError):
                    return {"status": "validation", "field": error.field_name}
                elif isinstance(error, InvalidFormatError):
                    return {"status": "validation", "field": error.field_name}
                elif isinstance(error, RangeError):
                    return {"status": "validation", "field": error.field_name}
                elif isinstance(error, InsufficientFundsError):
                    return {"status": "business", "shortage": error.details["shortage"]}
                elif isinstance(error, ProductOutOfStockError):
                    return {"status": "business", "product_id": error.product_id}
                elif isinstance(error, DatabaseError):
                    return {"status": "system", "operation": error.operation}
                elif isinstance(error, APIClientError):
                    return {"status": "system", "endpoint": error.endpoint}
                else:
                    return {"status": "unknown"}
        
        return ErrorHandler()
    
    def test_handle_validation_errors(self, error_handler):
        """Обработка различных ошибок валидации."""
        
        # RequiredFieldError
        result = error_handler.handle(RequiredFieldError("email"))
        assert result["status"] == "validation"
        assert result["field"] == "email"
        
        # InvalidFormatError
        result = error_handler.handle(InvalidFormatError("age", "int", "abc"))
        assert result["status"] == "validation"
        assert result["field"] == "age"
        
        # RangeError
        result = error_handler.handle(RangeError("age", 0, 150, 200))
        assert result["status"] == "validation"
    
    def test_handle_business_errors(self, error_handler):
        """Обработка бизнес-ошибок."""
        
        result = error_handler.handle(InsufficientFundsError(100, 200))
        assert result["status"] == "business"
        assert result["shortage"] == 100
        
        result = error_handler.handle(ProductOutOfStockError(123, 5, 2))
        assert result["status"] == "business"
        assert result["product_id"] == 123
    
    def test_handle_system_errors(self, error_handler):
        """Обработка системных ошибок."""
        
        result = error_handler.handle(DatabaseError("insert", "connection lost"))
        assert result["status"] == "system"
        assert result["operation"] == "insert"
        
        result = error_handler.handle(APIClientError("/users", 500, "Internal error"))
        assert result["status"] == "system"
        assert result["endpoint"] == "/users"
    
    def test_catch_base_exception(self, error_handler):
        """Перехват базового исключения."""
        
        class CustomError(BaseAppException):
            pass
        
        result = error_handler.handle(CustomError("test"))
        assert result["status"] == "unknown"
Задание 2.3. Тестирование с pytest.raises и match (10 минут)
python
class TestPytestRaisesMatch:
    """Тестирование с использованием match."""
    
    def test_required_field_error_match(self):
        """Проверка сообщения с match."""
        with pytest.raises(RequiredFieldError, match="Required field 'email' is missing"):
            raise RequiredFieldError("email")
    
    def test_range_error_match(self):
        """Проверка RangeError с регулярным выражением."""
        with pytest.raises(RangeError, match=r"between 18 and 120, got 150"):
            raise RangeError("age", 18, 120, 150)
    
    def test_insufficient_funds_match(self):
        """Проверка InsufficientFundsError."""
        with pytest.raises(InsufficientFundsError, match=r"requested 200\.0 RUB, available 100\.0 RUB"):
            raise InsufficientFundsError(100.0, 200.0)
    
    def test_any_validation_error(self):
        """Перехват любого ValidationError."""
        with pytest.raises(ValidationError):
            raise RequiredFieldError("email")
        
        with pytest.raises(ValidationError):
            raise InvalidFormatError("email", "format", "invalid")
    
    def test_any_business_error(self):
        """Перехват любого BusinessError."""
        with pytest.raises(BusinessError):
            raise InsufficientFundsError(100, 200)
        
        with pytest.raises(BusinessError):
            raise ProductOutOfStockError(1, 5, 2)
    
    def test_catch_multiple_exception_types(self):
        """Перехват нескольких типов исключений."""
        exceptions = [RequiredFieldError("a"), InsufficientFundsError(100, 200)]
        
        for exc in exceptions:
            with pytest.raises((ValidationError, BusinessError)):
                raise exc
Задание 2.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Проверка иерархии

| Исключение | BaseAppException | ValidationError | BusinessError | SystemError |
|:---|:---|:---|:---|:---|
| RequiredFieldError | ☐ | ☐ | ☐ | ☐ |
| InvalidFormatError | ☐ | ☐ | ☐ | ☐ |
| InsufficientFundsError | ☐ | ☐ | ☐ | ☐ |
| DatabaseError | ☐ | ☐ | ☐ | ☐ |

### Полиморфная обработка

| Тип ошибки | Результат обработки |
|:---|:---|
| RequiredFieldError | _____ |
| InsufficientFundsError | _____ |
| DatabaseError | _____ |

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться тестировать обработку ошибок в сервисе с полной цепочкой исключений.

Задания для сложного уровня
Задание 3.1. Тестирование UserService с моками (20 минут)
python
# test_exceptions_advanced.py

import pytest
from unittest.mock import Mock
from user_service import UserService
from exceptions import (
    RequiredFieldError, InvalidFormatError, RangeError,
    InsufficientFundsError, DatabaseError, APIClientError,
    ConnectionError, ValidationError
)


class TestUserServiceExceptions:
    """Тестирование обработки ошибок в UserService."""
    
    @pytest.fixture
    def mock_db(self):
        db = Mock()
        db.insert.return_value = 123
        db.get_user.return_value = {"id": 1, "balance": 1000, "card_token": "token123"}
        return db
    
    @pytest.fixture
    def mock_payment(self):
        payment = Mock()
        payment.charge.return_value = {"id": "tx_456"}
        return payment
    
    @pytest.fixture
    def user_service(self, mock_db, mock_payment):
        return UserService(
            db_client=mock_db,
            payment_gateway=mock_payment,
            email_service=Mock()
        )
    
    # ==========================================================
    # ТЕСТЫ ВАЛИДАЦИИ
    # ==========================================================
    
    def test_validate_user_missing_name(self, user_service):
        """Отсутствие имени -> RequiredFieldError."""
        user_data = {"email": "test@test.com", "age": 25}
        
        with pytest.raises(RequiredFieldError) as exc_info:
            user_service.validate_user_data(user_data)
        
        assert exc_info.value.field_name == "name"
    
    def test_validate_user_invalid_email(self, user_service):
        """Неверный email -> InvalidFormatError."""
        user_data = {"name": "Test", "email": "invalid", "age": 25}
        
        with pytest.raises(InvalidFormatError) as exc_info:
            user_service.validate_user_data(user_data)
        
        assert exc_info.value.field_name == "email"
    
    def test_validate_user_age_too_high(self, user_service):
        """Возраст слишком высок -> RangeError."""
        user_data = {"name": "Test", "email": "test@test.com", "age": 200}
        
        with pytest.raises(RangeError) as exc_info:
            user_service.validate_user_data(user_data)
        
        assert exc_info.value.field_name == "age"
        assert exc_info.value.max_value == 120
        assert exc_info.value.actual_value == 200
    
    def test_validate_user_age_too_low(self, user_service):
        """Возраст слишком низкий -> RangeError."""
        user_data = {"name": "Test", "email": "test@test.com", "age": 16}
        
        with pytest.raises(RangeError) as exc_info:
            user_service.validate_user_data(user_data)
        
        assert exc_info.value.field_name == "age"
        assert exc_info.value.min_value == 18
    
    def test_validate_user_valid(self, user_service):
        """Валидные данные не вызывают исключений."""
        user_data = {"name": "Test", "email": "test@test.com", "age": 30}
        
        # Не должно быть исключения
        user_service.validate_user_data(user_data)
    
    # ==========================================================
    # ТЕСТЫ create_user
    # ==========================================================
    
    def test_create_user_success(self, user_service):
        """Успешное создание пользователя."""
        user_data = {"name": "Alice", "email": "alice@test.com", "age": 25}
        
        user_id = user_service.create_user(user_data)
        
        assert user_id == 123
        user_service.db_client.insert.assert_called_once()
    
    def test_create_user_database_error(self, user_service, mock_db):
        """Ошибка базы данных -> DatabaseError."""
        mock_db.insert.side_effect = Exception("Connection lost")
        
        user_data = {"name": "Bob", "email": "bob@test.com", "age": 30}
        
        with pytest.raises(DatabaseError) as exc_info:
            user_service.create_user(user_data)
        
        assert "insert user" in exc_info.value.message
        assert exc_info.value.cause is not None
    
    # ==========================================================
    # ТЕСТЫ process_payment
    # ==========================================================
    
    def test_process_payment_success(self, user_service, mock_payment):
        """Успешная обработка платежа."""
        result = user_service.process_payment(user_id=1, amount=500)
        
        assert result["success"] is True
        assert result["transaction_id"] == "tx_456"
        mock_payment.charge.assert_called_once_with("token123", 500)
    
    def test_process_payment_insufficient_funds(self, user_service, mock_db):
        """Недостаточно средств -> InsufficientFundsError."""
        mock_db.get_user.return_value = {"id": 1, "balance": 100, "card_token": "token"}
        
        with pytest.raises(InsufficientFundsError) as exc_info:
            user_service.process_payment(1, 500)
        
        assert exc_info.value.balance == 100
        assert exc_info.value.requested == 500
        assert exc_info.value.details["shortage"] == 400
    
    def test_process_payment_api_error(self, user_service, mock_payment):
        """Ошибка API -> APIClientError."""
        mock_payment.charge.side_effect = Exception("Payment gateway timeout")
        
        with pytest.raises(APIClientError) as exc_info:
            user_service.process_payment(1, 500)
        
        assert exc_info.value.endpoint == "/charge"
        assert exc_info.value.status_code == 500
Задание 3.2. Тестирование цепочки исключений и контекста (10 минут)
python
class TestExceptionChaining:
    """Тестирование цепочек и контекста исключений."""
    
    def test_exception_chaining_with_cause(self):
        """Проверка связи исключений через cause."""
        try:
            try:
                raise ValueError("Original error")
            except ValueError as e:
                raise DatabaseError("query", "failed", cause=e)
        except DatabaseError as e:
            assert e.cause is not None
            assert isinstance(e.cause, ValueError)
            assert str(e.cause) == "Original error"
    
    def test_exception_chain_preserves_details(self):
        """Цепочка сохраняет детали исходной ошибки."""
        original = ValueError("Connection timeout")
        error = DatabaseError("select", "DB unreachable", cause=original)
        
        assert error.cause is original
        assert error.details["operation"] == "select"
    
    def test_to_dict_does_not_include_cause(self):
        """to_dict не должен включать cause для безопасности."""
        error = DatabaseError("query", "failed", cause=ValueError("secret"))
        error_dict = error.to_dict()
        
        assert "cause" not in error_dict
        assert "secret" not in str(error_dict)
Задание 3.3. Тестирование комплексных сценариев (10 минут)
python
class TestComplexScenarios:
    """Комплексные сценарии с исключениями."""
    
    @pytest.fixture
    def mock_db(self):
        db = Mock()
        db.get_user.return_value = {"id": 1, "balance": 1000}
        return db
    
    @pytest.fixture
    def mock_payment(self):
        payment = Mock()
        return payment
    
    @pytest.fixture
    def user_service(self, mock_db, mock_payment):
        return UserService(db_client=mock_db, payment_gateway=mock_payment)
    
    def test_balance_check_before_payment(self, user_service, mock_db):
        """Проверка баланса перед платежом."""
        # Сценарий: запрос на сумму больше баланса
        mock_db.get_user.return_value = {"id": 1, "balance": 100}
        
        with pytest.raises(InsufficientFundsError) as exc_info:
            user_service.process_payment(1, 500)
        
        # Проверяем, что платёж не был вызван
        user_service.payment_gateway.charge.assert_not_called()
    
    def test_full_user_creation_flow_with_validation(self, user_service, mock_db):
        """Полный сценарий создания пользователя с валидацией."""
        # Валидный пользователь
        valid_user = {"name": "Valid", "email": "valid@test.com", "age": 25}
        user_id = user_service.create_user(valid_user)
        assert user_id == 123
        
        # Невалидный email
        invalid_user = {"name": "Bad", "email": "bad", "age": 30}
        with pytest.raises(InvalidFormatError):
            user_service.create_user(invalid_user)
        
        # Невалидный возраст
        underage = {"name": "Young", "email": "young@test.com", "age": 16}
        with pytest.raises(RangeError):
            user_service.create_user(underage)
        
        # Ошибка БД
        mock_db.insert.side_effect = Exception("DB error")
        with pytest.raises(DatabaseError):
            user_service.create_user(valid_user)
    
    def test_error_recovery_flow(self, user_service, mock_payment):
        """Сценарий восстановления после ошибки."""
        # Первый вызов — ошибка API
        mock_payment.charge.side_effect = ConnectionError("payment", "timeout")
        
        with pytest.raises(ConnectionError):
            user_service.process_payment(1, 500)
        
        # Второй вызов — успех (после восстановления)
        mock_payment.charge.side_effect = None
        mock_payment.charge.return_value = {"id": "tx_789"}
        
        result = user_service.process_payment(1, 500)
        assert result["success"] is True
        assert result["transaction_id"] == "tx_789"
Задание 3.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (сложный уровень)

**Студент:** _________________
**Вариант №:** ___ (18-25)

### Результаты тестирования

| Компонент | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| Базовые исключения | _____ | _____ | _____ |
| Иерархия исключений | _____ | _____ | _____ |
| Полиморфная обработка | _____ | _____ | _____ |
| UserService валидация | _____ | _____ | _____ |
| UserService платежи | _____ | _____ | _____ |
| Цепочки исключений | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Проверенные исключения

| Исключение | Создание | Атрибуты | Иерархия | Обработка |
|:---|:---|:---|:---|:---|
| RequiredFieldError | ☐ | ☐ | ☐ | ☐ |
| InvalidFormatError | ☐ | ☐ | ☐ | ☐ |
| RangeError | ☐ | ☐ | ☐ | ☐ |
| InsufficientFundsError | ☐ | ☐ | ☐ | ☐ |
| ProductOutOfStockError | ☐ | ☐ | ☐ | ☐ |
| OrderStateError | ☐ | ☐ | ☐ | ☐ |
| DatabaseError | ☐ | ☐ | ☐ | ☐ |
| APIClientError | ☐ | ☐ | ☐ | ☐ |
| ConnectionError | ☐ | ☐ | ☐ | ☐ |

### Выводы

_______________________________________________________________

_______________________________________________________________
Карточка студента
text
ПР 3.7. ТЕСТИРОВАНИЕ ПОЛЬЗОВАТЕЛЬСКИХ ИСКЛЮЧЕНИЙ

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ Базовые исключения (базовый)
□ Атрибуты исключений (базовый)
□ to_dict() метод (базовый)
□ Иерархия исключений (средний)
□ Полиморфная обработка (средний)
□ pytest.raises с match (средний)
□ UserService тесты (сложный)
□ Цепочки исключений (сложный)
□ Комплексные сценарии (сложный)

=== ПРОВЕРЕННЫЕ АСПЕКТЫ ===

□ Тип исключения
□ Сообщение об ошибке
□ Атрибуты (field_name, код, детали)
□ Иерархия (isinstance)
□ Полиморфный перехват
□ Причина (cause)

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
- test_exceptions_base.py
- test_exceptions_intermediate.py
- test_exceptions_advanced.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Исключения не тестируются
3 (удовлетворительно)	Базовый	Отдельные исключения + атрибуты (10+ тестов)
4 (хорошо)	Средний	+ Иерархия + полиморфизм (20+ тестов)
5 (отлично)	Сложный	+ UserService + цепочки + сценарии (30+ тестов)
Контрольные вопросы (для защиты)
Как проверить атрибуты пользовательского исключения?

Как проверить, что исключение принадлежит определённой иерархии?

Как с помощью pytest.raises проверить сообщение об ошибке?

В чём разница между __cause__ и __context__?

Как протестировать полиморфную обработку исключений?

Как проверить, что при ошибке валидации сохраняются все детали?

Как протестировать, что исключение содержит корректный код ошибки?

Почему to_dict() не должен включать __cause__?

Как протестировать сквозной сценарий с ошибкой в середине?

Как проверить, что при ошибке БД сохраняется исходное исключение?

Следующее занятие: ПР 3.8 — Параметризация и мокирование.
