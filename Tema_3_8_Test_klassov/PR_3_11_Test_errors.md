# Практическое занятие 3.11: Тестирование обработки ошибок файлового ввода-вывода

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.7:** Тестирование файлового ввода-вывода  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться тестировать обработку ошибок файлового ввода-вывода с использованием `pytest.raises` и `monkeypatch`, эмулировать различные ошибки доступа к файлам и проверять корректность реакции программы на негативные сценарии.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Тестировать обработку `FileNotFoundError` при работе с файлами (ПК 3.2, ПК 3.3).
2. Использовать `monkeypatch` для эмуляции `PermissionError` и других ошибок ввода-вывода (ПК 3.2).
3. Разрабатывать негативные сценарии для файловых операций (ОК 02, ОК 05).
4. Проверять корректность сообщений об ошибках и состояние системы после исключений (ПК 3.3).

---

## Теоретическая справка

### Типичные ошибки файлового ввода-вывода

| Исключение | Условие возникновения |
|:---|:---|
| `FileNotFoundError` | Файл или директория не существует |
| `PermissionError` | Недостаточно прав доступа |
| `IsADirectoryError` | Ожидался файл, а указана директория |
| `NotADirectoryError` | Ожидалась директория, а указан файл |
| `OSError` | Общая ошибка ОС (диск заполнен, и т.д.) |

### Подходы к тестированию

