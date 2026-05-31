# Лекция 3.15: Комплексная работа: разработка тестового набора для реального модуля (ETL + API)

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Интегрировать все изученные техники тестирования для разработки комплексного тестового набора для реального модуля, сочетающего ETL-процессы и API-взаимодействие.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Разрабатывать комплексные тестовые наборы для модулей с ETL и API (ПК 3.1, ПК 3.2).
2. Комбинировать техники тестирования: исключения, мокирование, покрытие, интеграционные тесты (ПК 3.2, ПК 3.3).
3. Проектировать тестовую архитектуру для сложных систем (ПК 3.1).
4. Анализировать и оценивать качество тестового набора (ОК 02, ОК 05).

---

## 1. Объект тестирования: ETL + API модуль

### 1.1 Описание модуля
┌─────────────────────────────────────────────────────────────────────────────┐
│ АРХИТЕКТУРА ETL + API МОДУЛЯ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ │
│ │ Source │────►│ ETL │────►│ Database │ │
│ │ API │ │ Processor │ │ │ │
│ └─────────────┘ └──────┬──────┘ └─────────────┘ │
│ │ │
│ ▼ │
│ ┌─────────────┐ │
│ │ REST API │ │
│ │ (Query) │ │
│ └─────────────┘ │
│ │
│ Компоненты: │
│ 1. Source API Client - получение данных из внешнего API │
│ 2. ETL Processor - трансформация и валидация данных │
│ 3. Database - хранение обработанных данных │
│ 4. REST API - предоставление данных клиентам │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 1.2 Исходный код модуля

