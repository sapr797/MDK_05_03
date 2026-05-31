# Лекция 3.7: Кастомные исключения. Иерархия исключений. Тестирование обработки ошибок

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить создание иерархий кастомных исключений, освоить методы тестирования обработки ошибок, научиться проектировать устойчивые к ошибкам системы.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Создавать иерархии пользовательских исключений (ПК 3.2).
2. Проектировать обработку ошибок с использованием кастомных исключений (ПК 3.3).
3. Тестировать обработку ошибок в различных сценариях (ПК 3.2, ПК 3.3).
4. Анализировать и улучшать обработку ошибок в приложениях (ОК 02, ОК 05).

---

## 1. Зачем нужны кастомные исключения?

### 1.1 Проблемы стандартных исключений
┌─────────────────────────────────────────────────────────────────────────────┐
│ ПРОБЛЕМЫ СТАНДАРТНЫХ ИСКЛЮЧЕНИЙ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ 1. НЕДОСТАТОК ИНФОРМАЦИИ │
│ └── ValueError не говорит, какое именно значение неверно │
│ │
│ 2. НЕТ ВОЗМОЖНОСТИ КАТЕГОРИЗАЦИИ │
│ └── Все ошибки одного типа сложно обрабатывать по-разному │
│ │
│ 3. СЛОЖНОСТЬ ОТЛАДКИ │
│ └── Непонятно, на каком уровне произошла ошибка │
│ │
│ 4. НАРУШЕНИЕ ИНКАПСУЛЯЦИИ │
│ └── Клиентский код зависит от деталей реализации │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 1.2 Преимущества кастомных исключений