```python
# 1. Прямая проверка исключения
with pytest.raises(FileNotFoundError):
    open("/nonexistent/file.txt")

# 2. Мокирование ошибок
def mock_open_error(*args, **kwargs):
    raise PermissionError("Access denied")

monkeypatch.setattr("builtins.open", mock_open_error)
Объект тестирования
Листинг 1. Модуль file_handler.py
python
# file_handler.py

import os
import json
import shutil
from pathlib import Path
from typing import List, Dict, Any, Optional


class FileHandler:
    """Класс для работы с файлами с обработкой ошибок."""
    
    @staticmethod
    def safe_read(file_path: str) -> str:
        """
        Безопасное чтение файла.
        
        Args:
            file_path: Путь к файлу
        
        Returns:
            Содержимое файла
        
        Raises:
            FileNotFoundError: Если файл не существует
            PermissionError: Если нет прав на чтение
        """
        if not os.path.exists(file_path):
            raise FileNotFoundError(f"File not found: {file_path}")
        
        if not os.access(file_path, os.R_OK):
            raise PermissionError(f"No read permission for: {file_path}")
        
        with open(file_path, 'r', encoding='utf-8') as f:
            return f.read()
    
    @staticmethod
    def safe_write(file_path: str, content: str, overwrite: bool = True) -> bool:
        """
        Безопасная запись в файл.
        
        Args:
            file_path: Путь к файлу
            content: Содержимое для записи
            overwrite: Перезаписывать ли существующий файл
        
        Returns:
            True если запись успешна
        
        Raises:
            FileExistsError: Если файл существует и overwrite=False
            PermissionError: Если нет прав на запись
            OSError: При ошибках ввода-вывода
        """
        if os.path.exists(file_path) and not overwrite:
            raise FileExistsError(f"File already exists: {file_path}")
        
        # Проверяем директорию
        dir_path = os.path.dirname(file_path)
        if dir_path and not os.path.exists(dir_path):
            os.makedirs(dir_path, exist_ok=True)
        
        try:
            with open(file_path, 'w', encoding='utf-8') as f:
                f.write(content)
            return True
        except PermissionError as e:
            raise PermissionError(f"Cannot write to {file_path}: {e}")
        except OSError as e:
            raise OSError(f"IO error while writing to {file_path}: {e}")
    
    @staticmethod
    def safe_read_json(file_path: str) -> Dict[str, Any]:
        """
        Безопасное чтение JSON из файла.
        
        Args:
            file_path: Путь к JSON-файлу
        
        Returns:
            Десериализованный JSON
        
        Raises:
            FileNotFoundError: Если файл не существует
            json.JSONDecodeError: Если файл содержит невалидный JSON
        """
        content = FileHandler.safe_read(file_path)
        
        try:
            return json.loads(content)
        except json.JSONDecodeError as e:
            raise json.JSONDecodeError(
                f"Invalid JSON in {file_path}: {e.msg}",
                e.doc,
                e.pos
            )
    
    @staticmethod
    def safe_write_json(file_path: str, data: Dict[str, Any], indent: int = 2) -> bool:
        """
        Безопасная запись JSON в файл.
        """
        content = json.dumps(data, indent=indent, ensure_ascii=False)
        return FileHandler.safe_write(file_path, content)
    
    @staticmethod
    def copy_file(src: str, dst: str, overwrite: bool = False) -> bool:
        """
        Копирование файла.
        
        Raises:
            FileNotFoundError: Если исходный файл не существует
            FileExistsError: Если целевой файл существует и overwrite=False
            PermissionError: Если нет прав
        """
        if not os.path.exists(src):
            raise FileNotFoundError(f"Source file not found: {src}")
        
        if os.path.exists(dst) and not overwrite:
            raise FileExistsError(f"Destination file already exists: {dst}")
        
        try:
            shutil.copy2(src, dst)
            return True
        except PermissionError as e:
            raise PermissionError(f"Cannot copy: {e}")
    
    @staticmethod
    def delete_file(file_path: str, ignore_missing: bool = False) -> bool:
        """
        Удаление файла.
        
        Args:
            file_path: Путь к файлу
            ignore_missing: Игнорировать отсутствие файла
        
        Returns:
            True если файл был удалён или не существовал (при ignore_missing=True)
        
        Raises:
            FileNotFoundError: Если файл не существует и ignore_missing=False
            PermissionError: Если нет прав на удаление
        """
        if not os.path.exists(file_path):
            if ignore_missing:
                return False
            raise FileNotFoundError(f"File not found: {file_path}")
        
        try:
            os.remove(file_path)
            return True
        except PermissionError as e:
            raise PermissionError(f"Cannot delete {file_path}: {e}")
    
    @staticmethod
    def get_file_info(file_path: str) -> Dict[str, Any]:
        """
        Получение информации о файле.
        
        Returns:
            Словарь с информацией: size, created, modified, is_file, is_dir
        
        Raises:
            FileNotFoundError: Если файл не существует
        """
        if not os.path.exists(file_path):
            raise FileNotFoundError(f"File not found: {file_path}")
        
        stat = os.stat(file_path)
        
        return {
            "path": file_path,
            "size": stat.st_size,
            "created": stat.st_ctime,
            "modified": stat.st_mtime,
            "is_file": os.path.isfile(file_path),
            "is_dir": os.path.isdir(file_path)
        }
    
    @staticmethod
    def append_to_file(file_path: str, content: str) -> bool:
        """
        Добавление содержимого в конец файла.
        
        Raises:
            FileNotFoundError: Если файл не существует
            PermissionError: Если нет прав на запись
        """
        if not os.path.exists(file_path):
            raise FileNotFoundError(f"File not found: {file_path}")
        
        try:
            with open(file_path, 'a', encoding='utf-8') as f:
                f.write(content)
            return True
        except PermissionError as e:
            raise PermissionError(f"Cannot append to {file_path}: {e}")
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Тестирование FileNotFoundError, базовые сценарии
Средний (варианты 9-17)	Средние студенты	+ PermissionError, мокирование с monkeypatch
Сложный (варианты 18-25)	Сильные студенты	+ все типы ошибок, комплексные сценарии
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться тестировать обработку FileNotFoundError и базовые позитивные сценарии.

Задания для базового уровня
Задание 1.1. Создание тестов для safe_read (15 минут)
python
# test_file_handler_base.py

import pytest
import os
from file_handler import FileHandler


class TestSafeReadBase:
    """Базовые тесты для safe_read."""
    
    # ==========================================================
    # ПОЗИТИВНЫЕ СЦЕНАРИИ
    # ==========================================================
    
    def test_read_existing_file(self, tmp_path):
        """Чтение существующего файла."""
        file_path = tmp_path / "test.txt"
        file_path.write_text("Hello, World!")
        
        content = FileHandler.safe_read(str(file_path))
        
        assert content == "Hello, World!"
    
    def test_read_empty_file(self, tmp_path):
        """Чтение пустого файла."""
        file_path = tmp_path / "empty.txt"
        file_path.write_text("")
        
        content = FileHandler.safe_read(str(file_path))
        
        assert content == ""
    
    def test_read_file_with_unicode(self, tmp_path):
        """Чтение файла с Unicode символами."""
        file_path = tmp_path / "unicode.txt"
        content = "Привет, мир! 你好 Hello 🐍"
        file_path.write_text(content, encoding='utf-8')
        
        result = FileHandler.safe_read(str(file_path))
        
        assert result == content
    
    # ==========================================================
    # НЕГАТИВНЫЕ СЦЕНАРИИ - FileNotFoundError
    # ==========================================================
    
    def test_read_nonexistent_file(self, tmp_path):
        """Чтение несуществующего файла -> FileNotFoundError."""
        file_path = tmp_path / "nonexistent.txt"
        
        with pytest.raises(FileNotFoundError) as exc_info:
            FileHandler.safe_read(str(file_path))
        
        assert "File not found" in str(exc_info.value)
        assert str(file_path) in str(exc_info.value)
    
    def test_read_in_directory_not_exists(self, tmp_path):
        """Чтение файла в несуществующей директории."""
        file_path = tmp_path / "missing_dir" / "file.txt"
        
        with pytest.raises(FileNotFoundError):
            FileHandler.safe_read(str(file_path))
    
    def test_read_path_is_directory(self, tmp_path):
        """Чтение директории как файла."""
        dir_path = tmp_path / "test_dir"
        dir_path.mkdir()
        
        with pytest.raises(FileNotFoundError):
            # os.path.exists вернёт True, но open выдаст IsADirectoryError
            # Наша функция проверяет os.path.exists, но не является ли это директорией
            FileHandler.safe_read(str(dir_path))
Задание 1.2. Тестирование safe_write (15 минут)
python
class TestSafeWriteBase:
    """Базовые тесты для safe_write."""
    
    # ==========================================================
    # ПОЗИТИВНЫЕ СЦЕНАРИИ
    # ==========================================================
    
    def test_write_new_file(self, tmp_path):
        """Запись в новый файл."""
        file_path = tmp_path / "new.txt"
        
        result = FileHandler.safe_write(str(file_path), "test content")
        
        assert result is True
        assert file_path.exists()
        assert file_path.read_text() == "test content"
    
    def test_write_overwrite_existing(self, tmp_path):
        """Перезапись существующего файла."""
        file_path = tmp_path / "existing.txt"
        file_path.write_text("old content")
        
        result = FileHandler.safe_write(str(file_path), "new content", overwrite=True)
        
        assert result is True
        assert file_path.read_text() == "new content"
    
    def test_write_create_directories(self, tmp_path):
        """Создание промежуточных директорий."""
        file_path = tmp_path / "a" / "b" / "c" / "deep.txt"
        
        result = FileHandler.safe_write(str(file_path), "deep content")
        
        assert result is True
        assert file_path.exists()
        assert file_path.read_text() == "deep content"
    
    def test_write_empty_content(self, tmp_path):
        """Запись пустого содержимого."""
        file_path = tmp_path / "empty.txt"
        
        result = FileHandler.safe_write(str(file_path), "")
        
        assert result is True
        assert file_path.read_text() == ""
    
    # ==========================================================
    # НЕГАТИВНЫЕ СЦЕНАРИИ - FileExistsError
    # ==========================================================
    
    def test_write_no_overwrite_file_exists(self, tmp_path):
        """Запись без перезаписи существующего файла."""
        file_path = tmp_path / "existing.txt"
        file_path.write_text("original")
        
        with pytest.raises(FileExistsError) as exc_info:
            FileHandler.safe_write(str(file_path), "new", overwrite=False)
        
        assert "File already exists" in str(exc_info.value)
        assert file_path.read_text() == "original"  # Содержимое не изменилось
Задание 1.3. Тестирование delete_file (10 минут)
python
class TestDeleteFileBase:
    """Базовые тесты для delete_file."""
    
    def test_delete_existing_file(self, tmp_path):
        """Удаление существующего файла."""
        file_path = tmp_path / "to_delete.txt"
        file_path.write_text("delete me")
        
        result = FileHandler.delete_file(str(file_path))
        
        assert result is True
        assert not file_path.exists()
    
    def test_delete_nonexistent_file_raises(self, tmp_path):
        """Удаление несуществующего файла -> FileNotFoundError."""
        file_path = tmp_path / "missing.txt"
        
        with pytest.raises(FileNotFoundError) as exc_info:
            FileHandler.delete_file(str(file_path))
        
        assert "File not found" in str(exc_info.value)
    
    def test_delete_nonexistent_with_ignore(self, tmp_path):
        """Удаление несуществующего файла с ignore_missing=True."""
        file_path = tmp_path / "missing.txt"
        
        result = FileHandler.delete_file(str(file_path), ignore_missing=True)
        
        assert result is False
Задание 1.4. Тестирование get_file_info (10 минут)
python
class TestGetFileInfoBase:
    """Базовые тесты для get_file_info."""
    
    def test_get_info_existing_file(self, tmp_path):
        """Получение информации о существующем файле."""
        file_path = tmp_path / "info.txt"
        file_path.write_text("test")
        
        info = FileHandler.get_file_info(str(file_path))
        
        assert info["path"] == str(file_path)
        assert info["size"] == 4  # "test" = 4 байта
        assert info["is_file"] is True
        assert info["is_dir"] is False
        assert "created" in info
        assert "modified" in info
    
    def test_get_info_nonexistent_file(self, tmp_path):
        """Информация о несуществующем файле -> FileNotFoundError."""
        file_path = tmp_path / "missing.txt"
        
        with pytest.raises(FileNotFoundError):
            FileHandler.get_file_info(str(file_path))
Задание 1.5. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Результаты тестирования

| Функция | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| safe_read | _____ | _____ | _____ |
| safe_write | _____ | _____ | _____ |
| delete_file | _____ | _____ | _____ |
| get_file_info | _____ | _____ | _____ |

### Найденные дефекты

| ID | Описание | Серьёзность |
|:---|:---|:---|
| | | |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться тестировать PermissionError с использованием monkeypatch и эмулировать ошибки доступа.

Задания для среднего уровня
Задание 2.1. Мокирование PermissionError для safe_read (15 минут)
python
# test_file_handler_intermediate.py

import pytest
import os
from file_handler import FileHandler


class TestPermissionErrors:
    """Тесты для ошибок доступа с мокированием."""
    
    # ==========================================================
    # ТЕСТЫ С МОКИРОВАНИЕМ os.access
    # ==========================================================
    
    def test_read_no_permission(self, tmp_path, monkeypatch):
        """Чтение файла без прав доступа."""
        file_path = tmp_path / "secret.txt"
        file_path.write_text("secret content")
        
        # Мокируем os.access, чтобы он возвращал False для R_OK
        def mock_access(path, mode):
            if mode == os.R_OK:
                return False
            return os.access(path, mode)
        
        monkeypatch.setattr(os, "access", mock_access)
        
        with pytest.raises(PermissionError) as exc_info:
            FileHandler.safe_read(str(file_path))
        
        assert "No read permission" in str(exc_info.value)
    
    def test_read_permission_error_on_open(self, tmp_path, monkeypatch):
        """Ошибка PermissionError при открытии файла."""
        file_path = tmp_path / "locked.txt"
        file_path.write_text("locked")
        
        original_open = open
        
        def mock_open(*args, **kwargs):
            if args[0] == str(file_path):
                raise PermissionError("Access denied")
            return original_open(*args, **kwargs)
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        with pytest.raises(PermissionError) as exc_info:
            FileHandler.safe_read(str(file_path))
        
        assert "Access denied" in str(exc_info.value) or "permission" in str(exc_info.value).lower()
    
    # ==========================================================
    # МОКИРОВАНИЕ ПРОВЕРКИ СУЩЕСТВОВАНИЯ ФАЙЛА
    # ==========================================================
    
    def test_read_file_not_exists_with_mock(self, tmp_path, monkeypatch):
        """Файл не существует по мнению os.path.exists."""
        file_path = tmp_path / "real.txt"
        file_path.write_text("real")
        
        def mock_exists(path):
            return False
        
        monkeypatch.setattr(os.path, "exists", mock_exists)
        
        with pytest.raises(FileNotFoundError):
            FileHandler.safe_read(str(file_path))
Задание 2.2. Тестирование PermissionError для safe_write (15 минут)
python
class TestWritePermissionErrors:
    """Тесты ошибок доступа при записи."""
    
    def test_write_no_directory_permission(self, tmp_path, monkeypatch):
        """Нет прав на создание директории."""
        file_path = tmp_path / "subdir" / "file.txt"
        
        def mock_makedirs(path, exist_ok=False):
            raise PermissionError("Cannot create directory")
        
        monkeypatch.setattr(os, "makedirs", mock_makedirs)
        
        with pytest.raises(PermissionError) as exc_info:
            FileHandler.safe_write(str(file_path), "content")
        
        assert "Cannot create directory" in str(exc_info.value)
    
    def test_write_permission_error_on_open(self, tmp_path, monkeypatch):
        """PermissionError при открытии файла на запись."""
        file_path = tmp_path / "write_protected.txt"
        
        original_open = open
        
        def mock_open(*args, **kwargs):
            if 'w' in args or kwargs.get('mode') == 'w':
                raise PermissionError("Read-only file system")
            return original_open(*args, **kwargs)
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        with pytest.raises(PermissionError) as exc_info:
            FileHandler.safe_write(str(file_path), "content")
        
        assert "Read-only" in str(exc_info.value) or "permission" in str(exc_info.value).lower()
    
    def test_write_os_error_disk_full(self, tmp_path, monkeypatch):
        """OSError при заполнении диска."""
        file_path = tmp_path / "large.txt"
        
        original_open = open
        
        def mock_open(*args, **kwargs):
            if 'w' in args or kwargs.get('mode') == 'w':
                raise OSError("No space left on device")
            return original_open(*args, **kwargs)
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        with pytest.raises(OSError) as exc_info:
            FileHandler.safe_write(str(file_path), "large content")
        
        assert "No space left" in str(exc_info.value) or "IO error" in str(exc_info.value)
Задание 2.3. Тестирование JSON ошибок (10 минут)
python
class TestJsonErrors:
    """Тесты для ошибок при работе с JSON."""
    
    def test_read_invalid_json(self, tmp_path):
        """Чтение файла с невалидным JSON."""
        file_path = tmp_path / "invalid.json"
        file_path.write_text('{"invalid": json}')
        
        with pytest.raises(json.JSONDecodeError) as exc_info:
            FileHandler.safe_read_json(str(file_path))
        
        assert "Invalid JSON" in str(exc_info.value)
    
    def test_read_invalid_json_syntax(self, tmp_path):
        """JSON с синтаксической ошибкой."""
        file_path = tmp_path / "bad.json"
        file_path.write_text('{key: "value"}')  # Не хватает кавычек вокруг key
        
        with pytest.raises(json.JSONDecodeError):
            FileHandler.safe_read_json(str(file_path))
    
    def test_write_json_success(self, tmp_path):
        """Успешная запись JSON."""
        file_path = tmp_path / "data.json"
        data = {"name": "test", "value": 42}
        
        result = FileHandler.safe_write_json(str(file_path), data)
        
        assert result is True
        assert file_path.exists()
        
        loaded = json.loads(file_path.read_text())
        assert loaded == data
Задание 2.4. Тестирование copy_file (10 минут)
python
class TestCopyFile:
    """Тесты для копирования файлов."""
    
    def test_copy_success(self, tmp_path):
        """Успешное копирование."""
        src = tmp_path / "source.txt"
        dst = tmp_path / "dest.txt"
        src.write_text("copy me")
        
        result = FileHandler.copy_file(str(src), str(dst))
        
        assert result is True
        assert dst.exists()
        assert dst.read_text() == "copy me"
    
    def test_copy_source_not_exists(self, tmp_path):
        """Исходный файл не существует."""
        src = tmp_path / "missing.txt"
        dst = tmp_path / "dest.txt"
        
        with pytest.raises(FileNotFoundError) as exc_info:
            FileHandler.copy_file(str(src), str(dst))
        
        assert "Source file not found" in str(exc_info.value)
    
    def test_copy_dest_exists_no_overwrite(self, tmp_path):
        """Целевой файл существует и overwrite=False."""
        src = tmp_path / "source.txt"
        dst = tmp_path / "dest.txt"
        src.write_text("source")
        dst.write_text("original")
        
        with pytest.raises(FileExistsError) as exc_info:
            FileHandler.copy_file(str(src), str(dst), overwrite=False)
        
        assert "already exists" in str(exc_info.value)
        assert dst.read_text() == "original"  # Не перезаписан
    
    def test_copy_dest_exists_overwrite(self, tmp_path):
        """Целевой файл существует и overwrite=True."""
        src = tmp_path / "source.txt"
        dst = tmp_path / "dest.txt"
        src.write_text("new content")
        dst.write_text("old content")
        
        result = FileHandler.copy_file(str(src), str(dst), overwrite=True)
        
        assert result is True
        assert dst.read_text() == "new content"
Задание 2.5. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Результаты тестирования

| Функция | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| safe_read | _____ | _____ | _____ |
| safe_write | _____ | _____ | _____ |
| safe_read_json | _____ | _____ | _____ |
| safe_write_json | _____ | _____ | _____ |
| copy_file | _____ | _____ | _____ |
| delete_file | _____ | _____ | _____ |

### Ошибки, эмулированные с monkeypatch

| Ошибка | Функция | Способ эмуляции |
|:---|:---|:---|
| PermissionError | safe_read | |
| PermissionError | safe_write | |
| OSError | safe_write | |

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться тестировать все типы ошибок ввода-вывода, создавать комплексные негативные сценарии и проверять состояние системы после ошибок.

Задания для сложного уровня
Задание 3.1. Комплексное тестирование safe_write (15 минут)
python
# test_file_handler_advanced.py

import pytest
import os
import json
from file_handler import FileHandler


class TestComplexWriteScenarios:
    """Комплексные тесты для safe_write."""
    
    @pytest.mark.parametrize("error_type,exception_class", [
        ("permission", PermissionError),
        ("disk_full", OSError),
        ("read_only", PermissionError),
    ])
    def test_write_various_errors(self, tmp_path, monkeypatch, error_type, exception_class):
        """Параметризованный тест различных ошибок записи."""
        file_path = tmp_path / "test.txt"
        original_open = open
        
        def mock_open(*args, **kwargs):
            if 'w' in args or kwargs.get('mode') == 'w':
                if error_type == "permission":
                    raise PermissionError("Access denied")
                elif error_type == "disk_full":
                    raise OSError("No space left on device")
                elif error_type == "read_only":
                    raise PermissionError("Read-only file system")
            return original_open(*args, **kwargs)
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        with pytest.raises(exception_class):
            FileHandler.safe_write(str(file_path), "content")
    
    def test_write_creates_backup_on_error(self, tmp_path, monkeypatch):
        """При ошибке файл не должен быть частично записан."""
        file_path = tmp_path / "important.txt"
        file_path.write_text("original")
        
        write_called = False
        
        def mock_open(*args, **kwargs):
            nonlocal write_called
            if 'w' in args or kwargs.get('mode') == 'w':
                write_called = True
                raise OSError("Write error mid-operation")
            return open(*args, **kwargs)
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        with pytest.raises(OSError):
            FileHandler.safe_write(str(file_path), "new content")
        
        # Файл не должен быть изменён
        assert file_path.read_text() == "original"
    
    def test_write_does_not_create_partial_file(self, tmp_path, monkeypatch):
        """При ошибке не должен создаваться пустой файл."""
        file_path = tmp_path / "new.txt"
        
        def mock_open(*args, **kwargs):
            raise PermissionError("Cannot write")
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        with pytest.raises(PermissionError):
            FileHandler.safe_write(str(file_path), "content")
        
        assert not file_path.exists()
Задание 3.2. Тестирование цепочек операций и отката (15 минут)
python
class TestTransactionOperations:
    """Тесты для операций, которые должны быть атомарными."""
    
    def test_atomic_write_read_cycle(self, tmp_path):
        """Цикл записи-чтения должен быть консистентным."""
        file_path = tmp_path / "data.txt"
        original_content = "original data"
        
        # Записываем оригинал
        FileHandler.safe_write(str(file_path), original_content)
        
        # Читаем — должно совпадать
        assert FileHandler.safe_read(str(file_path)) == original_content
        
        # Перезаписываем
        new_content = "new data"
        FileHandler.safe_write(str(file_path), new_content, overwrite=True)
        
        # Читаем обновлённое
        assert FileHandler.safe_read(str(file_path)) == new_content
    
    def test_copy_then_delete_original(self, tmp_path):
        """Копирование файла с последующим удалением оригинала."""
        src = tmp_path / "original.txt"
        dst = tmp_path / "copy.txt"
        
        src.write_text("important data")
        
        # Копируем
        FileHandler.copy_file(str(src), str(dst))
        
        # Удаляем оригинал
        FileHandler.delete_file(str(src))
        
        # Копия должна остаться
        assert not src.exists()
        assert dst.exists()
        assert dst.read_text() == "important data"
    
    def test_operation_sequence_rollback(self, tmp_path, monkeypatch):
        """При ошибке в середине цепочки операции должны откатываться."""
        file1 = tmp_path / "file1.txt"
        file2 = tmp_path / "file2.txt"
        
        file1.write_text("data1")
        
        call_count = 0
        
        def mock_copy(*args, **kwargs):
            nonlocal call_count
            call_count += 1
            if call_count == 2:  # Вторая операция проваливается
                raise PermissionError("Copy failed")
            return shutil.copy2(*args, **kwargs)
        
        monkeypatch.setattr(shutil, "copy2", mock_copy)
        
        # Пытаемся выполнить последовательность операций
        try:
            FileHandler.copy_file(str(file1), str(tmp_path / "temp.txt"))
            FileHandler.copy_file(str(file1), str(file2))  # Здесь ошибка
        except PermissionError:
            pass
        
        # Проверяем, что частичные изменения откатились
        # (в данном случае первая копия могла остаться — это зависит от логики)
        # Этот тест показывает, что нужно продумывать транзакционность
        assert (tmp_path / "temp.txt").exists()  # Первая операция выполнена
Задание 3.3. Тестирование append_to_file (10 минут)
python
class TestAppendToFile:
    """Тесты для добавления в файл."""
    
    def test_append_to_existing_file(self, tmp_path):
        """Добавление в существующий файл."""
        file_path = tmp_path / "log.txt"
        file_path.write_text("line1\n")
        
        result = FileHandler.append_to_file(str(file_path), "line2\n")
        
        assert result is True
        assert file_path.read_text() == "line1\nline2\n"
    
    def test_append_to_nonexistent_file(self, tmp_path):
        """Добавление в несуществующий файл -> FileNotFoundError."""
        file_path = tmp_path / "missing.txt"
        
        with pytest.raises(FileNotFoundError):
            FileHandler.append_to_file(str(file_path), "content")
    
    def test_append_no_permission(self, tmp_path, monkeypatch):
        """Нет прав на добавление."""
        file_path = tmp_path / "protected.txt"
        file_path.write_text("content")
        
        def mock_open(*args, **kwargs):
            if 'a' in args or kwargs.get('mode') == 'a':
                raise PermissionError("No append permission")
            return open(*args, **kwargs)
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        with pytest.raises(PermissionError):
            FileHandler.append_to_file(str(file_path), "new line")
### Задание 3.4. Комплексный сценарий с несколькими ошибками (10 минут)

```python
class TestComplexErrorScenarios:
    """Комплексные сценарии с несколькими ошибками."""
    
    def test_multiple_consecutive_errors(self, tmp_path, monkeypatch):
        """Обработка нескольких последовательных ошибок."""
        file_path = tmp_path / "test.txt"
        
        error_count = 0
        
        def mock_open(*args, **kwargs):
            nonlocal error_count
            error_count += 1
            if error_count <= 2:
                raise OSError("Temporary error")
            return open(*args, **kwargs)
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        # Первая попытка — ошибка
        with pytest.raises(OSError):
            FileHandler.safe_write(str(file_path), "content")
        
        # Вторая попытка — снова ошибка
        with pytest.raises(OSError):
            FileHandler.safe_write(str(file_path), "content")
        
        # Очищаем мок для третьей попытки
        monkeypatch.undo()
        
        # Третья попытка — успех
        result = FileHandler.safe_write(str(file_path), "success")
        assert result is True
        assert file_path.read_text() == "success"
    
    def test_error_messages_are_descriptive(self, tmp_path):
        """Сообщения об ошибках должны быть информативными."""
        file_path = tmp_path / "nonexistent_dir" / "file.txt"
        
        with pytest.raises(FileNotFoundError) as exc_info:
            FileHandler.safe_read(str(file_path))
        
        message = str(exc_info.value)
        assert "File not found" in message or str(file_path) in message
    
    def test_error_does_not_corrupt_state(self, tmp_path, monkeypatch):
        """Ошибка не должна портить состояние системы."""
        file_path = tmp_path / "state.txt"
        original_content = "original data"
        file_path.write_text(original_content)
        
        def mock_open(*args, **kwargs):
            raise PermissionError("Access denied")
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        with pytest.raises(PermissionError):
            FileHandler.safe_write(str(file_path), "new data")
        
        # Состояние файла не должно измениться
        assert file_path.read_text() == original_content
    
    def test_partial_write_does_not_occur(self, tmp_path, monkeypatch):
        """Частичная запись не должна происходить."""
        file_path = tmp_path / "partial.txt"
        
        write_occurred = False
        
        class PartialWriteMock:
            def __init__(self, *args, **kwargs):
                self.write_count = 0
            
            def __enter__(self):
                return self
            
            def __exit__(self, *args):
                pass
            
            def write(self, data):
                nonlocal write_occurred
                write_occurred = True
                raise OSError("Disk full after partial write")
            
            def close(self):
                pass
        
        def mock_open(*args, **kwargs):
            return PartialWriteMock()
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        with pytest.raises(OSError, match="Disk full"):
            FileHandler.safe_write(str(file_path), "large amount of data")
        
        # Файл либо не создан, либо содержит исходные данные
        # В зависимости от реализации
        if file_path.exists():
            # Если файл существует, он должен быть пустым или содержать исходные данные
            content = file_path.read_text()
            assert content != "large amount of data"
    
    def test_rollback_on_failure(self, tmp_path, monkeypatch):
        """Откат операций при ошибке."""
        file1 = tmp_path / "file1.txt"
        file2 = tmp_path / "file2.txt"
        
        file1.write_text("data1")
        
        call_count = 0
        
        def mock_copy(*args, **kwargs):
            nonlocal call_count
            call_count += 1
            if call_count == 2:
                raise PermissionError("Copy failed")
            return shutil.copy2(*args, **kwargs)
        
        monkeypatch.setattr(shutil, "copy2", mock_copy)
        
        # Пытаемся выполнить последовательность операций
        try:
            FileHandler.copy_file(str(file1), str(tmp_path / "temp.txt"))
            FileHandler.copy_file(str(file1), str(file2))
        except PermissionError:
            pass
        
        # Первая операция могла быть выполнена
        assert (tmp_path / "temp.txt").exists()
        # Вторая операция не выполнена
        assert not file2.exists()
    
    def test_concurrent_error_handling(self, tmp_path):
        """Обработка ошибок при конкурентном доступе (имитация)."""
        import threading
        
        file_path = tmp_path / "concurrent.txt"
        results = []
        errors = []
        
        def worker(worker_id):
            try:
                FileHandler.safe_write(str(file_path), f"Worker {worker_id}")
                results.append(worker_id)
            except Exception as e:
                errors.append((worker_id, str(e)))
        
        threads = []
        for i in range(10):
            t = threading.Thread(target=worker, args=(i,))
            threads.append(t)
            t.start()
        
        for t in threads:
            t.join()
        
        # Хотя бы один поток должен записать успешно
        assert len(results) >= 1
        
        # Ошибки должны быть корректно обработаны
        # PermissionError может возникнуть при конкурентной записи
        # (зависит от ОС и реализации)
    
    def test_error_recovery_with_retry(self, tmp_path, monkeypatch):
        """Восстановление после ошибки с повторной попыткой."""
        file_path = tmp_path / "recovery.txt"
        
        attempt = 0
        
        def mock_open_with_retry(*args, **kwargs):
            nonlocal attempt
            attempt += 1
            if attempt < 3:
                raise OSError(f"Temporary error (attempt {attempt})")
            return open(*args, **kwargs)
        
        monkeypatch.setattr("builtins.open", mock_open_with_retry)
        
        # Имитируем логику повторных попыток
        max_retries = 3
        last_error = None
        
        for retry in range(max_retries):
            try:
                result = FileHandler.safe_write(str(file_path), "success")
                assert result is True
                break
            except OSError as e:
                last_error = e
                continue
        
        if attempt < 3:
            assert False, "Должна была быть ошибка до успеха"
        else:
            assert file_path.exists()
            assert file_path.read_text() == "success"
    
    def test_error_scenario_combinations(self, tmp_path, monkeypatch):
        """Комбинация различных типов ошибок."""
        file_path = tmp_path / "combination.txt"
        
        # Список ошибок для последовательного воспроизведения
        errors = [
            FileNotFoundError,      # Файл не найден
            PermissionError,        # Нет прав
            OSError,                # Общая ошибка
            None                    # Успех
        ]
        
        error_index = 0
        
        def mock_open(*args, **kwargs):
            nonlocal error_index
            error = errors[error_index]
            error_index += 1
            
            if error is not None:
                raise error(f"Simulated {error.__name__}")
            
            return open(*args, **kwargs)
        
        monkeypatch.setattr("builtins.open", mock_open)
        
        # Первая попытка: FileNotFoundError
        with pytest.raises(FileNotFoundError):
            FileHandler.safe_write(str(file_path), "data")
        
        # Вторая попытка: PermissionError
        with pytest.raises(PermissionError):
            FileHandler.safe_write(str(file_path), "data")
        
        # Третья попытка: OSError
        with pytest.raises(OSError):
            FileHandler.safe_write(str(file_path), "data")
        
        # Четвёртая попытка: успех
        result = FileHandler.safe_write(str(file_path), "final")
        assert result is True
        assert file_path.read_text() == "final"
