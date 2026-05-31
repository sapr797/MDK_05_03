# Лекция 3.2: Инструменты для мокирования: unittest.mock, pytest-mock, responses, requests-mock

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.1:** Тестирование API-клиентов и сетевых запросов  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить основные инструменты для мокирования HTTP-запросов в Python: `unittest.mock`, `pytest-mock`, `responses`, `requests-mock`. Научиться тестировать обработку ошибок сети и некорректных HTTP-статусов.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Выбирать подходящий инструмент для мокирования HTTP-запросов (ОК 02, ОК 05).
2. Использовать `unittest.mock`, `pytest-mock`, `responses`, `requests-mock` для тестирования (ПК 3.2).
3. Тестировать обработку различных сетевых ошибок (ПК 3.3).
4. Тестировать поведение клиента при разных HTTP-статусах (ПК 3.3).

---

## 1. Обзор инструментов мокирования
┌─────────────────────────────────────────────────────────────────────────────┐
│ ИНСТРУМЕНТЫ ДЛЯ МОКИРОВАНИЯ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ 1. unittest.mock (встроенный) │
│ └── Универсальное решение, не требует установки │
│ │
│ 2. pytest-mock │
│ └── Обёртка над unittest.mock с удобным синтаксисом для pytest │
│ │
│ 3. responses │
│ └── Специализирован для HTTP, регистрирует ожидаемые запросы │
│ │
│ 4. requests-mock │
│ └── Альтернатива responses, интеграция с адаптерами requests │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### Сравнительная таблица

| Характеристика | `unittest.mock` | `pytest-mock` | `responses` | `requests-mock` |
|:---|:---|:---|:---|:---|
| Установка | Встроен | `pip install pytest-mock` | `pip install responses` | `pip install requests-mock` |
| Синтаксис | Громоздкий | Краткий | Декларативный | Декларативный |
| Проверка параметров | Сложная | Средняя | Простая (matchers) | Простая |
| Проверка вызовов | Да | Да | Да | Да |
| Эмуляция ошибок | Да | Да | Да | Да |
| Рекомендация | Простые моки | Тесты с pytest | HTTP-тесты | HTTP-тесты |

---

## 2. Инструмент 1: unittest.mock

### 2.1 Базовое использование

