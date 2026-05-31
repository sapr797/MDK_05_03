# Лекция 3.14: Тестирование структур данных

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.9:** Тестирование структур данных и паттернов  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить методы тестирования основных структур данных: списков, словарей, множеств. Освоить проверку наличия элементов, уникальности, порядка и сравнение коллекций.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Тестировать списки: порядок элементов, доступ по индексу, срезы (ПК 3.2, ПК 3.3).
2. Тестировать словари: наличие ключей, значений, вложенные структуры (ПК 3.2).
3. Тестировать множества: уникальность элементов, операции над множествами (ПК 3.2).
4. Сравнивать коллекции с учётом порядка и без учёта порядка (ОК 02, ОК 05).

---

## 1. Тестирование списков (list)

### 1.1 Основные проверки для списков

| Что проверять | Как проверять | Пример |
|:---|:---|:---|
| Длина списка | `assert len(result) == expected` | `len(users) == 5` |
| Элемент по индексу | `assert result[index] == value` | `result[0] == "Alice"` |
| Наличие элемента | `assert item in result` | `assert "Bob" in names` |
| Отсутствие элемента | `assert item not in result` | `assert "Admin" not in users` |
| Порядок элементов | `assert result == expected_list` | `assert result == [1, 2, 3]` |
| Пустой список | `assert result == []` | `assert get_empty() == []` |
| Срезы | `assert result[start:end] == expected` | `result[1:3] == [2, 3]` |

### 1.2 Примеры тестирования списков