```python
# ❌ ПЛОХО: непонятно, что пошло не так
def transfer_money_bad(amount: float):
    if amount <= 0:
        raise ValueError("Invalid amount")
    if amount > balance:
        raise ValueError("Insufficient funds")
    raise ValueError("Transfer failed")

# ✅ ХОРОШО: каждое исключение несёт смысловую нагрузку
def transfer_money_good(amount: float):
    if amount <= 0:
        raise InvalidAmountError(amount)
    if amount > balance:
        raise InsufficientFundsError(balance, amount)
    raise TransferError("Connection failed", original_exception)
2. Создание иерархии кастомных исключений
2.1 Базовые принципы
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ИЕРАРХИЯ КАСТОМНЫХ ИСКЛЮЧЕНИЙ                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                        BaseError                                            │
│                           △                                                 │
│                           │                                                 │
│              ┌────────────┼────────────┐                                    │
│              │            │            │                                    │
│         ClientError   ServerError  ValidationError                          │
│              △            △              △                                   │
│              │            │              │                                   │
│      ┌───────┼───────┐    │         ┌────┼────┐                             │
│      │       │       │    │         │    │    │                             │
│   AuthError  │   NotFound  │   Required │   Format                          │
│              │             │      Field  │   Error                          │
│         PermissionError    │         Error                                   │
│                            │                                                 │
│                       ConnectionError                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
2.2 Реализация иерархии исключений
python
# exceptions.py

from typing import Optional, Any, Dict
from datetime import datetime


# ==========================================================
# БАЗОВЫЕ ИСКЛЮЧЕНИЯ
# ==========================================================

class AppException(Exception):
    """Базовое исключение приложения."""
    
    def __init__(self, message: str, code: int = None, details: Dict = None):
        self.message = message
        self.code = code
        self.details = details or {}
        self.timestamp = datetime.now()
        super().__init__(self.message)
    
    def to_dict(self) -> Dict:
        """Преобразует исключение в словарь для логирования."""
        return {
            "type": self.__class__.__name__,
            "message": self.message,
            "code": self.code,
            "details": self.details,
            "timestamp": self.timestamp.isoformat()
        }


class ClientError(AppException):
    """Ошибка на стороне клиента (4xx)."""
    pass


class ServerError(AppException):
    """Ошибка на стороне сервера (5xx)."""
    pass


# ==========================================================
# ОШИБКИ КЛИЕНТА (4xx)
# ==========================================================

class ValidationError(ClientError):
    """Ошибка валидации входных данных (400)."""
    
    def __init__(self, field: str, message: str, value: Any = None):
        self.field = field
        self.value = value
        super().__init__(
            message=f"Validation error in field '{field}': {message}",
            code=400,
            details={"field": field, "value": str(value) if value else None}
        )


class AuthenticationError(ClientError):
    """Ошибка аутентификации (401)."""
    pass


class PermissionError(ClientError):
    """Ошибка доступа (403)."""
    pass


class NotFoundError(ClientError):
    """Ресурс не найден (404)."""
    
    def __init__(self, resource_type: str, resource_id: Any):
        self.resource_type = resource_type
        self.resource_id = resource_id
        super().__init__(
            message=f"{resource_type} with id '{resource_id}' not found",
            code=404,
            details={"resource_type": resource_type, "resource_id": str(resource_id)}
        )


class ConflictError(ClientError):
    """Конфликт данных (409)."""
    pass


class RateLimitError(ClientError):
    """Превышение лимита запросов (429)."""
    
    def __init__(self, retry_after: int):
        self.retry_after = retry_after
        super().__init__(
            message=f"Rate limit exceeded. Retry after {retry_after} seconds",
            code=429,
            details={"retry_after": retry_after}
        )


# ==========================================================
# ОШИБКИ СЕРВЕРА (5xx)
# ==========================================================

class DatabaseError(ServerError):
    """Ошибка базы данных (500)."""
    pass


class ConnectionError(ServerError):
    """Ошибка соединения (502/503)."""
    pass


class TimeoutError(ServerError):
    """Таймаут операции (504)."""
    pass


class ExternalAPIError(ServerError):
    """Ошибка внешнего API (502/503)."""
    
    def __init__(self, service: str, status_code: int, response: str):
        self.service = service
        self.status_code = status_code
        self.response = response
        super().__init__(
            message=f"External API '{service}' returned error {status_code}",
            code=status_code,
            details={"service": service, "response": response[:200]}
        )


# ==========================================================
# БИЗНЕС-ЛОГИКА
# ==========================================================

class BusinessError(AppException):
    """Ошибка бизнес-логики."""
    pass


class InsufficientFundsError(BusinessError):
    """Недостаточно средств."""
    
    def __init__(self, balance: float, requested: float):
        self.balance = balance
        self.requested = requested
        super().__init__(
            message=f"Insufficient funds: requested {requested}, available {balance}",
            code=422,
            details={"balance": balance, "requested": requested}
        )


class ProductOutOfStockError(BusinessError):
    """Товар отсутствует на складе."""
    
    def __init__(self, product_id: int, available: int, requested: int):
        self.product_id = product_id
        self.available = available
        self.requested = requested
        super().__init__(
            message=f"Product {product_id} out of stock: requested {requested}, available {available}",
            code=422,
            details={"product_id": product_id, "available": available, "requested": requested}
        )


class OrderAlreadyCancelledError(BusinessError):
    """Заказ уже отменён."""
    pass


class OrderAlreadyConfirmedError(BusinessError):
    """Заказ уже подтверждён."""
    pass
3. Тестирование кастомных исключений
3.1 Базовое тестирование
python
# test_exceptions.py

import pytest
from exceptions import (
    AppException, ValidationError, NotFoundError, 
    RateLimitError, InsufficientFundsError, ExternalAPIError
)


class TestCustomExceptions:
    """Тесты кастомных исключений."""
    
    def test_validation_error_attributes(self):
        """Проверка атрибутов ValidationError."""
        
        with pytest.raises(ValidationError) as exc_info:
            raise ValidationError("email", "Invalid email format", "not-an-email")
        
        assert exc_info.value.field == "email"
        assert exc_info.value.value == "not-an-email"
        assert exc_info.value.code == 400
        assert "Validation error in field 'email'" in str(exc_info.value)
        assert exc_info.value.details["field"] == "email"
    
    def test_not_found_error(self):
        """Проверка NotFoundError."""
        
        with pytest.raises(NotFoundError) as exc_info:
            raise NotFoundError("User", 123)
        
        assert exc_info.value.resource_type == "User"
        assert exc_info.value.resource_id == 123
        assert exc_info.value.code == 404
        assert "User with id '123' not found" in str(exc_info.value)
    
    def test_rate_limit_error(self):
        """Проверка RateLimitError."""
        
        with pytest.raises(RateLimitError) as exc_info:
            raise RateLimitError(30)
        
        assert exc_info.value.retry_after == 30
        assert exc_info.value.code == 429
        assert "Rate limit exceeded. Retry after 30 seconds" in str(exc_info.value)
        assert exc_info.value.details["retry_after"] == 30
    
    def test_insufficient_funds_error(self):
        """Проверка InsufficientFundsError."""
        
        with pytest.raises(InsufficientFundsError) as exc_info:
            raise InsufficientFundsError(100.0, 150.0)
        
        assert exc_info.value.balance == 100.0
        assert exc_info.value.requested == 150.0
        assert exc_info.value.code == 422
        assert "insufficient funds" in str(exc_info.value).lower()
        assert exc_info.value.details["balance"] == 100.0
        assert exc_info.value.details["requested"] == 150.0
    
    def test_external_api_error(self):
        """Проверка ExternalAPIError."""
        
        with pytest.raises(ExternalAPIError) as exc_info:
            raise ExternalAPIError("PaymentGateway", 503, "Service temporarily unavailable")
        
        assert exc_info.value.service == "PaymentGateway"
        assert exc_info.value.status_code == 503
        assert "External API 'PaymentGateway' returned error 503" in str(exc_info.value)
        assert exc_info.value.details["response"] == "Service temporarily unavailable"
4. Тестирование обработки ошибок
4.1 Пример сервиса с обработкой ошибок
python
# payment_service.py

import requests
import logging
from typing import Dict, Optional
from exceptions import (
    ValidationError, NotFoundError, InsufficientFundsError,
    ExternalAPIError, DatabaseError, RateLimitError
)


class PaymentService:
    """Сервис для обработки платежей."""
    
    def __init__(self, api_client, db_session):
        self.api_client = api_client
        self.db = db_session
        self.logger = logging.getLogger(__name__)
    
    def process_payment(self, user_id: int, amount: float) -> Dict:
        """
        Обрабатывает платёж пользователя.
        
        Raises:
            ValidationError: Неверная сумма
            NotFoundError: Пользователь не найден
            InsufficientFundsError: Недостаточно средств
            ExternalAPIError: Ошибка внешнего API
            DatabaseError: Ошибка базы данных
        """
        # Валидация суммы
        if amount <= 0:
            raise ValidationError("amount", "Amount must be positive", amount)
        
        # Проверка пользователя
        user = self.db.get_user(user_id)
        if not user:
            raise NotFoundError("User", user_id)
        
        # Проверка баланса
        if user.balance < amount:
            raise InsufficientFundsError(user.balance, amount)
        
        try:
            # Вызов внешнего API
            result = self.api_client.charge(user.card_token, amount)
            
            if result.status_code == 429:
                raise RateLimitError(result.headers.get("Retry-After", 60))
            
            if result.status_code >= 500:
                raise ExternalAPIError(
                    "PaymentGateway",
                    result.status_code,
                    result.text[:200]
                )
            
            # Обновление баланса в БД
            user.balance -= amount
            self.db.save_user(user)
            
            return {"success": True, "transaction_id": result.json()["id"]}
            
        except requests.RequestException as e:
            self.logger.error(f"API error: {e}")
            raise DatabaseError(f"Failed to process payment: {e}")
4.2 Тестирование обработки ошибок
python
# test_payment_service.py

import pytest
from unittest.mock import Mock, patch
from payment_service import PaymentService
from exceptions import (
    ValidationError, NotFoundError, InsufficientFundsError,
    ExternalAPIError, DatabaseError, RateLimitError
)


class TestPaymentServiceErrorHandling:
    """Тестирование обработки ошибок в PaymentService."""
    
    @pytest.fixture
    def mock_api_client(self):
        return Mock()
    
    @pytest.fixture
    def mock_db(self):
        db = Mock()
        db.get_user.return_value = Mock(balance=1000, card_token="token123")
        return db
    
    @pytest.fixture
    def service(self, mock_api_client, mock_db):
        return PaymentService(mock_api_client, mock_db)
    
    def test_validation_error_on_negative_amount(self, service):
        """Негативная сумма -> ValidationError."""
        
        with pytest.raises(ValidationError) as exc_info:
            service.process_payment(1, -100)
        
        assert exc_info.value.field == "amount"
        assert exc_info.value.value == -100
        assert exc_info.value.code == 400
    
    def test_not_found_error_for_nonexistent_user(self, service, mock_db):
        """Пользователь не найден -> NotFoundError."""
        mock_db.get_user.return_value = None
        
        with pytest.raises(NotFoundError) as exc_info:
            service.process_payment(999, 100)
        
        assert exc_info.value.resource_type == "User"
        assert exc_info.value.resource_id == 999
        assert exc_info.value.code == 404
    
    def test_insufficient_funds_error(self, service, mock_db):
        """Недостаточно средств -> InsufficientFundsError."""
        mock_db.get_user.return_value.balance = 50
        
        with pytest.raises(InsufficientFundsError) as exc_info:
            service.process_payment(1, 100)
        
        assert exc_info.value.balance == 50
        assert exc_info.value.requested == 100
        assert exc_info.value.code == 422
    
    def test_external_api_error(self, service, mock_api_client):
        """Ошибка внешнего API -> ExternalAPIError."""
        mock_response = Mock()
        mock_response.status_code = 503
        mock_response.text = "Service unavailable"
        mock_api_client.charge.return_value = mock_response
        
        with pytest.raises(ExternalAPIError) as exc_info:
            service.process_payment(1, 100)
        
        assert exc_info.value.service == "PaymentGateway"
        assert exc_info.value.status_code == 503
        assert "Service unavailable" in exc_info.value.details["response"]
    
    def test_rate_limit_error(self, service, mock_api_client):
        """Rate limit -> RateLimitError."""
        mock_response = Mock()
        mock_response.status_code = 429
        mock_response.headers = {"Retry-After": "30"}
        mock_api_client.charge.return_value = mock_response
        
        with pytest.raises(RateLimitError) as exc_info:
            service.process_payment(1, 100)
        
        assert exc_info.value.retry_after == 30
        assert exc_info.value.code == 429
    
    def test_database_error_on_request_exception(self, service, mock_api_client):
        """RequestException -> DatabaseError."""
        mock_api_client.charge.side_effect = requests.RequestException("Connection failed")
        
        with pytest.raises(DatabaseError) as exc_info:
            service.process_payment(1, 100)
        
        assert "Failed to process payment" in str(exc_info.value)
5. Паттерны обработки ошибок
5.1 Иерархический перехват
python
def test_hierarchical_error_handling(service):
    """Перехват ошибок на разных уровнях иерархии."""
    
    # Перехват конкретного исключения
    try:
        service.process_payment(999, 100)
    except NotFoundError as e:
        assert e.code == 404
    
    # Перехват всех ошибок клиента
    try:
        service.process_payment(1, -100)
    except ClientError as e:
        assert e.code in (400, 401, 403, 404, 422, 429)
    
    # Перехват всех ошибок приложения
    try:
        service.process_payment(1, 1000000)  # недостаточно средств
    except AppException as e:
        assert e.code is not None
        assert e.timestamp is not None
5.2 Преобразование исключений
python
def exception_to_response(e: AppException) -> dict:
    """Преобразует исключение в HTTP-ответ."""
    return {
        "error": {
            "type": e.__class__.__name__,
            "message": e.message,
            "code": e.code,
            "details": e.details,
            "timestamp": e.timestamp.isoformat()
        }
    }


def test_exception_to_response():
    """Тест преобразования исключения в ответ."""
    
    error = ValidationError("email", "Invalid format", "test")
    response = exception_to_response(error)
    
    assert response["error"]["type"] == "ValidationError"
    assert response["error"]["code"] == 400
    assert response["error"]["details"]["field"] == "email"
6. Шпаргалка
python
# === СОЗДАНИЕ КАСТОМНОГО ИСКЛЮЧЕНИЯ ===
class MyError(Exception):
    def __init__(self, message, code):
        self.message = message
        self.code = code
        super().__init__(message)

# === ТЕСТИРОВАНИЕ АТРИБУТОВ ===
def test_error():
    with pytest.raises(MyError) as e:
        raise MyError("Error", 100)
    assert e.value.code == 100

# === ИЕРАРХИЯ ИСКЛЮЧЕНИЙ ===
class BaseError(Exception): pass
class ClientError(BaseError): pass
class ValidationError(ClientError): pass

# === ТЕСТИРОВАНИЕ ОБРАБОТКИ ОШИБОК ===
def test_error_handling():
    try:
        risky_operation()
    except ValidationError as e:
        assert e.code == 400
    except ClientError as e:
        assert e.code in (401, 403, 404)
    except BaseError as e:
        assert isinstance(e, BaseError)
Контрольные вопросы
Зачем нужны кастомные исключения вместо стандартных?

Как правильно спроектировать иерархию исключений?

Как протестировать атрибуты кастомного исключения?

Как проверить, что обработчик ошибок выбрасывает правильный тип исключения?

В чём преимущество иерархического перехвата исключений?

Как тестировать преобразование исключений в HTTP-ответы?

Почему важно включать контекстную информацию в исключения?

Как тестировать цепочки исключений (__cause__)?

Как обеспечить, чтобы обработка ошибок не раскрывала внутренние детали?

Какая информация должна быть в кастомном исключении для эффективной отладки?

Следующее занятие: ПР 3.7 — Практическое создание и тестирование кастомных исключений.
