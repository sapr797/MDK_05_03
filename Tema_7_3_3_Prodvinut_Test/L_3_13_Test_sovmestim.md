# Лекция 3.13: Тестирование совместимости и конфигурационное тестирование

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить методы тестирования совместимости программного обеспечения с различными средами выполнения, а также освоить подходы к конфигурационному тестированию для проверки работы системы в различных конфигурациях.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Планировать и проводить тестирование совместимости (ПК 3.1, ПК 3.2).
2. Выявлять проблемы совместимости с ОС, браузерами, устройствами (ПК 3.3).
3. Применять техники конфигурационного тестирования (ПК 3.2).
4. Использовать инструменты для автоматизации тестирования совместимости (ПК 3.2).

---

## 1. Тестирование совместимости

### 1.1 Что такое тестирование совместимости?

**Тестирование совместимости (Compatibility Testing)** — проверка способности программного обеспечения работать в различных средах, с разным оборудованием и программным обеспечением.
┌─────────────────────────────────────────────────────────────────────────────┐
│ ВИДЫ ТЕСТИРОВАНИЯ СОВМЕСТИМОСТИ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ 1. ОПЕРАЦИОННАЯ СИСТЕМА │
│ ├── Windows (10, 11, Server) │
│ ├── macOS (Ventura, Sonoma, Sequoia) │
│ ├── Linux (Ubuntu, CentOS, Debian) │
│ └── Мобильные OS (iOS, Android) │
│ │
│ 2. БРАУЗЕРЫ │
│ ├── Chrome, Firefox, Safari, Edge │
│ └── Мобильные браузеры │
│ │
│ 3. АППАРАТНОЕ ОБЕСПЕЧЕНИЕ │
│ ├── Разрешения экрана │
│ ├── Процессоры (x86, ARM) │
│ └── Память, дисковое пространство │
│ │
│ 4. ПРОГРАММНОЕ ОБЕСПЕЧЕНИЕ │
│ ├── Базы данных (MySQL, PostgreSQL) │
│ ├── Веб-серверы (Nginx, Apache) │
│ └── Версии Python, Java, Node.js │
│ │
│ 5. СЕТЕВЫЕ УСЛОВИЯ │
│ ├── Разные скорости соединения │
│ ├── Высокая задержка │
│ └── Прерывистое соединение │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 1.2 Матрица совместимости

**Матрица совместимости** — таблица, показывающая, какие комбинации окружений необходимо протестировать.