```python
# etl_api_module.py

import requests
import json
import sqlite3
from typing import List, Dict, Any, Optional
from dataclasses import dataclass, asdict
from datetime import datetime
from contextlib import contextmanager


# ============================================================
# ИСКЛЮЧЕНИЯ
# ============================================================

class ETLException(Exception):
    """Базовое исключение ETL модуля."""
    pass


class APIClientError(ETLException):
    """Ошибка при вызове внешнего API."""
    pass


class ValidationError(ETLException):
    """Ошибка валидации данных."""
    pass


class DatabaseError(ETLException):
    """Ошибка при работе с БД."""
    pass


# ============================================================
# МОДЕЛИ ДАННЫХ
# ============================================================

@dataclass
class RawUserData:
    """Сырые данные из API."""
    id: int
    name: str
    email: str
    phone: str
    website: str
    
    @classmethod
    def from_dict(cls, data: dict) -> "RawUserData":
        return cls(
            id=data.get("id", 0),
            name=data.get("name", ""),
            email=data.get("email", ""),
            phone=data.get("phone", ""),
            website=data.get("website", "")
        )


@dataclass
class ProcessedUser:
    """Обработанные данные для БД."""
    id: int
    name: str
    email: str
    phone_normalized: str
    domain: str
    processed_at: str
    
    def to_dict(self) -> dict:
        return asdict(self)


# ============================================================
# API КЛИЕНТ
# ============================================================

class SourceAPIClient:
    """Клиент для получения данных из внешнего API."""
    
    BASE_URL = "https://jsonplaceholder.typicode.com"
    
    def __init__(self, timeout: int = 30, max_retries: int = 3):
        self.timeout = timeout
        self.max_retries = max_retries
        self.session = requests.Session()
    
    def fetch_users(self) -> List[RawUserData]:
        """
        Получает список пользователей из API.
        
        Raises:
            APIClientError: При ошибках API или сети
        """
        url = f"{self.BASE_URL}/users"
        
        for attempt in range(self.max_retries):
            try:
                response = self.session.get(url, timeout=self.timeout)
                response.raise_for_status()
                data = response.json()
                return [RawUserData.from_dict(item) for item in data]
            
            except requests.RequestException as e:
                if attempt == self.max_retries - 1:
                    raise APIClientError(f"Failed to fetch users: {e}") from e
                
                import time
                time.sleep(2 ** attempt)
        
        return []
    
    def close(self):
        self.session.close()


# ============================================================
# ETL ПРОЦЕССОР
# ============================================================

class ETLProcessor:
    """Обработка и трансформация данных."""
    
    @staticmethod
    def normalize_phone(phone: str) -> str:
        """
        Нормализует номер телефона.
        
        Примеры:
        "1-770-736-8031 x56442" -> "17707368031"
        """
        import re
        digits = re.sub(r'\D', '', phone)
        return digits[:11] if digits else ""
    
    @staticmethod
    def extract_domain(email: str) -> str:
        """Извлекает домен из email."""
        if "@" not in email:
            return ""
        return email.split("@")[1]
    
    @classmethod
    def process_user(cls, raw_user: RawUserData) -> ProcessedUser:
        """
        Преобразует сырые данные в обработанные.
        
        Raises:
            ValidationError: При некорректных данных
        """
        if not raw_user.name:
            raise ValidationError(f"User {raw_user.id} has no name")
        
        if not raw_user.email or "@" not in raw_user.email:
            raise ValidationError(f"User {raw_user.id} has invalid email")
        
        return ProcessedUser(
            id=raw_user.id,
            name=raw_user.name.strip(),
            email=raw_user.email.lower(),
            phone_normalized=cls.normalize_phone(raw_user.phone),
            domain=cls.extract_domain(raw_user.email),
            processed_at=datetime.now().isoformat()
        )
    
    @classmethod
    def process_users(cls, raw_users: List[RawUserData]) -> tuple:
        """
        Обрабатывает список пользователей.
        
        Returns:
            tuple: (processed_users, errors)
        """
        processed = []
        errors = []
        
        for raw_user in raw_users:
            try:
                processed.append(cls.process_user(raw_user))
            except ValidationError as e:
                errors.append(str(e))
        
        return processed, errors


# ============================================================
# БАЗА ДАННЫХ
# ============================================================

class Database:
    """Работа с базой данных SQLite."""
    
    def __init__(self, db_path: str = ":memory:"):
        self.db_path = db_path
        self._init_db()
    
    @contextmanager
    def _get_connection(self):
        conn = sqlite3.connect(self.db_path)
        conn.row_factory = sqlite3.Row
        try:
            yield conn
            conn.commit()
        except Exception as e:
            conn.rollback()
            raise DatabaseError(f"Database error: {e}") from e
        finally:
            conn.close()
    
    def _init_db(self):
        """Инициализирует таблицы."""
        with self._get_connection() as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS users (
                    id INTEGER PRIMARY KEY,
                    name TEXT NOT NULL,
                    email TEXT NOT NULL,
                    phone_normalized TEXT,
                    domain TEXT,
                    processed_at TEXT
                )
            """)
    
    def save_users(self, users: List[ProcessedUser]) -> int:
        """
        Сохраняет пользователей в БД.
        
        Returns:
            Количество сохранённых записей
        """
        with self._get_connection() as conn:
            cursor = conn.cursor()
            
            for user in users:
                cursor.execute("""
                    INSERT OR REPLACE INTO users 
                    (id, name, email, phone_normalized, domain, processed_at)
                    VALUES (?, ?, ?, ?, ?, ?)
                """, (
                    user.id, user.name, user.email,
                    user.phone_normalized, user.domain, user.processed_at
                ))
            
            return len(users)
    
    def get_users(self, limit: int = 100) -> List[dict]:
        """Получает пользователей из БД."""
        with self._get_connection() as conn:
            cursor = conn.execute(
                "SELECT * FROM users LIMIT ?", (limit,)
            )
            return [dict(row) for row in cursor.fetchall()]
    
    def get_user_by_id(self, user_id: int) -> Optional[dict]:
        """Получает пользователя по ID."""
        with self._get_connection() as conn:
            cursor = conn.execute(
                "SELECT * FROM users WHERE id = ?", (user_id,)
            )
            row = cursor.fetchone()
            return dict(row) if row else None
    
    def clear(self):
        """Очищает таблицу (для тестов)."""
        with self._get_connection() as conn:
            conn.execute("DELETE FROM users")


# ============================================================
# REST API (для простоты — без реального сервера)
# ============================================================

class APIService:
    """Сервис для предоставления данных через API."""
    
    def __init__(self, database: Database):
        self.db = database
    
    def get_all_users(self) -> Dict[str, Any]:
        """GET /api/users"""
        users = self.db.get_users()
        return {
            "success": True,
            "count": len(users),
            "users": users
        }
    
    def get_user(self, user_id: int) -> Dict[str, Any]:
        """GET /api/users/{id}"""
        user = self.db.get_user_by_id(user_id)
        if user:
            return {"success": True, "user": user}
        return {"success": False, "error": f"User {user_id} not found"}
    
    def get_stats(self) -> Dict[str, Any]:
        """GET /api/stats"""
        users = self.db.get_users()
        domains = {}
        for user in users:
            domain = user.get("domain", "unknown")
            domains[domain] = domains.get(domain, 0) + 1
        
        return {
            "success": True,
            "total_users": len(users),
            "domains": domains
        }


# ============================================================
# ГЛАВНЫЙ КОМПОНЕНТ (ETL + API)
# ============================================================

class ETLAPIModule:
    """Главный модуль, объединяющий ETL и API."""
    
    def __init__(
        self,
        api_client: SourceAPIClient = None,
        etl_processor: ETLProcessor = None,
        database: Database = None,
        api_service: APIService = None
    ):
        self.api_client = api_client or SourceAPIClient()
        self.etl_processor = etl_processor or ETLProcessor()
        self.database = database or Database()
        self.api_service = api_service or APIService(self.database)
    
    def run_etl(self) -> Dict[str, Any]:
        """
        Запускает полный ETL процесс.
        
        Returns:
            Результат выполнения ETL
        """
        try:
            # Extract
            raw_users = self.api_client.fetch_users()
            
            # Transform
            processed_users, errors = self.etl_processor.process_users(raw_users)
            
            # Load
            saved_count = self.database.save_users(processed_users)
            
            return {
                "success": True,
                "total_fetched": len(raw_users),
                "processed": len(processed_users),
                "saved": saved_count,
                "errors": errors
            }
        
        except APIClientError as e:
            return {
                "success": False,
                "error": f"API error: {e}",
                "stage": "extract"
            }
        except DatabaseError as e:
            return {
                "success": False,
                "error": f"Database error: {e}",
                "stage": "load"
            }
        except Exception as e:
            return {
                "success": False,
                "error": f"Unexpected error: {e}",
                "stage": "unknown"
            }
    
    def query_api(self, endpoint: str, **kwargs) -> Dict[str, Any]:
        """Выполняет запрос к API."""
        if endpoint == "users":
            return self.api_service.get_all_users()
        elif endpoint == "user":
            return self.api_service.get_user(kwargs.get("user_id", 0))
        elif endpoint == "stats":
            return self.api_service.get_stats()
        else:
            return {"success": False, "error": f"Unknown endpoint: {endpoint}"}
    
    def close(self):
        self.api_client.close()
2. Комплексный тестовый набор
2.1 Тестирование исключений
python
# test_etl_api_exceptions.py

import pytest
from etl_api_module import (
    SourceAPIClient, APIClientError, ValidationError,
    ETLProcessor, RawUserData
)


class TestExceptions:
    """Тестирование исключений."""
    
    def test_api_client_connection_error(self, mocker):
        """Тест ошибки соединения с API."""
        client = SourceAPIClient(max_retries=1)
        
        mocker.patch("requests.Session.get", side_effect=requests.ConnectionError("No connection"))
        
        with pytest.raises(APIClientError, match="Failed to fetch users"):
            client.fetch_users()
    
    def test_validation_error_missing_name(self):
        """Тест ошибки валидации при отсутствии имени."""
        raw_user = RawUserData(id=1, name="", email="test@test.com", phone="", website="")
        
        with pytest.raises(ValidationError, match="User 1 has no name"):
            ETLProcessor.process_user(raw_user)
    
    def test_validation_error_invalid_email(self):
        """Тест ошибки валидации при неверном email."""
        raw_user = RawUserData(id=1, name="Test", email="invalid", phone="", website="")
        
        with pytest.raises(ValidationError, match="User 1 has invalid email"):
            ETLProcessor.process_user(raw_user)
2.2 Модульное тестирование с моками
python
# test_etl_api_unit.py

import pytest
from etl_api_module import (
    SourceAPIClient, ETLProcessor, Database, APIService, RawUserData, ProcessedUser
)


class TestUnitMocks:
    """Модульные тесты с моками."""
    
    @pytest.fixture
    def mock_api_response(self):
        return [
            {"id": 1, "name": "John Doe", "email": "john@example.com", 
             "phone": "123-456-7890", "website": "john.com"},
            {"id": 2, "name": "Jane Smith", "email": "jane@example.com",
             "phone": "098-765-4321", "website": "jane.com"}
        ]
    
    def test_fetch_users_with_mock(self, mocker, mock_api_response):
        """Тест API клиента с моком."""
        client = SourceAPIClient()
        
        mock_response = mocker.Mock()
        mock_response.json.return_value = mock_api_response
        mock_response.raise_for_status.return_value = None
        
        mocker.patch("requests.Session.get", return_value=mock_response)
        
        users = client.fetch_users()
        
        assert len(users) == 2
        assert users[0].name == "John Doe"
        assert users[1].email == "jane@example.com"
    
    def test_etl_processor_with_mock(self):
        """Тест ETL процессора."""
        raw_user = RawUserData(
            id=1, name="  Test User  ", 
            email=" TEST@EXAMPLE.COM ", phone="1-770-736-8031 x56442", website=""
        )
        
        processed = ETLProcessor.process_user(raw_user)
        
        assert processed.name == "Test User"
        assert processed.email == "test@example.com"
        assert processed.phone_normalized == "17707368031"
        assert processed.domain == "example.com"
    
    def test_database_mock(self, mocker):
        """Тест базы данных с моком."""
        db = Database()
        
        mock_conn = mocker.MagicMock()
        mock_cursor = mocker.MagicMock()
        mock_conn.cursor.return_value = mock_cursor
        
        mocker.patch("sqlite3.connect", return_value=mock_conn)
        
        users = [ProcessedUser(
            id=1, name="Test", email="test@test.com",
            phone_normalized="123", domain="test.com", processed_at="2024-01-01"
        )]
        
        db.save_users(users)
        mock_cursor.execute.assert_called()
2.3 Интеграционное тестирование
python
# test_etl_api_integration.py

import pytest
from etl_api_module import (
    SourceAPIClient, ETLProcessor, Database, APIService, ETLAPIModule
)


class TestIntegration:
    """Интеграционные тесты."""
    
    @pytest.fixture
    def module(self):
        """Создаёт модуль с реальными компонентами (кроме API)."""
        return ETLAPIModule()
    
    def test_database_operations(self, module):
        """Тест операций с БД."""
        # Очищаем БД
        module.database.clear()
        
        # Сохраняем пользователей
        users = [
            ProcessedUser(1, "Alice", "alice@test.com", "123", "test.com", "2024-01-01"),
            ProcessedUser(2, "Bob", "bob@test.com", "456", "test.com", "2024-01-01"),
        ]
        module.database.save_users(users)
        
        # Проверяем получение всех
        all_users = module.database.get_users()
        assert len(all_users) == 2
        
        # Проверяем получение по ID
        user = module.database.get_user_by_id(1)
        assert user["name"] == "Alice"
        
        # Проверяем получение несуществующего
        assert module.database.get_user_by_id(999) is None
    
    def test_api_service_with_real_db(self, module):
        """Тест API сервиса с реальной БД."""
        module.database.clear()
        
        # Заполняем БД тестовыми данными
        users = [
            ProcessedUser(1, "Alice", "alice@test.com", "123", "test.com", "2024-01-01"),
            ProcessedUser(2, "Bob", "bob@test.com", "456", "test.com", "2024-01-01"),
        ]
        module.database.save_users(users)
        
        # Тестируем API эндпоинты
        response = module.query_api("users")
        assert response["success"] is True
        assert response["count"] == 2
        
        response = module.query_api("user", user_id=1)
        assert response["success"] is True
        assert response["user"]["name"] == "Alice"
        
        response = module.query_api("user", user_id=999)
        assert response["success"] is False
        assert "not found" in response["error"]
        
        response = module.query_api("stats")
        assert response["success"] is True
        assert response["total_users"] == 2
    
    def test_full_etl_flow_with_mock_api(self, module, mocker):
        """Полный ETL процесс с моком API."""
        module.database.clear()
        
        mock_api_response = [
            {"id": 1, "name": "John", "email": "john@test.com", "phone": "1234567890", "website": ""},
            {"id": 2, "name": "Jane", "email": "jane@test.com", "phone": "0987654321", "website": ""},
        ]
        
        mock_response = mocker.Mock()
        mock_response.json.return_value = mock_api_response
        mock_response.raise_for_status.return_value = None
        
        mocker.patch("requests.Session.get", return_value=mock_response)
        
        result = module.run_etl()
        
        assert result["success"] is True
        assert result["total_fetched"] == 2
        assert result["processed"] == 2
        assert result["saved"] == 2
        assert result["errors"] == []
        
        # Проверяем, что данные сохранились в БД
        users = module.database.get_users()
        assert len(users) == 2
2.4 Тестирование покрытия
bash
# Запуск тестов с измерением покрытия
pytest --cov=etl_api_module --cov-branch --cov-report=term --cov-report=html
python
# coverage_config.py
"""
Для достижения высокого покрытия необходимо добавить тесты на:
- Обработку пустых списков
- Ошибки базы данных
- Неизвестные эндпоинты API
- Граничные случаи в ETL
"""
3. Итоговый отчёт
markdown
## Комплексный отчёт по тестированию ETL + API модуля

**Студент:** _________________
**Дата:** _________________

### 1. Статистика тестов

| Категория | Количество | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| Модульные тесты | _____ | _____ | _____ |
| Интеграционные тесты | _____ | _____ | _____ |
| Тесты исключений | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### 2. Покрытие кода

| Компонент | Покрытие строк | Покрытие ветвей |
|:---|:---|:---|
| SourceAPIClient | _____% | _____% |
| ETLProcessor | _____% | _____% |
| Database | _____% | _____% |
| APIService | _____% | _____% |
| ETLAPIModule | _____% | _____% |
| **Общее** | _____% | _____% |

### 3. Использованные техники

| Техника | Применение |
|:---|:---|
| pytest.raises | Проверка исключений |
| monkeypatch/mocker | Мокирование API |
| tmp_path | Тесты с файлами БД |
| parametrize | Параметризация тестов |
| fixtures | Подготовка тестовых данных |
| coverage | Измерение покрытия |

### 4. Выводы

_______________________________________________________________

_______________________________________________________________
Шпаргалка
python
# === ТЕСТИРОВАНИЕ ИСКЛЮЧЕНИЙ ===
with pytest.raises(APIClientError, match="Failed to fetch"):
    client.fetch_users()

# === МОКИРОВАНИЕ API ===
mock_response = mocker.Mock()
mock_response.json.return_value = test_data
mocker.patch("requests.Session.get", return_value=mock_response)

# === ТЕСТИРОВАНИЕ ETL ===
raw_user = RawUserData(id=1, name="Test", email="test@test.com", ...)
processed = ETLProcessor.process_user(raw_user)
assert processed.domain == "test.com"

# === ТЕСТИРОВАНИЕ БД ===
db = Database(":memory:")
db.save_users(users)
assert len(db.get_users()) == n

# === ЗАПУСК С ПОКРЫТИЕМ ===
pytest --cov=etl_api_module --cov-branch --cov-report=html
Контрольные вопросы
Какие компоненты входят в ETL + API модуль?

Как тестировать API клиент без реального внешнего API?

Какие исключения могут возникать в ETL процессе?

Как проверить, что данные корректно трансформируются?

Как тестировать базу данных в изоляции?

Как организовать интеграционные тесты для всего модуля?

Какие метрики покрытия важны для комплексного модуля?

Как мокирование помогает при тестировании ETL?

Какие граничные случаи нужно проверить для ETL?

Как оценить качество тестового набора?

Следующее занятие: Контрольная работа 3.3 по темам 3.9–3.15.
