# Лекция 3.12: Метрики качества тестов и ПО: плотность дефектов, надежность, стоимость исправления

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить основные метрики качества тестов и программного обеспечения: плотность дефектов, надежность, стоимость исправления ошибок. Научиться собирать, анализировать и интерпретировать эти метрики для оценки качества продукта.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Определять и рассчитывать основные метрики качества ПО (ОК 01, ОК 02).
2. Использовать плотность дефектов для оценки качества модулей (ПК 3.3).
3. Оценивать надежность программного обеспечения (ОК 05).
4. Анализировать стоимость исправления дефектов на разных этапах разработки (ОК 01, ОК 05).

---

## 1. Зачем нужны метрики качества?

### 1.1 Роль метрик в процессе разработки
┌─────────────────────────────────────────────────────────────────────────────┐
│ ЗАЧЕМ НУЖНЫ МЕТРИКИ КАЧЕСТВА │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ 1. ИЗМЕРЕНИЕ ПРОГРЕССА │
│ ├── Как меняется качество продукта во времени │
│ └── Достигнуты ли целевые показатели │
│ │
│ 2. ПРИНЯТИЕ РЕШЕНИЙ │
│ ├── Готов ли продукт к релизу? │
│ └── Какие модули требуют доработки? │
│ │
│ 3. ПРОГНОЗИРОВАНИЕ │
│ ├── Сколько ещё дефектов может быть обнаружено │
│ └── Когда можно завершить тестирование │
│ │
│ 4. ОПТИМИЗАЦИЯ │
│ ├── Какие процессы тестирования работают плохо │
│ └── Куда направлять ресурсы для улучшения качества │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 1.2 Классификация метрик

| Категория | Описание | Примеры |
|:---|:---|:---|
| **Метрики дефектов** | Количество, плотность, серьёзность | Плотность дефектов, количество критических багов |
| **Метрики тестов** | Покрытие, проходимость, эффективность | Покрытие кода, процент пройденных тестов |
| **Метрики процесса** | Время, стоимость, продуктивность | Время исправления бага, стоимость дефекта |
| **Метрики надёжности** | Стабильность, безотказность | MTBF, вероятность отказа |

---

## 2. Плотность дефектов

### 2.1 Определение и формула

**Плотность дефектов (Defect Density)** — количество дефектов на единицу размера программного кода.
┌─────────────────────────────────────────────────────────────────────────────┐
│ ПЛОТНОСТЬ ДЕФЕКТОВ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ Формула: Плотность дефектов = (Количество дефектов) / (Размер кода) │
│ │
│ Единицы измерения: │
│ • Дефектов на 1000 строк кода (Defects per KLOC) │
│ • Дефектов на 1 функцию │
│ • Дефектов на 1 модуль │
│ │
│ Интерпретация: │
│ • < 1 дефект на KLOC — отличное качество │
│ • 1-5 дефектов на KLOC — хорошее качество │
│ • 5-10 дефектов на KLOC — среднее качество │
│ • > 10 дефектов на KLOC — плохое качество │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 2.2 Пример расчёта

