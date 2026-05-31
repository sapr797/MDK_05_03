# Практическое занятие 3.2: Тестирование сценариев отказа внешнего API (эмуляция ошибок 4xx, 5xx)

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.1:** Тестирование API-клиентов и сетевых запросов  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться тестировать сценарии отказа внешнего API с эмуляцией ошибок клиента (4xx) и сервера (5xx), разрабатывать комплексные тестовые наборы для клиента REST API.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Эмулировать различные HTTP-статусы ошибок (4xx, 5xx) с помощью `requests-mock` (ПК 3.2, ПК 3.3).
2. Тестировать корректную обработку ошибок клиентом (ПК 3.3).
3. Разрабатывать комплексные тестовые наборы для REST API клиента (ПК 3.2, ПК 3.3).
4. Анализировать поведение клиента при различных сценариях отказа API (ОК 02, ОК 05).

---

## Теоретическая справка

### Классификация HTTP-статусов

| Диапазон | Категория | Примеры | Действие клиента |
|:---|:---|:---|:---|
| **2xx** | Успех | 200 OK, 201 Created, 204 No Content | Обработка данных |
| **4xx** | Ошибка клиента | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests | Исправить запрос / аутентифицироваться |
| **5xx** | Ошибка сервера | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout | Повторить позже / уведомить администратора |

### Стратегии обработки ошибок
┌─────────────────────────────────────────────────────────────────────────────┐
│ СТРАТЕГИИ ОБРАБОТКИ ОШИБОК API │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ 4xx (Ошибка клиента) │
│ ├── 400 Bad Request → Проверить валидацию входных данных │
│ ├── 401 Unauthorized → Запросить аутентификацию / обновить токен │
│ ├── 403 Forbidden → Сообщить о недостатке прав │
│ ├── 404 Not Found → Вернуть None или вызвать исключение │
│ └── 429 Too Many Requests → Реализовать задержку (retry-after) │
│ │
│ 5xx (Ошибка сервера) │
│ ├── 500, 502, 503, 504 → Реализовать повторные попытки (retry) │
│ └── Экспоненциальная задержка + лимит попыток │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

---

## Объект тестирования

### Листинг 1. Расширенный модуль api_client.py

