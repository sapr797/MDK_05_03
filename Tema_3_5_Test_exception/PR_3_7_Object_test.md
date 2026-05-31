# Практическое занятие 3.7: Тестирование функций для работы с коллекциями

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.5:** Тестирование исключений и коллекций  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться тестировать функции для работы с коллекциями: поиск дубликатов, объединение словарей, получение уникальных элементов. Освоить обработку краевых случаев и исключений.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Тестировать функции поиска дубликатов в списках (ПК 3.2).
2. Тестировать функции объединения словарей с различными стратегиями (ПК 3.2, ПК 3.3).
3. Тестировать функции получения уникальных элементов (ПК 3.2).
4. Обрабатывать пограничные случаи и исключения (ПК 3.3).

---

## Теоретическая справка

### Функции для работы с коллекциями

| Функция | Назначение | Вход | Выход |
|:---|:---|:---|:---|
| `find_duplicates` | Поиск дублирующихся элементов | `list` | `list` уникальных дубликатов |
| `merge_dicts` | Объединение словарей | `dict, dict, strategy` | `dict` |
| `get_unique_elements` | Получение уникальных элементов | `list` | `list` |

### Стратегии объединения словарей

| Стратегия | Поведение при конфликте ключей |
|:---|:---|
| `"override"` | Значение из второго словаря заменяет первое |
| `"skip"` | Сохраняется значение из первого словаря |
| `"error"` | Выбрасывается исключение `KeyError` |
| `"merge_lists"` | Объединяет списки, если оба значения — списки |

---

## Объект тестирования

### Листинг 1. Модуль collection_manager.py

