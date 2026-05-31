# Практическое занятие 3.15: Тестирование структур данных

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.9:** Тестирование структур данных и паттернов  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться тестировать основные структуры данных: списки, словари, множества. Освоить проверку наличия элементов, уникальности, порядка и сравнение коллекций.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Тестировать списки: порядок элементов, доступ по индексу, срезы (ПК 3.2, ПК 3.3).
2. Тестировать словари: наличие ключей, значений, вложенные структуры (ПК 3.2).
3. Тестировать множества: уникальность элементов, операции над множествами (ПК 3.2).
4. Сравнивать коллекции с учётом порядка и без учёта порядка (ОК 02, ОК 05).

---

## Теоретическая справка

### Основные проверки для структур данных

| Структура | Что проверять | Методы проверки |
|:---|:---|:---|
| **Список** | Длина, элемент по индексу, наличие, порядок, срезы | `len()`, `assert list[index]`, `in`, `==`, `[:]` |
| **Словарь** | Длина, наличие ключа, значение, все ключи/значения | `len()`, `key in dict`, `dict[key]`, `.keys()`, `.values()` |
| **Множество** | Длина, наличие, уникальность, операции | `len()`, `in`, `set()`, `\|`, `&`, `-`, `^` |

---

## Объект тестирования

### Листинг 1. Модуль data_structures.py

