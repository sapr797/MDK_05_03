# Практическое занятие 3.12: Анализ покрытия кода, выявление непокрытых строк. Написание тестов для увеличения покрытия

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться анализировать покрытие кода с помощью `pytest-cov`, выявлять непокрытые строки и ветви, разрабатывать тесты для увеличения покрытия до целевых значений.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Запускать `pytest-cov` с различными опциями для измерения покрытия (ПК 3.2).
2. Анализировать HTML-отчёты покрытия и выявлять непокрытые строки (ОК 02, ОК 05).
3. Разрабатывать тесты для увеличения покрытия строк и ветвей (ПК 3.2, ПК 3.3).
4. Достигать целевого покрытия (≥80%) для модуля (ПК 3.3).

---

## Теоретическая справка

### Команды pytest-cov

```bash
# Базовый запуск с покрытием
pytest --cov=module_name

# С покрытием ветвей
pytest --cov=module_name --cov-branch

# С генерацией HTML-отчёта
pytest --cov=module_name --cov-report=html

# С терминальным отчётом с пропущенными строками
pytest --cov=module_name --cov-report=term-missing

# С проверкой минимального порога
pytest --cov=module_name --cov-fail-under=80
Что означают цвета в HTML-отчёте
Цвет	Значение
Зелёный	Строка выполнена (покрыта)
Красный	Строка не выполнена (не покрыта)
Жёлтый	Частичное покрытие (не все ветви)
Серый	Пропущена (docstring, pass, и т.д.)
Объект тестирования
Листинг 1. Модуль data_parser.py (с низким покрытием)
python
# data_parser.py

import json
import csv
from typing import List, Dict, Any, Optional
from datetime import datetime
import re


class DataParser:
    """
    Парсер данных из различных форматов.
    Текущее покрытие: ~60%
    Непокрытые строки: обработка ошибок, граничные случаи, формат XML
    """
    
    def parse_json(self, json_str: str) -> Optional[Dict]:
        """
        Парсит JSON строку.
        Непокрыто: обработка пустой строки, невалидного JSON
        """
        if not json_str:
            return None
        
        try:
            return json.loads(json_str)
        except json.JSONDecodeError:
            return None
    
    def parse_csv(self, csv_str: str, has_header: bool = True) -> List[Dict]:
        """
        Парсит CSV строку в список словарей.
        Непокрыто: пустая строка, CSV без заголовка, неправильный формат
        """
        if not csv_str:
            return []
        
        lines = csv_str.strip().split('\n')
        if len(lines) < 2:
            return []
        
        if has_header:
            header = lines[0].split(',')
            data = []
            for line in lines[1:]:
                values = line.split(',')
                if len(values) == len(header):
                    data.append(dict(zip(header, values)))
            return data
        else:
            # Непокрытая ветвь: CSV без заголовка
            result = []
            for line in lines:
                result.append(line.split(','))
            return result
    
    def parse_date(self, date_str: str, formats: List[str] = None) -> Optional[datetime]:
        """
        Парсит дату из строки по заданным форматам.
        Непокрыто: пустая строка, все форматы не подошли
        """
        if not date_str:
            return None
        
        if formats is None:
            formats = ["%Y-%m-%d", "%d.%m.%Y", "%m/%d/%Y"]
        
        for fmt in formats:
            try:
                return datetime.strptime(date_str, fmt)
            except ValueError:
                continue
        
        # Непокрытая ветвь: ни один формат не подошёл
        return None
    
    def extract_numbers(self, text: str) -> List[int]:
        """
        Извлекает все числа из текста.
        Непокрыто: пустая строка, текст без чисел
        """
        if not text:
            return []
        
        numbers = re.findall(r'\d+', text)
        return [int(n) for n in numbers]
    
    def validate_email(self, email: str) -> bool:
        """
        Проверяет корректность email адреса.
        Непокрыто: пустая строка, email без домена, специальные символы
        """
        if not email:
            return False
        
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return bool(re.match(pattern, email))
    
    def merge_dicts(self, dict1: Dict, dict2: Dict, strategy: str = "override") -> Dict:
        """
        Объединяет два словаря.
        Непокрыто: стратегия 'skip', 'error', пустые словари
        """
        if not dict1 and not dict2:
            return {}
        
        result = dict1.copy()
        
        for key, value in dict2.items():
            if key in result:
                if strategy == "override":
                    result[key] = value
                elif strategy == "skip":
                    continue
                elif strategy == "error":
                    raise KeyError(f"Conflict on key '{key}'")
                else:
                    # Непокрытая ветвь: неизвестная стратегия
                    raise ValueError(f"Unknown strategy: {strategy}")
            else:
                result[key] = value
        
        return result
    
    def process_data(self, data: Any, operation: str) -> Any:
        """
        Обрабатывает данные в зависимости от операции.
        Непокрыто: неизвестная операция, граничные случаи
        """
        if operation == "json":
            return self.parse_json(data)
        elif operation == "csv":
            return self.parse_csv(data)
        elif operation == "numbers":
            return self.extract_numbers(data)
        elif operation == "email":
            return self.validate_email(data)
        else:
            # Непокрытая ветвь: неизвестная операция
            raise ValueError(f"Unknown operation: {operation}")
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Запуск coverage, анализ отчёта
Средний (варианты 9-17)	Средние студенты	+ выявление непокрытых строк + написание тестов
Сложный (варианты 18-25)	Сильные студенты	+ покрытие ветвей + достижение 90% покрытия
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться запускать pytest-cov и анализировать базовые отчёты о покрытии.

Задания для базового уровня
Задание 1.1. Первый запуск и анализ отчёта (15 минут)
python
# test_data_parser_base.py

import pytest
from data_parser import DataParser


class TestDataParserBase:
    """Базовые тесты для DataParser."""
    
    @pytest.fixture
    def parser(self):
        return DataParser()
    
    # ==========================================================
    # ТЕСТЫ ДЛЯ parse_json
    # ==========================================================
    
    def test_parse_json_valid(self, parser):
        """Парсинг валидного JSON."""
        result = parser.parse_json('{"name": "test", "value": 42}')
        assert result == {"name": "test", "value": 42}
    
    def test_parse_json_empty(self, parser):
        """Пустая строка."""
        result = parser.parse_json("")
        assert result is None
    
    # ==========================================================
    # ТЕСТЫ ДЛЯ parse_csv
    # ==========================================================
    
    def test_parse_csv_valid(self, parser):
        """Парсинг валидного CSV."""
        csv_data = "name,age\nAlice,30\nBob,25"
        result = parser.parse_csv(csv_data)
        
        assert len(result) == 2
        assert result[0]["name"] == "Alice"
        assert result[1]["age"] == "25"
    
    # ==========================================================
    # ТЕСТЫ ДЛЯ parse_date
    # ==========================================================
    
    def test_parse_date_iso(self, parser):
        """Парсинг ISO формата."""
        result = parser.parse_date("2024-01-15")
        assert result is not None
        assert result.year == 2024
        assert result.month == 1
        assert result.day == 15
    
    # ==========================================================
    # ТЕСТЫ ДЛЯ extract_numbers
    # ==========================================================
    
    def test_extract_numbers_valid(self, parser):
        """Извлечение чисел из текста."""
        result = parser.extract_numbers("abc123def456ghi")
        assert result == [123, 456]
    
    # ==========================================================
    # ТЕСТЫ ДЛЯ validate_email
    # ==========================================================
    
    def test_validate_email_valid(self, parser):
        """Валидный email."""
        assert parser.validate_email("user@example.com") is True
    
    def test_validate_email_invalid(self, parser):
        """Невалидный email."""
        assert parser.validate_email("invalid") is False
    
    # ==========================================================
    # ТЕСТЫ ДЛЯ merge_dicts
    # ==========================================================
    
    def test_merge_dicts_override(self, parser):
        """Объединение с перезаписью."""
        dict1 = {"a": 1, "b": 2}
        dict2 = {"b": 3, "c": 4}
        result = parser.merge_dicts(dict1, dict2, "override")
        assert result == {"a": 1, "b": 3, "c": 4}
    
    # ==========================================================
    # ТЕСТЫ ДЛЯ process_data
    # ==========================================================
    
    def test_process_data_json(self, parser):
        """Обработка JSON операции."""
        result = parser.process_data('{"key": "value"}', "json")
        assert result == {"key": "value"}
Задание 1.2. Запуск coverage и анализ отчёта (10 минут)
bash
# Запуск тестов с покрытием
pytest test_data_parser_base.py --cov=data_parser --cov-report=term --cov-report=term-missing
Заполните таблицу на основе вывода:

markdown
## Отчёт о покрытии (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Общая статистика

| Показатель | Значение |
|:---|:---|
| Всего строк в модуле | _____ |
| Выполнено строк | _____ |
| Покрытие строк | _____% |
| Покрытие ветвей | _____% |

### Непокрытые строки (из отчёта)

| Строка | Код | Причина |
|:---|:---|:---|
| _____ | _____ | _____ |
| _____ | _____ | _____ |
| _____ | _____ | _____ |

### Вывод

Текущее покрытие: _____%

Целевое покрытие: 80%

Необходимо добавить тестов для покрытия строк: _________________________________
Задание 1.3. Интерпретация HTML-отчёта (10 минут)
bash
# Генерация HTML-отчёта
pytest test_data_parser_base.py --cov=data_parser --cov-report=html

# Открыть htmlcov/index.html в браузере
Ответьте на вопросы:

Какие строки в функции parse_json отмечены красным?

Какие строки в функции parse_csv отмечены красным?

Какие строки в функции merge_dicts не покрыты?

Какая функция имеет наименьшее покрытие?

Задание 1.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о покрытии (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Текущее покрытие

| Функция | Покрытие строк | Покрытие ветвей |
|:---|:---|:---|
| parse_json | _____% | _____% |
| parse_csv | _____% | _____% |
| parse_date | _____% | _____% |
| extract_numbers | _____% | _____% |
| validate_email | _____% | _____% |
| merge_dicts | _____% | _____% |
| process_data | _____% | _____% |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться выявлять непокрытые строки и писать тесты для их покрытия.

Задания для среднего уровня
Задание 2.1. Выявление непокрытых строк (10 минут)
На основе HTML-отчёта заполните таблицу непокрытых строк:

markdown
## Непокрытые строки и ветви

### parse_json
| Строка | Код | Что нужно протестировать |
|:---|:---|:---|
| _____ | `if not json_str:` | Пустая строка (уже есть) |
| _____ | `except json.JSONDecodeError:` | Невалидный JSON |

### parse_csv
| Строка | Код | Что нужно протестировать |
|:---|:---|:---|
| _____ | `if not csv_str:` | _____ |
| _____ | `if len(lines) < 2:` | _____ |
| _____ | `else:` (без заголовка) | _____ |

### parse_date
| Строка | Код | Что нужно протестировать |
|:---|:---|:---|
| _____ | `if not date_str:` | _____ |
| _____ | `return None` (конец функции) | _____ |

### extract_numbers
| Строка | Код | Что нужно протестировать |
|:---|:---|:---|
| _____ | `if not text:` | _____ |

### validate_email
| Строка | Код | Что нужно протестировать |
|:---|:---|:---|
| _____ | `if not email:` | _____ |

### merge_dicts
| Строка | Код | Что нужно протестировать |
|:---|:---|:---|
| _____ | `elif strategy == "skip":` | _____ |
| _____ | `elif strategy == "error":` | _____ |
| _____ | `else:` (неизвестная стратегия) | _____ |

### process_data
| Строка | Код | Что нужно протестировать |
|:---|:---|:---|
| _____ | `else:` (неизвестная операция) | _____ |
Задание 2.2. Написание тестов для увеличения покрытия (20 минут)
python
# test_data_parser_intermediate.py

import pytest
from data_parser import DataParser


class TestDataParserCoverage:
    """Тесты для увеличения покрытия."""
    
    @pytest.fixture
    def parser(self):
        return DataParser()
    
    # ==========================================================
    # ДОПОЛНИТЕЛЬНЫЕ ТЕСТЫ ДЛЯ parse_json
    # ==========================================================
    
    def test_parse_json_invalid(self, parser):
        """Невалидный JSON."""
        result = parser.parse_json("{invalid json}")
        assert result is None
    
    # ==========================================================
    # ДОПОЛНИТЕЛЬНЫЕ ТЕСТЫ ДЛЯ parse_csv
    # ==========================================================
    
    def test_parse_csv_empty(self, parser):
        """Пустая CSV строка."""
        result = parser.parse_csv("")
        assert result == []
    
    def test_parse_csv_single_line(self, parser):
        """CSV с одной строкой (только заголовок)."""
        result = parser.parse_csv("name,age")
        assert result == []
    
    def test_parse_csv_without_header(self, parser):
        """CSV без заголовка."""
        csv_data = "Alice,30\nBob,25"
        result = parser.parse_csv(csv_data, has_header=False)
        assert result == [["Alice", "30"], ["Bob", "25"]]
    
    def test_parse_csv_mismatched_columns(self, parser):
        """Несовпадение количества колонок."""
        csv_data = "name,age\nAlice,30,extra\nBob,25"
        result = parser.parse_csv(csv_data)
        # Строка с несовпадением игнорируется
        assert len(result) == 1
        assert result[0]["name"] == "Bob"
    
    # ==========================================================
    # ДОПОЛНИТЕЛЬНЫЕ ТЕСТЫ ДЛЯ parse_date
    # ==========================================================
    
    def test_parse_date_empty(self, parser):
        """Пустая строка даты."""
        result = parser.parse_date("")
        assert result is None
    
    def test_parse_date_invalid_format(self, parser):
        """Дата в неверном формате."""
        result = parser.parse_date("invalid-date")
        assert result is None
    
    def test_parse_date_custom_formats(self, parser):
        """Пользовательские форматы."""
        formats = ["%d.%m.%Y"]
        result = parser.parse_date("15.01.2024", formats)
        assert result is not None
        assert result.year == 2024
    
    # ==========================================================
    # ДОПОЛНИТЕЛЬНЫЕ ТЕСТЫ ДЛЯ extract_numbers
    # ==========================================================
    
    def test_extract_numbers_empty(self, parser):
        """Пустая строка."""
        result = parser.extract_numbers("")
        assert result == []
    
    def test_extract_numbers_no_numbers(self, parser):
        """Текст без чисел."""
        result = parser.extract_numbers("no numbers here")
        assert result == []
    
    # ==========================================================
    # ДОПОЛНИТЕЛЬНЫЕ ТЕСТЫ ДЛЯ validate_email
    # ==========================================================
    
    def test_validate_email_empty(self, parser):
        """Пустой email."""
        assert parser.validate_email("") is False
    
    def test_validate_email_no_domain(self, parser):
        """Email без домена."""
        assert parser.validate_email("user@") is False
    
    # ==========================================================
    # ДОПОЛНИТЕЛЬНЫЕ ТЕСТЫ ДЛЯ merge_dicts
    # ==========================================================
    
    def test_merge_dicts_skip_strategy(self, parser):
        """Стратегия 'skip'."""
        dict1 = {"a": 1, "b": 2}
        dict2 = {"b": 3, "c": 4}
        result = parser.merge_dicts(dict1, dict2, "skip")
        assert result == {"a": 1, "b": 2, "c": 4}
    
    def test_merge_dicts_error_strategy(self, parser):
        """Стратегия 'error' при конфликте."""
        dict1 = {"a": 1, "b": 2}
        dict2 = {"b": 3, "c": 4}
        
        with pytest.raises(KeyError, match="Conflict on key 'b'"):
            parser.merge_dicts(dict1, dict2, "error")
    
    def test_merge_dicts_error_strategy_no_conflict(self, parser):
        """Стратегия 'error' без конфликта."""
        dict1 = {"a": 1}
        dict2 = {"b": 2}
        result = parser.merge_dicts(dict1, dict2, "error")
        assert result == {"a": 1, "b": 2}
    
    def test_merge_dicts_unknown_strategy(self, parser):
        """Неизвестная стратегия."""
        dict1 = {"a": 1}
        dict2 = {"b": 2}
        
        with pytest.raises(ValueError, match="Unknown strategy: unknown"):
            parser.merge_dicts(dict1, dict2, "unknown")
    
    def test_merge_dicts_both_empty(self, parser):
        """Оба словаря пустые."""
        result = parser.merge_dicts({}, {})
        assert result == {}
    
    # ==========================================================
    # ДОПОЛНИТЕЛЬНЫЕ ТЕСТЫ ДЛЯ process_data
    # ==========================================================
    
    def test_process_data_unknown_operation(self, parser):
        """Неизвестная операция."""
        with pytest.raises(ValueError, match="Unknown operation: unknown"):
            parser.process_data("data", "unknown")
    
    def test_process_data_csv(self, parser):
        """Операция CSV."""
        csv_data = "name,age\nAlice,30"
        result = parser.process_data(csv_data, "csv")
        assert len(result) == 1
    
    def test_process_data_numbers(self, parser):
        """Операция извлечения чисел."""
        result = parser.process_data("abc123def456", "numbers")
        assert result == [123, 456]
    
    def test_process_data_email(self, parser):
        """Операция валидации email."""
        result = parser.process_data("user@example.com", "email")
        assert result is True
Задание 2.3. Повторный запуск и анализ (10 минут)
bash
# Запуск улучшенных тестов
pytest test_data_parser_intermediate.py --cov=data_parser --cov-report=term-missing --cov-branch
Заполните таблицу сравнения:

markdown
## Сравнение покрытия

| Функция | Было | Стало | Изменение |
|:---|:---|:---|:---|
| parse_json | _____% | _____% | _____% |
| parse_csv | _____% | _____% | _____% |
| parse_date | _____% | _____% | _____% |
| extract_numbers | _____% | _____% | _____% |
| validate_email | _____% | _____% | _____% |
| merge_dicts | _____% | _____% | _____% |
| process_data | _____% | _____% | _____% |
| **Итого** | _____% | _____% | _____% |
Задание 2.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о покрытии (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Итоговое покрытие

| Показатель | Значение |
|:---|:---|
| Покрытие строк | _____% |
| Покрытие ветвей | _____% |

### Остались ли непокрытые строки?

□ Да, остались
□ Нет, покрытие 100%

Если остались, какие строки и почему?

_______________________________________________________________

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться анализировать покрытие ветвей, достигать 90% покрытия и создавать отчёт.

Задания для сложного уровня
Задание 3.1. Анализ покрытия ветвей (10 минут)
bash
# Запуск с анализом ветвей
pytest test_data_parser_intermediate.py --cov=data_parser --cov-branch --cov-report=html
Ответьте на вопросы:

Какие ветви в parse_csv остались непокрытыми?

Какие ветви в parse_date остались непокрытыми?

Какие ветви в merge_dicts остались непокрытыми?

Задание 3.2. Достижение 90% покрытия (20 минут)
Добавьте недостающие тесты для достижения 90% покрытия:

python
# test_data_parser_advanced.py

import pytest
from data_parser import DataParser


class TestDataParserAdvanced:
    """Дополнительные тесты для достижения 90% покрытия."""
    
    @pytest.fixture
    def parser(self):
        return DataParser()
    
    # ==========================================================
    # ДЛЯ ДОСТИЖЕНИЯ 100% ПОКРЫТИЯ parse_csv
    # ==========================================================
    
    def test_parse_csv_blank_lines(self, parser):
        """CSV с пустыми строками."""
        csv_data = "name,age\n\nAlice,30\n\nBob,25\n"
        result = parser.parse_csv(csv_data)
        assert len(result) == 2
    
    # ==========================================================
    # ДЛЯ ДОСТИЖЕНИЯ 100% ПОКРЫТИЯ parse_date
    # ==========================================================
    
    def test_parse_date_multiple_formats_first_fails(self, parser):
        """Первый формат не подходит, второй подходит."""
        result = parser.parse_date("15/01/2024", ["%Y-%m-%d", "%d/%m/%Y"])
        assert result is not None
        assert result.year == 2024
    
    # ==========================================================
    # ДЛЯ ДОСТИЖЕНИЯ 100% ПОКРЫТИЯ validate_email
    # ==========================================================
    
    @pytest.mark.parametrize("email,expected", [
        ("user@example.com", True),
        ("user.name@example.co.uk", True),
        ("user+tag@example.com", True),
        ("user@.com", False),
        ("@example.com", False),
        ("user@example", False),
        ("user@example.c", False),  # слишком короткий домен
        ("user space@example.com", False),
    ])
    def test_validate_email_parametrized(self, parser, email, expected):
        """Параметризованный тест email валидации."""
        assert parser.validate_email(email) == expected
    
    # ==========================================================
    # ДЛЯ ДОСТИЖЕНИЯ 100% ПОКРЫТИЯ extract_numbers
    # ==========================================================
    
    def test_extract_numbers_negative(self, parser):
        """Отрицательные числа (не извлекаются)."""
        result = parser.extract_numbers("temp -10 degrees")
        assert result == [10]  # знак минус не учитывается
    
    def test_extract_numbers_decimal(self, parser):
        """Десятичные числа."""
        result = parser.extract_numbers("price 19.99")
        assert result == [19, 99]  # точка разделяет
    
    # ==========================================================
    # ГРАНИЧНЫЕ СЛУЧАИ ДЛЯ merge_dicts
    # ==========================================================
    
    def test_merge_dicts_with_nested_dicts(self, parser):
        """Вложенные словари (копирование, а не глубокое слияние)."""
        dict1 = {"a": {"x": 1}}
        dict2 = {"a": {"y": 2}}
        result = parser.merge_dicts(dict1, dict2, "override")
        # Второй словарь заменяет значение целиком
        assert result["a"] == {"y": 2}
    
    def test_merge_dicts_with_different_types(self, parser):
        """Разные типы значений."""
        dict1 = {"a": 1}
        dict2 = {"a": "string"}
        result = parser.merge_dicts(dict1, dict2)
        assert result["a"] == "string"
Задание 3.3. Финальный запуск и анализ (10 минут)
bash
# Финальный запуск со всеми опциями
pytest test_data_parser_advanced.py --cov=data_parser --cov-branch --cov-report=term --cov-report=html --cov-fail-under=90
markdown
## Финальный отчёт о покрытии

**Студент:** _________________
**Вариант №:** ___ (18-25)

### Общие показатели

| Метрика | Значение |
|:---|:---|
| Покрытие строк | _____% |
| Покрытие ветвей | _____% |
| Покрытие функций | _____% |

### Покрытие по функциям

| Функция | Строк | Покрытие | Ветвей | Покрытие |
|:---|:---|:---|:---|:---|
| parse_json | | % | | % |
| parse_csv | | % | | % |
| parse_date | | % | | % |
| extract_numbers | | % | | % |
| validate_email | | % | | % |
| merge_dicts | | % | | % |
| process_data | | % | | % |

### Оставшиеся непокрытые строки (если есть)
(скопировать из htmlcov/index.html)

text

### Вывод

- [ ] Целевое покрытие (90%) достигнуто
- [ ] Целевое покрытие не достигнуто (___%)

### Рекомендации

_______________________________________________________________

_______________________________________________________________
Задание 3.4. Скриншоты и отчёт (5 минут)
Сделайте скриншот HTML-отчёта и приложите к отчёту.

markdown
## Итоговый отчёт по практической работе 3.12

**Студент:** _________________
**Группа:** _________________
**Дата:** _________________

### Результаты

| Показатель | Значение |
|:---|:---|
| Начальное покрытие | _____% |
| Конечное покрытие | _____% |
| Добавлено тестов | _____ |
| Покрытие ветвей | _____% |

### Скриншот HTML-отчёта

(вставка скриншота)

### Выводы

_______________________________________________________________

_______________________________________________________________
Карточка студента
text
ПР 3.12. АНАЛИЗ ПОКРЫТИЯ КОДА

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ Запуск coverage (базовый)
□ Анализ отчёта (базовый)
□ Выявление непокрытых строк (средний)
□ Написание тестов (средний)
□ Анализ покрытия ветвей (сложный)
□ Достижение 90% покрытия (сложный)

=== РЕЗУЛЬТАТЫ ===

Покрытие строк: _____%
Покрытие ветвей: _____%

Всего тестов: _____
Добавлено тестов: _____

=== ВРЕМЯ ВЫПОЛНЕНИЯ ===

Подготовка: _____ мин
Выполнение: _____ мин
Оформление: _____ мин

=== ОТЧЁТ ===

Файлы:
- test_data_parser_base.py
- test_data_parser_intermediate.py
- test_data_parser_advanced.py
- coverage_html/

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Coverage не запущен
3 (удовлетворительно)	Базовый	Запуск coverage, анализ отчёта (10+ тестов)
4 (хорошо)	Средний	+ Выявление непокрытых строк + улучшение (20+ тестов)
5 (отлично)	Сложный	+ Покрытие ветвей + 90% покрытие (30+ тестов)
Контрольные вопросы (для защиты)
Как запустить pytest с измерением покрытия?

Что означает красная строка в HTML-отчёте coverage?

Как сгенерировать отчёт с перечислением непокрытых строк?

Чем покрытие строк отличается от покрытия ветвей?

Как настроить минимальный порог покрытия?

Какие строки обычно исключают из измерений покрытия?

Как найти в HTML-отчёте непокрытые строки?

Как увеличить покрытие ветвей с 50% до 100%?

Что означает жёлтая строка в HTML-отчёте?

Как интегрировать coverage в CI/CD?

Следующее занятие: Контрольная работа 3.4 по темам 3.9–3.15.
