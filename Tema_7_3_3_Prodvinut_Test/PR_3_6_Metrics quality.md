# Практическое занятие 3.6: Расчёт метрик качества на основе результатов тестирования

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.3:** Продвинутое тестирование  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться собирать, анализировать и интерпретировать метрики качества ПО на основе результатов тестирования: плотность дефектов, надёжность, стоимость исправления, эффективность тестирования.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Собирать данные о дефектах и тестах для расчёта метрик (ПК 3.3).
2. Рассчитывать плотность дефектов для модулей и системы в целом (ПК 3.3).
3. Оценивать надёжность ПО по результатам тестирования (ОК 02, ОК 05).
4. Анализировать эффективность тестирования и стоимость исправления дефектов (ОК 01, ОК 05).

---

## Теоретическая справка

### Основные метрики качества

| Метрика | Формула | Интерпретация |
|:---|:---|:---|
| **Плотность дефектов** | Дефекты / KLOC | < 1 — отлично, 1-5 — хорошо, 5-10 — средне, >10 — плохо |
| **Эффективность тестирования** | (Найдено дефектов / Всего дефектов) × 100% | Чем выше, тем лучше |
| **Плотность дефектов по модулям** | Дефекты модуля / KLOC модуля | Выявление проблемных модулей |
| **MTTF** | Время до отказа | Среднее время безотказной работы |
| **Стоимость исправления** | Часы × Ставка × Коэффициент этапа | Раннее обнаружение дешевле |

---

## Объект анализа

### Данные о тестировании проекта «Интернет-магазин»

