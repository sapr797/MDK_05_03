# Практическое занятие 3.6: Тестирование декораторов. Мокирование времени, проверка кэширования, тестирование логирования и повторных попыток

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.6:** Тестирование декораторов  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться тестировать функции-декораторы с использованием мокирования времени (`time.sleep`, `datetime`), проверять работу кэширования, логирования и механизма повторных попыток (retry).

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Тестировать декораторы с использованием `monkeypatch` для подмены времени (ПК 3.2, ПК 3.3).
2. Проверять корректность работы кэширования результатов (ПК 3.2).
3. Тестировать логирование с использованием `capsys` (ПК 3.2).
4. Тестировать механизм повторных попыток с мокированием ошибок (ПК 3.3).

---

## Теоретическая справка

### Что такое декораторы?

**Декоратор** — это функция, которая принимает другую функцию и расширяет её поведение без изменения исходного кода.

```python
@timer
def my_function():
    pass

# Эквивалентно:
my_function = timer(my_function)
Тестируемые декораторы
Декоратор	Назначение	Сложность тестирования
@timer	Измеряет время выполнения	Требует мокирования time.time()
@cache_result	Кэширует результат вызова	Требует проверки повторных вызовов
@logger	Логирует вызов и результат	Требует перехвата вывода (capsys)
@retry	Повторяет вызов при ошибке	Требует мокирования исключений
Объект тестирования
Листинг 1. Модуль decorators.py
python
# decorators.py

import time
import functools
import logging
from typing import Any, Callable, Dict, Optional, Type, Union
from datetime import datetime


# ============================================================
# ДЕКОРАТОР 1: ТАЙМЕР
# ============================================================

def timer(func: Callable) -> Callable:
    """
    Декоратор, измеряющий время выполнения функции.
    Выводит время в консоль (stdout).
    
    Пример вывода:
    "Function 'my_func' executed in 0.1234 seconds"
    """
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        elapsed = end - start
        print(f"Function '{func.__name__}' executed in {elapsed:.4f} seconds")
        return result
    return wrapper


# ============================================================
# ДЕКОРАТОР 2: КЭШИРОВАНИЕ
# ============================================================

def cache_result(func: Callable) -> Callable:
    """
    Декоратор, кэширующий результаты вызова функции.
    При повторном вызове с теми же аргументами возвращает кэшированное значение.
    
    Особенности:
    - Кэш хранится в словаре {(args, frozenset(kwargs.items())): result}
    - При разных аргументах кэш не используется
    """
    cache: Dict[tuple, Any] = {}
    
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        # Создаём ключ кэша (args + kwargs)
        key = (args, frozenset(kwargs.items()))
        
        if key in cache:
            print(f"Cache hit for {func.__name__}{args}{kwargs}")
            return cache[key]
        
        print(f"Cache miss for {func.__name__}{args}{kwargs}")
        result = func(*args, **kwargs)
        cache[key] = result
        return result
    
    # Добавляем метод для очистки кэша (для тестирования)
    wrapper.clear_cache = lambda: cache.clear()
    wrapper.get_cache_size = lambda: len(cache)
    
    return wrapper


# ============================================================
# ДЕКОРАТОР 3: ЛОГГЕР
# ============================================================

def logger(func: Callable) -> Callable:
    """
    Декоратор, логирующий вызов функции и результат.
    Выводит в stderr:
    - START: вызов функции с аргументами
    - END: результат выполнения
    - ERROR: если возникло исключение
    """
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        import sys
        timestamp = datetime.now().strftime("%H:%M:%S")
        
        # Логируем начало выполнения
        print(f"[{timestamp}] START {func.__name__} args={args} kwargs={kwargs}", file=sys.stderr)
        
        try:
            result = func(*args, **kwargs)
            print(f"[{timestamp}] END {func.__name__} result={result}", file=sys.stderr)
            return result
        except Exception as e:
            print(f"[{timestamp}] ERROR {func.__name__} exception={type(e).__name__}: {e}", file=sys.stderr)
            raise
    
    return wrapper


# ============================================================
# ДЕКОРАТОР 4: ПОВТОРНЫЕ ПОПЫТКИ (RETRY)
# ============================================================

def retry(
    max_attempts: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    exceptions: Union[Type[Exception], tuple] = Exception
) -> Callable:
    """
    Декоратор для повторных попыток при возникновении исключений.
    
    Args:
        max_attempts: максимальное количество попыток
        delay: начальная задержка между попытками (секунды)
        backoff: множитель увеличения задержки (экспоненциальная задержка)
        exceptions: исключения, при которых выполняется повтор
    
    Example:
        @retry(max_attempts=5, delay=0.5, backoff=2)
        def unstable_function():
            ...
    """
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            current_delay = delay
            last_exception = None
            
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e
                    if attempt == max_attempts:
                        print(f"Retry: Last attempt {attempt}/{max_attempts} failed", file=sys.stderr)
                        raise
                    
                    print(f"Retry: Attempt {attempt}/{max_attempts} failed: {e}. Retrying in {current_delay}s...", file=sys.stderr)
                    time.sleep(current_delay)
                    current_delay *= backoff
            
            raise last_exception
        return wrapper
    return decorator


# ============================================================
# ТЕСТОВЫЕ ФУНКЦИИ
# ============================================================

@timer
def slow_function(seconds: float = 0.1) -> str:
    """Функция, которая выполняется с задержкой."""
    time.sleep(seconds)
    return "done"


@cache_result
def expensive_computation(x: int, y: int, factor: int = 1) -> int:
    """Функция с дорогими вычислениями."""
    print(f"Computing {x} * {y} * {factor}")
    return x * y * factor


@logger
def divide(a: float, b: float) -> float:
    """Функция с возможным исключением."""
    if b == 0:
        raise ValueError("Division by zero")
    return a / b


@retry(max_attempts=3, delay=0.1, backoff=2)
def unstable_network_call(succeed_on_attempt: int = 2) -> str:
    """
    Функция, эмулирующая нестабильный сетевой вызов.
    Успешно выполняется только на определённой попытке.
    """
    if not hasattr(unstable_network_call, "_attempt"):
        unstable_network_call._attempt = 0
    
    unstable_network_call._attempt += 1
    
    if unstable_network_call._attempt < succeed_on_attempt:
        raise ConnectionError(f"Network error on attempt {unstable_network_call._attempt}")
    
    return f"Success on attempt {unstable_network_call._attempt}"
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Тестирование @timer и @cache_result
Средний (варианты 9-17)	Средние студенты	+ Тестирование @logger и @retry
Сложный (варианты 18-25)	Сильные студенты	+ Комбинации декораторов, пограничные случаи
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться тестировать декораторы @timer и @cache_result с использованием мокирования времени.

Задания для базового уровня
Задание 1.1. Тестирование декоратора @timer (15 минут)
Протестируйте, что декоратор @timer корректно измеряет время и выводит сообщение в stdout.

python
# test_decorators_base.py

import pytest
import time
from decorators import timer, slow_function


class TestTimerDecorator:
    """Тесты для декоратора @timer."""
    
    def test_timer_output(self, capsys):
        """Проверка, что таймер выводит сообщение."""
        
        @timer
        def test_func():
            return "result"
        
        result = test_func()
        
        captured = capsys.readouterr()
        assert result == "result"
        assert "Function 'test_func' executed in" in captured.out
        assert "seconds" in captured.out
    
    def test_timer_with_arguments(self, capsys):
        """Таймер с аргументами функции."""
        
        @timer
        def add(a, b):
            return a + b
        
        result = add(3, 5)
        
        captured = capsys.readouterr()
        assert result == 8
        assert "Function 'add' executed in" in captured.out
    
    def test_timer_preserves_function_name(self):
        """Проверка, что декоратор сохраняет имя функции."""
        
        @timer
        def my_special_function():
            pass
        
        assert my_special_function.__name__ == "my_special_function"
    
    def test_timer_with_slow_function(self, capsys, monkeypatch):
        """Тестирование slow_function с мокированием времени."""
        
        # Мокируем time.sleep, чтобы тест не ждал
        sleep_calls = []
        
        def mock_sleep(seconds):
            sleep_calls.append(seconds)
        
        monkeypatch.setattr(time, "sleep", mock_sleep)
        
        # Мокируем time.time для фиксированного времени
        mock_times = [1000.0, 1000.5]  # start, end
        time_call_count = 0
        
        def mock_time():
            nonlocal time_call_count
            result = mock_times[time_call_count]
            time_call_count += 1
            return result
        
        monkeypatch.setattr(time, "time", mock_time)
        
        result = slow_function(0.5)
        
        captured = capsys.readouterr()
        assert result == "done"
        assert "Function 'slow_function' executed in 0.5000 seconds" in captured.out
        assert len(sleep_calls) == 1
        assert sleep_calls[0] == 0.5
    
    def test_timer_zero_time(self, capsys, monkeypatch):
        """Тестирование функции с нулевым временем выполнения."""
        
        @timer
        def fast_function():
            return "fast"
        
        # Мокируем время
        monkeypatch.setattr(time, "time", lambda: 1000.0)
        
        result = fast_function()
        
        captured = capsys.readouterr()
        assert result == "fast"
        # Время может быть 0.0000
        assert "executed in" in captured.out
Задание 1.2. Тестирование декоратора @cache_result (20 минут)
Протестируйте, что декоратор @cache_result корректно кэширует результаты.

python
class TestCacheDecorator:
    """Тесты для декоратора @cache_result."""
    
    def test_cache_miss_first_call(self, capsys):
        """Первый вызов — кэш промах."""
        
        @cache_result
        def compute(x):
            return x * 2
        
        result = compute(5)
        
        captured = capsys.readouterr()
        assert result == 10
        assert "Cache miss for compute(5,)" in captured.out
        assert "Cache hit" not in captured.out
    
    def test_cache_hit_second_call(self, capsys):
        """Второй вызов с теми же аргументами — кэш попадание."""
        
        @cache_result
        def compute(x):
            return x * 2
        
        # Первый вызов
        result1 = compute(5)
        captured1 = capsys.readouterr()
        
        # Второй вызов
        result2 = compute(5)
        captured2 = capsys.readouterr()
        
        assert result1 == 10
        assert result2 == 10
        assert "Cache miss" in captured1.out
        assert "Cache hit" in captured2.out
    
    def test_cache_different_args(self, capsys):
        """Разные аргументы — разные записи в кэше."""
        
        @cache_result
        def compute(x):
            return x * 2
        
        compute(5)
        compute(10)
        compute(5)  # Должен быть кэш-хит
        
        captured = capsys.readouterr()
        # Должно быть 2 miss и 1 hit
        assert captured.out.count("Cache miss") == 2
        assert captured.out.count("Cache hit") == 1
    
    def test_cache_with_kwargs(self, capsys):
        """Кэширование с именованными аргументами."""
        
        @cache_result
        def compute(x, multiplier=2):
            return x * multiplier
        
        compute(5, multiplier=2)
        compute(5, multiplier=2)  # Должен быть hit
        compute(5, multiplier=3)  # Должен быть miss
        
        captured = capsys.readouterr()
        assert captured.out.count("Cache miss") == 2
        assert captured.out.count("Cache hit") == 1
    
    def test_cache_expensive_computation(self, capsys):
        """Тестирование expensive_computation с кэшем."""
        
        @cache_result
        def expensive_computation(x, y, factor=1):
            return x * y * factor
        
        # Первый вызов — вычисление
        result1 = expensive_computation(2, 3, factor=4)
        captured1 = capsys.readouterr()
        
        # Второй вызов — кэш
        result2 = expensive_computation(2, 3, factor=4)
        captured2 = capsys.readouterr()
        
        assert result1 == 24
        assert result2 == 24
        assert "Cache miss" in captured1.out
        assert "Cache hit" in captured2.out
    
    def test_clear_cache_method(self, capsys):
        """Проверка метода clear_cache."""
        
        @cache_result
        def compute(x):
            return x * 2
        
        compute(5)
        compute(5)  # Hit
        
        # Очищаем кэш
        compute.clear_cache()
        
        compute(5)  # Должен быть miss снова
        compute(5)  # Hit
        
        captured = capsys.readouterr()
        # После очистки: miss, hit
        assert captured.out.count("Cache miss") == 2
        assert captured.out.count("Cache hit") == 2
    
    def test_cache_size_method(self):
        """Проверка метода get_cache_size."""
        
        @cache_result
        def compute(x):
            return x * 2
        
        assert compute.get_cache_size() == 0
        
        compute(5)
        assert compute.get_cache_size() == 1
        
        compute(10)
        assert compute.get_cache_size() == 2
        
        compute(5)  # Hit, размер не меняется
        assert compute.get_cache_size() == 2
Задание 1.3. Тестирование пограничных случаев кэша (10 минут)
python
class TestCacheEdgeCases:
    """Пограничные случаи кэширования."""
    
    def test_cache_with_none_argument(self, capsys):
        """Аргумент None."""
        
        @cache_result
        def func(value):
            return value
        
        func(None)
        func(None)  # Должен быть hit
        
        captured = capsys.readouterr()
        assert captured.out.count("Cache hit") == 1
    
    def test_cache_with_empty_args(self, capsys):
        """Функция без аргументов."""
        
        @cache_result
        def constant():
            return 42
        
        constant()
        constant()  # Hit
        
        captured = capsys.readouterr()
        assert captured.out.count("Cache hit") == 1
    
    def test_cache_with_mutable_args_warning(self, capsys):
        """С мутабельными аргументами (список) — разные ключи."""
        
        @cache_result
        def process(data):
            return sum(data)
        
        process([1, 2, 3])
        process([1, 2, 3])  # Разные списки → разные ключи, miss!
        
        captured = capsys.readouterr()
        # Оба вызова — miss (списки не хэшируются, но args кортеж)
        assert captured.out.count("Cache miss") == 2
    
    def test_cache_with_frozenset_for_hashing(self):
        """Использование frozenset для хэшируемых аргументов."""
        
        @cache_result
        def process_set(data):
            return sum(data)
        
        result1 = process_set(frozenset([1, 2, 3]))
        result2 = process_set(frozenset([1, 2, 3]))
        
        assert result1 == 6
        assert result2 == 6
        # Кэш должен работать, так как frozenset хэшируемый
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться тестировать декораторы @logger и @retry с использованием capsys и monkeypatch.

Задания для среднего уровня
Задание 2.1. Тестирование декоратора @logger (15 минут)
Протестируйте, что декоратор @logger корректно логирует начало, конец и ошибки.

python
# test_decorators_intermediate.py

import pytest
from decorators import logger, divide


class TestLoggerDecorator:
    """Тесты для декоратора @logger."""
    
    def test_logger_start_and_end(self, capsys):
        """Проверка логирования начала и конца."""
        
        @logger
        def greet(name):
            return f"Hello, {name}!"
        
        result = greet("Alice")
        
        captured = capsys.readouterr()
        assert result == "Hello, Alice!"
        assert "START greet args=('Alice',)" in captured.err
        assert "END greet result=Hello, Alice!" in captured.err
    
    def test_logger_with_multiple_args(self, capsys):
        """Логирование с несколькими аргументами."""
        
        @logger
        def calculate(a, b, c=0):
            return a + b + c
        
        calculate(1, 2, c=3)
        
        captured = capsys.readouterr()
        assert "START calculate args=(1, 2)" in captured.err
        assert "kwargs={'c': 3}" in captured.err
        assert "END calculate result=6" in captured.err
    
    def test_logger_with_exception(self, capsys):
        """Логирование исключения."""
        
        @logger
        def failing_function():
            raise ValueError("Something went wrong")
        
        with pytest.raises(ValueError, match="Something went wrong"):
            failing_function()
        
        captured = capsys.readouterr()
        assert "START failing_function" in captured.err
        assert "ERROR failing_function exception=ValueError: Something went wrong" in captured.err
    
    def test_logger_divide_success(self, capsys):
        """Тестирование функции divide с успешным выполнением."""
        
        result = divide(10, 2)
        
        captured = capsys.readouterr()
        assert result == 5.0
        assert "START divide args=(10, 2)" in captured.err
        assert "END divide result=5.0" in captured.err
    
    def test_logger_divide_by_zero(self, capsys):
        """Тестирование divide с исключением."""
        
        with pytest.raises(ValueError, match="Division by zero"):
            divide(10, 0)
        
        captured = capsys.readouterr()
        assert "START divide args=(10, 0)" in captured.err
        assert "ERROR divide exception=ValueError: Division by zero" in captured.err
    
    def test_logger_timestamp_format(self, capsys):
        """Проверка формата временной метки."""
        
        @logger
        def simple():
            pass
        
        simple()
        
        captured = capsys.readouterr()
        # Временная метка формата HH:MM:SS
        import re
        assert re.search(r"\[\d{2}:\d{2}:\d{2}\]", captured.err)
Задание 2.2. Тестирование декоратора @retry (20 минут)
Протестируйте механизм повторных попыток при ошибках.

python
class TestRetryDecorator:
    """Тесты для декоратора @retry."""
    
    def test_retry_success_first_attempt(self, capsys, monkeypatch):
        """Успех с первой попытки — повторных попыток нет."""
        
        call_count = 0
        
        @retry(max_attempts=3, delay=0.1)
        def always_success():
            nonlocal call_count
            call_count += 1
            return "success"
        
        result = always_success()
        
        captured = capsys.readouterr()
        assert result == "success"
        assert call_count == 1
        assert "Retry:" not in captured.err
    
    def test_retry_success_on_second_attempt(self, capsys, monkeypatch):
        """Успех на второй попытке."""
        
        call_count = 0
        
        @retry(max_attempts=3, delay=0.1)
        def succeed_on_second():
            nonlocal call_count
            call_count += 1
            if call_count == 1:
                raise ConnectionError("Temporary error")
            return "success"
        
        # Мокируем time.sleep, чтобы тест не ждал
        sleep_calls = []
        def mock_sleep(seconds):
            sleep_calls.append(seconds)
        monkeypatch.setattr(time, "sleep", mock_sleep)
        
        result = succeed_on_second()
        
        captured = capsys.readouterr()
        assert result == "success"
        assert call_count == 2
        assert len(sleep_calls) == 1
        assert sleep_calls[0] == 0.1
        assert "Retry: Attempt 1/3 failed" in captured.err
        assert "Retrying in 0.1s" in captured.err
    
    def test_retry_all_attempts_fail(self, capsys, monkeypatch):
        """Все попытки завершаются ошибкой."""
        
        call_count = 0
        
        @retry(max_attempts=3, delay=0.1)
        def always_fails():
            nonlocal call_count
            call_count += 1
            raise ValueError("Persistent error")
        
        # Мокируем time.sleep
        monkeypatch.setattr(time, "sleep", lambda x: None)
        
        with pytest.raises(ValueError, match="Persistent error"):
            always_fails()
        
        captured = capsys.readouterr()
        assert call_count == 3
        assert captured.err.count("Retry: Attempt") == 2  # 2 ошибки (3-я последняя)
        assert "Last attempt 3/3 failed" in captured.err
    
    def test_retry_exponential_backoff(self, monkeypatch):
        """Проверка экспоненциальной задержки."""
        
        call_count = 0
        sleep_calls = []
        
        def mock_sleep(seconds):
            sleep_calls.append(seconds)
        monkeypatch.setattr(time, "sleep", mock_sleep)
        
        @retry(max_attempts=4, delay=0.5, backoff=2)
        def fails_three_times():
            nonlocal call_count
            call_count += 1
            raise RuntimeError(f"Error {call_count}")
        
        with pytest.raises(RuntimeError):
            fails_three_times()
        
        # Задержки: 0.5, 1.0, 2.0
        assert len(sleep_calls) == 3
        assert sleep_calls[0] == 0.5
        assert sleep_calls[1] == 1.0
        assert sleep_calls[2] == 2.0
    
    def test_retry_only_certain_exceptions(self, capsys, monkeypatch):
        """Повтор только для определённых исключений."""
        
        call_count = 0
        
        @retry(max_attempts=3, delay=0.1, exceptions=ConnectionError)
        def function():
            nonlocal call_count
            call_count += 1
            raise ValueError("Not a connection error")
        
        monkeypatch.setattr(time, "sleep", lambda x: None)
        
        with pytest.raises(ValueError, match="Not a connection error"):
            function()
        
        # Повторных попыток не было, так как исключение не в списке
        assert call_count == 1
        assert "Retry:" not in capsys.readouterr().err
    
    def test_retry_with_unstable_function(self, capsys, monkeypatch):
        """Тестирование unstable_network_call."""
        
        # Сбрасываем счётчик попыток
        unstable_network_call._attempt = 0
        
        monkeypatch.setattr(time, "sleep", lambda x: None)
        
        result = unstable_network_call(succeed_on_attempt=2)
        
        captured = capsys.readouterr()
        assert result == "Success on attempt 2"
        assert "Retry: Attempt 1/3 failed" in captured.err
Задание 2.3. Тестирование unstable_network_call (10 минут)
python
class TestUnstableNetworkCall:
    """Специальные тесты для unstable_network_call."""
    
    def test_unstable_succeeds_on_third(self, monkeypatch):
        """Успех на третьей попытке."""
        
        # Сбрасываем счётчик
        unstable_network_call._attempt = 0
        
        monkeypatch.setattr(time, "sleep", lambda x: None)
        
        result = unstable_network_call(succeed_on_attempt=3)
        
        assert result == "Success on attempt 3"
    
    def test_unstable_fails_all(self, monkeypatch):
        """Все попытки неудачны."""
        
        unstable_network_call._attempt = 0
        monkeypatch.setattr(time, "sleep", lambda x: None)
        
        with pytest.raises(ConnectionError):
            unstable_network_call(succeed_on_attempt=5)  # max_attempts=3
        
        assert unstable_network_call._attempt == 3
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться тестировать комбинации декораторов и сложные пограничные случаи.

Задания для сложного уровня
Задание 3.1. Тестирование нескольких декораторов вместе (15 минут)
python
# test_decorators_advanced.py

import pytest
import time
from decorators import timer, cache_result, logger, retry


class TestCombinedDecorators:
    """Тесты для комбинации декораторов."""
    
    def test_timer_and_cache_together(self, capsys, monkeypatch):
        """Комбинация @timer и @cache_result."""
        
        # Мокируем время
        mock_times = [1000.0, 1000.1]  # start, end (0.1 сек)
        time_call_count = 0
        
        def mock_time():
            nonlocal time_call_count
            result = mock_times[time_call_count]
            time_call_count += 1
            return result
        
        monkeypatch.setattr(time, "time", mock_time)
        monkeypatch.setattr(time, "sleep", lambda x: None)
        
        @timer
        @cache_result
        def compute(x):
            return x * 2
        
        # Первый вызов — cache miss + timer
        result1 = compute(5)
        captured1 = capsys.readouterr()
        
        # Второй вызов — cache hit (timer тоже сработает!)
        result2 = compute(5)
        captured2 = capsys.readouterr()
        
        assert result1 == 10
        assert result2 == 10
        assert "Cache miss" in captured1.out
        assert "Cache hit" in captured2.out
        # Таймер срабатывает при каждом вызове
        assert "executed in" in captured1.out
        assert "executed in" in captured2.out
    
    def test_logger_and_retry_together(self, capsys, monkeypatch):
        """Комбинация @logger и @retry."""
        
        call_count = 0
        
        @logger
        @retry(max_attempts=3, delay=0.1)
        def unstable():
            nonlocal call_count
            call_count += 1
            if call_count < 2:
                raise ConnectionError("Temporary")
            return "success"
        
        monkeypatch.setattr(time, "sleep", lambda x: None)
        
        result = unstable()
        
        captured = capsys.readouterr()
        assert result == "success"
        assert call_count == 2
        
        # Логирование должно быть для каждой попытки
        assert captured.err.count("START unstable") == 2
        assert captured.err.count("END unstable result=success") == 1
        assert captured.err.count("ERROR unstable exception=ConnectionError: Temporary") == 1
        assert captured.err.count("Retry: Attempt 1/3 failed") == 1
Задание 3.2. Тестирование пограничных случаев (15 минут)
python
class TestEdgeCases:
    """Пограничные случаи для всех декораторов."""
    
    def test_timer_with_exception(self, capsys, monkeypatch):
        """Таймер при возникновении исключения."""
        
        @timer
        def failing():
            raise ValueError("Error")
        
        monkeypatch.setattr(time, "time", lambda: 1000.0)
        
        with pytest.raises(ValueError):
            failing()
        
        captured = capsys.readouterr()
        # Время всё равно должно быть выведено
        assert "executed in" in captured.out
    
    def test_cache_with_large_args(self):
        """Кэширование с большим количеством аргументов."""
        
        @cache_result
        def many_args(a, b, c, d, e, f, g, h):
            return a + b + c + d + e + f + g + h
        
        args = (1, 2, 3, 4, 5, 6, 7, 8)
        result1 = many_args(*args)
        result2 = many_args(*args)
        
        assert result1 == 36
        assert result2 == 36
    
    def test_retry_zero_attempts(self):
        """Повторных попыток нет."""
        
        @retry(max_attempts=0)
        def func():
            return "ok"
        
        # Декоратор должен корректно обработать max_attempts=0
        # (хотя логически это странно)
        result = func()
        assert result == "ok"
    
    def test_retry_single_attempt(self):
        """Только одна попытка."""
        
        call_count = 0
        
        @retry(max_attempts=1, delay=0.1)
        def fails():
            nonlocal call_count
            call_count += 1
            raise RuntimeError("Fail")
        
        with pytest.raises(RuntimeError):
            fails()
        
        assert call_count == 1
    
    def test_logger_preserves_return_value(self):
        """Логгер не изменяет возвращаемое значение."""
        
        @logger
        def compute():
            return {"complex": "data", "nested": [1, 2, 3]}
        
        result = compute()
        assert result == {"complex": "data", "nested": [1, 2, 3]}
    
    def test_all_decorators_preserve_docstring(self):
        """Проверка сохранения docstring всеми декораторами."""
        
        @timer
        @cache_result
        @logger
        @retry(max_attempts=2)
        def documented_function():
            """This is a docstring."""
            pass
        
        assert documented_function.__doc__ == "This is a docstring."
Задание 3.3. Комплексное тестирование с параметризацией (10 минут)
python
class TestParametrized:
    """Параметризованные тесты для декораторов."""
    
    @pytest.mark.parametrize("a,b,expected", [
        (10, 2, 5.0),
        (9, 3, 3.0),
        (0, 5, 0.0),
        (-10, 2, -5.0),
    ])
    def test_logger_divide_parametrized(self, capsys, a, b, expected):
        """Параметризованный тест для divide с логгером."""
        
        result = divide(a, b)
        
        captured = capsys.readouterr()
        assert result == expected
        assert f"END divide result={expected}" in captured.err
    
    @pytest.mark.parametrize("cache_hits", [1, 2, 5])
    def test_cache_multiple_hits(self, cache_hits):
        """Проверка множественных попаданий в кэш."""
        
        @cache_result
        def compute(x):
            return x
        
        compute(42)
        
        for _ in range(cache_hits):
            compute(42)
        
        # Размер кэша должен остаться 1
        assert compute.get_cache_size() == 1
Задание 3.4. Итоговый отчёт (5 минут)
markdown
## Итоговый отчёт по практической работе 3.6

**Студент:** _________________
**Уровень:** □ Базовый □ Средний □ Сложный
**Вариант №:** ___

### Результаты тестирования декораторов

| Декоратор | Количество тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| `@timer` | _____ | _____ | _____ |
| `@cache_result` | _____ | _____ | _____ |
| `@logger` | _____ | _____ | _____ |
| `@retry` | _____ | _____ | _____ |
| Комбинации | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Использованные техники мокирования

| Техника | Применение |
|:---|:---|
| `monkeypatch.setattr(time, "sleep", ...)` | |
| `monkeypatch.setattr(time, "time", ...)` | |
| `capsys.readouterr()` | |
| `monkeypatch.setenv()` | |
| `pytest.raises()` | |

### Найденные дефекты

| ID | Декоратор | Описание | Серьёзность |
|:---|:---|:---|:---|
| | | | |

### Выводы

_________________________________________________________________

_________________________________________________________________
Карточка студента
text
ПР 3.6. ТЕСТИРОВАНИЕ ДЕКОРАТОРОВ

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ @timer (базовый)
□ @cache_result (базовый)
□ @logger (средний)
□ @retry (средний)
□ Комбинации декораторов (сложный)
□ Пограничные случаи (сложный)
□ Параметризация (сложный)

=== ТЕХНИКИ МОКИРОВАНИЯ ===

□ monkeypatch.setattr(time, "time")
□ monkeypatch.setattr(time, "sleep")
□ capsys.readouterr()
□ pytest.raises

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
- test_decorators_base.py
- test_decorators_intermediate.py
- test_decorators_advanced.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Тесты не работают или не используют мокирование
3 (удовлетворительно)	Базовый	@timer и @cache_result (8+ тестов)
4 (хорошо)	Средний	+ @logger и @retry (15+ тестов)
5 (отлично)	Сложный	+ комбинации + пограничные случаи (20+ тестов)
Контрольные вопросы (для защиты)
Почему при тестировании @timer необходимо мокировать time.time()?

Как проверить, что @cache_result не кэширует вызовы с разными аргументами?

Чем отличается capsys.readouterr().out от .err?

Как с помощью monkeypatch проверить, что @retry делает паузу между попытками?

Почему при комбинации @timer и @cache_result таймер срабатывает даже при кэш-хите?

Как протестировать, что @logger выводит ошибку в stderr при исключении?

Что будет, если в @retry указать max_attempts=1?

Как проверить, что декоратор сохраняет имя и docstring исходной функции?

Следующее занятие: Лекция 3.8 — Интеграционное тестирование API с использованием requests и pytest.
