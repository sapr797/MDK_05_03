# Лекция 3.6: Углублённое тестирование исключений. Pytest.raises, match, проверка атрибутов

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить углублённые методы тестирования исключений в pytest: использование `pytest.raises` как контекстного менеджера, проверку сообщений об ошибках с помощью `match`, тестирование атрибутов пользовательских исключений.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Использовать `pytest.raises` для проверки типов исключений (ПК 3.2, ПК 3.3).
2. Проверять сообщения об ошибках с помощью параметра `match` (ПК 3.2).
3. Тестировать атрибуты пользовательских исключений (ПК 3.3).
4. Применять продвинутые техники тестирования исключений (ОК 02, ОК 05).

---

## 1. Основы pytest.raises

### 1.1 Базовый синтаксис

```python
import pytest

def test_basic_exception():
    with pytest.raises(ZeroDivisionError):
        10 / 0
1.2 Получение объекта исключения
python
def test_exception_with_info():
    with pytest.raises(ValueError) as exc_info:
        int("not a number")
    
    # Проверка типа исключения
    assert exc_info.type == ValueError
    
    # Проверка сообщения
    assert "invalid literal" in str(exc_info.value)
    
    # Проверка атрибутов
    assert exc_info.value.args[0] is not None
1.3 Структура ExceptionInfo
Атрибут	Описание	Пример
exc_info.type	Тип исключения	ValueError
exc_info.value	Экземпляр исключения	ValueError("invalid literal")
exc_info.traceback	Объект traceback	Информация о стеке
2. Проверка сообщений с параметром match
2.1 Синтаксис match
python
def test_match_simple():
    with pytest.raises(ValueError, match="invalid literal"):
        int("not a number")
2.2 Использование регулярных выражений
python
def test_match_regex():
    # Точное совпадение
    with pytest.raises(ValueError, match="Age must be positive"):
        raise ValueError("Age must be positive")
    
    # Частичное совпадение
    with pytest.raises(ValueError, match="positive"):
        raise ValueError("Age must be positive")
    
    # Регулярное выражение
    with pytest.raises(ValueError, match=r"Age.*positive"):
        raise ValueError("Age must be positive")
    
    # Специальные символы
    with pytest.raises(ValueError, match=r"\d+ is not valid"):
        raise ValueError("123 is not valid")
2.3 Сравнение подходов
python
def test_compare_match_approaches():
    # Способ 1: match (рекомендуется для простых проверок)
    with pytest.raises(ValueError, match="Invalid email"):
        validate_email("invalid")
    
    # Способ 2: exc_info (для сложных проверок)
    with pytest.raises(ValueError) as exc_info:
        validate_email("invalid")
    
    assert "Invalid email" in str(exc_info.value)
    assert exc_info.value.code == 1001  # проверка атрибута
3. Проверка атрибутов пользовательских исключений
3.1 Создание пользовательского исключения
python
class ValidationError(Exception):
    """Пользовательское исключение для ошибок валидации."""
    
    def __init__(self, field: str, message: str, code: int):
        self.field = field
        self.message = message
        self.code = code
        super().__init__(f"{field}: {message} (code {code})")


class APIError(Exception):
    """Исключение для ошибок API."""
    
    def __init__(self, status_code: int, message: str, details: dict = None):
        self.status_code = status_code
        self.message = message
        self.details = details or {}
        super().__init__(f"API Error {status_code}: {message}")


class RateLimitError(APIError):
    """Превышение лимита запросов."""
    
    def __init__(self, retry_after: int):
        super().__init__(429, "Rate limit exceeded", {"retry_after": retry_after})
        self.retry_after = retry_after
3.2 Тестирование атрибутов исключений
python
def test_validation_error_attributes():
    """Проверка атрибутов ValidationError."""
    
    with pytest.raises(ValidationError) as exc_info:
        raise ValidationError("email", "Invalid email format", 1001)
    
    # Проверка атрибутов
    assert exc_info.value.field == "email"
    assert exc_info.value.message == "Invalid email format"
    assert exc_info.value.code == 1001
    
    # Проверка сообщения
    assert "email: Invalid email format (code 1001)" in str(exc_info.value)


def test_api_error_attributes():
    """Проверка атрибутов APIError."""
    
    with pytest.raises(APIError) as exc_info:
        raise APIError(404, "User not found", {"user_id": 123})
    
    assert exc_info.value.status_code == 404
    assert exc_info.value.message == "User not found"
    assert exc_info.value.details["user_id"] == 123
    assert "API Error 404: User not found" in str(exc_info.value)


def test_rate_limit_error():
    """Проверка RateLimitError."""
    
    with pytest.raises(RateLimitError) as exc_info:
        raise RateLimitError(30)
    
    assert exc_info.value.status_code == 429
    assert exc_info.value.retry_after == 30
    assert "Rate limit exceeded" in exc_info.value.message
    assert exc_info.value.details["retry_after"] == 30
4. Тестирование цепочек исключений
4.1 Исключения с причиной (cause)
python
def process_data(data: dict) -> dict:
    """Обрабатывает данные с вложенным исключением."""
    try:
        result = int(data["value"])
        return {"result": result}
    except (KeyError, ValueError) as e:
        raise ProcessingError("Failed to process data") from e


class ProcessingError(Exception):
    pass


def test_chained_exceptions():
    """Проверка цепочки исключений."""
    
    with pytest.raises(ProcessingError) as exc_info:
        process_data({})
    
    # Проверяем причину
    assert exc_info.value.__cause__ is not None
    assert isinstance(exc_info.value.__cause__, KeyError)
    assert "'value'" in str(exc_info.value.__cause__)


def test_chained_exception_with_match():
    """Проверка цепочки с регулярным выражением."""
    
    with pytest.raises(ProcessingError, match="Failed to process data") as exc_info:
        process_data({"value": "not a number"})
    
    # Проверяем вложенную причину
    cause = exc_info.value.__cause__
    assert isinstance(cause, ValueError)
    assert "invalid literal" in str(cause)
4.2 Исключения с контекстом (context)
python
def test_exception_context():
    """Проверка контекста исключения."""
    
    try:
        try:
            raise ValueError("Original error")
        except ValueError:
            raise RuntimeError("Wrapper error")
    except RuntimeError as e:
        assert e.__context__ is not None
        assert isinstance(e.__context__, ValueError)
5. Проверка нескольких исключений
5.1 Проверка одного из нескольких типов
python
def test_multiple_exception_types():
    """Проверка, что выбрасывается одно из нескольких исключений."""
    
    # Допустимы ValueError или TypeError
    with pytest.raises((ValueError, TypeError)):
        int("not a number")
    
    # Только TypeError
    with pytest.raises(TypeError):
        int(None)
5.2 Проверка порядка исключений
python
def test_exception_order():
    """Проверка, что исключения выбрасываются в правильном порядке."""
    
    def multi_error_function(step: int):
        if step == 1:
            raise ValueError("First error")
        elif step == 2:
            raise TypeError("Second error")
        else:
            return "success"
    
    # Проверяем первое исключение
    with pytest.raises(ValueError, match="First error"):
        multi_error_function(1)
    
    # Проверяем второе исключение
    with pytest.raises(TypeError, match="Second error"):
        multi_error_function(2)
    
    # Проверяем успешный случай
    assert multi_error_function(3) == "success"
6. Тестирование отсутствия исключений
6.1 Проверка, что исключение не выбрасывается
python
def test_no_exception():
    """Проверка, что исключение не выбрасывается."""
    
    # Просто вызываем функцию без with pytest.raises
    result = int("123")
    assert result == 123


def test_no_exception_with_try():
    """Альтернативный способ проверки."""
    
    try:
        result = int("456")
        assert result == 456
    except Exception:
        pytest.fail("Exception was raised but not expected")
7. Практические примеры
7.1 Тестирование функции с несколькими исключениями
python
def validate_user(name: str, age: int, email: str) -> bool:
    """
    Валидация пользователя.
    
    Raises:
        ValueError: При неверных параметрах с разными кодами
    """
    if not name or len(name) < 2:
        raise ValidationError("name", "Name must be at least 2 characters", 1001)
    
    if not isinstance(age, int) or age < 0 or age > 150:
        raise ValidationError("age", "Age must be between 0 and 150", 1002)
    
    if "@" not in email or "." not in email:
        raise ValidationError("email", "Invalid email format", 1003)
    
    return True


def test_validate_user_all_errors():
    """Тестирование всех возможных ошибок валидации."""
    
    # Ошибка имени
    with pytest.raises(ValidationError) as exc_info:
        validate_user("", 25, "test@test.com")
    assert exc_info.value.field == "name"
    assert exc_info.value.code == 1001
    
    # Ошибка возраста
    with pytest.raises(ValidationError) as exc_info:
        validate_user("John", -5, "test@test.com")
    assert exc_info.value.field == "age"
    assert exc_info.value.code == 1002
    
    # Ошибка email
    with pytest.raises(ValidationError) as exc_info:
        validate_user("John", 25, "invalid")
    assert exc_info.value.field == "email"
    assert exc_info.value.code == 1003
    
    # Успешная валидация
    assert validate_user("John", 25, "john@test.com") is True
7.2 Параметризация тестов исключений
python
@pytest.mark.parametrize("name,age,email,expected_field,expected_code", [
    ("", 25, "test@test.com", "name", 1001),
    ("A", 25, "test@test.com", "name", 1001),  # слишком короткое имя
    ("John", -1, "test@test.com", "age", 1002),
    ("John", 200, "test@test.com", "age", 1002),
    ("John", 25, "invalid", "email", 1003),
    ("John", 25, "no-at.com", "email", 1003),
])
def test_validate_user_parametrized(name, age, email, expected_field, expected_code):
    """Параметризованный тест ошибок валидации."""
    
    with pytest.raises(ValidationError) as exc_info:
        validate_user(name, age, email)
    
    assert exc_info.value.field == expected_field
    assert exc_info.value.code == expected_code
8. Шпаргалка
python
# === БАЗОВАЯ ПРОВЕРКА ===
with pytest.raises(ZeroDivisionError):
    10 / 0

# === ПРОВЕРКА С СООБЩЕНИЕМ ===
with pytest.raises(ValueError, match="invalid literal"):
    int("abc")

# === ПОЛУЧЕНИЕ ОБЪЕКТА ИСКЛЮЧЕНИЯ ===
with pytest.raises(ValueError) as exc_info:
    int("abc")
assert "invalid" in str(exc_info.value)

# === ПРОВЕРКА АТРИБУТОВ ===
with pytest.raises(ValidationError) as exc_info:
    validate("")
assert exc_info.value.field == "name"
assert exc_info.value.code == 1001

# === ПРОВЕРКА НЕСКОЛЬКИХ ТИПОВ ===
with pytest.raises((ValueError, TypeError)):
    int(None)

# === ПРОВЕРКА ЦЕПОЧКИ ===
with pytest.raises(RuntimeError) as exc_info:
    process_data()
assert exc_info.value.__cause__ is not None

# === ПАРАМЕТРИЗАЦИЯ ===
@pytest.mark.parametrize("input,expected", [
    ("abc", "invalid"),
    ("", "empty"),
])
def test_parametrized(input, expected):
    with pytest.raises(ValueError, match=expected):
        validate(input)
Контрольные вопросы
Что возвращает pytest.raises() как контекстный менеджер?

Как проверить сообщение об ошибке с помощью параметра match?

Как проверить атрибуты пользовательского исключения?

В чём разница между match="text" и assert "text" in str(exc_info.value)?

Как проверить, что исключение было вызвано другим исключением (__cause__)?

Как проверить, что функция выбрасывает одно из нескольких исключений?

Как проверить, что исключение НЕ было выброшено?

Как параметризовать тесты для проверки разных исключений?

Что такое exc_info.type и exc_info.value?

Как проверить, что исключение содержит определённый атрибут с определённым значением?

Следующее занятие: ПР 3.6 — Практическое углублённое тестирование исключений.
