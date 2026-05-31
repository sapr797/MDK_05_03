# Практическое занятие 3.1: Знакомство с объектом тестирования. Применение подходов «чёрного, белого и серого» ящика

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.1:** Основы тестирования и жизненный цикл ПО  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться анализировать объект тестирования, выделять компоненты системы и применять различные подходы (чёрный, белый, серый ящик) для тестирования одной и той же функции.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Провести анализ учебного приложения и выделить компоненты для тестирования (ПК 3.1).
2. Применить подходы «чёрного», «белого» и «серого» ящика к одной функции (ПК 3.2).
3. Оформить результаты тестирования в виде таблиц и тест-кейсов (ОК 02, ОК 05).

---

## Необходимое программное обеспечение

- Браузер (Chrome/Firefox) с инструментами разработчика (F12)
- Текстовый редактор (VS Code, PhpStorm)
- Установленный Laravel (или любой другой учебный проект)
- Postman или Insomnia (для тестирования API)
- PHPUnit (встроен в Laravel)

---

## Теоретическая справка (5 минут)

### Что такое «объект тестирования»?

**Объект тестирования** — это конкретный компонент, модуль, функция или система в целом, которую вы проверяете.

**Примеры объектов тестирования:**
- Функция `calculateDiscount($total, $status)`
- Роут `POST /api/users`
- Форма входа на сайте
- Вся система интернет-магазина

### Компоненты системы для тестирования

При анализе приложения нужно выделить **компоненты** — логически независимые части:

| Тип компонента | Пример в Laravel | Что проверяем |
|:---|:---|:---|
| **Метод/функция** | `calculateTotal()` | Алгоритм, математику |
| **Контроллер** | `PostController@store` | Валидацию, вызов сервисов |
| **Роут + контроллер + БД** | `POST /posts` | Интеграцию |
| **API эндпоинт** | `/api/v1/users` | JSON-ответы, статусы |
| **UI компонент** | Форма регистрации | Поля, кнопки, сообщения |

---

## Ход работы

### Этап 1. Знакомство с объектом тестирования (20 минут)

В качестве объекта тестирования возьмём **функцию расчёта итоговой суммы заказа со скидкой** (аналог из лекции, но с дополнительной логикой).

#### Листинг 1. Исходный код для тестирования