```python
# api_client_advanced.py

import requests
import time
import json
from typing import Dict, List, Optional, Any, Callable
from urllib.parse import urljoin
from functools import wraps
from enum import Enum


class HttpMethod(Enum):
    GET = "GET"
    POST = "POST"
    PUT = "PUT"
    PATCH = "PATCH"
    DELETE = "DELETE"


class APIError(Exception):
    """Базовое исключение для ошибок API."""
    def __init__(self, message: str, status_code: int = None, response: requests.Response = None):
        super().__init__(message)
        self.status_code = status_code
        self.response = response


class ClientError(APIError):
    """Ошибка клиента (4xx)."""
    pass


class ServerError(APIError):
    """Ошибка сервера (5xx)."""
    pass


class RateLimitError(ClientError):
    """Превышение лимита запросов (429)."""
    def __init__(self, message: str, retry_after: int = None):
        super().__init__(message, status_code=429)
        self.retry_after = retry_after


def retry_on_failure(
    max_retries: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    retry_on_status: List[int] = None
):
    """
    Декоратор для повторных попыток при ошибках сервера.
    """
    if retry_on_status is None:
        retry_on_status = [500, 502, 503, 504]
    
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            current_delay = delay
            last_exception = None
            
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except ServerError as e:
                    last_exception = e
                    if e.status_code in retry_on_status and attempt < max_retries - 1:
                        time.sleep(current_delay)
                        current_delay *= backoff
                        continue
                    raise
                except RateLimitError as e:
                    # Для 429 используем заголовок Retry-After
                    if e.retry_after and attempt < max_retries - 1:
                        time.sleep(min(e.retry_after, 60))
                        continue
                    raise
            raise last_exception
        return wrapper
    return decorator


class ResilientAPIClient:
    """
    Устойчивый клиент для работы с REST API с обработкой ошибок.
    """
    
    def __init__(
        self,
        base_url: str,
        api_key: str = None,
        max_retries: int = 3,
        timeout: int = 30
    ):
        self.base_url = base_url.rstrip('/')
        self.api_key = api_key
        self.max_retries = max_retries
        self.timeout = timeout
        self.session = requests.Session()
        
        # Устанавливаем заголовки
        self.session.headers.update({
            "Content-Type": "application/json",
            "Accept": "application/json"
        })
        
        if api_key:
            self.session.headers.update({"X-API-Key": api_key})
    
    def _make_url(self, endpoint: str) -> str:
        """Формирует полный URL."""
        return f"{self.base_url}/{endpoint.lstrip('/')}"
    
    def _handle_response(self, response: requests.Response) -> Any:
        """
        Обрабатывает ответ API, преобразует ошибки в исключения.
        """
        if 200 <= response.status_code < 300:
            if response.status_code == 204:
                return None
            return response.json() if response.content else None
        
        # Обработка ошибок
        error_body = None
        try:
            error_body = response.json()
        except json.JSONDecodeError:
            error_body = {"error": response.text}
        
        message = error_body.get("message", error_body.get("error", "Unknown error"))
        
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            raise RateLimitError(
                f"Rate limit exceeded: {message}",
                retry_after=int(retry_after) if retry_after and retry_after.isdigit() else None
            )
        elif 400 <= response.status_code < 500:
            raise ClientError(message, response.status_code, response)
        elif 500 <= response.status_code < 600:
            raise ServerError(message, response.status_code, response)
        else:
            raise APIError(message, response.status_code, response)
    
    @retry_on_failure(max_retries=3)
    def request(
        self,
        method: HttpMethod,
        endpoint: str,
        params: Dict = None,
        data: Dict = None,
        headers: Dict = None
    ) -> Any:
        """
        Выполняет HTTP-запрос с повторными попытками при ошибках сервера.
        """
        url = self._make_url(endpoint)
        request_headers = self.session.headers.copy()
        if headers:
            request_headers.update(headers)
        
        response = self.session.request(
            method=method.value,
            url=url,
            params=params,
            json=data,
            headers=request_headers,
            timeout=self.timeout
        )
        
        return self._handle_response(response)
    
    # ==========================================================
    # Удобные методы для разных типов запросов
    # ==========================================================
    
    def get(self, endpoint: str, params: Dict = None) -> Any:
        return self.request(HttpMethod.GET, endpoint, params=params)
    
    def post(self, endpoint: str, data: Dict = None) -> Any:
        return self.request(HttpMethod.POST, endpoint, data=data)
    
    def put(self, endpoint: str, data: Dict = None) -> Any:
        return self.request(HttpMethod.PUT, endpoint, data=data)
    
    def patch(self, endpoint: str, data: Dict = None) -> Any:
        return self.request(HttpMethod.PATCH, endpoint, data=data)
    
    def delete(self, endpoint: str) -> Any:
        return self.request(HttpMethod.DELETE, endpoint)
    
    def close(self):
        self.session.close()
    
    
class UserService:
    """Сервис для работы с пользователями через ResilientAPIClient."""
    
    def __init__(self, client: ResilientAPIClient):
        self.client = client
    
    def get_user(self, user_id: int) -> Optional[Dict]:
        """Получает пользователя по ID."""
        try:
            return self.client.get(f"/users/{user_id}")
        except ClientError as e:
            if e.status_code == 404:
                return None
            raise
    
    def get_users(self, limit: int = None, offset: int = None) -> List[Dict]:
        """Получает список пользователей."""
        params = {}
        if limit:
            params["limit"] = limit
        if offset:
            params["offset"] = offset
        return self.client.get("/users", params=params)
    
    def create_user(self, name: str, email: str) -> Dict:
        """Создаёт нового пользователя."""
        data = {"name": name, "email": email}
        return self.client.post("/users", data=data)
    
    def update_user(self, user_id: int, **kwargs) -> Dict:
        """Обновляет пользователя."""
        return self.client.put(f"/users/{user_id}", data=kwargs)
    
    def delete_user(self, user_id: int) -> bool:
        """Удаляет пользователя."""
        try:
            self.client.delete(f"/users/{user_id}")
            return True
        except ClientError as e:
            if e.status_code == 404:
                return False
            raise
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Тестирование 4xx ошибок
Средний (варианты 9-17)	Средние студенты	+ 5xx ошибки, retry механизм
Сложный (варианты 18-25)	Сильные студенты	+ Rate limiting, полный тестовый набор
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться тестировать обработку ошибок клиента (4xx): 400, 401, 403, 404.

Задания для базового уровня
Задание 1.1. Тестирование ошибки 404 (15 минут)
python
# test_error_handling_base.py

import pytest
import requests_mock
from api_client_advanced import ResilientAPIClient, ClientError, UserService


class Test404Errors:
    """Тесты обработки ошибки 404."""
    
    @pytest.fixture
    def client(self):
        return ResilientAPIClient(base_url="https://test.api.com")
    
    @pytest.fixture
    def user_service(self, client):
        return UserService(client)
    
    def test_get_user_404_returns_none(self, user_service):
        """При 404 get_user возвращает None."""
        with requests_mock.Mocker() as m:
            m.get("https://test.api.com/users/999", status_code=404, json={"error": "Not found"})
            
            result = user_service.get_user(999)
            
            assert result is None
    
    def test_delete_user_404_returns_false(self, user_service):
        """При 404 delete_user возвращает False."""
        with requests_mock.Mocker() as m:
            m.delete("https://test.api.com/users/999", status_code=404)
            
            result = user_service.delete_user(999)
            
            assert result is False
    
    def test_update_user_404_raises_exception(self, user_service):
        """При 404 update_user выбрасывает исключение."""
        with requests_mock.Mocker() as m:
            m.put("https://test.api.com/users/999", status_code=404, json={"error": "User not found"})
            
            with pytest.raises(ClientError) as exc_info:
                user_service.update_user(999, name="New Name")
            
            assert exc_info.value.status_code == 404
            assert "User not found" in str(exc_info.value)
Задание 1.2. Тестирование ошибок 400, 401, 403 (15 минут)
python
class TestClientErrors:
    """Тесты ошибок клиента (4xx)."""
    
    @pytest.fixture
    def client(self):
        return ResilientAPIClient(base_url="https://test.api.com")
    
    @pytest.fixture
    def user_service(self, client):
        return UserService(client)
    
    # ==========================================================
    # ТЕСТЫ 400 BAD REQUEST
    # ==========================================================
    
    def test_create_user_400_invalid_email(self, user_service):
        """400 при создании пользователя с неверным email."""
        with requests_mock.Mocker() as m:
            m.post(
                "https://test.api.com/users",
                status_code=400,
                json={"error": "Invalid email format"}
            )
            
            with pytest.raises(ClientError) as exc_info:
                user_service.create_user(name="Test", email="invalid")
            
            assert exc_info.value.status_code == 400
            assert "Invalid email" in str(exc_info.value)
    
    # ==========================================================
    # ТЕСТЫ 401 UNAUTHORIZED
    # ==========================================================
    
    def test_get_users_401_missing_api_key(self):
        """401 при отсутствии API ключа."""
        client = ResilientAPIClient(base_url="https://test.api.com", api_key=None)
        user_service = UserService(client)
        
        with requests_mock.Mocker() as m:
            m.get("https://test.api.com/users", status_code=401, json={"error": "API key required"})
            
            with pytest.raises(ClientError) as exc_info:
                user_service.get_users()
            
            assert exc_info.value.status_code == 401
            assert "API key required" in str(exc_info.value)
    
    # ==========================================================
    # ТЕСТЫ 403 FORBIDDEN
    # ==========================================================
    
    def test_delete_user_403_forbidden(self, user_service):
        """403 при попытке удалить чужого пользователя."""
        with requests_mock.Mocker() as m:
            m.delete(
                "https://test.api.com/users/1",
                status_code=403,
                json={"error": "You don't have permission to delete this user"}
            )
            
            with pytest.raises(ClientError) as exc_info:
                user_service.delete_user(1)
            
            assert exc_info.value.status_code == 403
            assert "permission" in str(exc_info.value).lower()
Задание 1.3. Параметризованное тестирование 4xx ошибок (10 минут)
python
class Test4xxErrorsParametrized:
    """Параметризованные тесты 4xx ошибок."""
    
    @pytest.fixture
    def client(self):
        return ResilientAPIClient(base_url="https://test.api.com")
    
    @pytest.fixture
    def user_service(self, client):
        return UserService(client)
    
    @pytest.mark.parametrize("status_code,error_message", [
        (400, "Bad Request"),
        (401, "Unauthorized"),
        (403, "Forbidden"),
        (404, "Not Found"),
        (405, "Method Not Allowed"),
        (409, "Conflict"),
        (422, "Unprocessable Entity"),
    ])
    def test_client_error_status_codes(self, user_service, status_code, error_message):
        """Параметризованный тест различных 4xx статусов."""
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/users",
                status_code=status_code,
                json={"error": error_message}
            )
            
            with pytest.raises(ClientError) as exc_info:
                user_service.get_users()
            
            assert exc_info.value.status_code == status_code
            assert error_message in str(exc_info.value)
    
    @pytest.mark.parametrize("method,endpoint,request_func", [
        ("get", "/users/1", lambda s: s.get_user(1)),
        ("post", "/users", lambda s: s.create_user("Test", "test@test.com")),
        ("put", "/users/1", lambda s: s.update_user(1, name="New")),
        ("delete", "/users/1", lambda s: s.delete_user(1)),
    ])
    def test_all_methods_404(self, user_service, method, endpoint, request_func):
        """404 для всех методов HTTP."""
        with requests_mock.Mocker() as m:
            m.request(method.upper(), f"https://test.api.com/users/1", status_code=404)
            m.request(method.upper(), f"https://test.api.com/users", status_code=404)
            
            # Для методов, которые обрабатывают 404 особым образом
            if method == "get":
                result = request_func(user_service)
                assert result is None
            elif method == "delete":
                result = request_func(user_service)
                assert result is False
            else:
                with pytest.raises(ClientError) as exc_info:
                    request_func(user_service)
                assert exc_info.value.status_code == 404
Задание 1.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Результаты тестирования 4xx ошибок

| Статус | GET /users/{id} | GET /users | POST /users | PUT /users/{id} | DELETE /users/{id} |
|:---|:---|:---|:---|:---|:---|
| 400 | ☐ | ☐ | ☐ | ☐ | ☐ |
| 401 | ☐ | ☐ | ☐ | ☐ | ☐ |
| 403 | ☐ | ☐ | ☐ | ☐ | ☐ |
| 404 | ☐ | ☐ | ☐ | ☐ | ☐ |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться тестировать ошибки сервера (5xx) и механизм повторных попыток (retry).

Задания для среднего уровня
Задание 2.1. Тестирование 5xx ошибок (15 минут)
python
# test_error_handling_intermediate.py

import pytest
import requests_mock
from api_client_advanced import ResilientAPIClient, ServerError, UserService


class Test5xxErrors:
    """Тесты ошибок сервера (5xx)."""
    
    @pytest.fixture
    def client(self):
        return ResilientAPIClient(base_url="https://test.api.com", max_retries=1)
    
    @pytest.fixture
    def user_service(self, client):
        return UserService(client)
    
    @pytest.mark.parametrize("status_code", [500, 502, 503, 504])
    def test_server_error_statuses(self, user_service, status_code):
        """Параметризованный тест 5xx статусов."""
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/users",
                status_code=status_code,
                json={"error": f"Server error {status_code}"}
            )
            
            with pytest.raises(ServerError) as exc_info:
                user_service.get_users()
            
            assert exc_info.value.status_code == status_code
    
    def test_server_error_with_body(self, user_service):
        """5xx ошибка с детальным телом ответа."""
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/users",
                status_code=503,
                json={
                    "error": "Service Unavailable",
                    "message": "Database connection failed",
                    "retry_after": 30
                }
            )
            
            with pytest.raises(ServerError) as exc_info:
                user_service.get_users()
            
            assert exc_info.value.status_code == 503
            assert "Database connection failed" in str(exc_info.value)
Задание 2.2. Тестирование retry механизма (20 минут)
python
class TestRetryMechanism:
    """Тесты механизма повторных попыток."""
    
    def test_retry_on_500_success_after_retry(self):
        """Повторная попытка после 500 -> успех."""
        client = ResilientAPIClient(base_url="https://test.api.com", max_retries=3)
        user_service = UserService(client)
        
        with requests_mock.Mocker() as m:
            # Первый запрос: 500
            # Второй запрос: 200
            m.get(
                "https://test.api.com/users/1",
                [
                    {"status_code": 500, "json": {"error": "Internal error"}},
                    {"status_code": 200, "json": {"id": 1, "name": "User"}}
                ]
            )
            
            result = user_service.get_user(1)
            
            assert result is not None
            assert result["name"] == "User"
            assert m.call_count == 2
    
    def test_retry_on_503_exhausted(self):
        """Все попытки исчерпаны -> исключение."""
        client = ResilientAPIClient(base_url="https://test.api.com", max_retries=2)
        user_service = UserService(client)
        
        with requests_mock.Mocker() as m:
            # Все попытки возвращают 503
            m.get(
                "https://test.api.com/users/1",
                [
                    {"status_code": 503, "json": {"error": "Service unavailable"}},
                    {"status_code": 503, "json": {"error": "Service unavailable"}},
                ]
            )
            
            with pytest.raises(ServerError) as exc_info:
                user_service.get_user(1)
            
            assert exc_info.value.status_code == 503
            assert m.call_count == 2
    
    def test_retry_only_on_certain_statuses(self):
        """Повторные попытки только для определённых статусов."""
        client = ResilientAPIClient(base_url="https://test.api.com", max_retries=3)
        
        with requests_mock.Mocker() as m:
            # 400 не должен вызывать повторную попытку
            m.get("https://test.api.com/users/1", status_code=400)
            
            with pytest.raises(Exception):
                client.get("/users/1")
            
            # Проверяем, что была только одна попытка
            assert m.call_count == 1
    
    def test_retry_with_exponential_backoff(self, mocker):
        """Проверка экспоненциальной задержки."""
        import time
        
        client = ResilientAPIClient(base_url="https://test.api.com", max_retries=3)
        
        # Мокируем time.sleep
        mock_sleep = mocker.patch("time.sleep")
        
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/users/1",
                [
                    {"status_code": 500},
                    {"status_code": 500},
                    {"status_code": 200, "json": {"id": 1}}
                ]
            )
            
            client.get("/users/1")
            
            # Проверяем задержки: delay, delay*backoff, ...
            assert mock_sleep.call_count == 2
            # Первая задержка ~1, вторая ~2
            assert mock_sleep.call_args_list[0][0][0] == 1.0
            assert mock_sleep.call_args_list[1][0][0] == 2.0
Задание 2.3. Комбинированные сценарии (10 минут)
python
class TestMixedErrorScenarios:
    """Комбинированные сценарии с разными ошибками."""
    
    def test_404_then_500_then_200(self):
        """404 -> 500 -> 200."""
        client = ResilientAPIClient(base_url="https://test.api.com", max_retries=3)
        
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/users/1",
                [
                    {"status_code": 404},
                    {"status_code": 500},
                    {"status_code": 200, "json": {"id": 1}}
                ]
            )
            
            # 404 не вызывает retry, сразу исключение
            with pytest.raises(Exception):
                client.get("/users/1")
            
            assert m.call_count == 1
    
    def test_500_then_400_then_200(self):
        """500 -> 400 -> 200."""
        client = ResilientAPIClient(base_url="https://test.api.com", max_retries=3)
        
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/users/1",
                [
                    {"status_code": 500},
                    {"status_code": 400},
                ]
            )
            
            # Первая попытка 500 -> retry
            # Вторая попытка 400 -> исключение (не retry)
            with pytest.raises(Exception):
                client.get("/users/1")
            
            assert m.call_count == 2
    
    def test_success_after_retry(self):
        """Успех после повторной попытки."""
        client = ResilientAPIClient(base_url="https://test.api.com", max_retries=3)
        
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/users/1",
                [
                    {"status_code": 502, "json": {"error": "Bad Gateway"}},
                    {"status_code": 200, "json": {"id": 1}}
                ]
            )
            
            result = client.get("/users/1")
            
            assert result["id"] == 1
            assert m.call_count == 2
Задание 2.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Результаты тестирования 5xx ошибок

| Статус | GET | POST | PUT | DELETE |
|:---|:---|:---|:---|:---|
| 500 | ☐ | ☐ | ☐ | ☐ |
| 502 | ☐ | ☐ | ☐ | ☐ |
| 503 | ☐ | ☐ | ☐ | ☐ |
| 504 | ☐ | ☐ | ☐ | ☐ |

### Проверка retry механизма

| Сценарий | Результат |
|:---|:---|
| 500 → 200 (успех после retry) | ☐ |
| 500 → 500 → 500 (все попытки неудачны) | ☐ |
| 400 (без retry) | ☐ |
| Экспоненциальная задержка | ☐ |

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться тестировать rate limiting (429) и разрабатывать полный комплексный тестовый набор.

Задания для сложного уровня
Задание 3.1. Тестирование Rate Limiting (429) (15 минут)
python
# test_error_handling_advanced.py

import pytest
import requests_mock
from api_client_advanced import ResilientAPIClient, RateLimitError, UserService


class TestRateLimiting:
    """Тесты rate limiting (429)."""
    
    @pytest.fixture
    def client(self):
        return ResilientAPIClient(base_url="https://test.api.com", max_retries=3)
    
    @pytest.fixture
    def user_service(self, client):
        return UserService(client)
    
    def test_rate_limit_with_retry_after(self, user_service):
        """429 с заголовком Retry-After."""
        import time
        
        with requests_mock.Mocker() as m:
            # Первый запрос: 429 с Retry-After
            m.get(
                "https://test.api.com/users",
                [
                    {
                        "status_code": 429,
                        "headers": {"Retry-After": "1"},
                        "json": {"error": "Too many requests"}
                    },
                    {
                        "status_code": 200,
                        "json": [{"id": 1, "name": "User"}]
                    }
                ]
            )
            
            # Мокируем time.sleep
            start = time.time()
            result = user_service.get_users()
            elapsed = time.time() - start
            
            assert result is not None
            assert elapsed >= 1.0  # Должна быть задержка
    
    def test_rate_limit_no_retry_after(self, user_service):
        """429 без заголовка Retry-After."""
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/users",
                status_code=429,
                json={"error": "Rate limit exceeded"}
            )
            
            with pytest.raises(RateLimitError) as exc_info:
                user_service.get_users()
            
            assert exc_info.value.status_code == 429
            assert exc_info.value.retry_after is None
    
    def test_rate_limit_with_retry_exhausted(self, user_service):
        """Rate limit, все попытки исчерпаны."""
        with requests_mock.Mocker() as m:
            # Все попытки возвращают 429
            m.get(
                "https://test.api.com/users",
                [
                    {"status_code": 429, "headers": {"Retry-After": "1"}},
                    {"status_code": 429, "headers": {"Retry-After": "2"}},
                    {"status_code": 429, "headers": {"Retry-After": "3"}},
                ]
            )
            
            with pytest.raises(RateLimitError) as exc_info:
                user_service.get_users()
            
            assert exc_info.value.status_code == 429
            assert m.call_count == 3
Задание 3.2. Комплексный тестовый набор (20 минут)
python
class TestComprehensiveTestSuite:
    """Комплексный тестовый набор для клиента."""
    
    @pytest.fixture
    def client(self):
        return ResilientAPIClient(base_url="https://test.api.com", max_retries=2)
    
    @pytest.fixture
    def user_service(self, client):
        return UserService(client)
    
    # ==========================================================
    # СЧАСТЛИВЫЙ ПУТЬ (HAPPY PATH)
    # ==========================================================
    
    def test_happy_path_get_users(self, user_service):
        with requests_mock.Mocker() as m:
            m.get("https://test.api.com/users", json=[{"id": 1}, {"id": 2}])
            result = user_service.get_users()
            assert len(result) == 2
    
    def test_happy_path_get_user_by_id(self, user_service):
        with requests_mock.Mocker() as m:
            m.get("https://test.api.com/users/1", json={"id": 1, "name": "Alice"})
            result = user_service.get_user(1)
            assert result["name"] == "Alice"
    
    def test_happy_path_create_user(self, user_service):
        with requests_mock.Mocker() as m:
            m.post("https://test.api.com/users", json={"id": 100, "name": "Bob"}, status_code=201)
            result = user_service.create_user("Bob", "bob@test.com")
            assert result["id"] == 100
    
    def test_happy_path_update_user(self, user_service):
        with requests_mock.Mocker() as m:
            m.put("https://test.api.com/users/1", json={"id": 1, "name": "Updated"})
            result = user_service.update_user(1, name="Updated")
            assert result["name"] == "Updated"
    
    def test_happy_path_delete_user(self, user_service):
        with requests_mock.Mocker() as m:
            m.delete("https://test.api.com/users/1", status_code=200)
            result = user_service.delete_user(1)
            assert result is True
    
    # ==========================================================
    # ОШИБКИ КЛИЕНТА (4xx)
    # ==========================================================
    
    @pytest.mark.parametrize("method,func,args", [
        ("get", lambda s: s.get_user(1), []),
        ("get_list", lambda s: s.get_users(), []),
        ("post", lambda s: s.create_user("Test", "test@test.com"), []),
        ("put", lambda s: s.update_user(1, name="New"), []),
        ("delete", lambda s: s.delete_user(1), []),
    ])
    def test_all_methods_400(self, user_service, method, func, args):
        with requests_mock.Mocker() as m:
            m.request("GET", "https://test.api.com/users", status_code=400)
            m.request("POST", "https://test.api.com/users", status_code=400)
            m.request("PUT", "https://test.api.com/users/1", status_code=400)
            m.request("DELETE", "https://test.api.com/users/1", status_code=400)
            m.request("GET", "https://test.api.com/users/1", status_code=400)
            
            with pytest.raises(Exception):
                func(user_service)
    
    # ==========================================================
    # ОШИБКИ СЕРВЕРА (5xx)
    # ==========================================================
    
    @pytest.mark.parametrize("status_code", [500, 502, 503, 504])
    def test_server_error_retry_mechanism(self, user_service, status_code):
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/users",
                [
                    {"status_code": status_code},
                    {"status_code": 200, "json": []}
                ]
            )
            
            result = user_service.get_users()
            
            assert result == []
            assert m.call_count == 2
    
    # ==========================================================
    # ГРАНИЧНЫЕ СЛУЧАИ
    # ==========================================================
    
    def test_empty_response_204(self, user_service):
        with requests_mock.Mocker() as m:
            m.delete("https://test.api.com/users/1", status_code=204)
            result = user_service.delete_user(1)
            assert result is True
    
    def test_malformed_json_response(self, user_service):
        with requests_mock.Mocker() as m:
            m.get("https://test.api.com/users", text="Not a JSON", status_code=200)
            
            # Должно быть исключение при разборе JSON
            with pytest.raises(Exception):
                user_service.get_users()
    
    def test_empty_json_response(self, user_service):
        with requests_mock.Mocker() as m:
            m.get("https://test.api.com/users", text="", status_code=200)
            
            result = user_service.get_users()
            assert result is None
Задание 3.3. Тестирование с параметрами запросов (10 минут)
python
class TestQueryParameters:
    """Тесты с параметрами запросов."""
    
    @pytest.fixture
    def client(self):
        return ResilientAPIClient(base_url="https://test.api.com")
    
    @pytest.fixture
    def user_service(self, client):
        return UserService(client)
    
    def test_get_users_with_limit_and_offset(self, user_service):
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/users",
                json=[{"id": 10}, {"id": 11}]
            )
            
            result = user_service.get_users(limit=2, offset=10)
            
            last_request = m.request_history[-1]
            assert last_request.qs.get("limit") == ["2"]
            assert last_request.qs.get("offset") == ["10"]
            assert len(result) == 2
    
    def test_get_users_pagination_scenarios(self, user_service):
        """Сценарии пагинации с разными параметрами."""
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/users",
                [
                    {"json": [{"id": 1}, {"id": 2}], "status_code": 200},
                    {"json": [{"id": 3}, {"id": 4}], "status_code": 200},
                    {"json": [{"id": 5}], "status_code": 200},
                    {"json": [], "status_code": 200},
                ]
            )
            
            # Страница 1
            page1 = user_service.get_users(limit=2, offset=0)
            assert len(page1) == 2
            
            # Страница 2
            page2 = user_service.get_users(limit=2, offset=2)
            assert len(page2) == 2
            
            # Страница 3
            page3 = user_service.get_users(limit=2, offset=4)
            assert len(page3) == 1
            
            # Пустая страница
            page4 = user_service.get_users(limit=2, offset=6)
            assert len(page4) == 0
Задание 3.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (сложный уровень)

**Студент:** _________________
**Вариант №:** ___ (18-25)

### Комплексные результаты тестирования

| Категория | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| Happy Path (2xx) | _____ | _____ | _____ |
| Client Errors (4xx) | _____ | _____ | _____ |
| Server Errors (5xx) | _____ | _____ | _____ |
| Rate Limiting (429) | _____ | _____ | _____ |
| Retry Mechanism | _____ | _____ | _____ |
| Edge Cases | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Покрытие статусов

| Категория | Покрытие |
|:---|:---|
| 2xx (200, 201, 204) | ☐ |
| 4xx (400-429) | ☐ |
| 5xx (500-504) | ☐ |

### Выводы

_______________________________________________________________

_______________________________________________________________
Карточка студента
text
ПР 3.2. ТЕСТИРОВАНИЕ СЦЕНАРИЕВ ОТКАЗА ВНЕШНЕГО API

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ 4xx ошибки (базовый)
□ 404 обработка (базовый)
□ 5xx ошибки (средний)
□ Retry механизм (средний)
□ Rate limiting (429) (сложный)
□ Комплексный тестовый набор (сложный)

=== ПРОВЕРЕННЫЕ СТАТУСЫ ===

□ 200, 201, 204
□ 400, 401, 403, 404
□ 429
□ 500, 502, 503, 504

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
- test_error_handling_base.py
- test_error_handling_intermediate.py
- test_error_handling_advanced.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Тесты не работают или не используют requests-mock
3 (удовлетворительно)	Базовый	4xx ошибки + 404 обработка (10+ тестов)
4 (хорошо)	Средний	+ 5xx ошибки + retry механизм (20+ тестов)
5 (отлично)	Сложный	+ Rate limiting + полный тестовый набор (35+ тестов)
Контрольные вопросы (для защиты)
Чем отличаются ошибки 4xx от 5xx?

Как клиент должен обрабатывать ошибку 404?

Почему ошибки 5xx требуют повторных попыток, а 4xx — нет?

Как реализовать экспоненциальную задержку при повторных попытках?

Как тестировать заголовок Retry-After при ошибке 429?

Какие статусы должны вызывать повторную попытку, а какие — нет?

Как проверить, что при исчерпании всех попыток выбрасывается исключение?

Как мокировать последовательность ответов (ошибка → успех)?

Как тестировать обработку некорректного JSON в ответе?

Как организовать полный тестовый набор для REST API клиента?

Следующее занятие: Лекция 3.3 — Тестирование асинхронных HTTP-клиентов (aiohttp).
