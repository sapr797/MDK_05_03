# Лекция 3.12: Тестирование иерархий классов и полиморфизма

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.8:** Тестирование классов и объектов  
**Тип занятия:** Лекция (2 часа)

---

## Цель лекции

Изучить методы тестирования иерархий классов и полиморфного поведения, освоить параметризацию тестов по классам для проверки полиморфных контрактов.

## Планируемые результаты (по ФГОС СПО)

После этой лекции вы сможете:
1. Тестировать иерархии классов с общим интерфейсом (ПК 3.2, ПК 3.3).
2. Проверять соблюдение полиморфного контракта (Liskov Substitution Principle) (ОК 02, ОК 05).
3. Использовать параметризацию по классам для сокращения дублирования тестов (ПК 3.2).
4. Применять паттерн «Тест-дублёр» для иерархий (ПК 3.3).

---

## 1. Полиморфизм и принцип подстановки Лисков (LSP)

### 1.1 Что такое полиморфизм?

**Полиморфизм** — способность объектов разных классов с общим интерфейсом реагировать на одни и те же вызовы методов специфическим для класса образом.
┌─────────────────────────────────────────────────────────────────────────────┐
│ ПОЛИМОРФНОЕ ПОВЕДЕНИЕ │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ interface Shape │
│ ┌─────────────────┐ │
│ │ + area(): float │ │
│ │ + perimeter(): float │
│ └────────┬────────┘ │
│ │ │
│ ┌──────┼──────┬──────────┐ │
│ ↓ ↓ ↓ ↓ │
│ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ │
│ │Circle│ │Square│ │Rect │ │Triangle│ │
│ └──────┘ └──────┘ └──────┘ └──────┘ │
│ ↓ ↓ ↓ ↓ │
│ πr² a² w·h (b·h)/2 │
│ │
│ def test_shape_polymorphism(shapes): │
│ for shape in shapes: │
│ assert shape.area() > 0 # Все фигуры имеют площадь │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### 1.2 Принцип подстановки Лисков (LSP)

Принцип подстановки Лисков гласит: **объекты производного класса должны быть заменяемы объектами базового класса без изменения корректности программы**.