Задание 3.5. Итоговый отчёт (10 минут)
markdown
## Отчёт о тестировании обработки ошибок файлового ввода-вывода

**Студент:** _________________
**Вариант №:** ___ (18-25)
**Уровень:** Сложный

### Сводная таблица результатов

| Категория тестов | Количество | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| safe_read | _____ | _____ | _____ |
| safe_write | _____ | _____ | _____ |
| safe_read_json / safe_write_json | _____ | _____ | _____ |
| copy_file | _____ | _____ | _____ |
| delete_file | _____ | _____ | _____ |
| append_to_file | _____ | _____ | _____ |
| get_file_info | _____ | _____ | _____ |
| Комплексные сценарии | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Обработанные типы ошибок

| Тип ошибки | Проверено | Методы |
|:---|:---|:---|
| FileNotFoundError | ☐ | |
| PermissionError | ☐ | |
| FileExistsError | ☐ | |
| IsADirectoryError | ☐ | |
| OSError | ☐ | |
| json.JSONDecodeError | ☐ | |

### Использованные техники мокирования

| Техника | Где применялась |
|:---|:---|
| `monkeypatch.setattr("builtins.open")` | |
| `monkeypatch.setattr(os.path, "exists")` | |
| `monkeypatch.setattr(os, "access")` | |
| `monkeypatch.setattr(os, "makedirs")` | |
| `monkeypatch.setattr(shutil, "copy2")` | |

