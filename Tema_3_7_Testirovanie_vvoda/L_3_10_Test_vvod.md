# Лекция 3.10: Тестирование файлового ввода-вывода

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.7:** Тестирование файлового ввода-вывода  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить методы тестирования операций файлового ввода-вывода, освоить использование фикстуры `tmp_path` для изоляции тестов, научиться подменять стандартный ввод/вывод и тестировать обработку ошибок.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Тестировать файловые операции с использованием временных файлов (ПК 3.2, ПК 3.3).
2. Использовать `tmp_path` для изоляции тестов (ПК 3.2).
3. Подменять стандартный ввод/вывод с помощью `monkeypatch` (ПК 3.2).
4. Тестировать обработку ошибок ввода-вывода (ПК 3.3).

---

## 1. Проблемы тестирования файловых операций

### 1.1 Типичные проблемы

| Проблема | Описание | Риск |
|:---|:---|:---|
| **Загрязнение файловой системы** | Тесты создают файлы в реальной системе | Остаются после тестов |
| **Зависимость от окружения** | Тест использует реальные пути (например, `/home/user/data.txt`) | Не работает на CI |
| **Конфликт между тестами** | Один тест создаёт файл, другой его удаляет | Нестабильные результаты |
| **Права доступа** | Тест требует прав на запись | Ошибки на CI/CD |
| **Параллельный запуск** | Несколько тестов работают с одним файлом | Гонки |

### 1.2 Принципы тестирования файловых операций
┌─────────────────────────────────────────────────────────────────────────────┐
│ ПРИНЦИПЫ ТЕСТИРОВАНИЯ ФАЙЛОВОГО ВВОДА-ВЫВОДА │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ 1. ИЗОЛЯЦИЯ │
│ ├── Каждый тест использует свои временные файлы │
│ └── Файлы автоматически удаляются после теста │
│ │
│ 2. НЕЗАВИСИМОСТЬ ОТ ОКРУЖЕНИЯ │
│ ├── Не использовать абсолютные пути │
│ └── Использовать временные директории │
│ │
│ 3. ТЕСТИРОВАНИЕ ОШИБОК │
│ ├── Недостаточно прав │
│ ├── Диск заполнен │
│ └── Файл уже существует / не существует │
│ │
│ 4. ПОДМЕНА ВВОДА/ВЫВОДА │
│ ├── Подмена stdin для тестирования ввода │
│ └── Перехват stdout/stderr для проверки вывода │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

---

## 2. Фикстура tmp_path для временных файлов

### 2.1 Что такое tmp_path?

`tmp_path` — встроенная фикстура pytest, которая создаёт **уникальную временную директорию** для каждого теста и **автоматически удаляет** её после завершения.

### 2.2 Базовый синтаксис

