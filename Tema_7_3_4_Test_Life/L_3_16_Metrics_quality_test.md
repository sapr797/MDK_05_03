# Лекция 3.16: Метрики качества тестирования: эффективность тестов, ROI тестирования. Оценка экономической эффективности тестирования

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.4:** Тестирование в жизненном цикле ПО  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить метрики качества тестирования, методы оценки эффективности тестов, расчёт ROI тестирования и подходы к оценке экономической эффективности тестирования.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Рассчитывать метрики эффективности тестирования (ОК 01, ОК 02).
2. Оценивать ROI тестирования и экономическую эффективность (ОК 05).
3. Анализировать стоимость качества и упущенную выгоду (ПК 3.3).
4. Принимать обоснованные решения о стратегии тестирования (ОК 02).

---

## 1. Метрики эффективности тестирования

### 1.1 Основные метрики

| Метрика | Формула | Интерпретация |
|:---|:---|:---|
| **DRE (Defect Removal Efficiency)** | (Найдено тестированием / Всего дефектов) × 100% | Эффективность обнаружения дефектов |
| **Плотность дефектов** | Дефекты / KLOC | Качество кода |
| **Утечка дефектов** | (Дефекты в продакшене / Всего дефектов) × 100% | Качество тестирования |
| **TCO (Test Case Effectiveness)** | (Дефекты на тест / Всего тестов) × 100% | Ценность каждого теста |

### 1.2 Расчёт метрик эффективности