```python
def calculate_defect_density(defects: int, lines_of_code: int) -> float:
    """
    Рассчитывает плотность дефектов на 1000 строк кода.
    """
    return (defects / lines_of_code) * 1000


# Пример
module_1 = {"defects": 5, "lines": 1000}   # плотность: 5.0
module_2 = {"defects": 2, "lines": 500}    # плотность: 4.0
module_3 = {"defects": 15, "lines": 2000}  # плотность: 7.5

print(f"Module 1 density: {calculate_defect_density(**module_1):.1f}")  # 5.0
print(f"Module 2 density: {calculate_defect_density(**module_2):.1f}")  # 4.0
print(f"Module 3 density: {calculate_defect_density(**module_3):.1f}")  # 7.5
2.3 Анализ плотности дефектов по модулям
python
def analyze_defect_density(modules_data: dict) -> dict:
    """
    Анализирует плотность дефектов по модулям.
    
    Returns:
        dict с анализом: проблемные модули, средняя плотность, рекомендации
    """
    densities = {}
    for module, data in modules_data.items():
        density = (data["defects"] / data["lines"]) * 1000
        densities[module] = density
    
    avg_density = sum(densities.values()) / len(densities)
    problematic = [m for m, d in densities.items() if d > avg_density * 1.5]
    
    return {
        "densities": densities,
        "average": avg_density,
        "problematic_modules": problematic,
        "recommendation": "Focus testing on " + ", ".join(problematic) if problematic else "Quality is consistent"
    }


# Пример использования
modules = {
    "auth": {"defects": 3, "lines": 800},
    "payment": {"defects": 12, "lines": 1200},
    "user_profile": {"defects": 2, "lines": 600},
    "admin_panel": {"defects": 8, "lines": 900},
}

analysis = analyze_defect_density(modules)
print(f"Average density: {analysis['average']:.1f}")
print(f"Problematic modules: {analysis['problematic_modules']}")
3. Надёжность программного обеспечения
3.1 Основные метрики надёжности
Метрика	Описание	Формула
MTBF	Среднее время между отказами	Общее время работы / количество отказов
MTTF	Среднее время до первого отказа	Общее время до отказа / количество объектов
MTTR	Среднее время восстановления	Общее время простоя / количество отказов
Доступность	Вероятность, что система работает	MTBF / (MTBF + MTTR)
3.2 Расчёт метрик надёжности
python
class ReliabilityMetrics:
    """Класс для расчёта метрик надёжности."""
    
    def __init__(self):
        self.failures = []
        self.recovery_times = []
    
    def record_failure(self, timestamp: float, recovery_time: float):
        """Записывает информацию об отказе."""
        self.failures.append(timestamp)
        self.recovery_times.append(recovery_time)
    
    def calculate_mtbf(self, total_time: float) -> float:
        """
        Среднее время между отказами.
        
        Args:
            total_time: Общее время наблюдения
        
        Returns:
            MTBF в часах
        """
        if not self.failures:
            return total_time
        return total_time / len(self.failures)
    
    def calculate_mttr(self) -> float:
        """Среднее время восстановления."""
        if not self.recovery_times:
            return 0
        return sum(self.recovery_times) / len(self.recovery_times)
    
    def calculate_availability(self, total_time: float) -> float:
        """
        Коэффициент доступности.
        
        Returns:
            Значение от 0 до 1
        """
        mttr = self.calculate_mttr()
        mtbf = self.calculate_mtbf(total_time)
        return mtbf / (mtbf + mttr) if (mtbf + mttr) > 0 else 1.0


# Пример использования
metrics = ReliabilityMetrics()
metrics.record_failure(10, 0.5)   # отказ на 10-й час, восстановление 0.5ч
metrics.record_failure(25, 1.0)   # отказ на 25-й час, восстановление 1ч
metrics.record_failure(40, 0.75)  # отказ на 40-й час, восстановление 0.75ч

mtbf = metrics.calculate_mtbf(50)      # 50/3 = 16.67 ч
mttr = metrics.calculate_mttr()        # (0.5+1+0.75)/3 = 0.75 ч
availability = metrics.calculate_availability(50)  # 16.67/(16.67+0.75) = 0.957

print(f"MTBF: {mtbf:.2f} hours")
print(f"MTTR: {mttr:.2f} hours")
print(f"Availability: {availability:.2%}")
3.3 Модели надёжности
python
# Модель Джелински-Миранды (Jelinski-Moranda)
class JelinskiMorandaModel:
    """
    Модель для прогнозирования количества оставшихся дефектов.
    """
    
    def __init__(self, initial_defects: int, defect_rate: float):
        self.remaining = initial_defects
        self.defect_rate = defect_rate  # вероятность обнаружения дефекта
    
    def detect_defect(self) -> bool:
        """Имитация обнаружения дефекта."""
        import random
        if random.random() < self.defect_rate and self.remaining > 0:
            self.remaining -= 1
            return True
        return False
    
    def get_remaining_defects(self) -> int:
        return self.remaining


# Использование
model = JelinskiMorandaModel(initial_defects=100, defect_rate=0.1)
for _ in range(50):
    model.detect_defect()
print(f"Remaining defects: {model.get_remaining_defects()}")
4. Стоимость исправления дефектов
4.1 Закон увеличения стоимости
text
┌─────────────────────────────────────────────────────────────────────────────┐
│              СТОИМОСТЬ ИСПРАВЛЕНИЯ ДЕФЕКТОВ ПО ЭТАПАМ                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Стоимость                                                                  │
│      ↑                                                                       │
│  1x  │    ● Требования                                                       │
│      │                                                                       │
│  10x │        ● Проектирование                                               │
│      │                                                                       │
│ 100x │              ● Разработка                                             │
│      │                                                                       │
│1000x │                    ● Тестирование                                     │
│      │                                                                       │
│10000x│                          ● Эксплуатация                               │
│      │                                                                       │
│      └──────────────────────────────────────────────────────→ Этап          │
│                                                                              │
│   Цена исправления растёт экспоненциально с каждым этапом                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
4.2 Расчёт стоимости исправления
python
class DefectCostCalculator:
    """Калькулятор стоимости исправления дефектов."""
    
    # Коэффициенты стоимости по этапам (относительно требований)
    STAGE_COEFFICIENTS = {
        "requirements": 1,
        "design": 10,
        "development": 100,
        "testing": 1000,
        "production": 10000
    }
    
    def __init__(self, hourly_rate: float = 100):
        self.hourly_rate = hourly_rate
    
    def calculate_cost(self, hours_to_fix: float, stage: str) -> float:
        """
        Рассчитывает стоимость исправления дефекта.
        
        Args:
            hours_to_fix: Часы на исправление
            stage: Этап обнаружения
        
        Returns:
            Общая стоимость
        """
        coefficient = self.STAGE_COEFFICIENTS.get(stage, 1)
        return hours_to_fix * self.hourly_rate * coefficient
    
    def calculate_saving(self, hours_to_fix: float, stage_found: str, stage_fixed: str) -> float:
        """Рассчитывает экономию от раннего обнаружения."""
        cost_at_stage = self.calculate_cost(hours_to_fix, stage_found)
        cost_if_later = self.calculate_cost(hours_to_fix, stage_fixed)
        return cost_if_later - cost_at_stage


# Пример
calculator = DefectCostCalculator(hourly_rate=100)

print("Cost to fix at requirements:", calculator.calculate_cost(2, "requirements"))  # 200
print("Cost to fix at development:", calculator.calculate_cost(2, "development"))    # 20,000
print("Cost to fix at production:", calculator.calculate_cost(2, "production"))      # 2,000,000

saving = calculator.calculate_saving(2, "requirements", "production")
print(f"Saved by finding early: ${saving:,.0f}")  # 1,999,800
4.3 Анализ ROI тестирования
python
def calculate_roi(
    testing_cost: float,
    defects_found: int,
    avg_fix_hours: float,
    hourly_rate: float,
    stage_detected: str = "testing"
) -> dict:
    """
    Рассчитывает ROI от тестирования.
    """
    calculator = DefectCostCalculator(hourly_rate)
    
    # Стоимость исправления на этапе тестирования
    cost_at_testing = calculator.calculate_cost(avg_fix_hours, stage_detected) * defects_found
    
    # Предполагаемая стоимость, если бы дефекты дошли до продакшена
    cost_if_production = calculator.calculate_cost(avg_fix_hours, "production") * defects_found
    
    saved = cost_if_production - cost_at_testing
    net_saving = saved - testing_cost
    roi = (net_saving / testing_cost) * 100 if testing_cost > 0 else 0
    
    return {
        "testing_cost": testing_cost,
        "cost_at_testing": cost_at_testing,
        "cost_if_production": cost_if_production,
        "saved": saved,
        "net_saving": net_saving,
        "roi_percent": roi
    }


# Пример
result = calculate_roi(
    testing_cost=50000,
    defects_found=100,
    avg_fix_hours=2,
    hourly_rate=100,
    stage_detected="testing"
)

print(f"ROI from testing: {result['roi_percent']:.0f}%")
print(f"Net saving: ${result['net_saving']:,.0f}")
5. Сбор и визуализация метрик
5.1 Класс для сбора метрик
python
class MetricsCollector:
    """Сборщик метрик качества."""
    
    def __init__(self):
        self.defects = []
        self.test_results = []
        self.releases = []
    
    def add_defect(self, severity: str, module: str, stage_found: str):
        self.defects.append({
            "severity": severity,
            "module": module,
            "stage_found": stage_found,
            "timestamp": datetime.now()
        })
    
    def get_defect_density(self, module: str, lines_of_code: int) -> float:
        module_defects = [d for d in self.defects if d["module"] == module]
        return (len(module_defects) / lines_of_code) * 1000
    
    def get_defects_by_severity(self) -> dict:
        from collections import Counter
        return Counter(d["severity"] for d in self.defects)
    
    def generate_report(self) -> dict:
        return {
            "total_defects": len(self.defects),
            "defects_by_severity": self.get_defects_by_severity(),
            "defects_by_stage": Counter(d["stage_found"] for d in self.defects),
            "defects_by_module": Counter(d["module"] for d in self.defects)
        }


collector = MetricsCollector()
collector.add_defect("critical", "payment", "testing")
collector.add_defect("high", "auth", "development")
collector.add_defect("medium", "payment", "testing")
collector.add_defect("low", "user_profile", "testing")

print(collector.generate_report())
5.2 Дашборд метрик
python
import matplotlib.pyplot as plt

def plot_defect_trend(defects_by_day: dict):
    """
    Визуализация тренда обнаружения дефектов.
    """
    days = list(defects_by_day.keys())
    counts = list(defects_by_day.values())
    
    plt.figure(figsize=(10, 6))
    plt.plot(days, counts, marker='o')
    plt.title('Defect Detection Trend')
    plt.xlabel('Day')
    plt.ylabel('Number of Defects')
    plt.grid(True)
    plt.show()


def plot_severity_distribution(severity_counts: dict):
    """
    Визуализация распределения дефектов по серьёзности.
    """
    severities = list(severity_counts.keys())
    counts = list(severity_counts.values())
    colors = {'critical': 'red', 'high': 'orange', 'medium': 'yellow', 'low': 'green'}
    
    plt.figure(figsize=(8, 6))
    bars = plt.bar(severities, counts, color=[colors.get(s, 'blue') for s in severities])
    plt.title('Defects by Severity')
    plt.xlabel('Severity')
    plt.ylabel('Count')
    plt.show()
6. Шпаргалка
python
# === ПЛОТНОСТЬ ДЕФЕКТОВ ===
density = (defects / lines) * 1000  # defects per KLOC

# === НАДЁЖНОСТЬ ===
mtbf = total_time / failures
availability = mtbf / (mtbf + mttr)

# === СТОИМОСТЬ ИСПРАВЛЕНИЯ ===
cost = hours * rate * coefficient
# coefficients: requirements=1, design=10, development=100, testing=1000, production=10000

# === МЕТРИКИ ДЛЯ ОТЧЁТА ===
report = {
    "total_defects": 100,
    "critical": 2,
    "high": 8,
    "medium": 30,
    "low": 60,
    "defect_density": 4.2,  # per KLOC
    "test_pass_rate": 95.5,  # %
    "coverage": 85.0,        # %
}
Контрольные вопросы
Что такое плотность дефектов и как её рассчитать?

Какое значение плотности дефектов считается хорошим?

Что означает MTBF и как его интерпретировать?

Почему стоимость исправления дефекта растёт с каждым этапом?

Во сколько раз дороже исправлять дефект на этапе эксплуатации?

Как рассчитать коэффициент доступности системы?

Какие метрики помогают оценить эффективность тестирования?

Как раннее тестирование влияет на ROI?

Какие факторы влияют на надёжность ПО?

Как интерпретировать высокую плотность дефектов в одном модуле?

Следующее занятие: ПР 3.12 — Практический анализ метрик качества.
