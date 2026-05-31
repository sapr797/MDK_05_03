# Практическое занятие 3.9: Покрытие кода. Анализ отчёта coverage

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.6:** Покрытие кода и тестирование функций  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться измерять покрытие кода тестами с использованием `pytest-cov`, анализировать отчёты coverage, выявлять непокрытые строки и улучшать тесты для достижения целевого покрытия.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Устанавливать и настраивать `pytest-cov` для измерения покрытия (ПК 3.2).
2. Запускать тесты с генерацией отчётов о покрытии (ПК 3.2, ПК 3.3).
3. Анализировать HTML-отчёты coverage (ОК 02, ОК 05).
4. Выявлять непокрытые строки и улучшать тесты (ПК 3.2, ПК 3.3).

---

## Теоретическая справка

### Что такое покрытие кода?

**Покрытие кода (code coverage)** — метрика, показывающая, какой процент исходного кода выполняется во время тестирования.

| Тип покрытия | Описание | Формула |
|:---|:---|:---|
| **Покрытие строк (Line coverage)** | Какие строки кода были выполнены | `(выполненные строки / всего строк) × 100%` |
| **Покрытие ветвей (Branch coverage)** | Какие ветви условий были пройдены | `(выполненные ветви / всего ветвей) × 100%` |
| **Покрытие функций (Function coverage)** | Какие функции были вызваны | `(выполненные функции / всего функций) × 100%` |

### Установка pytest-cov