```python
# test_unittest_mock.py

import unittest
from unittest.mock import patch, Mock
import requests


class ApiClient:
    """Простой API-клиент."""
    
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.session = requests.Session()
    
    def get_user(self, user_id: int) -> dict:
        """Получает пользователя по ID."""
        response = self.session.get(f"{self.base_url}/users/{user_id}")
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 404:
            return None
        else:
            response.raise_for_status()


class TestApiClientUnittest(unittest.TestCase):
    """Тесты с использованием unittest.mock."""
    
    def test_get_user_success(self):
        """Успешное получение пользователя."""
        # Создаём мок-ответ
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {"id": 1, "name": "Alice"}
        
        # Патчим session.get
        with patch.object(ApiClient, 'session') as mock_session:
            mock_session.get.return_value = mock_response
            
            client = ApiClient("https://api.example.com")
            result = client.get_user(1)
            
            self.assertEqual(result, {"id": 1, "name": "Alice"})
            mock_session.get.assert_called_once_with("https://api.example.com/users/1")
    
    def test_get_user_not_found(self):
        """Пользователь не найден."""
        mock_response = Mock()
        mock_response.status_code = 404
        
        with patch.object(ApiClient, 'session') as mock_session:
            mock_session.get.return_value = mock_response
            
            client = ApiClient("https://api.example.com")
            result = client.get_user(999)
            
            self.assertIsNone(result)
    
    def test_get_user_server_error(self):
        """Ошибка сервера."""
        mock_response = Mock()
        mock_response.status_code = 500
        mock_response.raise_for_status.side_effect = requests.HTTPError("500 Server Error")
        
        with patch.object(ApiClient, 'session') as mock_session:
            mock_session.get.return_value = mock_response
            
            client = ApiClient("https://api.example.com")
            
            with self.assertRaises(requests.HTTPError):
                client.get_user(1)
3. Инструмент 2: pytest-mock
3.1 Установка и базовое использование
bash
pip install pytest-mock
3.2 Примеры использования
python
# test_pytest_mock.py

import pytest
import requests
from api_client import ApiClient


class TestApiClientPytestMock:
    """Тесты с использованием pytest-mock."""
    
    def test_get_user_success(self, mocker):
        """Успешное получение пользователя."""
        # Создаём мок-ответ
        mock_response = mocker.Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {"id": 1, "name": "Alice"}
        
        # Патчим метод get
        mock_get = mocker.patch("requests.Session.get")
        mock_get.return_value = mock_response
        
        client = ApiClient("https://api.example.com")
        result = client.get_user(1)
        
        assert result == {"id": 1, "name": "Alice"}
        mock_get.assert_called_once_with("https://api.example.com/users/1")
    
    def test_get_user_several_calls(self, mocker):
        """Несколько последовательных вызовов."""
        # Создаём список ответов
        responses = [
            mocker.Mock(status_code=200, json=lambda: {"id": 1, "name": "Alice"}),
            mocker.Mock(status_code=200, json=lambda: {"id": 2, "name": "Bob"}),
            mocker.Mock(status_code=404),
        ]
        
        mock_get = mocker.patch("requests.Session.get")
        mock_get.side_effect = responses
        
        client = ApiClient("https://api.example.com")
        
        result1 = client.get_user(1)
        result2 = client.get_user(2)
        result3 = client.get_user(3)
        
        assert result1["name"] == "Alice"
        assert result2["name"] == "Bob"
        assert result3 is None
        assert mock_get.call_count == 3
    
    def test_network_error(self, mocker):
        """Сетевая ошибка."""
        mock_get = mocker.patch("requests.Session.get")
        mock_get.side_effect = requests.ConnectionError("Network unreachable")
        
        client = ApiClient("https://api.example.com")
        
        with pytest.raises(requests.ConnectionError):
            client.get_user(1)
    
    def test_timeout_error(self, mocker):
        """Таймаут."""
        mock_get = mocker.patch("requests.Session.get")
        mock_get.side_effect = requests.Timeout("Request timed out")
        
        client = ApiClient("https://api.example.com")
        
        with pytest.raises(requests.Timeout):
            client.get_user(1)
    
    def test_http_error_codes(self, mocker):
        """Параметризованный тест HTTP-статусов."""
        @pytest.mark.parametrize("status_code,expected", [
            (200, {"id": 1}),
            (404, None),
            (401, None),  # Обработка зависит от реализации
            (500, None),
        ])
        def test_status_codes(self, mocker, status_code, expected):
            mock_response = mocker.Mock()
            mock_response.status_code = status_code
            
            if status_code == 200:
                mock_response.json.return_value = {"id": 1}
            
            mock_get = mocker.patch("requests.Session.get")
            mock_get.return_value = mock_response
            
            client = ApiClient("https://api.example.com")
            
            if status_code == 500:
                mock_response.raise_for_status.side_effect = requests.HTTPError()
                with pytest.raises(requests.HTTPError):
                    client.get_user(1)
            else:
                result = client.get_user(1)
                assert result == expected
3.3 Фикстура mocker
pytest-mock предоставляет фикстуру mocker, которая автоматически:

Создаёт моки

Управляет их жизненным циклом

Восстанавливает оригинальные объекты после теста

python
def test_with_mocker_fixture(mocker):
    # mocker.patch возвращает Mock, который автоматически очищается
    mock = mocker.patch("module.function")
    mock.return_value = "mocked"
    
    # mocker.spy для отслеживания вызовов
    spy = mocker.spy(requests.Session, "get")
    
    # mocker.stopall() вызывается автоматически
4. Инструмент 3: responses
4.1 Установка и базовое использование
bash
pip install responses
4.2 Примеры использования
python
# test_responses.py

import pytest
import responses
import requests
from api_client import ApiClient


class TestApiClientResponses:
    """Тесты с использованием библиотеки responses."""
    
    @responses.activate
    def test_get_user_success(self):
        """Успешное получение пользователя."""
        # Регистрируем ожидаемый запрос
        responses.add(
            responses.GET,
            "https://api.example.com/users/1",
            json={"id": 1, "name": "Alice"},
            status=200
        )
        
        client = ApiClient("https://api.example.com")
        result = client.get_user(1)
        
        assert result == {"id": 1, "name": "Alice"}
        assert len(responses.calls) == 1
    
    @responses.activate
    def test_get_user_not_found(self):
        """Пользователь не найден."""
        responses.add(
            responses.GET,
            "https://api.example.com/users/999",
            status=404
        )
        
        client = ApiClient("https://api.example.com")
        result = client.get_user(999)
        
        assert result is None
    
    @responses.activate
    def test_get_user_server_error(self):
        """Ошибка сервера."""
        responses.add(
            responses.GET,
            "https://api.example.com/users/1",
            status=500,
            body="Internal Server Error"
        )
        
        client = ApiClient("https://api.example.com")
        
        with pytest.raises(requests.HTTPError):
            client.get_user(1)
    
    @responses.activate
    def test_multiple_endpoints(self):
        """Несколько эндпоинтов."""
        # Регистрируем разные эндпоинты
        responses.add(
            responses.GET,
            "https://api.example.com/users/1",
            json={"id": 1, "name": "Alice"},
            status=200
        )
        
        responses.add(
            responses.GET,
            "https://api.example.com/users/2",
            json={"id": 2, "name": "Bob"},
            status=200
        )
        
        client = ApiClient("https://api.example.com")
        
        alice = client.get_user(1)
        bob = client.get_user(2)
        
        assert alice["name"] == "Alice"
        assert bob["name"] == "Bob"
        assert len(responses.calls) == 2
    
    @responses.activate
    def test_network_error_response(self):
        """Эмуляция сетевой ошибки."""
        responses.add(
            responses.GET,
            "https://api.example.com/users/1",
            body=requests.ConnectionError("Connection refused")
        )
        
        client = ApiClient("https://api.example.com")
        
        with pytest.raises(requests.ConnectionError):
            client.get_user(1)
4.3 Расширенные возможности responses
python
class TestResponsesAdvanced:
    """Расширенные возможности responses."""
    
    @responses.activate
    def test_query_parameters(self):
        """Проверка параметров запроса."""
        import re
        
        # Использование регулярного выражения
        responses.add(
            responses.GET,
            re.compile(r"https://api.example.com/users/\d+"),
            json={"id": 1, "name": "Alice"},
            status=200
        )
        
        client = ApiClient("https://api.example.com")
        result = client.get_user(123)
        
        assert result is not None
    
    @responses.activate
    def test_custom_matcher(self):
        """Кастомный matcher для параметров."""
        responses.add(
            responses.GET,
            "https://api.example.com/users",
            json={"users": []},
            status=200,
            match=[
                responses.matchers.query_param_matcher({"active": "true"}),
                responses.matchers.header_matcher({"X-API-Key": "test-key"})
            ]
        )
        
        # Этот запрос совпадёт с matcher'ом
        response = requests.get(
            "https://api.example.com/users",
            params={"active": "true"},
            headers={"X-API-Key": "test-key"}
        )
        
        assert response.status_code == 200
    
    @responses.activate
    def test_dynamic_response(self):
        """Динамический ответ на основе запроса."""
        def request_callback(request):
            user_id = request.url.split("/")[-1]
            return (200, {}, json.dumps({"id": int(user_id), "name": f"User{user_id}"}))
        
        responses.add_callback(
            responses.GET,
            re.compile(r"https://api.example.com/users/\d+"),
            callback=request_callback,
            content_type="application/json"
        )
        
        client = ApiClient("https://api.example.com")
        
        result1 = client.get_user(10)
        result2 = client.get_user(20)
        
        assert result1["id"] == 10
        assert result1["name"] == "User10"
        assert result2["id"] == 20
        assert result2["name"] == "User20"
5. Инструмент 4: requests-mock
5.1 Установка и базовое использование
bash
pip install requests-mock
5.2 Примеры использования
python
# test_requests_mock.py

import pytest
import requests_mock
from api_client import ApiClient


class TestApiClientRequestsMock:
    """Тесты с использованием requests-mock."""
    
    def test_get_user_success(self):
        """Успешное получение пользователя."""
        with requests_mock.Mocker() as m:
            m.get(
                "https://api.example.com/users/1",
                json={"id": 1, "name": "Alice"},
                status_code=200
            )
            
            client = ApiClient("https://api.example.com")
            result = client.get_user(1)
            
            assert result == {"id": 1, "name": "Alice"}
            assert m.called_once
            assert m.call_count == 1
    
    def test_get_user_not_found(self):
        """Пользователь не найден."""
        with requests_mock.Mocker() as m:
            m.get(
                "https://api.example.com/users/999",
                status_code=404
            )
            
            client = ApiClient("https://api.example.com")
            result = client.get_user(999)
            
            assert result is None
    
    def test_get_user_server_error(self):
        """Ошибка сервера."""
        import requests
        
        with requests_mock.Mocker() as m:
            m.get(
                "https://api.example.com/users/1",
                status_code=500,
                text="Internal Server Error"
            )
            
            client = ApiClient("https://api.example.com")
            
            with pytest.raises(requests.HTTPError):
                client.get_user(1)
    
    def test_connection_error(self):
        """Ошибка соединения."""
        import requests
        
        with requests_mock.Mocker() as m:
            m.get(
                "https://api.example.com/users/1",
                exc=requests.ConnectionError("Connection refused")
            )
            
            client = ApiClient("https://api.example.com")
            
            with pytest.raises(requests.ConnectionError):
                client.get_user(1)
    
    def test_request_matcher(self):
        """Проверка параметров запроса."""
        with requests_mock.Mocker() as m:
            def request_matcher(request):
                return request.qs.get("active") == ["true"]
            
            m.get(
                "https://api.example.com/users",
                json={"users": []},
                status_code=200,
                additional_matcher=request_matcher
            )
            
            response = requests.get(
                "https://api.example.com/users",
                params={"active": "true"}
            )
            
            assert response.status_code == 200
    
    def test_multiple_responses(self):
        """Несколько последовательных ответов."""
        with requests_mock.Mocker() as m:
            m.get(
                "https://api.example.com/users/1",
                [
                    {"json": {"id": 1, "name": "Alice"}, "status_code": 200},
                    {"json": {"id": 1, "name": "Alice Updated"}, "status_code": 200},
                ]
            )
            
            client = ApiClient("https://api.example.com")
            
            result1 = client.get_user(1)
            result2 = client.get_user(1)
            
            assert result1["name"] == "Alice"
            assert result2["name"] == "Alice Updated"
5.3 requests-mock как фикстура pytest
python
# conftest.py
import pytest
import requests_mock


@pytest.fixture
def mock_requests():
    """Фикстура для requests-mock."""
    with requests_mock.Mocker() as m:
        yield m


# test_with_fixture.py
def test_api_client_with_fixture(mock_requests):
    """Использование фикстуры mock_requests."""
    mock_requests.get(
        "https://api.example.com/users/1",
        json={"id": 1, "name": "Alice"},
        status_code=200
    )
    
    client = ApiClient("https://api.example.com")
    result = client.get_user(1)
    
    assert result["name"] == "Alice"
6. Тестирование обработки сетевых ошибок
6.1 Полный набор тестов для обработки ошибок
python
# test_error_handling.py

import pytest
import requests
from unittest.mock import patch, Mock
from api_client import ApiClient


class TestErrorHandling:
    """Комплексное тестирование обработки ошибок."""
    
    # ==========================================================
    # ТЕСТЫ С unittest.mock
    # ==========================================================
    
    def test_connection_error_mock(self):
        """ConnectionError через unittest.mock."""
        with patch("requests.Session.get") as mock_get:
            mock_get.side_effect = requests.ConnectionError("Connection failed")
            
            client = ApiClient("https://api.example.com")
            
            with pytest.raises(requests.ConnectionError):
                client.get_user(1)
    
    def test_timeout_error_mock(self):
        """Timeout через unittest.mock."""
        with patch("requests.Session.get") as mock_get:
            mock_get.side_effect = requests.Timeout("Request timeout")
            
            client = ApiClient("https://api.example.com")
            
            with pytest.raises(requests.Timeout):
                client.get_user(1)
    
    # ==========================================================
    # ТЕСТЫ С pytest-mock
    # ==========================================================
    
    def test_connection_error_pytest_mock(self, mocker):
        """ConnectionError через pytest-mock."""
        mock_get = mocker.patch("requests.Session.get")
        mock_get.side_effect = requests.ConnectionError("Network unreachable")
        
        client = ApiClient("https://api.example.com")
        
        with pytest.raises(requests.ConnectionError):
            client.get_user(1)
    
    # ==========================================================
    # ТЕСТЫ С responses
    # ==========================================================
    
    @responses.activate
    def test_connection_error_responses(self):
        """ConnectionError через responses."""
        responses.add(
            responses.GET,
            "https://api.example.com/users/1",
            body=requests.ConnectionError("Connection refused")
        )
        
        client = ApiClient("https://api.example.com")
        
        with pytest.raises(requests.ConnectionError):
            client.get_user(1)
    
    @responses.activate
    def test_timeout_error_responses(self):
        """Timeout через responses."""
        responses.add(
            responses.GET,
            "https://api.example.com/users/1",
            body=requests.Timeout("Request timed out")
        )
        
        client = ApiClient("https://api.example.com")
        
        with pytest.raises(requests.Timeout):
            client.get_user(1)
    
    # ==========================================================
    # ТЕСТЫ С requests-mock
    # ==========================================================
    
    def test_connection_error_requests_mock(self):
        """ConnectionError через requests-mock."""
        with requests_mock.Mocker() as m:
            m.get(
                "https://api.example.com/users/1",
                exc=requests.ConnectionError("Connection refused")
            )
            
            client = ApiClient("https://api.example.com")
            
            with pytest.raises(requests.ConnectionError):
                client.get_user(1)
    
    @pytest.mark.parametrize("status_code,expected_behavior", [
        (200, "success"),
        (400, "client_error"),
        (401, "auth_error"),
        (403, "forbidden"),
        (404, "not_found"),
        (500, "server_error"),
        (502, "bad_gateway"),
        (503, "service_unavailable"),
    ])
    def test_http_status_codes(self, status_code, expected_behavior, mocker):
        """Параметризованный тест HTTP-статусов."""
        mock_response = Mock()
        mock_response.status_code = status_code
        
        if status_code == 200:
            mock_response.json.return_value = {"data": "ok"}
        
        if status_code >= 400:
            mock_response.raise_for_status.side_effect = requests.HTTPError()
        
        mock_get = mocker.patch("requests.Session.get")
        mock_get.return_value = mock_response
        
        client = ApiClient("https://api.example.com")
        
        if status_code == 200:
            result = client.get_user(1)
            assert result is not None
        elif status_code == 404:
            result = client.get_user(1)
            assert result is None
        else:
            with pytest.raises(requests.HTTPError):
                client.get_user(1)
7. Сравнительный анализ инструментов
7.1 Производительность
Инструмент	Время выполнения	Накладные расходы
unittest.mock	Низкое	Минимальные
pytest-mock	Низкое	Минимальные
responses	Среднее	Небольшие
requests-mock	Среднее	Небольшие
7.2 Рекомендации по выбору
python
# Ситуация 1: Простой мок одного вызова
# Выбор: pytest-mock (кратко)
def test_simple(mocker):
    mock = mocker.patch("requests.get")
    mock.return_value.status_code = 200

# Ситуация 2: Сложные сценарии с несколькими запросами
# Выбор: responses (декларативный)
@responses.activate
def test_complex():
    responses.add(responses.GET, "https://api.com/users", json={"users": []})
    responses.add(responses.POST, "https://api.com/users", status=201)

# Ситуация 3: Нет возможности установить сторонние библиотеки
# Выбор: unittest.mock
with patch("requests.get") as mock_get:
    mock_get.return_value.json.return_value = {"data": "test"}
8. Шпаргалка
python
# === unittest.mock ===
from unittest.mock import patch, Mock

with patch("requests.get") as mock_get:
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"key": "value"}
    mock_get.return_value = mock_response

# === pytest-mock ===
def test_example(mocker):
    mock_get = mocker.patch("requests.get")
    mock_get.return_value.status_code = 200
    mock_get.return_value.json.return_value = {"key": "value"}

# === responses ===
import responses

@responses.activate
def test_example():
    responses.add(
        responses.GET,
        "https://api.example.com/endpoint",
        json={"key": "value"},
        status=200
    )

# === requests-mock ===
import requests_mock

def test_example():
    with requests_mock.Mocker() as m:
        m.get("https://api.example.com/endpoint", json={"key": "value"})
Контрольные вопросы
Какие основные инструменты для мокирования HTTP-запросов существуют в Python?

В чём преимущество responses перед unittest.mock?

Как с помощью responses эмулировать таймаут?

Что такое mocker в pytest-mock и как он работает?

Как проверить, что запрос был сделан с определёнными параметрами?

Как эмулировать несколько последовательных ответов для одного URL?

Какой инструмент лучше всего подходит для тестирования сложных сценариев с множеством запросов?

Как в requests-mock создать фикстуру для повторного использования?

Чем отличается responses от requests-mock?

Как протестировать обработку ошибки 429 (Too Many Requests)?

Следующее занятие: ПР 3.2 — Практическое тестирование API-клиентов с использованием различных инструментов мокирования