### Найденные дефекты

| ID | Описание | Функция | Серьёзность | Статус |
|:---|:---|:---|:---|:---|
| BUG-01 | | | | |
| BUG-02 | | | | |
| BUG-03 | | | | |

### Выводы

_______________________________________________________________

_______________________________________________________________

_______________________________________________________________

### Рекомендации

1. 
2. 
3. 

---

**Дата выполнения:** _____________
**Подпись студента:** _____________
**Подпись преподавателя:** _____________
Карточка студента (итоговая)
text
ПР 3.11. ТЕСТИРОВАНИЕ ОБРАБОТКИ ОШИБОК ФАЙЛОВОГО ВВОДА-ВЫВОДА

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ safe_read (базовый)
□ safe_write (базовый)
□ delete_file (базовый)
□ get_file_info (базовый)
□ PermissionError мокирование (средний)
□ json ошибки (средний)
□ copy_file (средний)
□ Комплексные сценарии ошибок (сложный)
□ Множественные последовательные ошибки (сложный)
□ Восстановление после ошибок (сложный)

=== ПРОВЕРЕННЫЕ ИСКЛЮЧЕНИЯ ===

□ FileNotFoundError
□ PermissionError
□ FileExistsError
□ OSError
□ json.JSONDecodeError

=== ИСПОЛЬЗОВАННЫЕ ТЕХНИКИ ===

□ pytest.raises
□ monkeypatch.setattr (builtins.open)
□ monkeypatch.setattr (os.path)
□ monkeypatch.setattr (os.access)
□ tmp_path фикстура

=== РЕЗУЛЬТАТЫ ===

Всего тестов: _____
Пройдено: _____
Не пройдено: _____

Найдено дефектов: _____

=== ВРЕМЯ ВЫПОЛНЕНИЯ ===

Подготовка: _____ мин
Выполнение: _____ мин
Оформление: _____ мин

=== ОТЧЁТ ===

Файлы:
- test_file_handler_base.py
- test_file_handler_intermediate.py
- test_file_handler_advanced.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки (полная шкала)
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Тесты не работают или не используют pytest.raises
3 (удовлетворительно)	Базовый	FileNotFoundError + базовые тесты (10+ тестов)
4 (хорошо)	Средний	+ PermissionError с monkeypatch + JSON + copy (20+ тестов)
5 (отлично)	Сложный	+ комплексные сценарии + множественные ошибки + восстановление (30+ тестов)
Следующее занятие: ПР 3.13 — Тестирование иерархий классов и полиморфизма.
