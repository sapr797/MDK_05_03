# Лекция 3.9: Тестирование файлового ввода-вывода. Фикстура tmp_path. Параметризация фикстур. Тестирование ошибок доступа

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить методы тестирования операций файлового ввода-вывода с использованием фикстуры `tmp_path`, освоить параметризацию фикстур для создания различных тестовых файлов, научиться тестировать обработку ошибок доступа (`PermissionError`, `FileNotFoundError`).

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Использовать фикстуру `tmp_path` для изоляции файловых тестов (ПК 3.2, ПК 3.3).
2. Применять параметризацию фикстур для создания различных тестовых файлов (ПК 3.2).
3. Тестировать обработку `FileNotFoundError` и `PermissionError` (ПК 3.3).
4. Проверять состояние файловой системы после операций (ОК 02, ОК 05).

---

## 1. Фикстура tmp_path

### 1.1 Что такое tmp_path?

`tmp_path` — встроенная фикстура pytest, которая создаёт **уникальную временную директорию** для каждого теста и **автоматически удаляет** её после завершения.
┌─────────────────────────────────────────────────────────────────────────────┐
│ ПРИНЦИП РАБОТЫ tmp_path │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ def test_example(tmp_path): │
│ # tmp_path — объект pathlib.Path │
│ file = tmp_path / "data.txt" │
│ file.write_text("content") │
│ assert file.exists() │
│ │
│ Временная директория: /tmp/pytest-of-user/pytest-current/test_0/ │
│ После теста: директория автоматически удаляется │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 1.2 Базовое использование

