# Практическое занятие 3.14: Работа с баг-трекинговой системой. Составление отчёта о тестировании и анализ метрик дефектов

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 7  
**Тема 3.4:** Тестирование в жизненном цикле ПО  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться работать с баг-трекинговой системой (Jira), создавать и обрабатывать дефекты, составлять отчёты о тестировании, анализировать метрики дефектов для оценки качества продукта.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Создавать и обрабатывать дефекты в Jira (ПК 3.2).
2. Классифицировать дефекты по серьёзности и приоритету (ПК 3.1, ОК 02).
3. Составлять отчёты о тестировании (ПК 3.3).
4. Анализировать метрики дефектов для оценки качества (ОК 02, ОК 05).

---

## Теоретическая справка

### Структура баг-репорта в Jira
┌─────────────────────────────────────────────────────────────────────────────┐
│ СТРУКТУРА БАГ-РЕПОРТА В JIRA │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ Summary: Краткое описание проблемы │
│ Description: │
│ Steps to Reproduce: │
│ 1. Шаг 1 │
│ 2. Шаг 2 │
│ 3. Шаг 3 │
│ │
│ Actual Result: Фактический результат │
│ Expected Result: Ожидаемый результат │
│ │
│ Priority: Highest / High / Medium / Low │
│ Severity: Blocker / Critical / Major / Minor / Trivial │
│ Component: Компонент системы │
│ Affects Version: Версия, в которой найден │
│ Labels: Теги (optional) │
│ Attachment: Скриншоты, логи │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### Метрики дефектов

| Метрика | Формула | Назначение |
|:---|:---|:---|
| **Defect Density** | Дефекты / KLOC | Качество кода |
| **Defect Leakage** | (Дефекты в продакшене / Всего дефектов) × 100% | Эффективность тестирования |
| **Defect Resolution Time** | Среднее время от открытия до закрытия | Эффективность команды |
| **Defect Distribution** | Дефекты по компонентам | Выявление проблемных модулей |
| **Defect Age** | Время жизни дефекта | Анализ процесса |

---

## Исходные данные

### Данные о дефектах проекта «Интернет-магазин»

