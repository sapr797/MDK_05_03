# Лекция 3.6: Фикстуры с очисткой. Встроенные фикстуры pytest

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.7:** Фикстуры с очисткой и встроенные фикстуры pytest  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель занятия

Научиться создавать фикстуры с очисткой ресурсов (teardown) с использованием `yield`, а также использовать встроенные фикстуры pytest: `tmp_path`, `capsys`, `monkeypatch`.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Создавать фикстуры с автоматической очисткой после тестов (ПК 3.2).
2. Использовать встроенную фикстуру `tmp_path` для работы с временными файлами (ПК 3.3).
3. Использовать встроенную фикстуру `capsys` для перехвата stdout/stderr (ПК 3.2).
4. Использовать встроенную фикстуру `monkeypatch` для подмены функций и переменных окружения (ПК 3.2, ПК 3.3).

---

## Теоретическая справка

### Фикстуры с очисткой (yield)

```python
@pytest.fixture
def resource_with_cleanup():
    """Setup: создание ресурса."""
    resource = create_resource()
    yield resource  # Передаём ресурс в тест
    # Teardown: очистка после теста
    resource.cleanup()
Встроенные фикстуры pytest
Фикстура	Назначение	Пример использования
tmp_path	Временная директория (pathlib.Path)	file = tmp_path / "data.txt"
tmpdir	Временная директория (str)	устаревшая, использовать tmp_path
capsys	Перехват stdout/stderr	capsys.readouterr()
monkeypatch	Подмена функций, переменных, окружения	monkeypatch.setattr(obj, "attr", value)
request	Информация о текущем тесте	request.node.name
Схема работы tmp_path
text
┌─────────────────────────────────────────────────────────────────┐
│                     tmp_path ФИКСТУРА                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   def test_files(tmp_path):                                     │
│       # tmp_path — временная директория                          │
│       file = tmp_path / "test.txt"                              │
│       file.write_text("data")                                   │
│       assert file.read_text() == "data"                         │
│                                                                  │
│   # После теста директория автоматически удаляется              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
Схема работы monkeypatch
text
┌─────────────────────────────────────────────────────────────────┐
│                    monkeypatch ФИКСТУРА                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   # Подмена переменной окружения                                │
│   monkeypatch.setenv("API_KEY", "test-key")                     │
│                                                                  │
│   # Подмена атрибута объекта                                     │
│   monkeypatch.setattr(MyClass, "method", lambda: "mocked")      │
│                                                                  │
│   # Подмена возвращаемого значения                               │
│   monkeypatch.setattr("requests.get", mock_get)                 │
│                                                                  │
│   # Удаление атрибута                                            │
│   monkeypatch.delattr(obj, "attr")                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
Объект тестирования
Будем тестировать класс FileProcessor для работы с файлами и внешними сервисами.

Листинг 1. Исходный код (file_processor.py)
python
# file_processor.py

import os
import json
import sys
from pathlib import Path
from typing import List, Dict, Any, Optional


class FileProcessor:
    """Класс для работы с файлами и данными."""
    
    def __init__(self, config_path: Optional[Path] = None):
        self.config_path = config_path
        self._config = None
        self._log = []
    
    def read_file(self, file_path: Path) -> str:
        """Читает содержимое файла."""
        if not file_path.exists():
            raise FileNotFoundError(f"Файл не найден: {file_path}")
        
        with open(file_path, 'r', encoding='utf-8') as f:
            return f.read()
    
    def write_file(self, file_path: Path, content: str) -> bool:
        """Записывает содержимое в файл."""
        try:
            file_path.parent.mkdir(parents=True, exist_ok=True)
            with open(file_path, 'w', encoding='utf-8') as f:
                f.write(content)
            return True
        except Exception as e:
            self._log.append(f"Ошибка записи: {e}")
            return False
    
    def read_json(self, file_path: Path) -> Dict[str, Any]:
        """Читает JSON из файла."""
        content = self.read_file(file_path)
        return json.loads(content)
    
    def write_json(self, file_path: Path, data: Dict[str, Any]) -> bool:
        """Записывает JSON в файл."""
        content = json.dumps(data, indent=2, ensure_ascii=False)
        return self.write_file(file_path, content)
    
    def process_data_file(self, input_path: Path, output_path: Path) -> List[str]:
        """
        Обрабатывает файл с данными (каждая строка -> upper case).
        Возвращает список обработанных строк.
        """
        content = self.read_file(input_path)
        lines = content.strip().split('\n')
        processed = [line.upper() for line in lines if line.strip()]
        
        # Записываем результат
        self.write_file(output_path, '\n'.join(processed))
        
        return processed
    
    def load_config(self) -> Dict[str, Any]:
        """Загружает конфигурацию из файла."""
        if self.config_path and self.config_path.exists():
            with open(self.config_path, 'r') as f:
                self._config = json.load(f)
        else:
            self._config = {}
        return self._config
    
    def print_summary(self, data: List[str]) -> None:
        """Выводит сводку в stdout."""
        print(f"Всего элементов: {len(data)}")
        for i, item in enumerate(data):
            print(f"{i+1}. {item}")
    
    def log_error(self, message: str) -> None:
        """Логирует ошибку в stderr."""
        print(f"ERROR: {message}", file=sys.stderr)
        self._log.append(message)
    
    def get_log(self) -> List[str]:
        """Возвращает лог ошибок."""
        return self._log


class ExternalService:
    """Класс для работы с внешним API."""
    
    def __init__(self, api_key: str = None):
        self.api_key = api_key or os.getenv("API_KEY", "default-key")
        self._base_url = "https://api.example.com"
    
    def fetch_data(self, endpoint: str) -> Dict[str, Any]:
        """Запрашивает данные из внешнего API."""
        # В реальном коде здесь был бы requests.get()
        # Для тестов используем заглушку
        return {"status": "ok", "data": f"Data from {endpoint}"}
    
    def send_notification(self, user: str, message: str) -> bool:
        """Отправляет уведомление."""
        # В реальном коде — вызов API
        print(f"Sending to {user}: {message}")
        return True
    
    def get_api_key(self) -> str:
        """Возвращает API ключ."""
        return self.api_key
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Фикстуры с yield (teardown), tmp_path
Средний (варианты 9-17)	Средние студенты	+ capsys, monkeypatch (переменные окружения)
Сложный (варианты 18-25)	Сильные студенты	+ monkeypatch для подмены методов, комбинация фикстур
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться создавать фикстуры с очисткой (yield) и использовать встроенную фикстуру tmp_path.

Задания для базового уровня
Задание 1.1. Фикстуры с teardown (yield) (15 минут)
Создайте фикстуры, которые автоматически очищают ресурсы после тестов.

python
# test_file_processor_base.py

import pytest
from pathlib import Path
from file_processor import FileProcessor


# ============================================================
# ФИКСТУРЫ С ОЧИСТКОЙ (YIELD)
# ============================================================

@pytest.fixture
def file_processor():
    """
    Фикстура, возвращающая FileProcessor.
    После теста очищает лог.
    """
    processor = FileProcessor()
    yield processor
    # Очистка после теста
    processor._log.clear()
    print("\n[TEARDOWN] Лог очищен")


@pytest.fixture
def temp_file(tmp_path):
    """
    Фикстура, создающая временный файл и удаляющая его после теста.
    Использует встроенную фикстуру tmp_path.
    """
    file_path = tmp_path / "temp_data.txt"
    yield file_path
    # Очистка: файл удаляется автоматически с tmp_path


@pytest.fixture
def file_with_content(tmp_path):
    """
    Фикстура, создающая файл с предустановленным содержимым.
    """
    file_path = tmp_path / "input.txt"
    content = "line1\nline2\nline3\n"
    file_path.write_text(content)
    
    yield file_path, content
    # Очистка автоматическая


# ============================================================
# ТЕСТЫ
# ============================================================

class TestTeardownFixtures:
    """Тесты с фикстурами, выполняющими очистку."""
    
    def test_file_processor_cleanup(self, file_processor):
        """Проверка, что фикстура работает и очищает лог."""
        file_processor.log_error("Test error")
        assert len(file_processor.get_log()) == 1
        # После теста лог очистится
    
    def test_temp_file_creation(self, temp_file):
        """Проверка работы с временным файлом."""
        temp_file.write_text("Hello, World!")
        assert temp_file.read_text() == "Hello, World!"
        assert temp_file.exists()
        # После теста файл удалится
    
    def test_file_with_content_fixture(self, file_with_content):
        """Проверка фикстуры с предустановленным содержимым."""
        file_path, expected_content = file_with_content
        actual_content = file_path.read_text()
        assert actual_content == expected_content
    
    def test_multiple_temp_files(self, tmp_path):
        """Создание нескольких временных файлов в одном тесте."""
        file1 = tmp_path / "a.txt"
        file2 = tmp_path / "b.txt"
        
        file1.write_text("A")
        file2.write_text("B")
        
        assert file1.read_text() == "A"
        assert file2.read_text() "B"
        # Оба файла будут удалены после теста
Задание 1.2. Работа с tmp_path (15 минут)
Напишите тесты для FileProcessor, используя временную директорию.

python
class TestFileProcessorWithTempPath:
    """Тесты FileProcessor с использованием tmp_path."""
    
    def test_write_and_read_file(self, tmp_path):
        """Запись и чтение файла через FileProcessor."""
        processor = FileProcessor()
        file_path = tmp_path / "test.txt"
        
        # Запись
        result = processor.write_file(file_path, "Hello, pytest!")
        assert result is True
        
        # Чтение
        content = processor.read_file(file_path)
        assert content == "Hello, pytest!"
    
    def test_read_nonexistent_file(self, tmp_path):
        """Чтение несуществующего файла вызывает исключение."""
        processor = FileProcessor()
        file_path = tmp_path / "missing.txt"
        
        with pytest.raises(FileNotFoundError) as exc_info:
            processor.read_file(file_path)
        
        assert "Файл не найден" in str(exc_info.value)
    
    def test_write_json(self, tmp_path):
        """Запись JSON в файл."""
        processor = FileProcessor()
        file_path = tmp_path / "data.json"
        data = {"name": "Test", "value": 42}
        
        result = processor.write_json(file_path, data)
        assert result is True
        
        # Проверяем содержимое
        saved = processor.read_json(file_path)
        assert saved == data
    
    def test_process_data_file(self, tmp_path):
        """Обработка файла с данными."""
        processor = FileProcessor()
        input_file = tmp_path / "input.txt"
        output_file = tmp_path / "output.txt"
        
        # Создаём входной файл
        input_file.write_text("hello\nworld\npytest\n")
        
        # Обрабатываем
        result = processor.process_data_file(input_file, output_file)
        
        # Проверяем результат
        assert result == ["HELLO", "WORLD", "PYTEST"]
        assert output_file.read_text() == "HELLO\nWORLD\nPYTEST"
    
    def test_process_empty_file(self, tmp_path):
        """Обработка пустого файла."""
        processor = FileProcessor()
        input_file = tmp_path / "empty.txt"
        output_file = tmp_path / "output.txt"
        
        input_file.write_text("")
        
        result = processor.process_data_file(input_file, output_file)
        
        assert result == []  # Пустой список
        assert output_file.read_text() == ""  # Пустой файл
Задание 1.3. Фикстура с подготовкой данных (10 минут)
Создайте фикстуру, которая создаёт несколько файлов для тестов.

python
@pytest.fixture
def test_environment(tmp_path):
    """
    Фикстура, создающая полноценную тестовую среду с файлами.
    """
    # Создаём структуру директорий
    data_dir = tmp_path / "data"
    output_dir = tmp_path / "output"
    data_dir.mkdir()
    output_dir.mkdir()
    
    # Создаём тестовые файлы
    (data_dir / "users.json").write_text(
        '{"users": [{"name": "Alice"}, {"name": "Bob"}]}'
    )
    (data_dir / "config.txt").write_text("mode=test\nlevel=debug")
    
    # Создаём вложенную структуру
    (data_dir / "subdir").mkdir()
    (data_dir / "subdir" / "nested.txt").write_text("nested content")
    
    yield {
        "tmp_path": tmp_path,
        "data_dir": data_dir,
        "output_dir": output_dir,
        "files": {
            "users": data_dir / "users.json",
            "config": data_dir / "config.txt",
            "nested": data_dir / "subdir" / "nested.txt"
        }
    }
    # Вся директория удаляется автоматически


class TestEnvironmentFixture:
    """Тесты с фикстурой test_environment."""
    
    def test_environment_structure(self, test_environment):
        """Проверка структуры тестовой среды."""
        assert test_environment["data_dir"].exists()
        assert test_environment["output_dir"].exists()
        assert test_environment["files"]["users"].exists()
        assert test_environment["files"]["config"].exists()
    
    def test_read_users_file(self, test_environment):
        """Чтение файла пользователей."""
        processor = FileProcessor()
        content = processor.read_json(test_environment["files"]["users"])
        assert "users" in content
        assert len(content["users"]) == 2
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться использовать встроенные фикстуры capsys (перехват вывода) и monkeypatch (подмена переменных окружения).

Задания для среднего уровня
Задание 2.1. Фикстура capsys (перехват stdout/stderr) (15 минут)
Используйте capsys для тестирования функций, которые выводят текст в консоль.

python
# test_file_processor_intermediate.py

import pytest
import sys
from file_processor import FileProcessor


class TestCapsys:
    """Тесты с использованием capsys (capture output)."""
    
    def test_print_summary(self, capsys):
        """Проверка вывода сводки."""
        processor = FileProcessor()
        data = ["apple", "banana", "cherry"]
        
        processor.print_summary(data)
        
        # Перехватываем вывод
        captured = capsys.readouterr()
        
        assert "Всего элементов: 3" in captured.out
        assert "1. apple" in captured.out
        assert "2. banana" in captured.out
        assert "3. cherry" in captured.out
    
    def test_log_error_to_stderr(self, capsys):
        """Проверка вывода ошибок в stderr."""
        processor = FileProcessor()
        
        processor.log_error("Something went wrong")
        
        captured = capsys.readouterr()
        assert "ERROR: Something went wrong" in captured.err
    
    def test_multiple_outputs(self, capsys):
        """Проверка нескольких выводов подряд."""
        processor = FileProcessor()
        
        processor.print_summary(["a", "b"])
        processor.log_error("Error 1")
        processor.print_summary(["c"])
        processor.log_error("Error 2")
        
        captured = capsys.readouterr()
        
        assert "Всего элементов: 2" in captured.out
        assert "Всего элементов: 1" in captured.out
        assert "ERROR: Error 1" in captured.err
        assert "ERROR: Error 2" in captured.err
    
    def test_no_output(self, capsys):
        """Проверка, что вывод отсутствует."""
        processor = FileProcessor()
        
        # Ничего не выводим
        captured = capsys.readouterr()
        assert captured.out == ""
        assert captured.err == ""
    
    def test_print_and_error_interleaved(self, capsys):
        """Перемешанный вывод в stdout и stderr."""
        print("Normal message")
        print("Error message", file=sys.stderr)
        print("Another normal message")
        
        captured = capsys.readouterr()
        
        assert "Normal message" in captured.out
        assert "Another normal message" in captured.out
        assert "Error message" in captured.err
    
    def test_capsys_clears_between_tests(self, capsys):
        """capsys очищается между тестами."""
        print("This should not appear in other tests")
        captured = capsys.readouterr()
        assert "should not appear" in captured.out
        # Следующий тест начнёт с пустого буфера
Задание 2.2. Фикстура monkeypatch (переменные окружения) (15 минут)
Используйте monkeypatch для подмены переменных окружения.

python
class TestMonkeypatchEnv:
    """Тесты с monkeypatch для переменных окружения."""
    
    def test_default_api_key(self, monkeypatch):
        """Проверка значения API ключа по умолчанию."""
        # Удаляем переменную окружения, если она есть
        monkeypatch.delenv("API_KEY", raising=False)
        
        from file_processor import ExternalService
        service = ExternalService()
        assert service.get_api_key() == "default-key"
    
    def test_custom_api_key_from_env(self, monkeypatch):
        """Проверка чтения API ключа из переменной окружения."""
        monkeypatch.setenv("API_KEY", "my-super-secret-key")
        
        from file_processor import ExternalService
        service = ExternalService()
        assert service.get_api_key() == "my-super-secret-key"
    
    def test_api_key_from_constructor(self, monkeypatch):
        """API ключ из конструктора имеет приоритет над env."""
        monkeypatch.setenv("API_KEY", "env-key")
        
        from file_processor import ExternalService
        service = ExternalService(api_key="constructor-key")
        assert service.get_api_key() == "constructor-key"
    
    def test_multiple_env_vars(self, monkeypatch):
        """Подмена нескольких переменных окружения."""
        monkeypatch.setenv("DB_HOST", "localhost")
        monkeypatch.setenv("DB_PORT", "5432")
        monkeypatch.setenv("DB_NAME", "testdb")
        
        assert os.getenv("DB_HOST") == "localhost"
        assert os.getenv("DB_PORT") == "5432"
        assert os.getenv("DB_NAME") == "testdb"
    
    def test_temporary_env_change(self, monkeypatch):
        """Временная подмена переменной."""
        original = os.getenv("API_KEY", "not-set")
        
        monkeypatch.setenv("API_KEY", "temp-value")
        assert os.getenv("API_KEY") == "temp-value"
        
        # После теста переменная восстанавливается
        # (проверяем в следующем тесте)
    
    def test_env_restored_after_test(self, monkeypatch):
        """Проверка, что переменные восстанавливаются."""
        # После предыдущего теста API_KEY должен вернуться к исходному
        assert os.getenv("API_KEY") != "temp-value"
Задание 2.3. Комбинация tmp_path и capsys (10 минут)
Создайте тесты, которые одновременно используют временные файлы и проверяют вывод.

python
class TestCombinedFixtures:
    """Комбинация tmp_path и capsys."""
    
    def test_process_and_print(self, tmp_path, capsys):
        """Обработка файла и проверка вывода."""
        processor = FileProcessor()
        input_file = tmp_path / "data.txt"
        output_file = tmp_path / "result.txt"
        
        input_file.write_text("line1\nline2\nline3")
        
        result = processor.process_data_file(input_file, output_file)
        processor.print_summary(result)
        
        captured = capsys.readouterr()
        assert "Всего элементов: 3" in captured.out
        assert output_file.exists()
    
    def test_error_logging_to_stderr(self, tmp_path, capsys):
        """Логирование ошибок в stderr."""
        processor = FileProcessor()
        missing_file = tmp_path / "missing.txt"
        
        with pytest.raises(FileNotFoundError):
            processor.read_file(missing_file)
        
        captured = capsys.readouterr()
        # Ошибка не выводится в stderr, только исключение
        # Но log_error выводит
        processor.log_error("Test error")
        captured = capsys.readouterr()
        assert "ERROR: Test error" in captured.err
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться использовать monkeypatch для подмены методов и функций, комбинировать несколько встроенных фикстур.

Задания для сложного уровня
Задание 3.1. monkeypatch для подмены методов (15 минут)
Используйте monkeypatch.setattr() для подмены методов класса.

python
# test_file_processor_advanced.py

import pytest
from pathlib import Path
from file_processor import FileProcessor, ExternalService


class TestMonkeypatchMethods:
    """Тесты с monkeypatch для подмены методов."""
    
    def test_mock_read_file(self, monkeypatch, tmp_path):
        """Подмена метода read_file для возврата фиктивных данных."""
        processor = FileProcessor()
        file_path = tmp_path / "test.txt"
        
        # Подменяем метод read_file
        def mock_read(path):
            return "mocked content"
        
        monkeypatch.setattr(processor, "read_file", mock_read)
        
        # Теперь read_file всегда возвращает "mocked content"
        content = processor.read_file(file_path)
        assert content == "mocked content"
        # Реальный файл даже не создавался
    
    def test_mock_write_file(self, monkeypatch):
        """Подмена метода write_file для проверки вызовов."""
        processor = FileProcessor()
        calls = []
        
        def mock_write(path, content):
            calls.append((path, content))
            return True
        
        monkeypatch.setattr(processor, "write_file", mock_write)
        
        processor.write_file(Path("a.txt"), "data1")
        processor.write_file(Path("b.txt"), "data2")
        
        assert len(calls) == 2
        assert calls[0][0] == Path("a.txt")
        assert calls[0][1] == "data1"
    
    def test_mock_external_service_fetch(self, monkeypatch):
        """Подмена метода fetch_data внешнего сервиса."""
        service = ExternalService()
        
        def mock_fetch(endpoint):
            return {"status": "mock", "data": f"Mocked {endpoint}"}
        
        monkeypatch.setattr(service, "fetch_data", mock_fetch)
        
        result = service.fetch_data("/users")
        assert result["status"] == "mock"
        assert result["data"] == "Mocked /users"
    
    def test_mock_with_side_effect(self, monkeypatch):
        """Подмена метода с побочным эффектом (исключение)."""
        processor = FileProcessor()
        
        def mock_read_raises(path):
            raise PermissionError("Access denied")
        
        monkeypatch.setattr(processor, "read_file", mock_read_raises)
        
        with pytest.raises(PermissionError, match="Access denied"):
            processor.read_file(Path("any.txt"))
    
    def test_mock_only_for_test(self, monkeypatch):
        """Мок действует только в пределах теста."""
        def mock_method():
            return "mocked"
        
        monkeypatch.setattr(FileProcessor, "get_log", mock_method)
        
        processor = FileProcessor()
        assert processor.get_log() == "mocked"
        # В других тестах метод остаётся оригинальным
Задание 3.2. monkeypatch для подмены глобальных функций и импортов (10 минут)
python
class TestMonkeypatchGlobals:
    """Подмена глобальных функций и импортов."""
    
    def test_mock_os_path_exists(self, monkeypatch):
        """Подмена os.path.exists для эмуляции отсутствия файла."""
        import os
        
        def mock_exists(path):
            return False  # Всегда говорим, что файла нет
        
        monkeypatch.setattr(os.path, "exists", mock_exists)
        
        processor = FileProcessor()
        # config_path не существует, но мы сами сказали, что нет
        assert processor.load_config() == {}
    
    def test_mock_open(self, monkeypatch):
        """Подмена встроенной функции open."""
        from io import StringIO
        
        mock_file = StringIO("mocked content")
        
        def mock_open(*args, **kwargs):
            return mock_file
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        processor = FileProcessor()
        # Чтение файла вернёт mocked content
        # Но реальный файл не создаётся
        content = processor.read_file(Path("any.txt"))
        assert content == "mocked content"
    
    def test_mock_datetime(self, monkeypatch):
        """Подмена datetime для тестирования времени."""
        import datetime
        
        class MockDateTime:
            @classmethod
            def now(cls):
                return datetime.datetime(2024, 1, 1, 12, 0, 0)
        
        monkeypatch.setattr(datetime, "datetime", MockDateTime)
        
        # Теперь datetime.now() возвращает фиксированную дату
        assert datetime.datetime.now().year == 2024
Задание 3.3. Комбинация нескольких встроенных фикстур (15 минут)
Создайте сложные тесты, использующие tmp_path, capsys и monkeypatch вместе.

python
class TestComplexFixtures:
    """Комбинация нескольких встроенных фикстур."""
    
    def test_full_pipeline(self, tmp_path, capsys, monkeypatch):
        """
        Полный тест: временные файлы + перехват вывода + моки.
        """
        processor = FileProcessor()
        input_file = tmp_path / "input.txt"
        output_file = tmp_path / "output.txt"
        
        # 1. Создаём тестовый файл
        input_file.write_text("test\ndata\nlines")
        
        # 2. Подменяем метод write_file для логирования
        write_calls = []
        original_write = processor.write_file
        
        def mock_write(path, content):
            write_calls.append((path, content))
            return original_write(path, content)
        
        monkeypatch.setattr(processor, "write_file", mock_write)
        
        # 3. Выполняем обработку
        result = processor.process_data_file(input_file, output_file)
        
        # 4. Проверяем вывод
        processor.print_summary(result)
        captured = capsys.readouterr()
        
        # 5. Проверки
        assert result == ["TEST", "DATA", "LINES"]
        assert len(write_calls) == 1
        assert write_calls[0][0] == output_file
        assert "Всего элементов: 3" in captured.out
    
    def test_config_loading_with_mock(self, tmp_path, monkeypatch):
        """Тест загрузки конфигурации с моком."""
        config_file = tmp_path / "config.json"
        config_file.write_text('{"mode": "test", "debug": true}')
        
        processor = FileProcessor(config_path=config_file)
        
        # Подменяем read_file для проверки
        read_calls = []
        original_read = processor.read_file
        
        def mock_read(path):
            read_calls.append(path)
            return original_read(path)
        
        monkeypatch.setattr(processor, "read_file", mock_read)
        
        config = processor.load_config()
        
        assert config["mode"] == "test"
        assert config["debug"] is True
        assert len(read_calls) == 1
        assert read_calls[0] == config_file
    
    def test_error_handling_with_monkeypatch(self, tmp_path, capsys):
        """Тест обработки ошибок с подменой."""
        processor = FileProcessor()
        bad_file = tmp_path / "bad.txt"
        
        # Создаём файл с правами только для чтения
        bad_file.write_text("content")
        bad_file.chmod(0o444)  # Только чтение
        
        # Пытаемся записать (должна быть ошибка)
        result = processor.write_file(bad_file, "new content")
        
        assert result is False
        assert len(processor.get_log()) == 1
        assert "Ошибка записи" in processor.get_log()[0]
    
    def test_monkeypatch_preserves_original(self, tmp_path, monkeypatch):
        """Проверка, что monkeypatch восстанавливает оригиналы."""
        processor = FileProcessor()
        file_path = tmp_path / "test.txt"
        
        def mock_exists(path):
            return True
        
        # Сохраняем оригинальный метод
        original_exists = processor.read_file
        
        monkeypatch.setattr(processor, "read_file", mock_exists)
        assert processor.read_file is mock_exists
        
        # После теста (или вручную) можно восстановить
        monkeypatch.undo()
Задание 3.4. Полный тестовый набор (15 минут)
Создайте полный набор тестов для ExternalService с использованием всех встроенных фикстур.

python
class TestExternalServiceComplete:
    """Полный набор тестов для ExternalService."""
    
    @pytest.fixture
    def service(self):
        """Фикстура с сервисом."""
        return ExternalService()
    
    def test_fetch_data_mocked(self, monkeypatch, service):
        """Тест fetch_data с моком."""
        def mock_get(endpoint):
            return {"status": "mocked", "data": "fake"}
        
        monkeypatch.setattr(service, "fetch_data", mock_get)
        
        result = service.fetch_data("/test")
        assert result["status"] == "mocked"
    
    def test_send_notification_captures_output(self, capsys, service):
        """Тест отправки уведомления с перехватом вывода."""
        result = service.send_notification("user@test.com", "Hello!")
        
        assert result is True
        captured = capsys.readouterr()
        assert "Sending to user@test.com: Hello!" in captured.out
    
    def test_api_key_from_env(self, monkeypatch):
        """Тест API ключа из окружения."""
        monkeypatch.setenv("API_KEY", "env-key-123")
        
        from file_processor import ExternalService
        service = ExternalService()
        assert service.get_api_key() == "env-key-123"
    
    def test_api_key_not_in_env(self, monkeypatch):
        """Тест API ключа по умолчанию."""
        monkeypatch.delenv("API_KEY", raising=False)
        
        from file_processor import ExternalService
        service = ExternalService()
        assert service.get_api_key() == "default-key"
    
    def test_send_multiple_notifications(self, capsys, service):
        """Тест множественных уведомлений."""
        users = ["alice", "bob", "charlie"]
        for user in users:
            service.send_notification(user, "Test")
        
        captured = capsys.readouterr()
        for user in users:
            assert f"Sending to {user}" in captured.out
Задание 3.5. Итоговый отчёт (5 минут)
markdown
## Итоговый отчёт по практической работе 3.7

**Студент:** _________________
**Уровень:** □ Базовый □ Средний □ Сложный
**Вариант №:** ___

### Использованные встроенные фикстуры

| Фикстура | Где использовалась | Количество тестов |
|:---|:---|:---|
| `tmp_path` | _____ | _____ |
| `capsys` | _____ | _____ |
| `monkeypatch` | _____ | _____ |

### Результаты тестирования

| Уровень | Всего тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| Базовый | _____ | _____ | _____ |
| Средний | _____ | _____ | _____ |
| Сложный | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Типичные ошибки при работе с фикстурами

1. 
2. 

### Выводы

_________________________________________________________________

_________________________________________________________________
