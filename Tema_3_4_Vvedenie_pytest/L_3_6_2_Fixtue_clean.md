# Лекция 3.7: Фикстуры с очисткой. Встроенные фикстуры pytest

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.7:** Фикстуры с очисткой. Встроенные фикстуры pytest  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить механизмы создания фикстур с автоматической очисткой ресурсов (teardown) с использованием `yield`, а также освоить встроенные фикстуры pytest: `tmp_path`, `capsys`, `monkeypatch` для решения типовых задач тестирования.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Создавать фикстуры с `yield` для реализации setup/teardown (ПК 3.2).
2. Использовать встроенную фикстуру `tmp_path` для работы с временными файлами (ПК 3.3).
3. Использовать встроенную фикстуру `capsys` для перехвата вывода в консоль (ПК 3.2).
4. Использовать встроенную фикстуру `monkeypatch` для подмены функций и переменных окружения (ПК 3.2, ПК 3.3).

---

## 1. Фикстуры с очисткой (yield / teardown)

### 1.1 Проблема: ресурсы, которые нужно освобождать

При тестировании часто требуется:
- Создать временный файл → удалить после теста
- Подключиться к базе данных → закрыть соединение
- Запустить сервер → остановить после теста
- Изменить конфигурацию → вернуть обратно

Фикстуры с `yield` позволяют выполнить **очистку (teardown)** автоматически.

### 1.2 Синтаксис фикстур с yield