```python
# test_metrics_data.py

import json
from datetime import datetime, timedelta
from typing import List, Dict, Any
from dataclasses import dataclass, field


@dataclass
class Defect:
    """Модель дефекта."""
    id: int
    title: str
    severity: str  # critical, high, medium, low
    module: str    # auth, payment, catalog, cart, order
    stage_found: str  # requirements, design, development, testing, production
    hours_to_fix: float
    detected_at: datetime
    fixed_at: datetime = None


@dataclass
class ModuleInfo:
    """Информация о модуле."""
    name: str
    lines_of_code: int
    complexity: str  # low, medium, high


@dataclass
class TestRun:
    """Результаты прогона тестов."""
    date: datetime
    total_tests: int
    passed: int
    failed: int
    skipped: int
    execution_time_seconds: float


# ============================================================
# ТЕСТОВЫЕ ДАННЫЕ
# ============================================================

def load_defects() -> List[Defect]:
    """Загружает данные о дефектах."""
    base_date = datetime(2024, 1, 1)
    
    return [
        Defect(1, "Login fails with valid credentials", "critical", "auth", "testing", 4.0,
               base_date + timedelta(days=5), base_date + timedelta(days=6)),
        Defect(2, "Session timeout too short", "high", "auth", "development", 2.0,
               base_date + timedelta(days=2), base_date + timedelta(days=3)),
        Defect(3, "SQL injection vulnerability", "critical", "auth", "testing", 8.0,
               base_date + timedelta(days=7), base_date + timedelta(days=9)),
        Defect(4, "Payment gateway timeout not handled", "high", "payment", "development", 3.0,
               base_date + timedelta(days=3), base_date + timedelta(days=4)),
        Defect(5, "Incorrect discount calculation", "high", "cart", "testing", 2.5,
               base_date + timedelta(days=4), base_date + timedelta(days=5)),
        Defect(6, "Product image not loading", "medium", "catalog", "testing", 1.5,
               base_date + timedelta(days=6), base_date + timedelta(days=7)),
        Defect(7, "Order status not updating", "critical", "order", "production", 6.0,
               base_date + timedelta(days=15), base_date + timedelta(days=16)),
        Defect(8, "Email confirmation not sent", "high", "order", "testing", 3.0,
               base_date + timedelta(days=8), base_date + timedelta(days=9)),
        Defect(9, "Cart total rounding error", "medium", "cart", "development", 1.0,
               base_date + timedelta(days=1), base_date + timedelta(days=1)),
        Defect(10, "Search returns wrong results", "medium", "catalog", "testing", 2.0,
              base_date + timedelta(days=9), base_date + timedelta(days=10)),
        Defect(11, "Checkout page crashes on mobile", "high", "order", "testing", 4.0,
               base_date + timedelta(days=10), base_date + timedelta(days=12)),
        Defect(12, "Password reset link expired too fast", "low", "auth", "development", 1.5,
               base_date + timedelta(days=2), base_date + timedelta(days=3)),
    ]


def load_modules() -> List[ModuleInfo]:
    """Загружает информацию о модулях."""
    return [
        ModuleInfo("auth", 2500, "medium"),
        ModuleInfo("payment", 1800, "high"),
        ModuleInfo("catalog", 3500, "low"),
        ModuleInfo("cart", 1200, "medium"),
        ModuleInfo("order", 2200, "high"),
    ]


def load_test_runs() -> List[TestRun]:
    """Загружает результаты тестовых прогонов."""
    base_date = datetime(2024, 1, 1)
    
    return [
        TestRun(base_date + timedelta(days=1), 150, 120, 25, 5, 180),
        TestRun(base_date + timedelta(days=3), 155, 135, 15, 5, 175),
        TestRun(base_date + timedelta(days=5), 160, 148, 8, 4, 170),
        TestRun(base_date + timedelta(days=7), 165, 158, 4, 3, 168),
        TestRun(base_date + timedelta(days=10), 170, 165, 2, 3, 165),
        TestRun(base_date + timedelta(days=14), 175, 172, 1, 2, 162),
        TestRun(base_date + timedelta(days=21), 180, 179, 0, 1, 160),
    ]
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Расчёт плотности дефектов
Средний (варианты 9-17)	Средние студенты	+ анализ надёжности и трендов
Сложный (варианты 18-25)	Сильные студенты	+ стоимость, ROI, рекомендации
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться рассчитывать плотность дефектов для модулей и системы в целом.

Задания для базового уровня
Задание 1.1. Расчёт общей плотности дефектов (10 минут)
python
# metrics_base.py

import json
from test_metrics_data import load_defects, load_modules, Defect, ModuleInfo


class MetricsCalculator:
    """Калькулятор метрик качества."""
    
    def __init__(self, defects: List[Defect], modules: List[ModuleInfo]):
        self.defects = defects
        self.modules = {m.name: m for m in modules}
        self.total_loc = sum(m.lines_of_code for m in modules)
    
    def calculate_overall_defect_density(self) -> float:
        """
        Рассчитывает общую плотность дефектов на 1000 строк кода.
        
        Returns:
            Плотность дефектов (дефектов на KLOC)
        """
        total_defects = len(self.defects)
        total_k_loc = self.total_loc / 1000
        return total_defects / total_k_loc if total_k_loc > 0 else 0
    
    def get_defects_by_severity(self) -> dict:
        """Возвращает количество дефектов по серьёзности."""
        severity_counts = {"critical": 0, "high": 0, "medium": 0, "low": 0}
        for d in self.defects:
            severity_counts[d.severity] = severity_counts.get(d.severity, 0) + 1
        return severity_counts


def main():
    defects = load_defects()
    modules = load_modules()
    
    calculator = MetricsCalculator(defects, modules)
    
    print("=" * 50)
    print("МЕТРИКИ КАЧЕСТВА (БАЗОВЫЙ УРОВЕНЬ)")
    print("=" * 50)
    
    # Общая плотность дефектов
    density = calculator.calculate_overall_defect_density()
    print(f"\nОбщая плотность дефектов: {density:.2f} дефектов на 1000 строк")
    
    # Оценка качества
    if density < 1:
        quality = "Отлично"
    elif density < 5:
        quality = "Хорошо"
    elif density < 10:
        quality = "Средне"
    else:
        quality = "Плохо"
    print(f"Оценка качества: {quality}")
    
    # Распределение по серьёзности
    by_severity = calculator.get_defects_by_severity()
    print(f"\nРаспределение по серьёзности:")
    for severity, count in by_severity.items():
        print(f"  {severity}: {count}")


if __name__ == "__main__":
    main()
Ожидаемый вывод:

text
==================================================
МЕТРИКИ КАЧЕСТВА (БАЗОВЫЙ УРОВЕНЬ)
==================================================

Общая плотность дефектов: 1.20 дефектов на 1000 строк
Оценка качества: Хорошо

Распределение по серьёзности:
  critical: 3
  high: 4
  medium: 3
  low: 2
Задание 1.2. Расчёт плотности дефектов по модулям (15 минут)
python
class ModuleMetricsCalculator(MetricsCalculator):
    """Расширенный калькулятор с метриками по модулям."""
    
    def calculate_module_defect_density(self) -> dict:
        """
        Рассчитывает плотность дефектов для каждого модуля.
        
        Returns:
            Словарь {имя_модуля: плотность}
        """
        module_defects = {name: 0 for name in self.modules.keys()}
        
        for defect in self.defects:
            if defect.module in module_defects:
                module_defects[defect.module] += 1
        
        densities = {}
        for name, module_info in self.modules.items():
            k_loc = module_info.lines_of_code / 1000
            densities[name] = module_defects[name] / k_loc if k_loc > 0 else 0
        
        return densities
    
    def identify_problematic_modules(self, threshold: float = 3.0) -> List[str]:
        """
        Выявляет модули с плотностью дефектов выше порога.
        
        Args:
            threshold: Пороговое значение плотности дефектов
        
        Returns:
            Список проблемных модулей
        """
        densities = self.calculate_module_defect_density()
        return [name for name, density in densities.items() if density > threshold]
    
    def get_defects_by_module(self) -> dict:
        """Возвращает количество дефектов по модулям."""
        module_defects = {name: 0 for name in self.modules.keys()}
        for defect in self.defects:
            if defect.module in module_defects:
                module_defects[defect.module] += 1
        return module_defects


def main_advanced():
    defects = load_defects()
    modules = load_modules()
    
    calculator = ModuleMetricsCalculator(defects, modules)
    
    print("=" * 50)
    print("МЕТРИКИ ПО МОДУЛЯМ")
    print("=" * 50)
    
    # Плотность по модулям
    densities = calculator.calculate_module_defect_density()
    defect_counts = calculator.get_defects_by_module()
    
    print("\nПлотность дефектов по модулям (дефектов на KLOC):")
    for module, density in sorted(densities.items(), key=lambda x: x[1], reverse=True):
        loc = modules[module].lines_of_code
        print(f"  {module:10} | LOC: {loc:5} | Дефектов: {defect_counts[module]:2} | "
              f"Плотность: {density:.2f}")
    
    # Проблемные модули
    problematic = calculator.identify_problematic_modules(threshold=3.0)
    if problematic:
        print(f"\n⚠️ Проблемные модули (плотность > 3.0): {', '.join(problematic)}")
    else:
        print("\n✅ Все модули имеют приемлемую плотность дефектов")


if __name__ == "__main__":
    main_advanced()
Ожидаемый вывод:

text
==================================================
МЕТРИКИ ПО МОДУЛЯМ
==================================================

Плотность дефектов по модулям (дефектов на KLOC):
  auth       | LOC:  2500 | Дефектов:  4 | Плотность: 1.60
  order      | LOC:  2200 | Дефектов:  3 | Плотность: 1.36
  payment    | LOC:  1800 | Дефектов:  2 | Плотность: 1.11
  cart       | LOC:  1200 | Дефектов:  2 | Плотность: 1.67
  catalog    | LOC:  3500 | Дефектов:  2 | Плотность: 0.57

✅ Все модули имеют приемлемую плотность дефектов
Задание 1.3. Расчёт эффективности тестирования (10 минут)
python
class TestEffectivenessCalculator:
    """Калькулятор эффективности тестирования."""
    
    def __init__(self, defects: List[Defect]):
        self.defects = defects
    
    def calculate_effectiveness(self) -> dict:
        """
        Рассчитывает эффективность тестирования по этапам обнаружения.
        
        Returns:
            Словарь с процентным распределением
        """
        stages = ["requirements", "design", "development", "testing", "production"]
        stage_counts = {stage: 0 for stage in stages}
        
        for defect in self.defects:
            if defect.stage_found in stage_counts:
                stage_counts[defect.stage_found] += 1
        
        total = len(self.defects)
        effectiveness = {stage: (count / total * 100) for stage, count in stage_counts.items()}
        
        return effectiveness
    
    def get_testing_defects_percentage(self) -> float:
        """Возвращает процент дефектов, найденных на этапе тестирования."""
        effectiveness = self.calculate_effectiveness()
        return effectiveness.get("testing", 0)


def calculate_effectiveness():
    defects = load_defects()
    calculator = TestEffectivenessCalculator(defects)
    
    print("=" * 50)
    print("ЭФФЕКТИВНОСТЬ ТЕСТИРОВАНИЯ")
    print("=" * 50)
    
    effectiveness = calculator.calculate_effectiveness()
    
    print("\nРаспределение дефектов по этапам обнаружения:")
    for stage, percent in sorted(effectiveness.items(), key=lambda x: x[1], reverse=True):
        print(f"  {stage:12}: {percent:5.1f}%")
    
    testing_percent = calculator.get_testing_defects_percentage()
    print(f"\nДефектов, найденных на этапе тестирования: {testing_percent:.1f}%")
    
    if testing_percent > 50:
        print("✅ Тестирование эффективно: большинство дефектов найдено до релиза")
    else:
        print("⚠️ Много дефектов уходит в продакшн — требуется усиление тестирования")


if __name__ == "__main__":
    calculate_effectiveness()
Задание 1.4. Итоговый отчёт (5 минут)
markdown
## Отчёт по расчёту метрик качества (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Основные метрики

| Метрика | Значение | Оценка |
|:---|:---|:---|
| Общая плотность дефектов | _____ дефектов/KLOC | _____ |
| Всего дефектов | _____ | — |
| Критические дефекты | _____ | — |
| Высокие дефекты | _____ | — |

### Плотность по модулям

| Модуль | LOC | Дефекты | Плотность |
|:---|:---|:---|:---|
| auth | 2500 | _____ | _____ |
| payment | 1800 | _____ | _____ |
| catalog | 3500 | _____ | _____ |
| cart | 1200 | _____ | _____ |
| order | 2200 | _____ | _____ |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться анализировать тренды дефектов и рассчитывать показатели надёжности.

Задания для среднего уровня
Задание 2.1. Анализ тренда дефектов (15 минут)
python
# metrics_intermediate.py

from datetime import datetime, timedelta
from collections import defaultdict
import matplotlib.pyplot as plt


class TrendAnalyzer:
    """Анализ трендов дефектов."""
    
    def __init__(self, defects: List[Defect], test_runs: List[TestRun]):
        self.defects = defects
        self.test_runs = test_runs
    
    def get_defects_by_day(self) -> dict:
        """Группирует дефекты по дням."""
        defects_by_day = defaultdict(int)
        for defect in self.defects:
            day = defect.detected_at.date()
            defects_by_day[day] += 1
        return dict(defects_by_day)
    
    def get_cumulative_defects(self) -> List[tuple]:
        """Возвращает накопленное количество дефектов по дням."""
        defects_by_day = self.get_defects_by_day()
        days = sorted(defects_by_day.keys())
        
        cumulative = []
        total = 0
        for day in days:
            total += defects_by_day[day]
            cumulative.append((day, total))
        return cumulative
    
    def calculate_defect_detection_rate(self) -> float:
        """
        Рассчитывает среднюю скорость обнаружения дефектов.
        (дефектов в день)
        """
        if not self.defects:
            return 0
        
        first_defect = min(d.detected_at for d in self.defects)
        last_defect = max(d.detected_at for d in self.defects)
        days = (last_defect - first_defect).days
        return len(self.defects) / days if days > 0 else 0
    
    def calculate_fix_rate(self) -> float:
        """Рассчитывает среднюю скорость исправления дефектов."""
        fixed = [d for d in self.defects if d.fixed_at]
        if not fixed:
            return 0
        
        total_hours = sum(d.hours_to_fix for d in fixed)
        return total_hours / len(fixed)
    
    def plot_defect_trend(self):
        """Визуализирует тренд обнаружения дефектов."""
        cumulative = self.get_cumulative_defects()
        
        days = [c[0] for c in cumulative]
        counts = [c[1] for c in cumulative]
        
        plt.figure(figsize=(10, 6))
        plt.plot(days, counts, marker='o', linewidth=2)
        plt.title('Cumulative Defects Over Time')
        plt.xlabel('Date')
        plt.ylabel('Total Defects Found')
        plt.grid(True)
        plt.xticks(rotation=45)
        plt.tight_layout()
        plt.show()


def analyze_trends():
    defects = load_defects()
    test_runs = load_test_runs()
    
    analyzer = TrendAnalyzer(defects, test_runs)
    
    print("=" * 50)
    print("АНАЛИЗ ТРЕНДОВ")
    print("=" * 50)
    
    detection_rate = analyzer.calculate_defect_detection_rate()
    print(f"\nСкорость обнаружения дефектов: {detection_rate:.2f} дефектов/день")
    
    fix_rate = analyzer.calculate_fix_rate()
    print(f"Среднее время исправления дефекта: {fix_rate:.1f} часов")
    
    # Накопленное количество
    cumulative = analyzer.get_cumulative_defects()
    if cumulative:
        print(f"\nДинамика обнаружения дефектов:")
        for day, count in cumulative:
            print(f"  {day}: {count} дефектов (накоплено)")


if __name__ == "__main__":
    analyze_trends()
Задание 2.2. Анализ надёжности (15 минут)
python
class ReliabilityAnalyzer:
    """Анализ надёжности ПО."""
    
    def __init__(self, defects: List[Defect], test_runs: List[TestRun]):
        self.defects = defects
        self.test_runs = test_runs
    
    def get_production_defects(self) -> List[Defect]:
        """Возвращает дефекты, найденные в продакшене."""
        return [d for d in self.defects if d.stage_found == "production"]
    
    def calculate_mtbf(self, total_operating_hours: float = 720) -> float:
        """
        Рассчитывает среднее время между отказами.
        
        Args:
            total_operating_hours: Общее время работы (часы)
        """
        production_defects = self.get_production_defects()
        if not production_defects:
            return total_operating_hours
        
        return total_operating_hours / len(production_defects)
    
    def calculate_defect_leakage(self) -> float:
        """
        Рассчитывает процент дефектов, утекших в продакшн.
        """
        total = len(self.defects)
        production = len(self.get_production_defects())
        return (production / total * 100) if total > 0 else 0
    
    def calculate_test_pass_rate_trend(self) -> List[dict]:
        """Анализирует тренд проходимости тестов."""
        trend = []
        for run in self.test_runs:
            pass_rate = (run.passed / run.total_tests) * 100
            trend.append({
                "date": run.date,
                "pass_rate": pass_rate,
                "failed": run.failed
            })
        return trend
    
    def calculate_execution_time_trend(self) -> List[dict]:
        """Анализирует тренд времени выполнения тестов."""
        return [{"date": run.date, "time": run.execution_time_seconds} for run in self.test_runs]


def analyze_reliability():
    defects = load_defects()
    test_runs = load_test_runs()
    
    analyzer = ReliabilityAnalyzer(defects, test_runs)
    
    print("=" * 50)
    print("АНАЛИЗ НАДЁЖНОСТИ")
    print("=" * 50)
    
    # Дефекты в продакшене
    prod_defects = analyzer.get_production_defects()
    print(f"\nДефектов, обнаруженных в продакшене: {len(prod_defects)}")
    if prod_defects:
        for d in prod_defects:
            print(f"  - {d.title} ({d.severity})")
    
    # MTBF
    mtbf = analyzer.calculate_mtbf(total_operating_hours=720)
    print(f"\nСреднее время между отказами (MTBF): {mtbf:.1f} часов")
    
    # Утечка дефектов
    leakage = analyzer.calculate_defect_leakage()
    print(f"Утечка дефектов в продакшн: {leakage:.1f}%")
    
    if leakage < 5:
        print("✅ Низкий уровень утечки дефектов")
    elif leakage < 15:
        print("⚠️ Средний уровень утечки дефектов")
    else:
        print("❌ Высокий уровень утечки дефектов — требуется улучшение тестирования")


if __name__ == "__main__":
    analyze_reliability()
Задание 2.3. Анализ тренда тестов (10 минут)
python
def analyze_test_trends():
    """Анализ тренда проходимости тестов."""
    test_runs = load_test_runs()
    analyzer = ReliabilityAnalyzer([], test_runs)
    
    print("=" * 50)
    print("ТРЕНД ПРОХОДИМОСТИ ТЕСТОВ")
    print("=" * 50)
    
    pass_rate_trend = analyzer.calculate_test_pass_rate_trend()
    
    print("\nДинамика проходимости тестов:")
    for point in pass_rate_trend:
        print(f"  {point['date'].date()}: {point['pass_rate']:.1f}% (неудачно: {point['failed']})")
    
    if pass_rate_trend:
        first = pass_rate_trend[0]["pass_rate"]
        last = pass_rate_trend[-1]["pass_rate"]
        improvement = last - first
        
        print(f"\nУлучшение проходимости тестов: {improvement:+.1f}%")
        
        if improvement > 0:
            print("✅ Качество улучшается, тесты становятся стабильнее")
        else:
            print("⚠️ Требуется внимание: проходимость тестов не улучшается")
    
    # Время выполнения
    exec_trend = analyzer.calculate_execution_time_trend()
    print(f"\nДинамика времени выполнения тестов:")
    for point in exec_trend:
        print(f"  {point['date'].date()}: {point['time']:.0f} сек")


if __name__ == "__main__":
    analyze_test_trends()
Задание 2.4. Итоговый отчёт (5 минут)
markdown
## Отчёт по расчёту метрик качества (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Тренды дефектов

| Метрика | Значение |
|:---|:---|
| Скорость обнаружения дефектов | _____ дефектов/день |
| Среднее время исправления | _____ часов |
| Дефектов в продакшене | _____ |

### Надёжность

| Метрика | Значение | Оценка |
|:---|:---|:---|
| Утечка дефектов | _____% | _____ |
| MTBF (среднее время между отказами) | _____ часов | _____ |

### Тренды тестирования

| Дата | Проходимость | Время выполнения |
|:---|:---|:---|
| _____ | _____% | _____ сек |
| _____ | _____% | _____ сек |
| _____ | _____% | _____ сек |

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться рассчитывать стоимость исправления дефектов и ROI тестирования.

Задания для сложного уровня
Задание 3.1. Расчёт стоимости исправления дефектов (15 минут)
python
# metrics_advanced.py

from test_metrics_data import load_defects, load_modules, Defect


class CostCalculator:
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
    
    def calculate_defect_cost(self, defect: Defect) -> float:
        """
        Рассчитывает стоимость исправления одного дефекта.
        """
        coefficient = self.STAGE_COEFFICIENTS.get(defect.stage_found, 1000)
        return defect.hours_to_fix * self.hourly_rate * coefficient
    
    def calculate_total_cost(self, defects: List[Defect]) -> float:
        """Рассчитывает общую стоимость исправления всех дефектов."""
        return sum(self.calculate_defect_cost(d) for d in defects)
    
    def calculate_cost_by_stage(self, defects: List[Defect]) -> dict:
        """Рассчитывает стоимость дефектов, сгруппированную по этапам."""
        stage_costs = {stage: 0 for stage in self.STAGE_COEFFICIENTS.keys()}
        
        for defect in defects:
            stage_costs[defect.stage_found] += self.calculate_defect_cost(defect)
        
        return stage_costs
    
    def calculate_cost_saving(self, defects: List[Defect]) -> dict:
        """
        Рассчитывает экономию от раннего обнаружения.
        """
        total_actual = self.calculate_total_cost(defects)
        
        # Считаем, сколько бы стоили те же дефекты, если бы дошли до продакшена
        production_cost = sum(
            defect.hours_to_fix * self.hourly_rate * self.STAGE_COEFFICIENTS["production"]
            for defect in defects
        )
        
        saved = production_cost - total_actual
        
        return {
            "actual_cost": total_actual,
            "potential_production_cost": production_cost,
            "saved": saved,
            "roi_percent": (saved / total_actual) * 100 if total_actual > 0 else 0
        }


def calculate_costs():
    defects = load_defects()
    calculator = CostCalculator(hourly_rate=100)
    
    print("=" * 50)
    print("СТОИМОСТЬ ИСПРАВЛЕНИЯ ДЕФЕКТОВ")
    print("=" * 50)
    
    # Стоимость по этапам
    stage_costs = calculator.calculate_cost_by_stage(defects)
    print("\nСтоимость исправления дефектов по этапам обнаружения:")
    
    for stage, cost in sorted(stage_costs.items(), key=lambda x: x[1], reverse=True):
        if cost > 0:
            print(f"  {stage:12}: ${cost:10,.2f}")
    
    # Общая стоимость
    total_cost = calculator.calculate_total_cost(defects)
    print(f"\nОбщая стоимость исправления дефектов: ${total_cost:,.2f}")
    
    # Экономия
    saving = calculator.calculate_cost_saving(defects)
    print(f"\nПотенциальная стоимость в продакшене: ${saving['potential_production_cost']:,.2f}")
    print(f"Экономия от раннего обнаружения: ${saving['saved']:,.2f}")
    print(f"ROI тестирования: {saving['roi_percent']:.0f}%")


if __name__ == "__main__":
    calculate_costs()
Задание 3.2. Комплексный анализ и рекомендации (20 минут)
python
class QualityReportGenerator:
    """Генератор полного отчёта о качестве."""
    
    def __init__(self, defects: List[Defect], modules: List[ModuleInfo], test_runs: List[TestRun]):
        self.defects = defects
        self.modules = modules
        self.test_runs = test_runs
        self.metrics = ModuleMetricsCalculator(defects, modules)
        self.reliability = ReliabilityAnalyzer(defects, test_runs)
        self.cost = CostCalculator(hourly_rate=100)
    
    def generate_complete_report(self) -> dict:
        """Генерирует полный отчёт о качестве."""
        
        # 1. Метрики дефектов
        defects_by_severity = self.metrics.get_defects_by_severity()
        module_densities = self.metrics.calculate_module_defect_density()
        
        # 2. Надёжность
        leakage = self.reliability.calculate_defect_leakage()
        mtbf = self.reliability.calculate_mtbf(720)
        
        # 3. Стоимость
        cost_stats = self.cost.calculate_cost_saving(self.defects)
        
        # 4. Рекомендации
        recommendations = self._generate_recommendations(
            module_densities, leakage, cost_stats
        )
        
        return {
            "defects_summary": {
                "total": len(self.defects),
                "by_severity": defects_by_severity,
                "module_densities": module_densities
            },
            "reliability": {
                "defect_leakage_percent": leakage,
                "mtbf_hours": mtbf
            },
            "cost": cost_stats,
            "recommendations": recommendations
        }
    
    def _generate_recommendations(self, module_densities, leakage, cost_stats) -> List[str]:
        """Генерирует рекомендации на основе метрик."""
        recommendations = []
        
        # Рекомендации по модулям
        high_density_modules = [m for m, d in module_densities.items() if d > 3.0]
        if high_density_modules:
            recommendations.append(
                f"Усилить тестирование модулей: {', '.join(high_density_modules)}"
            )
        
        # Рекомендации по утечке дефектов
        if leakage > 10:
            recommendations.append(
                "Улучшить тестовое покрытие критических сценариев для снижения утечки дефектов"
            )
        
        # Рекомендации по ROI
        if cost_stats["roi_percent"] < 100:
            recommendations.append(
                "Оптимизировать процесс тестирования для повышения ROI"
            )
        
        if not recommendations:
            recommendations.append("Процесс тестирования эффективен. Поддерживать текущий уровень.")
        
        return recommendations
    
    def print_report(self):
        """Печатает отчёт в консоль."""
        report = self.generate_complete_report()
        
        print("=" * 60)
        print("        КОМПЛЕКСНЫЙ ОТЧЁТ О КАЧЕСТВЕ ПО")
        print("=" * 60)
        
        print("\n1. СВОДКА ПО ДЕФЕКТАМ")
        print(f"   Всего дефектов: {report['defects_summary']['total']}")
        print("   По серьёзности:")
        for severity, count in report['defects_summary']['by_severity'].items():
            print(f"     {severity}: {count}")
        
        print("\n2. ПЛОТНОСТЬ ДЕФЕКТОВ ПО МОДУЛЯМ (дефектов/KLOC)")
        for module, density in sorted(report['defects_summary']['module_densities'].items(), 
                                       key=lambda x: x[1], reverse=True):
            status = "⚠️" if density > 3.0 else "✅"
            print(f"   {status} {module:10}: {density:.2f}")
        
        print("\n3. НАДЁЖНОСТЬ")
        print(f"   Утечка дефектов в продакшн: {report['reliability']['defect_leakage_percent']:.1f}%")
        print(f"   MTBF (среднее время между отказами): {report['reliability']['mtbf_hours']:.0f} часов")
        
        print("\n4. ЭКОНОМИЧЕСКИЕ ПОКАЗАТЕЛИ")
        print(f"   Фактическая стоимость исправления: ${report['cost']['actual_cost']:,.2f}")
        print(f"   Потенциальная стоимость в продакшене: ${report['cost']['potential_production_cost']:,.2f}")
        print(f"   Экономия: ${report['cost']['saved']:,.2f}")
        print(f"   ROI тестирования: {report['cost']['roi_percent']:.0f}%")
        
        print("\n5. РЕКОМЕНДАЦИИ")
        for i, rec in enumerate(report['recommendations'], 1):
            print(f"   {i}. {rec}")
        
        print("\n" + "=" * 60)


def generate_full_report():
    defects = load_defects()
    modules = load_modules()
    test_runs = load_test_runs()
    
    reporter = QualityReportGenerator(defects, modules, test_runs)
    reporter.print_report()


if __name__ == "__main__":
    generate_full_report()
Задание 3.3. Итоговый отчёт (5 минут)
markdown
## Комплексный отчёт о качестве ПО

**Студент:** _________________
**Вариант №:** ___ (18-25)
**Дата:** _________________

### Общая оценка качества

| Показатель | Значение | Оценка |
|:---|:---|:---|
| Плотность дефектов | _____ дефектов/KLOC | _____ |
| Утечка дефектов | _____% | _____ |
| MTBF | _____ часов | _____ |
| ROI тестирования | _____% | _____ |

### Анализ по модулям

| Модуль | LOC | Дефекты | Плотность | Статус |
|:---|:---|:---|:---|:---|
| auth | 2500 | _____ | _____ | ☐ |
| payment | 1800 | _____ | _____ | ☐ |
| catalog | 3500 | _____ | _____ | ☐ |
| cart | 1200 | _____ | _____ | ☐ |
| order | 2200 | _____ | _____ | ☐ |

### Экономический анализ

| Показатель | Сумма |
|:---|:---|
| Общая стоимость исправления | $_____ |
| Экономия от раннего обнаружения | $_____ |
| ROI тестирования | _____% |

### Рекомендации по улучшению

1. _______________________________________________________________
2. _______________________________________________________________
3. _______________________________________________________________

### Вывод о готовности к релизу

- [ ] Система готова к релизу
- [ ] Требуется доработка (причина: _______________)

---

**Подпись:** _________________
Карточка студента
text
ПР 3.6. РАСЧЁТ МЕТРИК КАЧЕСТВА НА ОСНОВЕ ТЕСТИРОВАНИЯ

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ Расчёт плотности дефектов (базовый)
□ Плотность по модулям (базовый)
□ Эффективность тестирования (базовый)
□ Анализ трендов (средний)
□ Надёжность (средний)
□ Стоимость исправления (сложный)
□ ROI тестирования (сложный)
□ Рекомендации (сложный)

=== РЕЗУЛЬТАТЫ ===

Общая плотность дефектов: _____
Утечка дефектов: _____%
MTBF: _____ часов
ROI: _____%

=== ВРЕМЯ ВЫПОЛНЕНИЯ ===

Подготовка: _____ мин
Выполнение: _____ мин
Оформление: _____ мин

=== ОТЧЁТ ===

Файлы:
- metrics_base.py
- metrics_intermediate.py
- metrics_advanced.py
- quality_report.json

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Метрики не рассчитаны
3 (удовлетворительно)	Базовый	Плотность дефектов + эффективность (3+ метрики)
4 (хорошо)	Средний	+ Тренды + надёжность (5+ метрик)
5 (отлично)	Сложный	+ Стоимость + ROI + рекомендации (все метрики)
Контрольные вопросы (для защиты)
Что такое плотность дефектов и как её интерпретировать?

Какое значение плотности дефектов считается хорошим?

Как рассчитать утечку дефектов в продакшн?

Что означает MTBF и как его использовать для оценки надёжности?

Почему стоимость исправления дефекта растёт с каждым этапом?

Как рассчитать ROI тестирования?

Какие метрики помогают оценить эффективность тестирования?

Как интерпретировать высокую плотность дефектов в одном модуле?

Какие выводы можно сделать из тренда проходимости тестов?

Как результаты расчёта метрик влияют на решение о релизе?