```python
class Bird:
    def fly(self) -> str:
        return "Flying"
    
    def get_speed(self) -> float:
        return 50.0


class Sparrow(Bird):
    def fly(self) -> str:
        return "Sparrow flying"
    
    def get_speed(self) -> float:
        return 40.0


class Penguin(Bird):
    def fly(self) -> str:
        raise NotImplementedError("Penguins can't fly!")  # ❌ Нарушает LSP!
    
    def get_speed(self) -> float:
        return 0.0
2. Тестирование полиморфного контракта
2.1 Пример иерархии фигур
python
from abc import ABC, abstractmethod
from math import pi
import pytest


class Shape(ABC):
    """Абстрактный базовый класс для всех фигур."""
    
    @abstractmethod
    def area(self) -> float:
        """Возвращает площадь фигуры."""
        pass
    
    @abstractmethod
    def perimeter(self) -> float:
        """Возвращает периметр фигуры."""
        pass
    
    @abstractmethod
    def get_name(self) -> str:
        """Возвращает имя фигуры."""
        pass
    
    def scale(self, factor: float) -> None:
        """Масштабирует фигуру (изменяет состояние)."""
        pass


class Circle(Shape):
    def __init__(self, radius: float):
        if radius <= 0:
            raise ValueError("Radius must be positive")
        self.radius = radius
    
    def area(self) -> float:
        return pi * self.radius ** 2
    
    def perimeter(self) -> float:
        return 2 * pi * self.radius
    
    def get_name(self) -> str:
        return "Circle"
    
    def scale(self, factor: float) -> None:
        if factor <= 0:
            raise ValueError("Scale factor must be positive")
        self.radius *= factor


class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        if width <= 0 or height <= 0:
            raise ValueError("Width and height must be positive")
        self.width = width
        self.height = height
    
    def area(self) -> float:
        return self.width * self.height
    
    def perimeter(self) -> float:
        return 2 * (self.width + self.height)
    
    def get_name(self) -> str:
        return "Rectangle"
    
    def scale(self, factor: float) -> None:
        if factor <= 0:
            raise ValueError("Scale factor must be positive")
        self.width *= factor
        self.height *= factor


class Square(Rectangle):
    def __init__(self, side: float):
        super().__init__(side, side)
    
    def get_name(self) -> str:
        return "Square"
    
    def scale(self, factor: float) -> None:
        if factor <= 0:
            raise ValueError("Scale factor must be positive")
        self.width *= factor
        self.height *= factor
2.2 Базовые тесты для иерархии
python
class BaseShapeTest:
    """Базовый класс с общими тестами для всех фигур."""
    
    def create_shape(self) -> Shape:
        """Метод-фабрика, переопределяется в наследниках."""
        raise NotImplementedError
    
    def test_area_positive(self):
        """Площадь всегда положительна."""
        shape = self.create_shape()
        assert shape.area() > 0
    
    def test_perimeter_positive(self):
        """Периметр всегда положителен."""
        shape = self.create_shape()
        assert shape.perimeter() > 0
    
    def test_get_name_returns_string(self):
        """Название фигуры — непустая строка."""
        shape = self.create_shape()
        assert isinstance(shape.get_name(), str)
        assert len(shape.get_name()) > 0
    
    def test_scaling_changes_area(self):
        """Масштабирование меняет площадь."""
        shape = self.create_shape()
        original_area = shape.area()
        
        shape.scale(2.0)
        new_area = shape.area()
        
        # Для всех фигур при масштабировании в 2 раза площадь растёт в 4 раза
        assert new_area == pytest.approx(original_area * 4)
    
    def test_negative_scale_raises_error(self):
        """Отрицательный коэффициент масштабирования вызывает ошибку."""
        shape = self.create_shape()
        with pytest.raises(ValueError, match="Scale factor must be positive"):
            shape.scale(-1.0)
2.3 Наследники с конкретными реализациями
python
class TestCircle(BaseShapeTest):
    """Тесты для Circle."""
    
    def create_shape(self) -> Shape:
        return Circle(radius=5.0)
    
    def test_circle_specific_behavior(self):
        """Специфическое поведение круга."""
        circle = Circle(radius=10)
        assert circle.radius == 10
        assert circle.area() == pytest.approx(pi * 100)
    
    def test_invalid_radius(self):
        with pytest.raises(ValueError, match="Radius must be positive"):
            Circle(radius=-5)


class TestRectangle(BaseShapeTest):
    """Тесты для Rectangle."""
    
    def create_shape(self) -> Shape:
        return Rectangle(width=4, height=6)
    
    def test_rectangle_specific_behavior(self):
        rect = Rectangle(4, 6)
        assert rect.width == 4
        assert rect.height == 6
        assert rect.area() == 24
    
    def test_invalid_dimensions(self):
        with pytest.raises(ValueError, match="Width and height must be positive"):
            Rectangle(-4, 6)


class TestSquare(BaseShapeTest):
    """Тесты для Square."""
    
    def create_shape(self) -> Shape:
        return Square(side=5)
    
    def test_square_properties(self):
        square = Square(5)
        assert square.width == 5
        assert square.height == 5
        assert square.area() == 25
        assert square.perimeter() == 20
3. Параметризация тестов по классам
3.1 Параметризация с использованием pytest
python
# Параметризация по классам
@pytest.mark.parametrize("shape_class,params,expected_area", [
    (Circle, {"radius": 5}, pi * 25),
    (Rectangle, {"width": 4, "height": 6}, 24),
    (Square, {"side": 5}, 25),
])
def test_shape_area_parametrized(shape_class, params, expected_area):
    """Один тест для всех классов фигур."""
    shape = shape_class(**params)
    assert shape.area() == pytest.approx(expected_area)


@pytest.mark.parametrize("shape_class,params,expected_perimeter", [
    (Circle, {"radius": 5}, 2 * pi * 5),
    (Rectangle, {"width": 4, "height": 6}, 2 * (4 + 6)),
    (Square, {"side": 5}, 4 * 5),
])
def test_shape_perimeter_parametrized(shape_class, params, expected_perimeter):
    shape = shape_class(**params)
    assert shape.perimeter() == pytest.approx(expected_perimeter)
3.2 Фикстура с параметризацией по классам
python
@pytest.fixture(params=[
    (Circle, {"radius": 5}),
    (Rectangle, {"width": 4, "height": 6}),
    (Square, {"side": 5}),
])
def shape_instance(request):
    """Фикстура, создающая экземпляры разных фигур."""
    shape_class, params = request.param
    shape = shape_class(**params)
    yield shape


def test_shape_polymorphism(shape_instance):
    """Полиморфный тест для всех фигур."""
    shape = shape_instance
    
    # Все фигуры должны иметь положительную площадь
    assert shape.area() > 0
    
    # Все фигуры должны иметь положительный периметр
    assert shape.perimeter() > 0
    
    # Название должно быть непустой строкой
    assert isinstance(shape.get_name(), str)
    assert shape.get_name()
4. Тестирование контракта (Protocol / Duck Typing)
4.1 Протокол в Python
python
from typing import Protocol
import pytest


class Drawable(Protocol):
    """Протокол для рисуемых объектов."""
    
    def draw(self) -> str:
        ...
    
    def get_position(self) -> tuple[int, int]:
        ...


class Point:
    def __init__(self, x: int, y: int):
        self.x = x
        self.y = y
    
    def draw(self) -> str:
        return f"Point at ({self.x}, {self.y})"
    
    def get_position(self) -> tuple[int, int]:
        return (self.x, self.y)


class Line:
    def __init__(self, x1: int, y1: int, x2: int, y2: int):
        self.x1, self.y1 = x1, y1
        self.x2, self.y2 = x2, y2
    
    def draw(self) -> str:
        return f"Line from ({self.x1},{self.y1}) to ({self.x2},{self.y2})"
    
    def get_position(self) -> tuple[int, int]:
        return (self.x1, self.y1)


class Circle:
    def __init__(self, x: int, y: int, radius: int):
        self.x = x
        self.y = y
        self.radius = radius
    
    def draw(self) -> str:
        return f"Circle at ({self.x},{self.y}) with radius {self.radius}"
    
    def get_position(self) -> tuple[int, int]:
        return (self.x, self.y)
4.2 Тестирование протокола
python
@pytest.mark.parametrize("drawable", [
    Point(10, 20),
    Line(0, 0, 100, 100),
    Circle(50, 50, 25),
])
def test_drawable_protocol(drawable):
    """Тест для всех объектов, реализующих протокол Drawable."""
    
    # Проверка метода draw
    result = drawable.draw()
    assert isinstance(result, str)
    assert len(result) > 0
    
    # Проверка метода get_position
    pos = drawable.get_position()
    assert isinstance(pos, tuple)
    assert len(pos) == 2
    assert isinstance(pos[0], int)
    assert isinstance(pos[1], int)
5. Тестирование абстрактного базового класса
5.1 Проверка, что ABC нельзя инстанцировать
python
def test_abstract_class_cannot_be_instantiated():
    """Абстрактный класс нельзя создать напрямую."""
    with pytest.raises(TypeError):
        Shape()  # TypeError: Can't instantiate abstract class Shape
5.2 Проверка, что все абстрактные методы реализованы
python
class IncompleteShape(Shape):
    """Класс, не реализующий все абстрактные методы."""
    def area(self) -> float:
        return 10
    
    # Нет реализации perimeter() и get_name()


def test_incomplete_class_cannot_be_instantiated():
    """Класс с нереализованными методами нельзя создать."""
    with pytest.raises(TypeError):
        IncompleteShape()
6. Паттерн «Тест-дублёр» для иерархий
6.1 Мокирование при тестировании полиморфизма
python
from unittest.mock import Mock


def test_polymorphic_behavior_with_mock():
    """Использование мок-объектов для тестирования полиморфизма."""
    
    # Создаём моки для разных фигур
    circle_mock = Mock(spec=Shape)
    circle_mock.area.return_value = 78.5
    circle_mock.perimeter.return_value = 31.4
    circle_mock.get_name.return_value = "Circle"
    
    rectangle_mock = Mock(spec=Shape)
    rectangle_mock.area.return_value = 24.0
    rectangle_mock.perimeter.return_value = 20.0
    rectangle_mock.get_name.return_value = "Rectangle"
    
    # Функция, работающая с любыми Shape
    def total_area(shapes: list[Shape]) -> float:
        return sum(shape.area() for shape in shapes)
    
    shapes = [circle_mock, rectangle_mock]
    assert total_area(shapes) == 78.5 + 24.0
    
    # Проверяем вызовы методов
    circle_mock.area.assert_called_once()
    rectangle_mock.area.assert_called_once()
7. Шпаргалка
python
# === БАЗОВЫЙ ТЕСТ ДЛЯ ИЕРАРХИИ ===
class BaseTest:
    def create_instance(self):
        raise NotImplementedError
    
    def test_common_behavior(self):
        obj = self.create_instance()
        assert obj.common_method() == expected

class TestChild(BaseTest):
    def create_instance(self):
        return Child()

# === ПАРАМЕТРИЗАЦИЯ ПО КЛАССАМ ===
@pytest.mark.parametrize("cls,kwargs", [
    (Circle, {"radius": 5}),
    (Rect, {"width": 4, "height": 6}),
])
def test_shape(cls, kwargs):
    shape = cls(**kwargs)
    assert shape.area() > 0

# === ФИКСТУРА С ПАРАМЕТРИЗАЦИЕЙ ===
@pytest.fixture(params=[Circle(5), Rect(4, 6)])
def shape(request):
    return request.param

def test_polymorphism(shape):
    assert shape.area() > 0

# === ТЕСТИРОВАНИЕ ПРОТОКОЛА ===
@pytest.mark.parametrize("obj", [Point(1,2), Line(0,0,1,1)])
def test_drawable(obj):
    assert hasattr(obj, "draw")
    assert isinstance(obj.draw(), str)
Контрольные вопросы
Что такое полиморфизм и как его тестировать?

В чём суть принципа подстановки Лисков (LSP)?

Как организовать общие тесты для всей иерархии классов?

Как параметризовать тесты по классам в pytest?

Чем тестирование протокола отличается от тестирования наследования?

Как проверить, что абстрактный класс нельзя инстанцировать?

Как протестировать, что все наследники реализуют нужные методы?

Когда стоит использовать моки вместо реальных объектов при тестировании полиморфизма?

Как тестировать полиморфное поведение с побочными эффектами?

Как проверить, что наследник не нарушает контракт базового класса?

Следующее занятие: ПР 3.12 — Практическое тестирование иерархий классов и полиморфизма.
