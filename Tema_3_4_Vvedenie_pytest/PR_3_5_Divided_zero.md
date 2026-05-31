# Практическое занятие 3.5: Тестирование деления на ноль и факториала

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.5:** Обработка исключений и граничные случаи  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться тестировать функции с критическими случаями (деление на ноль, факториал отрицательных чисел), использовать `pytest.raises` для проверки исключений, проверять сообщения об ошибках и граничные случаи.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Писать тесты для функций с исключениями (ПК 3.2, ПК 3.3).
2. Использовать `pytest.raises` для проверки типов исключений и сообщений об ошибках (ПК 3.2).
3. Тестировать граничные случаи (факториал 0, факториал 1, деление на ноль) (ПК 3.2, ПК 3.3).
4. Оформлять тест-кейсы для проверки обработки ошибок (ОК 02, ОК 05).

---

## Теоретическая справка

### Деление на ноль (ZeroDivisionError)

В Python деление на ноль вызывает исключение `ZeroDivisionError`:

```python
>>> 10 / 0
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ZeroDivisionError: division by zero
Факториал (математическое определение)
text
n! = n × (n-1) × (n-2) × ... × 1
0! = 1
1! = 1
5! = 5 × 4 × 3 × 2 × 1 = 120
Ограничения:

Факториал определён только для неотрицательных целых чисел

Для отрицательных чисел должно быть исключение

Для больших чисел может быть переполнение (но Python поддерживает большие целые)

pytest.raises
python
import pytest

def test_division_by_zero():
    with pytest.raises(ZeroDivisionError) as exc_info:
        result = 10 / 0
    
    # Проверка сообщения об ошибке
    assert str(exc_info.value) == "division by zero"
Объект тестирования (Python)
Листинг 1. Исходный код для тестирования (calculator.py)
python
# calculator.py

import math
from typing import Union, Optional


class Calculator:
    """
    Калькулятор с базовыми операциями.
    
    Функции:
    - divide(a, b): деление a на b с проверкой деления на ноль
    - factorial(n): вычисление факториала числа n
    - power(a, b): возведение в степень (дополнительно)
    - sqrt(x): квадратный корень (дополнительно)
    """
    
    @staticmethod
    def divide(a: Union[int, float], b: Union[int, float]) -> float:
        """
        Делит a на b.
        
        Args:
            a: делимое
            b: делитель
        
        Returns:
            Результат деления
        
        Raises:
            ZeroDivisionError: если b == 0
            TypeError: если a или b не числа
        """
        if not isinstance(a, (int, float)) or not isinstance(b, (int, float)):
            raise TypeError("Оба аргумента должны быть числами")
        
        if b == 0:
            raise ZeroDivisionError("Деление на ноль запрещено")
        
        return a / b
    
    @staticmethod
    def factorial(n: int) -> int:
        """
        Вычисляет факториал числа n.
        
        Args:
            n: неотрицательное целое число
        
        Returns:
            n!
        
        Raises:
            ValueError: если n отрицательное или не целое
            TypeError: если n не число
        """
        if not isinstance(n, int):
            raise TypeError(f"n должен быть целым числом, получен {type(n).__name__}")
        
        if n < 0:
            raise ValueError(f"Факториал определён только для неотрицательных чисел, получено {n}")
        
        if n == 0 or n == 1:
            return 1
        
        result = 1
        for i in range(2, n + 1):
            result *= i
        
        return result
    
    @staticmethod
    def power(a: Union[int, float], b: Union[int, float]) -> Union[int, float]:
        """
        Возводит a в степень b.
        
        Args:
            a: основание
            b: показатель степени
        
        Returns:
            a ** b
        
        Raises:
            ValueError: если 0 ** отрицательное число
        """
        if a == 0 and b < 0:
            raise ValueError("Ноль в отрицательной степени не определён")
        
        return a ** b
    
    @staticmethod
    def sqrt(x: Union[int, float]) -> float:
        """
        Вычисляет квадратный корень из x.
        
        Args:
            x: неотрицательное число
        
        Returns:
            √x
        
        Raises:
            ValueError: если x отрицательный
        """
        if x < 0:
            raise ValueError("Квадратный корень из отрицательного числа не определён")
        
        return math.sqrt(x)
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Тестирование divide и factorial с pytest.raises
Средний (варианты 9-17)	Средние студенты	+ граничные случаи + параметризация
Сложный (варианты 18-25)	Сильные студенты	+ тестирование power и sqrt + полное покрытие
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться тестировать функции с исключениями, используя pytest.raises.

Задания для базового уровня
Задание 1.1. Анализ объекта тестирования (5 минут)
Ответьте на вопросы:

Какие исключения может вызвать функция divide?

При каких условиях функция divide вызывает исключение?

Какие исключения может вызвать функция factorial?

Чему равен 0! и 1!?

Формат ответа:

markdown
**Функция divide:**
| Условие | Исключение | Сообщение |
|:---|:---|:---|
| b == 0 | ZeroDivisionError | "Деление на ноль запрещено" |
| a или b не число | TypeError | "Оба аргумента должны быть числами" |

**Функция factorial:**
| Условие | Исключение | Сообщение |
|:---|:---|:---|
| n < 0 | ValueError | "Факториал определён только для неотрицательных чисел, получено {n}" |
| n не целое | TypeError | "n должен быть целым числом, получен {type}" |
Задание 1.2. Написание тестов для функции divide (15 минут)
Напишите тесты для функции divide с использованием pytest.raises.

python
# test_calculator_base.py

import pytest
from calculator import Calculator


class TestDivideBase:
    """Базовые тесты для функции divide."""
    
    # ========== НОРМАЛЬНЫЕ СЛУЧАИ ==========
    
    def test_divide_positive_numbers(self):
        """Деление двух положительных чисел."""
        result = Calculator.divide(10, 2)
        assert result == 5.0
    
    def test_divide_negative_numbers(self):
        """Деление отрицательных чисел."""
        result = Calculator.divide(-10, -2)
        assert result == 5.0
    
    def test_divide_mixed_sign(self):
        """Деление чисел с разными знаками."""
        result = Calculator.divide(10, -2)
        assert result == -5.0
    
    def test_divide_by_one(self):
        """Деление на 1."""
        result = Calculator.divide(42, 1)
        assert result == 42.0
    
    def test_divide_zero_by_number(self):
        """Деление нуля на число."""
        result = Calculator.divide(0, 5)
        assert result == 0.0
    
    # ========== ТЕСТЫ ИСКЛЮЧЕНИЙ ==========
    
    def test_divide_by_zero(self):
        """Деление на ноль → ZeroDivisionError."""
        with pytest.raises(ZeroDivisionError) as exc_info:
            Calculator.divide(10, 0)
        
        # Проверка сообщения об ошибке
        assert str(exc_info.value) == "Деление на ноль запрещено"
    
    def test_divide_by_zero_with_negative(self):
        """Деление отрицательного числа на ноль."""
        with pytest.raises(ZeroDivisionError):
            Calculator.divide(-5, 0)
    
    def test_divide_non_numeric_first_arg(self):
        """Первый аргумент не число → TypeError."""
        with pytest.raises(TypeError) as exc_info:
            Calculator.divide("10", 2)
        
        assert "Оба аргумента должны быть числами" in str(exc_info.value)
    
    def test_divide_non_numeric_second_arg(self):
        """Второй аргумент не число → TypeError."""
        with pytest.raises(TypeError):
            Calculator.divide(10, "2")
Задание: Допишите недостающие тесты (отмеченные комментариями).

Задание 1.3. Написание тестов для функции factorial (15 минут)
Напишите тесты для функции factorial с использованием pytest.raises.

python
class TestFactorialBase:
    """Базовые тесты для функции factorial."""
    
    # ========== НОРМАЛЬНЫЕ СЛУЧАИ ==========
    
    def test_factorial_zero(self):
        """0! = 1."""
        result = Calculator.factorial(0)
        assert result == 1
    
    def test_factorial_one(self):
        """1! = 1."""
        result = Calculator.factorial(1)
        assert result == 1
    
    def test_factorial_small_number(self):
        """5! = 120."""
        result = Calculator.factorial(5)
        assert result == 120
    
    def test_factorial_medium_number(self):
        """10! = 3628800."""
        result = Calculator.factorial(10)
        assert result == 3_628_800
    
    # ========== ТЕСТЫ ИСКЛЮЧЕНИЙ ==========
    
    def test_factorial_negative(self):
        """Отрицательное число → ValueError."""
        with pytest.raises(ValueError) as exc_info:
            Calculator.factorial(-5)
        
        assert "Факториал определён только для неотрицательных чисел" in str(exc_info.value)
        assert "-5" in str(exc_info.value)
    
    def test_factorial_float(self):
        """Не целое число → TypeError."""
        with pytest.raises(TypeError) as exc_info:
            Calculator.factorial(5.5)
        
        assert "n должен быть целым числом" in str(exc_info.value)
    
    def test_factorial_string(self):
        """Строка вместо числа → TypeError."""
        with pytest.raises(TypeError):
            Calculator.factorial("5")
    
    @pytest.mark.parametrize("n,expected", [
        (0, 1),
        (1, 1),
        (2, 2),
        (3, 6),
        (4, 24),
        (5, 120),
    ])
    def test_factorial_parametrized(self, n, expected):
        """Параметризованный тест для факториала."""
        assert Calculator.factorial(n) == expected
Задание:

Допишите тест для factorial(20) (ожидаемый результат можно вычислить)

Добавьте тест для очень большого числа (например, 100) — проверьте, что нет переполнения

Задание 1.4. Выполнение тестов и отчёт (5 минут)
bash
pytest test_calculator_base.py -v
Заполните отчёт:

markdown
## Отчёт (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Результаты выполнения тестов

| Функция | Всего тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| divide | _____ | _____ | _____ |
| factorial | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Найденные дефекты

| ID | Описание | Функция |
|:---|:---|:---|
| | | |

### Вывод

- [ ] Все тесты пройдены
- [ ] Есть ошибки (требуется исправление)
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться тестировать граничные случаи и использовать параметризацию pytest.

Задания для среднего уровня
Задание 2.1. Расширенные тесты для divide (15 минут)
Добавьте тесты для граничных случаев и проверки точности.

python
# test_calculator_intermediate.py

import pytest
from calculator import Calculator


class TestDivideIntermediate:
    """Расширенные тесты для функции divide."""
    
    # ========== ГРАНИЧНЫЕ СЛУЧАИ ==========
    
    def test_divide_very_small_numbers(self):
        """Деление очень маленьких чисел."""
        result = Calculator.divide(1e-10, 1e-5)
        assert result == 1e-5  # 1e-10 / 1e-5 = 1e-5
    
    def test_divide_very_large_numbers(self):
        """Деление очень больших чисел."""
        result = Calculator.divide(1e100, 1e50)
        assert result == 1e50
    
    def test_divide_precision(self):
        """Проверка точности деления (плавающая точка)."""
        result = Calculator.divide(1, 3)
        # 1/3 = 0.3333333333333333
        assert abs(result - 0.3333333333333333) < 1e-10
    
    def test_divide_negative_zero(self):
        """Деление на -0 (в Python -0 == 0)."""
        with pytest.raises(ZeroDivisionError):
            Calculator.divide(10, -0.0)
    
    # ========== ТИПЫ ДАННЫХ ==========
    
    def test_divide_float_result(self):
        """Результат деления целых чисел — float."""
        result = Calculator.divide(5, 2)
        assert isinstance(result, float)
        assert result == 2.5
    
    def test_divide_int_by_float(self):
        """Деление целого на float."""
        result = Calculator.divide(10, 2.5)
        assert result == 4.0
    
    # ========== ПАРАМЕТРИЗОВАННЫЕ ТЕСТЫ ==========
    
    @pytest.mark.parametrize("a,b,expected", [
        (10, 2, 5.0),
        (10, -2, -5.0),
        (-10, 2, -5.0),
        (-10, -2, 5.0),
        (0, 5, 0.0),
        (5, 1, 5.0),
        (5, -1, -5.0),
        (2.5, 0.5, 5.0),
    ])
    def test_divide_parametrized(self, a, b, expected):
        """Параметризованный тест для нормальных случаев."""
        assert Calculator.divide(a, b) == expected
    
    @pytest.mark.parametrize("a,b", [
        (10, 0),
        (-5, 0),
        (0, 0),
        (1e100, 0),
    ])
    def test_divide_by_zero_parametrized(self, a, b):
        """Параметризованный тест деления на ноль."""
        with pytest.raises(ZeroDivisionError, match="Деление на ноль запрещено"):
            Calculator.divide(a, b)
    
    @pytest.mark.parametrize("a,b", [
        ("10", 2),
        (10, "2"),
        (None, 2),
        (10, None),
        ([1, 2], 2),
        (10, [1, 2]),
    ])
    def test_divide_non_numeric_parametrized(self, a, b):
        """Параметризованный тест для нечисловых аргументов."""
        with pytest.raises(TypeError, match="должны быть числами"):
            Calculator.divide(a, b)
Задание 2.2. Расширенные тесты для factorial (15 минут)
Добавьте тесты для граничных случаев и больших чисел.

python
class TestFactorialIntermediate:
    """Расширенные тесты для функции factorial."""
    
    # ========== ГРАНИЧНЫЕ СЛУЧАИ ==========
    
    def test_factorial_two(self):
        """2! = 2."""
        assert Calculator.factorial(2) == 2
    
    def test_factorial_three(self):
        """3! = 6."""
        assert Calculator.factorial(3) == 6
    
    def test_factorial_four(self):
        """4! = 24."""
        assert Calculator.factorial(4) == 24
    
    def test_factorial_large_number(self):
        """20! = 2,432,902,008,176,640,000."""
        assert Calculator.factorial(20) == 2_432_902_008_176_640_000
    
    # ========== ПРОВЕРКА ТИПОВ ==========
    
    def test_factorial_returns_int(self):
        """Факториал возвращает целое число."""
        result = Calculator.factorial(5)
        assert isinstance(result, int)
    
    # ========== ПАРАМЕТРИЗОВАННЫЕ ТЕСТЫ ==========
    
    @pytest.mark.parametrize("n,expected", [
        (0, 1),
        (1, 1),
        (2, 2),
        (3, 6),
        (4, 24),
        (5, 120),
        (6, 720),
        (7, 5040),
        (8, 40320),
        (9, 362880),
        (10, 3628800),
    ])
    def test_factorial_small_numbers(self, n, expected):
        """Параметризованный тест для малых чисел."""
        assert Calculator.factorial(n) == expected
    
    @pytest.mark.parametrize("n,expected", [
        (11, 39916800),
        (12, 479001600),
        (13, 6227020800),
        (14, 87178291200),
        (15, 1307674368000),
    ])
    def test_factorial_medium_numbers(self, n, expected):
        """Параметризованный тест для средних чисел."""
        assert Calculator.factorial(n) == expected
    
    @pytest.mark.parametrize("n", [
        -1, -5, -10, -100
    ])
    def test_factorial_negative_parametrized(self, n):
        """Параметризованный тест для отрицательных чисел."""
        with pytest.raises(ValueError) as exc_info:
            Calculator.factorial(n)
        
        assert "неотрицательных" in str(exc_info.value)
    
    @pytest.mark.parametrize("n", [
        1.5, 2.0, 3.7, -1.5
    ])
    def test_factorial_float_parametrized(self, n):
        """Параметризованный тест для чисел с плавающей точкой."""
        with pytest.raises(TypeError, match="целым числом"):
            Calculator.factorial(n)
    
    @pytest.mark.parametrize("n", [
        "5", None, [5], (5,), {5}
    ])
    def test_factorial_invalid_types(self, n):
        """Параметризованный тест для нечисловых типов."""
        with pytest.raises(TypeError):
            Calculator.factorial(n)
Задание 2.3. Тестирование переполнения и производительности (10 минут)
python
class TestPerformance:
    """Тесты производительности и граничных случаев."""
    
    def test_factorial_does_not_overflow(self):
        """Python поддерживает большие целые, переполнения быть не должно."""
        # 100! — очень большое число, но Python справляется
        result = Calculator.factorial(100)
        assert result > 0
        # Проверяем, что результат целый и содержит много цифр
        assert len(str(result)) > 150  # 100! имеет 158 цифр
    
    def test_factorial_1000(self):
        """1000! не вызывает переполнения (Python big integers)."""
        result = Calculator.factorial(1000)
        assert result > 0
        # 1000! имеет 2568 цифр
        assert len(str(result)) > 2500
    
    @pytest.mark.slow
    def test_factorial_10000_performance(self):
        """10000! вычисляется за разумное время."""
        import time
        start = time.time()
        result = Calculator.factorial(10000)
        elapsed = time.time() - start
        
        assert result > 0
        assert elapsed < 1.0, f"Вычисление 10000! заняло {elapsed:.2f} сек"
Задание: Запустите тест test_factorial_10000_performance (помечен @pytest.mark.slow). Сколько времени он занял на вашей машине?

Задание 2.4. Сравнительный анализ (5 минут)
markdown
## Сравнение базового и среднего уровня

| Аспект | Базовый уровень | Средний уровень |
|:---|:---|:---|
| Количество тестов для divide | 9 | 20+ |
| Количество тестов для factorial | 10 | 25+ |
| Использование параметризации | Нет | Да |
| Тестирование граничных случаев | Минимум | Полное |
| Проверка сообщений об ошибках | Частично | Полная |

**Преимущества параметризации:**
- Сокращение дублирования кода
- Лёгкость добавления новых тестов
- Наглядность входных данных и ожидаемых результатов
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться тестировать все функции калькулятора (power, sqrt) с полным покрытием и готовить отчёт.

Задания для сложного уровня
Задание 3.1. Тестирование функции power (15 минут)
Напишите полные тесты для функции power (возведение в степень).

python
# test_calculator_advanced.py

import pytest
from calculator import Calculator


class TestPower:
    """Тесты для функции power."""
    
    # ========== НОРМАЛЬНЫЕ СЛУЧАИ ==========
    
    def test_power_positive_base_positive_exponent(self):
        """Положительное основание в положительной степени."""
        assert Calculator.power(2, 3) == 8
        assert Calculator.power(5, 2) == 25
        assert Calculator.power(10, 0) == 1
    
    def test_power_negative_base_even_exponent(self):
        """Отрицательное основание, чётная степень → положительный результат."""
        assert Calculator.power(-2, 2) == 4
        assert Calculator.power(-3, 4) == 81
    
    def test_power_negative_base_odd_exponent(self):
        """Отрицательное основание, нечётная степень → отрицательный результат."""
        assert Calculator.power(-2, 3) == -8
        assert Calculator.power(-3, 3) == -27
    
    def test_power_zero_positive_exponent(self):
        """Ноль в положительной степени = 0."""
        assert Calculator.power(0, 5) == 0
        assert Calculator.power(0, 1) == 0
    
    def test_power_any_number_zero_exponent(self):
        """Любое число в степени 0 = 1."""
        assert Calculator.power(123, 0) == 1
        assert Calculator.power(-456, 0) == 1
        assert Calculator.power(0, 0) == 1  # Математическая условность
    
    def test_power_fractional_exponent(self):
        """Дробный показатель степени (квадратный корень)."""
        result = Calculator.power(9, 0.5)
        assert result == 3.0
    
    def test_power_negative_exponent(self):
        """Отрицательная степень (1 / a^|b|)."""
        assert Calculator.power(2, -2) == 0.25
        assert Calculator.power(10, -1) == 0.1
    
    # ========== ИСКЛЮЧЕНИЯ ==========
    
    def test_power_zero_negative_exponent(self):
        """0 в отрицательной степени → ValueError."""
        with pytest.raises(ValueError, match="Ноль в отрицательной степени не определён"):
            Calculator.power(0, -1)
        
        with pytest.raises(ValueError):
            Calculator.power(0, -5)
    
    def test_power_negative_base_fractional_exponent(self):
        """Отрицательное основание в дробной степени → комплексное число (ошибка)."""
        # В стандартной математике — комплексное число, в Python — ошибка
        with pytest.raises(ValueError):
            Calculator.power(-4, 0.5)
    
    # ========== ГРАНИЧНЫЕ СЛУЧАИ ==========
    
    @pytest.mark.parametrize("a,b,expected", [
        (2, 10, 1024),
        (2, -1, 0.5),
        (10, -2, 0.01),
        (0.5, 2, 0.25),
        (0.5, -1, 2.0),
        (1, 1000, 1),
        (-1, 1000, 1),   # -1 в чётной степени = 1
        (-1, 999, -1),   # -1 в нечётной степени = -1
    ])
    def test_power_parametrized(self, a, b, expected):
        """Параметризованный тест для power."""
        assert Calculator.power(a, b) == expected
Задание 3.2. Тестирование функции sqrt (15 минут)
Напишите полные тесты для функции sqrt (квадратный корень).

python
class TestSqrt:
    """Тесты для функции sqrt."""
    
    # ========== НОРМАЛЬНЫЕ СЛУЧАИ ==========
    
    def test_sqrt_perfect_square(self):
        """Квадратный корень из полного квадрата."""
        assert Calculator.sqrt(4) == 2.0
        assert Calculator.sqrt(9) == 3.0
        assert Calculator.sqrt(16) == 4.0
        assert Calculator.sqrt(100) == 10.0
    
    def test_sqrt_zero(self):
        """Квадратный корень из 0 = 0."""
        assert Calculator.sqrt(0) == 0.0
    
    def test_sqrt_one(self):
        """Квадратный корень из 1 = 1."""
        assert Calculator.sqrt(1) == 1.0
    
    def test_sqrt_non_perfect(self):
        """Квадратный корень из неполного квадрата."""
        result = Calculator.sqrt(2)
        assert abs(result - 1.4142135623730951) < 1e-10
    
    def test_sqrt_float(self):
        """Квадратный корень из float."""
        result = Calculator.sqrt(2.25)
        assert result == 1.5
    
    def test_sqrt_large_number(self):
        """Квадратный корень из большого числа."""
        result = Calculator.sqrt(1e100)
        assert result == 1e50
    
    def test_sqrt_small_number(self):
        """Квадратный корень из очень маленького числа."""
        result = Calculator.sqrt(1e-10)
        assert result == 1e-5
    
    # ========== ИСКЛЮЧЕНИЯ ==========
    
    def test_sqrt_negative(self):
        """Квадратный корень из отрицательного числа → ValueError."""
        with pytest.raises(ValueError, match="Квадратный корень из отрицательного числа не определён"):
            Calculator.sqrt(-1)
        
        with pytest.raises(ValueError):
            Calculator.sqrt(-100)
    
    def test_sqrt_negative_float(self):
        """Квадратный корень из отрицательного float."""
        with pytest.raises(ValueError):
            Calculator.sqrt(-2.5)
    
    # ========== ТОЧНОСТЬ ==========
    
    def test_sqrt_precision(self):
        """Проверка точности вычислений."""
        result = Calculator.sqrt(2)
        # 1.4142135623730951 — ожидаемое значение
        assert result == pytest.approx(1.4142135623730951)
    
    @pytest.mark.parametrize("x,expected", [
        (0, 0),
        (1, 1),
        (4, 2),
        (9, 3),
        (16, 4),
        (25, 5),
        (36, 6),
        (49, 7),
        (64, 8),
        (81, 9),
        (100, 10),
        (0.25, 0.5),
        (2.25, 1.5),
    ])
    def test_sqrt_parametrized(self, x, expected):
        """Параметризованный тест для sqrt."""
        assert Calculator.sqrt(x) == expected
    
    @pytest.mark.parametrize("x", [-1, -2, -4, -0.01])
    def test_sqrt_negative_parametrized(self, x):
        """Параметризованный тест для отрицательных чисел."""
        with pytest.raises(ValueError, match="отрицательного числа"):
            Calculator.sqrt(x)
Задание 3.3. Полный набор тестов с покрытием (15 минут)
Создайте файл test_calculator_complete.py, объединяющий все тесты.

python
# test_calculator_complete.py

import pytest
from calculator import Calculator


# ============================================================
# ТЕСТЫ ДЛЯ DIVIDE
# ============================================================

class TestDivideComplete:
    """Полные тесты для divide."""
    
    # Нормальные случаи
    def test_divide_normal(self):
        assert Calculator.divide(10, 2) == 5.0
    
    def test_divide_negative(self):
        assert Calculator.divide(-10, -2) == 5.0
    
    def test_divide_by_one(self):
        assert Calculator.divide(42, 1) == 42.0
    
    def test_divide_zero(self):
        assert Calculator.divide(0, 5) == 0.0
    
    # Исключения
    def test_divide_by_zero(self):
        with pytest.raises(ZeroDivisionError, match="Деление на ноль запрещено"):
            Calculator.divide(10, 0)
    
    def test_divide_non_numeric(self):
        with pytest.raises(TypeError, match="должны быть числами"):
            Calculator.divide("10", 2)
    
    # Параметризация
    @pytest.mark.parametrize("a,b,expected", [
        (10, 2, 5.0),
        (10, -2, -5.0),
        (-10, 2, -5.0),
        (-10, -2, 5.0),
        (0, 5, 0.0),
        (5, 1, 5.0),
        (2.5, 0.5, 5.0),
    ])
    def test_divide_param(self, a, b, expected):
        assert Calculator.divide(a, b) == expected


# ============================================================
# ТЕСТЫ ДЛЯ FACTORIAL
# ============================================================

class TestFactorialComplete:
    """Полные тесты для factorial."""
    
    def test_factorial_zero(self):
        assert Calculator.factorial(0) == 1
    
    def test_factorial_one(self):
        assert Calculator.factorial(1) == 1
    
    def test_factorial_small(self):
        assert Calculator.factorial(5) == 120
    
    def test_factorial_large(self):
        assert Calculator.factorial(20) == 2_432_902_008_176_640_000
    
    def test_factorial_negative(self):
        with pytest.raises(ValueError, match="неотрицательных"):
            Calculator.factorial(-5)
    
    def test_factorial_float(self):
        with pytest.raises(TypeError, match="целым числом"):
            Calculator.factorial(5.5)
    
    @pytest.mark.parametrize("n,expected", [
        (0, 1), (1, 1), (2, 2), (3, 6), (4, 24), (5, 120),
        (6, 720), (7, 5040), (8, 40320), (9, 362880), (10, 3628800),
    ])
    def test_factorial_param(self, n, expected):
        assert Calculator.factorial(n) == expected


# ============================================================
# ТЕСТЫ ДЛЯ POWER
# ============================================================

class TestPowerComplete:
    """Полные тесты для power."""
    
    def test_power_positive(self):
        assert Calculator.power(2, 3) == 8
    
    def test_power_zero_exponent(self):
        assert Calculator.power(123, 0) == 1
    
    def test_power_negative_exponent(self):
        assert Calculator.power(2, -2) == 0.25
    
    def test_power_zero_positive_exponent(self):
        assert Calculator.power(0, 5) == 0
    
    def test_power_zero_negative_exponent(self):
        with pytest.raises(ValueError, match="Ноль в отрицательной степени"):
            Calculator.power(0, -1)
    
    @pytest.mark.parametrize("a,b,expected", [
        (2, 3, 8),
        (2, -1, 0.5),
        (10, -2, 0.01),
        (0.5, 2, 0.25),
        (0.5, -1, 2.0),
        (-2, 2, 4),
        (-2, 3, -8),
    ])
    def test_power_param(self, a, b, expected):
        assert Calculator.power(a, b) == expected


# ============================================================
# ТЕСТЫ ДЛЯ SQRT
# ============================================================

class TestSqrtComplete:
    """Полные тесты для sqrt."""
    
    def test_sqrt_perfect(self):
        assert Calculator.sqrt(4) == 2.0
    
    def test_sqrt_zero(self):
        assert Calculator.sqrt(0) == 0.0
    
    def test_sqrt_non_perfect(self):
        assert Calculator.sqrt(2) == pytest.approx(1.4142135623730951)
    
    def test_sqrt_negative(self):
        with pytest.raises(ValueError, match="отрицательного числа"):
            Calculator.sqrt(-1)
    
    @pytest.mark.parametrize("x,expected", [
        (0, 0), (1, 1), (4, 2), (9, 3), (16, 4), (25, 5),
        (0.25, 0.5), (2.25, 1.5), (100, 10),
    ])
    def test_sqrt_param(self, x, expected):
        assert Calculator.sqrt(x) == expected


# ============================================================
# ТЕСТЫ НА ПОКРЫТИЕ И ПРОИЗВОДИТЕЛЬНОСТЬ
# ============================================================

class TestCoverage:
    """Тесты для проверки покрытия кода."""
    
    def test_all_functions_work(self):
        """Проверка, что все функции вызываются."""
        assert Calculator.divide(10, 2) == 5.0
        assert Calculator.factorial(5) == 120
        assert Calculator.power(2, 3) == 8
        assert Calculator.sqrt(9) == 3.0
    
    def test_factorial_large_performance(self):
        """Проверка производительности factorial."""
        import time
        start = time.time()
        Calculator.factorial(1000)
        elapsed = time.time() - start
        assert elapsed < 0.5, f"Медленно: {elapsed:.2f} сек"
Задание 3.4. Запуск тестов и генерация отчёта о покрытии (10 минут)
bash
# Запуск всех тестов с отчётом о покрытии
pytest test_calculator_complete.py -v --cov=calculator --cov-report=html --cov-report=term

# Запуск только медленных тестов (если отмечены)
pytest test_calculator_complete.py -m slow -v

# Запуск с детализацией ошибок
pytest test_calculator_complete.py -v --tb=short
Заполните итоговый отчёт:

markdown
## Итоговый отчёт по тестированию Calculator

**Студент:** _________________
**Вариант №:** ___ (18-25)
**Дата:** _________________

### Статистика тестов

| Функция | Количество тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| divide | _____ | _____ | _____ |
| factorial | _____ | _____ | _____ |
| power | _____ | _____ | _____ |
| sqrt | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Покрытие кода

| Модуль | Покрытие строк | Покрытие ветвей |
|:---|:---|:---|
| calculator.py | _____% | _____% |

### Найденные дефекты

| ID | Функция | Описание | Серьёзность | Статус |
|:---|:---|:---|:---|:---|
| BUG-01 | power | _____________ | High | Открыт |
| BUG-02 | sqrt | _____________ | Medium | Исправлен |

### Проверка исключений (pytest.raises)

| Функция | Исключение | Проверено |
|:---|:---|:---|
| divide | ZeroDivisionError | ☐ |
| divide | TypeError | ☐ |
| factorial | ValueError | ☐ |
| factorial | TypeError | ☐ |
| power | ValueError | ☐ |
| sqrt | ValueError | ☐ |

### Граничные случаи

| Функция | Граница | Значение | Результат |
|:---|:---|:---|:---|
| divide | Деление на ноль | b = 0 | ZeroDivisionError |
| factorial | n = 0 | 0! = 1 | OK |
| factorial | n = 1 | 1! = 1 | OK |
| factorial | n = -1 | Ошибка | ValueError |
| power | 0^0 | 1 | OK |
| power | 0^(-1) | Ошибка | ValueError |
| sqrt | x = 0 | 0 | OK |
| sqrt | x = -1 | Ошибка | ValueError |

### Выводы

- [ ] Все функции работают корректно
- [ ] Требуется доработка (_____________)
- [ ] Покрытие кода достаточное (≥80%)
- [ ] Граничные случаи обработаны правильно

### Рекомендации

1. 
2. 
Задание 3.5. Мутационное тестирование (дополнительно, со звёздочкой)
Установите mutmut для мутационного тестирования:

bash
pip install mutmut
mutmut run --paths-to-mutate calculator.py --tests-dir .
mutmut results
Задание: Внесите намеренные ошибки в код и проверьте, какие тесты их обнаружат.

Пример мутации (дефект в factorial):

python
# Мутация: изменить условие if n == 0 or n == 1 на if n == 0
def factorial_mutated(n: int) -> int:
    if not isinstance(n, int):
        raise TypeError(...)
    if n < 0:
        raise ValueError(...)
    if n == 0:  # ДЕФЕКТ: пропущено условие n == 1
        return 1
    
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result
Задание: Для каждой мутации запишите, какой тест её обнаружит.

Карточка студента
text
ПР 3.5. ТЕСТИРОВАНИЕ ДЕЛЕНИЯ НА НОЛЬ И ФАКТОРИАЛА

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ Тесты для divide (базовые)
□ Тесты для factorial (базовые)
□ Граничные случаи (средний)
□ Параметризация (средний)
□ Тесты для power (сложный)
□ Тесты для sqrt (сложный)
□ Полное покрытие (сложный)

=== ИСПОЛЬЗОВАННЫЕ ИНСТРУМЕНТЫ ===

□ pytest
□ pytest.raises
□ pytest.mark.parametrize
□ pytest.approx
□ pytest --cov

=== РЕЗУЛЬТАТЫ ТЕСТИРОВАНИЯ ===

Всего тестов: _____
Пройдено: _____
Не пройдено: _____

Покрытие кода: _____%

=== НАЙДЕННЫЕ ДЕФЕКТЫ ===

| ID | Функция | Описание |
|:---|:---|:---|
| | | |

=== ВРЕМЯ ВЫПОЛНЕНИЯ ===

Подготовка: _____ мин
Выполнение: _____ мин
Оформление: _____ мин

=== ОТЧЁТ ===

Файлы:
- test_calculator_base.py
- test_calculator_intermediate.py
- test_calculator_advanced.py
- test_calculator_complete.py
- coverage_report/

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Тесты не работают или отсутствуют
3 (удовлетворительно)	Базовый	divide + factorial с pytest.raises, 10+ тестов
4 (хорошо)	Средний	+ Граничные случаи + параметризация + тесты производительности
5 (отлично)	Сложный	+ power + sqrt + полное покрытие (≥80%) + отчёт
Контрольные вопросы (для защиты)
Зачем нужен pytest.raises? Чем он отличается от try/except?

Чему равен 0! и почему это важно для тестирования?

Как протестировать деление на ноль с отрицательным делителем?

Почему Python не переполняется при вычислении 100!?

Как проверить сообщение об ошибке в pytest.raises?

Что такое pytest.approx и когда его использовать?

Как запустить тесты с отчётом о покрытии?

Какие граничные случаи нужно проверить для функции sqrt?

Почему 0 ** 0 возвращает 1 в Python, а не ошибку?

Что такое мутационное тестирование и зачем оно нужно?

Следующее занятие: Лекция 3.6 — Интеграционное и системное тестирование. Тестирование API и баз данных.
