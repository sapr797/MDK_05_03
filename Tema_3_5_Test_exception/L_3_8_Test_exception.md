# Лекция 3.8: Тестирование исключений. pytest.raises() как контекстный менеджер

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.5:** Тестирование исключений и коллекций  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить методы тестирования исключений в pytest, освоить работу с `pytest.raises()` как контекстным менеджером, научиться проверять типы исключений, сообщения об ошибках и атрибуты, а также разрабатывать негативные сценарии тестирования.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Использовать `pytest.raises()` для проверки возникновения исключений (ПК 3.2).
2. Проверять тип исключения, сообщение об ошибке и атрибуты (ПК 3.2, ПК 3.3).
3. Разрабатывать негативные тестовые сценарии (ОК 02, ОК 05).
4. Тестировать граничные случаи, приводящие к исключениям (ПК 3.3).

---

## 1. Почему нужно тестировать исключения?

### 1.1 Значение тестирования исключений

Исключения — это **не ошибка в коде**, а **корректное поведение** системы в ответ на некорректные входные данные или состояние.

| Сценарий | Исключение | Почему это нормально |
|:---|:---|:---|
| Деление на ноль | `ZeroDivisionError` | Математически не определено |
| Открытие несуществующего файла | `FileNotFoundError` | Файла нет — это факт |
| Ввод отрицательного возраста | `ValueError` | Возраст не может быть отрицательным |
| Доступ без авторизации | `PermissionError` | Пользователь не имеет прав |

### 1.2 Что нужно проверять при тестировании исключений
┌─────────────────────────────────────────────────────────────────────────────┐
│ ЧТО ПРОВЕРЯТЬ В ИСКЛЮЧЕНИИ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ 1. ТИП ИСКЛЮЧЕНИЯ │
│ └── Правильный ли класс исключения? (ValueError, TypeError и т.д.) │
│ │
│ 2. СООБЩЕНИЕ ОБ ОШИБКЕ │
│ └── Понятное ли сообщение для пользователя? │
│ └── Содержит ли нужную информацию? │
│ │
│ 3. АТРИБУТЫ ИСКЛЮЧЕНИЯ │
│ └── Есть ли дополнительные поля (код ошибки, детали)? │
│ │
│ 4. ПОБОЧНЫЕ ЭФФЕКТЫ │
│ └── Изменилось ли состояние системы после исключения? │
│ └── Данные не были частично сохранены? │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

---

## 2. pytest.raises() как контекстный менеджер

### 2.1 Базовый синтаксис

