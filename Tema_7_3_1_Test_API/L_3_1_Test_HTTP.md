# Лекция 3.1: Тестирование HTTP-клиентов. Библиотека requests. Проблема внешних зависимостей. Моки (mock) при тестировании сетевых вызовов

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.1:** Тестирование API-клиентов и сетевых запросов  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить методы тестирования HTTP-клиентов, использующих библиотеку `requests`, понять проблемы внешних зависимостей при тестировании сетевых вызовов, освоить техники мокирования (mock) для изоляции тестов от реальной сети.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Объяснить проблемы тестирования кода с внешними сетевыми зависимостями (ОК 01, ОК 02).
2. Использовать `unittest.mock` для подмены HTTP-запросов (ПК 3.2, ПК 3.3).
3. Создавать моки для библиотеки `requests` с помощью `responses` и `pytest-mock` (ПК 3.2).
4. Тестировать обработку различных HTTP-статусов и сетевых ошибок (ПК 3.3).

---

## 1. Проблема внешних зависимостей при тестировании

### 1.1 Почему тестирование сетевых вызовов — проблема?
┌─────────────────────────────────────────────────────────────────────────────┐
│ ПРОБЛЕМЫ ТЕСТИРОВАНИЯ СЕТЕВЫХ ЗАПРОСОВ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ 1. ВНЕШНЯЯ ЗАВИСИМОСТЬ │
│ ├── Тест зависит от доступности внешнего API │
│ └── При недоступности API тест падает │
│ │
│ 2. НЕПРЕДСКАЗУЕМОСТЬ │
│ ├── Данные на внешнем API могут меняться │
│ └── Нельзя гарантировать одинаковые результаты │
│ │
│ 3. СКОРОСТЬ │
│ ├── Реальный HTTP-запрос занимает 100-1000 мс │
│ └── Тысячи тестов будут выполняться часами │
│ │
│ 4. СТОИМОСТЬ │
│ ├── Некоторые API платные │
│ └── Каждый тестовый запрос может стоить денег │
│ │
│ 5. БЕЗОПАСНОСТЬ │
│ ├── Нельзя отправлять реальные данные в тестах │
│ └── Риск случайного изменения данных на продакшене │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 1.2 Решение: мокирование (Mock)

**Мокирование** — это замена реальных объектов (например, HTTP-клиента) на тестовые объекты-заглушки, которые имитируют их поведение.
┌─────────────────────────────────────────────────────────────────────────────┐
│ ПРИНЦИП МОКИРОВАНИЯ HTTP-ЗАПРОСОВ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ БЕЗ МОКИРОВАНИЯ: │
│ Тест ──► requests.get(url) ──► Реальный API ──► Медленно, нестабильно │
│ │
│ С МОКИРОВАНИЕМ: │
│ Тест ──► requests.get(url) ──► МОК ──► Быстрый, предсказуемый ответ │
│ │ │
│ └── Возвращает заранее заданные данные │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

---

## 2. Библиотека requests и её тестирование

### 2.1 Пример HTTP-клиента