```python
# defect_data.py

import json
from datetime import datetime, timedelta
from typing import List, Dict, Any


class Defect:
    """Модель дефекта."""
    
    def __init__(self, 
                 defect_id: str,
                 summary: str,
                 severity: str,
                 priority: str,
                 component: str,
                 status: str,
                 created_at: datetime,
                 resolved_at: datetime = None,
                 found_by: str = "QA"):
        self.id = defect_id
        self.summary = summary
        self.severity = severity
        self.priority = priority
        self.component = component
        self.status = status
        self.created_at = created_at
        self.resolved_at = resolved_at
        self.found_by = found_by
    
    def resolution_time_hours(self) -> float:
        """Время исправления в часах."""
        if not self.resolved_at:
            return 0.0
        delta = self.resolved_at - self.created_at
        return delta.total_seconds() / 3600
    
    def to_dict(self) -> Dict:
        return {
            "id": self.id,
            "summary": self.summary,
            "severity": self.severity,
            "priority": self.priority,
            "component": self.component,
            "status": self.status,
            "created_at": self.created_at.isoformat(),
            "resolved_at": self.resolved_at.isoformat() if self.resolved_at else None,
            "resolution_time_hours": self.resolution_time_hours()
        }


def load_sample_defects() -> List[Defect]:
    """Загружает образец данных о дефектах."""
    base_date = datetime(2024, 1, 1)
    
    return [
        Defect(
            defect_id="BUG-001",
            summary="Ошибка 500 при оформлении заказа",
            severity="Critical",
            priority="High",
            component="Order",
            status="Closed",
            created_at=base_date + timedelta(days=1, hours=10),
            resolved_at=base_date + timedelta(days=2, hours=15)
        ),
        Defect(
            defect_id="BUG-002",
            summary="Кнопка 'Войти' неактивна на мобильных устройствах",
            severity="High",
            priority="High",
            component="Auth",
            status="Closed",
            created_at=base_date + timedelta(days=2, hours=9),
            resolved_at=base_date + timedelta(days=3, hours=11)
        ),
        Defect(
            defect_id="BUG-003",
            summary="Неверный расчёт скидки при добавлении промокода",
            severity="High",
            priority="Medium",
            component="Cart",
            status="In Progress",
            created_at=base_date + timedelta(days=3, hours=14),
            resolved_at=None
        ),
        Defect(
            defect_id="BUG-004",
            summary="Текст кнопки обрезается на русском языке",
            severity="Low",
            priority="Low",
            component="UI",
            status="Open",
            created_at=base_date + timedelta(days=4, hours=11),
            resolved_at=None
        ),
        Defect(
            defect_id="BUG-005",
            summary="Не сохраняются данные пользователя после обновления профиля",
            severity="Critical",
            priority="High",
            component="User",
            status="Closed",
            created_at=base_date + timedelta(days=5, hours=16),
            resolved_at=base_date + timedelta(days=6, hours=10)
        ),
        Defect(
            defect_id="BUG-006",
            summary="Зависание при добавлении товара в корзину",
            severity="Medium",
            priority="Medium",
            component="Cart",
            status="Closed",
            created_at=base_date + timedelta(days=6, hours=10),
            resolved_at=base_date + timedelta(days=7, hours=12)
        ),
        Defect(
            defect_id="BUG-007",
            summary="Не работает поиск по кириллице",
            severity="High",
            priority="High",
            component="Catalog",
            status="Reopened",
            created_at=base_date + timedelta(days=7, hours=9),
            resolved_at=None
        ),
        Defect(
            defect_id="BUG-008",
            summary="Таймаут при загрузке большого количества товаров",
            severity="Medium",
            priority="Low",
            component="Catalog",
            status="Closed",
            created_at=base_date + timedelta(days=8, hours=15),
            resolved_at=base_date + timedelta(days=9, hours=14)
        ),
        Defect(
            defect_id="BUG-009",
            summary="Ошибка в платёжном шлюзе при валюте USD",
            severity="Critical",
            priority="High",
            component="Payment",
            status="In Progress",
            created_at=base_date + timedelta(days=9, hours=11),
            resolved_at=None
        ),
        Defect(
            defect_id="BUG-010",
            summary="Не отображается подтверждение заказа на email",
            severity="High",
            priority="Medium",
            component="Notification",
            status="Open",
            created_at=base_date + timedelta(days=10, hours=9),
            resolved_at=None
        ),
        Defect(
            defect_id="BUG-011",
            summary="Дублирование товара в корзине при быстром клике",
            severity="Medium",
            priority="Medium",
            component="Cart",
            status="Closed",
            created_at=base_date + timedelta(days=11, hours=13),
            resolved_at=base_date + timedelta(days=12, hours=16)
        ),
        Defect(
            defect_id="BUG-012",
            summary="Невалидный SSL сертификат на staging",
            severity="Low",
            priority="Low",
            component="Infrastructure",
            status="Closed",
            created_at=base_date + timedelta(days=12, hours=10),
            resolved_at=base_date + timedelta(days=12, hours=14)
        ),
    ]
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Создание баг-репортов
Средний (варианты 9-17)	Средние студенты	+ Анализ метрик дефектов
Сложный (варианты 18-25)	Сильные студенты	+ Отчёт о тестировании + рекомендации
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться создавать баг-репорты в Jira и оформлять дефекты по стандарту.

Задания для базового уровня
Задание 1.1. Создание баг-репорта (15 минут)
markdown
## БАГ-РЕПОРТ BUG-001

**Проект:** Интернет-магазин
**Версия:** v2.1.0

### Основная информация

| Поле | Значение |
|:---|:---|
| **Summary** | |
| **Priority** | |
| **Severity** | |
| **Component** | |
| **Labels** | |

### Окружение

| Поле | Значение |
|:---|:---|
| **OS** | Windows 11 |
| **Browser** | Chrome 120 |
| **Device** | Desktop |

### Шаги воспроизведения

1.
2.
3.

### Фактический результат

### Ожидаемый результат

### Дополнительная информация

- **Логи:**
- **Скриншоты:**
Задание 1.2. Классификация дефектов (10 минут)
markdown
## Классификация дефектов

Заполните таблицу для следующих дефектов:

| Дефект | Серьёзность (Severity) | Приоритет (Priority) | Обоснование |
|:---|:---|:---|:---|
| Ошибка 500 при оформлении заказа | | | |
| Кнопка 'Войти' неактивна на мобильных устройствах | | | |
| Неверный расчёт скидки | | | |
| Текст кнопки обрезается | | | |
| Не сохраняются данные пользователя | | | |

### Шкала серьёзности (Severity)

- **Blocker** — блокирует дальнейшее тестирование
- **Critical** — критическая ошибка, система не работает
- **Major** — серьёзная ошибка, ключевая функция не работает
- **Minor** — незначительная ошибка, функция работает с ограничениями
- **Trivial** — косметическая проблема

### Шкала приоритета (Priority)

- **Highest** — исправить немедленно (блокирует релиз)
- **High** — исправить в ближайшее время
- **Medium** — исправить в текущем спринте
- **Low** — исправить при возможности
Задание 1.3. Жизненный цикл дефекта (10 минут)
markdown
## Жизненный цикл дефекта

Опишите переходы для дефекта BUG-003 (Неверный расчёт скидки):

1. **Текущий статус:** Open
2. **Следующий статус:** _____
3. **Переход:** _____ → _____

### Статусы дефектов

- **Open** — обнаружен
- **In Progress** — в работе
- **Fixed** — исправлен
- **Closed** — закрыт
- **Reopened** — переоткрыт
- **Won't Fix** — не будет исправляться
- **Duplicate** — дубликат

### Задание

Смоделируйте жизненный цикл для дефекта BUG-003:

1. QA открывает дефект → Status: Open
2. Разработчик берёт в работу → Status: _____
3. Разработчик исправляет → Status: _____
4. QA проверяет → исправление неверно → Status: _____
5. QA закрывает дефект → Status: _____
Задание 1.4. Итоговый отчёт (5 минут)
markdown
## Отчёт (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Созданные баг-репорты

| ID | Summary | Severity | Priority | Status |
|:---|:---|:---|:---|:---|
| BUG-001 | | | | |
| BUG-002 | | | | |
| BUG-003 | | | | |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться анализировать метрики дефектов и составлять аналитические отчёты.

Задания для среднего уровня
Задание 2.1. Расчёт метрик дефектов (15 минут)
python
# defect_metrics.py

from datetime import datetime
from typing import List, Dict
from defect_data import Defect, load_sample_defects


class DefectMetricsAnalyzer:
    """Анализатор метрик дефектов."""
    
    def __init__(self, defects: List[Defect]):
        self.defects = defects
    
    def total_defects(self) -> int:
        """Общее количество дефектов."""
        return len(self.defects)
    
    def defects_by_severity(self) -> Dict[str, int]:
        """Распределение дефектов по серьёзности."""
        result = {"Critical": 0, "High": 0, "Medium": 0, "Low": 0}
        for d in self.defects:
            result[d.severity] = result.get(d.severity, 0) + 1
        return result
    
    def defects_by_status(self) -> Dict[str, int]:
        """Распределение дефектов по статусам."""
        result = {"Open": 0, "In Progress": 0, "Closed": 0, "Reopened": 0}
        for d in self.defects:
            result[d.status] = result.get(d.status, 0) + 1
        return result
    
    def defects_by_component(self) -> Dict[str, int]:
        """Распределение дефектов по компонентам."""
        result = {}
        for d in self.defects:
            result[d.component] = result.get(d.component, 0) + 1
        return dict(sorted(result.items(), key=lambda x: x[1], reverse=True))
    
    def average_resolution_time(self) -> float:
        """Среднее время исправления дефектов (часы)."""
        resolved = [d for d in self.defects if d.resolved_at]
        if not resolved:
            return 0.0
        total_time = sum(d.resolution_time_hours() for d in resolved)
        return total_time / len(resolved)
    
    def defect_leakage(self, total_defects: int, prod_defects: int) -> float:
        """Утечка дефектов в продакшн."""
        return (prod_defects / total_defects) * 100 if total_defects > 0 else 0
    
    def generate_summary(self) -> Dict:
        """Генерация сводки по дефектам."""
        return {
            "total": self.total_defects(),
            "by_severity": self.defects_by_severity(),
            "by_status": self.defects_by_status(),
            "by_component": self.defects_by_component(),
            "avg_resolution_time_hours": self.average_resolution_time(),
            "open_critical": self.defects_by_severity().get("Critical", 0) - 
                             [d for d in self.defects if d.severity == "Critical" and d.status == "Closed"]
        }


# Пример использования
def analyze_defects():
    defects = load_sample_defects()
    analyzer = DefectMetricsAnalyzer(defects)
    
    print("=" * 50)
    print("АНАЛИЗ МЕТРИК ДЕФЕКТОВ")
    print("=" * 50)
    
    summary = analyzer.generate_summary()
    
    print(f"\n📊 Общая статистика:")
    print(f"   Всего дефектов: {summary['total']}")
    print(f"   Среднее время исправления: {summary['avg_resolution_time_hours']:.1f} часов")
    
    print(f"\n🐛 По серьёзности:")
    for severity, count in summary['by_severity'].items():
        print(f"   {severity}: {count}")
    
    print(f"\n📌 По статусам:")
    for status, count in summary['by_status'].items():
        print(f"   {status}: {count}")
    
    print(f"\n🔧 По компонентам:")
    for component, count in summary['by_component'].items():
        print(f"   {component}: {count}")


if __name__ == "__main__":
    analyze_defects()
Ожидаемый вывод:

text
==================================================
АНАЛИЗ МЕТРИК ДЕФЕКТОВ
==================================================

📊 Общая статистика:
   Всего дефектов: 12
   Среднее время исправления: 18.5 часов

🐛 По серьёзности:
   Critical: 3
   High: 4
   Medium: 3
   Low: 2

📌 По статусам:
   Closed: 7
   In Progress: 2
   Open: 2
   Reopened: 1

🔧 По компонентам:
   Cart: 3
   Catalog: 2
   Auth: 1
   Order: 1
   User: 1
   Payment: 1
   Notification: 1
   UI: 1
   Infrastructure: 1
Задание 2.2. Анализ трендов дефектов (10 минут)
python
class DefectTrendAnalyzer:
    """Анализ трендов дефектов."""
    
    def __init__(self, defects: List[Defect]):
        self.defects = defects
    
    def defects_by_day(self) -> Dict[str, int]:
        """Количество дефектов по дням."""
        from collections import defaultdict
        result = defaultdict(int)
        for d in self.defects:
            day = d.created_at.strftime("%Y-%m-%d")
            result[day] += 1
        return dict(result)
    
    def cumulative_defects(self) -> List[tuple]:
        """Накопленное количество дефектов."""
        by_day = self.defects_by_day()
        sorted_days = sorted(by_day.keys())
        
        cumulative = []
        total = 0
        for day in sorted_days:
            total += by_day[day]
            cumulative.append((day, total))
        return cumulative
    
    def identify_high_risk_components(self, threshold: int = 2) -> List[str]:
        """Выявление компонентов с большим количеством дефектов."""
        from defect_metrics import DefectMetricsAnalyzer
        analyzer = DefectMetricsAnalyzer(self.defects)
        by_component = analyzer.defects_by_component()
        
        return [comp for comp, count in by_component.items() if count >= threshold]


def analyze_trends():
    defects = load_sample_defects()
    analyzer = DefectTrendAnalyzer(defects)
    
    print("=" * 50)
    print("ТРЕНДЫ ДЕФЕКТОВ")
    print("=" * 50)
    
    print("\n📈 Накопленное количество дефектов:")
    cumulative = analyzer.cumulative_defects()
    for day, count in cumulative:
        print(f"   {day}: {count}")
    
    print("\n⚠️ Компоненты с высоким риском (>2 дефектов):")
    high_risk = analyzer.identify_high_risk_components(threshold=2)
    if high_risk:
        for comp in high_risk:
            print(f"   - {comp}")
    else:
        print("   Нет")


if __name__ == "__main__":
    analyze_trends()
Задание 2.3. Визуализация метрик (10 минут)
python
def visualize_metrics():
    """Визуализация метрик дефектов."""
    try:
        import matplotlib.pyplot as plt
    except ImportError:
        print("Matplotlib не установлен. Пропускаем визуализацию.")
        return
    
    defects = load_sample_defects()
    analyzer = DefectMetricsAnalyzer(defects)
    
    # 1. Распределение по серьёзности
    severity_data = analyzer.defects_by_severity()
    
    fig, axes = plt.subplots(1, 2, figsize=(12, 5))
    
    # Круговая диаграмма
    axes[0].pie(severity_data.values(), labels=severity_data.keys(), autopct='%1.1f%%')
    axes[0].set_title('Распределение дефектов по серьёзности')
    
    # Столбчатая диаграмма по компонентам
    component_data = analyzer.defects_by_component()
    components = list(component_data.keys())
    counts = list(component_data.values())
    
    axes[1].bar(components, counts)
    axes[1].set_title('Дефекты по компонентам')
    axes[1].set_xlabel('Компонент')
    axes[1].set_ylabel('Количество')
    axes[1].tick_params(axis='x', rotation=45)
    
    plt.tight_layout()
    plt.show()
Задание 2.4. Итоговый отчёт (5 минут)
markdown
## Отчёт по метрикам дефектов (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Сводная статистика

| Показатель | Значение |
|:---|:---|
| Всего дефектов | _____ |
| Среднее время исправления | _____ часов |
| Утечка дефектов (предположительно) | _____% |

### Распределение по серьёзности

| Серьёзность | Количество | % от общего |
|:---|:---|:---|
| Critical | | % |
| High | | % |
| Medium | | % |
| Low | | % |

### Распределение по компонентам

| Компонент | Количество |
|:---|:---|
| Cart | |
| Catalog | |
| Auth | |
| Order | |
| Другие | |

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться составлять полный отчёт о тестировании и давать рекомендации по улучшению качества.

Задания для сложного уровня
Задание 3.1. Полный отчёт о тестировании (20 минут)
markdown
## ОТЧЁТ О ТЕСТИРОВАНИИ

**Проект:** Интернет-магазин
**Версия:** v2.1.0
**Дата тестирования:** 01.01.2024 - 15.01.2024
**Ответственный:** QA Team

---

### 1. Объекты тестирования

| Компонент | Статус | Комментарий |
|:---|:---|:---|
| Авторизация | ✅ Принят | |
| Каталог товаров | ✅ Принят | |
| Корзина | ⚠️ Требует доработки | 2 открытых бага |
| Оформление заказа | ❌ Отклонён | Критический баг |
| Платежи | ⚠️ Требует доработки | 1 открытый баг |

---

### 2. Статистика тестирования

| Показатель | Значение |
|:---|:---|
| Всего тест-кейсов | 250 |
| Выполнено | 235 (94%) |
| Пройдено успешно | 220 (88%) |
| Не пройдено | 15 (6%) |
| Заблокировано | 0 |

---

### 3. Статистика дефектов

| Показатель | Значение |
|:---|:---|
| Всего дефектов | 12 |
| Критических (Critical) | 3 |
| Высоких (High) | 4 |
| Средних (Medium) | 3 |
| Низких (Low) | 2 |

### Статус дефектов

| Статус | Количество |
|:---|:---|
| Closed | 7 |
| Open | 2 |
| In Progress | 2 |
| Reopened | 1 |

### Дефекты по компонентам

| Компонент | Critical | High | Medium | Low | Всего |
|:---|:---|:---|:---|:---|:---|
| Cart | 0 | 1 | 2 | 0 | 3 |
| Catalog | 0 | 1 | 0 | 1 | 2 |
| Order | 1 | 0 | 0 | 0 | 1 |
| Payment | 1 | 0 | 0 | 0 | 1 |
| User | 1 | 0 | 0 | 0 | 1 |
| Другие | 0 | 2 | 1 | 1 | 4 |

---

### 4. Ключевые дефекты

#### BUG-001: Ошибка 500 при оформлении заказа
- **Серьёзность:** Critical
- **Компонент:** Order
- **Описание:** При оформлении заказа с промокодом возникает ошибка 500
- **Рекомендация:** Исправить до релиза

#### BUG-009: Ошибка в платёжном шлюзе при валюте USD
- **Серьёзность:** Critical
- **Компонент:** Payment
- **Описание:** Платеж не проходит при выборе валюты USD
- **Рекомендация:** Исправить до релиза

---

### 5. Метрики качества

| Метрика | Значение | Цель | Статус |
|:---|:---|:---|:---|
| Pass Rate | 88% | ≥95% | ❌ |
| Defect Density | 2.4/KLOC | <3 | ✅ |
| Defect Leakage | 8% | <10% | ✅ |
| Avg Fix Time | 18.5ч | <24ч | ✅ |

---

### 6. Риски и рекомендации

**Риски:**
1. Критические базы в Order и Payment блокируют релиз
2. Низкий Pass Rate указывает на проблемы со стабильностью

**Рекомендации:**
1. 🔴 Исправить критические дефекты BUG-001 и BUG-009 до релиза
2. 🟡 Провести дополнительное тестирование Order и Payment компонентов
3. 🟢 Увеличить покрытие автоматизированными тестами для регрессии

---

### 7. Заключение

| Решение | Обоснование |
|:---|:---|
| ❌ Релиз нельзя выпускать | Критические дефекты в ключевых компонентах |

**Следующие шаги:**
1. Исправление критических дефектов (2 дня)
2. Регрессионное тестирование (1 день)
3. Повторная оценка готовности к релизу

---

**Подписи:**
- **QA Инженер:** _________________
- **QA Lead:** _________________
- **Project Manager:** _________________
Задание 3.2. Расчёт экономического эффекта (10 минут)
python
class EconomicImpactCalculator:
    """Калькулятор экономического эффекта от тестирования."""
    
    def __init__(self):
        self.hourly_rate_developer = 100
        self.hourly_rate_support = 80
        self.lost_revenue_per_hour = 5000
    
    def calculate_cost_of_defects(self, defects: List[Defect]) -> Dict:
        """Расчёт стоимости дефектов."""
        total_cost = 0
        cost_by_severity = {"Critical": 0, "High": 0, "Medium": 0, "Low": 0}
        
        for defect in defects:
            # Стоимость исправления
            fix_cost = defect.resolution_time_hours() * self.hourly_rate_developer
            
            # Дополнительная стоимость в зависимости от серьёзности
            if defect.severity == "Critical":
                severity_cost = self.lost_revenue_per_hour * 4
            elif defect.severity == "High":
                severity_cost = self.lost_revenue_per_hour * 2
            elif defect.severity == "Medium":
                severity_cost = self.hourly_rate_support * 4
            else:
                severity_cost = self.hourly_rate_support * 1
            
            total_cost += fix_cost + severity_cost
            cost_by_severity[defect.severity] += fix_cost + severity_cost
        
        return {
            "total_cost": total_cost,
            "cost_by_severity": cost_by_severity,
            "average_cost_per_defect": total_cost / len(defects) if defects else 0
        }
    
    def calculate_savings(self, defects: List[Defect], testing_cost: float) -> Dict:
        """Расчёт экономии от тестирования."""
        # Предполагаем, что без тестирования все дефекты дошли бы до продакшена
        cost_without_testing = self.calculate_cost_of_defects(defects)["total_cost"]
        
        # Стоимость с тестированием (исправление на этапе тестирования дешевле)
        # Используем коэффициент 0.1 (тестирование в 10 раз дешевле продакшена)
        cost_with_testing = cost_without_testing * 0.1 + testing_cost
        
        savings = cost_without_testing - cost_with_testing
        roi = (savings / testing_cost) * 100 if testing_cost > 0 else 0
        
        return {
            "cost_without_testing": cost_without_testing,
            "cost_with_testing": cost_with_testing,
            "testing_cost": testing_cost,
            "savings": savings,
            "roi_percent": roi
        }


def calculate_impact():
    defects = load_sample_defects()
    calculator = EconomicImpactCalculator()
    
    # Стоимость дефектов
    cost_result = calculator.calculate_cost_of_defects(defects)
    print("=" * 50)
    print("ЭКОНОМИЧЕСКИЙ ЭФФЕКТ ТЕСТИРОВАНИЯ")
    print("=" * 50)
    
    print(f"\n💰 Стоимость дефектов (при попадании в продакшен):")
    print(f"   Общая стоимость: ${cost_result['total_cost']:,.2f}")
    print(f"   Средняя стоимость дефекта: ${cost_result['average_cost_per_defect']:.2f}")
    
    print(f"\n📈 Экономия от тестирования (при бюджете $50,000):")
    savings = calculator.calculate_savings(defects, testing_cost=50000)
    print(f"   Стоимость без тестирования: ${savings['cost_without_testing']:,.2f}")
    print(f"   Стоимость с тестированием: ${savings['cost_with_testing']:,.2f}")
    print(f"   Экономия: ${savings['savings']:,.2f}")
    print(f"   ROI тестирования: {savings['roi_percent']:.0f}%")


if __name__ == "__main__":
    calculate_impact()
Задание 3.3. Итоговый отчёт (5 минут)
markdown
## Комплексный отчёт по управлению дефектами

**Студент:** _________________
**Вариант №:** ___ (18-25)

### Созданные баг-репорты

| ID | Summary | Severity | Priority | Status | Resolution Time |
|:---|:---|:---|:---|:---|:---|
| BUG-001 | | | | | |
| ... | | | | | |

### Анализ метрик

| Метрика | Значение | Оценка |
|:---|:---|:---|
| Defect Density | _____/KLOC | |
| Defect Leakage | _____% | |
| Avg Resolution Time | _____ ч | |
| Pass Rate | _____% | |

### Экономический эффект

| Показатель | Значение |
|:---|:---|
| Стоимость без тестирования | $_____ |
| Стоимость с тестированием | $_____ |
| Экономия | $_____ |
| ROI | _____% |

### Рекомендации по улучшению

1. _______________________________________________________________
2. _______________________________________________________________
3. _______________________________________________________________

### Вывод о готовности к релизу

- [ ] ✅ Готов к релизу
- [ ] ❌ Требуется доработка

_______________________________________________________________
Карточка студента
text
ПР 3.14. РАБОТА С БАГ-ТРЕКИНГОВОЙ СИСТЕМОЙ

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ Создание баг-репортов (базовый)
□ Классификация дефектов (базовый)
□ Жизненный цикл дефекта (базовый)
□ Расчёт метрик (средний)
□ Анализ трендов (средний)
□ Визуализация (средний)
□ Полный отчёт о тестировании (сложный)
□ Экономический эффект (сложный)

=== РЕЗУЛЬТАТЫ ===

Создано баг-репортов: _____
Проанализировано дефектов: _____
Defect Density: _____
Defect Leakage: _____%
ROI тестирования: _____%

=== ВРЕМЯ ВЫПОЛНЕНИЯ ===

Подготовка: _____ мин
Выполнение: _____ мин
Оформление: _____ мин

=== ОТЧЁТ ===

Файлы:
- defect_data.py
- defect_metrics.py
- test_report.md

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Баг-репорты не созданы
3 (удовлетворительно)	Базовый	3+ баг-репорта, классификация (5+ дефектов)
4 (хорошо)	Средний	+ Метрики + тренды + визуализация (10+ дефектов)
5 (отлично)	Сложный	+ Полный отчёт + экономический эффект (12+ дефектов)
Контрольные вопросы (для защиты)
Какие поля обязательно должны быть в баг-репорте?

Чем отличается серьёзность от приоритета?

Какие статусы может проходить дефект?

Как рассчитывается утечка дефектов?

Какое значение Defect Density считается хорошим?

Как визуализировать распределение дефектов по компонентам?

Как рассчитать ROI тестирования?

Какие компоненты нужно проверять в первую очередь, если в них много дефектов?

Как определить, готов ли продукт к релизу?

Какие метрики важны для оценки эффективности команды?

Следующее занятие: Контрольная работа 3.4 по темам 3.9–3.15.

text