```php
<?php
// app/Services/OrderCalculator.php

namespace App\Services;

class OrderCalculator
{
    /**
     * Рассчитать итоговую сумму заказа с учётом скидок и доставки
     * 
     * @param float $subtotal Сумма товаров без скидки
     * @param string $userTier Уровень пользователя: 'bronze', 'silver', 'gold'
     * @param float $deliveryFee Стоимость доставки (по умолчанию 300)
     * @param bool $usePromocode Использовать ли промокод на 10% (только для gold)
     * @return float Итоговая сумма
     */
    public function calculate($subtotal, $userTier, $deliveryFee = 300, $usePromocode = false)
    {
        // 1. Рассчитываем скидку в зависимости от уровня
        $discountPercent = 0;
        
        switch ($userTier) {
            case 'gold':
                $discountPercent = 15; // 15% скидка
                break;
            case 'silver':
                $discountPercent = 10; // 10% скидка
                break;
            case 'bronze':
                $discountPercent = 5;  // 5% скидка
                break;
            default:
                $discountPercent = 0;
        }
        
        // 2. Дополнительная скидка по промокоду (только для gold)
        if ($usePromocode && $userTier === 'gold') {
            $discountPercent += 10;
        }
        
        // 3. Применяем скидку
        $discount = $subtotal * ($discountPercent / 100);
        $afterDiscount = $subtotal - $discount;
        
        // 4. Бесплатная доставка для сумм больше 5000
        if ($afterDiscount > 5000) {
            $deliveryFee = 0;
        }
        
        // 5. Итог
        return round($afterDiscount + $deliveryFee, 2);
    }
}
Ваша задача: Прочитайте код и выпишите в тетрадь:

Какие входные параметры принимает функция?

Какие есть ветвления (if, switch)?

Какие граничные значения нужно проверить?

Этап 2. Определение компонентов для тестирования (15 минут)
Заполните таблицу компонентов для данного объекта тестирования:

Уровень	Компонент	Что делает	Как тестировать
Модульный	Метод calculate()	Считает сумму со скидкой	Написать юнит-тест (белый ящик)
Интеграционный	Сервис + Контроллер	Вызов метода из контроллера	Проверить, что контроллер вызывает метод с правильными параметрами
API	Роут /api/calculate	Возвращает JSON с итогом	Отправить POST-запрос (серый ящик)
UI	Форма заказа	Пользователь вводит данные, видит результат	Ручное или автоматизированное тестирование (чёрный ящик)
Этап 3. Тестирование одной функции с трёх точек зрения (60 минут)
3.1 Тестирование методом «белого ящика»
Задача: Написать модульные тесты, покрывающие все ветвления функции.

Создайте файл tests/Unit/OrderCalculatorTest.php:

php
<?php
namespace Tests\Unit;

use Tests\TestCase;
use App\Services\OrderCalculator;

class OrderCalculatorTest extends TestCase
{
    private $calculator;
    
    protected function setUp(): void
    {
        parent::setUp();
        $this->calculator = new OrderCalculator();
    }
    
    /**
     * Тест 1: Пользователь gold с промокодом
     * Скидка: 15% + 10% = 25%
     * Сумма 10000 - 25% = 7500, доставка бесплатно (>5000)
     * Итог: 7500
     */
    public function testGoldUserWithPromocode()
    {
        $result = $this->calculator->calculate(10000, 'gold', 300, true);
        $this->assertEquals(7500, $result);
    }
    
    /**
     * Тест 2: Пользователь gold без промокода
     * Скидка: 15%
     * Сумма 10000 - 15% = 8500, доставка бесплатно (>5000)
     * Итог: 8500
     */
    public function testGoldUserWithoutPromocode()
    {
        $result = $this->calculator->calculate(10000, 'gold', 300, false);
        $this->assertEquals(8500, $result);
    }
    
    /**
     * Тест 3: Пользователь silver
     * Скидка: 10%
     * Сумма 3000 - 10% = 2700, доставка 300 (так как 2700 < 5000)
     * Итог: 3000
     */
    public function testSilverUser()
    {
        $result = $this->calculator->calculate(3000, 'silver', 300, false);
        $this->assertEquals(3000, $result);
    }
    
    /**
     * Тест 4: Пользователь bronze
     * Скидка: 5%
     * Сумма 1000 - 5% = 950, доставка 300
     * Итог: 1250
     */
    public function testBronzeUser()
    {
        $result = $this->calculator->calculate(1000, 'bronze', 300, false);
        $this->assertEquals(1250, $result);
    }
    
    /**
     * Тест 5: Гость (неизвестный уровень)
     * Скидка: 0
     * Сумма 1000, доставка 300
     * Итог: 1300
     */
    public function testGuestUser()
    {
        $result = $this->calculator->calculate(1000, 'unknown', 300, false);
        $this->assertEquals(1300, $result);
    }
    
    /**
     * Тест 6: Граничное значение для бесплатной доставки
     * Сумма после скидки = 5000 (доставка должна быть бесплатной?)
     * По логике: if ($afterDiscount > 5000) - строго больше, значит 5000 = платная доставка
     */
    public function testDeliveryOnBoundary()
    {
        // 5000 / 0.85 ≈ 5882.35, чтобы после 15% скидки получить 5000
        $result = $this->calculator->calculate(5882.35, 'gold', 300, false);
        // 5882.35 - 15% = 5000.00, доставка платная (300)
        $this->assertEquals(5300, $result);
    }
}
Задание для студентов: Допишите тесты для следующих сценариев:

Промокод не работает для silver (должно быть без изменений)

Сумма после скидки ровно 5000.01 (доставка должна стать бесплатной)

Отрицательная сумма (обработка ошибки)

3.2 Тестирование методом «серого ящика»
Задача: Протестировать API эндпоинт, который использует наш калькулятор. Вы знаете структуру запроса/ответа, но не знаете деталей реализации внутри контроллера.

Шаг 1: Создайте контроллер (если его ещё нет):

bash
php artisan make:controller OrderController
Шаг 2: Добавьте метод в OrderController.php:

php
<?php
namespace App\Http\Controllers;

use App\Services\OrderCalculator;
use Illuminate\Http\Request;

class OrderController extends Controller
{
    private $calculator;
    
    public function __construct(OrderCalculator $calculator)
    {
        $this->calculator = $calculator;
    }
    
    public function calculateTotal(Request $request)
    {
        $validated = $request->validate([
            'subtotal' => 'required|numeric|min:0',
            'user_tier' => 'required|string|in:bronze,silver,gold,guest',
            'delivery_fee' => 'numeric|min:0',
            'use_promocode' => 'boolean'
        ]);
        
        $total = $this->calculator->calculate(
            $validated['subtotal'],
            $validated['user_tier'],
            $validated['delivery_fee'] ?? 300,
            $validated['use_promocode'] ?? false
        );
        
        return response()->json([
            'success' => true,
            'total' => $total,
            'currency' => 'RUB'
        ]);
    }
}
Шаг 3: Добавьте роут в routes/api.php:

php
Route::post('/calculate', [OrderController::class, 'calculateTotal']);
Шаг 4: Напишите интеграционный тест (серый ящик) в tests/Feature/OrderApiTest.php:

php
<?php
namespace Tests\Feature;

use Tests\TestCase;
use App\Services\OrderCalculator;

class OrderApiTest extends TestCase
{
    /**
     * Тест API: пользователь gold с промокодом
     * Знаем API и БД, но не знаем, как именно контроллер вызывает метод
     */
    public function testApiCalculateGoldWithPromocode()
    {
        $response = $this->postJson('/api/calculate', [
            'subtotal' => 10000,
            'user_tier' => 'gold',
            'delivery_fee' => 300,
            'use_promocode' => true
        ]);
        
        $response->assertStatus(200)
                 ->assertJson([
                     'success' => true,
                     'total' => 7500,
                     'currency' => 'RUB'
                 ]);
    }
    
    /**
     * Тест API: валидация — неверный user_tier
     */
    public function testApiValidationInvalidTier()
    {
        $response = $this->postJson('/api/calculate', [
            'subtotal' => 1000,
            'user_tier' => 'platinum', // недопустимое значение
        ]);
        
        $response->assertStatus(422); // Unprocessable Entity
        $response->assertJsonValidationErrors(['user_tier']);
    }
    
    /**
     * Тест API: отрицательная сумма
     */
    public function testApiValidationNegativeSubtotal()
    {
        $response = $this->postJson('/api/calculate', [
            'subtotal' => -100,
            'user_tier' => 'gold',
        ]);
        
        $response->assertStatus(422);
        $response->assertJsonValidationErrors(['subtotal']);
    }
}
Задание для студентов: Допишите тесты API для:

Отсутствующего поля user_tier

Пустого массива данных

Неверного формата use_promocode (передана строка вместо булевого)

3.3 Тестирование методом «чёрного ящика»
Задача: Протестировать функцию через пользовательский интерфейс (если есть форма) или через Postman, не зная внутреннего устройства.

Шаг 1: Запустите сервер Laravel:

bash
php artisan serve
Шаг 2: Откройте Postman и создайте запрос:

Параметр	Значение
Method	POST
URL	http://127.0.0.1:8000/api/calculate
Headers	Content-Type: application/json
Body (raw JSON)	{"subtotal": 10000, "user_tier": "gold", "delivery_fee": 300, "use_promocode": true}
Ожидаемый ответ: {"success":true,"total":7500,"currency":"RUB"}

Шаг 3: Составьте чек-лист для ручного тестирования (чёрный ящик):

№	Тест-кейс	Входные данные	Ожидаемый результат	Факт	Статус
1	Gold + промокод, сумма > 5000	{"subtotal":10000,"user_tier":"gold","use_promocode":true}	total = 7500		☐
2	Gold без промокода, сумма > 5000	{"subtotal":10000,"user_tier":"gold","use_promocode":false}	total = 8500		☐
3	Silver, сумма < 5000	{"subtotal":3000,"user_tier":"silver"}	total = 3000		☐
4	Bronze, сумма 1000	{"subtotal":1000,"user_tier":"bronze"}	total = 1250		☐
5	Гость (неизвестный уровень)	{"subtotal":1000,"user_tier":"guest"}	total = 1300		☐
6	Неверный уровень (platinum)	{"subtotal":1000,"user_tier":"platinum"}	Ошибка 422, сообщение о неверном поле		☐
7	Поле user_tier отсутствует	{"subtotal":1000}	Ошибка 422		☐
8	Отрицательная сумма	{"subtotal":-500,"user_tier":"gold"}	Ошибка 422		☐
9	Строка вместо числа	{"subtotal":"abc","user_tier":"gold"}	Ошибка 422		☐
Задание для студентов: Выполните ручное тестирование по чек-листу. Заполните столбцы «Факт» и «Статус». Найдите хотя бы один дефект (если всё работает — предложите граничный случай, который может быть не обработан).

Этап 4. Сравнение подходов (10 минут)
Заполните таблицу на основе вашего опыта тестирования функции calculate():

Критерий	Чёрный ящик	Белый ящик	Серый ящик
Нужно ли знать код?			
Кто выполняет?			
Какие дефекты нашли?			
Сколько времени заняло?			
Что легче автоматизировать?			
Этап 5. Оформление отчёта (30 минут)
Оформите результаты работы в виде отчёта по шаблону:

Шаблон отчёта по практической работе 3.1
markdown
# Отчёт по практической работе 3.1

**Студент:** _________________  
**Группа:** _________________  
**Дата:** _________________

## 1. Объект тестирования

**Название:** Функция `calculate()` класса `OrderCalculator`

**Назначение:** Расчёт итоговой суммы заказа со скидками и доставкой

**Входные параметры:**
- `subtotal` (float)
- `userTier` (string: bronze, silver, gold, guest)
- `deliveryFee` (float, default 300)
- `usePromocode` (bool, default false)

**Выходные данные:** float (округлено до 2 знаков)

## 2. Компоненты системы для тестирования

| Уровень | Компонент | Метод тестирования |
|:---|:---|:---|
| Модульный | Метод calculate() | PHPUnit (белый ящик) |
| Интеграционный | OrderController + OrderCalculator | Feature test (серый ящик) |
| API | POST /api/calculate | Postman + автоматизация |
| UI | (если есть форма) | Ручной чек-лист |

## 3. Результаты тестирования

### 3.1 Белый ящик (модульные тесты)

| Тест | Результат | Найденные дефекты |
|:---|:---|:---|
| testGoldUserWithPromocode | Прошел / Не прошел | |
| testGoldUserWithoutPromocode | | |
| testSilverUser | | |
| testBronzeUser | | |
| testGuestUser | | |
| testDeliveryOnBoundary | | |

**Найденные дефекты при белом ящике:**
1. 
2. 

### 3.2 Серый ящик (API тесты)

| Тест | Результат | Найденные дефекты |
|:---|:---|:---|
| testApiCalculateGoldWithPromocode | | |
| testApiValidationInvalidTier | | |
| testApiValidationNegativeSubtotal | | |

**Найденные дефекты при сером ящике:**
1. 

### 3.3 Чёрный ящик (ручное тестирование)

| № кейса | Статус | Фактический результат |
|:---|:---|:---|
| 1 | | |
| 2 | | |
| ... | | |

**Найденные дефекты при чёрном ящике:**
1. 

## 4. Сравнение подходов

| Критерий | Чёрный ящик | Белый ящик | Серый ящик |
|:---|:---|:---|:---|
| Нужно ли знать код? | Нет | Да | Частично |
| Кто выполняет? | Тестировщик, заказчик | Разработчик | QA-автоматизатор |
| Какие дефекты нашли? | | | |
| Время выполнения | | | |

## 5. Вывод

*(Что нового узнали? Какой подход лучше подходит для тестирования математических функций? Какой для API?)*
Дополнительные задания (для сильных студентов)
Задание А. Тестирование с мутациями
Измените код функции calculate() (внесите типичную ошибку) и проверьте, какой из подходов (чёрный/белый/серый) её обнаружит.

Примеры мутаций (ошибок):

Изменить > на >= в условии бесплатной доставки

Заменить $discountPercent += 10 на $discountPercent = 10

Убрать round() в конце

Задание Б. Тестирование через браузер
Создайте простую HTML-форму, которая отправляет AJAX-запрос к API /api/calculate и выводит результат. Протестируйте её вручную (чёрный ящик).

Задание В. Автоматизация чёрного ящика
Напишите простой скрипт на Python или JavaScript, который отправляет 10 различных запросов к API и сравнивает ответ с ожидаемым. (Это превращает чёрный ящик в автоматизированный, но без знания кода.)

Критерии оценки
Критерий	Макс. балл	Критерии достижения
Анализ объекта тестирования	5	Выделены все входные параметры, ветвления, граничные значения
Тесты белого ящика (Unit)	10	Написано минимум 6 тестов, покрывающих основные ветки
Тесты серого ящика (API)	10	Написано минимум 3 теста, проверяющих валидацию
Тесты чёрного ящика (ручные)	5	Заполнен чек-лист, выполнено минимум 9 проверок
Сравнительный анализ	5	Заполнена таблица сравнения подходов
Оформление отчёта	5	Отчёт структурирован, есть выводы
Итого	40	Проходной балл: 24
Контрольные вопросы (для защиты работы)
Что такое «объект тестирования»? Приведите пример из вашей работы.

Какие компоненты системы вы выделили для тестирования функции calculate()?

В чём разница между чёрным, белым и серым ящиком на примере этой функции?

Какой подход эффективнее для проверки граничных значений? Почему?

Какой подход вы бы выбрали для тестирования API, которое вы не разрабатывали, но у вас есть спецификация?

Почему в реальных проектах используют комбинацию всех трёх подходов?