```python
import pytest

def test_division_by_zero():
    with pytest.raises(ZeroDivisionError):
        result = 10 / 0
2.2 Схема работы контекстного менеджера
python
with pytest.raises(ExpectedException) as exc_info:
    # Код, который ДОЛЖЕН вызвать исключение
    function_that_raises()
    
# После выхода из блока — проверяем exc_info
assert exc_info.type == ExpectedException
assert str(exc_info.value) == "Expected error message"
2.3 Объект exc_info
pytest.raises() возвращает объект ExceptionInfo, который содержит:

Атрибут	Описание	Пример
exc_info.type	Тип исключения	ZeroDivisionError
exc_info.value	Экземпляр исключения	ZeroDivisionError("division by zero")
exc_info.traceback	Объект traceback	Информация о стеке вызовов
python
def test_exception_details():
    with pytest.raises(ValueError) as exc_info:
        int("not a number")
    
    assert exc_info.type == ValueError
    assert "invalid literal" in str(exc_info.value)
3. Проверка типа исключения
3.1 Проверка точного типа
python
def test_exact_type():
    with pytest.raises(ZeroDivisionError):
        10 / 0
    
    # Сработает только если исключение именно ZeroDivisionError
    # (не сработает для наследников)
3.2 Проверка иерархии исключений
python
def test_hierarchy():
    # Проверка на любое исключение
    with pytest.raises(Exception):
        10 / 0
    
    # Проверка на ArithmeticError (родитель ZeroDivisionError)
    with pytest.raises(ArithmeticError):
        10 / 0
3.3 Проверка нескольких типов исключений
python
def test_multiple_exceptions():
    with pytest.raises((ValueError, TypeError)):
        # Может быть либо ValueError, либо TypeError
        int("abc")  # ValueError
4. Проверка сообщения об ошибке
4.1 Точное совпадение сообщения
python
def test_exact_message():
    with pytest.raises(ValueError) as exc_info:
        raise ValueError("Age must be positive")
    
    assert str(exc_info.value) == "Age must be positive"
4.2 Частичное совпадение (подстрока)
python
def test_partial_message():
    with pytest.raises(ValueError) as exc_info:
        raise ValueError("Invalid value: 123")
    
    assert "Invalid value" in str(exc_info.value)
    assert "123" in str(exc_info.value)
4.3 Использование match (рекомендуемый способ)
python
def test_match_parameter():
    # match проверяет регулярное выражение в сообщении
    with pytest.raises(ValueError, match="Age must be positive"):
        raise ValueError("Age must be positive")
    
    # С регулярным выражением
    with pytest.raises(ValueError, match=r"Invalid.*\d+"):
        raise ValueError("Invalid value: 123")
4.4 Сравнение подходов
Подход	Преимущества	Недостатки
match="текст"	Кратко, поддерживает regex	Нельзя проверить несколько условий
assert in str(exc_info.value)	Гибкость, несколько проверок	Более многословно
5. Проверка атрибутов исключения
5.1 Пользовательские исключения с атрибутами
python
class ValidationError(Exception):
    def __init__(self, field: str, message: str, code: int):
        self.field = field
        self.message = message
        self.code = code
        super().__init__(f"{field}: {message}")

def test_custom_exception_attributes():
    with pytest.raises(ValidationError) as exc_info:
        raise ValidationError("email", "Invalid format", 1001)
    
    # Проверяем атрибуты
    assert exc_info.value.field == "email"
    assert exc_info.value.message == "Invalid format"
    assert exc_info.value.code == 1001
5.2 Проверка вложенных исключений (cause)
python
def test_chained_exceptions():
    try:
        try:
            raise ValueError("Original error")
        except ValueError as e:
            raise RuntimeError("Wrapper") from e
    except RuntimeError as e:
        assert e.__cause__ is not None
        assert isinstance(e.__cause__, ValueError)

def test_chained_with_pytest():
    with pytest.raises(RuntimeError) as exc_info:
        try:
            raise ValueError("Original")
        except ValueError as e:
            raise RuntimeError("Wrapper") from e
    
    assert exc_info.value.__cause__ is not None
    assert str(exc_info.value.__cause__) == "Original"
6. Негативные сценарии тестирования
6.1 Что такое негативные сценарии
Позитивный сценарий (Happy Path) — проверка, что при корректных данных функция работает правильно.

Негативный сценарий (Negative Path) — проверка, что при некорректных данных функция выбрасывает ожидаемое исключение.

6.2 Пример: тестирование функции set_age()
python
class User:
    def set_age(self, age: int):
        if not isinstance(age, int):
            raise TypeError("Age must be integer")
        if age < 0:
            raise ValueError("Age cannot be negative")
        if age > 150:
            raise ValueError("Age cannot exceed 150")
        self._age = age

def test_set_age_positive_scenarios():
    """Позитивные сценарии."""
    user = User()
    user.set_age(25)   # Должно работать
    user.set_age(0)    # Минимальное допустимое
    user.set_age(150)  # Максимальное допустимое

def test_set_age_negative_scenarios():
    """Негативные сценарии."""
    user = User()
    
    # Неверный тип
    with pytest.raises(TypeError, match="Age must be integer"):
        user.set_age("25")
    
    # Отрицательный возраст
    with pytest.raises(ValueError, match="Age cannot be negative"):
        user.set_age(-1)
    
    # Слишком большой возраст
    with pytest.raises(ValueError, match="Age cannot exceed 150"):
        user.set_age(151)
6.3 Матрица негативных сценариев
Входное значение	Ожидаемое исключение	Причина
-1	ValueError	Возраст не может быть отрицательным
151	ValueError	Превышение максимального возраста
"25"	TypeError	Неправильный тип
None	TypeError	Пустое значение
3.14	TypeError	Не целое число
7. Практические примеры
7.1 Тестирование функции деления
python
def divide(a: float, b: float) -> float:
    """Делит a на b."""
    if b == 0:
        raise ZeroDivisionError("Cannot divide by zero")
    return a / b

def test_divide_positive():
    """Позитивный тест."""
    assert divide(10, 2) == 5.0

def test_divide_negative():
    """Негативные тесты."""
    with pytest.raises(ZeroDivisionError, match="Cannot divide by zero"):
        divide(10, 0)
    
    with pytest.raises(ZeroDivisionError):
        divide(-5, 0)
7.2 Тестирование парсера JSON
python
import json

def parse_json(data: str) -> dict:
    """Парсит JSON строку."""
    try:
        return json.loads(data)
    except json.JSONDecodeError as e:
        raise ValueError(f"Invalid JSON: {e}")

def test_parse_json_positive():
    assert parse_json('{"key": "value"}') == {"key": "value"}

def test_parse_json_negative():
    with pytest.raises(ValueError) as exc_info:
        parse_json('{invalid json}')
    
    assert "Invalid JSON" in str(exc_info.value)
7.3 Тестирование с побочными эффектами
python
class BankAccount:
    def __init__(self):
        self.balance = 0
    
    def withdraw(self, amount: float):
        if amount > self.balance:
            raise ValueError("Insufficient funds")
        self.balance -= amount
        return self.balance

def test_withdraw_negative_no_side_effects():
    """Проверка, что при исключении состояние не меняется."""
    account = BankAccount()
    account.balance = 100
    
    with pytest.raises(ValueError, match="Insufficient funds"):
        account.withdraw(200)
    
    # Баланс не должен измениться
    assert account.balance == 100
8. Типичные ошибки при тестировании исключений
8.1 Исключение не возникло
python
# ❌ НЕПРАВИЛЬНО: тест пройдёт, даже если исключения не было
def test_bad():
    try:
        function_that_should_raise()
        assert False  # Эта строка не выполнится, если было исключение
    except ValueError:
        pass

# ✅ ПРАВИЛЬНО: pytest.raises сам проверит
def test_good():
    with pytest.raises(ValueError):
        function_that_should_raise()
8.2 Слишком широкий перехват
python
# ❌ НЕПРАВИЛЬНО: поймает любое исключение
def test_bad():
    with pytest.raises(Exception):
        function_that_should_raise_ValueError()

# ✅ ПРАВИЛЬНО: проверяем конкретный тип
def test_good():
    with pytest.raises(ValueError):
        function_that_should_raise_ValueError()
8.3 Код после исключения
python
# ❌ НЕПРАВИЛЬНО: код после блока with не выполнится
def test_bad():
    with pytest.raises(ValueError):
        raise ValueError("Error")
        print("Это never print")  # Недостижимый код

# ✅ ПРАВИЛЬНО: проверка состояния после исключения
def test_good():
    obj = MyClass()
    with pytest.raises(ValueError):
        obj.bad_method()
    assert obj.state == "unchanged"  # Выполнится после исключения
9. Шпаргалка
python
# === БАЗОВАЯ ПРОВЕРКА ===
with pytest.raises(ZeroDivisionError):
    10 / 0

# === ПРОВЕРКА ТИПА И СООБЩЕНИЯ ===
with pytest.raises(ValueError, match="Age must be positive"):
    set_age(-5)

# === ПОЛУЧЕНИЕ ОБЪЕКТА ИСКЛЮЧЕНИЯ ===
with pytest.raises(ValueError) as exc_info:
    int("abc")
assert exc_info.type == ValueError
assert "invalid literal" in str(exc_info.value)

# === ПРОВЕРКА НЕСКОЛЬКИХ ТИПОВ ===
with pytest.raises((ValueError, TypeError)):
    function_that_may_raise_either()

# === ПРОВЕРКА АТРИБУТОВ ===
class MyError(Exception):
    def __init__(self, code):
        self.code = code

with pytest.raises(MyError) as exc_info:
    raise MyError(100)
assert exc_info.value.code == 100

# === ПРОВЕРКА ОТСУТСТВИЯ ИСКЛЮЧЕНИЯ ===
# (просто вызываем функцию без with)
def test_no_exception():
    function_that_should_not_raise()  # Если выбросит — тест упадёт
Контрольные вопросы
В чём разница между позитивным и негативным тестовым сценарием?

Что возвращает pytest.raises() как контекстный менеджер?

Как проверить, что исключение содержит определённую подстроку в сообщении?

Чем отличается проверка match="текст" от assert in str(exc_info.value)?

Как проверить атрибуты пользовательского исключения?

Почему не стоит использовать with pytest.raises(Exception) для проверки конкретных исключений?

Как проверить, что после исключения состояние системы не изменилось?

Что произойдёт, если код внутри with pytest.raises() не выбросит исключение?

Как проверить цепочку исключений (__cause__)?

Зачем нужны негативные сценарии, если «всё должно работать»?

Следующее занятие: ПР 3.7 — Практическое тестирование исключений и коллекций с использованием pytest.raises().
