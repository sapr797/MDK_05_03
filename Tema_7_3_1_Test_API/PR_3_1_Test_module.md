# Практическое занятие 3.1: Написание тестов для модуля, отправляющего HTTP-запросы, с использованием requests-mock

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.1:** Тестирование API-клиентов и сетевых запросов  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться писать тесты для модулей, отправляющих HTTP-запросы, с использованием библиотеки `requests-mock`. Освоить мокирование различных типов запросов (GET, POST, PUT, DELETE), тестирование обработки HTTP-статусов и сетевых ошибок.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Использовать `requests-mock` для мокирования HTTP-запросов (ПК 3.2, ПК 3.3).
2. Тестировать обработку различных HTTP-статусов (200, 400, 401, 404, 500) (ПК 3.3).
3. Мокировать GET, POST, PUT, DELETE запросы с разными телами ответа (ПК 3.2).
4. Тестировать обработку сетевых ошибок и таймаутов (ПК 3.3).

---

## Теоретическая справка

### Основы requests-mock

```python
import requests_mock

# Базовое использование
with requests_mock.Mocker() as m:
    m.get("https://api.example.com/users", json={"users": []})
    response = requests.get("https://api.example.com/users")
    assert response.status_code == 200
Основные методы requests-mock
Метод	Назначение	Пример
m.get(url, ...)	Мокирование GET-запроса	m.get("/users", json={"id": 1})
m.post(url, ...)	Мокирование POST-запроса	m.post("/users", status_code=201)
m.put(url, ...)	Мокирование PUT-запроса	m.put("/users/1", json={"updated": True})
m.delete(url, ...)	Мокирование DELETE-запроса	m.delete("/users/1", status_code=204)
m.patch(url, ...)	Мокирование PATCH-запроса	m.patch("/users/1", json={"patched": True})
Параметры мокирования
Параметр	Описание	Пример
status_code	HTTP-статус ответа	status_code=201
json	JSON-тело ответа	json={"name": "Alice"}
text	Текстовое тело ответа	text="Hello World"
content	Бинарное тело ответа	content=b"binary data"
headers	Заголовки ответа	headers={"X-Custom": "value"}
exc	Исключение для эмуляции ошибки	exc=requests.Timeout()
Объект тестирования
Листинг 1. Модуль api_client.py
python
# api_client.py

import requests
from typing import Dict, List, Optional, Any
from urllib.parse import urljoin


class TodoClient:
    """
    Клиент для работы с JSONPlaceholder API (тестовое REST API).
    """
    
    BASE_URL = "https://jsonplaceholder.typicode.com"
    
    def __init__(self, base_url: str = None):
        self.base_url = base_url or self.BASE_URL
        self.session = requests.Session()
        self.session.headers.update({
            "Content-Type": "application/json"
        })
    
    def _make_url(self, endpoint: str) -> str:
        """Формирует полный URL."""
        return urljoin(self.base_url, endpoint)
    
    def _handle_response(self, response: requests.Response) -> Any:
        """
        Обрабатывает ответ API.
        
        Raises:
            requests.HTTPError: При HTTP-ошибках (4xx, 5xx)
        """
        if response.status_code >= 400:
            response.raise_for_status()
        return response.json() if response.content else None
    
    # ==========================================================
    # GET-запросы
    # ==========================================================
    
    def get_todos(self, limit: int = None) -> List[Dict]:
        """
        Получает список задач.
        
        Args:
            limit: Максимальное количество задач
        
        Returns:
            Список задач
        """
        url = self._make_url("/todos")
        params = {"_limit": limit} if limit else None
        
        response = self.session.get(url, params=params)
        return self._handle_response(response)
    
    def get_todo(self, todo_id: int) -> Optional[Dict]:
        """
        Получает задачу по ID.
        
        Returns:
            Задача или None, если не найдена
        """
        url = self._make_url(f"/todos/{todo_id}")
        
        try:
            response = self.session.get(url)
            if response.status_code == 404:
                return None
            return self._handle_response(response)
        except requests.HTTPError as e:
            if e.response.status_code == 404:
                return None
            raise
    
    def get_user_todos(self, user_id: int) -> List[Dict]:
        """Получает задачи пользователя."""
        url = self._make_url("/todos")
        params = {"userId": user_id}
        
        response = self.session.get(url, params=params)
        return self._handle_response(response)
    
    # ==========================================================
    # POST-запросы
    # ==========================================================
    
    def create_todo(self, title: str, completed: bool = False, user_id: int = 1) -> Dict:
        """
        Создаёт новую задачу.
        
        Returns:
            Созданная задача с ID
        """
        url = self._make_url("/todos")
        data = {
            "title": title,
            "completed": completed,
            "userId": user_id
        }
        
        response = self.session.post(url, json=data)
        return self._handle_response(response)
    
    # ==========================================================
    # PUT-запросы
    # ==========================================================
    
    def update_todo(self, todo_id: int, **kwargs) -> Optional[Dict]:
        """
        Обновляет задачу.
        
        Args:
            todo_id: ID задачи
            **kwargs: Поля для обновления (title, completed, userId)
        
        Returns:
            Обновлённая задача или None, если не найдена
        """
        url = self._make_url(f"/todos/{todo_id}")
        
        response = self.session.put(url, json=kwargs)
        
        if response.status_code == 404:
            return None
        return self._handle_response(response)
    
    # ==========================================================
    # PATCH-запросы
    # ==========================================================
    
    def patch_todo(self, todo_id: int, **kwargs) -> Optional[Dict]:
        """Частичное обновление задачи."""
        url = self._make_url(f"/todos/{todo_id}")
        
        response = self.session.patch(url, json=kwargs)
        
        if response.status_code == 404:
            return None
        return self._handle_response(response)
    
    # ==========================================================
    # DELETE-запросы
    # ==========================================================
    
    def delete_todo(self, todo_id: int) -> bool:
        """
        Удаляет задачу.
        
        Returns:
            True если удаление успешно
        """
        url = self._make_url(f"/todos/{todo_id}")
        
        response = self.session.delete(url)
        
        if response.status_code == 404:
            return False
        response.raise_for_status()
        return response.status_code == 200
    
    # ==========================================================
    # Дополнительные методы
    # ==========================================================
    
    def get_todos_batch(self, todo_ids: List[int]) -> List[Dict]:
        """
        Получает несколько задач (разные запросы).
        
        Args:
            todo_ids: Список ID задач
        
        Returns:
            Список найденных задач
        """
        results = []
        for todo_id in todo_ids:
            todo = self.get_todo(todo_id)
            if todo:
                results.append(todo)
        return results
    
    def close(self):
        """Закрывает сессию."""
        self.session.close()
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Тестирование GET и POST запросов
Средний (варианты 9-17)	Средние студенты	+ PUT, DELETE, обработка статусов
Сложный (варианты 18-25)	Сильные студенты	+ пакетные операции, ошибки, таймауты
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться мокировать GET и POST запросы, проверять параметры и обработку успешных ответов.

Задания для базового уровня
Задание 1.1. Тестирование GET-запросов (20 минут)
python
# test_api_client_base.py

import pytest
import requests_mock
from api_client import TodoClient


class TestTodoClientGet:
    """Тесты GET-запросов TodoClient."""
    
    @pytest.fixture
    def client(self):
        """Фикстура с клиентом."""
        return TodoClient(base_url="https://test.api.com")
    
    # ==========================================================
    # ТЕСТЫ get_todos
    # ==========================================================
    
    def test_get_todos_success(self, client):
        """Успешное получение списка задач."""
        with requests_mock.Mocker() as m:
            # Мокируем GET-запрос
            mock_response = [
                {"id": 1, "title": "Task 1", "completed": False},
                {"id": 2, "title": "Task 2", "completed": True},
            ]
            m.get("https://test.api.com/todos", json=mock_response, status_code=200)
            
            result = client.get_todos()
            
            assert len(result) == 2
            assert result[0]["id"] == 1
            assert result[0]["title"] == "Task 1"
            assert result[1]["id"] == 2
    
    def test_get_todos_with_limit(self, client):
        """Получение задач с ограничением количества."""
        with requests_mock.Mocker() as m:
            mock_response = [{"id": i, "title": f"Task {i}"} for i in range(1, 6)]
            m.get("https://test.api.com/todos", json=mock_response, status_code=200)
            
            result = client.get_todos(limit=3)
            
            # Проверяем, что запрос был с параметром _limit=3
            assert len(result) == 5  # Мок возвращает 5, ограничение не применяется на клиенте
            # Проверяем, что параметр был передан
            last_request = m.request_history[-1]
            assert last_request.qs.get("_limit") == ["3"]
    
    def test_get_todos_empty(self, client):
        """Пустой список задач."""
        with requests_mock.Mocker() as m:
            m.get("https://test.api.com/todos", json=[], status_code=200)
            
            result = client.get_todos()
            
            assert result == []
    
    # ==========================================================
    # ТЕСТЫ get_todo
    # ==========================================================
    
    def test_get_todo_by_id_success(self, client):
        """Получение задачи по ID."""
        with requests_mock.Mocker() as m:
            mock_response = {"id": 42, "title": "Special Task", "completed": False}
            m.get("https://test.api.com/todos/42", json=mock_response, status_code=200)
            
            result = client.get_todo(42)
            
            assert result is not None
            assert result["id"] == 42
            assert result["title"] == "Special Task"
    
    def test_get_todo_not_found(self, client):
        """Задача не найдена."""
        with requests_mock.Mocker() as m:
            m.get("https://test.api.com/todos/999", status_code=404)
            
            result = client.get_todo(999)
            
            assert result is None
    
    # ==========================================================
    # ТЕСТЫ get_user_todos
    # ==========================================================
    
    def test_get_user_todos(self, client):
        """Получение задач пользователя."""
        with requests_mock.Mocker() as m:
            mock_response = [
                {"id": 1, "title": "User 1 Task", "userId": 1},
                {"id": 2, "title": "User 1 Task 2", "userId": 1},
            ]
            m.get("https://test.api.com/todos", json=mock_response, status_code=200)
            
            result = client.get_user_todos(user_id=1)
            
            assert len(result) == 2
            assert result[0]["userId"] == 1
            assert result[1]["userId"] == 1
            
            # Проверяем параметр userId
            last_request = m.request_history[-1]
            assert last_request.qs.get("userId") == ["1"]
Задание 1.2. Тестирование POST-запросов (15 минут)
python
class TestTodoClientPost:
    """Тесты POST-запросов TodoClient."""
    
    @pytest.fixture
    def client(self):
        return TodoClient(base_url="https://test.api.com")
    
    def test_create_todo_success(self, client):
        """Успешное создание задачи."""
        with requests_mock.Mocker() as m:
            # Мокируем POST-запрос
            mock_response = {
                "id": 201,
                "title": "New Task",
                "completed": False,
                "userId": 1
            }
            m.post("https://test.api.com/todos", json=mock_response, status_code=201)
            
            result = client.create_todo(title="New Task")
            
            assert result["id"] == 201
            assert result["title"] == "New Task"
            assert result["completed"] is False
            
            # Проверяем тело запроса
            last_request = m.request_history[-1]
            assert last_request.json()["title"] == "New Task"
    
    def test_create_todo_with_completed_status(self, client):
        """Создание завершённой задачи."""
        with requests_mock.Mocker() as m:
            mock_response = {"id": 202, "title": "Completed Task", "completed": True, "userId": 1}
            m.post("https://test.api.com/todos", json=mock_response, status_code=201)
            
            result = client.create_todo(title="Completed Task", completed=True)
            
            assert result["completed"] is True
            
            last_request = m.request_history[-1]
            assert last_request.json()["completed"] is True
    
    def test_create_todo_with_custom_user(self, client):
        """Создание задачи для конкретного пользователя."""
        with requests_mock.Mocker() as m:
            mock_response = {"id": 203, "title": "User Task", "completed": False, "userId": 5}
            m.post("https://test.api.com/todos", json=mock_response, status_code=201)
            
            result = client.create_todo(title="User Task", user_id=5)
            
            assert result["userId"] == 5
            
            last_request = m.request_history[-1]
            assert last_request.json()["userId"] == 5
Задание 1.3. Проверка параметров запроса (10 минут)
python
class TestRequestParameters:
    """Тесты проверки параметров запросов."""
    
    @pytest.fixture
    def client(self):
        return TodoClient(base_url="https://test.api.com")
    
    def test_get_todo_has_correct_headers(self, client):
        """Проверка заголовков запроса."""
        with requests_mock.Mocker() as m:
            m.get("https://test.api.com/todos/1", json={"id": 1}, status_code=200)
            
            client.get_todo(1)
            
            last_request = m.request_history[-1]
            assert "Content-Type" in last_request.headers
            assert last_request.headers["Content-Type"] == "application/json"
    
    def test_create_todo_sends_json_body(self, client):
        """Проверка тела POST-запроса."""
        with requests_mock.Mocker() as m:
            m.post("https://test.api.com/todos", json={"id": 1}, status_code=201)
            
            client.create_todo(title="Test", completed=True, user_id=10)
            
            last_request = m.request_history[-1]
            body = last_request.json()
            
            assert body["title"] == "Test"
            assert body["completed"] is True
            assert body["userId"] == 10
Задание 1.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Результаты тестирования

| Метод | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| GET /todos | _____ | _____ | _____ |
| GET /todos/{id} | _____ | _____ | _____ |
| GET /todos (user filter) | _____ | _____ | _____ |
| POST /todos | _____ | _____ | _____ |

### Проверенные сценарии

| Сценарий | Статус |
|:---|:---|
| Успешное получение списка | ☐ |
| Получение с limit | ☐ |
| Получение по ID (успех) | ☐ |
| Получение по ID (404) | ☐ |
| Создание задачи | ☐ |
| Проверка заголовков | ☐ |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться мокировать PUT, PATCH, DELETE запросы и тестировать обработку различных HTTP-статусов.

Задания для среднего уровня
Задание 2.1. Тестирование PUT и PATCH запросов (20 минут)
python
# test_api_client_intermediate.py

import pytest
import requests_mock
from api_client import TodoClient


class TestTodoClientPutPatch:
    """Тесты PUT и PATCH запросов."""
    
    @pytest.fixture
    def client(self):
        return TodoClient(base_url="https://test.api.com")
    
    # ==========================================================
    # ТЕСТЫ PUT
    # ==========================================================
    
    def test_update_todo_success(self, client):
        """Успешное обновление задачи."""
        with requests_mock.Mocker() as m:
            mock_response = {
                "id": 1,
                "title": "Updated Title",
                "completed": True,
                "userId": 1
            }
            m.put("https://test.api.com/todos/1", json=mock_response, status_code=200)
            
            result = client.update_todo(1, title="Updated Title", completed=True)
            
            assert result is not None
            assert result["title"] == "Updated Title"
            assert result["completed"] is True
            
            # Проверяем тело запроса
            last_request = m.request_history[-1]
            assert last_request.json()["title"] == "Updated Title"
    
    def test_update_todo_not_found(self, client):
        """Обновление несуществующей задачи."""
        with requests_mock.Mocker() as m:
            m.put("https://test.api.com/todos/999", status_code=404)
            
            result = client.update_todo(999, title="New Title")
            
            assert result is None
    
    def test_update_todo_partial_fields(self, client):
        """Обновление только некоторых полей."""
        with requests_mock.Mocker() as m:
            mock_response = {"id": 1, "title": "New Title Only", "completed": False, "userId": 1}
            m.put("https://test.api.com/todos/1", json=mock_response, status_code=200)
            
            result = client.update_todo(1, title="New Title Only")
            
            assert result["title"] == "New Title Only"
            assert result["completed"] is False  # Не изменилось
    
    # ==========================================================
    # ТЕСТЫ PATCH
    # ==========================================================
    
    def test_patch_todo_success(self, client):
        """Частичное обновление задачи."""
        with requests_mock.Mocker() as m:
            mock_response = {"id": 1, "title": "Patched Title", "completed": False, "userId": 1}
            m.patch("https://test.api.com/todos/1", json=mock_response, status_code=200)
            
            result = client.patch_todo(1, title="Patched Title")
            
            assert result["title"] == "Patched Title"
    
    def test_patch_todo_not_found(self, client):
        """PATCH для несуществующей задачи."""
        with requests_mock.Mocker() as m:
            m.patch("https://test.api.com/todos/999", status_code=404)
            
            result = client.patch_todo(999, title="New")
            
            assert result is None
    
    # ==========================================================
    # ТЕСТЫ DELETE
    # ==========================================================
    
    def test_delete_todo_success(self, client):
        """Успешное удаление задачи."""
        with requests_mock.Mocker() as m:
            m.delete("https://test.api.com/todos/1", status_code=200)
            
            result = client.delete_todo(1)
            
            assert result is True
    
    def test_delete_todo_not_found(self, client):
        """Удаление несуществующей задачи."""
        with requests_mock.Mocker() as m:
            m.delete("https://test.api.com/todos/999", status_code=404)
            
            result = client.delete_todo(999)
            
            assert result is False
Задание 2.2. Тестирование обработки HTTP-статусов (15 минут)
python
class TestHttpStatusHandling:
    """Тесты обработки различных HTTP-статусов."""
    
    @pytest.fixture
    def client(self):
        return TodoClient(base_url="https://test.api.com")
    
    @pytest.mark.parametrize("status_code,expected_raises", [
        (400, True),
        (401, True),
        (403, True),
        (500, True),
        (502, True),
        (503, True),
        (200, False),
        (201, False),
    ])
    def test_get_todos_status_codes(self, client, status_code, expected_raises):
        """Параметризованный тест HTTP-статусов для GET."""
        import requests
        
        with requests_mock.Mocker() as m:
            m.get("https://test.api.com/todos", status_code=status_code)
            
            if expected_raises:
                with pytest.raises(requests.HTTPError):
                    client.get_todos()
            else:
                # Для успешных статусов тело может быть пустым
                result = client.get_todos()
                # response.json() вызовет ошибку при пустом теле
                # в реальном API при 200 всегда есть тело
                pass
    
    @pytest.mark.parametrize("status_code,expected", [
        (200, {"id": 1}),
        (404, None),
        (500, None),  # Будет исключение
    ])
    def test_get_todo_status_codes(self, client, status_code, expected):
        """Параметризованный тест для get_todo."""
        import requests
        
        with requests_mock.Mocker() as m:
            m.get("https://test.api.com/todos/1", status_code=status_code)
            
            if status_code == 500:
                with pytest.raises(requests.HTTPError):
                    client.get_todo(1)
            else:
                result = client.get_todo(1)
                assert result == expected
    
    @pytest.mark.parametrize("status_code,expected", [
        (201, True),
        (400, True),  # Должно быть исключение
        (401, True),
        (500, True),
    ])
    def test_create_todo_status_codes(self, client, status_code, expected):
        """Параметризованный тест для create_todo."""
        import requests
        
        with requests_mock.Mocker() as m:
            m.post("https://test.api.com/todos", status_code=status_code, json={})
            
            if status_code >= 400:
                with pytest.raises(requests.HTTPError):
                    client.create_todo(title="Test")
            else:
                result = client.create_todo(title="Test")
                assert result is not None
Задание 2.3. Тестирование мокирования с параметрами (10 минут)
python
class TestRequestMatching:
    """Тесты точного сопоставления запросов."""
    
    @pytest.fixture
    def client(self):
        return TodoClient(base_url="https://test.api.com")
    
    def test_different_responses_for_different_ids(self, client):
        """Разные ответы для разных ID."""
        with requests_mock.Mocker() as m:
            m.get("https://test.api.com/todos/1", json={"id": 1, "title": "Task 1"})
            m.get("https://test.api.com/todos/2", json={"id": 2, "title": "Task 2"})
            m.get("https://test.api.com/todos/3", json={"id": 3, "title": "Task 3"})
            
            todo1 = client.get_todo(1)
            todo2 = client.get_todo(2)
            todo3 = client.get_todo(3)
            
            assert todo1["title"] == "Task 1"
            assert todo2["title"] == "Task 2"
            assert todo3["title"] == "Task 3"
    
    def test_url_with_query_params(self, client):
        """URL с параметрами запроса."""
        with requests_mock.Mocker() as m:
            # Мокируем конкретный URL с параметрами
            m.get(
                "https://test.api.com/todos?userId=5",
                json=[{"id": 1, "userId": 5}]
            )
            
            result = client.get_user_todos(user_id=5)
            
            assert len(result) == 1
            assert result[0]["userId"] == 5
    
    def test_post_with_body_matching(self, client):
        """POST с проверкой тела запроса."""
        with requests_mock.Mocker() as m:
            def match_json(request):
                return request.json().get("title") == "Special"
            
            m.post(
                "https://test.api.com/todos",
                json={"id": 100},
                additional_matcher=match_json
            )
            
            result = client.create_todo(title="Special")
            
            assert result["id"] == 100
Задание 2.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Результаты тестирования

| Метод | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| PUT /todos/{id} | _____ | _____ | _____ |
| PATCH /todos/{id} | _____ | _____ | _____ |
| DELETE /todos/{id} | _____ | _____ | _____ |
| Обработка статусов | _____ | _____ | _____ |

### Проверенные HTTP-статусы

| Статус | GET | POST | PUT | DELETE |
|:---|:---|:---|:---|:---|
| 200 | ☐ | ☐ | ☐ | ☐ |
| 201 | ☐ | ☐ | ☐ | ☐ |
| 400 | ☐ | ☐ | ☐ | ☐ |
| 401 | ☐ | ☐ | ☐ | ☐ |
| 404 | ☐ | ☐ | ☐ | ☐ |
| 500 | ☐ | ☐ | ☐ | ☐ |

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться тестировать пакетные операции, эмуляцию сетевых ошибок и таймаутов, сложные сценарии.

Задания для сложного уровня
Задание 3.1. Тестирование пакетных операций (15 минут)
python
# test_api_client_advanced.py

import pytest
import requests_mock
from api_client import TodoClient


class TestBatchOperations:
    """Тесты пакетных операций."""
    
    @pytest.fixture
    def client(self):
        return TodoClient(base_url="https://test.api.com")
    
    def test_get_todos_batch_success(self, client):
        """Пакетное получение задач."""
        with requests_mock.Mocker() as m:
            # Регистрируем ответы для разных ID
            m.get("https://test.api.com/todos/1", json={"id": 1, "title": "Task 1"})
            m.get("https://test.api.com/todos/2", json={"id": 2, "title": "Task 2"})
            m.get("https://test.api.com/todos/3", json={"id": 3, "title": "Task 3"})
            
            result = client.get_todos_batch([1, 2, 3])
            
            assert len(result) == 3
            assert [t["id"] for t in result] == [1, 2, 3]
            assert m.call_count == 3
    
    def test_get_todos_batch_with_missing(self, client):
        """Пакетное получение с отсутствующими ID."""
        with requests_mock.Mocker() as m:
            m.get("https://test.api.com/todos/1", json={"id": 1, "title": "Task 1"})
            m.get("https://test.api.com/todos/2", status_code=404)
            m.get("https://test.api.com/todos/3", json={"id": 3, "title": "Task 3"})
            
            result = client.get_todos_batch([1, 2, 3])
            
            # Только существующие задачи
            assert len(result) == 2
            assert result[0]["id"] == 1
            assert result[1]["id"] == 3
Задание 3.2. Тестирование сетевых ошибок и таймаутов (15 минут)
python
class TestNetworkErrors:
    """Тесты сетевых ошибок и таймаутов."""
    
    @pytest.fixture
    def client(self):
        return TodoClient(base_url="https://test.api.com")
    
    def test_connection_error(self, client):
        """Эмуляция ConnectionError."""
        import requests
        
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/todos/1",
                exc=requests.ConnectionError("Connection refused")
            )
            
            with pytest.raises(requests.ConnectionError):
                client.get_todo(1)
    
    def test_timeout_error(self, client):
        """Эмуляция Timeout."""
        import requests
        
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/todos/1",
                exc=requests.Timeout("Request timed out")
            )
            
            with pytest.raises(requests.Timeout):
                client.get_todo(1)
    
    def test_retry_on_timeout(self, client, mocker):
        """Эмуляция повторной попытки при таймауте."""
        import requests
        
        # Создаём мок для session.get
        mock_get = mocker.patch.object(client.session, "get")
        
        # Первый вызов — таймаут, второй — успех
        mock_get.side_effect = [
            requests.Timeout("Timeout"),
            mocker.Mock(status_code=200, json=lambda: {"id": 1})
        ]
        
        # Функция с повторной попыткой (упрощённая)
        def get_with_retry(todo_id, max_retries=2):
            for attempt in range(max_retries):
                try:
                    return client.get_todo(todo_id)
                except requests.Timeout:
                    if attempt == max_retries - 1:
                        raise
            return None
        
        result = get_with_retry(1)
        
        assert result is not None
        assert mock_get.call_count == 2
    
    def test_http_error_with_response_body(self, client):
        """HTTP-ошибка с телом ответа."""
        import requests
        
        with requests_mock.Mocker() as m:
            m.get(
                "https://test.api.com/todos/1",
                status_code=400,
                json={"error": "Bad Request", "message": "Invalid ID"}
            )
            
            with pytest.raises(requests.HTTPError) as exc_info:
                client.get_todo(1)
            
            # Проверяем, что тело ошибки доступно
            assert exc_info.value.response.status_code == 400
            assert exc_info.value.response.json()["error"] == "Bad Request"
Задание 3.3. Комплексные сценарии (10 минут)
python
class TestComplexScenarios:
    """Комплексные сценарии тестирования."""
    
    @pytest.fixture
    def client(self):
        return TodoClient(base_url="https://test.api.com")
    
    def test_full_crud_workflow(self, client):
        """Полный CRUD-цикл."""
        with requests_mock.Mocker() as m:
            # 1. Создание
            m.post("https://test.api.com/todos", json={"id": 100, "title": "New Task"}, status_code=201)
            
            # 2. Чтение
            m.get("https://test.api.com/todos/100", json={"id": 100, "title": "New Task"})
            
            # 3. Обновление
            m.put("https://test.api.com/todos/100", json={"id": 100, "title": "Updated Task"})
            
            # 4. Удаление
            m.delete("https://test.api.com/todos/100", status_code=200)
            
            # Выполняем workflow
            created = client.create_todo(title="New Task")
            assert created["id"] == 100
            
            read = client.get_todo(100)
            assert read["title"] == "New Task"
            
            updated = client.update_todo(100, title="Updated Task")
            assert updated["title"] == "Updated Task"
            
            deleted = client.delete_todo(100)
            assert deleted is True
            
            assert m.call_count == 4
    
    def test_concurrent_requests(self, client):
        """Тестирование конкурентных запросов."""
        import threading
        import requests_mock
        
        results = []
        errors = []
        
        def worker(worker_id):
            with requests_mock.Mocker() as m:
                m.get(f"https://test.api.com/todos/{worker_id}", json={"id": worker_id})
                try:
                    result = client.get_todo(worker_id)
                    results.append(result)
                except Exception as e:
                    errors.append(e)
        
        threads = []
        for i in range(1, 11):
            t = threading.Thread(target=worker, args=(i,))
            threads.append(t)
            t.start()
        
        for t in threads:
            t.join()
        
        assert len(results) == 10
        assert len(errors) == 0
Задание 3.4. Использование фикстуры requests-mock (5 минут)
python
# conftest.py (добавить в файл conftest.py в корне тестов)
import pytest
import requests_mock


@pytest.fixture
def mock_requests():
    """Фикстура для requests-mock."""
    with requests_mock.Mocker() as m:
        yield m


# test_with_fixture.py
class TestWithFixture:
    """Тесты с использованием фикстуры."""
    
    def test_with_fixture(self, mock_requests, client):
        """Использование фикстуры mock_requests."""
        mock_requests.get(
            "https://test.api.com/todos/1",
            json={"id": 1, "title": "Test"}
        )
        
        result = client.get_todo(1)
        
        assert result["title"] == "Test"
Задание 3.5. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (сложный уровень)

**Студент:** _________________
**Вариант №:** ___ (18-25)

### Результаты тестирования

| Категория | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| GET-запросы | _____ | _____ | _____ |
| POST-запросы | _____ | _____ | _____ |
| PUT/PATCH-запросы | _____ | _____ | _____ |
| DELETE-запросы | _____ | _____ | _____ |
| Пакетные операции | _____ | _____ | _____ |
| Сетевые ошибки | _____ | _____ | _____ |
| Комплексные сценарии | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Проверенные типы ошибок

| Тип ошибки | Проверено |
|:---|:---|
| ConnectionError | ☐ |
| Timeout | ☐ |
| HTTP 4xx | ☐ |
| HTTP 5xx | ☐ |

### Выводы

_______________________________________________________________

_______________________________________________________________
Карточка студента
text
ПР 3.1. ТЕСТИРОВАНИЕ HTTP-КЛИЕНТА С REQUESTS-MOCK

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ GET-запросы (базовый)
□ POST-запросы (базовый)
□ PUT/PATCH-запросы (средний)
□ DELETE-запросы (средний)
□ Обработка HTTP-статусов (средний)
□ Пакетные операции (сложный)
□ Сетевые ошибки (сложный)
□ Комплексные сценарии (сложный)

=== ИСПОЛЬЗОВАННЫЕ ВОЗМОЖНОСТИ ===

□ requests_mock.Mocker()
□ m.get() / m.post() / m.put() / m.delete()
□ Параметр status_code
□ Параметр json / text
□ Параметр exc для ошибок
□ request_history для проверки

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
- test_api_client_base.py
- test_api_client_intermediate.py
- test_api_client_advanced.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Тесты не работают или не используют requests-mock
3 (удовлетворительно)	Базовый	GET и POST тесты (10+ тестов)
4 (хорошо)	Средний	+ PUT, PATCH, DELETE + статусы (20+ тестов)
5 (отлично)	Сложный	+ Пакетные операции + сетевые ошибки (30+ тестов)
Контрольные вопросы (для защиты)
Что такое requests-mock и зачем он нужен?

Как зарегистрировать мок для GET-запроса с параметрами?

Как проверить, что запрос был отправлен с правильными заголовками?

Как эмулировать ошибку ConnectionError?

Как мокировать POST-запрос с проверкой тела?

Как создать разные ответы для одного URL в разных тестах?

Как проверить, что был сделан ровно один запрос?

Как эмулировать задержку ответа?

Как использовать request_history для проверки вызовов?

Как создать фикстуру mock_requests для повторного использования?

Следующее занятие: Лекция 3.3 — Тестирование асинхронных HTTP-клиентов (aiohttp).

text