```python
# collection_manager.py

from typing import List, Dict, Any, Union, Optional
from collections.abc import Iterable


def find_duplicates(items: List[Any]) -> List[Any]:
    """
    Находит все дублирующиеся элементы в списке.
    
    Args:
        items: Список элементов
    
    Returns:
        Список уникальных элементов, которые встречаются более одного раза
        Порядок сохраняется (по первому вхождению)
    
    Examples:
        >>> find_duplicates([1, 2, 2, 3, 3, 3, 4])
        [2, 3]
    
    Raises:
        TypeError: Если items не является списком
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
    
    # Сохраняем порядок по первому вхождению
    result = []
    for item in items:
        if item in duplicates and item not in result:
            result.append(item)
    
    return result


def merge_dicts(
    dict1: Dict[Any, Any],
    dict2: Dict[Any, Any],
    strategy: str = "override"
) -> Dict[Any, Any]:
    """
    Объединяет два словаря с заданной стратегией разрешения конфликтов.
    
    Args:
        dict1: Первый словарь
        dict2: Второй словарь
        strategy: Стратегия разрешения конфликтов
            - "override": значение из dict2 заменяет значение из dict1
            - "skip": сохраняется значение из dict1
            - "error": выбрасывается KeyError при конфликте
            - "merge_lists": объединяет списки (если оба значения — списки)
    
    Returns:
        Объединённый словарь
    
    Raises:
        TypeError: Если dict1 или dict2 не являются словарями
        KeyError: При strategy="error" и наличии конфликтующих ключей
        ValueError: При неизвестной стратегии
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
                    # При несовместимых типах — override
                    result[key] = value
        else:
            result[key] = value
    
    return result


def get_unique_elements(items: List[Any]) -> List[Any]:
    """
    Возвращает список уникальных элементов с сохранением порядка.
    
    Args:
        items: Входной список
    
    Returns:
        Список уникальных элементов в порядке первого появления
    
    Examples:
        >>> get_unique_elements([1, 2, 2, 3, 1, 4])
        [1, 2, 3, 4]
    
    Raises:
        TypeError: Если items не является списком
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


# Дополнительные функции для сложного уровня

def group_by_key(elements: List[Dict], key: str) -> Dict[Any, List[Dict]]:
    """
    Группирует список словарей по значению указанного ключа.
    
    Args:
        elements: Список словарей
        key: Ключ для группировки
    
    Returns:
        Словарь, где ключи — значения ключа group_key,
        значения — списки элементов
    
    Raises:
        TypeError: Если elements не список
        KeyError: Если key отсутствует в каком-либо элементе
    """
    if not isinstance(elements, list):
        raise TypeError("Elements must be a list")
    
    result = {}
    
    for element in elements:
        if not isinstance(element, dict):
            raise TypeError("Each element must be a dictionary")
        
        if key not in element:
            raise KeyError(f"Key '{key}' not found in element {element}")
        
        group_value = element[key]
        if group_value not in result:
            result[group_value] = []
        result[group_value].append(element)
    
    return result


def flatten_list(nested: List[Any], depth: int = -1) -> List[Any]:
    """
    Разворачивает вложенный список на указанную глубину.
    
    Args:
        nested: Вложенный список
        depth: Глубина разворачивания (-1 = полностью)
    
    Returns:
        Развёрнутый список
    
    Raises:
        TypeError: Если nested не список
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
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	find_duplicates, get_unique_elements
Средний (варианты 9-17)	Средние студенты	+ merge_dicts со всеми стратегиями
Сложный (варианты 18-25)	Сильные студенты	+ group_by_key, flatten_list + пограничные случаи
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться тестировать функции поиска дубликатов и получения уникальных элементов.

Задания для базового уровня
Задание 1.1. Тестирование find_duplicates (20 минут)
python
# test_collection_manager_base.py

import pytest
from collection_manager import find_duplicates


class TestFindDuplicates:
    """Тесты для функции find_duplicates."""
    
    # ==========================================================
    # ПОЗИТИВНЫЕ СЦЕНАРИИ
    # ==========================================================
    
    def test_empty_list(self):
        """Пустой список → пустой результат."""
        assert find_duplicates([]) == []
    
    def test_no_duplicates(self):
        """Список без дубликатов."""
        assert find_duplicates([1, 2, 3, 4]) == []
    
    def test_one_duplicate(self):
        """Один дублирующийся элемент."""
        assert find_duplicates([1, 2, 2, 3]) == [2]
    
    def test_multiple_duplicates(self):
        """Несколько дублирующихся элементов."""
        assert find_duplicates([1, 2, 2, 3, 3, 3, 4]) == [2, 3]
    
    def test_all_duplicates(self):
        """Все элементы дублируются."""
        assert find_duplicates([1, 1, 2, 2, 3, 3]) == [1, 2, 3]
    
    def test_strings(self):
        """Список строк."""
        assert find_duplicates(["a", "b", "a", "c", "b"]) == ["a", "b"]
    
    def test_mixed_types(self):
        """Смешанные типы данных."""
        assert find_duplicates([1, "1", 1, "1", 2]) == [1, "1"]
    
    def test_duplicates_order_preserved(self):
        """Порядок дубликатов сохраняется."""
        result = find_duplicates([3, 1, 2, 1, 3, 4, 2])
        # Первое вхождение 3, потом 1, потом 2
        assert result == [3, 1, 2]
    
    def test_none_values(self):
        """None как элемент списка."""
        assert find_duplicates([None, 1, None, 2]) == [None]
    
    def test_booleans(self):
        """Булевы значения."""
        assert find_duplicates([True, False, True, True, False]) == [True, False]
    
    # ==========================================================
    # НЕГАТИВНЫЕ СЦЕНАРИИ (ИСКЛЮЧЕНИЯ)
    # ==========================================================
    
    def test_not_a_list(self):
        """Входной параметр не является списком."""
        with pytest.raises(TypeError, match="Input must be a list"):
            find_duplicates("not a list")
    
    def test_none_input(self):
        """None как входной параметр."""
        with pytest.raises(TypeError, match="Input must be a list"):
            find_duplicates(None)
    
    def test_dict_input(self):
        """Словарь вместо списка."""
        with pytest.raises(TypeError, match="Input must be a list"):
            find_duplicates({"a": 1, "b": 2})
Задание 1.2. Параметризация тестов find_duplicates (10 минут)
python
class TestFindDuplicatesParametrized:
    """Параметризованные тесты для find_duplicates."""
    
    @pytest.mark.parametrize("input_list,expected", [
        ([], []),
        ([1], []),
        ([1, 1], [1]),
        ([1, 2, 1], [1]),
        ([1, 2, 2, 3, 3], [2, 3]),
        (["a", "b", "a", "c"], ["a"]),
        ([1, "1", 1, "1"], [1, "1"]),
        ([True, False, True], [True]),
        ([None, None, 1], [None]),
    ])
    def test_find_duplicates_param(self, input_list, expected):
        """Параметризованный тест."""
        assert find_duplicates(input_list) == expected
    
    @pytest.mark.parametrize("invalid_input", [
        "string",
        123,
        3.14,
        None,
        {"key": "value"},
        (1, 2, 3),
    ])
    def test_invalid_inputs(self, invalid_input):
        """Невалидные входные данные."""
        with pytest.raises(TypeError, match="Input must be a list"):
            find_duplicates(invalid_input)
Задание 1.3. Тестирование get_unique_elements (15 минут)
python
from collection_manager import get_unique_elements


class TestGetUniqueElements:
    """Тесты для функции get_unique_elements."""
    
    # ==========================================================
    # ПОЗИТИВНЫЕ СЦЕНАРИИ
    # ==========================================================
    
    def test_empty_list(self):
        """Пустой список."""
        assert get_unique_elements([]) == []
    
    def test_all_unique(self):
        """Все элементы уникальны."""
        assert get_unique_elements([1, 2, 3, 4]) == [1, 2, 3, 4]
    
    def test_with_duplicates(self):
        """С дубликатами."""
        assert get_unique_elements([1, 2, 2, 3, 1, 4]) == [1, 2, 3, 4]
    
    def test_order_preserved(self):
        """Порядок сохраняется."""
        assert get_unique_elements([3, 1, 2, 1, 3, 4]) == [3, 1, 2, 4]
    
    def test_strings(self):
        """Строки."""
        assert get_unique_elements(["a", "b", "a", "c"]) == ["a", "b", "c"]
    
    def test_mixed_types(self):
        """Смешанные типы."""
        assert get_unique_elements([1, "1", 1, "1", 2]) == [1, "1", 2]
    
    def test_none_values(self):
        """None как элемент."""
        assert get_unique_elements([None, 1, None, 2]) == [None, 1, 2]
    
    def test_booleans(self):
        """Булевы значения."""
        assert get_unique_elements([True, False, True, True]) == [True, False]
    
    def test_large_list(self):
        """Большой список."""
        large_list = list(range(1000)) * 2
        result = get_unique_elements(large_list)
        assert len(result) == 1000
        assert result == list(range(1000))
    
    # ==========================================================
    # НЕГАТИВНЫЕ СЦЕНАРИИ
    # ==========================================================
    
    def test_not_a_list(self):
        """Входной параметр не список."""
        with pytest.raises(TypeError, match="Input must be a list"):
            get_unique_elements("not a list")
    
    def test_none_input(self):
        """None как входной параметр."""
        with pytest.raises(TypeError, match="Input must be a list"):
            get_unique_elements(None)
Задание 1.4. Сравнение find_duplicates и get_unique_elements (5 минут)
python
class TestComparison:
    """Сравнение функций find_duplicates и get_unique_elements."""
    
    def test_relationship(self):
        """Проверка взаимосвязи функций."""
        data = [1, 2, 2, 3, 3, 3, 4]
        
        unique = get_unique_elements(data)
        duplicates = find_duplicates(data)
        
        # Дубликаты — это элементы, которые есть в unique,
        # но встречаются в данных более одного раза
        for item in duplicates:
            assert item in unique
            assert data.count(item) > 1
        
        # Уникальные элементы — это все элементы без повторов
        assert set(unique) == set(data)
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться тестировать функцию merge_dicts с различными стратегиями и обработкой исключений.

Задания для среднего уровня
Задание 2.1. Тестирование merge_dicts со стратегией "override" (15 минут)
python
# test_collection_manager_intermediate.py

import pytest
from collection_manager import merge_dicts


class TestMergeDictsOverride:
    """Тесты стратегии 'override'."""
    
    def test_no_conflicts(self):
        """Нет конфликтующих ключей."""
        dict1 = {"a": 1, "b": 2}
        dict2 = {"c": 3, "d": 4}
        
        result = merge_dicts(dict1, dict2, strategy="override")
        
        assert result == {"a": 1, "b": 2, "c": 3, "d": 4}
    
    def test_with_conflicts(self):
        """Конфликтующие ключи — dict2 перезаписывает dict1."""
        dict1 = {"a": 1, "b": 2, "c": 3}
        dict2 = {"b": 20, "c": 30, "d": 40}
        
        result = merge_dicts(dict1, dict2, strategy="override")
        
        assert result == {"a": 1, "b": 20, "c": 30, "d": 40}
    
    def test_empty_dicts(self):
        """Пустые словари."""
        assert merge_dicts({}, {}, "override") == {}
        assert merge_dicts({"a": 1}, {}, "override") == {"a": 1}
        assert merge_dicts({}, {"b": 2}, "override") == {"b": 2}
    
    def test_nested_dicts(self):
        """Вложенные словари (замена целиком)."""
        dict1 = {"a": {"x": 1}, "b": 2}
        dict2 = {"a": {"y": 2}, "c": 3}
        
        result = merge_dicts(dict1, dict2, "override")
        
        # Вложенный словарь заменяется целиком, а не сливается
        assert result == {"a": {"y": 2}, "b": 2, "c": 3}
        assert result["a"] == {"y": 2}  # Не {"x": 1, "y": 2}
    
    def test_different_types(self):
        """Разные типы значений."""
        dict1 = {"a": 1, "b": "string"}
        dict2 = {"a": [1, 2], "b": None}
        
        result = merge_dicts(dict1, dict2, "override")
        
        assert result == {"a": [1, 2], "b": None}
Задание 2.2. Тестирование стратегий "skip" и "error" (15 минут)
python
class TestMergeDictsSkip:
    """Тесты стратегии 'skip'."""
    
    def test_skip_conflicts(self):
        """Конфликтующие ключи — сохраняются значения из dict1."""
        dict1 = {"a": 1, "b": 2, "c": 3}
        dict2 = {"b": 20, "c": 30, "d": 40}
        
        result = merge_dicts(dict1, dict2, strategy="skip")
        
        assert result == {"a": 1, "b": 2, "c": 3, "d": 40}
    
    def test_skip_no_conflicts(self):
        """Без конфликтов — как обычное объединение."""
        dict1 = {"a": 1}
        dict2 = {"b": 2}
        
        result = merge_dicts(dict1, dict2, "skip")
        
        assert result == {"a": 1, "b": 2}


class TestMergeDictsError:
    """Тесты стратегии 'error'."""
    
    def test_error_no_conflicts(self):
        """Без конфликтов — исключения нет."""
        dict1 = {"a": 1}
        dict2 = {"b": 2}
        
        result = merge_dicts(dict1, dict2, strategy="error")
        
        assert result == {"a": 1, "b": 2}
    
    def test_error_with_conflicts(self):
        """При конфликте выбрасывается KeyError."""
        dict1 = {"a": 1, "b": 2}
        dict2 = {"b": 20, "c": 30}
        
        with pytest.raises(KeyError, match="Conflict on key 'b'"):
            merge_dicts(dict1, dict2, strategy="error")
    
    def test_error_multiple_conflicts(self):
        """При множественных конфликтах — первый вызвавший."""
        dict1 = {"a": 1, "b": 2, "c": 3}
        dict2 = {"b": 20, "c": 30, "d": 40}
        
        with pytest.raises(KeyError) as exc_info:
            merge_dicts(dict1, dict2, strategy="error")
        
        # Должен быть конфликт на 'b' (первый при итерации dict2)
        assert "b" in str(exc_info.value)
Задание 2.3. Тестирование стратегии "merge_lists" (10 минут)
python
class TestMergeDictsMergeLists:
    """Тесты стратегии 'merge_lists'."""
    
    def test_merge_lists_both_lists(self):
        """Оба значения — списки, объединяются."""
        dict1 = {"a": [1, 2], "b": [3]}
        dict2 = {"a": [3, 4], "b": [4, 5], "c": [6]}
        
        result = merge_dicts(dict1, dict2, strategy="merge_lists")
        
        assert result["a"] == [1, 2, 3, 4]  # Объединены
        assert result["b"] == [3, 4, 5]     # Объединены
        assert result["c"] == [6]           # Добавлен
    
    def test_merge_lists_one_not_list(self):
        """Одно из значений не список — override."""
        dict1 = {"a": [1, 2], "b": "not a list"}
        dict2 = {"a": [3, 4], "b": [4, 5]}
        
        result = merge_dicts(dict1, dict2, strategy="merge_lists")
        
        # 'a' — оба списки → объединены
        assert result["a"] == [1, 2, 3, 4]
        # 'b' — dict1["b"] не список → override
        assert result["b"] == [4, 5]
    
    def test_merge_lists_both_not_lists(self):
        """Оба значения не списки — override."""
        dict1 = {"a": 1, "b": 2}
        dict2 = {"a": 3, "b": 4}
        
        result = merge_dicts(dict1, dict2, strategy="merge_lists")
        
        assert result == {"a": 3, "b": 4}
    
    def test_merge_lists_empty_lists(self):
        """Пустые списки."""
        dict1 = {"a": []}
        dict2 = {"a": [1, 2]}
        
        result = merge_dicts(dict1, dict2, strategy="merge_lists")
        
        assert result["a"] == [1, 2]
Задание 2.4. Негативные сценарии для merge_dicts (10 минут)
python
class TestMergeDictsNegative:
    """Негативные сценарии для merge_dicts."""
    
    def test_invalid_strategy(self):
        """Неизвестная стратегия."""
        dict1 = {"a": 1}
        dict2 = {"b": 2}
        
        with pytest.raises(ValueError, match="Unknown strategy: invalid"):
            merge_dicts(dict1, dict2, strategy="invalid")
    
    def test_first_arg_not_dict(self):
        """Первый аргумент не словарь."""
        with pytest.raises(TypeError, match="Both arguments must be dictionaries"):
            merge_dicts("not a dict", {"a": 1})
    
    def test_second_arg_not_dict(self):
        """Второй аргумент не словарь."""
        with pytest.raises(TypeError, match="Both arguments must be dictionaries"):
            merge_dicts({"a": 1}, "not a dict")
    
    def test_both_args_not_dict(self):
        """Оба аргумента не словари."""
        with pytest.raises(TypeError, match="Both arguments must be dictionaries"):
            merge_dicts(123, 456)
    
    def test_none_instead_of_dict(self):
        """None вместо словаря."""
        with pytest.raises(TypeError, match="Both arguments must be dictionaries"):
            merge_dicts(None, {"a": 1})
    
    @pytest.mark.parametrize("strategy", ["override", "skip", "error", "merge_lists"])
    def test_all_strategies_with_empty_dicts(self, strategy):
        """Все стратегии работают с пустыми словарями."""
        assert merge_dicts({}, {}, strategy) == {}
    
    @pytest.mark.parametrize("strategy", [
        "OVERRIDE", "Skip", "ERROR", "MERGE_LISTS", "", None, 123
    ])
    def test_case_sensitive_strategies(self, strategy):
        """Стратегии чувствительны к регистру."""
        with pytest.raises(ValueError, match="Unknown strategy"):
            merge_dicts({"a": 1}, {"b": 2}, strategy=strategy)
Задание 2.5. Параметризация merge_dicts (5 минут)
python
class TestMergeDictsParametrized:
    """Параметризованные тесты для merge_dicts."""
    
    @pytest.mark.parametrize("dict1,dict2,strategy,expected", [
        # override
        ({"a": 1}, {"b": 2}, "override", {"a": 1, "b": 2}),
        ({"a": 1}, {"a": 2}, "override", {"a": 2}),
        # skip
        ({"a": 1}, {"a": 2}, "skip", {"a": 1}),
        # error (без конфликта)
        ({"a": 1}, {"b": 2}, "error", {"a": 1, "b": 2}),
        # merge_lists
        ({"a": [1]}, {"a": [2]}, "merge_lists", {"a": [1, 2]}),
    ])
    def test_merge_scenarios(self, dict1, dict2, strategy, expected):
        """Различные сценарии объединения."""
        assert merge_dicts(dict1, dict2, strategy) == expected
    
    @pytest.mark.parametrize("dict1,dict2,expected_conflict_key", [
        ({"a": 1, "b": 2}, {"b": 20}, "b"),
        ({"x": 1, "y": 2}, {"x": 10, "y": 20, "z": 30}, "x"),
        ({"common": 1}, {"common": 2}, "common"),
    ])
    def test_error_strategy_conflicts(self, dict1, dict2, expected_conflict_key):
        """Проверка конфликтов при strategy='error'."""
        with pytest.raises(KeyError) as exc_info:
            merge_dicts(dict1, dict2, strategy="error")
        
        assert expected_conflict_key in str(exc_info.value)
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться тестировать дополнительные функции group_by_key и flatten_list, а также сложные пограничные случаи.

Задания для сложного уровня
Задание 3.1. Тестирование group_by_key (15 минут)
python
# test_collection_manager_advanced.py

import pytest
from collection_manager import group_by_key


class TestGroupByKey:
    """Тесты для функции group_by_key."""
    
    # ==========================================================
    # ПОЗИТИВНЫЕ СЦЕНАРИИ
    # ==========================================================
    
    def test_empty_list(self):
        """Пустой список."""
        assert group_by_key([], "category") == {}
    
    def test_single_element(self):
        """Один элемент."""
        elements = [{"id": 1, "category": "A"}]
        
        result = group_by_key(elements, "category")
        
        assert result == {"A": [{"id": 1, "category": "A"}]}
    
    def test_multiple_elements_same_group(self):
        """Несколько элементов в одной группе."""
        elements = [
            {"id": 1, "category": "A"},
            {"id": 2, "category": "A"},
            {"id": 3, "category": "A"},
        ]
        
        result = group_by_key(elements, "category")
        
        assert len(result["A"]) == 3
        assert result["A"][0]["id"] == 1
        assert result["A"][1]["id"] == 2
        assert result["A"][2]["id"] == 3
    
    def test_multiple_groups(self):
        """Несколько групп."""
        elements = [
            {"id": 1, "type": "fruit", "name": "apple"},
            {"id": 2, "type": "fruit", "name": "banana"},
            {"id": 3, "type": "vegetable", "name": "carrot"},
            {"id": 4, "type": "fruit", "name": "orange"},
        ]
        
        result = group_by_key(elements, "type")
        
        assert "fruit" in result
        assert "vegetable" in result
        assert len(result["fruit"]) == 3
        assert len(result["vegetable"]) == 1
    
    def test_order_preserved(self):
        """Порядок элементов внутри групп сохраняется."""
        elements = [
            {"id": 3, "group": "A"},
            {"id": 1, "group": "A"},
            {"id": 2, "group": "A"},
        ]
        
        result = group_by_key(elements, "group")
        
        assert result["A"][0]["id"] == 3
        assert result["A"][1]["id"] == 1
        assert result["A"][2]["id"] == 2
    
    def test_different_key_types(self):
        """Ключи разных типов."""
        elements = [
            {"id": 1, "status": 1},
            {"id": 2, "status": "active"},
            {"id": 3, "status": 1},
            {"id": 4, "status": None},
        ]
        
        result = group_by_key(elements, "status")
        
        assert 1 in result
        assert "active" in result
        assert None in result
        assert len(result[1]) == 2
    
    # ==========================================================
    # НЕГАТИВНЫЕ СЦЕНАРИИ (ИСКЛЮЧЕНИЯ)
    # ==========================================================
    
    def test_not_a_list(self):
        """Входной параметр не список."""
        with pytest.raises(TypeError, match="Elements must be a list"):
            group_by_key("not a list", "key")
    
    def test_element_not_dict(self):
        """Элемент списка не словарь."""
        elements = [{"id": 1}, "not a dict", {"id": 3}]
        
        with pytest.raises(TypeError, match="Each element must be a dictionary"):
            group_by_key(elements, "id")
    
    def test_key_not_found(self):
        """Ключ отсутствует в элементе."""
        elements = [
            {"id": 1, "category": "A"},
            {"id": 2, "category": "B"},
            {"id": 3, "wrong_key": "C"},  # Нет поля 'category'
        ]
        
        with pytest.raises(KeyError) as exc_info:
            group_by_key(elements, "category")
        
        assert "category" in str(exc_info.value)
    
    def test_key_not_found_with_details(self):
        """Проверка сообщения об ошибке."""
        elements = [{"name": "test"}]
        
        with pytest.raises(KeyError, match="Key 'id' not found"):
            group_by_key(elements, "id")
Задание 3.2. Тестирование flatten_list (15 минут)
python
from collection_manager import flatten_list


class TestFlattenList:
    """Тесты для функции flatten_list."""
    
    # ==========================================================
    # ПОЗИТИВНЫЕ СЦЕНАРИИ
    # ==========================================================
    
    def test_flat_list(self):
        """Уже плоский список."""
        assert flatten_list([1, 2, 3]) == [1, 2, 3]
    
    def test_one_level_nesting(self):
        """Один уровень вложенности."""
        assert flatten_list([1, [2, 3], 4]) == [1, 2, 3, 4]
    
    def test_deep_nesting(self):
        """Глубокая вложенность."""
        assert flatten_list([1, [2, [3, [4, 5]]]]) == [1, 2, 3, 4, 5]
    
    def test_empty_lists(self):
        """Пустые вложенные списки."""
        assert flatten_list([1, [], 2, [], [3]]) == [1, 2, 3]
    
    def test_mixed_types(self):
        """Смешанные типы элементов."""
        assert flatten_list([1, "a", [2, "b", [3, None]]]) == [1, "a", 2, "b", 3, None]
    
    def test_depth_1(self):
        """Глубина разворачивания = 1."""
        nested = [1, [2, 3], [4, [5, 6]]]
        
        result = flatten_list(nested, depth=1)
        
        # Развернуты только элементы первого уровня вложенности
        assert result == [1, 2, 3, 4, [5, 6]]
    
    def test_depth_2(self):
        """Глубина разворачивания = 2."""
        nested = [1, [2, [3, [4, 5]]]]
        
        result = flatten_list(nested, depth=2)
        
        assert result == [1, 2, 3, [4, 5]]
    
    def test_depth_0(self):
        """Глубина 0 — без изменений."""
        nested = [1, [2, [3]]]
        
        result = flatten_list(nested, depth=0)
        
        assert result == nested
    
    def test_depth_large(self):
        """Глубина больше максимальной."""
        nested = [1, [2, [3, [4]]]]
        
        result = flatten_list(nested, depth=100)
        
        # Полностью развёрнуто
        assert result == [1, 2, 3, 4]
    
    def test_empty_input(self):
        """Пустой список."""
        assert flatten_list([]) == []
    
    def test_single_element(self):
        """Один элемент."""
        assert flatten_list([42]) == [42]
    
    # ==========================================================
    # НЕГАТИВНЫЕ СЦЕНАРИИ
    # ==========================================================
    
    def test_not_a_list(self):
        """Входной параметр не список."""
        with pytest.raises(TypeError, match="Input must be a list"):
            flatten_list("not a list")
    
    def test_none_input(self):
        """None как входной параметр."""
        with pytest.raises(TypeError, match="Input must be a list"):
            flatten_list(None)
Задание 3.3. Пограничные случаи и большие объёмы данных (10 минут)
python
class TestEdgeCasesAndPerformance:
    """Пограничные случаи и производительность."""
    
    def test_very_large_list_find_duplicates(self):
        """Большой список для find_duplicates."""
        large_list = list(range(10000)) + list(range(5000))
        result = find_duplicates(large_list)
        # Дубликаты — числа от 0 до 4999
        assert len(result) == 5000
        assert result == list(range(5000))
    
    def test_very_large_list_unique(self):
        """Большой список для get_unique_elements."""
        large_list = list(range(10000)) * 2
        result = get_unique_elements(large_list)
        assert len(result) == 10000
    
    def test_recursive_structure_flatten(self):
        """Рекурсивная структура (список содержит себя)."""
        recursive_list = [1, 2]
        recursive_list.append(recursive_list)
        
        # Должно быть безопасно (ограничение глубины)
        # В текущей реализации без защиты от рекурсии — осторожно!
        with pytest.raises(RecursionError):
            flatten_list(recursive_list)
    
    def test_mixed_nesting_flatten(self):
        """Смешанное вложение с разными типами."""
        data = [1, [2, 3, [4, 5, [6]]], "string", [7, 8], 9]
        result = flatten_list(data)
        assert result == [1, 2, 3, 4, 5, 6, "string", 7, 8, 9]
    
    def test_group_by_key_many_elements(self):
        """Группировка большого количества элементов."""
        elements = [{"letter": chr(65 + i % 26), "index": i} for i in range(1000)]
        
        result = group_by_key(elements, "letter")
        
        assert len(result) == 26
        for letter, group in result.items():
            assert len(group) == 1000 // 26 + (1 if ord(letter) - 65 < 1000 % 26 else 0)
Задание 3.4. Интеграционное тестирование (10 минут)
python
class TestIntegration:
    """Интеграционные тесты (комбинация функций)."""
    
    def test_find_duplicates_then_unique(self):
        """Комбинация find_duplicates и get_unique_elements."""
        data = [1, 2, 2, 3, 3, 3, 4, 4, 5]
        
        duplicates = find_duplicates(data)
        unique = get_unique_elements(data)
        
        # Дубликаты — подмножество уникальных элементов
        assert set(duplicates).issubset(set(unique))
        
        # Все дубликаты встречаются более одного раза
        for item in duplicates:
            assert data.count(item) > 1
    
    def test_group_by_key_then_flatten(self):
        """Комбинация group_by_key и flatten_list."""
        elements = [
            {"id": 1, "tags": ["a", "b"]},
            {"id": 2, "tags": ["b", "c"]},
            {"id": 3, "tags": ["a", "c"]},
        ]
        
        # Группируем по первому тегу
        grouped = group_by_key(elements, "tags")
        
        # Разворачиваем структуру
        # grouped = {'a': [elem1, elem3], 'b': [elem1, elem2], 'c': [elem2, elem3]}
        
        for tag, group in grouped.items():
            assert len(group) >= 1
    
    def test_pipeline_all_functions(self):
        """Конвейер из всех функций."""
        # Исходные данные
        data = [
            {"category": "A", "values": [1, 2, 3, 1]},
            {"category": "B", "values": [2, 3, 4, 2]},
            {"category": "A", "values": [3, 4, 5, 4]},
            {"category": "C", "values": [1, 1, 1, 1]},
        ]
        
        # 1. Группируем по категории
        grouped = group_by_key(data, "category")
        
        # 2. Для каждой категории собираем все значения
        all_values = {}
        for category, items in grouped.items():
            values = []
            for item in items:
                values.extend(item["values"])
            all_values[category] = values
        
        # 3. Находим дубликаты в значениях каждой категории
        duplicates_by_category = {}
        for category, values in all_values.items():
            duplicates_by_category[category] = find_duplicates(values)
        
        # Проверки
        assert "A" in duplicates_by_category
        assert duplicates_by_category["A"] == [1, 3, 4]  # 1,3,4 встречаются дважды
        assert duplicates_by_category["B"] == [2, 3, 4]
        assert duplicates_by_category["C"] == [1]
Задание 3.5. Итоговый отчёт (5 минут)
markdown
## Итоговый отчёт по практической работе 3.7

**Студент:** _________________
**Уровень:** □ Базовый □ Средний □ Сложный
**Вариант №:** ___

### Результаты тестирования функций

| Функция | Количество тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| `find_duplicates` | _____ | _____ | _____ |
| `get_unique_elements` | _____ | _____ | _____ |
| `merge_dicts` | _____ | _____ | _____ |
| `group_by_key` | _____ | _____ | _____ |
| `flatten_list` | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Проверка исключений

| Функция | Проверяемые исключения | Статус |
|:---|:---|:---|
| `find_duplicates` | `TypeError` | ☐ |
| `merge_dicts` | `TypeError`, `KeyError`, `ValueError` | ☐ |
| `group_by_key` | `TypeError`, `KeyError` | ☐ |
| `flatten_list` | `TypeError` | ☐ |

### Покрытие стратегий merge_dicts

| Стратегия | Протестирована |
|:---|:---|
| `"override"` | ☐ |
| `"skip"` | ☐ |
| `"error"` | ☐ |
| `"merge_lists"` | ☐ |

### Найденные дефекты

| ID | Функция | Описание | Серьёзность |
|:---|:---|:---|:---|
| | | | |

### Выводы

_________________________________________________________________

_________________________________________________________________
Карточка студента
text
ПР 3.7. ТЕСТИРОВАНИЕ ФУНКЦИЙ ДЛЯ РАБОТЫ С КОЛЛЕКЦИЯМИ

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ find_duplicates (базовый)
□ get_unique_elements (базовый)
□ merge_dicts (средний)
□ group_by_key (сложный)
□ flatten_list (сложный)
□ Интеграционные тесты (сложный)

=== ПРОВЕРЕННЫЕ СЦЕНАРИИ ===

□ Пустые списки/словари
□ Элементы с дубликатами
□ Элементы без дубликатов
□ Смешанные типы данных
□ None как элемент/значение
□ Исключения (TypeError, KeyError, ValueError)

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
- test_collection_manager_base.py
- test_collection_manager_intermediate.py
- test_collection_manager_advanced.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Тесты не работают или отсутствуют
3 (удовлетворительно)	Базовый	find_duplicates + get_unique_elements (10+ тестов)
4 (хорошо)	Средний	+ merge_dicts со всеми стратегиями (20+ тестов)
5 (отлично)	Сложный	+ group_by_key + flatten_list + интеграция (30+ тестов)
Контрольные вопросы (для защиты)
Чем отличается find_duplicates от get_unique_elements?

Какую структуру данных лучше использовать для поиска дубликатов? Почему?

Какие стратегии объединения словарей вы знаете? В каких ситуациях какая стратегия полезна?

Как функция merge_dicts с strategy="merge_lists" обрабатывает значения, не являющиеся списками?

Почему при тестировании group_by_key важно проверять порядок элементов?

Какой параметр flatten_list контролирует глубину разворачивания?

Как протестировать обработку очень больших списков?

Почему важно тестировать не только позитивные, но и негативные сценарии?

Следующее занятие: Лекция 3.9 — Мокирование в pytest. Подмена внешних зависимостей.

text
