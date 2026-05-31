# Практическое занятие 3.14: Тестирование паттерна Factory

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.9:** Тестирование структур данных и паттернов  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться тестировать фабричный паттерн (Factory Method / Simple Factory): проверять создание объектов по типу, обработку неизвестных типов, интеграцию с полиморфными тестами.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Тестировать фабрику, создающую объекты разных типов (ПК 3.2, ПК 3.3).
2. Проверять обработку неизвестных типов и исключения (ПК 3.2).
3. Интегрировать фабрику с полиморфными тестами (ПК 3.2, ПК 3.3).
4. Использовать параметризацию для тестирования различных типов объектов (ОК 02, ОК 05).

---

## Теоретическая справка

### Что такое паттерн Factory?

**Фабрика (Factory)** — порождающий паттерн проектирования, который предоставляет интерфейс для создания объектов, позволяя подклассам решать, какой класс инстанцировать.
┌─────────────────────────────────────────────────────────────────────────────┐
│ ПАТТЕРН FACTORY │
├─────────────────────────────────────────────────────────────────────────────┤
│ │
│ Клиент ──► NotificationFactory ──► create("email") ──► EmailNotification │
│ │ │
│ ├── create("sms") ──► SMSNotification │
│ ├── create("push") ──► PushNotification │
│ └── create("slack") ──► SlackNotification │
│ │
│ Преимущества: │
│ • Единое место создания объектов │
│ • Сокрытие логики инициализации │
│ • Легкость добавления новых типов │
│ │
└─────────────────────────────────────────────────────────────────────────────┘

text

### Объект тестирования (дополнение к inheritance_example.py)