```python
class TestingMetrics:
    """Класс для расчёта метрик эффективности тестирования."""
    
    def __init__(self):
        self.defects_found_in_testing = 0
        self.defects_found_in_production = 0
        self.total_test_cases = 0
        self.total_defects = 0
        self.lines_of_code = 0
        self.test_execution_time_hours = 0
        self.test_development_hours = 0
    
    def defect_removal_efficiency(self) -> float:
        """
        DRE = Дефекты, найденные тестированием / Всего дефектов
        """
        if self.total_defects == 0:
            return 100.0
        return (self.defects_found_in_testing / self.total_defects) * 100
    
    def defect_leakage(self) -> float:
        """
        Утечка дефектов = Дефекты в продакшене / Всего дефектов
        """
        if self.total_defects == 0:
            return 0.0
        return (self.defects_found_in_production / self.total_defects) * 100
    
    def defect_density(self) -> float:
        """
        Плотность дефектов на 1000 строк кода
        """
        if self.lines_of_code == 0:
            return 0.0
        return (self.total_defects / self.lines_of_code) * 1000
    
    def test_case_effectiveness(self) -> float:
        """
        TCE = (Дефекты на тест / Всего тестов) × 100%
        """
        if self.total_test_cases == 0:
            return 0.0
        return (self.defects_found_in_testing / self.total_test_cases) * 100
    
    def test_efficiency(self) -> float:
        """
        Эффективность тестирования = Дефекты / часы
        """
        total_hours = self.test_execution_time_hours + self.test_development_hours
        if total_hours == 0:
            return 0.0
        return self.defects_found_in_testing / total_hours


class TestEfficiencyCalculator:
    """Калькулятор эффективности тестирования."""
    
    @staticmethod
    def calculate_dre(defects_tested: int, defects_total: int) -> float:
        """Defect Removal Efficiency."""
        if defects_total == 0:
            return 100.0
        return (defects_tested / defects_total) * 100
    
    @staticmethod
    def calculate_leakage(defects_prod: int, defects_total: int) -> float:
        """Утечка дефектов."""
        if defects_total == 0:
            return 0.0
        return (defects_prod / defects_total) * 100
    
    @staticmethod
    def calculate_density(defects: int, kloc: float) -> float:
        """Плотность дефектов (defects per KLOC)."""
        if kloc == 0:
            return 0.0
        return defects / kloc


# Пример расчёта
metrics = TestingMetrics()
metrics.defects_found_in_testing = 85
metrics.defects_found_in_production = 15
metrics.total_defects = 100
metrics.total_test_cases = 500
metrics.lines_of_code = 50000

print(f"DRE: {metrics.defect_removal_efficiency():.1f}%")
print(f"Defect Leakage: {metrics.defect_leakage():.1f}%")
print(f"Defect Density: {metrics.defect_density():.1f} per KLOC")
print(f"Test Case Effectiveness: {metrics.test_case_effectiveness():.1f}%")
Пример вывода:

text
DRE: 85.0%
Defect Leakage: 15.0%
Defect Density: 2.0 per KLOC
Test Case Effectiveness: 17.0%
2. ROI тестирования
2.1 Формула ROI
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ROI (Return on Investment) ТЕСТИРОВАНИЯ                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ROI = (Экономия от предотвращённых дефектов - Стоимость тестирования)    │
│          / Стоимость тестирования × 100%                                    │
│                                                                              │
│   Экономия = Количество дефектов × Стоимость исправления на этапе           │
│                                                                              │
│   Стоимость тестирования = Затраты на:                                       │
│   • Написание тестов                                                        │
│   • Выполнение тестов                                                       │
│   • Инструменты                                                             │
│   • Обучение                                                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
2.2 Расчёт ROI
python
class ROICalculator:
    """Калькулятор ROI тестирования."""
    
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
    
    def calculate_defect_cost(self, hours_to_fix: float, stage: str) -> float:
        """Стоимость исправления дефекта на заданном этапе."""
        coefficient = self.STAGE_COEFFICIENTS.get(stage, 1000)
        return hours_to_fix * self.hourly_rate * coefficient
    
    def calculate_savings(
        self,
        defects_found: int,
        avg_fix_hours: float,
        stage_found: str
    ) -> float:
        """
        Экономия от обнаружения дефектов на раннем этапе.
        (по сравнению с исправлением в продакшене)
        """
        cost_production = self.calculate_defect_cost(avg_fix_hours, "production")
        cost_actual = self.calculate_defect_cost(avg_fix_hours, stage_found)
        return defects_found * (cost_production - cost_actual)
    
    def calculate_roi(
        self,
        testing_cost: float,
        defects_found: int,
        avg_fix_hours: float,
        stage_found: str
    ) -> dict:
        """
        Расчёт ROI тестирования.
        """
        savings = self.calculate_savings(defects_found, avg_fix_hours, stage_found)
        net_gain = savings - testing_cost
        roi_percent = (net_gain / testing_cost) * 100 if testing_cost > 0 else 0
        
        return {
            "testing_cost": testing_cost,
            "savings": savings,
            "net_gain": net_gain,
            "roi_percent": roi_percent
        }


# Пример расчёта
calculator = ROICalculator(hourly_rate=100)

# Сценарий: 50 дефектов найдено на этапе тестирования
result = calculator.calculate_roi(
    testing_cost=50000,
    defects_found=50,
    avg_fix_hours=4,
    stage_found="testing"
)

print(f"Testing cost: ${result['testing_cost']:,.2f}")
print(f"Savings (vs production): ${result['savings']:,.2f}")
print(f"Net gain: ${result['net_gain']:,.2f}")
print(f"ROI: {result['roi_percent']:.0f}%")
Пример вывода:

text
Testing cost: $50,000.00
Savings (vs production): $1,980,000.00
Net gain: $1,930,000.00
ROI: 3860%
3. Стоимость качества (CoQ)
3.1 Компоненты стоимости качества
text
┌─────────────────────────────────────────────────────────────────────────────┐
│                    СТОИМОСТЬ КАЧЕСТВА (Cost of Quality)                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   СТОИМОСТЬ КАЧЕСТВА = CoGQ + CoPQ                                          │
│                                                                              │
│   CoGQ (Cost of Good Quality) — стоимость хорошего качества                 │
│   ├── Предупредительные затраты (Prevention)                                │
│   │   • Обучение, код-ревью, статический анализ                            │
│   └── Оценочные затраты (Appraisal)                                         │
│       • Тестирование, инспекции, аудиты                                     │
│                                                                              │
│   CoPQ (Cost of Poor Quality) — стоимость плохого качества                  │
│   ├── Внутренние отказы (Internal Failure)                                  │
│   │   • Исправление дефектов, переделка, простой                            │
│   └── Внешние отказы (External Failure)                                     │
│       • Поддержка, возвраты, потеря репутации, штрафы                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
3.2 Расчёт стоимости качества
python
class CostOfQualityCalculator:
    """Калькулятор стоимости качества."""
    
    def __init__(self):
        self.prevention_costs = 0.0      # обучение, инструменты
        self.appraisal_costs = 0.0       # тестирование, инспекции
        self.internal_failure_costs = 0.0  # исправление багов, переделка
        self.external_failure_costs = 0.0  # поддержка, возвраты, штрафы
    
    def add_prevention_cost(self, amount: float):
        self.prevention_costs += amount
    
    def add_appraisal_cost(self, amount: float):
        self.appraisal_costs += amount
    
    def add_internal_failure_cost(self, amount: float):
        self.internal_failure_costs += amount
    
    def add_external_failure_cost(self, amount: float):
        self.external_failure_costs += amount
    
    def cogq(self) -> float:
        """Cost of Good Quality."""
        return self.prevention_costs + self.appraisal_costs
    
    def copq(self) -> float:
        """Cost of Poor Quality."""
        return self.internal_failure_costs + self.external_failure_costs
    
    def total_coq(self) -> float:
        """Total Cost of Quality."""
        return self.cogq() + self.copq()
    
    def quality_roi(self) -> float:
        """
        ROI от инвестиций в качество.
        """
        if self.cogq() == 0:
            return 0.0
        return ((self.copq_avoided or self.copq()) / self.cogq()) * 100


# Пример
coq = CostOfQualityCalculator()
coq.add_prevention_cost(20000)      # обучение, лицензии
coq.add_appraisal_cost(50000)       # команда тестирования
coq.add_internal_failure_cost(30000)  # исправление багов
coq.add_external_failure_cost(100000) # поддержка продакшена

print(f"CoGQ (Prevention + Appraisal): ${coq.cogq():,.2f}")
print(f"CoPQ (Internal + External): ${coq.copq():,.2f}")
print(f"Total CoQ: ${coq.total_coq():,.2f}")
4. Оценка экономической эффективности
4.1 Модель экономической эффективности
python
class EconomicEfficiencyAnalyzer:
    """Анализатор экономической эффективности тестирования."""
    
    def __init__(self):
        self.data = {
            "defects": {
                "found_in_testing": 0,
                "found_in_production": 0,
                "prevented": 0
            },
            "costs": {
                "testing_team": 0,
                "tools": 0,
                "infrastructure": 0,
                "training": 0
            },
            "benefits": {
                "reduced_support": 0,
                "reduced_development": 0,
                "revenue_protected": 0
            }
        }
    
    def calculate_total_costs(self) -> float:
        """Общие затраты на тестирование."""
        return sum(self.data["costs"].values())
    
    def calculate_total_benefits(self) -> float:
        """Общие выгоды от тестирования."""
        return sum(self.data["benefits"].values())
    
    def calculate_net_benefit(self) -> float:
        """Чистая выгода."""
        return self.calculate_total_benefits() - self.calculate_total_costs()
    
    def calculate_benefit_cost_ratio(self) -> float:
        """Соотношение выгоды и затрат (BCR)."""
        total_costs = self.calculate_total_costs()
        if total_costs == 0:
            return 0.0
        return self.calculate_total_benefits() / total_costs
    
    def calculate_payback_period(self, monthly_savings: float) -> float:
        """Срок окупаемости (месяцы)."""
        total_costs = self.calculate_total_costs()
        if monthly_savings == 0:
            return float('inf')
        return total_costs / monthly_savings
    
    def generate_report(self) -> dict:
        """Генерация отчёта."""
        total_costs = self.calculate_total_costs()
        total_benefits = self.calculate_total_benefits()
        
        # Расчёт DRE
        total_defects = (self.data["defects"]["found_in_testing"] + 
                        self.data["defects"]["found_in_production"])
        dre = (self.data["defects"]["found_in_testing"] / total_defects * 100) if total_defects > 0 else 0
        
        return {
            "total_costs": total_costs,
            "total_benefits": total_benefits,
            "net_benefit": total_benefits - total_costs,
            "benefit_cost_ratio": self.calculate_benefit_cost_ratio(),
            "defect_removal_efficiency": dre,
            "defects_prevented": self.data["defects"]["prevented"]
        }


# Пример использования
analyzer = EconomicEfficiencyAnalyzer()
analyzer.data["costs"]["testing_team"] = 150000
analyzer.data["costs"]["tools"] = 20000
analyzer.data["costs"]["infrastructure"] = 10000
analyzer.data["benefits"]["reduced_support"] = 80000
analyzer.data["benefits"]["reduced_development"] = 120000
analyzer.data["benefits"]["revenue_protected"] = 500000
analyzer.data["defects"]["found_in_testing"] = 120
analyzer.data["defects"]["found_in_production"] = 8

report = analyzer.generate_report()
print(f"Total Costs: ${report['total_costs']:,.2f}")
print(f"Total Benefits: ${report['total_benefits']:,.2f}")
print(f"Net Benefit: ${report['net_benefit']:,.2f}")
print(f"Benefit-Cost Ratio: {report['benefit_cost_ratio']:.1f}")
print(f"DRE: {report['defect_removal_efficiency']:.1f}%")
5. Практические примеры
5.1 Сравнение сценариев тестирования
python
class TestingStrategyComparison:
    """Сравнение экономической эффективности стратегий тестирования."""
    
    def __init__(self):
        self.scenarios = {}
    
    def add_scenario(self, name: str, data: dict):
        self.scenarios[name] = data
    
    def compare(self) -> dict:
        """Сравнение стратегий."""
        results = {}
        for name, data in self.scenarios.items():
            testing_cost = data.get("testing_cost", 0)
            defects_found = data.get("defects_found", 0)
            avg_fix_cost = data.get("avg_fix_cost", 1000)
            
            savings = defects_found * avg_fix_cost
            net_saving = savings - testing_cost
            roi = (net_saving / testing_cost) * 100 if testing_cost > 0 else 0
            
            results[name] = {
                "testing_cost": testing_cost,
                "savings": savings,
                "net_saving": net_saving,
                "roi": roi
            }
        return results


# Сравнение трёх стратегий
comparison = TestingStrategyComparison()

# Стратегия A: Минимальное тестирование
comparison.add_scenario("Minimal Testing", {
    "testing_cost": 10000,
    "defects_found": 20,
    "avg_fix_cost": 5000  # дорогое исправление в продакшене
})

# Стратегия B: Стандартное тестирование
comparison.add_scenario("Standard Testing", {
    "testing_cost": 50000,
    "defects_found": 80,
    "avg_fix_cost": 5000
})

# Стратегия C: Расширенное тестирование (автоматизация)
comparison.add_scenario("Advanced Testing", {
    "testing_cost": 150000,
    "defects_found": 100,
    "avg_fix_cost": 5000
})

results = comparison.compare()
for name, data in results.items():
    print(f"\n{name}:")
    print(f"  Cost: ${data['testing_cost']:,.2f}")
    print(f"  Savings: ${data['savings']:,.2f}")
    print(f"  Net: ${data['net_saving']:,.2f}")
    print(f"  ROI: {data['roi']:.0f}%")
6. Шпаргалка
python
# === МЕТРИКИ ЭФФЕКТИВНОСТИ ===
dre = (defects_tested / defects_total) * 100
leakage = (defects_prod / defects_total) * 100
density = defects_total / (lines_of_code / 1000)

# === ROI ===
savings = defects_found * cost_per_defect
roi = ((savings - testing_cost) / testing_cost) * 100

# === СТОИМОСТЬ КАЧЕСТВА ===
cogq = prevention_costs + appraisal_costs
copq = internal_failure_costs + external_failure_costs
total_coq = cogq + copq

# === BCR (Benefit-Cost Ratio) ===
bcr = total_benefits / total_costs
Контрольные вопросы
Что такое DRE и как его интерпретировать?

Как рассчитать утечку дефектов?

Какое значение плотности дефектов считается хорошим?

Что показывает ROI тестирования?

Как рассчитать ROI тестирования?

Из каких компонентов состоит стоимость качества?

Чем отличаются внутренние и внешние отказы?

Что такое Benefit-Cost Ratio (BCR)?

Как оценить экономическую эффективность тестирования?

Как сравнить разные стратегии тестирования?

Следующее занятие: ПР 3.13 — Практический расчёт метрик и ROI тестирования.