```python
# data_structures.py

from typing import List, Dict, Any, Set, Optional
from collections import Counter, defaultdict
import json


class DataProcessor:
    """Класс для обработки различных структур данных."""
    
    # ==========================================================
    # ОПЕРАЦИИ СО СПИСКАМИ
    # ==========================================================
    
    @staticmethod
    def get_unique_in_order(items: List[Any]) -> List[Any]:
        """
        Возвращает уникальные элементы списка с сохранением порядка.
        
        Пример:
            [3, 1, 2, 1, 3, 4] → [3, 1, 2, 4]
        """
        seen = set()
        result = []
        for item in items:
            if item not in seen:
                seen.add(item)
                result.append(item)
        return result
    
    @staticmethod
    def rotate_list(items: List[Any], k: int) -> List[Any]:
        """
        Циклический сдвиг списка вправо на k позиций.
        
        Пример:
            [1, 2, 3, 4, 5], k=2 → [4, 5, 1, 2, 3]
        """
        if not items:
            return []
        k = k % len(items)
        return items[-k:] + items[:-k]
    
    @staticmethod
    def merge_sorted_lists(list1: List[int], list2: List[int]) -> List[int]:
        """Объединяет два отсортированных списка в один отсортированный."""
        result = []
        i = j = 0
        while i < len(list1) and j < len(list2):
            if list1[i] < list2[j]:
                result.append(list1[i])
                i += 1
            else:
                result.append(list2[j])
                j += 1
        result.extend(list1[i:])
        result.extend(list2[j:])
        return result
    
    @staticmethod
    def chunk_list(items: List[Any], chunk_size: int) -> List[List[Any]]:
        """Разбивает список на чанки указанного размера."""
        if chunk_size <= 0:
            raise ValueError("chunk_size must be positive")
        return [items[i:i + chunk_size] for i in range(0, len(items), chunk_size)]
    
    # ==========================================================
    # ОПЕРАЦИИ СО СЛОВАРЯМИ
    # ==========================================================
    
    @staticmethod
    def invert_dict(d: Dict[Any, Any]) -> Dict[Any, List[Any]]:
        """
        Инвертирует словарь: значения становятся ключами.
        Если несколько ключей имеют одинаковое значение,
        они собираются в список.
        
        Пример:
            {'a': 1, 'b': 2, 'c': 1} → {1: ['a', 'c'], 2: ['b']}
        """
        result = defaultdict(list)
        for key, value in d.items():
            result[value].append(key)
        return dict(result)
    
    @staticmethod
    def deep_merge(dict1: Dict, dict2: Dict) -> Dict:
        """
        Глубокое слияние двух словарей.
        При конфликте значения из dict2 имеют приоритет.
        """
        result = dict1.copy()
        for key, value in dict2.items():
            if key in result and isinstance(result[key], dict) and isinstance(value, dict):
                result[key] = DataProcessor.deep_merge(result[key], value)
            else:
                result[key] = value
        return result
    
    @staticmethod
    def filter_by_value(d: Dict[Any, Any], condition: callable) -> Dict[Any, Any]:
        """Фильтрует словарь по условию, применяемому к значениям."""
        return {k: v for k, v in d.items() if condition(v)}
    
    # ==========================================================
    # ОПЕРАЦИИ С МНОЖЕСТВАМИ
    # ==========================================================
    
    @staticmethod
    def find_common_elements(sets: List[Set]) -> Set:
        """Находит элементы, присутствующие во всех множествах."""
        if not sets:
            return set()
        result = sets[0].copy()
        for s in sets[1:]:
            result &= s
        return result
    
    @staticmethod
    def symmetric_difference_all(sets: List[Set]) -> Set:
        """Находит элементы, встречающиеся нечётное количество раз."""
        from collections import Counter
        counter = Counter()
        for s in sets:
            counter.update(s)
        return {elem for elem, count in counter.items() if count % 2 == 1}
    
    # ==========================================================
    # СРАВНЕНИЕ КОЛЛЕКЦИЙ
    # ==========================================================
    
    @staticmethod
    def are_equal_ignore_order(list1: List, list2: List) -> bool:
        """Сравнивает два списка без учёта порядка."""
        return Counter(list1) == Counter(list2)
    
    @staticmethod
    def find_differences(list1: List, list2: List) -> Dict[str, List]:
        """
        Находит различия между двумя списками.
        Возвращает словарь с ключами 'only_in_first', 'only_in_second', 'common'.
        """
        set1 = set(list1)
        set2 = set(list2)
        return {
            'only_in_first': list(set1 - set2),
            'only_in_second': list(set2 - set1),
            'common': list(set1 & set2)
        }
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Тестирование списков
Средний (варианты 9-17)	Средние студенты	+ тестирование словарей
Сложный (варианты 18-25)	Сильные студенты	+ множества + сравнение коллекций
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться тестировать операции со списками: уникальность, порядок, срезы, слияние.

Задания для базового уровня
Задание 1.1. Тестирование get_unique_in_order (15 минут)
python
# test_structures_base.py

import pytest
from data_structures import DataProcessor


class TestListOperations:
    """Тесты для операций со списками."""
    
    # ==========================================================
    # ТЕСТЫ get_unique_in_order
    # ==========================================================
    
    def test_unique_in_order_empty_list(self):
        """Пустой список."""
        result = DataProcessor.get_unique_in_order([])
        assert result == []
    
    def test_unique_in_order_all_unique(self):
        """Все элементы уникальны."""
        result = DataProcessor.get_unique_in_order([1, 2, 3, 4])
        assert result == [1, 2, 3, 4]
    
    def test_unique_in_order_with_duplicates(self):
        """С дубликатами."""
        result = DataProcessor.get_unique_in_order([1, 2, 2, 3, 1, 4])
        assert result == [1, 2, 3, 4]
    
    def test_unique_in_order_preserves_order(self):
        """Порядок сохраняется."""
        result = DataProcessor.get_unique_in_order([3, 1, 2, 1, 3, 4])
        assert result == [3, 1, 2, 4]
    
    def test_unique_in_order_strings(self):
        """Строки."""
        result = DataProcessor.get_unique_in_order(["a", "b", "a", "c"])
        assert result == ["a", "b", "c"]
    
    def test_unique_in_order_mixed_types(self):
        """Смешанные типы."""
        result = DataProcessor.get_unique_in_order([1, "1", 1, "1", 2])
        assert result == [1, "1", 2]
    
    def test_unique_in_order_none_values(self):
        """None как элемент."""
        result = DataProcessor.get_unique_in_order([None, 1, None, 2])
        assert result == [None, 1, 2]
    
    # ==========================================================
    # ТЕСТЫ rotate_list
    # ==========================================================
    
    def test_rotate_empty_list(self):
        """Пустой список."""
        result = DataProcessor.rotate_list([], 3)
        assert result == []
    
    def test_rotate_by_zero(self):
        """Сдвиг на 0."""
        result = DataProcessor.rotate_list([1, 2, 3, 4], 0)
        assert result == [1, 2, 3, 4]
    
    def test_rotate_by_one(self):
        """Сдвиг на 1."""
        result = DataProcessor.rotate_list([1, 2, 3, 4], 1)
        assert result == [4, 1, 2, 3]
    
    def test_rotate_by_two(self):
        """Сдвиг на 2."""
        result = DataProcessor.rotate_list([1, 2, 3, 4, 5], 2)
        assert result == [4, 5, 1, 2, 3]
    
    def test_rotate_by_full_length(self):
        """Сдвиг на длину списка (эквивалентно 0)."""
        result = DataProcessor.rotate_list([1, 2, 3], 3)
        assert result == [1, 2, 3]
    
    def test_rotate_by_more_than_length(self):
        """Сдвиг больше длины списка."""
        result = DataProcessor.rotate_list([1, 2, 3, 4], 6)  # 6 % 4 = 2
        assert result == [3, 4, 1, 2]
    
    def test_rotate_by_negative(self):
        """Отрицательный сдвиг (влево)."""
        result = DataProcessor.rotate_list([1, 2, 3, 4], -1)
        assert result == [2, 3, 4, 1]
    
    # ==========================================================
    # ТЕСТЫ merge_sorted_lists
    # ==========================================================
    
    def test_merge_sorted_both_empty(self):
        """Оба списка пустые."""
        result = DataProcessor.merge_sorted_lists([], [])
        assert result == []
    
    def test_merge_sorted_first_empty(self):
        """Первый список пустой."""
        result = DataProcessor.merge_sorted_lists([], [1, 2, 3])
        assert result == [1, 2, 3]
    
    def test_merge_sorted_second_empty(self):
        """Второй список пустой."""
        result = DataProcessor.merge_sorted_lists([1, 2, 3], [])
        assert result == [1, 2, 3]
    
    def test_merge_sorted_simple(self):
        """Простое слияние."""
        result = DataProcessor.merge_sorted_lists([1, 3, 5], [2, 4, 6])
        assert result == [1, 2, 3, 4, 5, 6]
    
    def test_merge_sorted_with_duplicates(self):
        """С дубликатами."""
        result = DataProcessor.merge_sorted_lists([1, 2, 2, 3], [2, 3, 4])
        assert result == [1, 2, 2, 2, 3, 3, 4]
    
    def test_merge_sorted_different_lengths(self):
        """Разной длины."""
        result = DataProcessor.merge_sorted_lists([1, 5, 9], [2, 3, 4, 6, 7, 8])
        assert result == [1, 2, 3, 4, 5, 6, 7, 8, 9]
Задание 1.2. Тестирование chunk_list (10 минут)
python
class TestChunkList:
    """Тесты для chunk_list."""
    
    def test_chunk_size_one(self):
        """Размер чанка = 1."""
        result = DataProcessor.chunk_list([1, 2, 3, 4], 1)
        assert result == [[1], [2], [3], [4]]
    
    def test_chunk_size_two(self):
        """Размер чанка = 2."""
        result = DataProcessor.chunk_list([1, 2, 3, 4, 5], 2)
        assert result == [[1, 2], [3, 4], [5]]
    
    def test_chunk_size_larger_than_list(self):
        """Размер чанка больше длины списка."""
        result = DataProcessor.chunk_list([1, 2], 5)
        assert result == [[1, 2]]
    
    def test_chunk_size_equal_to_list(self):
        """Размер чанка равен длине списка."""
        result = DataProcessor.chunk_list([1, 2, 3], 3)
        assert result == [[1, 2, 3]]
    
    def test_chunk_empty_list(self):
        """Пустой список."""
        result = DataProcessor.chunk_list([], 2)
        assert result == []
    
    def test_chunk_negative_size_raises(self):
        """Отрицательный размер чанка -> ValueError."""
        with pytest.raises(ValueError, match="chunk_size must be positive"):
            DataProcessor.chunk_list([1, 2, 3], -1)
    
    def test_chunk_zero_size_raises(self):
        """Нулевой размер чанка -> ValueError."""
        with pytest.raises(ValueError, match="chunk_size must be positive"):
            DataProcessor.chunk_list([1, 2, 3], 0)
Задание 1.3. Параметризация для списков (10 минут)
python
class TestListParametrized:
    """Параметризованные тесты для списков."""
    
    @pytest.mark.parametrize("input_list,expected", [
        ([], []),
        ([1, 2, 3], [1, 2, 3]),
        ([1, 1, 2, 2, 3], [1, 2, 3]),
        (["a", "b", "a"], ["a", "b"]),
        ([1, "1", 1, "1"], [1, "1"]),
    ])
    def test_unique_in_order_parametrized(self, input_list, expected):
        """Параметризованный тест get_unique_in_order."""
        assert DataProcessor.get_unique_in_order(input_list) == expected
    
    @pytest.mark.parametrize("input_list,k,expected", [
        ([1, 2, 3, 4], 0, [1, 2, 3, 4]),
        ([1, 2, 3, 4], 1, [4, 1, 2, 3]),
        ([1, 2, 3, 4], 2, [3, 4, 1, 2]),
        ([1, 2, 3, 4], 3, [2, 3, 4, 1]),
        ([1, 2, 3, 4], 4, [1, 2, 3, 4]),
        ([1, 2, 3, 4], 5, [4, 1, 2, 3]),
    ])
    def test_rotate_parametrized(self, input_list, k, expected):
        """Параметризованный тест rotate_list."""
        assert DataProcessor.rotate_list(input_list, k) == expected
    
    @pytest.mark.parametrize("list1,list2,expected", [
        ([1, 3, 5], [2, 4, 6], [1, 2, 3, 4, 5, 6]),
        ([1, 2, 3], [4, 5, 6], [1, 2, 3, 4, 5, 6]),
        ([1, 3, 5, 7], [2, 4, 6], [1, 2, 3, 4, 5, 6, 7]),
        ([], [1, 2, 3], [1, 2, 3]),
        ([1, 2, 3], [], [1, 2, 3]),
    ])
    def test_merge_sorted_parametrized(self, list1, list2, expected):
        """Параметризованный тест merge_sorted_lists."""
        assert DataProcessor.merge_sorted_lists(list1, list2) == expected
Задание 1.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Результаты тестирования списков

| Функция | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| get_unique_in_order | _____ | _____ | _____ |
| rotate_list | _____ | _____ | _____ |
| merge_sorted_lists | _____ | _____ | _____ |
| chunk_list | _____ | _____ | _____ |

### Проверенные операции со списками

| Операция | Статус |
|:---|:---|
| Длина списка | ☐ |
| Доступ по индексу | ☐ |
| Проверка наличия элемента | ☐ |
| Срезы | ☐ |
| Порядок элементов | ☐ |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться тестировать операции со словарями: инверсия, глубокое слияние, фильтрация.

Задания для среднего уровня
Задание 2.1. Тестирование invert_dict (15 минут)
python
# test_structures_intermediate.py

import pytest
from data_structures import DataProcessor


class TestDictOperations:
    """Тесты для операций со словарями."""
    
    # ==========================================================
    # ТЕСТЫ invert_dict
    # ==========================================================
    
    def test_invert_empty_dict(self):
        """Пустой словарь."""
        result = DataProcessor.invert_dict({})
        assert result == {}
    
    def test_invert_simple_dict(self):
        """Простой словарь с уникальными значениями."""
        result = DataProcessor.invert_dict({'a': 1, 'b': 2, 'c': 3})
        assert result == {1: ['a'], 2: ['b'], 3: ['c']}
    
    def test_invert_with_duplicate_values(self):
        """Словарь с повторяющимися значениями."""
        result = DataProcessor.invert_dict({'a': 1, 'b': 2, 'c': 1, 'd': 2})
        assert result == {1: ['a', 'c'], 2: ['b', 'd']}
    
    def test_invert_preserves_order_of_keys(self):
        """Порядок ключей при дубликатах."""
        result = DataProcessor.invert_dict({'x': 1, 'y': 1, 'z': 1})
        assert result[1] == ['x', 'y', 'z']
    
    def test_invert_with_strings(self):
        """Строковые ключи и значения."""
        result = DataProcessor.invert_dict({'name': 'Alice', 'city': 'Moscow'})
        assert result == {'Alice': ['name'], 'Moscow': ['city']}
    
    def test_invert_with_mixed_types(self):
        """Смешанные типы."""
        result = DataProcessor.invert_dict({1: 'one', 'two': 2})
        assert result == {'one': [1], 2: ['two']}
    
    # ==========================================================
    # ТЕСТЫ deep_merge
    # ==========================================================
    
    def test_deep_merge_both_empty(self):
        """Оба словаря пустые."""
        result = DataProcessor.deep_merge({}, {})
        assert result == {}
    
    def test_deep_merge_no_conflict(self):
        """Нет конфликтующих ключей."""
        dict1 = {'a': 1, 'b': 2}
        dict2 = {'c': 3, 'd': 4}
        result = DataProcessor.deep_merge(dict1, dict2)
        assert result == {'a': 1, 'b': 2, 'c': 3, 'd': 4}
    
    def test_deep_merge_simple_conflict(self):
        """Простой конфликт — dict2 перезаписывает dict1."""
        dict1 = {'a': 1, 'b': 2}
        dict2 = {'b': 20, 'c': 30}
        result = DataProcessor.deep_merge(dict1, dict2)
        assert result == {'a': 1, 'b': 20, 'c': 30}
    
    def test_deep_merge_nested(self):
        """Вложенные словари сливаются рекурсивно."""
        dict1 = {'a': 1, 'b': {'x': 10, 'y': 20}}
        dict2 = {'b': {'y': 200, 'z': 30}, 'c': 3}
        result = DataProcessor.deep_merge(dict1, dict2)
        assert result == {'a': 1, 'b': {'x': 10, 'y': 200, 'z': 30}, 'c': 3}
    
    def test_deep_merge_deep_nested(self):
        """Глубоко вложенные словари."""
        dict1 = {'a': {'b': {'c': 1}}}
        dict2 = {'a': {'b': {'d': 2}}}
        result = DataProcessor.deep_merge(dict1, dict2)
        assert result == {'a': {'b': {'c': 1, 'd': 2}}}
    
    def test_deep_merge_type_conflict(self):
        """Конфликт типов — значение из dict2 заменяет."""
        dict1 = {'a': {'b': 1}}
        dict2 = {'a': 100}
        result = DataProcessor.deep_merge(dict1, dict2)
        assert result == {'a': 100}
    
    # ==========================================================
    # ТЕСТЫ filter_by_value
    # ==========================================================
    
    def test_filter_by_value_empty(self):
        """Пустой словарь."""
        result = DataProcessor.filter_by_value({}, lambda x: x > 0)
        assert result == {}
    
    def test_filter_by_value_greater_than(self):
        """Фильтрация по условию > 0."""
        data = {'a': 1, 'b': -2, 'c': 3, 'd': -4}
        result = DataProcessor.filter_by_value(data, lambda x: x > 0)
        assert result == {'a': 1, 'c': 3}
    
    def test_filter_by_value_even(self):
        """Фильтрация чётных чисел."""
        data = {'a': 1, 'b': 2, 'c': 3, 'd': 4}
        result = DataProcessor.filter_by_value(data, lambda x: x % 2 == 0)
        assert result == {'b': 2, 'd': 4}
    
    def test_filter_by_value_string_length(self):
        """Фильтрация строк по длине."""
        data = {'a': 'cat', 'b': 'elephant', 'c': 'dog'}
        result = DataProcessor.filter_by_value(data, lambda x: len(x) <= 3)
        assert result == {'a': 'cat', 'c': 'dog'}
Задание 2.2. Параметризация для словарей (10 минут)
python
class TestDictParametrized:
    """Параметризованные тесты для словарей."""
    
    @pytest.mark.parametrize("input_dict,expected", [
        ({}, {}),
        ({'a': 1}, {1: ['a']}),
        ({'a': 1, 'b': 2}, {1: ['a'], 2: ['b']}),
        ({'a': 1, 'b': 1}, {1: ['a', 'b']}),
        ({'x': 10, 'y': 20, 'z': 10}, {10: ['x', 'z'], 20: ['y']}),
    ])
    def test_invert_dict_parametrized(self, input_dict, expected):
        """Параметризованный тест invert_dict."""
        assert DataProcessor.invert_dict(input_dict) == expected
    
    @pytest.mark.parametrize("dict1,dict2,expected", [
        ({'a': 1}, {'b': 2}, {'a': 1, 'b': 2}),
        ({'a': 1}, {'a': 2}, {'a': 2}),
        ({'a': {'b': 1}}, {'a': {'c': 2}}, {'a': {'b': 1, 'c': 2}}),
        ({'a': {'b': 1}}, {'a': 2}, {'a': 2}),
    ])
    def test_deep_merge_parametrized(self, dict1, dict2, expected):
        """Параметризованный тест deep_merge."""
        assert DataProcessor.deep_merge(dict1, dict2) == expected
    
    @pytest.mark.parametrize("data,condition,expected", [
        ({'a': 1, 'b': 2, 'c': 3}, lambda x: x > 2, {'c': 3}),
        ({'a': 1, 'b': 2, 'c': 3}, lambda x: x < 3, {'a': 1, 'b': 2}),
        ({'a': 1, 'b': 2, 'c': 3}, lambda x: x == 2, {'b': 2}),
    ])
    def test_filter_by_value_parametrized(self, data, condition, expected):
        """Параметризованный тест filter_by_value."""
        assert DataProcessor.filter_by_value(data, condition) == expected
Задание 2.3. Тестирование проверки наличия ключей и значений (10 минут)
python
class TestDictExistence:
    """Тесты для проверки наличия элементов в словарях."""
    
    def test_key_exists(self):
        """Проверка наличия ключа."""
        data = {'name': 'Alice', 'age': 30}
        
        assert 'name' in data
        assert 'age' in data
        assert 'email' not in data
    
    def test_value_exists(self):
        """Проверка наличия значения."""
        data = {'name': 'Alice', 'age': 30}
        
        assert 'Alice' in data.values()
        assert 30 in data.values()
        assert 'Bob' not in data.values()
    
    def test_key_value_pair_exists(self):
        """Проверка наличия пары ключ-значение."""
        data = {'name': 'Alice', 'age': 30}
        
        assert data.get('name') == 'Alice'
        assert data.get('age') == 30
        assert data.get('email') is None
    
    def test_nested_key_exists(self):
        """Проверка наличия ключа во вложенном словаре."""
        data = {'user': {'name': 'Alice', 'settings': {'theme': 'dark'}}}
        
        assert 'user' in data
        assert 'name' in data['user']
        assert 'theme' in data['user']['settings']
        assert 'language' not in data['user']['settings']
    
    def test_get_with_default(self):
        """Проверка метода get с значением по умолчанию."""
        data = {'name': 'Alice'}
        
        assert data.get('name', 'Unknown') == 'Alice'
        assert data.get('age', 0) == 0
        assert data.get('city', 'Moscow') == 'Moscow'
Задание 2.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Результаты тестирования словарей

| Функция | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| invert_dict | _____ | _____ | _____ |
| deep_merge | _____ | _____ | _____ |
| filter_by_value | _____ | _____ | _____ |
| Проверка наличия | _____ | _____ | _____ |

### Проверенные операции со словарями

| Операция | Статус |
|:---|:---|
| Длина словаря | ☐ |
| Проверка наличия ключа | ☐ |
| Проверка наличия значения | ☐ |
| Доступ по ключу | ☐ |
| Вложенные словари | ☐ |

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться тестировать операции с множествами и сравнение коллекций.

Задания для сложного уровня
Задание 3.1. Тестирование операций с множествами (15 минут)
python
# test_structures_advanced.py

import pytest
from data_structures import DataProcessor


class TestSetOperations:
    """Тесты для операций с множествами."""
    
    # ==========================================================
    # ТЕСТЫ find_common_elements
    # ==========================================================
    
    def test_common_empty_list(self):
        """Пустой список множеств."""
        result = DataProcessor.find_common_elements([])
        assert result == set()
    
    def test_common_single_set(self):
        """Одно множество."""
        result = DataProcessor.find_common_elements([{1, 2, 3}])
        assert result == {1, 2, 3}
    
    def test_common_two_sets(self):
        """Два множества."""
        result = DataProcessor.find_common_elements([{1, 2, 3}, {2, 3, 4}])
        assert result == {2, 3}
    
    def test_common_multiple_sets(self):
        """Несколько множеств."""
        result = DataProcessor.find_common_elements([
            {1, 2, 3, 4},
            {2, 3, 4, 5},
            {3, 4, 5, 6}
        ])
        assert result == {3, 4}
    
    def test_common_no_intersection(self):
        """Нет общих элементов."""
        result = DataProcessor.find_common_elements([{1, 2}, {3, 4}, {5, 6}])
        assert result == set()
    
    def test_common_with_empty_set(self):
        """Одно из множеств пустое."""
        result = DataProcessor.find_common_elements([{1, 2, 3}, set(), {2, 3, 4}])
        assert result == set()
    
    # ==========================================================
    # ТЕСТЫ symmetric_difference_all
    # ==========================================================
    
    def test_symdiff_empty_list(self):
        """Пустой список множеств."""
        result = DataProcessor.symmetric_difference_all([])
        assert result == set()
    
    def test_symdiff_single_set(self):
        """Одно множество."""
        result = DataProcessor.symmetric_difference_all([{1, 2, 3}])
        assert result == {1, 2, 3}
    
    def test_symdiff_two_sets(self):
        """Два множества."""
        result = DataProcessor.symmetric_difference_all([{1, 2, 3}, {2, 3, 4}])
        # 1 и 4 встречаются по одному разу (нечётно)
        # 2 и 3 встречаются дважды (чётно)
        assert result == {1, 4}
    
    def test_symdiff_three_sets(self):
        """Три множества."""
        result = DataProcessor.symmetric_difference_all([
            {1, 2, 3},
            {2, 3, 4},
            {3, 4, 5}
        ])
        # 1: 1 раз, 2: 2 раза, 3: 3 раза, 4: 2 раза, 5: 1 раз
        # Нечётно: 1, 3, 5
        assert result == {1, 3, 5}
    
    # ==========================================================
    # ТЕСТЫ уникальности множеств
    # ==========================================================
    
    def test_set_uniqueness(self):
        """Проверка уникальности элементов во множестве."""
        duplicate_list = [1, 2, 2, 3, 3, 3, 4]
        unique_set = set(duplicate_list)
        
        assert len(unique_set) == 4
        assert unique_set == {1, 2, 3, 4}
    
    def test_set_operations(self):
        """Проверка операций над множествами."""
        A = {1, 2, 3}
        B = {3, 4, 5}
        
        # Объединение
        assert A | B == {1, 2, 3, 4, 5}
        
        # Пересечение
        assert A & B == {3}
        
        # Разность
        assert A - B == {1, 2}
        assert B - A == {4, 5}
        
        # Симметрическая разность
        assert A ^ B == {1, 2, 4, 5}
    
    def test_set_membership(self):
        """Проверка принадлежности элемента множеству."""
        colors = {'red', 'green', 'blue'}
        
        assert 'red' in colors
        assert 'yellow' not in colors
Задание 3.2. Тестирование сравнения коллекций (15 минут)
python
class TestCollectionComparison:
    """Тесты для сравнения коллекций."""
    
    # ==========================================================
    # ТЕСТЫ are_equal_ignore_order
    # ==========================================================
    
    def test_equal_ignore_order_empty(self):
        """Пустые списки."""
        assert DataProcessor.are_equal_ignore_order([], []) is True
    
    def test_equal_ignore_order_same_order(self):
        """Одинаковые списки с одинаковым порядком."""
        assert DataProcessor.are_equal_ignore_order([1, 2, 3], [1, 2, 3]) is True
    
    def test_equal_ignore_order_different_order(self):
        """Одинаковые элементы в разном порядке."""
        assert DataProcessor.are_equal_ignore_order([1, 2, 3], [3, 2, 1]) is True
    
    def test_equal_ignore_order_with_duplicates(self):
        """Списки с дубликатами."""
        assert DataProcessor.are_equal_ignore_order([1, 1, 2, 2], [2, 2, 1, 1]) is True
    
    def test_equal_ignore_order_different_length(self):
        """Списки разной длины."""
        assert DataProcessor.are_equal_ignore_order([1, 2], [1, 2, 3]) is False
    
    def test_equal_ignore_order_different_elements(self):
        """Разные элементы."""
        assert DataProcessor.are_equal_ignore_order([1, 2, 3], [1, 2, 4]) is False
    
    # ==========================================================
    # ТЕСТЫ find_differences
    # ==========================================================
    
    def test_find_differences_empty(self):
        """Оба списка пустые."""
        result = DataProcessor.find_differences([], [])
        assert result['only_in_first'] == []
        assert result['only_in_second'] == []
        assert result['common'] == []
    
    def test_find_differences_identical(self):
        """Идентичные списки."""
        result = DataProcessor.find_differences([1, 2, 3], [1, 2, 3])
        assert result['only_in_first'] == []
        assert result['only_in_second'] == []
        assert result['common'] == [1, 2, 3]
    
    def test_find_differences_first_extra(self):
        """В первом списке есть лишние элементы."""
        result = DataProcessor.find_differences([1, 2, 3, 4, 5], [1, 2, 3])
        assert set(result['only_in_first']) == {4, 5}
        assert result['only_in_second'] == []
        assert set(result['common']) == {1, 2, 3}
    
    def test_find_differences_second_extra(self):
        """Во втором списке есть лишние элементы."""
        result = DataProcessor.find_differences([1, 2, 3], [1, 2, 3, 4, 5])
        assert result['only_in_first'] == []
        assert set(result['only_in_second']) == {4, 5}
        assert set(result['common']) == {1, 2, 3}
    
    def test_find_differences_both_extra(self):
        """В обоих списках есть уникальные элементы."""
        result = DataProcessor.find_differences([1, 2, 3, 4], [1, 3, 5, 7])
        assert set(result['only_in_first']) == {2, 4}
        assert set(result['only_in_second']) == {5, 7}
        assert set(result['common']) == {1, 3}
    
    def test_find_differences_with_duplicates(self):
        """Списки с дубликатами (учитываются как множества)."""
        result = DataProcessor.find_differences([1, 1, 2, 2, 3], [2, 2, 3, 3, 4])
        assert set(result['only_in_first']) == {1}
        assert set(result['only_in_second']) == {4}
        assert set(result['common']) == {2, 3}
Задание 3.3. Комплексные сценарии (10 минут)
python
class TestComplexCollectionScenarios:
    """Комплексные сценарии с различными коллекциями."""
    
    def test_pipeline_all_collections(self):
        """Конвейерная обработка всех типов коллекций."""
        # Исходные данные
        data = [1, 2, 2, 3, 3, 3, 4, 1, 5]
        
        # 1. Уникальные с сохранением порядка (список)
        unique = DataProcessor.get_unique_in_order(data)
        assert unique == [1, 2, 3, 4, 5]
        
        # 2. Преобразуем в словарь (частота)
        freq = {item: data.count(item) for item in unique}
        assert freq == {1: 2, 2: 2, 3: 3, 4: 1, 5: 1}
        
        # 3. Фильтруем по частоте (только те, что встречаются > 1 раза)
        duplicates = DataProcessor.filter_by_value(freq, lambda x: x > 1)
        assert duplicates == {1: 2, 2: 2, 3: 3}
        
        # 4. Инвертируем словарь
        inverted = DataProcessor.invert_dict(duplicates)
        assert inverted == {2: [1, 2], 3: [3]}
        
        # 5. Находим общие элементы с другими данными
        other = {1, 3, 5}
        common = set(inverted.keys()) & other
        assert common == {3}
    
    def test_collection_conversions(self):
        """Преобразования между типами коллекций."""
        # Список → множество (уникальность)
        list_data = [1, 2, 2, 3, 3, 3]
        set_data = set(list_data)
        assert set_data == {1, 2, 3}
        assert len(set_data) == 3
        
        # Множество → список (порядок может быть любым)
        list_from_set = list(set_data)
        assert set(list_from_set) == set_data
        
        # Словарь → список ключей
        dict_data = {'a': 1, 'b': 2, 'c': 3}
        keys = list(dict_data.keys())
        assert set(keys) == {'a', 'b', 'c'}
        
        # Словарь → список значений
        values = list(dict_data.values())
        assert set(values) == {1, 2, 3}
    
    def test_nested_collection_comparison(self):
        """Сравнение вложенных коллекций."""
        actual = {
            'users': ['Alice', 'Bob', 'Charlie'],
            'counts': {'active': 2, 'inactive': 1},
            'tags': {'python', 'testing', 'pytest'}
        }
        
        expected = {
            'users': ['Alice', 'Charlie', 'Bob'],  # другой порядок
            'counts': {'inactive': 1, 'active': 2},  # другой порядок ключей
            'tags': {'pytest', 'testing', 'python'}  # множество не имеет порядка
        }
        
        # Сравнение с учётом порядка списков
        assert actual['users'] != expected['users']
        
        # Сравнение без учёта порядка
        assert DataProcessor.are_equal_ignore_order(actual['users'], expected['users'])
        
        # Словари сравниваются независимо от порядка
        assert actual['counts'] == expected['counts']
        
        # Множества сравниваются независимо от порядка
        assert actual['tags'] == expected['tags']
Задание 3.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (сложный уровень)

**Студент:** _________________
**Вариант №:** ___ (18-25)

### Результаты тестирования

| Тип коллекции | Функций | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|:---|
| Списки | 4 | _____ | _____ | _____ |
| Словари | 3 | _____ | _____ | _____ |
| Множества | 2 | _____ | _____ | _____ |
| Сравнение | 2 | _____ | _____ | _____ |
| **Итого** | **11** | _____ | _____ | _____ |

### Проверенные операции

| Операция | Списки | Словари | Множества |
|:---|:---|:---|:---|
| Длина | ☐ | ☐ | ☐ |
| Наличие элемента | ☐ | ☐ | ☐ |
| Уникальность | ☐ | N/A | ☐ |
| Порядок | ☐ | ☐ | N/A |
| Объединение | N/A | N/A | ☐ |
| Пересечение | N/A | N/A | ☐ |
| Разность | N/A | N/A | ☐ |

### Покрытие сценариев

| Сценарий | Покрыт |
|:---|:---|
| Пустые коллекции | ☐ |
| Коллекции с одним элементом | ☐ |
| Коллекции с дубликатами | ☐ |
| Вложенные структуры | ☐ |
| Сравнение с учётом порядка | ☐ |
| Сравнение без учёта порядка | ☐ |

### Выводы

_______________________________________________________________

_______________________________________________________________
Карточка студента
text
ПР 3.15. ТЕСТИРОВАНИЕ СТРУКТУР ДАННЫХ

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ get_unique_in_order (базовый)
□ rotate_list (базовый)
□ merge_sorted_lists (базовый)
□ chunk_list (базовый)
□ invert_dict (средний)
□ deep_merge (средний)
□ filter_by_value (средний)
□ find_common_elements (сложный)
□ symmetric_difference_all (сложный)
□ Сравнение коллекций (сложный)

=== ПРОВЕРЕННЫЕ ОПЕРАЦИИ ===

□ Списки (длина, индекс, срезы, наличие, порядок)
□ Словари (ключи, значения, вложенность, фильтрация)
□ Множества (уникальность, объединение, пересечение, разность)
□ Сравнение (с порядком, без порядка)

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
- test_structures_base.py
- test_structures_intermediate.py
- test_structures_advanced.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Тесты не работают или структуры не тестируются
3 (удовлетворительно)	Базовый	Тесты для списков (10+ тестов)
4 (хорошо)	Средний	+ Тесты для словарей (20+ тестов)
5 (отлично)	Сложный	+ Тесты для множеств + сравнение (30+ тестов)
Контрольные вопросы (для защиты)
Как проверить, что список отсортирован?

Как проверить, что элемент есть в словаре?

Как проверить уникальность элементов во множестве?

Как сравнить два списка с одинаковыми элементами, но в разном порядке?

Как проверить, что множество является подмножеством другого множества?

Как получить значение из словаря с значением по умолчанию в тесте?

Как проверить, что словарь пуст?

Как сравнить два списка с дубликатами без учёта порядка?

Как проверить, что все элементы списка уникальны?

Как протестировать вложенные структуры данных?

Следующее занятие: Контрольная работа 3.3 по темам 3.9–3.14.