```python
# Дополнение к модулю inheritance_example.py

class NotificationFactory:
    """
    Фабрика для создания уведомлений разных типов.
    """
    
    @staticmethod
    def create(notification_type: str, **kwargs) -> Notification:
        """
        Создаёт уведомление указанного типа.
        
        Args:
            notification_type: Тип уведомления ('email', 'sms', 'push', 'slack')
            **kwargs: Параметры для конкретного типа уведомления
        
        Returns:
            Объект уведомления
        
        Raises:
            ValueError: Если тип уведомления неизвестен
        """
        if notification_type == "email":
            return EmailNotification(
                recipient=kwargs.get("recipient", "default@example.com"),
                subject=kwargs.get("subject", "Default Subject"),
                body=kwargs.get("body", "Default body"),
                sender=kwargs.get("sender", "noreply@example.com")
            )
        elif notification_type == "sms":
            return SMSNotification(
                phone_number=kwargs.get("phone_number", "+1234567890"),
                message=kwargs.get("message", "Default SMS message")
            )
        elif notification_type == "push":
            return PushNotification(
                device_token=kwargs.get("device_token", "default_token_1234567890"),
                title=kwargs.get("title", "Default Title"),
                body=kwargs.get("body", "Default push body")
            )
        elif notification_type == "slack":
            return SlackNotification(
                webhook_url=kwargs.get("webhook_url", "https://hooks.slack.com/default"),
                channel=kwargs.get("channel", "general"),
                message=kwargs.get("message", "Default Slack message")
            )
        else:
            raise ValueError(f"Unknown notification type: {notification_type}")
    
    @staticmethod
    def create_batch(notifications_config: List[Dict]) -> List[Notification]:
        """
        Создаёт несколько уведомлений из конфигурации.
        
        Args:
            notifications_config: Список словарей с параметрами
        
        Returns:
            Список созданных уведомлений
        """
        notifications = []
        for config in notifications_config:
            notification_type = config.pop("type")
            notifications.append(NotificationFactory.create(notification_type, **config))
        return notifications
    
    @staticmethod
    def get_supported_types() -> List[str]:
        """Возвращает список поддерживаемых типов уведомлений."""
        return ["email", "sms", "push", "slack"]
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Тестирование создания отдельных типов
Средний (варианты 9-17)	Средние студенты	+ обработка ошибок + параметризация
Сложный (варианты 18-25)	Сильные студенты	+ create_batch + полиморфная интеграция
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться тестировать создание объектов разных типов через фабрику.

Задания для базового уровня
Задание 1.1. Создание базовых тестов для фабрики (15 минут)
python
# test_factory_base.py

import pytest
from inheritance_example import (
    NotificationFactory, 
    EmailNotification, 
    SMSNotification, 
    PushNotification, 
    SlackNotification
)


class TestNotificationFactoryBase:
    """Базовые тесты для фабрики уведомлений."""
    
    # ==========================================================
    # ТЕСТЫ СОЗДАНИЯ ОТДЕЛЬНЫХ ТИПОВ
    # ==========================================================
    
    def test_create_email(self):
        """Создание email уведомления."""
        notification = NotificationFactory.create("email")
        
        assert isinstance(notification, EmailNotification)
        assert notification.get_type() == "email"
    
    def test_create_sms(self):
        """Создание SMS уведомления."""
        notification = NotificationFactory.create("sms")
        
        assert isinstance(notification, SMSNotification)
        assert notification.get_type() == "sms"
    
    def test_create_push(self):
        """Создание push уведомления."""
        notification = NotificationFactory.create("push")
        
        assert isinstance(notification, PushNotification)
        assert notification.get_type() == "push"
    
    def test_create_slack(self):
        """Создание slack уведомления."""
        notification = NotificationFactory.create("slack")
        
        assert isinstance(notification, SlackNotification)
        assert notification.get_type() == "slack"
    
    # ==========================================================
    # ТЕСТЫ С ПОЛЬЗОВАТЕЛЬСКИМИ ПАРАМЕТРАМИ
    # ==========================================================
    
    def test_create_email_with_custom_params(self):
        """Создание email с пользовательскими параметрами."""
        notification = NotificationFactory.create(
            "email",
            recipient="custom@test.com",
            subject="Custom Subject",
            body="Custom body",
            sender="custom@example.com"
        )
        
        assert notification.recipient == "custom@test.com"
        assert notification.subject == "Custom Subject"
        assert notification.body == "Custom body"
        assert notification.sender == "custom@example.com"
    
    def test_create_sms_with_custom_params(self):
        """Создание SMS с пользовательскими параметрами."""
        notification = NotificationFactory.create(
            "sms",
            phone_number="+79991234567",
            message="Custom SMS"
        )
        
        assert notification.recipient == "+79991234567"
        assert notification.message == "Custom SMS"
    
    def test_create_push_with_custom_params(self):
        """Создание push с пользовательскими параметрами."""
        notification = NotificationFactory.create(
            "push",
            device_token="custom_token_123",
            title="Custom Title",
            body="Custom push body"
        )
        
        assert notification.recipient == "custom_token_123"
        assert notification.title == "Custom Title"
        assert notification.body == "Custom push body"
    
    def test_create_slack_with_custom_params(self):
        """Создание slack с пользовательскими параметрами."""
        notification = NotificationFactory.create(
            "slack",
            webhook_url="https://custom.hooks.slack.com/xxx",
            channel="alerts",
            message="Custom Slack message"
        )
        
        assert notification.recipient == "https://custom.hooks.slack.com/xxx"
        assert notification.channel == "alerts"
        assert notification.message == "Custom Slack message"
    
    # ==========================================================
    # ТЕСТЫ ЗНАЧЕНИЙ ПО УМОЛЧАНИЮ
    # ==========================================================
    
    def test_email_default_values(self):
        """Проверка значений по умолчанию для email."""
        notification = NotificationFactory.create("email")
        
        assert notification.recipient == "default@example.com"
        assert notification.subject == "Default Subject"
        assert notification.body == "Default body"
        assert notification.sender == "noreply@example.com"
    
    def test_sms_default_values(self):
        """Проверка значений по умолчанию для SMS."""
        notification = NotificationFactory.create("sms")
        
        assert notification.recipient == "+1234567890"
        assert notification.message == "Default SMS message"
    
    def test_push_default_values(self):
        """Проверка значений по умолчанию для push."""
        notification = NotificationFactory.create("push")
        
        assert notification.recipient == "default_token_1234567890"
        assert notification.title == "Default Title"
        assert notification.body == "Default push body"
    
    def test_slack_default_values(self):
        """Проверка значений по умолчанию для slack."""
        notification = NotificationFactory.create("slack")
        
        assert notification.recipient == "https://hooks.slack.com/default"
        assert notification.channel == "general"
        assert notification.message == "Default Slack message"
Задание 1.2. Тестирование метода get_supported_types (5 минут)
python
class TestGetSupportedTypes:
    """Тесты для get_supported_types."""
    
    def test_get_supported_types_returns_list(self):
        """get_supported_types возвращает список."""
        types = NotificationFactory.get_supported_types()
        
        assert isinstance(types, list)
    
    def test_get_supported_types_contains_all_types(self):
        """Список содержит все поддерживаемые типы."""
        types = NotificationFactory.get_supported_types()
        
        assert "email" in types
        assert "sms" in types
        assert "push" in types
        assert "slack" in types
    
    def test_get_supported_types_count(self):
        """Количество поддерживаемых типов."""
        types = NotificationFactory.get_supported_types()
        
        assert len(types) == 4
Задание 1.3. Тестирование неизвестного типа (5 минут)
python
class TestUnknownType:
    """Тесты обработки неизвестного типа."""
    
    def test_create_unknown_type_raises_error(self):
        """Создание неизвестного типа вызывает ValueError."""
        with pytest.raises(ValueError) as exc_info:
            NotificationFactory.create("unknown")
        
        assert "Unknown notification type: unknown" in str(exc_info.value)
    
    def test_create_invalid_type_string(self):
        """Пустая строка как тип."""
        with pytest.raises(ValueError):
            NotificationFactory.create("")
    
    def test_create_none_type(self):
        """None как тип."""
        with pytest.raises(ValueError):
            NotificationFactory.create(None)
Задание 1.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании фабрики (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Результаты тестирования

| Тип уведомления | Создание | Параметры | Значения по умолчанию |
|:---|:---|:---|:---|
| email | ☐ | ☐ | ☐ |
| sms | ☐ | ☐ | ☐ |
| push | ☐ | ☐ | ☐ |
| slack | ☐ | ☐ | ☐ |

### Проверка неизвестных типов

| Тип | Ожидаемое поведение | Статус |
|:---|:---|:---|
| "unknown" | ValueError | ☐ |
| "" | ValueError | ☐ |
| None | ValueError | ☐ |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться использовать параметризацию для тестирования фабрики и проверять частичные параметры.

Задания для среднего уровня
Задание 2.1. Параметризация тестов фабрики (15 минут)
python
# test_factory_intermediate.py

import pytest
from inheritance_example import NotificationFactory


class TestFactoryParametrized:
    """Параметризованные тесты для фабрики."""
    
    @pytest.mark.parametrize("notification_type,expected_class", [
        ("email", EmailNotification),
        ("sms", SMSNotification),
        ("push", PushNotification),
        ("slack", SlackNotification),
    ])
    def test_create_returns_correct_type(self, notification_type, expected_class):
        """Параметризованный тест типа возвращаемого объекта."""
        notification = NotificationFactory.create(notification_type)
        
        assert isinstance(notification, expected_class)
        assert notification.get_type() == notification_type
    
    @pytest.mark.parametrize("notification_type,expected_type_str", [
        ("email", "email"),
        ("sms", "sms"),
        ("push", "push"),
        ("slack", "slack"),
    ])
    def test_get_type_returns_correct_string(self, notification_type, expected_type_str):
        """Параметризованный тест get_type."""
        notification = NotificationFactory.create(notification_type)
        
        assert notification.get_type() == expected_type_str
    
    @pytest.mark.parametrize("notification_type,key_param,default_value", [
        ("email", "recipient", "default@example.com"),
        ("email", "subject", "Default Subject"),
        ("sms", "recipient", "+1234567890"),
        ("sms", "message", "Default SMS message"),
        ("push", "recipient", "default_token_1234567890"),
        ("push", "title", "Default Title"),
        ("slack", "channel", "general"),
    ])
    def test_default_values_parametrized(self, notification_type, key_param, default_value):
        """Параметризованный тест значений по умолчанию."""
        notification = NotificationFactory.create(notification_type)
        
        assert getattr(notification, key_param) == default_value
    
    @pytest.mark.parametrize("notification_type,partial_params", [
        ("email", {"recipient": "partial@test.com"}),
        ("sms", {"phone_number": "+79991234567"}),
        ("push", {"device_token": "partial_token"}),
        ("slack", {"channel": "partial_channel"}),
    ])
    def test_partial_parameters(self, notification_type, partial_params):
        """Тест с частичными параметрами (остальные — по умолчанию)."""
        notification = NotificationFactory.create(notification_type, **partial_params)
        
        # Проверяем, что переданные параметры установлены
        for key, value in partial_params.items():
            # Находим соответствующий атрибут
            if notification_type == "email" and key == "recipient":
                assert notification.recipient == value
            elif notification_type == "sms" and key == "phone_number":
                assert notification.recipient == value
            elif notification_type == "push" and key == "device_token":
                assert notification.recipient == value
            elif notification_type == "slack" and key == "channel":
                assert notification.channel == value
Задание 2.2. Тестирование обработки ошибок (10 минут)
python
class TestFactoryErrorHandling:
    """Тесты обработки ошибок фабрики."""
    
    @pytest.mark.parametrize("invalid_type", [
        "EMAIL", "Email", "MAIL", "sms!", " push ", "slack ", " telegram"
    ])
    def test_case_sensitive_types(self, invalid_type):
        """Типы чувствительны к регистру."""
        with pytest.raises(ValueError, match=f"Unknown notification type: {invalid_type}"):
            NotificationFactory.create(invalid_type)
    
    @pytest.mark.parametrize("extra_params", [
        {"extra": "value"},
        {"unexpected": 123},
        {"random_key": True},
        {"another": "param", "and": "another"},
    ])
    def test_extra_parameters_ignored(self, extra_params):
        """Лишние параметры не должны вызывать ошибку."""
        # Должно работать даже с лишними параметрами
        notification = NotificationFactory.create("email", **extra_params)
        
        assert isinstance(notification, EmailNotification)
    
    def test_missing_required_params(self):
        """Отсутствие обязательных параметров (должны быть значения по умолчанию)."""
        # У email есть значения по умолчанию для всех параметров
        notification = NotificationFactory.create("email")
        assert notification is not None
        
        # У sms есть значения по умолчанию
        notification = NotificationFactory.create("sms")
        assert notification is not None
    
    @pytest.mark.parametrize("notification_type,invalid_param,invalid_value", [
        ("email", "recipient", ""),
        ("sms", "phone_number", ""),
        ("push", "device_token", ""),
        ("slack", "webhook_url", ""),
    ])
    def test_empty_string_params(self, notification_type, invalid_param, invalid_value):
        """Пустые строки в параметрах."""
        params = {invalid_param: invalid_value}
        notification = NotificationFactory.create(notification_type, **params)
        
        # Объект должен создаться, но send() должен вернуть False
        assert notification.send() is False
Задание 2.3. Тестирование валидации через фабрику (10 минут)
python
class TestFactoryValidation:
    """Тесты валидации через фабрику."""
    
    @pytest.mark.parametrize("notification_type,valid_params", [
        ("email", {"recipient": "valid@test.com", "subject": "Test", "body": "Hello"}),
        ("sms", {"phone_number": "+1234567890", "message": "Hello"}),
        ("push", {"device_token": "valid_token_1234567890", "title": "Test", "body": "Body"}),
        ("slack", {"webhook_url": "https://hooks.slack.com/valid", "channel": "test", "message": "Hi"}),
    ])
    def test_create_valid_notification_sends_success(self, notification_type, valid_params):
        """Создание валидного уведомления должно успешно отправляться."""
        notification = NotificationFactory.create(notification_type, **valid_params)
        
        result = notification.send()
        
        assert result is True
        assert notification.is_sent() is True
    
    @pytest.mark.parametrize("notification_type,invalid_params", [
        ("email", {"recipient": "invalid"}),
        ("sms", {"phone_number": "123"}),
        ("push", {"device_token": "short"}),
        ("slack", {"webhook_url": "invalid"}),
    ])
    def test_create_invalid_notification_send_fails(self, notification_type, invalid_params):
        """Создание невалидного уведомления не должно отправляться успешно."""
        notification = NotificationFactory.create(notification_type, **invalid_params)
        
        result = notification.send()
        
        assert result is False
        assert notification.is_sent() is False
Задание 2.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании фабрики (средний уровень)

**Студент:** _________________
**Вариант №:** ___ (9-17)

### Результаты параметризованных тестов

| Тип | Тип класса | Default values | Частичные параметры |
|:---|:---|:---|:---|
| email | ☐ | ☐ | ☐ |
| sms | ☐ | ☐ | ☐ |
| push | ☐ | ☐ | ☐ |
| slack | ☐ | ☐ | ☐ |

### Обработка ошибок

| Сценарий | Результат |
|:---|:---|
| Неизвестный тип | ValueError |
| Чувствительность к регистру | ☐ |
| Лишние параметры | Игнорируются |
| Пустые строки | Объект создаётся, send=False |

### Вывод

_______________________________________________________________
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться тестировать метод create_batch и интегрировать фабрику с полиморфными тестами.

Задания для сложного уровня
Задание 3.1. Тестирование create_batch (15 минут)
python
# test_factory_advanced.py

import pytest
from inheritance_example import NotificationFactory, EmailNotification, SMSNotification


class TestFactoryBatch:
    """Тесты для метода create_batch."""
    
    def test_create_batch_empty_list(self):
        """Пустой список конфигураций."""
        result = NotificationFactory.create_batch([])
        
        assert result == []
    
    def test_create_batch_single_notification(self):
        """Пакет с одним уведомлением."""
        configs = [
            {"type": "email", "recipient": "test@test.com"}
        ]
        
        result = NotificationFactory.create_batch(configs)
        
        assert len(result) == 1
        assert isinstance(result[0], EmailNotification)
        assert result[0].recipient == "test@test.com"
    
    def test_create_batch_multiple_notifications(self):
        """Пакет с несколькими уведомлениями разных типов."""
        configs = [
            {"type": "email", "recipient": "alice@test.com"},
            {"type": "sms", "phone_number": "+1234567890"},
            {"type": "push", "device_token": "token123"},
        ]
        
        result = NotificationFactory.create_batch(configs)
        
        assert len(result) == 3
        assert result[0].get_type() == "email"
        assert result[1].get_type() == "sms"
        assert result[2].get_type() == "push"
    
    def test_create_batch_preserves_order(self):
        """Порядок уведомлений сохраняется."""
        configs = [
            {"type": "email"},
            {"type": "push"},
            {"type": "sms"},
            {"type": "slack"},
        ]
        
        result = NotificationFactory.create_batch(configs)
        
        assert result[0].get_type() == "email"
        assert result[1].get_type() == "push"
        assert result[2].get_type() == "sms"
        assert result[3].get_type() == "slack"
    
    def test_create_batch_with_unknown_type_raises(self):
        """Неизвестный тип в пакете вызывает ошибку."""
        configs = [
            {"type": "email"},
            {"type": "unknown"},
            {"type": "sms"},
        ]
        
        with pytest.raises(ValueError, match="Unknown notification type: unknown"):
            NotificationFactory.create_batch(configs)
    
    def test_create_batch_with_mixed_valid_invalid(self):
        """При ошибке в одном элементе весь пакет не создаётся."""
        configs = [
            {"type": "email"},
            {"type": "invalid_type"},
        ]
        
        with pytest.raises(ValueError):
            NotificationFactory.create_batch(configs)
    
    @pytest.mark.parametrize("configs,expected_types", [
        ([{"type": "email"}], ["email"]),
        ([{"type": "sms"}], ["sms"]),
        ([{"type": "push"}], ["push"]),
        ([{"type": "slack"}], ["slack"]),
        ([{"type": "email"}, {"type": "sms"}], ["email", "sms"]),
    ])
    def test_create_batch_parametrized(self, configs, expected_types):
        """Параметризованный тест create_batch."""
        result = NotificationFactory.create_batch(configs)
        
        assert len(result) == len(expected_types)
        for i, notification in enumerate(result):
            assert notification.get_type() == expected_types[i]
Задание 3.2. Интеграция фабрики с полиморфными тестами (15 минут)
python
class TestFactoryPolymorphicIntegration:
    """Интеграция фабрики с полиморфными тестами."""
    
    @pytest.fixture
    def all_notifications_from_factory(self):
        """Фикстура, создающая все типы уведомлений через фабрику."""
        return [
            NotificationFactory.create("email"),
            NotificationFactory.create("sms"),
            NotificationFactory.create("push"),
            NotificationFactory.create("slack"),
        ]
    
    def test_polymorphic_behavior_with_factory(self, all_notifications_from_factory):
        """Проверка полиморфного поведения объектов, созданных фабрикой."""
        for notification in all_notifications_from_factory:
            # Все уведомления должны иметь общие методы
            assert hasattr(notification, "send")
            assert hasattr(notification, "get_type")
            assert hasattr(notification, "format_message")
            assert hasattr(notification, "get_info")
            
            # get_type возвращает непустую строку
            assert isinstance(notification.get_type(), str)
            assert len(notification.get_type()) > 0
    
    def test_factory_notifications_can_be_sent_polymorphically(self):
        """Уведомления из фабрики можно отправлять полиморфно."""
        types = ["email", "sms", "push", "slack"]
        
        for t in types:
            notification = NotificationFactory.create(t)
            # Отправка не должна вызывать ошибку
            result = notification.send()
            assert isinstance(result, bool)
    
    def test_factory_and_direct_creation_equivalent(self):
        """Объекты из фабрики эквивалентны созданным напрямую."""
        # Создаём email напрямую
        direct = EmailNotification("test@test.com", "Subject", "Body", "sender@test.com")
        
        # Создаём через фабрику
        factory_created = NotificationFactory.create(
            "email",
            recipient="test@test.com",
            subject="Subject",
            body="Body",
            sender="sender@test.com"
        )
        
        assert direct.recipient == factory_created.recipient
        assert direct.subject == factory_created.subject
        assert direct.body == factory_created.body
        assert direct.sender == factory_created.sender
    
    def test_bulk_send_from_factory_batch(self):
        """Массовая отправка уведомлений, созданных через фабрику."""
        configs = [
            {"type": "email", "recipient": "user1@test.com"},
            {"type": "sms", "phone_number": "+1234567890"},
            {"type": "push", "device_token": "token1234567890"},
        ]
        
        notifications = NotificationFactory.create_batch(configs)
        results = []
        
        for notification in notifications:
            results.append({
                "type": notification.get_type(),
                "sent": notification.send()
            })
        
        assert len(results) == 3
        for result in results:
            assert "type" in result
            assert "sent" in result
Задание 3.3. Комплексное тестирование фабрики (10 минут)
python
class TestFactoryComplex:
    """Комплексные тесты фабрики."""
    
    def test_factory_creates_notifications_for_all_supported_types(self):
        """Фабрика создаёт уведомления для всех поддерживаемых типов."""
        supported = NotificationFactory.get_supported_types()
        
        for notification_type in supported:
            notification = NotificationFactory.create(notification_type)
            assert notification is not None
            assert notification.get_type() == notification_type
    
    def test_factory_notifications_are_independent(self):
        """Уведомления, созданные фабрикой, независимы."""
        email1 = NotificationFactory.create("email", recipient="user1@test.com")
        email2 = NotificationFactory.create("email", recipient="user2@test.com")
        
        email1.send()
        
        assert email1.is_sent() is True
        assert email2.is_sent() is False
    
    def test_factory_type_registry_consistency(self):
        """Типы из get_supported_types совпадают с реально создаваемыми."""
        supported = NotificationFactory.get_supported_types()
        created_types = []
        
        for t in supported:
            notification = NotificationFactory.create(t)
            created_types.append(notification.get_type())
        
        assert supported == created_types
    
    @pytest.mark.parametrize("notification_type", [
        "email", "sms", "push", "slack"
    ])
    def test_factory_notification_can_be_serialized(self, notification_type):
        """Уведомления из фабрики можно сериализовать."""
        notification = NotificationFactory.create(notification_type)
        info = notification.get_info()
        
        assert isinstance(info, dict)
        assert info["type"] == notification_type
        assert "recipient" in info
        assert "created_at" in info
Задание 3.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании фабрики (сложный уровень)

**Студент:** _________________
**Вариант №:** ___ (18-25)

### Результаты тестирования

| Компонент | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| create (отдельные типы) | _____ | _____ | _____ |
| create (параметризация) | _____ | _____ | _____ |
| create_batch | _____ | _____ | _____ |
| Полное покрытие фабрики | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Проверка интеграции

| Интеграция | Статус |
|:---|:---|
| Фабрика + Полиморфные тесты | ☐ |
| Фабрика + Batch операции | ☐ |
| Фабрика + Сериализация | ☐ |

### Покрытие сценариев использования фабрики

| Сценарий | Покрыт |
|:---|:---|
| Создание одного уведомления | ☐ |
| Создание уведомления с параметрами | ☐ |
| Создание уведомления с параметрами по умолчанию | ☐ |
| Создание пакета уведомлений | ☐ |
| Обработка неизвестного типа | ☐ |
| Обработка пустого пакета | ☐ |
| Получение списка поддерживаемых типов | ☐ |

### Выводы

_______________________________________________________________

_______________________________________________________________

### Рекомендации по улучшению фабрики

1. 
2. 
3.
Карточка студента
text
ПР 3.14. ТЕСТИРОВАНИЕ ПАТТЕРНА FACTORY

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ Тестирование create для email (базовый)
□ Тестирование create для sms/push/slack (базовый)
□ Тестирование get_supported_types (базовый)
□ Параметризация тестов (средний)
□ Тестирование обработки ошибок (средний)
□ Тестирование create_batch (сложный)
□ Полиморфная интеграция (сложный)
□ Комплексные тесты (сложный)

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
- test_factory_base.py
- test_factory_intermediate.py
- test_factory_advanced.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Тесты не работают или фабрика не тестируется
3 (удовлетворительно)	Базовый	Все 4 типа создаются, тесты на параметры (10+ тестов)
4 (хорошо)	Средний	+ Параметризация + обработка ошибок (20+ тестов)
5 (отлично)	Сложный	+ create_batch + полиморфная интеграция (30+ тестов)
Контрольные вопросы (для защиты)
Какие преимущества даёт использование фабрики для тестирования?

Как проверить, что фабрика создаёт объект правильного типа?

Как тестировать значения по умолчанию в фабрике?

Как параметризация помогает в тестировании фабрики?

Как проверить обработку неизвестного типа?

Как протестировать метод create_batch?

Как интегрировать фабрику с полиморфными тестами?

Почему важно тестировать не только создание, но и поведение созданных объектов?

Как проверить, что фабрика не изменяет переданные параметры?

Какие тесты нужны для поддержки нового типа уведомлений?

Следующее занятие: Контрольная работа 3.3 по темам 3.9–3.14.
