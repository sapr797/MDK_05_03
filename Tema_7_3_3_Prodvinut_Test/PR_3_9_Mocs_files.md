# Практическое занятие 3.9: Мокирование файловой системы, ошибок доступа и проверка вызовов (assert_called_with)

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться мокировать файловую систему и ошибки доступа с использованием `monkeypatch` и `pytest-mock`, проверять вызовы методов с помощью `assert_called_with` и `assert_called_once_with`.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Мокировать файловые операции (чтение, запись, удаление) (ПК 3.2, ПК 3.3).
2. Эмулировать ошибки доступа (`PermissionError`, `FileNotFoundError`, `OSError`) (ПК 3.2).
3. Проверять вызовы методов с помощью `assert_called_with` и `assert_called_once_with` (ПК 3.3).
4. Тестировать обработку ошибок файловой системы (ОК 02, ОК 05).

---

## Теоретическая справка

### Мокирование файловой системы

```python
# Базовое мокирование open
def mock_open(*args, **kwargs):
    raise PermissionError("Access denied")

monkeypatch.setattr("builtins.open", mock_open)

# Мокирование os.path.exists
def mock_exists(path):
    return path == "/valid/path"

monkeypatch.setattr(os.path, "exists", mock_exists)
Проверка вызовов
python
# Проверка одного вызова
mock.method.assert_called_once()
mock.method.assert_called_once_with(arg1, arg2)

# Проверка с частичным совпадением
mock.method.assert_called_with(arg1, arg2)

# Проверка последовательности вызовов
mock.method.assert_has_calls([call(1), call(2), call(3)])
Объект тестирования
Листинг 1. Модуль file_manager.py
python
# file_manager.py

import os
import json
import shutil
from pathlib import Path
from typing import List, Dict, Any, Optional
from datetime import datetime


class FileManager:
    """Менеджер файловых операций с обработкой ошибок."""
    
    def __init__(self, base_path: str = "."):
        self.base_path = Path(base_path)
        self._operation_log = []
    
    def _log_operation(self, operation: str, path: str, success: bool):
        """Логирует операцию."""
        self._operation_log.append({
            "operation": operation,
            "path": path,
            "success": success,
            "timestamp": datetime.now().isoformat()
        })
    
    def read_file(self, file_path: str) -> Optional[str]:
        """
        Читает содержимое файла.
        
        Returns:
            Содержимое файла или None при ошибке
        """
        full_path = self.base_path / file_path
        
        try:
            with open(full_path, 'r', encoding='utf-8') as f:
                content = f.read()
            self._log_operation("read", str(full_path), True)
            return content
        except FileNotFoundError:
            self._log_operation("read", str(full_path), False)
            return None
        except PermissionError:
            self._log_operation("read", str(full_path), False)
            return None
        except Exception as e:
            self._log_operation("read", str(full_path), False)
            raise IOError(f"Unexpected error reading file: {e}")
    
    def write_file(self, file_path: str, content: str, overwrite: bool = True) -> bool:
        """
        Записывает содержимое в файл.
        
        Returns:
            True при успешной записи
        """
        full_path = self.base_path / file_path
        
        # Проверка на существование
        if full_path.exists() and not overwrite:
            self._log_operation("write", str(full_path), False)
            return False
        
        try:
            # Создаём директории при необходимости
            full_path.parent.mkdir(parents=True, exist_ok=True)
            
            with open(full_path, 'w', encoding='utf-8') as f:
                f.write(content)
            
            self._log_operation("write", str(full_path), True)
            return True
        except PermissionError:
            self._log_operation("write", str(full_path), False)
            return False
        except OSError as e:
            self._log_operation("write", str(full_path), False)
            raise IOError(f"OS error during write: {e}")
    
    def delete_file(self, file_path: str, ignore_missing: bool = False) -> bool:
        """
        Удаляет файл.
        
        Returns:
            True если файл удалён или не существовал (с ignore_missing)
        """
        full_path = self.base_path / file_path
        
        if not full_path.exists():
            if ignore_missing:
                return False
            raise FileNotFoundError(f"File not found: {full_path}")
        
        try:
            full_path.unlink()
            self._log_operation("delete", str(full_path), True)
            return True
        except PermissionError:
            self._log_operation("delete", str(full_path), False)
            raise PermissionError(f"Cannot delete {full_path}: Permission denied")
    
    def copy_file(self, src: str, dst: str, overwrite: bool = False) -> bool:
        """
        Копирует файл.
        
        Returns:
            True при успешном копировании
        """
        src_path = self.base_path / src
        dst_path = self.base_path / dst
        
        if not src_path.exists():
            raise FileNotFoundError(f"Source file not found: {src_path}")
        
        if dst_path.exists() and not overwrite:
            return False
        
        try:
            dst_path.parent.mkdir(parents=True, exist_ok=True)
            shutil.copy2(src_path, dst_path)
            self._log_operation("copy", f"{src} -> {dst}", True)
            return True
        except PermissionError:
            self._log_operation("copy", f"{src} -> {dst}", False)
            raise PermissionError(f"Cannot copy: permission denied")
    
    def read_json(self, file_path: str) -> Optional[Dict]:
        """Читает JSON из файла."""
        content = self.read_file(file_path)
        if content is None:
            return None
        
        try:
            return json.loads(content)
        except json.JSONDecodeError as e:
            raise ValueError(f"Invalid JSON in {file_path}: {e}")
    
    def write_json(self, file_path: str, data: Dict, indent: int = 2) -> bool:
        """Записывает JSON в файл."""
        content = json.dumps(data, indent=indent, ensure_ascii=False)
        return self.write_file(file_path, content)
    
    def get_operation_log(self) -> List[Dict]:
        """Возвращает лог операций."""
        return self._operation_log.copy()
    
    def clear_log(self):
        """Очищает лог."""
        self._operation_log.clear()
    
    def file_exists(self, file_path: str) -> bool:
        """Проверяет существование файла."""
        return (self.base_path / file_path).exists()
    
    def get_file_size(self, file_path: str) -> int:
        """Возвращает размер файла в байтах."""
        full_path = self.base_path / file_path
        if not full_path.exists():
            raise FileNotFoundError(f"File not found: {full_path}")
        return full_path.stat().st_size
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Мокирование read/write, проверка вызовов
Средний (варианты 9-17)	Средние студенты	+ ошибки доступа + мокирование os.path
Сложный (варианты 18-25)	Сильные студенты	+ сложные сценарии + полный лог проверок
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться мокировать чтение и запись файлов, проверять вызовы методов.

Задания для базового уровня
Задание 1.1. Мокирование чтения файла (15 минут)
python
# test_file_manager_base.py

import pytest
import json
from pathlib import Path
from file_manager import FileManager


class TestFileManagerReadMock:
    """Тесты мокирования чтения файлов."""
    
    @pytest.fixture
    def file_manager(self):
        return FileManager(base_path="/test")
    
    # ==========================================================
    # МОКИРОВАНИЕ read_file
    # ==========================================================
    
    def test_mock_read_file_success(self, mocker, file_manager):
        """Мокирование успешного чтения файла."""
        mock_content = "Hello, World!"
        
        # Мокируем open
        mock_open = mocker.patch("builtins.open")
        mock_file = mocker.MagicMock()
        mock_file.read.return_value = mock_content
        mock_open.return_value.__enter__.return_value = mock_file
        
        result = file_manager.read_file("test.txt")
        
        assert result == mock_content
        mock_open.assert_called_once_with(Path("/test/test.txt"), 'r', encoding='utf-8')
    
    def test_mock_read_file_not_found(self, mocker, file_manager):
        """Мокирование FileNotFoundError."""
        mocker.patch("builtins.open", side_effect=FileNotFoundError("File not found"))
        
        result = file_manager.read_file("missing.txt")
        
        assert result is None
    
    def test_mock_read_file_permission_error(self, mocker, file_manager):
        """Мокирование PermissionError."""
        mocker.patch("builtins.open", side_effect=PermissionError("Access denied"))
        
        result = file_manager.read_file("protected.txt")
        
        assert result is None
    
    # ==========================================================
    # ПРОВЕРКА ВЫЗОВОВ
    # ==========================================================
    
    def test_assert_called_with(self, mocker, file_manager):
        """Проверка аргументов вызова."""
        mock_open = mocker.patch("builtins.open")
        mock_open.side_effect = FileNotFoundError()
        
        file_manager.read_file("data/config.txt")
        
        # Проверяем, что open был вызван с правильными аргументами
        expected_path = Path("/test/data/config.txt")
        mock_open.assert_called_once_with(expected_path, 'r', encoding='utf-8')
    
    def test_assert_called_with_partial(self, mocker, file_manager):
        """Проверка с частичным совпадением (assert_called_with)."""
        mock_log = mocker.patch.object(file_manager, '_log_operation')
        
        file_manager.read_file("test.txt")
        
        # Проверяем, что _log_operation был вызван с определёнными аргументами
        mock_log.assert_called_with("read", str(Path("/test/test.txt")), False)
    
    def test_assert_called_once(self, mocker, file_manager):
        """Проверка, что метод вызван ровно один раз."""
        mock_open = mocker.patch("builtins.open")
        mock_open.side_effect = FileNotFoundError()
        
        file_manager.read_file("test.txt")
        
        mock_open.assert_called_once()
        
        # Повторный вызов — снова один вызов? Нет, будет второй
        file_manager.read_file("test2.txt")
        assert mock_open.call_count == 2
Задание 1.2. Мокирование записи файла (15 минут)
python
class TestFileManagerWriteMock:
    """Тесты мокирования записи файлов."""
    
    @pytest.fixture
    def file_manager(self):
        return FileManager(base_path="/test")
    
    def test_mock_write_file_success(self, mocker, file_manager):
        """Мокирование успешной записи."""
        mock_open = mocker.patch("builtins.open")
        mock_file = mocker.MagicMock()
        mock_open.return_value.__enter__.return_value = mock_file
        
        # Мокируем создание директорий
        mocker.patch("pathlib.Path.mkdir")
        
        result = file_manager.write_file("output.txt", "content")
        
        assert result is True
        mock_open.assert_called_once_with(Path("/test/output.txt"), 'w', encoding='utf-8')
        mock_file.write.assert_called_once_with("content")
    
    def test_mock_write_file_permission_error(self, mocker, file_manager):
        """Мокирование PermissionError при записи."""
        mocker.patch("builtins.open", side_effect=PermissionError("Read-only"))
        mocker.patch("pathlib.Path.mkdir")
        
        result = file_manager.write_file("protected.txt", "content")
        
        assert result is False
    
    def test_mock_write_file_no_overwrite(self, mocker, file_manager):
        """Запись без перезаписи существующего файла."""
        # Мокируем, что файл существует
        mocker.patch("pathlib.Path.exists", return_value=True)
        
        result = file_manager.write_file("existing.txt", "content", overwrite=False)
        
        assert result is False
        # open не должен вызываться
        mocker.patch("builtins.open").assert_not_called()
    
    def test_assert_called_with_write(self, mocker, file_manager):
        """Проверка вызова write с правильным содержимым."""
        mock_open = mocker.patch("builtins.open")
        mock_file = mocker.MagicMock()
        mock_open.return_value.__enter__.return_value = mock_file
        mocker.patch("pathlib.Path.mkdir")
        
        file_manager.write_file("data.txt", "Hello, pytest!")
        
        mock_file.write.assert_called_once_with("Hello, pytest!")
    
    def test_assert_multiple_calls(self, mocker, file_manager):
        """Проверка последовательности вызовов."""
        mock_log = mocker.patch.object(file_manager, '_log_operation')
        mocker.patch("builtins.open", side_effect=FileNotFoundError())
        
        file_manager.read_file("a.txt")
        file_manager.read_file("b.txt")
        file_manager.read_file("c.txt")
        
        # Проверяем, что _log_operation был вызван 3 раза
        assert mock_log.call_count == 3
        
        # Проверяем последовательность вызовов
        expected_calls = [
            mocker.call("read", str(Path("/test/a.txt")), False),
            mocker.call("read", str(Path("/test/b.txt")), False),
            mocker.call("read", str(Path("/test/c.txt")), False),
        ]
        mock_log.assert_has_calls(expected_calls)
Задание 1.3. Мокирование JSON операций (10 минут)
python
class TestFileManagerJsonMock:
    """Тесты мокирования JSON операций."""
    
    @pytest.fixture
    def file_manager(self):
        return FileManager(base_path="/test")
    
    def test_mock_read_json(self, mocker, file_manager):
        """Мокирование чтения JSON."""
        mock_data = {"name": "test", "value": 42}
        
        # Мокируем read_file
        mocker.patch.object(file_manager, 'read_file', return_value=json.dumps(mock_data))
        
        result = file_manager.read_json("data.json")
        
        assert result == mock_data
    
    def test_mock_read_json_invalid(self, mocker, file_manager):
        """Мокирование невалидного JSON."""
        mocker.patch.object(file_manager, 'read_file', return_value="{invalid json}")
        
        with pytest.raises(ValueError, match="Invalid JSON"):
            file_manager.read_json("bad.json")
    
    def test_mock_write_json(self, mocker, file_manager):
        """Мокирование записи JSON."""
        mock_write = mocker.patch.object(file_manager, 'write_file', return_value=True)
        
        data = {"key": "value"}
        result = file_manager.write_json("data.json", data)
        
        assert result is True
        mock_write.assert_called_once()
        # Проверяем, что переданный JSON содержит нужные данные
        called_content = mock_write.call_args[0][1]
        assert '"key": "value"' in called_content
Задание 1.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Результаты тестирования

| Функция | Мокирование | Проверка вызовов | Статус |
|:---|:---|:---|:---|
| read_file | ☐ | ☐ | ☐ |
| write_file | ☐ | ☐ | ☐ |
| read_json | ☐ | ☐ | ☐ |
| write_json | ☐ | ☐ | ☐ |

### Проверки assert_called

| Проверка | Использована |
|:---|:---|
| assert_called_once | ☐ |
| assert_called_once_with | ☐ |
| assert_called_with | ☐ |
| assert_has_calls | ☐ |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться мокировать ошибки доступа и системные вызовы, проверять сложные сценарии.

Задания для среднего уровня
Задание 2.1. Мокирование os.path и ошибок доступа (15 минут)
python
# test_file_manager_intermediate.py

import pytest
import os
from pathlib import Path
from file_manager import FileManager


class TestFileManagerPermissionErrors:
    """Тесты ошибок доступа."""
    
    @pytest.fixture
    def file_manager(self):
        return FileManager(base_path="/test")
    
    def test_mock_permission_error_on_read(self, mocker, file_manager):
        """PermissionError при чтении."""
        mocker.patch("builtins.open", side_effect=PermissionError("Access denied"))
        
        result = file_manager.read_file("secret.txt")
        
        assert result is None
    
    def test_mock_permission_error_on_write(self, mocker, file_manager):
        """PermissionError при записи."""
        mocker.patch("builtins.open", side_effect=PermissionError("Read-only"))
        mocker.patch("pathlib.Path.mkdir")
        
        result = file_manager.write_file("protected.txt", "content")
        
        assert result is False
    
    def test_mock_permission_error_on_delete(self, mocker, file_manager):
        """PermissionError при удалении."""
        mocker.patch("pathlib.Path.exists", return_value=True)
        mocker.patch("pathlib.Path.unlink", side_effect=PermissionError("Cannot delete"))
        
        with pytest.raises(PermissionError, match="Permission denied"):
            file_manager.delete_file("locked.txt")
    
    def test_mock_os_path_exists(self, mocker, file_manager):
        """Мокирование os.path.exists."""
        # Мокируем Path.exists
        mock_exists = mocker.patch("pathlib.Path.exists")
        mock_exists.return_value = False
        
        result = file_manager.file_exists("missing.txt")
        
        assert result is False
        mock_exists.assert_called_once()
    
    def test_mock_os_path_exists_for_overwrite_check(self, mocker, file_manager):
        """Мокирование exists для проверки перезаписи."""
        mock_exists = mocker.patch("pathlib.Path.exists")
        mock_exists.return_value = True  # Файл существует
        
        result = file_manager.write_file("existing.txt", "new", overwrite=False)
        
        assert result is False
        # open не должен вызываться
        mocker.patch("builtins.open").assert_not_called()
Задание 2.2. Мокирование mkdir и сложных операций (15 минут)
python
class TestFileManagerMkdir:
    """Тесты мокирования создания директорий."""
    
    @pytest.fixture
    def file_manager(self):
        return FileManager(base_path="/test")
    
    def test_mock_mkdir_success(self, mocker, file_manager):
        """Успешное создание директории."""
        mock_mkdir = mocker.patch("pathlib.Path.mkdir")
        mock_open = mocker.patch("builtins.open")
        mock_file = mocker.MagicMock()
        mock_open.return_value.__enter__.return_value = mock_file
        
        result = file_manager.write_file("deep/nested/path/file.txt", "content")
        
        assert result is True
        mock_mkdir.assert_called_once_with(parents=True, exist_ok=True)
    
    def test_mock_mkdir_permission_error(self, mocker, file_manager):
        """Ошибка при создании директории."""
        mocker.patch("pathlib.Path.mkdir", side_effect=PermissionError("Cannot create directory"))
        mocker.patch("builtins.open")  # не должен быть вызван
        
        result = file_manager.write_file("new/dir/file.txt", "content")
        
        assert result is False
    
    def test_mock_copy_file(self, mocker, file_manager):
        """Мокирование копирования файла."""
        mock_copy2 = mocker.patch("shutil.copy2")
        mocker.patch("pathlib.Path.exists", return_value=True)
        mocker.patch("pathlib.Path.mkdir")
        
        result = file_manager.copy_file("src.txt", "dst.txt")
        
        assert result is True
        mock_copy2.assert_called_once()
    
    def test_mock_copy_file_source_not_exists(self, mocker, file_manager):
        """Копирование несуществующего файла."""
        mocker.patch("pathlib.Path.exists", return_value=False)
        
        with pytest.raises(FileNotFoundError, match="Source file not found"):
            file_manager.copy_file("missing.txt", "dst.txt")
    
    def test_mock_copy_file_permission_error(self, mocker, file_manager):
        """PermissionError при копировании."""
        mocker.patch("pathlib.Path.exists", return_value=True)
        mocker.patch("shutil.copy2", side_effect=PermissionError("Access denied"))
        
        with pytest.raises(PermissionError, match="Cannot copy"):
            file_manager.copy_file("src.txt", "dst.txt")
Задание 2.3. Проверка вызовов с assert_called_with (10 минут)
python
class TestAssertCalledWith:
    """Продвинутые проверки вызовов."""
    
    @pytest.fixture
    def file_manager(self):
        return FileManager(base_path="/test")
    
    def test_assert_called_with_exact_args(self, mocker, file_manager):
        """Проверка точных аргументов вызова."""
        mock_log = mocker.patch.object(file_manager, '_log_operation')
        mocker.patch("builtins.open", side_effect=FileNotFoundError())
        
        file_manager.read_file("logs/app.log")
        
        expected_path = str(Path("/test/logs/app.log"))
        mock_log.assert_called_once_with("read", expected_path, False)
    
    def test_assert_called_with_any_args(self, mocker, file_manager):
        """Использование ANY для игнорирования части аргументов."""
        from unittest.mock import ANY
        
        mock_log = mocker.patch.object(file_manager, '_log_operation')
        mocker.patch("builtins.open", side_effect=FileNotFoundError())
        
        file_manager.read_file("test.txt")
        
        # ANY означает "любое значение"
        mock_log.assert_called_once_with("read", ANY, False)
    
    def test_assert_has_calls_order(self, mocker, file_manager):
        """Проверка последовательности вызовов."""
        from unittest.mock import call
        
        mock_log = mocker.patch.object(file_manager, '_log_operation')
        mocker.patch("builtins.open", side_effect=FileNotFoundError())
        
        file_manager.read_file("a.txt")
        file_manager.read_file("b.txt")
        file_manager.read_file("c.txt")
        
        expected_calls = [
            call("read", str(Path("/test/a.txt")), False),
            call("read", str(Path("/test/b.txt")), False),
            call("read", str(Path("/test/c.txt")), False),
        ]
        mock_log.assert_has_calls(expected_calls)
    
    def test_assert_not_called(self, mocker, file_manager):
        """Проверка, что метод не был вызван."""
        mock_write = mocker.patch.object(file_manager, 'write_file')
        mock_open = mocker.patch("builtins.open")
        
        # Не вызываем write_file
        mock_write.assert_not_called()
    
    def test_assert_call_count(self, mocker, file_manager):
        """Проверка количества вызовов."""
        mock_log = mocker.patch.object(file_manager, '_log_operation')
        mocker.patch("builtins.open", side_effect=FileNotFoundError())
        
        file_manager.read_file("f1.txt")
        file_manager.read_file("f2.txt")
        file_manager.read_file("f3.txt")
        
        assert mock_log.call_count == 3
Задание 2.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Мокированные ошибки

| Ошибка | read_file | write_file | delete_file | copy_file |
|:---|:---|:---|:---|:---|
| FileNotFoundError | ☐ | ☐ | ☐ | ☐ |
| PermissionError | ☐ | ☐ | ☐ | ☐ |
| OSError | ☐ | ☐ | ☐ | ☐ |

### Проверки вызовов

| Метод проверки | Использован |
|:---|:---|
| assert_called_once_with | ☐ |
| assert_called_with | ☐ |
| assert_has_calls | ☐ |
| assert_not_called | ☐ |

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться тестировать сложные сценарии с цепочками операций и полной проверкой лога.

Задания для сложного уровня
Задание 3.1. Комплексные сценарии с моками (15 минут)
python
# test_file_manager_advanced.py

import pytest
import json
from unittest.mock import call, ANY
from file_manager import FileManager


class TestComplexScenarios:
    """Комплексные сценарии тестирования."""
    
    @pytest.fixture
    def file_manager(self):
        return FileManager(base_path="/test")
    
    def test_full_file_lifecycle_with_mocks(self, mocker, file_manager):
        """
        Полный жизненный цикл файла с моками.
        """
        # Моки для записи
        mock_open = mocker.patch("builtins.open")
        mock_file = mocker.MagicMock()
        mock_open.return_value.__enter__.return_value = mock_file
        mocker.patch("pathlib.Path.mkdir")
        mocker.patch("pathlib.Path.exists", return_value=False)
        
        # 1. Запись файла
        result1 = file_manager.write_file("data.txt", "initial content")
        assert result1 is True
        
        # 2. Проверка, что запись была с правильным содержимым
        mock_file.write.assert_called_with("initial content")
        
        # 3. Моки для чтения
        mock_open.reset_mock()
        mock_file.read.return_value = "initial content"
        
        # 4. Чтение файла
        content = file_manager.read_file("data.txt")
        assert content == "initial content"
        
        # 5. Обновление файла
        mock_file.write.reset_mock()
        file_manager.write_file("data.txt", "updated content")
        mock_file.write.assert_called_with("updated content")
    
    def test_error_recovery_scenario(self, mocker, file_manager):
        """
        Сценарий: первая попытка записи失败, вторая успешна.
        """
        mock_open = mocker.patch("builtins.open")
        mocker.patch("pathlib.Path.mkdir")
        mocker.patch("pathlib.Path.exists", return_value=False)
        
        # Первая попытка: PermissionError
        mock_open.side_effect = PermissionError("Read-only")
        
        result1 = file_manager.write_file("file.txt", "content")
        assert result1 is False
        
        # Вторая попытка: успех
        mock_open.side_effect = None
        mock_file = mocker.MagicMock()
        mock_open.return_value.__enter__.return_value = mock_file
        
        result2 = file_manager.write_file("file.txt", "content")
        assert result2 is True
    
    def test_batch_operations_log_verification(self, mocker, file_manager):
        """
        Проверка лога операций при пакетной обработке.
        """
        mock_log = mocker.patch.object(file_manager, '_log_operation')
        
        # Мокируем операции
        mocker.patch("builtins.open", side_effect=[
            FileNotFoundError(),  # первое чтение
            PermissionError(),    # второе чтение
        ])
        mocker.patch("pathlib.Path.mkdir")
        mocker.patch("pathlib.Path.exists", return_value=True)
        
        # Выполняем операции
        file_manager.read_file("missing.txt")
        file_manager.read_file("protected.txt")
        file_manager.write_file("output.txt", "data")
        file_manager.delete_file("to_delete.txt")
        
        # Проверяем, что все операции залогированы
        assert mock_log.call_count == 4
        
        # Проверяем, что для чтения с ошибкой залогировано failure
        read_calls = [c for c in mock_log.call_args_list if c[0][0] == "read"]
        assert len(read_calls) == 2
        for call in read_calls:
            assert call[0][2] is False  # success = False
Задание 3.2. Тестирование логирования операций (10 минут)
python
class TestOperationLogging:
    """Тестирование логирования операций."""
    
    @pytest.fixture
    def file_manager(self):
        return FileManager(base_path="/test")
    
    def test_log_contains_operation_details(self, mocker, file_manager):
        """Проверка, что лог содержит детали операции."""
        mock_log = mocker.patch.object(file_manager, '_log_operation')
        mocker.patch("builtins.open", side_effect=FileNotFoundError())
        
        file_manager.read_file("missing.txt")
        
        mock_log.assert_called_once()
        args = mock_log.call_args[0]
        assert args[0] == "read"  # operation
        assert "missing.txt" in args[1]  # path
        assert args[2] is False  # success
    
    def test_log_on_successful_operation(self, mocker, file_manager):
        """Логирование успешной операции."""
        mock_log = mocker.patch.object(file_manager, '_log_operation')
        mock_open = mocker.patch("builtins.open")
        mock_file = mocker.MagicMock()
        mock_open.return_value.__enter__.return_value = mock_file
        mocker.patch("pathlib.Path.mkdir")
        
        file_manager.write_file("output.txt", "data")
        
        mock_log.assert_called_once()
        args = mock_log.call_args[0]
        assert args[2] is True  # success = True
    
    def test_clear_log(self, file_manager):
        """Очистка лога."""
        # Добавляем запись в лог (через прямую манипуляцию для теста)
        file_manager._operation_log.append({"test": "data"})
        assert len(file_manager.get_operation_log()) == 1
        
        file_manager.clear_log()
        assert len(file_manager.get_operation_log()) == 0
Задание 3.3. Тестирование get_file_size (10 минут)
python
class TestFileSize:
    """Тесты получения размера файла."""
    
    @pytest.fixture
    def file_manager(self):
        return FileManager(base_path="/test")
    
    def test_get_file_size_success(self, mocker, file_manager):
        """Успешное получение размера файла."""
        mock_stat = mocker.MagicMock()
        mock_stat.st_size = 1024
        
        mock_path = mocker.patch("pathlib.Path")
        mock_path.return_value.exists.return_value = True
        mock_path.return_value.stat.return_value = mock_stat
        
        size = file_manager.get_file_size("data.txt")
        
        assert size == 1024
    
    def test_get_file_size_not_found(self, mocker, file_manager):
        """Файл не найден."""
        mocker.patch("pathlib.Path.exists", return_value=False)
        
        with pytest.raises(FileNotFoundError):
            file_manager.get_file_size("missing.txt")
    
    def test_get_file_size_permission_error(self, mocker, file_manager):
        """Ошибка доступа при получении размера."""
        mock_path = mocker.patch("pathlib.Path")
        mock_path.return_value.exists.return_value = True
        mock_path.return_value.stat.side_effect = PermissionError("Access denied")
        
        with pytest.raises(PermissionError):
            file_manager.get_file_size("protected.txt")
Задание 3.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (сложный уровень)

**Студент:** _________________
**Вариант №:** ___ (18-25)

### Комплексные результаты

| Сценарий | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| Чтение файлов | _____ | _____ | _____ |
| Запись файлов | _____ | _____ | _____ |
| Удаление файлов | _____ | _____ | _____ |
| Копирование файлов | _____ | _____ | _____ |
| JSON операции | _____ | _____ | _____ |
| Ошибки доступа | _____ | _____ | _____ |
| Логирование | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Мокированные компоненты

| Компонент | Использован |
|:---|:---|
| builtins.open | ☐ |
| pathlib.Path.exists | ☐ |
| pathlib.Path.mkdir | ☐ |
| pathlib.Path.unlink | ☐ |
| shutil.copy2 | ☐ |
| os.path | ☐ |

### Выводы

_______________________________________________________________

_______________________________________________________________
Карточка студента
text
ПР 3.9. МОКИРОВАНИЕ ФАЙЛОВОЙ СИСТЕМЫ

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ Мокирование read_file (базовый)
□ Мокирование write_file (базовый)
□ Мокирование JSON (базовый)
□ assert_called_with (базовый)
□ Ошибки доступа (средний)
□ Мокирование mkdir (средний)
□ assert_has_calls (средний)
□ Комплексные сценарии (сложный)
□ Логирование (сложный)
□ get_file_size (сложный)

=== ПРОВЕРЕННЫЕ АССЕРТЫ ===

□ assert_called_once()
□ assert_called_once_with()
□ assert_called_with()
□ assert_has_calls()
□ assert_not_called()

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
- test_file_manager_base.py
- test_file_manager_intermediate.py
- test_file_manager_advanced.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Мокирование не работает
3 (удовлетворительно)	Базовый	read/write моки + assert_called (10+ тестов)
4 (хорошо)	Средний	+ Ошибки доступа + сложные моки (20+ тестов)
5 (отлично)	Сложный	+ Комплексные сценарии + полный лог (30+ тестов)
Контрольные вопросы (для защиты)
Как мокировать open с помощью pytest-mock?

Как проверить, что open был вызван с правильными аргументами?

Как эмулировать PermissionError при чтении файла?

Как проверить, что метод был вызван ровно один раз с определёнными аргументами?

Как проверить последовательность вызовов метода?

Как мокировать os.path.exists для тестирования проверки существования файла?

Как протестировать, что при ошибке записи файл не создаётся?

Как проверить, что mkdir был вызван с parents=True?

Как протестировать сквозной сценарий с несколькими операциями?

Как проверить, что после серии операций лог содержит все записи?

Следующее занятие: ПР 3.10 — Анализ покрытия кода и увеличение метрик.

text