```python
import pytest

@pytest.fixture
def resource_with_cleanup():
    """
    SETUP: выполняется ДО теста
    """
    print("\n[SETUP] Создание ресурса")
    resource = {"data": "test_value", "connection": "open"}
    
    yield resource  # Передаём ресурс в тест
    
    """
    TEARDOWN: выполняется ПОСЛЕ теста
    """
    print("\n[TEARDOWN] Закрытие ресурса")
    resource["connection"] = "closed"
    # Здесь: закрытие файла, откат транзакции, остановка сервера
1.3 Схема работы фикстуры с yield
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ЖИЗНЕННЫЙ ЦИКЛ ФИКСТУРЫ С YIELD                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   @pytest.fixture                                                            │
│   def my_fixture():                                                          │
│       print("SETUP: создание ресурса")                                       │
│       resource = create()                                                    │
│       ┌──────────────────────────────────────────────────────────────┐       │
│       │  yield resource   ←─────────── передача ресурса в тест        │       │
│       └──────────────────────────────────────────────────────────────┘       │
│       print("TEARDOWN: очистка ресурса")                                     │
│       resource.cleanup()                                                     │
│                                                                              │
│   def test_example(my_fixture):                                             │
│       # Тест использует resource                                            │
│       assert my_fixture["status"] == "ready"                                 │
│       # После выхода из теста → выполняется код после yield                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
1.4 Пример: фикстура с очисткой для класса UserManager
python
import pytest
from user_manager import UserManager

@pytest.fixture
def user_manager_with_cleanup():
    """Фикстура, которая очищает данные после теста."""
    print("\n[SETUP] Создание UserManager")
    manager = UserManager()
    
    # Добавляем тестовых пользователей
    manager.add_user("alice", "alice@test.com")
    manager.add_user("bob", "bob@test.com")
    
    yield manager  # Передаём менеджер в тест
    
    print("\n[TEARDOWN] Очистка данных")
    # Удаляем всех пользователей
    for user_id in list(manager._users.keys()):
        manager.delete_user(user_id)


def test_user_manager_with_cleanup(user_manager_with_cleanup):
    """Тест использует фикстуру с автоматической очисткой."""
    assert user_manager_with_cleanup.count() == 2
    
    # Добавляем ещё одного пользователя
    user_manager_with_cleanup.add_user("charlie", "charlie@test.com")
    assert user_manager_with_cleanup.count() == 3
    
# После теста — автоматический вызов teardown (очистка)
1.5 Несколько секций yield (не поддерживается)
Важно: В одной фикстуре может быть только один yield. Если нужно несколько этапов очистки, используйте вложенные фикстуры:

python
@pytest.fixture
def db_connection():
    conn = create_connection()
    yield conn
    conn.close()

@pytest.fixture
def transaction(db_connection):
    db_connection.begin()
    yield db_connection
    db_connection.rollback()  # Откат после теста
1.6 Обработка исключений в фикстурах
Если в фикстуре возникает исключение, teardown всё равно выполнится:

python
@pytest.fixture
def risky_resource():
    resource = create_resource()
    try:
        yield resource
    finally:
        # Этот код выполнится даже если в тесте была ошибка
        resource.cleanup()
2. Встроенная фикстура tmp_path
2.1 Назначение
tmp_path предоставляет временную директорию, которая автоматически удаляется после теста. Возвращает объект pathlib.Path.

2.2 Синтаксис и примеры
python
def test_with_tmp_path(tmp_path):
    """
    tmp_path — временная директория, уникальная для каждого теста.
    """
    # Создаём файл во временной директории
    file_path = tmp_path / "data.txt"
    file_path.write_text("Hello, World!")
    
    # Проверяем
    assert file_path.read_text() == "Hello, World!"
    assert file_path.exists()
    
    # После теста директория удаляется
2.3 Создание вложенных директорий
python
def test_nested_directories(tmp_path):
    """Создание сложной структуры."""
    # Создаём вложенную структуру
    data_dir = tmp_path / "data" / "subdir"
    data_dir.mkdir(parents=True)
    
    # Создаём несколько файлов
    (data_dir / "config.json").write_text('{"key": "value"}')
    (data_dir / "log.txt").write_text("log data")
    
    assert (tmp_path / "data" / "subdir" / "config.json").exists()
    # Вся структура удалится после теста
2.4 Сравнение tmp_path и tmpdir
Характеристика	tmp_path	tmpdir
Тип возвращаемого значения	pathlib.Path	py.path.local (строка)
Статус	Современный (рекомендуемый)	Устаревший
Методы	.read_text(), .write_text(), .mkdir()	.read(), .write(), .mkdir()
Поддержка типов	Полная типизация	Строковые операции
2.5 Пример: тестирование записи файла
python
from file_processor import FileProcessor

def test_write_file(tmp_path):
    """Тест записи файла с использованием временной директории."""
    processor = FileProcessor()
    file_path = tmp_path / "output.txt"
    
    result = processor.write_file(file_path, "test content")
    
    assert result is True
    assert file_path.read_text() == "test content"
    # Файл автоматически удалится после теста
3. Встроенная фикстура capsys
3.1 Назначение
capsys позволяет перехватывать вывод в stdout и stderr во время выполнения теста.

3.2 Синтаксис и примеры
python
def test_capture_output(capsys):
    """Перехват вывода в stdout."""
    print("Hello, World!")
    
    captured = capsys.readouterr()
    assert captured.out == "Hello, World!\n"
    assert captured.err == ""

def test_capture_stderr(capsys):
    """Перехват вывода в stderr."""
    import sys
    print("Error message", file=sys.stderr)
    
    captured = capsys.readouterr()
    assert captured.err == "Error message\n"
3.3 Методы capsys
Метод	Описание	Пример
capsys.readouterr()	Возвращает (out, err) и очищает буферы	out, err = capsys.readouterr()
capsys.disabled()	Контекстный менеджер для временного отключения перехвата	with capsys.disabled(): print("visible")
3.4 Пример: проверка логирования ошибок
python
def test_error_logging(capsys):
    """Проверка, что ошибка выводится в stderr."""
    processor = FileProcessor()
    
    processor.log_error("Connection failed")
    
    captured = capsys.readouterr()
    assert "ERROR: Connection failed" in captured.err

def test_no_output(capsys):
    """Проверка, что вывод отсутствует."""
    # Ничего не выводим
    captured = capsys.readouterr()
    assert captured.out == ""
    assert captured.err == ""
3.5 Пример: проверка форматированного вывода
python
def test_formatted_output(capsys):
    """Проверка многострочного вывода."""
    data = ["apple", "banana", "cherry"]
    
    print("Список фруктов:")
    for i, item in enumerate(data, 1):
        print(f"{i}. {item}")
    
    captured = capsys.readouterr()
    assert "Список фруктов:" in captured.out
    assert "1. apple" in captured.out
    assert "2. banana" in captured.out
    assert "3. cherry" in captured.out
4. Встроенная фикстура monkeypatch
4.1 Назначение
monkeypatch позволяет временно подменять:

Атрибуты объектов

Переменные окружения

Функции и методы

Словари и значения

Все изменения автоматически откатываются после теста.

4.2 Основные методы monkeypatch
Метод	Описание	Пример
setattr(obj, name, value)	Подменяет атрибут объекта	monkeypatch.setattr(MyClass, "method", mock_method)
delattr(obj, name)	Удаляет атрибут	monkeypatch.delattr(obj, "attr")
setitem(dict, key, value)	Подменяет значение в словаре	monkeypatch.setitem(os.environ, "KEY", "value")
delitem(dict, key)	Удаляет ключ из словаря	monkeypatch.delitem(os.environ, "KEY")
setenv(name, value)	Устанавливает переменную окружения	monkeypatch.setenv("API_KEY", "test-key")
delenv(name)	Удаляет переменную окружения	monkeypatch.delenv("API_KEY")
syspath_prepend(path)	Добавляет путь в sys.path	monkeypatch.syspath_prepend("/my/libs")
4.3 Пример: подмена переменной окружения
python
import os

def test_api_key_from_env(monkeypatch):
    """Тест чтения API ключа из переменной окружения."""
    # Устанавливаем тестовую переменную окружения
    monkeypatch.setenv("API_KEY", "test-secret-key-123")
    
    # В тесте переменная доступна
    assert os.getenv("API_KEY") == "test-secret-key-123"
    
    # После теста переменная восстанавливается

def test_missing_api_key(monkeypatch):
    """Тест поведения при отсутствии API ключа."""
    # Удаляем переменную, если она есть
    monkeypatch.delenv("API_KEY", raising=False)
    
    assert os.getenv("API_KEY") is None
4.4 Пример: подмена метода класса
python
class ExternalService:
    def fetch_data(self):
        return "real data"

def test_mock_fetch_data(monkeypatch):
    """Подмена метода fetch_data для тестирования."""
    service = ExternalService()
    
    def mock_fetch():
        return "mocked data"
    
    monkeypatch.setattr(service, "fetch_data", mock_fetch)
    
    assert service.fetch_data() == "mocked data"
    # В других тестах метод остаётся оригинальным
4.5 Пример: подмена поведения для тестирования ошибок
python
def test_network_error(monkeypatch):
    """Эмуляция ошибки сети."""
    def mock_request(*args, **kwargs):
        raise ConnectionError("Network unreachable")
    
    import requests
    monkeypatch.setattr(requests, "get", mock_request)
    
    with pytest.raises(ConnectionError):
        requests.get("https://api.example.com")
4.6 Пример: подмена значения словаря
python
def test_config_override(monkeypatch):
    """Подмена значения в конфигурации."""
    config = {"host": "localhost", "port": 8080}
    
    monkeypatch.setitem(config, "port", 9090)
    
    assert config["port"] == 9090
    assert config["host"] == "localhost"  # Не изменилось
4.7 monkeypatch vs unittest.mock
Характеристика	monkeypatch	unittest.mock
Простота использования	Высокая	Средняя
Подмена переменных окружения	Да (setenv)	Нет
Подмена атрибутов объектов	Да (setattr)	Да (patch)
Проверка вызовов	Нет	Да (assert_called_with)
Рекомендация	Для простых подмен	Для сложных моков
5. Комбинирование встроенных фикстур
5.1 Пример: tmp_path + capsys
python
def test_file_processing_with_output(tmp_path, capsys):
    """Комбинация временного файла и перехвата вывода."""
    processor = FileProcessor()
    file_path = tmp_path / "data.txt"
    
    file_path.write_text("line1\nline2\nline3")
    
    result = processor.process_data_file(file_path, tmp_path / "out.txt")
    processor.print_summary(result)
    
    captured = capsys.readouterr()
    assert "Всего элементов: 3" in captured.out
5.2 Пример: monkeypatch + tmp_path
python
def test_config_loading_with_mock(tmp_path, monkeypatch):
    """Комбинация временного файла и подмены переменной окружения."""
    config_file = tmp_path / "config.json"
    config_file.write_text('{"mode": "test"}')
    
    # Подменяем путь к конфигу через переменную окружения
    monkeypatch.setenv("CONFIG_PATH", str(config_file))
    
    # В тесте используется подменённый путь
    assert os.getenv("CONFIG_PATH") == str(config_file)
6. Сравнительная таблица встроенных фикстур
Фикстура	Назначение	Возвращаемый тип	Автоочистка
tmp_path	Временная директория	pathlib.Path	Да (удаление директории)
tmpdir	Временная директория (устаревшая)	py.path.local	Да
capsys	Перехват stdout/stderr	Объект с методами	Да (сброс буферов)
monkeypatch	Подмена атрибутов и переменных	Объект с методами	Да (откат изменений)
request	Информация о тесте	FixtureRequest	—
7. Практические рекомендации
7.1 Когда использовать yield
Сценарий	Решение
Создание временного файла	yield + удаление после теста
Подключение к БД	yield + закрытие соединения
Изменение глобального состояния	monkeypatch (лучше чем ручной teardown)
Очистка данных после теста	yield в фикстуре
7.2 Когда использовать встроенные фикстуры
Задача	Фикстура
Создание временных файлов	tmp_path
Проверка вывода в консоль	capsys
Подмена API ключа	monkeypatch.setenv
Подмена метода	monkeypatch.setattr
Эмуляция ошибки	monkeypatch.setattr + исключение
7.3 Типичные ошибки
python
# ❌ НЕПРАВИЛЬНО: несколько yield
@pytest.fixture
def bad_fixture():
    yield 1
    yield 2  # Ошибка! Только один yield

# ✅ ПРАВИЛЬНО: вложенные фикстуры
@pytest.fixture
def outer():
    data = {}
    yield data
    data.clear()

@pytest.fixture
def inner(outer):
    outer["key"] = "value"
    yield outer
python
# ❌ НЕПРАВИЛЬНО: забыли прочитать вывод capsys
def test_bad(capsys):
    print("test")
    # Нет вызова capsys.readouterr()
    # Буфер не очищен

# ✅ ПРАВИЛЬНО: читаем вывод
def test_good(capsys):
    print("test")
    captured = capsys.readouterr()
    assert captured.out == "test\n"
8. Шпаргалка
python
# === ФИКСТУРА С YIELD (TEARDOWN) ===
@pytest.fixture
def resource():
    print("Setup")
    resource = create()
    yield resource
    print("Teardown")
    resource.cleanup()

# === TMP_PATH ===
def test_files(tmp_path):
    file = tmp_path / "data.txt"
    file.write_text("content")
    assert file.read_text() == "content"
    # Автоудаление

# === CAPSYS ===
def test_output(capsys):
    print("Hello")
    captured = capsys.readouterr()
    assert captured.out == "Hello\n"

# === MONKEYPATCH (переменные окружения) ===
def test_env(monkeypatch):
    monkeypatch.setenv("API_KEY", "test-key")
    assert os.getenv("API_KEY") == "test-key"

# === MONKEYPATCH (подмена метода) ===
def test_method(monkeypatch):
    def mock_method():
        return "mocked"
    monkeypatch.setattr(MyClass, "method", mock_method)
Контрольные вопросы
В чём разница между фикстурой с return и фикстурой с yield?

Как выполнить очистку ресурсов, если в тесте возникло исключение?

В каких ситуациях tmp_path удобнее, чем создание файлов вручную?

Как с помощью capsys проверить, что сообщение об ошибке выводится в stderr?

Чем monkeypatch.setenv() отличается от ручного изменения os.environ?

Как с помощью monkeypatch подменить метод, который вызывается внутри тестируемой функции?

Почему в одной фикстуре нельзя использовать несколько yield?

Какая встроенная фикстура подходит для тестирования функций, которые работают с файловой системой?

Как с помощью capsys проверить, что функция ничего не выводит в консоль?

Восстанавливает ли monkeypatch изменения автоматически после теста?

Следующее занятие: ПР 3.8 — Практическое применение фикстур для тестирования файловых операций и внешних API.
