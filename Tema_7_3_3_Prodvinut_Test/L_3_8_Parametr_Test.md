# Лекция 3.8: Параметризация тестов. Продвинутое мокирование

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить продвинутые методы параметризации тестов в pytest, освоить сложные техники мокирования с использованием `unittest.mock` и `monkeypatch` для подмены времени, HTTP-запросов, файловой системы, а также проверки вызовов моков.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Применять продвинутую параметризацию тестов (ПК 3.2).
2. Использовать `monkeypatch` для подмены времени, файловой системы, HTTP (ПК 3.2, ПК 3.3).
3. Создавать сложные моки с помощью `unittest.mock` (ПК 3.2).
4. Проверять вызовы моков с помощью `assert_called_with`, `assert_called_once`, `assert_has_calls` (ПК 3.3).

---

## 1. Продвинутая параметризация тестов

### 1.1 Базовый синтаксис параметризации

```python
import pytest


@pytest.mark.parametrize("test_input,expected", [
    (1, 2),
    (2, 4),
    (3, 6),
])
def test_multiply_by_two(test_input, expected):
    assert test_input * 2 == expected
1.2 Параметризация с несколькими аргументами
python
@pytest.mark.parametrize("a,b,expected", [
    (1, 2, 3),
    (10, 20, 30),
    (-1, 1, 0),
    (0, 0, 0),
])
def test_add(a, b, expected):
    assert a + b == expected
1.3 Вложенная параметризация (декартово произведение)
python
@pytest.mark.parametrize("x", [1, 2, 3])
@pytest.mark.parametrize("y", [10, 20, 30])
def test_combinations(x, y):
    # Запустится 3 × 3 = 9 раз
    assert isinstance(x + y, int)


@pytest.mark.parametrize("name", ["Alice", "Bob"])
@pytest.mark.parametrize("age", [25, 30, 35])
def test_user_combinations(name, age):
    # Запустится 2 × 3 = 6 раз
    assert len(name) > 0
    assert age > 0
1.4 Параметризация с использованием объектов
python
@pytest.fixture(params=[
    {"name": "Alice", "age": 25},
    {"name": "Bob", "age": 30},
    {"name": "Charlie", "age": 35},
])
def user_data(request):
    return request.param


def test_user_data(user_data):
    assert user_data["age"] >= 18
    assert isinstance(user_data["name"], str)
1.5 Динамическая параметризация
python
import pytest


def generate_test_data():
    """Генерация тестовых данных динамически."""
    data = []
    for i in range(10):
        data.append((i, i ** 2))
    return data


@pytest.mark.parametrize("n,expected", generate_test_data())
def test_square(n, expected):
    assert n ** 2 == expected


# Параметризация из файла
def load_test_data():
    import json
    with open("test_data.json") as f:
        return json.load(f)


@pytest.mark.parametrize("test_case", load_test_data())
def test_from_file(test_case):
    result = process(test_case["input"])
    assert result == test_case["expected"]
1.6 Параметризация с маркировкой
python
@pytest.mark.parametrize("n,expected", [
    pytest.param(1, 1, id="case_one"),
    pytest.param(2, 4, id="case_two"),
    pytest.param(3, 9, id="case_three"),
    pytest.param(0, 0, marks=pytest.mark.xfail, id="zero_case"),
    pytest.param(-1, 1, marks=pytest.mark.skip, id="negative_case"),
])
def test_square_with_ids(n, expected):
    assert n ** 2 == expected
2. Мокирование времени (time / datetime)
2.1 Мокирование time.time() с monkeypatch
python
import time
import pytest
from datetime import datetime


def test_current_time():
    start_time = time.time()
    # какой-то код
    end_time = time.time()


def test_mocked_time(monkeypatch):
    """Мокирование time.time() с monkeypatch."""
    mock_times = [1000.0, 1000.5, 1001.0]
    call_count = 0
    
    def mock_time():
        nonlocal call_count
        result = mock_times[call_count % len(mock_times)]
        call_count += 1
        return result
    
    monkeypatch.setattr(time, "time", mock_time)
    
    assert time.time() == 1000.0
    assert time.time() == 1000.5
    assert time.time() == 1001.0


def test_mocked_datetime(monkeypatch):
    """Мокирование datetime.now()."""
    class MockDateTime:
        @classmethod
        def now(cls):
            return datetime(2024, 1, 1, 12, 0, 0)
    
    monkeypatch.setattr(datetime, "datetime", MockDateTime)
    
    assert datetime.now().year == 2024
    assert datetime.now().month == 1
2.2 Мокирование времени с pytest-mock
python
def test_mocked_time_with_mocker(mocker):
    """Мокирование времени с pytest-mock."""
    mock_time = mocker.patch("time.time")
    mock_time.side_effect = [1000.0, 1000.5, 1001.0]
    
    assert time.time() == 1000.0
    assert time.time() == 1000.5
    assert time.time() == 1001.0
3. Мокирование HTTP-запросов
3.1 Мокирование requests.get с monkeypatch
python
import requests
import pytest


def test_mocked_http_with_monkeypatch(monkeypatch):
    """Мокирование HTTP-запроса с monkeypatch."""
    
    class MockResponse:
        def __init__(self, json_data, status_code):
            self.json_data = json_data
            self.status_code = status_code
        
        def json(self):
            return self.json_data
    
    def mock_get(*args, **kwargs):
        return MockResponse({"data": "mocked"}, 200)
    
    monkeypatch.setattr(requests, "get", mock_get)
    
    response = requests.get("https://api.example.com/users")
    assert response.status_code == 200
    assert response.json() == {"data": "mocked"}
3.2 Сложное мокирование HTTP с side_effect
python
def test_mocked_http_with_side_effect(mocker):
    """Мокирование с side_effect для разных ответов."""
    mock_get = mocker.patch("requests.get")
    
    # Создаём моки-ответы
    mock_response_200 = mocker.Mock()
    mock_response_200.status_code = 200
    mock_response_200.json.return_value = {"status": "ok"}
    
    mock_response_404 = mocker.Mock()
    mock_response_404.status_code = 404
    mock_response_404.raise_for_status.side_effect = requests.HTTPError("404 Error")
    
    mock_get.side_effect = [mock_response_200, mock_response_404]
    
    # Первый запрос
    response1 = requests.get("https://api.example.com/users/1")
    assert response1.status_code == 200
    assert response1.json() == {"status": "ok"}
    
    # Второй запрос
    with pytest.raises(requests.HTTPError):
        requests.get("https://api.example.com/users/999")
4. Мокирование файловой системы
4.1 Мокирование open с monkeypatch
python
import pytest
from io import StringIO


def test_mocked_file_reading(monkeypatch):
    """Мокирование чтения файла."""
    
    mock_file = StringIO("mocked content\nline2")
    
    def mock_open(*args, **kwargs):
        return mock_file
    
    monkeypatch.setattr("builtins.open", mock_open)
    
    with open("any_file.txt") as f:
        content = f.read()
    
    assert content == "mocked content\nline2"


def test_mocked_file_error(monkeypatch):
    """Мокирование ошибки при открытии файла."""
    
    def mock_open(*args, **kwargs):
        raise PermissionError("Access denied")
    
    monkeypatch.setattr("builtins.open", mock_open)
    
    with pytest.raises(PermissionError, match="Access denied"):
        with open("/protected/file.txt") as f:
            f.read()
4.2 Мокирование os.path.exists
python
import os
import pytest


def test_mocked_path_exists(monkeypatch):
    """Мокирование os.path.exists."""
    
    def mock_exists(path):
        return path == "/valid/path"
    
    monkeypatch.setattr(os.path, "exists", mock_exists)
    
    assert os.path.exists("/valid/path") is True
    assert os.path.exists("/invalid/path") is False
5. Создание сложных моков
5.1 Использование create_autospec
python
from unittest.mock import create_autospec


class UserService:
    def get_user(self, user_id: int) -> dict:
        pass
    
    def create_user(self, name: str, email: str) -> int:
        pass


def test_create_autospec():
    """Создание мока с проверкой сигнатуры."""
    
    # create_autospec проверяет, что мок соответствует интерфейсу
    mock_service = create_autospec(UserService)
    
    # Настройка поведения
    mock_service.get_user.return_value = {"id": 1, "name": "Alice"}
    
    result = mock_service.get_user(1)
    assert result["name"] == "Alice"
    
    # Проверка вызова
    mock_service.get_user.assert_called_once_with(1)
    
    # При вызове с неправильной сигнатурой будет ошибка
    # mock_service.get_user()  # TypeError: missing argument


def test_autospec_with_real_object():
    """create_autospec с реальным объектом."""
    
    real_service = UserService()
    mock_service = create_autospec(real_service)
    
    mock_service.get_user.return_value = {"id": 1}
    result = mock_service.get_user(1)
    
    assert result["id"] == 1
5.2 Моки с Property
python
class Config:
    @property
    def api_key(self) -> str:
        return "real-key"


def test_mock_property(mocker):
    """Мокирование property."""
    
    mock_config = mocker.Mock(spec=Config)
    mock_config.api_key = "mocked-key"
    
    assert mock_config.api_key == "mocked-key"
5.3 Моки с возвратом разных значений
python
def test_mock_side_effect_sequence(mocker):
    """Мок с последовательностью возвращаемых значений."""
    
    mock_func = mocker.Mock()
    mock_func.side_effect = [1, 2, 3, Exception("Error!")]
    
    assert mock_func() == 1
    assert mock_func() == 2
    assert mock_func() == 3
    with pytest.raises(Exception, match="Error!"):
        mock_func()


def test_mock_side_effect_function(mocker):
    """Мок с функцией в side_effect."""
    
    def dynamic_response(*args, **kwargs):
        if args[0] == "admin":
            return {"role": "admin"}
        return {"role": "user"}
    
    mock_func = mocker.Mock()
    mock_func.side_effect = dynamic_response
    
    assert mock_func("admin")["role"] == "admin"
    assert mock_func("user")["role"] == "user"
6. Проверка вызовов моков
6.1 Базовые проверки
python
def test_call_assertions(mocker):
    """Базовые проверки вызовов моков."""
    
    mock = mocker.Mock()
    
    # Вызовы
    mock(1, 2, key="value")
    mock.method(3, 4)
    
    # Проверка количества вызовов
    assert mock.call_count == 1
    mock.method.assert_called_once()
    mock.method.assert_called_once_with(3, 4)
    
    # Проверка с частичным совпадением
    mock.method.assert_called_with(3, 4)


def test_multiple_calls(mocker):
    """Проверка нескольких вызовов."""
    
    mock = mocker.Mock()
    mock(1)
    mock(2)
    mock(3)
    
    assert mock.call_count == 3
    assert mock.mock_calls == [((1,), {}), ((2,), {}), ((3,), {})]
6.2 Продвинутые проверки вызовов
python
def test_advanced_call_assertions(mocker):
    """Продвинутые проверки вызовов моков."""
    
    mock = mocker.Mock()
    
    # Различные вызовы
    mock.method("first")
    mock.method("second", flag=True)
    mock.method("third", flag=False, count=3)
    
    # Проверка конкретного вызова
    mock.method.assert_any_call("second", flag=True)
    
    # Проверка последовательности вызовов
    expected_calls = [
        mocker.call("first"),
        mocker.call("second", flag=True),
        mocker.call("third", flag=False, count=3),
    ]
    mock.method.assert_has_calls(expected_calls)
    
    # Проверка, что других вызовов не было
    assert mock.method.call_count == 3


def test_call_args_extraction(mocker):
    """Извлечение аргументов вызовов."""
    
    mock = mocker.Mock()
    mock(10, 20, key="value")
    
    # Получение аргументов последнего вызова
    args, kwargs = mock.call_args
    assert args == (10, 20)
    assert kwargs == {"key": "value"}
7. Шпаргалка
python
# === ПАРАМЕТРИЗАЦИЯ ===
@pytest.mark.parametrize("x,y", [(1,2), (3,4)])
def test_params(x, y):
    assert x + y > 0

# === МОКИРОВАНИЕ ВРЕМЕНИ ===
def test_time(monkeypatch):
    monkeypatch.setattr(time, "time", lambda: 1000.0)

# === МОКИРОВАНИЕ HTTP ===
def test_http(mocker):
    mock_get = mocker.patch("requests.get")
    mock_get.return_value.status_code = 200

# === МОКИРОВАНИЕ ФАЙЛОВ ===
def test_file(monkeypatch):
    monkeypatch.setattr("builtins.open", mock_open)

# === СОЗДАНИЕ МОКОВ ===
mock = mocker.Mock()
mock.method.return_value = "result"
mock.method.side_effect = [1, 2, Exception()]

# === ПРОВЕРКА ВЫЗОВОВ ===
mock.method.assert_called_once()
mock.method.assert_called_once_with(1, 2)
mock.method.assert_any_call(3, 4)
assert mock.method.call_count == 5
Контрольные вопросы
Как параметризовать тест с несколькими аргументами?

Как создать вложенную параметризацию (декартово произведение)?

Как мокировать time.time() с помощью monkeypatch?

Как мокировать requests.get с разными ответами?

Как мокировать чтение файла и эмулировать ошибку?

Что делает create_autospec и зачем он нужен?

Как задать последовательность возвращаемых значений для мока?

Как проверить, что мок был вызван с определёнными аргументами?

Как проверить последовательность вызовов мока?

Как извлечь аргументы вызова мока?

Следующее занятие: ПР 3.8 — Практическая параметризация и мокирование.
