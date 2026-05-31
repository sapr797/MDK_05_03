# Лекция 3.10: Углублённое покрытие кода (pytest-cov). Анализ отчётов. Стратегии увеличения покрытия

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить углублённые методы измерения покрытия кода с использованием `pytest-cov`, научиться анализировать отчёты о покрытии, разрабатывать стратегии увеличения покрытия до целевых значений.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Использовать `pytest-cov` для измерения покрытия строк, ветвей и функций (ПК 3.2).
2. Анализировать HTML- и XML-отчёты о покрытии (ОК 02, ОК 05).
3. Выявлять непокрытые строки и ветви кода (ПК 3.3).
4. Разрабатывать стратегии увеличения покрытия до целевых значений (ОК 02, ПК 3.2).

---

## 1. Установка и основы pytest-cov

### 1.1 Установка

```bash
pip install pytest-cov
1.2 Базовые команды
bash
# Базовый запуск с покрытием
pytest --cov=my_module

# С указанием нескольких модулей
pytest --cov=my_module --cov=utils

# С покрытием всех файлов в директории src
pytest --cov=src

# Исключение тестовых файлов из покрытия
pytest --cov=src --cov-ignore=*/test_*
2. Типы покрытия кода
2.1 Сравнение типов покрытия
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ТИПЫ ПОКРЫТИЯ КОДА                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. ПОКРЫТИЕ СТРОК (Line Coverage)                                         │
│      ├── Какие строки кода были выполнены                                   │
│      └── Формула: (выполненные строки / всего строк) × 100%                 │
│                                                                              │
│   2. ПОКРЫТИЕ ВЕТВЕЙ (Branch Coverage)                                      │
│      ├── Какие ветви условий (if/else) были пройдены                        │
│      └── Формула: (выполненные ветви / всего ветвей) × 100%                 │
│                                                                              │
│   3. ПОКРЫТИЕ ФУНКЦИЙ (Function Coverage)                                   │
│      ├── Какие функции были вызваны                                         │
│      └── Формула: (выполненные функции / всего функций) × 100%              │
│                                                                              │
│   4. ПОКРЫТИЕ ОПЕРАТОРОВ (Statement Coverage)                                │
│      ├── Более детальный аналог покрытия строк                              │
│      └── Учитывает множественные операторы на одной строке                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
2.2 Команды для разных типов покрытия
bash
# Только покрытие строк (по умолчанию)
pytest --cov=src --cov-report=term

# Покрытие ветвей (рекомендуется)
pytest --cov=src --cov-branch --cov-report=term

# Все типы покрытия
pytest --cov=src --cov-branch --cov-report=term --cov-report=html
2.3 Пример: разница между покрытием строк и ветвей
python
def process_status(status: str) -> str:
    if status == "active":           # строка 2, ветвь 1
        return "User is active"      # строка 3, ветвь 1
    elif status == "inactive":       # строка 4, ветвь 2
        return "User is inactive"    # строка 5, ветвь 2
    else:                            # строка 6, ветвь 3
        return "Unknown status"      # строка 7, ветвь 3
Тест	Покрытие строк	Покрытие ветвей
process_status("active")	50% (строки 2,3)	33% (ветвь 1)
process_status("active") + process_status("inactive")	75% (строки 2,3,4,5)	66% (ветви 1,2)
Все три вызова	100%	100%
3. Форматы отчётов о покрытии
3.1 Терминальный отчёт (term)
bash
pytest --cov=src --cov-report=term
text
----------- coverage: platform linux, python 3.11.0 -----------
Name                    Stmts   Miss  Cover   Missing
-----------------------------------------------------
src/calculator.py          15      0   100%
src/user_manager.py        42      5    88%   23-25, 67-68
src/file_handler.py        28      8    71%   12-15, 34-36, 52
src/validator.py           10      0   100%
-----------------------------------------------------
TOTAL                      95     13    86%
3.2 Терминальный отчёт с пропущенными строками (term-missing)
bash
pytest --cov=src --cov-report=term-missing
text
----------- coverage: platform linux, python 3.11.0 -----------
Name                    Stmts   Miss  Cover   Missing
-----------------------------------------------------
src/calculator.py          15      0   100%
src/user_manager.py        42      5    88%   23-25, 67-68
src/file_handler.py        28      8    71%   12-15, 34-36, 52
src/validator.py           10      0   100%
-----------------------------------------------------
TOTAL                      95     13    86%
3.3 HTML-отчёт
bash
pytest --cov=src --cov-report=html
open htmlcov/index.html
Что видно в HTML-отчёте:

Зелёные строки — выполнены

Красные строки — не выполнены

Жёлтые строки — частично выполнены (не все ветви)

Серые строки — пропущены (docstring, pass и т.д.)

3.4 XML-отчёт (для CI/CD)
bash
pytest --cov=src --cov-report=xml
XML-отчёт используется в CI/CD для автоматической проверки порога покрытия:

yaml
# .github/workflows/tests.yml
- name: Run tests with coverage
  run: pytest --cov=src --cov-report=xml --cov-fail-under=80
3.5 JSON-отчёт
bash
pytest --cov=src --cov-report=json
4. Анализ отчётов о покрытии
4.1 Что означают цвета в HTML-отчёте
python
def calculate_discount(price: float, discount: float) -> float:
    # Зелёная строка (выполнена)
    if discount < 0:                     # Зелёная
        raise ValueError("Negative discount")  # Красная (не выполнена)
    
    # Жёлтая строка (частичное покрытие)
    if price > 1000:                     # Жёлтая (только одна ветвь)
        return price * (100 - discount) / 100  # Жёлтая
    return price                         # Красная
4.2 Поиск непокрытых строк
python
# Анализируем отчёт: Missing: 23-25, 67-68
# Это значит, что строки 23, 24, 25 и 67, 68 не покрыты

def find_uncovered_lines(coverage_report: str) -> list:
    """Парсит отчёт coverage для поиска непокрытых строк."""
    import re
    pattern = r"Missing:\s+([\d,\s-]+)"
    match = re.search(pattern, coverage_report)
    if match:
        return match.group(1).split(", ")
    return []
4.3 Анализ покрытия ветвей
python
def test_branch_coverage():
    """
    Если ветвь не покрыта, в HTML-отчёте она отмечена жёлтым.
    Например, строка с if может быть жёлтой, если проверена только одна ветка.
    """
    # Чтобы покрыть все ветви, нужно:
    # 1. Вызвать функцию с условием True
    # 2. Вызвать функцию с условием False
    pass
5. Стратегии увеличения покрытия
5.1 Пирамида приоритетов
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    СТРАТЕГИЯ УВЕЛИЧЕНИЯ ПОКРЫТИЯ                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. СНАЧАЛА НЕПОКРЫТЫЕ ВЕТВИ (Branch Coverage)                             │
│      └── Добавить тесты для непроверенных условий                           │
│                                                                              │
│   2. ЗАТЕМ НЕПОКРЫТЫЕ СТРОКИ (Line Coverage)                                 │
│      └── Добавить тесты для непокрытых функций/методов                      │
│                                                                              │
│   3. ПОТОМ ГРАНИЧНЫЕ СЛУЧАИ                                                 │
│      └── Проверить null, пустые значения, границы                           │
│                                                                              │
│   4. В КОНЦЕ ОБРАБОТКА ОШИБОК                                                │
│      └── Добавить тесты для except-блоков                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
5.2 Техники увеличения покрытия
Техника	Описание	Пример
Параметризация	Один тест на несколько случаев	@pytest.mark.parametrize
Тестирование исключений	Проверка всех except-блоков	pytest.raises
Граничные значения	Проверка границ диапазонов	L-1, L, L+1, R-1, R, R+1
Мокирование	Изоляция сложных зависимостей	monkeypatch, unittest.mock
5.3 Пример: увеличение покрытия с 70% до 95%
python
# Исходный код
class OrderProcessor:
    def process(self, order: dict) -> dict:
        if not order.get("items"):           # Непокрыто: ветка if
            return {"error": "Empty order"}  # Непокрыто: строка
        if order["items"] == []:             # Непокрыто: ветка
            return {"error": "No items"}     # Непокрыто: строка
        total = sum(item["price"] for item in order["items"])
        if total > 10000:                    # Непокрыто: ветка
            return {"error": "Order too large"}  # Непокрыто: строка
        return {"success": True, "total": total}

# Тесты для увеличения покрытия
class TestOrderProcessor:
    def test_empty_order(self):
        processor = OrderProcessor()
        result = processor.process({})
        assert result["error"] == "Empty order"
    
    def test_order_without_items(self):
        processor = OrderProcessor()
        result = processor.process({"items": []})
        assert result["error"] == "No items"
    
    def test_order_too_large(self):
        processor = OrderProcessor()
        result = processor.process({"items": [{"price": 20000}]})
        assert result["error"] == "Order too large"
    
    def test_valid_order(self):
        processor = OrderProcessor()
        result = processor.process({"items": [{"price": 100}, {"price": 200}]})
        assert result["success"] is True
        assert result["total"] == 300
6. Настройка покрытия в pyproject.toml
toml
# pyproject.toml

[tool.coverage.run]
source = ["src"]
omit = [
    "*/tests/*",
    "*/__pycache__/*",
    "*/migrations/*",
    "*/venv/*",
    "src/__init__.py",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "if self.debug:",
    "if __name__ == .__main__.:",
    "raise AssertionError",
    "raise NotImplementedError",
    "pass",
]

fail_under = 85
show_missing = True
skip_covered = True

[tool.coverage.html]
directory = "coverage_report"
title = "Coverage Report"
7. CI/CD интеграция
7.1 GitHub Actions с проверкой покрытия
yaml
# .github/workflows/coverage.yml
name: Coverage Check

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  coverage:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        pip install pytest pytest-cov
        pip install -r requirements.txt
    
    - name: Run tests with coverage
      run: |
        pytest --cov=src --cov-branch --cov-report=xml --cov-report=term --cov-fail-under=85
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: unittests
        name: codecov-umbrella
        fail_ci_if_error: true
7.2 GitLab CI
yaml
# .gitlab-ci.yml
coverage:
  stage: test
  script:
    - pip install pytest pytest-cov
    - pytest --cov=src --cov-branch --cov-report=term --cov-report=html --cov-fail-under=85
  coverage: '/TOTAL.*\s+(\d+%)/'
  artifacts:
    paths:
      - coverage_html/
    expire_in: 30 days
8. Шпаргалка
bash
# === БАЗОВЫЙ ЗАПУСК ===
pytest --cov=src

# === С ВЕТВЯМИ ===
pytest --cov=src --cov-branch

# === С ОТЧЁТАМИ ===
pytest --cov=src --cov-report=term --cov-report=html

# === С ПРОВЕРКОЙ ПОРОГА ===
pytest --cov=src --cov-fail-under=85

# === С КОНФИГУРАЦИЕЙ ===
pytest --cov=src --cov-config=.coveragerc

# === ТОЛЬКО ОПРЕДЕЛЁННЫЕ ФАЙЛЫ ===
pytest --cov=src/calculator.py --cov=src/utils.py
python
# === ИСКЛЮЧЕНИЕ СТРОК ИЗ ПОКРЫТИЯ ===
def example():
    # pragma: no cover
    code_that_should_be_excluded()
    
    if debug:  # pragma: no cover
        print("Debug mode")
Контрольные вопросы
Чем отличается покрытие строк от покрытия ветвей?

Как сгенерировать HTML-отчёт о покрытии?

Что означает красная строка в HTML-отчёте coverage?

Как настроить минимальный порог покрытия в CI/CD?

Какие строки обычно исключают из измерений покрытия?

Как найти непокрытые строки в терминальном отчёте?

Почему 100% покрытие не гарантирует отсутствие багов?

Как увеличить покрытие ветвей с 50% до 100%?

Как настроить исключения из покрытия в pyproject.toml?

Как интегрировать coverage в GitHub Actions?

Следующее занятие: ПР 3.10 — Практический анализ покрытия и увеличение