```python
def test_with_tmp_path(tmp_path):
    """
    tmp_path — объект pathlib.Path.
    Директория уникальна для каждого теста.
    """
    # Создание файла
    file_path = tmp_path / "data.txt"
    file_path.write_text("Hello, World!")
    
    # Проверка
    assert file_path.exists()
    assert file_path.read_text() == "Hello, World!"
    
    # После теста директория удаляется автоматически
2.3 Создание вложенных структур
python
def test_nested_directories(tmp_path):
    """Создание сложной структуры каталогов."""
    # Создаём вложенные директории
    data_dir = tmp_path / "app" / "data"
    data_dir.mkdir(parents=True)
    
    # Создаём несколько файлов
    (data_dir / "config.json").write_text('{"mode": "test"}')
    (data_dir / "log.txt").write_text("Log entry")
    
    # Проверяем
    assert (tmp_path / "app" / "data" / "config.json").exists()
    assert (tmp_path / "app" / "data" / "log.txt").exists()
    
    # Вся структура удалится после теста
2.4 Пример: тестирование чтения и записи файлов
python
# file_processor.py
class FileProcessor:
    @staticmethod
    def write_file(path, content):
        with open(path, 'w') as f:
            f.write(content)
        return True
    
    @staticmethod
    def read_file(path):
        with open(path, 'r') as f:
            return f.read()
    
    @staticmethod
    def append_to_file(path, content):
        with open(path, 'a') as f:
            f.write(content)

# test_file_processor.py
def test_write_and_read(tmp_path):
    """Тест записи и чтения файла."""
    processor = FileProcessor()
    file_path = tmp_path / "test.txt"
    
    processor.write_file(file_path, "content")
    result = processor.read_file(file_path)
    
    assert result == "content"

def test_append_to_file(tmp_path):
    """Тест добавления в файл."""
    processor = FileProcessor()
    file_path = tmp_path / "log.txt"
    
    processor.write_file(file_path, "line1\n")
    processor.append_to_file(file_path, "line2\n")
    
    content = processor.read_file(file_path)
    assert content == "line1\nline2\n"
3. Подмена стандартного ввода/вывода
3.1 Подмена stdin (ввод с клавиатуры)
python
from io import StringIO

def test_user_input_with_monkeypatch(monkeypatch):
    """Подмена стандартного ввода для тестирования input()."""
    
    # Функция, которая читает ввод пользователя
    def greet():
        name = input("Enter your name: ")
        return f"Hello, {name}!"
    
    # Подменяем stdin
    monkeypatch.setattr('sys.stdin', StringIO("Alice\n"))
    
    result = greet()
    assert result == "Hello, Alice!"
3.2 Подмена stdout (перехват вывода)
python
def test_capture_output(capsys):
    """Перехват вывода в stdout."""
    print("Hello, World!")
    
    captured = capsys.readouterr()
    assert captured.out == "Hello, World!\n"

def test_redirect_stdout(monkeypatch):
    """Перенаправление stdout в StringIO."""
    from io import StringIO
    
    fake_stdout = StringIO()
    monkeypatch.setattr('sys.stdout', fake_stdout)
    
    print("Hidden message")
    
    assert fake_stdout.getvalue() == "Hidden message\n"
3.3 Комплексный пример: интерактивная программа
python
# interactive.py
def ask_question(question, valid_answers):
    """Задаёт вопрос и проверяет ответ."""
    while True:
        answer = input(question + " ")
        if answer in valid_answers:
            return answer
        print("Invalid answer. Try again.")

# test_interactive.py
def test_ask_question_with_valid_answer(monkeypatch, capsys):
    """Тест с валидным ответом."""
    monkeypatch.setattr('sys.stdin', StringIO("yes\n"))
    
    result = ask_question("Continue?", ["yes", "no"])
    
    assert result == "yes"

def test_ask_question_with_invalid_then_valid(monkeypatch, capsys):
    """Тест с неверным, затем верным ответом."""
    monkeypatch.setattr('sys.stdin', StringIO("maybe\nyes\n"))
    
    result = ask_question("Continue?", ["yes", "no"])
    
    captured = capsys.readouterr()
    assert "Invalid answer" in captured.out
    assert result == "yes"
4. Тестирование ошибок ввода-вывода
4.1 Эмуляция ошибок доступа
python
def test_permission_error(monkeypatch, tmp_path):
    """Тестирование ошибки доступа к файлу."""
    
    def mock_open(*args, **kwargs):
        raise PermissionError("Access denied")
    
    monkeypatch.setattr("builtins.open", mock_open)
    
    processor = FileProcessor()
    file_path = tmp_path / "test.txt"
    
    with pytest.raises(PermissionError, match="Access denied"):
        processor.write_file(file_path, "content")
4.2 Эмуляция заполненного диска
python
def test_disk_full_error(monkeypatch, tmp_path):
    """Тестирование ошибки при заполненном диске."""
    
    class DiskFullError(OSError):
        def __init__(self):
            super().__init__("No space left on device")
    
    def mock_open(*args, **kwargs):
        raise DiskFullError()
    
    monkeypatch.setattr("builtins.open", mock_open)
    
    processor = FileProcessor()
    file_path = tmp_path / "test.txt"
    
    with pytest.raises(OSError, match="No space left on device"):
        processor.write_file(file_path, "large content")
4.3 Эмуляция отсутствия файла
python
def test_file_not_found(tmp_path):
    """Тестирование чтения несуществующего файла."""
    processor = FileProcessor()
    nonexistent = tmp_path / "missing.txt"
    
    with pytest.raises(FileNotFoundError):
        processor.read_file(nonexistent)
4.4 Эмуляция битого файла
python
def test_corrupted_json_file(tmp_path):
    """Тестирование обработки битого JSON."""
    
    # Создаём битый файл
    file_path = tmp_path / "data.json"
    file_path.write_text('{"invalid": json}')  # Некорректный JSON
    
    processor = FileProcessor()
    
    with pytest.raises(json.JSONDecodeError):
        processor.read_json(file_path)
5. Фикстуры для подготовки файлов
5.1 Фикстура с предсозданным файлом
python
@pytest.fixture
def data_file(tmp_path):
    """Фикстура, создающая файл с тестовыми данными."""
    file_path = tmp_path / "data.txt"
    file_path.write_text("line1\nline2\nline3\n")
    return file_path

def test_read_data_file(data_file):
    """Использование подготовленного файла."""
    processor = FileProcessor()
    content = processor.read_file(data_file)
    assert content == "line1\nline2\nline3\n"
5.2 Фикстура с несколькими файлами
python
@pytest.fixture
def test_environment(tmp_path):
    """Создание сложной файловой структуры."""
    # Структура:
    # tmp_path/
    #   config/
    #     settings.json
    #   data/
    #     input.txt
    #     output/
    config_dir = tmp_path / "config"
    data_dir = tmp_path / "data"
    output_dir = data_dir / "output"
    
    config_dir.mkdir()
    data_dir.mkdir()
    output_dir.mkdir()
    
    (config_dir / "settings.json").write_text('{"debug": true}')
    (data_dir / "input.txt").write_text("test data")
    
    yield {
        "root": tmp_path,
        "config": config_dir,
        "data": data_dir,
        "output": output_dir,
        "settings": config_dir / "settings.json",
        "input": data_dir / "input.txt"
    }

def test_with_environment(test_environment):
    """Тест с подготовленной файловой структурой."""
    assert test_environment["settings"].exists()
    assert test_environment["input"].read_text() == "test data"
6. Тестирование больших файлов и производительности
6.1 Тестирование обработки больших файлов
python
def test_large_file_processing(tmp_path):
    """Тестирование обработки большого файла."""
    file_path = tmp_path / "large.txt"
    
    # Создаём большой файл (10000 строк)
    content = "\n".join(f"Line {i}" for i in range(10000))
    file_path.write_text(content)
    
    processor = FileProcessor()
    
    # Тестируем чтение
    start = time.time()
    result = processor.read_file(file_path)
    elapsed = time.time() - start
    
    assert len(result.splitlines()) == 10000
    assert elapsed < 1.0, f"Slow: {elapsed}s"
6.2 Тестирование буферизации
python
def test_buffered_writes(tmp_path):
    """Тестирование записи с буферизацией."""
    file_path = tmp_path / "output.txt"
    
    with open(file_path, 'w', buffering=1024) as f:
        for i in range(1000):
            f.write(f"Line {i}\n")
    
    # Проверяем, что файл создан
    assert file_path.exists()
    assert file_path.stat().st_size > 0
7. Практические рекомендации
7.1 Что следует делать
Рекомендация	Пример
Всегда использовать tmp_path	file = tmp_path / "test.txt"
Тестировать ошибки	pytest.raises(FileNotFoundError)
Проверять содержимое файлов	assert file.read_text() == expected
Очищать ресурсы	Использовать yield в фикстурах
7.2 Чего следует избегать
Антипаттерн	Почему плохо
open("/tmp/test.txt", "w")	Остаётся после тестов
os.remove() после теста	Не выполнится при исключении
Реальные абсолютные пути	Не работают на CI
Общие файлы для нескольких тестов	Конфликты при параллельном запуске
8. Шпаргалка
python
# === ВРЕМЕННАЯ ДИРЕКТОРИЯ ===
def test_example(tmp_path):
    file = tmp_path / "data.txt"
    file.write_text("content")
    assert file.read_text() == "content"

# === ПОДМЕНА СТАНДАРТНОГО ВВОДА ===
def test_input(monkeypatch):
    monkeypatch.setattr('sys.stdin', StringIO("Alice\n"))
    name = input("Name: ")
    assert name == "Alice"

# === ПЕРЕХВАТ ВЫВОДА ===
def test_output(capsys):
    print("Hello")
    captured = capsys.readouterr()
    assert captured.out == "Hello\n"

# === ТЕСТИРОВАНИЕ ОШИБОК ===
def test_file_not_found(tmp_path):
    with pytest.raises(FileNotFoundError):
        open(tmp_path / "missing.txt")

# === ФИКСТУРА С ФАЙЛОМ ===
@pytest.fixture
def prepared_file(tmp_path):
    file = tmp_path / "prepared.txt"
    file.write_text("initial")
    return file

def test_with_prepared_file(prepared_file):
    assert prepared_file.read_text() == "initial"
Контрольные вопросы
Почему не следует создавать файлы напрямую в /tmp?

Как tmp_path обеспечивает изоляцию между тестами?

Как подменить input() для тестирования интерактивных программ?

Чем отличается capsys от monkeypatch при подмене stdout?

Как протестировать обработку ошибки PermissionError?

Как создать фикстуру с несколькими файлами?

Как проверить, что файл был создан с правильным содержимым?

Как тестировать обработку очень больших файлов?

Почему tmp_path автоматически удаляет файлы, а ручное создание — нет?

Как эмулировать ошибку «диск заполнен» при записи файла?

