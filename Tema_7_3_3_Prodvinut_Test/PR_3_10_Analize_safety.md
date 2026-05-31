# Практическое занятие 3.10: Анализ безопасности кода с bandit (поиск уязвимостей)

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться использовать инструмент `bandit` для статического анализа безопасности кода на Python, выявлять типовые уязвимости, интерпретировать результаты и устранять найденные проблемы.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Устанавливать и настраивать `bandit` для анализа безопасности (ПК 3.2).
2. Интерпретировать отчёты `bandit` и определять уровень серьёзности уязвимостей (ОК 02, ОК 05).
3. Выявлять типовые уязвимости: инъекции, использование `eval`, незащищённые пароли, утечки информации (ПК 3.3).
4. Устранять найденные уязвимости и повышать безопасность кода (ПК 3.3).

---

## Теоретическая справка

### Что такое bandit?

**Bandit** — инструмент для статического анализа безопасности кода на Python. Он сканирует код на наличие типовых уязвимостей и проблем безопасности.

```bash
# Установка
pip install bandit

# Базовый запуск
bandit -r src/

# Запуск с подробным отчётом
bandit -r src/ -f html -o report.html

# Запуск только для определённых уровней
bandit -r src/ -ll  # только средние и высокие
bandit -r src/ -lll # только высокие
Уровни серьёзности
Уровень	Обозначение	Описание
HIGH	(H)	Высокий риск — критическая уязвимость
MEDIUM	(M)	Средний риск — потенциальная проблема
LOW	(L)	Низкий риск — рекомендация к улучшению
Типы выявляемых уязвимостей
Категория	Примеры
Инъекции	SQL injection, командная инъекция
Использование опасных функций	eval(), exec(), pickle
Проблемы с паролями и ключами	Хардкод паролей
Утечки информации	Traceback в продакшене
Проблемы с SSL/TLS	Проверка сертификатов
Небезопасные десериализации	yaml.load, pickle.loads
Объект тестирования
Листинг 1. Модуль vulnerable.py (уязвимый код)
python
# vulnerable.py

import subprocess
import pickle
import yaml
import hashlib
import sqlite3
import os
import warnings
import requests


# ============================================================
# ОПАСНОСТЬ 1: ХАРДКОД ПАРОЛЕЙ И КЛЮЧЕЙ
# ============================================================

PASSWORD = "admin123"  # B105: hardcoded password
API_KEY = "sk-abc123456789"  # B105: hardcoded API key
SECRET = "my_secret_key_123"  # B105: hardcoded secret


# ============================================================
# ОПАСНОСТЬ 2: SQL ИНЪЕКЦИЯ
# ============================================================

def get_user_by_name_sql_injection(username: str):
    """Уязвим к SQL инъекции."""
    conn = sqlite3.connect("users.db")
    cursor = conn.cursor()
    # B608: потенциальная SQL инъекция
    query = f"SELECT * FROM users WHERE username = '{username}'"
    cursor.execute(query)
    return cursor.fetchone()


# ============================================================
# ОПАСНОСТЬ 3: КОМАНДНАЯ ИНЪЕКЦИЯ
# ============================================================

def run_system_command(user_input: str):
    """Уязвим к командной инъекции."""
    # B605: использование subprocess с shell=True
    result = subprocess.run(f"ls {user_input}", shell=True, capture_output=True)
    return result.stdout.decode()


def delete_file_vulnerable(filename: str):
    """Уязвим к командной инъекции через os.system."""
    # B602: использование os.system с пользовательским вводом
    os.system(f"rm {filename}")


# ============================================================
# ОПАСНОСТЬ 4: ИСПОЛЬЗОВАНИЕ eval И exec
# ============================================================

def calculate_expression(expression: str):
    """Опасно: eval с пользовательским вводом."""
    # B307: использование eval
    return eval(expression)


def execute_code(code: str):
    """Опасно: exec с пользовательским вводом."""
    # B102: использование exec
    exec(code)


# ============================================================
# ОПАСНОСТЬ 5: НЕБЕЗОПАСНАЯ ДЕСЕРИАЛИЗАЦИЯ
# ============================================================

def load_pickle(data: bytes):
    """Опасно: pickle.loads может выполнить произвольный код."""
    # B301: использование pickle
    return pickle.loads(data)


def load_yaml(data: str):
    """Опасно: yaml.load без SafeLoader."""
    # B506: использование yaml.load
    return yaml.load(data)


# ============================================================
# ОПАСНОСТЬ 6: УТЕЧКА ИНФОРМАЦИИ В ИСКЛЮЧЕНИЯХ
# ============================================================

def divide_numbers(a: int, b: int):
    """Может раскрыть детали реализации."""
    try:
        return a / b
    except ZeroDivisionError as e:
        # B110: полный traceback может попасть пользователю
        warnings.warn(str(e))
        raise


# ============================================================
# ОПАСНОСТЬ 7: СЛАБАЯ КРИПТОГРАФИЯ
# ============================================================

def hash_password_md5(password: str) -> str:
    """MD5 небезопасен для паролей."""
    # B303: использование MD5
    return hashlib.md5(password.encode()).hexdigest()


def hash_password_sha1(password: str) -> str:
    """SHA1 небезопасен для паролей."""
    # B303: использование SHA1
    return hashlib.sha1(password.encode()).hexdigest()


# ============================================================
# ОПАСНОСТЬ 8: ОТСУТСТВИЕ ПРОВЕРКИ SSL
# ============================================================

def fetch_insecure(url: str):
    """Отключает проверку SSL сертификата."""
    # B501: verify=False отключает проверку SSL
    response = requests.get(url, verify=False)
    return response.text


# ============================================================
# ОПАСНОСТЬ 9: ИСПОЛЬЗОВАНИЕ ВРЕМЕННЫХ ФАЙЛОВ
# ============================================================

def create_temp_file():
    """Небезопасное создание временного файла."""
    # B108: использование mktemp
    import tempfile
    temp_file = tempfile.mktemp()
    with open(temp_file, 'w') as f:
        f.write("sensitive data")
    return temp_file


# ============================================================
# ОПАСНОСТЬ 10: ОТКРЫТИЕ ФАЙЛОВ СЧЁТЧИКОМ ССЫЛОК
# ============================================================

def read_any_file(filepath: str) -> str:
    """Позволяет читать любые файлы."""
    # Не проверяет путь на traversal
    with open(filepath, 'r') as f:
        return f.read()
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Установка и базовый запуск bandit
Средний (варианты 9-17)	Средние студенты	+ анализ отчёта + исправление уязвимостей
Сложный (варианты 18-25)	Сильные студенты	+ интеграция в CI/CD + пользовательские плагины
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться устанавливать bandit, запускать анализ и читать базовый отчёт.

Задания для базового уровня
Задание 1.1. Установка и первый запуск (5 минут)
bash
# Установка bandit
pip install bandit

# Проверка версии
bandit --version

# Базовый запуск на тестовом модуле
bandit vulnerable.py
Ожидаемый вывод (фрагмент):

text
Run started:2024-01-01 12:00:00.123456

Test results:
>> Issue: [B105:hardcoded_password_string] Hardcoded password string
   Severity: LOW   Confidence: MEDIUM
   Location: ./vulnerable.py:13
   More Info: https://bandit.readthedocs.io/en/latest/plugins/b105_hardcoded_password_string.html
13 PASSWORD = "admin123"  # B105: hardcoded password

--------------------------------------------------
>> Issue: [B608:sql_injection] Possible SQL injection vector through string-based query construction
   Severity: MEDIUM   Confidence: LOW
   Location: ./vulnerable.py:27
   More Info: https://bandit.readthedocs.io/en/latest/plugins/b608_sql_injection.html
27     query = f"SELECT * FROM users WHERE username = '{username}'"

--------------------------------------------------
>> Issue: [B605:subprocess_with_shell_equals_true] subprocess call with shell=True identified, security issue.
   Severity: HIGH   Confidence: MEDIUM
   Location: ./vulnerable.py:37
   More Info: https://bandit.readthedocs.io/en/latest/plugins/b605_subprocess_with_shell_equals_true.html
37     result = subprocess.run(f"ls {user_input}", shell=True, capture_output=True)

--------------------------------------------------
Задание 1.2. Анализ отчёта bandit (15 минут)
markdown
## Задание: Анализ отчёта bandit

На основе вывода `bandit` заполните таблицу:

| Строка | Тип уязвимости | Серьёзность | Уверенность | Краткое описание |
|:---|:---|:---|:---|:---|
| 13 | B105 | LOW | MEDIUM | Hardcoded password |
| 27 | B608 | MEDIUM | LOW | SQL injection |
| 37 | B605 | HIGH | MEDIUM | subprocess with shell=True |
| _____ | _____ | _____ | _____ | _____ |
| _____ | _____ | _____ | _____ | _____ |

**Вопросы:**
1. Какая уязвимость имеет наивысший уровень серьёзности?
2. Какая уязвимость имеет наименьший уровень уверенности?
3. Какие уязвимости связаны с инъекциями?
4. Какие уязвимости связаны с хардкодом?
Задание 1.3. Запуск с разными уровнями (10 минут)
bash
# Только высокие (HIGH) уязвимости
bandit vulnerable.py -lll

# Только средние и выше (MEDIUM и HIGH)
bandit vulnerable.py -ll

# Игнорирование определённых уязвимостей
bandit vulnerable.py -s B105,B608

# Вывод в формате JSON
bandit vulnerable.py -f json -o report.json

# Вывод в формате HTML
bandit vulnerable.py -f html -o report.html
Задание: Заполните таблицу результатов при разных уровнях:

Команда	Количество найденных уязвимостей
bandit vulnerable.py	_____
bandit vulnerable.py -lll	_____
bandit vulnerable.py -s B105,B608	_____
Задание 1.4. Итоговый отчёт (10 минут)
markdown
## Отчёт по анализу безопасности (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Общая статистика

| Показатель | Значение |
|:---|:---|
| Всего уязвимостей | _____ |
| HIGH (высокий) | _____ |
| MEDIUM (средний) | _____ |
| LOW (низкий) | _____ |

### Распределение по типам

| Тип уязвимости | Количество |
|:---|:---|
| Хардкод паролей (B105) | _____ |
| SQL инъекция (B608) | _____ |
| Командная инъекция (B605) | _____ |
| eval/exec (B307, B102) | _____ |
| pickle (B301) | _____ |
| yaml.load (B506) | _____ |
| SSL verify=False (B501) | _____ |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться интерпретировать детальные отчёты bandit и устранять уязвимости.

Задания для среднего уровня
Задание 2.1. Детальный анализ и исправление (20 минут)
python
# fixed.py - исправленная версия

import subprocess
import pickle
import yaml
import hashlib
import secrets
import sqlite3
import os
import warnings
import requests
from pathlib import Path


# ============================================================
# ИСПРАВЛЕНИЕ 1: Удаление хардкода
# ============================================================

# ✅ Хорошо: использование переменных окружения
import os

PASSWORD = os.environ.get("APP_PASSWORD", "")
API_KEY = os.environ.get("API_KEY", "")
SECRET = os.environ.get("SECRET_KEY", "")

if not PASSWORD:
    raise ValueError("APP_PASSWORD environment variable not set")


# ============================================================
# ИСПРАВЛЕНИЕ 2: SQL инъекция → параметризованные запросы
# ============================================================

def get_user_by_name_safe(username: str):
    """Безопасный запрос с параметризацией."""
    conn = sqlite3.connect("users.db")
    cursor = conn.cursor()
    # ✅ Используем параметризованный запрос
    query = "SELECT * FROM users WHERE username = ?"
    cursor.execute(query, (username,))
    return cursor.fetchone()


# ============================================================
# ИСПРАВЛЕНИЕ 3: Командная инъекция
# ============================================================

def run_system_command_safe(user_input: str):
    """Безопасное выполнение команды без shell."""
    # ✅ Используем список аргументов вместо shell=True
    result = subprocess.run(["ls", user_input], capture_output=True, text=True)
    return result.stdout


def delete_file_safe(filename: str):
    """Безопасное удаление файла."""
    # ✅ Используем pathlib вместо os.system
    path = Path(filename)
    if path.exists() and path.is_file():
        path.unlink()


# ============================================================
# ИСПРАВЛЕНИЕ 4: Замена eval
# ============================================================

def calculate_expression_safe(expression: str):
    """Безопасный парсинг выражений (только числа и операции)."""
    import ast
    import operator
    
    allowed_ops = {
        ast.Add: operator.add,
        ast.Sub: operator.sub,
        ast.Mult: operator.mul,
        ast.Div: operator.truediv,
    }
    
    def _eval(node):
        if isinstance(node, ast.Constant):
            return node.value
        elif isinstance(node, ast.BinOp):
            left = _eval(node.left)
            right = _eval(node.right)
            return allowed_ops[type(node.op)](left, right)
        raise ValueError(f"Unsupported operation: {type(node).__name__}")
    
    tree = ast.parse(expression, mode='eval')
    return _eval(tree.body)


# ============================================================
# ИСПРАВЛЕНИЕ 5: Безопасная десериализация
# ============================================================

def load_pickle_safe(data: bytes):
    """Безопасная загрузка pickle (только из доверенных источников)."""
    # ✅ Используем json для непроверенных данных
    import json
    try:
        return json.loads(data.decode())
    except (json.JSONDecodeError, UnicodeDecodeError):
        raise ValueError("Invalid data format")


def load_yaml_safe(data: str):
    """Безопасная загрузка YAML."""
    # ✅ Используем SafeLoader
    return yaml.safe_load(data)


# ============================================================
# ИСПРАВЛЕНИЕ 6: Безопасные хеши
# ============================================================

def hash_password_secure(password: str) -> str:
    """Безопасное хеширование с использованием bcrypt."""
    import bcrypt
    salt = bcrypt.gensalt()
    return bcrypt.hashpw(password.encode(), salt).decode()


def hash_password_argon2(password: str) -> str:
    """Безопасное хеширование с использованием Argon2."""
    from argon2 import PasswordHasher
    ph = PasswordHasher()
    return ph.hash(password)


def generate_secure_token() -> str:
    """Генерация безопасного токена."""
    return secrets.token_urlsafe(32)


# ============================================================
# ИСПРАВЛЕНИЕ 7: Включение SSL проверки
# ============================================================

def fetch_secure(url: str):
    """Безопасный запрос с проверкой SSL."""
    # ✅ verify=True по умолчанию
    response = requests.get(url, timeout=30)
    return response.text


# ============================================================
# ИСПРАВЛЕНИЕ 8: Безопасные временные файлы
# ============================================================

def create_secure_temp_file():
    """Безопасное создание временного файла."""
    import tempfile
    # ✅ Используем NamedTemporaryFile
    with tempfile.NamedTemporaryFile(mode='w', delete=False) as f:
        f.write("sensitive data")
        return f.name


# ============================================================
# ИСПРАВЛЕНИЕ 9: Защита от path traversal
# ============================================================

def read_safe_file(filepath: str, base_dir: str = "/safe"):
    """Безопасное чтение файла с проверкой пути."""
    base = Path(base_dir).resolve()
    requested = (base / filepath).resolve()
    
    if not str(requested).startswith(str(base)):
        raise ValueError("Access denied: path traversal detected")
    
    if not requested.exists():
        raise FileNotFoundError(f"File not found: {filepath}")
    
    with open(requested, 'r') as f:
        return f.read()
Задание 2.2. Сравнение отчётов до и после (15 минут)
bash
# Запуск на уязвимой версии
bandit vulnerable.py -f json -o vulnerable_report.json

# Запуск на исправленной версии
bandit fixed.py -f json -o fixed_report.json

# Сравнение количества уязвимостей
Задание: Заполните таблицу сравнения:

Тип уязвимости	vulnerable.py	fixed.py
B105 (хардкод паролей)	_____	_____
B608 (SQL инъекция)	_____	_____
B605 (shell=True)	_____	_____
B307 (eval)	_____	_____
B301 (pickle)	_____	_____
B506 (yaml.load)	_____	_____
B501 (SSL verify)	_____	_____
Всего	_____	_____
Задание 2.3. Исправление уязвимостей (10 минут)
Выберите 3 уязвимости из vulnerable.py и напишите исправленный код:

markdown
### Уязвимость 1: 
**Исходный код:** 
\`\`\`python
# код
\`\`\`
**Исправление:**
\`\`\`python
# исправленный код
\`\`\`

### Уязвимость 2:
**Исходный код:** ...
**Исправление:** ...

### Уязвимость 3:
**Исходный код:** ...
**Исправление:** ...
Задание 2.4. Итоговый отчёт (5 минут)
markdown
## Отчёт по анализу безопасности (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Результаты до исправления

| Уровень | Количество |
|:---|:---|
| HIGH | _____ |
| MEDIUM | _____ |
| LOW | _____ |

### Результаты после исправления

| Уровень | Количество |
|:---|:---|
| HIGH | _____ |
| MEDIUM | _____ |
| LOW | _____ |

### Исправленные уязвимости

| Уязвимость | Тип | Способ исправления |
|:---|:---|:---|
| 1 | _____ | _____ |
| 2 | _____ | _____ |
| 3 | _____ | _____ |

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться настраивать bandit для проекта, интегрировать его в CI/CD и создавать пользовательские правила.

Задания для сложного уровня
Задание 3.1. Настройка .bandit файла (10 минут)
yaml
# .bandit
# Конфигурационный файл для bandit

[bandit]
# Исключить тестовые файлы
exclude: tests,test_,venv,__pycache__

# Уровень серьёзности (HIGH, MEDIUM, LOW)
severity: MEDIUM

# Уровень уверенности (HIGH, MEDIUM, LOW)
confidence: MEDIUM

# Игнорировать определённые проверки
skips: B101,B307

# Формат вывода
format: json

# Выходной файл
output: bandit_report.json

# Рекурсивный обход
recursive: true

# Пользовательские плагины
plugins:
    - custom_plugins.py
bash
# Запуск с конфигурацией
bandit -c .bandit -r src/
Задание 3.2. Интеграция в CI/CD (15 минут)
yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 0 * * 0'  # Еженедельно по воскресеньям

jobs:
  bandit:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        pip install bandit
        pip install -r requirements.txt || true
    
    - name: Run Bandit security scan
      run: |
        bandit -r src/ -f json -o bandit_report.json -lll
    
    - name: Upload Bandit report
      uses: actions/upload-artifact@v3
      with:
        name: bandit-report
        path: bandit_report.json
    
    - name: Check for HIGH severity issues
      run: |
        HIGH_COUNT=$(jq '.results | map(select(.issue_severity == "HIGH")) | length' bandit_report.json)
        if [ "$HIGH_COUNT" -gt 0 ]; then
          echo "Found $HIGH_COUNT HIGH severity security issues"
          exit 1
        fi
Задание: Объясните каждый шаг в CI-пайплайне:

Шаг	Назначение
actions/checkout	
setup-python	
pip install bandit	
bandit -r src/ -lll	
actions/upload-artifact	
Проверка HIGH_COUNT	
Задание 3.3. Создание пользовательского плагина (15 минут)
python
# custom_plugins.py
"""
Пользовательские плагины для bandit.
"""

import ast
from bandit.core import test_properties as test


@test.test("Custom rule: no print statements in production")
def no_print_statements(context):
    """
    Запрещает использование print() в продакшен-коде.
    """
    if isinstance(context.node, ast.Call):
        if isinstance(context.node.func, ast.Name) and context.node.func.id == 'print':
            return bandit.Issue(
                severity=bandit.MEDIUM,
                confidence=bandit.HIGH,
                text="Print statement found. Use logging instead."
            )
    return None


@test.test("Custom rule: no direct os.environ access")
def no_direct_os_environ(context):
    """
    Запрещает прямой доступ к os.environ.
    """
    if isinstance(context.node, ast.Attribute):
        if isinstance(context.node.value, ast.Name) and context.node.value.id == 'os':
            if context.node.attr == 'environ':
                return bandit.Issue(
                    severity=bandit.LOW,
                    confidence=bandit.MEDIUM,
                    text="Direct os.environ access. Use config module instead."
                )
    return None
bash
# Регистрация плагина
bandit -r src/ --plugins custom_plugins.py
Задание 3.4. Интеграция с другими инструментами (10 минут)
bash
# Интеграция с pytest
pytest --bandit src/

# Интеграция с pre-commit
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/PyCQA/bandit
    rev: '1.7.5'
    hooks:
      - id: bandit
        args: ["-lll", "--recursive", "src/"]
Задание: Напишите pre-commit конфигурацию для запуска bandit перед каждым коммитом.

yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/PyCQA/bandit
    rev: ''  # укажите версию
    hooks:
      - id: bandit
        args: 
          - -lll
          - --recursive
          - src/
Задание 3.5. Итоговый отчёт (5 минут)
markdown
## Отчёт по анализу безопасности (сложный уровень)

**Студент:** _________________
**Вариант №:** ___ (18-25)

### Конфигурация bandit

- Файл конфигурации: `.bandit`
- Уровень серьёзности: _____
- Исключённые проверки: _____

### CI/CD интеграция

- Платформа: _____ (GitHub Actions / GitLab CI)
- Триггеры: _____
- Действия при HIGH уязвимостях: _____

### Пользовательские плагины

| Плагин | Назначение | Уровень |
|:---|:---|:---|
| no_print_statements | Запрет print() | MEDIUM |
| _____ | _____ | _____ |

### Интеграция с pre-commit

- [ ] pre-commit настроен
- [ ] bandit запускается перед коммитом

### Выводы

_______________________________________________________________

_______________________________________________________________
Карточка студента
text
ПР 3.10. АНАЛИЗ БЕЗОПАСНОСТИ КОДА С BANDIT

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ Установка bandit (базовый)
□ Базовый запуск (базовый)
□ Анализ отчёта (базовый)
□ Исправление уязвимостей (средний)
□ Сравнение отчётов (средний)
□ Конфигурация .bandit (сложный)
□ CI/CD интеграция (сложный)
□ Пользовательские плагины (сложный)

=== ОБНАРУЖЕННЫЕ УЯЗВИМОСТИ ===

| Тип | Было | Стало |
|:---|:---|:---|
| HIGH | _____ | _____ |
| MEDIUM | _____ | _____ |
| LOW | _____ | _____ |

=== ИСПРАВЛЕННЫЕ УЯЗВИМОСТИ ===

□ Хардкод паролей
□ SQL инъекции
□ Командные инъекции
□ eval/exec
□ Небезопасная десериализация
□ SSL verify=False

=== ВРЕМЯ ВЫПОЛНЕНИЯ ===

Подготовка: _____ мин
Выполнение: _____ мин
Оформление: _____ мин

=== ОТЧЁТ ===

Файлы:
- vulnerable.py
- fixed.py
- bandit_report.json
- .bandit
- custom_plugins.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Bandit не установлен или не запущен
3 (удовлетворительно)	Базовый	Установка + базовый анализ (10+ уязвимостей)
4 (хорошо)	Средний	+ Исправление уязвимостей + сравнение (15+ уязвимостей)
5 (отлично)	Сложный	+ CI/CD + пользовательские плагины (все уязвимости)
Контрольные вопросы (для защиты)
Что такое bandit и для чего он используется?

Какие уровни серьёзности уязвимостей существуют в bandit?

Как запустить bandit только для высоких уязвимостей?

Как игнорировать определённые типы уязвимостей?

Как исправить SQL инъекцию в Python коде?

Почему опасно использовать eval() с пользовательским вводом?

Как безопасно загружать YAML файлы?

Как интегрировать bandit в CI/CD?

Как создать пользовательский плагин для bandit?

Почему использование shell=True в subprocess опасно?

Следующее занятие: Лекция 3.16 — Тестирование производительности и нагрузочное тестирование.