```python
# weather_client.py

import requests
from typing import Dict, Optional


class WeatherClient:
    """Клиент для получения данных о погоде."""
    
    BASE_URL = "https://api.weatherapi.com/v1"
    
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.session = requests.Session()
    
    def get_current_weather(self, city: str) -> Optional[Dict]:
        """
        Получает текущую погоду для города.
        
        Args:
            city: Название города
        
        Returns:
            Словарь с данными о погоде или None при ошибке
        
        Raises:
            requests.RequestException: При сетевых ошибках
        """
        url = f"{self.BASE_URL}/current.json"
        params = {
            "key": self.api_key,
            "q": city
        }
        
        response = self.session.get(url, params=params)
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 401:
            raise ValueError("Invalid API key")
        elif response.status_code == 404:
            return None
        else:
            response.raise_for_status()
    
    def get_forecast(self, city: str, days: int = 3) -> Optional[Dict]:
        """Получает прогноз погоды на несколько дней."""
        url = f"{self.BASE_URL}/forecast.json"
        params = {
            "key": self.api_key,
            "q": city,
            "days": days
        }
        
        response = self.session.get(url, params=params)
        
        if response.status_code == 200:
            return response.json()
        return None
    
    def close(self):
        """Закрывает сессию."""
        self.session.close()
2.2 Проблемы при тестировании такого кода
python
# ❌ ПЛОХОЙ ТЕСТ (зависит от реального API)
def test_get_current_weather():
    client = WeatherClient(api_key="real_key")
    weather = client.get_current_weather("London")
    assert weather is not None  # Падает, если API недоступен или ключ недействителен
3. Мокирование с unittest.mock
3.1 Базовое мокирование requests.get
python
# test_weather_client_mock.py

import pytest
from unittest.mock import Mock, patch
from weather_client import WeatherClient


class TestWeatherClientMock:
    """Тесты с мокированием requests."""
    
    def test_get_current_weather_success(self):
        """Успешный запрос погоды."""
        # Создаём мок-ответ
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {
            "location": {"name": "London"},
            "current": {"temp_c": 15}
        }
        
        # Мокируем requests.get
        with patch("requests.Session.get") as mock_get:
            mock_get.return_value = mock_response
            
            client = WeatherClient(api_key="test_key")
            result = client.get_current_weather("London")
            
            assert result is not None
            assert result["location"]["name"] == "London"
            assert result["current"]["temp_c"] == 15
            
            # Проверяем, что запрос был сделан с правильными параметрами
            mock_get.assert_called_once()
            call_args = mock_get.call_args
            assert call_args[1]["params"]["q"] == "London"
    
    def test_get_current_weather_not_found(self):
        """Город не найден."""
        mock_response = Mock()
        mock_response.status_code = 404
        
        with patch("requests.Session.get") as mock_get:
            mock_get.return_value = mock_response
            
            client = WeatherClient(api_key="test_key")
            result = client.get_current_weather("UnknownCity")
            
            assert result is None
    
    def test_get_current_weather_invalid_api_key(self):
        """Неверный API ключ."""
        mock_response = Mock()
        mock_response.status_code = 401
        
        with patch("requests.Session.get") as mock_get:
            mock_get.return_value = mock_response
            
            client = WeatherClient(api_key="bad_key")
            
            with pytest.raises(ValueError, match="Invalid API key"):
                client.get_current_weather("London")
    
    def test_get_current_weather_network_error(self):
        """Сетевая ошибка."""
        import requests
        
        with patch("requests.Session.get") as mock_get:
            mock_get.side_effect = requests.ConnectionError("Network unreachable")
            
            client = WeatherClient(api_key="test_key")
            
            with pytest.raises(requests.ConnectionError):
                client.get_current_weather("London")
4. Мокирование с pytest-mock
4.1 Установка и базовое использование
bash
pip install pytest-mock
python
# test_weather_client_pytest_mock.py

import pytest
import requests


class TestWeatherClientPytestMock:
    """Тесты с использованием pytest-mock."""
    
    def test_get_current_weather_success(self, mocker):
        """Успешный запрос с pytest-mock."""
        from weather_client import WeatherClient
        
        # Создаём мок-ответ
        mock_response = mocker.Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {"temp": 20}
        
        # Мокируем requests.Session.get
        mock_get = mocker.patch("requests.Session.get")
        mock_get.return_value = mock_response
        
        client = WeatherClient("test_key")
        result = client.get_current_weather("Moscow")
        
        assert result == {"temp": 20}
        mock_get.assert_called_once()
    
    def test_multiple_calls(self, mocker):
        """Тестирование нескольких вызовов."""
        from weather_client import WeatherClient
        
        # Создаём разные ответы для разных вызовов
        mock_response_200 = mocker.Mock()
        mock_response_200.status_code = 200
        mock_response_200.json.return_value = {"status": "ok"}
        
        mock_response_404 = mocker.Mock()
        mock_response_404.status_code = 404
        
        mock_get = mocker.patch("requests.Session.get")
        mock_get.side_effect = [mock_response_200, mock_response_404]
        
        client = WeatherClient("test_key")
        
        # Первый вызов
        result1 = client.get_current_weather("Moscow")
        assert result1 == {"status": "ok"}
        
        # Второй вызов
        result2 = client.get_current_weather("Unknown")
        assert result2 is None
        
        assert mock_get.call_count == 2
5. Библиотека responses
5.1 Установка и базовое использование
bash
pip install responses
Responses — библиотека, специально созданная для мокирования requests. Она позволяет регистрировать ожидаемые запросы и ответы.

python
# test_weather_client_responses.py

import pytest
import responses
from weather_client import WeatherClient


class TestWeatherClientResponses:
    """Тесты с использованием библиотеки responses."""
    
    @responses.activate
    def test_get_current_weather_success(self):
        """Успешный запрос с responses."""
        # Регистрируем ожидаемый запрос и ответ
        responses.add(
            responses.GET,
            "https://api.weatherapi.com/v1/current.json",
            json={"location": {"name": "London"}, "current": {"temp_c": 15}},
            status=200
        )
        
        client = WeatherClient(api_key="test_key")
        result = client.get_current_weather("London")
        
        assert result["location"]["name"] == "London"
        assert result["current"]["temp_c"] == 15
        
        # Проверяем, что запрос был сделан
        assert len(responses.calls) == 1
        assert responses.calls[0].request.params["q"] == "London"
    
    @responses.activate
    def test_get_current_weather_not_found(self):
        """Город не найден."""
        responses.add(
            responses.GET,
            "https://api.weatherapi.com/v1/current.json",
            status=404
        )
        
        client = WeatherClient(api_key="test_key")
        result = client.get_current_weather("UnknownCity")
        
        assert result is None
    
    @responses.activate
    def test_get_current_weather_invalid_api_key(self):
        """Неверный API ключ."""
        responses.add(
            responses.GET,
            "https://api.weatherapi.com/v1/current.json",
            status=401
        )
        
        client = WeatherClient(api_key="bad_key")
        
        with pytest.raises(ValueError, match="Invalid API key"):
            client.get_current_weather("London")
    
    @responses.activate
    def test_get_forecast_success(self):
        """Прогноз погоды."""
        responses.add(
            responses.GET,
            "https://api.weatherapi.com/v1/forecast.json",
            json={"forecast": {"forecastday": [{"day": {"avgtemp_c": 20}}]}},
            status=200
        )
        
        client = WeatherClient(api_key="test_key")
        result = client.get_forecast("London", days=5)
        
        assert result is not None
        assert "forecast" in result
        
        # Проверяем параметры запроса
        assert responses.calls[0].request.params["days"] == "5"
5.2 Регистрация нескольких запросов
python
@responses.activate
def test_multiple_cities(self):
    """Тестирование запросов к разным городам."""
    # Регистрируем ответы для разных городов
    responses.add(
        responses.GET,
        "https://api.weatherapi.com/v1/current.json",
        json={"location": {"name": "Moscow"}, "current": {"temp_c": -5}},
        status=200,
        match=[responses.matchers.query_param_matcher({"q": "Moscow"})]
    )
    
    responses.add(
        responses.GET,
        "https://api.weatherapi.com/v1/current.json",
        json={"location": {"name": "London"}, "current": {"temp_c": 15}},
        status=200,
        match=[responses.matchers.query_param_matcher({"q": "London"})]
    )
    
    client = WeatherClient(api_key="test_key")
    
    moscow_weather = client.get_current_weather("Moscow")
    london_weather = client.get_current_weather("London")
    
    assert moscow_weather["location"]["name"] == "Moscow"
    assert london_weather["location"]["name"] == "London"
5.3 Использование matchers для точного сопоставления
python
@responses.activate
def test_exact_url_matching(self):
    """Точное сопоставление URL с параметрами."""
    import re
    
    responses.add(
        responses.GET,
        re.compile(r"https://api.weatherapi.com/v1/current.json.*"),
        json={"temp": 20},
        status=200
    )
    
    client = WeatherClient(api_key="test_key")
    result = client.get_current_weather("Paris")
    
    assert result is not None
6. Тестирование обработки ошибок и таймаутов
6.1 Эмуляция сетевых ошибок
python
class TestNetworkErrors:
    """Тестирование сетевых ошибок."""
    
    @responses.activate
    def test_connection_timeout(self):
        """Эмуляция таймаута."""
        import requests
        
        responses.add(
            responses.GET,
            "https://api.weatherapi.com/v1/current.json",
            body=requests.exceptions.Timeout("Connection timeout")
        )
        
        client = WeatherClient(api_key="test_key")
        
        with pytest.raises(requests.exceptions.Timeout):
            client.get_current_weather("London")
    
    @responses.activate
    def test_connection_error(self):
        """Эмуляция ошибки соединения."""
        import requests
        
        responses.add(
            responses.GET,
            "https://api.weatherapi.com/v1/current.json",
            body=requests.exceptions.ConnectionError("DNS lookup failed")
        )
        
        client = WeatherClient(api_key="test_key")
        
        with pytest.raises(requests.exceptions.ConnectionError):
            client.get_current_weather("London")
    
    def test_timeout_with_mock(self, mocker):
        """Таймаут через pytest-mock."""
        import requests
        
        mock_get = mocker.patch("requests.Session.get")
        mock_get.side_effect = requests.exceptions.Timeout("Timeout")
        
        client = WeatherClient(api_key="test_key")
        
        with pytest.raises(requests.exceptions.Timeout):
            client.get_current_weather("London")
7. Сравнение подходов к мокированию
Подход	Преимущества	Недостатки
unittest.mock	Встроен в Python, гибкий	Многословный, сложно проверять параметры запроса
pytest-mock	Краткий синтаксис, интеграция с pytest	Тот же уровень сложности при проверке параметров
responses	Специализирован для HTTP, удобные matchers	Требует отдельной установки, магия с декораторами
Рекомендация
Для простых тестов (1-2 запроса) → pytest-mock

Для сложных сценариев (множественные запросы, проверка параметров) → responses

Если нет возможности установить доп. библиотеки → unittest.mock

8. Шпаргалка
python
# === unittest.mock ===
from unittest.mock import patch, Mock

with patch("requests.get") as mock_get:
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"data": "test"}
    mock_get.return_value = mock_response

# === pytest-mock ===
def test_example(mocker):
    mock_get = mocker.patch("requests.get")
    mock_get.return_value.status_code = 200
    mock_get.return_value.json.return_value = {"data": "test"}

# === responses ===
import responses

@responses.activate
def test_example():
    responses.add(
        responses.GET,
        "https://api.example.com/endpoint",
        json={"data": "test"},
        status=200
    )
    # Тест здесь
    assert len(responses.calls) == 1
Контрольные вопросы
Почему нельзя тестировать HTTP-клиенты с реальными сетевыми запросами?

Что такое мок (mock) и для чего он используется?

В чём разница между patch из unittest.mock и responses?

Как проверить, что запрос был сделан с правильными параметрами?

Как эмулировать ошибку 404 с помощью responses?

Как эмулировать таймаут соединения?

Как мокировать несколько последовательных запросов с разными ответами?

Что делает декоратор @responses.activate?

Как с помощью responses проверить, что был сделан именно один запрос?

Как мокировать POST-запрос с телом JSON?

Следующее занятие: ПР 3.1 — Практическое тестирование HTTP-клиентов с мокированием.