```bash
pip install pytest-cov
Запуск тестов с измерением покрытия
bash
# Базовый запуск с покрытием
pytest --cov=имя_модуля

# С указанием директории
pytest --cov=src

# С несколькими модулями
pytest --cov=src.calculator --cov=src.user_manager

# С генерацией HTML-отчёта
pytest --cov=src --cov-report=html

# С генерацией XML-отчёта (для CI/CD)
pytest --cov=src --cov-report=xml

# С терминальным отчётом (кратким)
pytest --cov=src --cov-report=term

# С пропуском тестовых файлов
pytest --cov=src --cov-report=term --cov-fail-under=80
Объект тестирования
Листинг 1. Модуль collection_manager.py (полная версия)
python
# collection_manager.py

from typing import List, Dict, Any, Union, Optional
from collections import defaultdict


def find_duplicates(items: List[Any]) -> List[Any]:
    """
    Находит все дублирующиеся элементы в списке.
    """
    if not isinstance(items, list):
        raise TypeError("Input must be a list")
    
    seen = set()
    duplicates = set()
    
    for item in items:
        if item in seen:
            duplicates.add(item)
        else:
            seen.add(item)
    
    result = []
    for item in items:
        if item in duplicates and item not in result:
            result.append(item)
    
    return result


def get_unique_elements(items: List[Any]) -> List[Any]:
    """
    Возвращает список уникальных элементов с сохранением порядка.
    """
    if not isinstance(items, list):
        raise TypeError("Input must be a list")
    
    seen = set()
    result = []
    
    for item in items:
        if item not in seen:
            seen.add(item)
            result.append(item)
    
    return result


def merge_dicts(
    dict1: Dict[Any, Any],
    dict2: Dict[Any, Any],
    strategy: str = "override"
) -> Dict[Any, Any]:
    """
    Объединяет два словаря с заданной стратегией.
    """
    if not isinstance(dict1, dict) or not isinstance(dict2, dict):
        raise TypeError("Both arguments must be dictionaries")
    
    if strategy not in ["override", "skip", "error", "merge_lists"]:
        raise ValueError(f"Unknown strategy: {strategy}")
    
    result = dict1.copy()
    
    for key, value in dict2.items():
        if key in result:
            if strategy == "override":
                result[key] = value
            elif strategy == "skip":
                continue
            elif strategy == "error":
                raise KeyError(f"Conflict on key '{key}'")
            elif strategy == "merge_lists":
                if isinstance(result[key], list) and isinstance(value, list):
                    result[key] = result[key] + value
                else:
                    result[key] = value
        else:
            result[key] = value
    
    return result


def group_by_key(elements: List[Dict], key: str) -> Dict[Any, List[Dict]]:
    """
    Группирует список словарей по значению указанного ключа.
    """
    if not isinstance(elements, list):
        raise TypeError("Elements must be a list")
    
    result = defaultdict(list)
    
    for element in elements:
        if not isinstance(element, dict):
            raise TypeError("Each element must be a dictionary")
        
        if key not in element:
            raise KeyError(f"Key '{key}' not found in element {element}")
        
        result[element[key]].append(element)
    
    return dict(result)


def flatten_list(nested: List[Any], depth: int = -1) -> List[Any]:
    """
    Разворачивает вложенный список на указанную глубину.
    """
    if not isinstance(nested, list):
        raise TypeError("Input must be a list")
    
    if depth == 0:
        return nested
    
    result = []
    for item in nested:
        if isinstance(item, list) and (depth == -1 or depth > 0):
            result.extend(flatten_list(item, depth - 1 if depth > 0 else -1))
        else:
            result.append(item)
    
    return result


def chunk_list(items: List[Any], chunk_size: int) -> List[List[Any]]:
    """
    Разбивает список на чанки указанного размера.
    """
    if not isinstance(items, list):
        raise TypeError("Items must be a list")
    
    if not isinstance(chunk_size, int):
        raise TypeError("Chunk size must be an integer")
    
    if chunk_size <= 0:
        raise ValueError("Chunk size must be positive")
    
    result = []
    for i in range(0, len(items), chunk_size):
        result.append(items[i:i + chunk_size])
    
    return result


def count_frequencies(items: List[Any]) -> Dict[Any, int]:
    """
    Подсчитывает частоту встречаемости элементов в списке.
    """
    if not isinstance(items, list):
        raise TypeError("Input must be a list")
    
    frequencies = {}
    for item in items:
        frequencies[item] = frequencies.get(item, 0) + 1
    
    return frequencies
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Установка и базовый запуск coverage
Средний (варианты 9-17)	Средние студенты	Анализ HTML-отчёта, улучшение покрытия
Сложный (варианты 18-25)	Сильные студенты	Полный анализ, достижение 90% покрытия
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться устанавливать pytest-cov, запускать тесты с измерением покрытия и читать терминальный отчёт.

Задания для базового уровня
Задание 1.1. Установка и первичный запуск (5 минут)
Выполните в терминале:

bash
# Установка pytest-cov
pip install pytest-cov

# Проверка установки
pytest --version
pip show pytest-cov
Задание 1.2. Создание простых тестов (20 минут)
Создайте файл test_collection_manager_base.py с минимальными тестами.

python
# test_collection_manager_base.py

import pytest
from collection_manager import (
    find_duplicates,
    get_unique_elements,
    merge_dicts,
    group_by_key,
    flatten_list,
    chunk_list,
    count_frequencies
)


class TestFindDuplicates:
    """Тесты для find_duplicates."""
    
    def test_empty_list(self):
        assert find_duplicates([]) == []
    
    def test_no_duplicates(self):
        assert find_duplicates([1, 2, 3]) == []
    
    def test_with_duplicates(self):
        assert find_duplicates([1, 2, 2, 3, 3, 3]) == [2, 3]
    
    def test_type_error(self):
        with pytest.raises(TypeError, match="Input must be a list"):
            find_duplicates("not a list")


class TestGetUniqueElements:
    """Тесты для get_unique_elements."""
    
    def test_empty_list(self):
        assert get_unique_elements([]) == []
    
    def test_all_unique(self):
        assert get_unique_elements([1, 2, 3]) == [1, 2, 3]
    
    def test_with_duplicates(self):
        assert get_unique_elements([1, 2, 2, 3, 1]) == [1, 2, 3]
    
    def test_type_error(self):
        with pytest.raises(TypeError, match="Input must be a list"):
            get_unique_elements(123)


class TestMergeDicts:
    """Тесты для merge_dicts."""
    
    def test_override_no_conflict(self):
        result = merge_dicts({"a": 1}, {"b": 2}, "override")
        assert result == {"a": 1, "b": 2}
    
    def test_override_with_conflict(self):
        result = merge_dicts({"a": 1, "b": 2}, {"b": 20, "c": 30}, "override")
        assert result == {"a": 1, "b": 20, "c": 30}
    
    def test_skip(self):
        result = merge_dicts({"a": 1, "b": 2}, {"b": 20, "c": 30}, "skip")
        assert result == {"a": 1, "b": 2, "c": 30}
    
    def test_error_no_conflict(self):
        result = merge_dicts({"a": 1}, {"b": 2}, "error")
        assert result == {"a": 1, "b": 2}
    
    def test_type_error(self):
        with pytest.raises(TypeError, match="Both arguments must be dictionaries"):
            merge_dicts("not dict", {"a": 1})


class TestGroupByKey:
    """Тесты для group_by_key."""
    
    def test_empty_list(self):
        assert group_by_key([], "key") == {}
    
    def test_single_group(self):
        elements = [{"type": "A", "id": 1}, {"type": "A", "id": 2}]
        result = group_by_key(elements, "type")
        assert len(result["A"]) == 2


class TestFlattenList:
    """Тесты для flatten_list."""
    
    def test_flat_list(self):
        assert flatten_list([1, 2, 3]) == [1, 2, 3]
    
    def test_one_level(self):
        assert flatten_list([1, [2, 3], 4]) == [1, 2, 3, 4]


class TestChunkList:
    """Тесты для chunk_list."""
    
    def test_chunk_basic(self):
        result = chunk_list([1, 2, 3, 4, 5], 2)
        assert result == [[1, 2], [3, 4], [5]]
    
    def test_empty_list(self):
        assert chunk_list([], 3) == []


class TestCountFrequencies:
    """Тесты для count_frequencies."""
    
    def test_basic(self):
        result = count_frequencies([1, 2, 2, 3, 3, 3])
        assert result == {1: 1, 2: 2, 3: 3}
    
    def test_empty_list(self):
        assert count_frequencies([]) == {}
Задание 1.3. Запуск тестов с покрытием (10 минут)
Выполните команды и сохраните вывод:

bash
# Базовый запуск с покрытием
pytest test_collection_manager_base.py --cov=collection_manager --cov-report=term

# С кратким отчётом
pytest test_collection_manager_base.py --cov=collection_manager --cov-report=term-missing

# С подробным отчётом
pytest test_collection_manager_base.py --cov=collection_manager --cov-report=term --cov-branch
Задание 1.4. Анализ отчёта (10 минут)
Зафиксируйте результаты:

markdown
## Отчёт о покрытии (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Результаты запуска

| Функция | Строк в модуле | Выполнено | Покрытие |
|:---|:---|:---|:---|
| find_duplicates | | | % |
| get_unique_elements | | | % |
| merge_dicts | | | % |
| group_by_key | | | % |
| flatten_list | | | % |
| chunk_list | | | % |
| count_frequencies | | | % |
| **Итого** | | | **___%** |

### Непокрытые строки (из отчёта)
(вставьте вывод --cov-report=term-missing)

text

### Вывод

Текущее покрытие: ___%

Целевое покрытие: 80%

Необходимо добавить тестов для: _________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться генерировать и анализировать HTML-отчёты coverage, выявлять непокрытые строки и улучшать тесты.

Задания для среднего уровня
Задание 2.1. Генерация HTML-отчёта (10 минут)
bash
# Генерация HTML-отчёта
pytest test_collection_manager_base.py --cov=collection_manager --cov-report=html

# Открыть отчёт в браузере
# Windows: start htmlcov/index.html
# macOS: open htmlcov/index.html
# Linux: xdg-open htmlcov/index.html
Задание 2.2. Анализ HTML-отчёта (15 минут)
Откройте htmlcov/index.html в браузере и ответьте на вопросы:

markdown
## Анализ HTML-отчёта

1. Общее покрытие модуля: _____%

2. Какая функция имеет наименьшее покрытие? _____________

3. Какая строка в `find_duplicates` не покрыта? _________

4. Какая строка в `merge_dicts` не покрыта? ___________

5. Какая ветвь в `group_by_key` не протестирована? ______

6. Какая ветвь в `flatten_list` не протестирована? ______

7. Какое исключение в `chunk_list` не проверено? ________

8. Какая функция полностью покрыта? ____________________
Задание 2.3. Улучшение тестов (20 минут)
Добавьте недостающие тесты для достижения покрытия ≥ 80%.

python
# test_collection_manager_improved.py

# Добавьте тесты для непокрытых строк:

class TestAdditionalCoverage:
    """Дополнительные тесты для улучшения покрытия."""
    
    # ==========================================================
    # TODO: Добавить тесты для find_duplicates
    # Непокрытые строки: какая ветвь не протестирована?
    # ==========================================================
    
    def test_find_duplicates_with_strings(self):
        """Строки в списке."""
        result = find_duplicates(["a", "b", "a", "c", "b"])
        assert result == ["a", "b"]
    
    def test_find_duplicates_with_mixed_types(self):
        """Смешанные типы."""
        result = find_duplicates([1, "1", 1, "1"])
        assert result == [1, "1"]
    
    def test_find_duplicates_with_none(self):
        """None как элемент."""
        result = find_duplicates([None, 1, None, 2])
        assert result == [None]
    
    # ==========================================================
    # TODO: Добавить тесты для merge_dicts
    # Непокрытые стратегии: merge_lists
    # ==========================================================
    
    def test_merge_dicts_merge_lists_both_lists(self):
        """merge_lists: оба значения — списки."""
        dict1 = {"a": [1, 2]}
        dict2 = {"a": [3, 4]}
        result = merge_dicts(dict1, dict2, "merge_lists")
        assert result == {"a": [1, 2, 3, 4]}
    
    def test_merge_dicts_merge_lists_not_both_lists(self):
        """merge_lists: одно значение не список."""
        dict1 = {"a": [1, 2]}
        dict2 = {"a": 3}
        result = merge_dicts(dict1, dict2, "merge_lists")
        assert result == {"a": 3}  # override при несовместимости
    
    def test_merge_dicts_error_conflict(self):
        """error: конфликт ключей."""
        with pytest.raises(KeyError, match="Conflict on key 'a'"):
            merge_dicts({"a": 1}, {"a": 2}, "error")
    
    def test_merge_dicts_invalid_strategy(self):
        """Неизвестная стратегия."""
        with pytest.raises(ValueError, match="Unknown strategy: invalid"):
            merge_dicts({"a": 1}, {"b": 2}, "invalid")
    
    # ==========================================================
    # TODO: Добавить тесты для group_by_key
    # Непокрытые строки: исключения
    # ==========================================================
    
    def test_group_by_key_element_not_dict(self):
        """Элемент не словарь."""
        elements = [{"id": 1}, "not dict"]
        with pytest.raises(TypeError, match="Each element must be a dictionary"):
            group_by_key(elements, "id")
    
    def test_group_by_key_key_not_found(self):
        """Ключ отсутствует в элементе."""
        elements = [{"name": "test"}]
        with pytest.raises(KeyError, match="Key 'id' not found"):
            group_by_key(elements, "id")
    
    # ==========================================================
    # TODO: Добавить тесты для flatten_list
    # Непокрытые строки: глубина разворачивания
    # ==========================================================
    
    def test_flatten_list_depth_1(self):
        """Глубина разворачивания = 1."""
        nested = [1, [2, 3], [4, [5, 6]]]
        result = flatten_list(nested, depth=1)
        assert result == [1, 2, 3, 4, [5, 6]]
    
    def test_flatten_list_depth_2(self):
        """Глубина разворачивания = 2."""
        nested = [1, [2, [3, [4, 5]]]]
        result = flatten_list(nested, depth=2)
        assert result == [1, 2, 3, [4, 5]]
    
    def test_flatten_list_depth_0(self):
        """Глубина 0 — без изменений."""
        nested = [1, [2, 3]]
        result = flatten_list(nested, depth=0)
        assert result == nested
    
    def test_flatten_list_not_a_list(self):
        """Входной параметр не список."""
        with pytest.raises(TypeError, match="Input must be a list"):
            flatten_list("not a list")
    
    # ==========================================================
    # TODO: Добавить тесты для chunk_list
    # Непокрытые строки: исключения
    # ==========================================================
    
    def test_chunk_list_invalid_chunk_size_type(self):
        """Неверный тип chunk_size."""
        with pytest.raises(TypeError, match="Chunk size must be an integer"):
            chunk_list([1, 2, 3], "2")
    
    def test_chunk_list_zero_chunk_size(self):
        """Chunk_size = 0."""
        with pytest.raises(ValueError, match="Chunk size must be positive"):
            chunk_list([1, 2, 3], 0)
    
    def test_chunk_list_negative_chunk_size(self):
        """Отрицательный chunk_size."""
        with pytest.raises(ValueError, match="Chunk size must be positive"):
            chunk_list([1, 2, 3], -1)
    
    def test_chunk_list_not_a_list(self):
        """items не список."""
        with pytest.raises(TypeError, match="Items must be a list"):
            chunk_list("not a list", 2)
    
    # ==========================================================
    # TODO: Добавить тесты для count_frequencies
    # Непокрытые строки: исключения
    # ==========================================================
    
    def test_count_frequencies_not_a_list(self):
        """Входной параметр не список."""
        with pytest.raises(TypeError, match="Input must be a list"):
            count_frequencies("not a list")
Задание 2.4. Повторный запуск и анализ (10 минут)
bash
# Запуск с улучшенными тестами
pytest test_collection_manager_improved.py --cov=collection_manager --cov-report=html --cov-report=term
Зафиксируйте результаты:

markdown
## Отчёт о покрытии (средний уровень)

### Покрытие ДО улучшения: ___%

### Покрытие ПОСЛЕ улучшения: ___%

### Изменение покрытия по функциям

| Функция | Было | Стало | Изменение |
|:---|:---|:---|:---|
| find_duplicates | % | % | +% |
| merge_dicts | % | % | +% |
| group_by_key | % | % | +% |
| flatten_list | % | % | +% |
| chunk_list | % | % | +% |
| count_frequencies | % | % | +% |

### Остались ли непокрытые строки?

□ Да, остались
□ Нет, покрытие 100%

Если остались, какие строки и почему?

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться анализировать покрытие ветвей, работать с конфигурацией coverage, достигать 90% покрытия и интегрировать coverage в CI/CD.

Задания для сложного уровня
Задание 3.1. Анализ покрытия ветвей (15 минут)
bash
# Запуск с анализом ветвей
pytest --cov=collection_manager --cov-report=term --cov-branch --cov-report=html
markdown
## Анализ покрытия ветвей

### Что такое покрытие ветвей?

Покрытие ветвей показывает, какие ветви условий (`if/else`) были протестированы.

### Текущее покрытие ветвей: ___%

### Функции с низким покрытием ветвей:

| Функция | Покрытие строк | Покрытие ветвей | Разница |
|:---|:---|:---|:---|
| | % | % | % |
| | % | % | % |

### Непокрытые ветви:

1. `merge_dicts` — какая ветвь не покрыта? _______________
2. `flatten_list` — какая ветвь не покрыта? ______________
3. `group_by_key` — какая ветвь не покрыта? _____________

### Какие тесты нужно добавить для покрытия этих ветвей?

_______________________________________________________________
Задание 3.2. Конфигурация coverage (10 минут)
Создайте файл .coveragerc или добавьте настройки в pyproject.toml:

toml
# pyproject.toml (добавить в существующий)

[tool.coverage.run]
source = ["."]
omit = [
    "tests/*",
    "*/__pycache__/*",
    "*/site-packages/*",
    "test_*.py",
    "conftest.py"
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "pass",
    "return NotImplemented"
]

fail_under = 85
show_missing = True
skip_covered = True

[tool.coverage.html]
directory = "coverage_report"
title = "Coverage Report for Collection Manager"
Задание: Объясните, для чего нужна каждая секция.

Задание 3.3. Достижение целевого покрытия (25 минут)
Цель: Достичь покрытия ≥ 90% для всего модуля.

Список задач для выполнения:

markdown
## Чек-лист достижения 90% покрытия

### find_duplicates (цель: 100%)
- [ ] Тест с пустым списком
- [ ] Тест без дубликатов
- [ ] Тест с дубликатами
- [ ] Тест со строками
- [ ] Тест со смешанными ти