```python
class CompatibilityMatrix:
    """
    Матрица совместимости для тестирования.
    """
    
    def __init__(self):
        self.os_list = ["Windows 10", "Windows 11", "macOS 14", "Ubuntu 22.04"]
        self.browsers = ["Chrome", "Firefox", "Safari", "Edge"]
        self.resolutions = ["1920x1080", "1366x768", "375x667 (mobile)"]
        self._tests = []
    
    def generate_test_cases(self, max_combinations: int = None):
        """
        Генерирует тестовые случаи по матрице совместимости.
        """
        test_cases = []
        for os in self.os_list:
            for browser in self.browsers:
                for resolution in self.resolutions:
                    test_cases.append({
                        "os": os,
                        "browser": browser,
                        "resolution": resolution,
                        "id": f"{os}_{browser}_{resolution}".replace(" ", "_")
                    })
        
        if max_combinations and len(test_cases) > max_combinations:
            # Сокращаем количество комбинаций (попарное тестирование)
            test_cases = self._reduce_combinations(test_cases, max_combinations)
        
        return test_cases
    
    def _reduce_combinations(self, test_cases: list, max_count: int) -> list:
        """Упрощённая версия попарного сокращения."""
        # В реальном проекте используется попарное тестирование
        return test_cases[:max_count]


matrix = CompatibilityMatrix()
test_cases = matrix.generate_test_cases(max_combinations=10)

for tc in test_cases:
    print(f"Test: OS={tc['os']}, Browser={tc['browser']}, Resolution={tc['resolution']}")
2. Конфигурационное тестирование
2.1 Что такое конфигурационное тестирование?
Конфигурационное тестирование — проверка работы системы при различных настройках и параметрах конфигурации.

text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    КОНФИГУРАЦИОННОЕ ТЕСТИРОВАНИЕ                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ЧТО ТЕСТИРУЕТСЯ:                                                          │
│   • Параметры запуска (флаги, аргументы)                                    │
│   • Файлы конфигурации (.env, config.yml, settings.py)                     │
│   • Переменные окружения                                                    │
│   • Настройки базы данных                                                   │
│   • Настройки логирования                                                   │
│   • Функциональные переключатели (feature flags)                            │
│                                                                              │
│   ПРИМЕРЫ КОНФИГУРАЦИЙ:                                                     │
│   • DEBUG=True/False                                                        │
│   • DATABASE_URL: production vs test                                        │
│   • LOG_LEVEL: DEBUG, INFO, WARNING, ERROR                                  │
│   • CACHE_ENABLED: True/False                                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
2.2 Пример: тестирование различных конфигураций
python
import os
import pytest
from typing import Dict, Any


class ConfigTester:
    """Тестирование различных конфигураций."""
    
    @staticmethod
    def test_with_env_vars(monkeypatch, config_changes: Dict[str, str]):
        """Тестирует поведение при изменении переменных окружения."""
        for key, value in config_changes.items():
            monkeypatch.setenv(key, value)
        
        # Перезагружаем конфигурацию
        import importlib
        import config
        importlib.reload(config)
        
        # Проверяем поведение
        return config.settings


class TestConfigurations:
    """Параметризованные тесты конфигураций."""
    
    @pytest.mark.parametrize("debug_mode,expected_log_level", [
        (True, "DEBUG"),
        (False, "INFO"),
    ])
    def test_log_level_by_debug_mode(self, monkeypatch, debug_mode, expected_log_level):
        """Проверка уровня логирования в зависимости от DEBUG режима."""
        monkeypatch.setenv("DEBUG", str(debug_mode))
        
        from app.config import get_log_level
        assert get_log_level() == expected_log_level
    
    @pytest.mark.parametrize("db_url,expected_type", [
        ("postgresql://localhost/test", "postgres"),
        ("sqlite:///test.db", "sqlite"),
        ("mysql://localhost/test", "mysql"),
    ])
    def test_database_type_detection(self, monkeypatch, db_url, expected_type):
        """Проверка определения типа БД по URL."""
        monkeypatch.setenv("DATABASE_URL", db_url)
        
        from app.config import get_database_type
        assert get_database_type() == expected_type
2.3 Фикстуры для конфигураций
python
@pytest.fixture(params=[
    {"debug": True, "cache": False, "workers": 1},
    {"debug": True, "cache": True, "workers": 1},
    {"debug": False, "cache": False, "workers": 4},
    {"debug": False, "cache": True, "workers": 4},
])
def app_config(request):
    """Фикстура с различными конфигурациями приложения."""
    config = request.param
    # Применяем конфигурацию
    app = create_app(config)
    yield app
    # Очистка
    app.shutdown()


def test_app_with_configurations(app_config):
    """Тест приложения с разными конфигурациями."""
    response = app_config.get("/health")
    assert response.status_code == 200
3. Инструменты для тестирования совместимости
3.1 BrowserStack и Sauce Labs
python
# Пример конфигурации для BrowserStack
browserstack_config = {
    "browser": "Chrome",
    "browser_version": "latest",
    "os": "Windows",
    "os_version": "10",
    "resolution": "1920x1080"
}

# В тестах:
def test_cross_browser(browserstack_driver):
    driver.get("https://example.com")
    assert "Example" in driver.title
3.2 Selenium Grid
python
from selenium import webdriver
from selenium.webdriver.common.desired_capabilities import DesiredCapabilities


def test_selenium_grid():
    """Запуск теста на Selenium Grid."""
    capabilities = {
        "browserName": "chrome",
        "browserVersion": "latest",
        "platformName": "Windows 10"
    }
    
    driver = webdriver.Remote(
        command_executor="http://localhost:4444/wd/hub",
        desired_capabilities=capabilities
    )
    
    try:
        driver.get("https://example.com")
        assert "Example" in driver.title
    finally:
        driver.quit()
3.3 Docker Compose для тестирования совместимости
yaml
# docker-compose.test.yml
version: '3.8'

services:
  app:
    build: .
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/test
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
  
  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=password
  
  redis:
    image: redis:7-alpine
4. Тестирование адаптивности (Responsive Testing)
4.1 Проверка адаптивного дизайна
python
class ResponsiveTester:
    """Тестирование адаптивного дизайна."""
    
    VIEWPORT_SIZES = {
        "mobile": (375, 667),
        "tablet": (768, 1024),
        "desktop": (1920, 1080),
        "wide": (2560, 1440)
    }
    
    @staticmethod
    def test_responsive_layout(driver, url: str):
        """Проверяет отображение на разных разрешениях."""
        results = {}
        
        for name, (width, height) in ResponsiveTester.VIEWPORT_SIZES.items():
            driver.set_window_size(width, height)
            driver.get(url)
            
            # Проверка, что элементы видны
            is_menu_visible = driver.find_element(By.CSS_SELECTOR, ".menu").is_displayed()
            results[name] = {
                "width": width,
                "height": height,
                "menu_visible": is_menu_visible
            }
        
        return results


def test_mobile_layout():
    driver = webdriver.Chrome()
    try:
        results = ResponsiveTester.test_responsive_layout(driver, "https://example.com")
        
        # На мобильных устройствах меню может быть скрыто
        assert results["mobile"]["menu_visible"] is False
        assert results["desktop"]["menu_visible"] is True
    finally:
        driver.quit()
4.2 Параметризация размеров экрана
python
@pytest.mark.parametrize("viewport", [
    (375, 667),   # iPhone SE
    (390, 844),   # iPhone 12
    (768, 1024),  # iPad
    (1280, 720),  # HD
    (1920, 1080), # Full HD
    (2560, 1440), # 2K
])
def test_viewport_responsive(viewport, driver):
    """Тест на разных разрешениях."""
    width, height = viewport
    driver.set_window_size(width, height)
    driver.get("https://example.com")
    
    # Проверка, что страница загружается и элементы отображаются
    assert "Example" in driver.title
5. Тестирование сетевых условий
5.1 Эмуляция различных сетевых условий
python
class NetworkConditionTester:
    """Тестирование при различных сетевых условиях."""
    
    PROFILES = {
        "fast_3g": {"latency": 75, "throughput": 1.6, "packet_loss": 0.2},
        "slow_3g": {"latency": 300, "throughput": 0.5, "packet_loss": 1.0},
        "4g": {"latency": 50, "throughput": 12.0, "packet_loss": 0.1},
        "edge": {"latency": 500, "throughput": 0.1, "packet_loss": 2.0}
    }
    
    @staticmethod
    def set_network_condition(driver, profile_name: str):
        """Устанавливает сетевые условия в Chrome DevTools."""
        profile = NetworkConditionTester.PROFILES[profile_name]
        
        driver.execute_cdp_cmd("Network.emulateNetworkConditions", {
            "offline": False,
            "latency": profile["latency"],
            "downloadThroughput": profile["throughput"] * 1024 * 1024,
            "uploadThroughput": profile["throughput"] * 1024 * 1024,
        })


@pytest.mark.parametrize("network_profile", ["fast_3g", "slow_3g", "4g", "edge"])
def test_app_on_slow_network(network_profile, driver):
    """Тест приложения при разных скоростях сети."""
    NetworkConditionTester.set_network_condition(driver, network_profile)
    
    start_time = time.time()
    driver.get("https://example.com")
    load_time = time.time() - start_time
    
    # Проверка, что страница загрузилась
    assert "Example" in driver.title
    
    # На медленных сетях загрузка может быть дольше
    if network_profile in ["slow_3g", "edge"]:
        assert load_time > 2.0
6. Шпаргалка
python
# === МАТРИЦА СОВМЕСТИМОСТИ ===
test_matrix = [
    {"os": "Windows", "browser": "Chrome", "version": "latest"},
    {"os": "macOS", "browser": "Safari", "version": "latest"},
    {"os": "Linux", "browser": "Firefox", "version": "latest"},
]

# === ТЕСТИРОВАНИЕ КОНФИГУРАЦИЙ ===
@pytest.mark.parametrize("config", [
    {"debug": True},
    {"debug": False},
])
def test_config(config, monkeypatch):
    for key, value in config.items():
        monkeypatch.setenv(key, str(value))
    # тест

# === ТЕСТИРОВАНИЕ РАЗРЕШЕНИЙ ===
@pytest.mark.parametrize("resolution", [(375, 667), (1920, 1080)])
def test_resolution(resolution, driver):
    driver.set_window_size(*resolution)
    # тест

# === ТЕСТИРОВАНИЕ СЕТИ ===
def test_network_condition(driver):
    driver.execute_cdp_cmd("Network.emulateNetworkConditions", {
        "latency": 300,
        "downloadThroughput": 500 * 1024,
        "uploadThroughput": 500 * 1024,
    })
Контрольные вопросы
Что такое тестирование совместимости и какие виды существуют?

Как составляется матрица совместимости?

Чем конфигурационное тестирование отличается от тестирования совместимости?

Какие инструменты используются для кросс-браузерного тестирования?

Как тестировать адаптивный дизайн?

Как эмулировать различные сетевые условия в тестах?

Как параметризация помогает в тестировании конфигураций?

Какие переменные окружения важно тестировать?

Как использовать Docker для тестирования совместимости?

Какие аспекты нужно учитывать при тестировании на мобильных устройствах?

Следующее занятие: ПР 3.13 — Практическое тестирование совместимости и конфигураций.