```python
import pytest
from pathlib import Path


def test_write_and_read(tmp_path):
    """Запись и чтение файла во временной директории."""
    # Создаём файл
    file_path = tmp_path / "test.txt"
    file_path.write_text("Hello, World!")
    
    # Проверяем
    assert file_path.exists()
    assert file_path.read_text() == "Hello, World!"


def test_create_subdirectory(tmp_path):
    """Создание вложенной структуры."""
    # Создаём поддиректорию
    data_dir = tmp_path / "data" / "subdir"
    data_dir.mkdir(parents=True)
    
    # Создаём несколько файлов
    (data_dir / "config.json").write_text('{"mode": "test"}')
    (data_dir / "log.txt").write_text("Log entry")
    
    assert (data_dir / "config.json").exists()
    assert (data_dir / "log.json").exists() is False


def test_multiple_files(tmp_path):
    """Работа с несколькими файлами."""
    files = ["a.txt", "b.txt", "c.txt"]
    
    for name in files:
        (tmp_path / name).write_text(f"Content of {name}")
    
    for name in files:
        assert (tmp_path / name).exists()
        assert (tmp_path / name).read_text() == f"Content of {name}"
2. Параметризация фикстур для работы с файлами
2.1 Параметризация данных для файловых тестов
python
@pytest.mark.parametrize("filename,content", [
    ("test1.txt", "Hello"),
    ("test2.txt", "World"),
    ("data.json", '{"key": "value"}'),
    ("empty.txt", ""),
])
def test_write_various_files(tmp_path, filename, content):
    """Параметризованный тест записи разных файлов."""
    file_path = tmp_path / filename
    file_path.write_text(content)
    
    assert file_path.exists()
    assert file_path.read_text() == content


@pytest.mark.parametrize("file_size", [0, 100, 1024, 1024 * 1024])
def test_large_file(tmp_path, file_size):
    """Тест файлов разного размера."""
    file_path = tmp_path / f"size_{file_size}.txt"
    content = "A" * file_size
    file_path.write_text(content)
    
    assert file_path.stat().st_size == file_size
2.2 Параметризация фикстур (фикстура с params)
python
@pytest.fixture(params=[
    ("data.csv", "name,age\nAlice,30\nBob,25"),
    ("config.json", '{"debug": true}'),
    ("log.txt", "INFO: Started\nERROR: Failed"),
])
def test_file(request):
    """Параметризованная фикстура с разными файлами."""
    filename, content = request.param
    file_path = tmp_path / filename
    file_path.write_text(content)
    return file_path, content


def test_file_content(test_file):
    """Тест с параметризованной фикстурой."""
    file_path, expected_content = test_file
    assert file_path.exists()
    assert file_path.read_text() == expected_content
2.3 Комбинация параметризации фикстур и тестов
python
@pytest.fixture(params=["json", "csv", "txt"])
def file_extension(request):
    """Фикстура с расширением файла."""
    return request.param


@pytest.mark.parametrize("content", [
    '{"data": "test"}',
    "name,value\na,1",
    "plain text",
])
def test_combined_params(tmp_path, file_extension, content):
    """Комбинация параметризации фикстуры и теста."""
    filename = f"data.{file_extension}"
    file_path = tmp_path / filename
    file_path.write_text(content)
    
    assert file_path.exists()
    assert file_path.suffix == f".{file_extension}"
3. Тестирование FileNotFoundError
3.1 Чтение несуществующего файла
python
def test_read_nonexistent_file(tmp_path):
    """Чтение несуществующего файла -> FileNotFoundError."""
    file_path = tmp_path / "missing.txt"
    
    with pytest.raises(FileNotFoundError) as exc_info:
        file_path.read_text()
    
    assert "missing.txt" in str(exc_info.value)


def test_safe_read_function(tmp_path):
    """Функция с обработкой FileNotFoundError."""
    
    def safe_read(file_path: Path) -> str:
        try:
            return file_path.read_text()
        except FileNotFoundError:
            return ""
    
    missing_file = tmp_path / "not_here.txt"
    assert safe_read(missing_file) == ""
    
    existing_file = tmp_path / "exists.txt"
    existing_file.write_text("content")
    assert safe_read(existing_file) == "content"
3.2 Тестирование удаления несуществующего файла
python
def test_delete_nonexistent_file(tmp_path):
    """Удаление несуществующего файла с разными стратегиями."""
    file_path = tmp_path / "missing.txt"
    
    # Без ignore_missing
    with pytest.raises(FileNotFoundError):
        file_path.unlink()
    
    # С ignore_missing
    try:
        file_path.unlink(missing_ok=True)
    except FileNotFoundError:
        pytest.fail("Should not raise with missing_ok=True")


def test_safe_delete_function(tmp_path):
    """Функция безопасного удаления."""
    
    def safe_delete(file_path: Path, ignore_missing: bool = False) -> bool:
        try:
            file_path.unlink()
            return True
        except FileNotFoundError:
            return ignore_missing
        except PermissionError:
            return False
    
    missing = tmp_path / "missing.txt"
    assert safe_delete(missing, ignore_missing=True) is True
    assert safe_delete(missing, ignore_missing=False) is False
4. Тестирование PermissionError
4.1 Эмуляция PermissionError с monkeypatch
python
def test_permission_error_on_read(tmp_path, monkeypatch):
    """Эмуляция ошибки доступа при чтении."""
    file_path = tmp_path / "protected.txt"
    file_path.write_text("secret")
    
    original_open = open
    
    def mock_open(*args, **kwargs):
        if args[0] == str(file_path) and 'r' in args[1]:
            raise PermissionError("Access denied")
        return original_open(*args, **kwargs)
    
    monkeypatch.setattr("builtins.open", mock_open)
    
    with pytest.raises(PermissionError, match="Access denied"):
        file_path.read_text()


def test_permission_error_on_write(tmp_path, monkeypatch):
    """Эмуляция ошибки доступа при записи."""
    file_path = tmp_path / "readonly.txt"
    
    original_open = open
    
    def mock_open(*args, **kwargs):
        if args[0] == str(file_path) and 'w' in args[1]:
            raise PermissionError("Read-only file system")
        return original_open(*args, **kwargs)
    
    monkeypatch.setattr("builtins.open", mock_open)
    
    with pytest.raises(PermissionError, match="Read-only"):
        file_path.write_text("cannot write")
4.2 Эмуляция PermissionError с pytest-mock
python
def test_permission_error_with_mocker(tmp_path, mocker):
    """Эмуляция PermissionError с pytest-mock."""
    file_path = tmp_path / "protected.txt"
    file_path.write_text("content")
    
    mock_open = mocker.patch("builtins.open")
    mock_open.side_effect = PermissionError("Access denied")
    
    with pytest.raises(PermissionError):
        with open(file_path, 'r') as f:
            f.read()


def test_os_access_mock(mocker):
    """Мокирование os.access для эмуляции отсутствия прав."""
    import os
    
    mocker.patch("os.access", return_value=False)
    mocker.patch("os.path.exists", return_value=True)
    
    assert os.access("/some/file", os.R_OK) is False
5. Комплексное тестирование файловых операций
5.1 Пример: класс FileProcessor
python
# file_processor.py

from pathlib import Path
from typing import Optional


class FileProcessor:
    """Класс для безопасной работы с файлами."""
    
    def read_file(self, path: Path) -> Optional[str]:
        """Безопасное чтение файла."""
        try:
            return path.read_text()
        except FileNotFoundError:
            return None
        except PermissionError:
            return None
    
    def write_file(self, path: Path, content: str, overwrite: bool = True) -> bool:
        """Безопасная запись файла."""
        if path.exists() and not overwrite:
            return False
        
        try:
            path.write_text(content)
            return True
        except PermissionError:
            return False
        except OSError:
            return False
    
    def copy_file(self, src: Path, dst: Path, overwrite: bool = False) -> bool:
        """Копирование файла."""
        if not src.exists():
            return False
        
        if dst.exists() and not overwrite:
            return False
        
        try:
            dst.write_text(src.read_text())
            return True
        except (PermissionError, OSError):
            return False
5.2 Тестирование FileProcessor
python
# test_file_processor.py

import pytest
from pathlib import Path
from file_processor import FileProcessor


class TestFileProcessor:
    """Тесты для FileProcessor."""
    
    @pytest.fixture
    def processor(self):
        return FileProcessor()
    
    # ==========================================================
    # ТЕСТЫ read_file
    # ==========================================================
    
    def test_read_existing_file(self, tmp_path, processor):
        file_path = tmp_path / "data.txt"
        file_path.write_text("content")
        
        result = processor.read_file(file_path)
        assert result == "content"
    
    def test_read_nonexistent_file(self, tmp_path, processor):
        file_path = tmp_path / "missing.txt"
        
        result = processor.read_file(file_path)
        assert result is None
    
    def test_read_permission_error(self, tmp_path, processor, monkeypatch):
        file_path = tmp_path / "protected.txt"
        file_path.write_text("secret")
        
        def mock_open(*args, **kwargs):
            raise PermissionError("Access denied")
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        result = processor.read_file(file_path)
        assert result is None
    
    # ==========================================================
    # ТЕСТЫ write_file
    # ==========================================================
    
    def test_write_new_file(self, tmp_path, processor):
        file_path = tmp_path / "new.txt"
        
        result = processor.write_file(file_path, "Hello")
        
        assert result is True
        assert file_path.exists()
        assert file_path.read_text() == "Hello"
    
    def test_write_overwrite_existing(self, tmp_path, processor):
        file_path = tmp_path / "existing.txt"
        file_path.write_text("old")
        
        result = processor.write_file(file_path, "new", overwrite=True)
        
        assert result is True
        assert file_path.read_text() == "new"
    
    def test_write_no_overwrite(self, tmp_path, processor):
        file_path = tmp_path / "existing.txt"
        file_path.write_text("original")
        
        result = processor.write_file(file_path, "new", overwrite=False)
        
        assert result is False
        assert file_path.read_text() == "original"
    
    def test_write_permission_error(self, tmp_path, processor, monkeypatch):
        file_path = tmp_path / "readonly.txt"
        
        def mock_open(*args, **kwargs):
            raise PermissionError("Read-only")
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        result = processor.write_file(file_path, "content")
        
        assert result is False
        assert not file_path.exists()
    
    # ==========================================================
    # ТЕСТЫ copy_file
    # ==========================================================
    
    def test_copy_file_success(self, tmp_path, processor):
        src = tmp_path / "source.txt"
        dst = tmp_path / "dest.txt"
        src.write_text("copy me")
        
        result = processor.copy_file(src, dst)
        
        assert result is True
        assert dst.exists()
        assert dst.read_text() == "copy me"
    
    def test_copy_file_source_not_found(self, tmp_path, processor):
        src = tmp_path / "missing.txt"
        dst = tmp_path / "dest.txt"
        
        result = processor.copy_file(src, dst)
        
        assert result is False
        assert not dst.exists()
    
    def test_copy_file_dest_exists_no_overwrite(self, tmp_path, processor):
        src = tmp_path / "source.txt"
        dst = tmp_path / "dest.txt"
        src.write_text("new")
        dst.write_text("old")
        
        result = processor.copy_file(src, dst, overwrite=False)
        
        assert result is False
        assert dst.read_text() == "old"
6. Шпаргалка
python
# === tmp_path ===
def test_example(tmp_path):
    file = tmp_path / "data.txt"
    file.write_text("content")
    assert file.exists()

# === ПАРАМЕТРИЗАЦИЯ ФИКСТУРЫ ===
@pytest.fixture(params=["a.txt", "b.txt"])
def test_file(tmp_path, request):
    file = tmp_path / request.param
    file.write_text("data")
    return file

# === FileNotFoundError ===
def test_not_found(tmp_path):
    with pytest.raises(FileNotFoundError):
        (tmp_path / "missing.txt").read_text()

# === PermissionError (мокирование) ===
def test_permission_error(monkeypatch, tmp_path):
    def mock_open(*args):
        raise PermissionError("Access denied")
    monkeypatch.setattr("builtins.open", mock_open)
    
    with pytest.raises(PermissionError):
        (tmp_path / "file.txt").write_text("data")
Контрольные вопросы
Что делает фикстура tmp_path и почему она полезна?

Как создать вложенную структуру директорий с tmp_path?

Как параметризовать фикстуру для создания разных файлов?

Как проверить, что при чтении несуществующего файла выбрасывается FileNotFoundError?

Как эмулировать PermissionError при записи файла?

Как протестировать функцию, которая возвращает None при ошибках чтения?

Как убедиться, что временные файлы удаляются после теста?

Как мокировать os.access для проверки прав доступа?

Как протестировать копирование файла с ошибкой доступа?

Как комбинировать параметризацию фикстуры и теста?

Следующее занятие: ПР 3.9 — Практическое тестирование файлового ввода-вывода.