```python
def test_list_length():
    """Проверка длины списка."""
    result = [1, 2, 3, 4, 5]
    assert len(result) == 5

def test_list_element_by_index():
    """Проверка элемента по индексу."""
    result = ["apple", "banana", "cherry"]
    assert result[0] == "apple"
    assert result[-1] == "cherry"

def test_list_contains():
    """Проверка наличия элемента."""
    result = [10, 20, 30, 40]
    assert 20 in result
    assert 50 not in result

def test_list_order():
    """Проверка порядка элементов."""
    result = sorted([3, 1, 4, 2])
    assert result == [1, 2, 3, 4]

def test_list_slice():
    """Проверка срезов."""
    result = [1, 2, 3, 4, 5]
    assert result[1:4] == [2, 3, 4]
    assert result[:3] == [1, 2, 3]
    assert result[3:] == [4, 5]

def test_empty_list():
    """Проверка пустого списка."""
    empty = []
    assert empty == []
    assert len(empty) == 0
    assert not empty
1.3 Проверка преобразований списков
python
def test_list_comprehension():
    """Проверка спискового включения."""
    squares = [x**2 for x in range(5)]
    assert squares == [0, 1, 4, 9, 16]

def test_filter_list():
    """Проверка фильтрации списка."""
    numbers = [1, 2, 3, 4, 5, 6]
    even = [n for n in numbers if n % 2 == 0]
    assert even == [2, 4, 6]

def test_map_list():
    """Проверка преобразования элементов."""
    strings = ["a", "ab", "abc"]
    lengths = [len(s) for s in strings]
    assert lengths == [1, 2, 3]
2. Тестирование словарей (dict)
2.1 Основные проверки для словарей
Что проверять	Как проверять	Пример
Длина словаря	assert len(result) == n	len(user) == 3
Наличие ключа	assert key in result	assert "name" in user
Отсутствие ключа	assert key not in result	assert "age" not in user
Значение по ключу	assert result[key] == value	user["name"] == "Alice"
Полное сравнение	assert result == expected	user == {"name": "Alice"}
Пустой словарь	assert result == {}	assert get_empty() == {}
Получение с значением по умолчанию	assert result.get(key, default) == expected	user.get("age", 0) == 0
2.2 Примеры тестирования словарей
python
def test_dict_length():
    """Проверка количества пар ключ-значение."""
    user = {"name": "Alice", "age": 30, "city": "Moscow"}
    assert len(user) == 3

def test_dict_has_key():
    """Проверка наличия ключа."""
    config = {"debug": True, "port": 8080}
    assert "debug" in config
    assert "host" not in config

def test_dict_value_by_key():
    """Проверка значения по ключу."""
    person = {"name": "Bob", "email": "bob@test.com"}
    assert person["name"] == "Bob"
    assert person["email"] == "bob@test.com"

def test_dict_get_with_default():
    """Проверка метода get с значением по умолчанию."""
    settings = {"theme": "dark"}
    assert settings.get("theme") == "dark"
    assert settings.get("language", "en") == "en"
    assert settings.get("font_size") is None

def test_dict_keys():
    """Проверка списка ключей."""
    data = {"a": 1, "b": 2, "c": 3}
    keys = list(data.keys())
    assert keys == ["a", "b", "c"]

def test_dict_values():
    """Проверка списка значений."""
    data = {"a": 1, "b": 2, "c": 3}
    values = list(data.values())
    assert values == [1, 2, 3]

def test_dict_items():
    """Проверка пар ключ-значение."""
    data = {"x": 10, "y": 20}
    items = list(data.items())
    assert items == [("x", 10), ("y", 20)]

def test_empty_dict():
    """Проверка пустого словаря."""
    empty = {}
    assert empty == {}
    assert len(empty) == 0
    assert not empty
2.3 Тестирование вложенных словарей
python
def test_nested_dict_access():
    """Проверка доступа к вложенным словарям."""
    config = {
        "database": {
            "host": "localhost",
            "port": 5432,
            "credentials": {
                "user": "admin",
                "password": "secret"
            }
        }
    }
    
    assert config["database"]["host"] == "localhost"
    assert config["database"]["credentials"]["user"] == "admin"

def test_nested_dict_update():
    """Проверка обновления вложенного словаря."""
    original = {"a": {"b": 1, "c": 2}}
    original["a"]["b"] = 10
    
    assert original["a"]["b"] == 10
    assert original["a"]["c"] == 2

def test_deep_nested_comparison():
    """Сравнение вложенных словарей."""
    expected = {
        "user": {
            "name": "Alice",
            "address": {
                "city": "Moscow",
                "zip": "101000"
            }
        }
    }
    
    actual = {
        "user": {
            "name": "Alice",
            "address": {
                "city": "Moscow",
                "zip": "101000"
            }
        }
    }
    
    assert actual == expected
3. Тестирование множеств (set)
3.1 Основные проверки для множеств
Что проверять	Как проверять	Пример
Длина множества	assert len(result) == n	len({1,2,3}) == 3
Наличие элемента	assert elem in result	assert 2 in {1,2,3}
Отсутствие элемента	assert elem not in result	assert 5 not in {1,2,3}
Уникальность (автоматически)	assert len(set) == len(list)	len(set([1,1,2])) == 2
Пустое множество	assert result == set()	assert get_empty() == set()
Объединение	assert set1 | set2 == expected	{1,2} | {2,3} == {1,2,3}
Пересечение	assert set1 & set2 == expected	{1,2} & {2,3} == {2}
Разность	assert set1 - set2 == expected	{1,2,3} - {2} == {1,3}
3.2 Примеры тестирования множеств
python
def test_set_length():
    """Проверка длины множества."""
    numbers = {1, 2, 3, 4, 5}
    assert len(numbers) == 5

def test_set_contains():
    """Проверка наличия элемента."""
    colors = {"red", "green", "blue"}
    assert "green" in colors
    assert "yellow" not in colors

def test_set_uniqueness():
    """Проверка уникальности элементов."""
    duplicates = [1, 2, 2, 3, 3, 3, 4]
    unique = set(duplicates)
    assert unique == {1, 2, 3, 4}
    assert len(unique) == 4

def test_set_union():
    """Проверка объединения множеств."""
    A = {1, 2, 3}
    B = {3, 4, 5}
    union = A | B
    assert union == {1, 2, 3, 4, 5}

def test_set_intersection():
    """Проверка пересечения множеств."""
    A = {1, 2, 3, 4}
    B = {3, 4, 5, 6}
    intersection = A & B
    assert intersection == {3, 4}

def test_set_difference():
    """Проверка разности множеств."""
    A = {1, 2, 3, 4}
    B = {2, 4}
    difference = A - B
    assert difference == {1, 3}

def test_set_symmetric_difference():
    """Проверка симметрической разности."""
    A = {1, 2, 3}
    B = {3, 4, 5}
    sym_diff = A ^ B
    assert sym_diff == {1, 2, 4, 5}

def test_set_subset():
    """Проверка на подмножество."""
    A = {1, 2}
    B = {1, 2, 3, 4}
    assert A.issubset(B) is True
    assert B.issuperset(A) is True
    assert {5}.issubset(B) is False

def test_empty_set():
    """Проверка пустого множества."""
    empty = set()
    assert empty == set()
    assert len(empty) == 0
    assert not empty
4. Сравнение коллекций
4.1 Сравнение с учётом порядка
python
def test_list_order_matters():
    """При сравнении списков порядок важен."""
    assert [1, 2, 3] == [1, 2, 3]   # ✅ одинаковый порядок
    assert [1, 2, 3] != [3, 2, 1]   # ❌ разный порядок

def test_tuple_order_matters():
    """При сравнении кортежей порядок важен."""
    assert (1, 2, 3) == (1, 2, 3)   # ✅
    assert (1, 2, 3) != (3, 2, 1)   # ❌

def test_list_with_pytest_approx():
    """Сравнение списков с плавающей точкой."""
    actual = [0.1 + 0.2, 0.3 + 0.4]
    expected = [0.3, 0.7]
    assert actual == pytest.approx(expected)
4.2 Сравнение без учёта порядка
python
def test_compare_lists_ignore_order():
    """Сравнение списков без учёта порядка."""
    actual = [3, 1, 4, 2]
    expected = [1, 2, 3, 4]
    
    # Способ 1: сортировка
    assert sorted(actual) == sorted(expected)
    
    # Способ 2: преобразование в множество
    assert set(actual) == set(expected)
    
    # Способ 3: сравнение Counters (с учётом кратности)
    from collections import Counter
    assert Counter(actual) == Counter(expected)

def test_compare_lists_with_duplicates_ignore_order():
    """Сравнение списков с дубликатами."""
    actual = [1, 2, 2, 3]
    expected = [2, 1, 3, 2]
    
    # set не подходит (потеряет дубликаты)
    assert set(actual) != set(expected)  # оба {1,2,3}
    
    # Counter подходит
    from collections import Counter
    assert Counter(actual) == Counter(expected)

def test_compare_dicts_ignore_order():
    """Словари не чувствительны к порядку ключей."""
    dict1 = {"a": 1, "b": 2, "c": 3}
    dict2 = {"c": 3, "a": 1, "b": 2}
    assert dict1 == dict2  # Порядок не важен
4.3 Сравнение вложенных коллекций
python
def test_nested_collections_comparison():
    """Сравнение вложенных коллекций."""
    actual = {
        "users": ["Alice", "Bob", "Charlie"],
        "counts": {"active": 2, "inactive": 1}
    }
    
    expected = {
        "users": ["Alice", "Bob", "Charlie"],
        "counts": {"active": 2, "inactive": 1}
    }
    
    assert actual == expected

def test_nested_with_pytest_approx():
    """Сравнение вложенных коллекций с плавающей точкой."""
    actual = {
        "values": [0.1 + 0.2, 0.3 + 0.4],
        "nested": {"x": 1.0 / 3.0}
    }
    
    expected = {
        "values": [0.3, 0.7],
        "nested": {"x": pytest.approx(0.33333333)}
    }
    
    assert actual == expected
5. Практические приёмы
5.1 Фикстуры для коллекций
python
@pytest.fixture
def sample_list():
    """Фикстура с образцом списка."""
    return [1, 2, 3, 4, 5]

@pytest.fixture
def sample_dict():
    """Фикстура с образцом словаря."""
    return {"name": "Alice", "age": 30, "city": "Moscow"}

@pytest.fixture
def sample_set():
    """Фикстура с образцом множества."""
    return {1, 2, 3, 4, 5}

def test_with_fixtures(sample_list, sample_dict, sample_set):
    assert len(sample_list) == 5
    assert sample_dict["name"] == "Alice"
    assert 3 in sample_set
5.2 Параметризация для коллекций
python
@pytest.mark.parametrize("input_list,expected_length", [
    ([], 0),
    ([1], 1),
    ([1, 2, 3], 3),
    ([1] * 100, 100),
])
def test_list_length_parametrized(input_list, expected_length):
    assert len(input_list) == expected_length

@pytest.mark.parametrize("test_dict,key,expected", [
    ({"a": 1}, "a", 1),
    ({"a": 1, "b": 2}, "b", 2),
    ({"x": 10, "y": 20}, "z", None),
])
def test_dict_get_parametrized(test_dict, key, expected):
    assert test_dict.get(key) == expected
6. Шпаргалка
python
# === СПИСКИ ===
assert len(list) == n
assert item in list
assert item not in list
assert list == [1, 2, 3]
assert sorted(list) == sorted(expected)  # без учёта порядка

# === СЛОВАРИ ===
assert len(dict) == n
assert key in dict
assert dict[key] == value
assert dict.get(key, default) == expected
assert dict == {"a": 1, "b": 2}

# === МНОЖЕСТВА ===
assert len(set) == n
assert elem in set
assert set1 | set2 == union
assert set1 & set2 == intersection
assert set1 - set2 == difference

# === СРАВНЕНИЕ ===
from collections import Counter
assert Counter(list1) == Counter(list2)  # с дубликатами без порядка
Контрольные вопросы
Как проверить, что элемент есть в списке?

Как проверить, что ключ отсутствует в словаре?

Почему для сравнения множеств не нужна сортировка?

Как сравнить два списка с одинаковыми элементами, но в разном порядке?

Как сравнить два списка с дубликатами без учёта порядка?

Как проверить, что множество является подмножеством другого множества?

Как получить значение из словаря с значением по умолчанию в тесте?

Как сравнить два словаря с вложенными структурами?

Как проверить, что список отсортирован?

Какую структуру данных использовать для проверки уникальности элементов?

Следующее занятие: ПР 3.14 — Практическое тестирование структур данных.
