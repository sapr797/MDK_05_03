# Практическое занятие 3.13: Тестирование иерархий классов и полиморфизма

**Дисциплина:** МДК 05.03 Тестирование информационных систем  
**Семестр:** 6  
**Тема 3.8:** Тестирование классов и объектов  
**Тип занятия:** Практическое (4 часа по УП 2.1)

---

## Цель практического занятия

Научиться тестировать иерархии классов с использованием полиморфного подхода, применять параметризацию по классам для сокращения дублирования тестов и проверять соблюдение контракта наследования.

## Планируемые результаты

После выполнения практической работы вы сможете:
1. Тестировать иерархии классов с общим интерфейсом (ПК 3.2, ПК 3.3).
2. Применять параметризацию по классам для полиморфного тестирования (ПК 3.2).
3. Проверять соблюдение принципа подстановки Лисков (LSP) (ОК 02, ОК 05).
4. Создавать базовые тестовые классы для иерархий (ПК 3.2).

---

## Теоретическая справка

### Принцип подстановки Лисков (LSP)

Принцип гласит: **объекты производного класса должны быть заменяемы объектами базового класса без изменения корректности программы.**

```python
# ❌ Нарушение LSP
class Bird:
    def fly(self): return "Flying"

class Penguin(Bird):
    def fly(self): raise NotImplementedError()  # Пингвины не летают!

# ✅ Соблюдение LSP
class Bird:
    def move(self): return "Moving"

class Sparrow(Bird):
    def move(self): return "Flying"

class Penguin(Bird):
    def move(self): return "Swimming"
Полиморфное тестирование
Полиморфное тестирование позволяет проверить, что все классы иерархии ведут себя предсказуемо через единый интерфейс.

Объект тестирования
Листинг 1. Модуль inheritance_example.py
python
# inheritance_example.py

from abc import ABC, abstractmethod
from typing import Dict, Any, Optional
from datetime import datetime


class Notification(ABC):
    """
    Абстрактный базовый класс для всех уведомлений.
    """
    
    def __init__(self, recipient: str, subject: str):
        self.recipient = recipient
        self.subject = subject
        self.created_at = datetime.now()
        self._sent = False
    
    @abstractmethod
    def send(self) -> bool:
        """Отправляет уведомление. Возвращает True при успехе."""
        pass
    
    @abstractmethod
    def get_type(self) -> str:
        """Возвращает тип уведомления."""
        pass
    
    @abstractmethod
    def format_message(self) -> str:
        """Форматирует сообщение для отправки."""
        pass
    
    def is_sent(self) -> bool:
        """Проверяет, было ли отправлено уведомление."""
        return self._sent
    
    def get_info(self) -> Dict[str, Any]:
        """Возвращает информацию об уведомлении."""
        return {
            "type": self.get_type(),
            "recipient": self.recipient,
            "subject": self.subject,
            "sent": self._sent,
            "created_at": self.created_at.isoformat()
        }


class EmailNotification(Notification):
    """Email уведомление."""
    
    def __init__(self, recipient: str, subject: str, body: str, sender: str = "noreply@example.com"):
        super().__init__(recipient, subject)
        self.body = body
        self.sender = sender
    
    def send(self) -> bool:
        """Имитация отправки email."""
        if not self.recipient or "@" not in self.recipient:
            return False
        
        if not self.body:
            return False
        
        # Имитация успешной отправки
        self._sent = True
        return True
    
    def get_type(self) -> str:
        return "email"
    
    def format_message(self) -> str:
        return f"To: {self.recipient}\nFrom: {self.sender}\nSubject: {self.subject}\n\n{self.body}"
    
    def get_sender(self) -> str:
        """Дополнительный метод, специфичный для Email."""
        return self.sender


class SMSNotification(Notification):
    """SMS уведомление."""
    
    def __init__(self, phone_number: str, message: str):
        super().__init__(phone_number, "SMS")
        self.message = message
    
    def send(self) -> bool:
        """Имитация отправки SMS."""
        if not self.recipient or len(self.recipient) < 10:
            return False
        
        if not self.message or len(self.message) > 160:
            return False
        
        self._sent = True
        return True
    
    def get_type(self) -> str:
        return "sms"
    
    def format_message(self) -> str:
        return f"SMS to {self.recipient}: {self.message}"
    
    def get_character_count(self) -> int:
        """Дополнительный метод, специфичный для SMS."""
        return len(self.message)


class PushNotification(Notification):
    """Push-уведомление для мобильного приложения."""
    
    def __init__(self, device_token: str, title: str, body: str):
        super().__init__(device_token, title)
        self.body = body
        self.title = title
    
    def send(self) -> bool:
        """Имитация отправки push-уведомления."""
        if not self.recipient or len(self.recipient) < 10:
            return False
        
        if not self.body:
            return False
        
        self._sent = True
        return True
    
    def get_type(self) -> str:
        return "push"
    
    def format_message(self) -> str:
        return f"Push to {self.recipient}: {self.title} - {self.body}"
    
    def get_title(self) -> str:
        """Дополнительный метод, специфичный для Push."""
        return self.title


class SlackNotification(Notification):
    """Уведомление в Slack."""
    
    def __init__(self, webhook_url: str, channel: str, message: str):
        super().__init__(webhook_url, f"Slack message to {channel}")
        self.channel = channel
        self.message = message
    
    def send(self) -> bool:
        """Имитация отправки в Slack."""
        if not self.recipient or not self.recipient.startswith("https://"):
            return False
        
        if not self.message:
            return False
        
        self._sent = True
        return True
    
    def get_type(self) -> str:
        return "slack"
    
    def format_message(self) -> str:
        return f"Slack #{self.channel}: {self.message}"
    
    def get_channel(self) -> str:
        """Дополнительный метод, специфичный для Slack."""
        return self.channel
Уровни сложности заданий
Уровень	Студенты	Что нужно сделать
Базовый (варианты 1-8)	Слабые студенты	Тестирование одного класса иерархии
Средний (варианты 9-17)	Средние студенты	Базовый тестовый класс + параметризация
Сложный (варианты 18-25)	Сильные студенты	Полная параметризация + проверка LSP + фабрика
БАЗОВЫЙ УРОВЕНЬ (Варианты 1-8)
Цель для базового уровня
Научиться тестировать отдельные классы иерархии уведомлений.

Задания для базового уровня
Задание 1.1. Тестирование EmailNotification (15 минут)
python
# test_inheritance_base.py

import pytest
from datetime import datetime
from inheritance_example import EmailNotification, SMSNotification, PushNotification, SlackNotification


class TestEmailNotification:
    """Тесты для EmailNotification."""
    
    # ==========================================================
    # ТЕСТЫ КОНСТРУКТОРА
    # ==========================================================
    
    def test_constructor_sets_attributes(self):
        """Проверка установки атрибутов конструктором."""
        email = EmailNotification("user@test.com", "Hello", "Body text", "sender@test.com")
        
        assert email.recipient == "user@test.com"
        assert email.subject == "Hello"
        assert email.body == "Body text"
        assert email.sender == "sender@test.com"
        assert email.is_sent() is False
        assert isinstance(email.created_at, datetime)
    
    def test_constructor_default_sender(self):
        """Проверка значения отправителя по умолчанию."""
        email = EmailNotification("user@test.com", "Hello", "Body")
        
        assert email.sender == "noreply@example.com"
    
    # ==========================================================
    # ТЕСТЫ МЕТОДОВ
    # ==========================================================
    
    def test_get_type(self):
        """Проверка типа уведомления."""
        email = EmailNotification("test@test.com", "Subj", "Body")
        assert email.get_type() == "email"
    
    def test_format_message(self):
        """Проверка форматирования сообщения."""
        email = EmailNotification("user@test.com", "Hello", "Test body", "sender@test.com")
        
        formatted = email.format_message()
        
        assert "To: user@test.com" in formatted
        assert "From: sender@test.com" in formatted
        assert "Subject: Hello" in formatted
        assert "Test body" in formatted
    
    def test_send_success(self):
        """Успешная отправка email."""
        email = EmailNotification("user@test.com", "Hello", "Body")
        
        result = email.send()
        
        assert result is True
        assert email.is_sent() is True
    
    def test_send_fails_invalid_recipient(self):
        """Отправка с неверным email."""
        email = EmailNotification("invalid", "Hello", "Body")
        
        result = email.send()
        
        assert result is False
        assert email.is_sent() is False
    
    def test_send_fails_empty_body(self):
        """Отправка с пустым телом письма."""
        email = EmailNotification("user@test.com", "Hello", "")
        
        result = email.send()
        
        assert result is False
        assert email.is_sent() is False
    
    def test_get_info(self):
        """Проверка получения информации."""
        email = EmailNotification("user@test.com", "Test", "Body")
        
        info = email.get_info()
        
        assert info["type"] == "email"
        assert info["recipient"] == "user@test.com"
        assert info["subject"] == "Test"
        assert info["sent"] is False
        assert "created_at" in info
    
    def test_get_sender_specific_method(self):
        """Проверка специфичного метода EmailNotification."""
        email = EmailNotification("user@test.com", "Subj", "Body", "custom@test.com")
        
        assert email.get_sender() == "custom@test.com"
Задание 1.2. Тестирование SMSNotification (10 минут)
python
class TestSMSNotification:
    """Тесты для SMSNotification."""
    
    def test_constructor_sets_attributes(self):
        """Проверка установки атрибутов."""
        sms = SMSNotification("+1234567890", "Hello world")
        
        assert sms.recipient == "+1234567890"
        assert sms.subject == "SMS"
        assert sms.message == "Hello world"
        assert sms.is_sent() is False
    
    def test_get_type(self):
        """Проверка типа."""
        sms = SMSNotification("+1234567890", "Hi")
        assert sms.get_type() == "sms"
    
    def test_format_message(self):
        """Проверка форматирования."""
        sms = SMSNotification("+1234567890", "Hello")
        
        formatted = sms.format_message()
        
        assert "SMS to +1234567890: Hello" in formatted
    
    def test_send_success(self):
        """Успешная отправка SMS."""
        sms = SMSNotification("+1234567890", "Hello")
        
        result = sms.send()
        
        assert result is True
        assert sms.is_sent() is True
    
    def test_send_fails_short_number(self):
        """Номер телефона слишком короткий."""
        sms = SMSNotification("123", "Hello")
        
        result = sms.send()
        
        assert result is False
        assert sms.is_sent() is False
    
    def test_send_fails_empty_message(self):
        """Пустое сообщение."""
        sms = SMSNotification("+1234567890", "")
        
        result = sms.send()
        
        assert result is False
    
    def test_send_fails_message_too_long(self):
        """Сообщение длиннее 160 символов."""
        long_message = "a" * 161
        sms = SMSNotification("+1234567890", long_message)
        
        result = sms.send()
        
        assert result is False
    
    def test_get_character_count(self):
        """Проверка подсчёта символов."""
        sms = SMSNotification("+1234567890", "Hello")
        
        assert sms.get_character_count() == 5
Задание 1.3. Тестирование PushNotification и SlackNotification (10 минут)
python
class TestPushNotification:
    """Тесты для PushNotification."""
    
    def test_constructor(self):
        push = PushNotification("device_token_123", "Title", "Body")
        
        assert push.recipient == "device_token_123"
        assert push.subject == "Title"
        assert push.body == "Body"
    
    def test_get_type(self):
        push = PushNotification("token", "Title", "Body")
        assert push.get_type() == "push"
    
    def test_send_success(self):
        push = PushNotification("device_token_1234567890", "Title", "Body")
        
        result = push.send()
        
        assert result is True
        assert push.is_sent() is True
    
    def test_send_fails_short_token(self):
        push = PushNotification("short", "Title", "Body")
        
        result = push.send()
        
        assert result is False
    
    def test_get_title(self):
        push = PushNotification("token", "My Title", "Body")
        assert push.get_title() == "My Title"
    
    def test_format_message(self):
        push = PushNotification("token123", "Alert", "New message")
        
        formatted = push.format_message()
        
        assert "Push to token123: Alert - New message" in formatted


class TestSlackNotification:
    """Тесты для SlackNotification."""
    
    def test_constructor(self):
        slack = SlackNotification("https://hooks.slack.com/xxx", "general", "Hello team")
        
        assert slack.recipient == "https://hooks.slack.com/xxx"
        assert slack.channel == "general"
        assert slack.message == "Hello team"
    
    def test_get_type(self):
        slack = SlackNotification("https://hooks.slack.com/xxx", "general", "Hi")
        assert slack.get_type() == "slack"
    
    def test_send_success(self):
        slack = SlackNotification("https://hooks.slack.com/valid", "general", "Message")
        
        result = slack.send()
        
        assert result is True
        assert slack.is_sent() is True
    
    def test_send_fails_invalid_webhook(self):
        slack = SlackNotification("invalid-url", "general", "Message")
        
        result = slack.send()
        
        assert result is False
    
    def test_send_fails_empty_message(self):
        slack = SlackNotification("https://hooks.slack.com/xxx", "general", "")
        
        result = slack.send()
        
        assert result is False
    
    def test_get_channel(self):
        slack = SlackNotification("https://hooks.slack.com/xxx", "random", "Hi")
        assert slack.get_channel() == "random"
    
    def test_format_message(self):
        slack = SlackNotification("https://hooks.slack.com/xxx", "general", "Hello")
        
        formatted = slack.format_message()
        
        assert "Slack #general: Hello" in formatted
Задание 1.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (базовый уровень)

**Студент:** _________________
**Вариант №:** ___ (1-8)

### Результаты тестирования классов

| Класс | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| EmailNotification | _____ | _____ | _____ |
| SMSNotification | _____ | _____ | _____ |
| PushNotification | _____ | _____ | _____ |
| SlackNotification | _____ | _____ | _____ |

### Вывод

_______________________________________________________________
СРЕДНИЙ УРОВЕНЬ (Варианты 9-17)
Цель для среднего уровня
Научиться создавать базовый тестовый класс для иерархии и использовать параметризацию.

Задания для среднего уровня
Задание 2.1. Базовый тестовый класс для всех уведомлений (15 минут)
python
# test_inheritance_intermediate.py

import pytest
from abc import ABC, abstractmethod
from inheritance_example import (
    Notification, EmailNotification, SMSNotification, 
    PushNotification, SlackNotification
)


class BaseNotificationTest(ABC):
    """
    Абстрактный базовый класс для тестирования всех уведомлений.
    Все наследники должны реализовать create_notification.
    """
    
    @abstractmethod
    def create_notification(self) -> Notification:
        """Создаёт экземпляр уведомления для тестирования."""
        pass
    
    def test_initial_state(self):
        """Проверка начального состояния уведомления."""
        notification = self.create_notification()
        
        assert notification.is_sent() is False
        assert isinstance(notification.created_at, type(datetime.now()))
    
    def test_get_type_returns_string(self):
        """get_type возвращает непустую строку."""
        notification = self.create_notification()
        
        type_str = notification.get_type()
        assert isinstance(type_str, str)
        assert len(type_str) > 0
    
    def test_format_message_returns_string(self):
        """format_message возвращает непустую строку."""
        notification = self.create_notification()
        
        message = notification.format_message()
        assert isinstance(message, str)
        assert len(message) > 0
    
    def test_get_info_contains_required_fields(self):
        """get_info содержит все обязательные поля."""
        notification = self.create_notification()
        
        info = notification.get_info()
        
        assert "type" in info
        assert "recipient" in info
        assert "subject" in info
        assert "sent" in info
        assert "created_at" in info
    
    def test_send_twice_does_not_change_state(self):
        """Повторная отправка не должна портить состояние."""
        notification = self.create_notification()
        
        first_result = notification.send()
        sent_status = notification.is_sent()
        
        second_result = notification.send()
        
        assert second_result == first_result
        assert notification.is_sent() == sent_status


class TestEmailNotificationPolymorphic(BaseNotificationTest):
    """Полиморфные тесты для EmailNotification."""
    
    def create_notification(self) -> Notification:
        return EmailNotification("user@test.com", "Test", "Body content")


class TestSMSNotificationPolymorphic(BaseNotificationTest):
    """Полиморфные тесты для SMSNotification."""
    
    def create_notification(self) -> Notification:
        return SMSNotification("+1234567890", "Test message")


class TestPushNotificationPolymorphic(BaseNotificationTest):
    """Полиморфные тесты для PushNotification."""
    
    def create_notification(self) -> Notification:
        return PushNotification("device_token_1234567890", "Title", "Body")


class TestSlackNotificationPolymorphic(BaseNotificationTest):
    """Полиморфные тесты для SlackNotification."""
    
    def create_notification(self) -> Notification:
        return SlackNotification("https://hooks.slack.com/valid", "general", "Test message")
Задание 2.2. Параметризация по классам (15 минут)
python
class TestPolymorphicParametrized:
    """Параметризованные полиморфные тесты."""
    
    @pytest.mark.parametrize("notification_class,kwargs,expected_type", [
        (EmailNotification, {"recipient": "test@test.com", "subject": "Hi", "body": "Hello"}, "email"),
        (SMSNotification, {"phone_number": "+1234567890", "message": "Hello"}, "sms"),
        (PushNotification, {"device_token": "token1234567890", "title": "Alert", "body": "Body"}, "push"),
        (SlackNotification, {"webhook_url": "https://hooks.slack.com/valid", "channel": "general", "message": "Hi"}, "slack"),
    ])
    def test_get_type_parametrized(self, notification_class, kwargs, expected_type):
        """Параметризованный тест get_type для всех классов."""
        notification = notification_class(**kwargs)
        assert notification.get_type() == expected_type
    
    @pytest.mark.parametrize("notification_class,kwargs,should_succeed", [
        # Email
        (EmailNotification, {"recipient": "user@test.com", "subject": "Hi", "body": "Hello"}, True),
        (EmailNotification, {"recipient": "invalid", "subject": "Hi", "body": "Hello"}, False),
        (EmailNotification, {"recipient": "user@test.com", "subject": "Hi", "body": ""}, False),
        # SMS
        (SMSNotification, {"phone_number": "+1234567890", "message": "Hi"}, True),
        (SMSNotification, {"phone_number": "123", "message": "Hi"}, False),
        (SMSNotification, {"phone_number": "+1234567890", "message": ""}, False),
        # Push
        (PushNotification, {"device_token": "token1234567890", "title": "Title", "body": "Body"}, True),
        (PushNotification, {"device_token": "short", "title": "Title", "body": "Body"}, False),
        # Slack
        (SlackNotification, {"webhook_url": "https://hooks.slack.com/valid", "channel": "general", "message": "Hi"}, True),
        (SlackNotification, {"webhook_url": "invalid", "channel": "general", "message": "Hi"}, False),
    ])
    def test_send_parametrized(self, notification_class, kwargs, should_succeed):
        """Параметризованный тест отправки."""
        notification = notification_class(**kwargs)
        
        result = notification.send()
        
        assert result == should_succeed
        assert notification.is_sent() == should_succeed
    
    @pytest.mark.parametrize("notification_class,kwargs", [
        (EmailNotification, {"recipient": "test@test.com", "subject": "Hi", "body": "Hello"}),
        (SMSNotification, {"phone_number": "+1234567890", "message": "Hi"}),
        (PushNotification, {"device_token": "token1234567890", "title": "Title", "body": "Body"}),
        (SlackNotification, {"webhook_url": "https://hooks.slack.com/valid", "channel": "general", "message": "Hi"}),
    ])
    def test_info_contains_correct_data(self, notification_class, kwargs):
        """Проверка корректности данных в get_info."""
        notification = notification_class(**kwargs)
        
        info = notification.get_info()
        
        assert info["type"] == notification.get_type()
        assert info["sent"] == notification.is_sent()
        assert "created_at" in info
Задание 2.3. Проверка специфичных методов (10 минут)
python
class TestSpecificMethods:
    """Тесты специфичных методов каждого класса."""
    
    @pytest.mark.parametrize("class_name,method_name", [
        (EmailNotification, "get_sender"),
        (SMSNotification, "get_character_count"),
        (PushNotification, "get_title"),
        (SlackNotification, "get_channel"),
    ])
    def test_specific_methods_exist(self, class_name, method_name):
        """Проверка наличия специфичных методов."""
        assert hasattr(class_name, method_name)
    
    def test_email_get_sender(self):
        email = EmailNotification("test@test.com", "Subj", "Body", "custom@test.com")
        assert email.get_sender() == "custom@test.com"
    
    def test_sms_get_character_count(self):
        sms = SMSNotification("+1234567890", "Hello World")
        assert sms.get_character_count() == 11
    
    def test_push_get_title(self):
        push = PushNotification("token", "My Title", "Body")
        assert push.get_title() == "My Title"
    
    def test_slack_get_channel(self):
        slack = SlackNotification("https://hooks.slack.com/xxx", "random", "Hi")
        assert slack.get_channel() == "random"
СЛОЖНЫЙ УРОВЕНЬ (Варианты 18-25)
Цель для сложного уровня
Научиться создавать фабрики для тестовых объектов, проверять принцип LSP и писать комплексные полиморфные тесты.

Задания для сложного уровня
Задание 3.1. Фикстура-фабрика для уведомлений (10 минут)
python
# test_inheritance_advanced.py

import pytest
from inheritance_example import (
    Notification, EmailNotification, SMSNotification, 
    PushNotification, SlackNotification
)


@pytest.fixture
def notification_factory():
    """Фабрика для создания различных уведомлений."""
    
    def _create_notification(notification_type: str, **kwargs):
        if notification_type == "email":
            return EmailNotification(
                recipient=kwargs.get("recipient", "test@test.com"),
                subject=kwargs.get("subject", "Test"),
                body=kwargs.get("body", "Body"),
                sender=kwargs.get("sender", "noreply@example.com")
            )
        elif notification_type == "sms":
            return SMSNotification(
                phone_number=kwargs.get("phone_number", "+1234567890"),
                message=kwargs.get("message", "Test message")
            )
        elif notification_type == "push":
            return PushNotification(
                device_token=kwargs.get("device_token", "token1234567890"),
                title=kwargs.get("title", "Test Title"),
                body=kwargs.get("body", "Test Body")
            )
        elif notification_type == "slack":
            return SlackNotification(
                webhook_url=kwargs.get("webhook_url", "https://hooks.slack.com/valid"),
                channel=kwargs.get("channel", "general"),
                message=kwargs.get("message", "Test message")
            )
        else:
            raise ValueError(f"Unknown notification type: {notification_type}")
    
    return _create_notification


@pytest.fixture
def all_notifications_fixture(notification_factory):
    """Фикстура, возвращающая список всех типов уведомлений."""
    return [
        notification_factory("email"),
        notification_factory("sms"),
        notification_factory("push"),
        notification_factory("slack"),
    ]


class TestNotificationFactory:
    """Тесты с использованием фабрики."""
    
    def test_factory_creates_email(self, notification_factory):
        notification = notification_factory("email")
        assert isinstance(notification, EmailNotification)
        assert notification.get_type() == "email"
    
    def test_factory_creates_sms(self, notification_factory):
        notification = notification_factory("sms")
        assert isinstance(notification, SMSNotification)
        assert notification.get_type() == "sms"
    
    def test_factory_creates_push(self, notification_factory):
        notification = notification_factory("push")
        assert isinstance(notification, PushNotification)
        assert notification.get_type() == "push"
    
    def test_factory_creates_slack(self, notification_factory):
        notification = notification_factory("slack")
        assert isinstance(notification, SlackNotification)
        assert notification.get_type() == "slack"
    
    def test_factory_with_custom_params(self, notification_factory):
        email = notification_factory("email", recipient="custom@test.com", body="Custom body")
        assert email.recipient == "custom@test.com"
        assert email.body == "Custom body"
Задание 3.2. Проверка принципа LSP (15 минут)
python
class TestLiskovSubstitutionPrinciple:
    """Тесты для проверки принципа подстановки Лисков."""
    
    def test_all_notifications_can_be_used_polymorphically(self, all_notifications_fixture):
        """Все уведомления можно использовать через единый интерфейс."""
        for notification in all_notifications_fixture:
            # Все уведомления должны иметь эти методы
            assert hasattr(notification, "send")
            assert hasattr(notification, "get_type")
            assert hasattr(notification, "format_message")
            assert hasattr(notification, "get_info")
            assert hasattr(notification, "is_sent")
    
    def test_all_notifications_return_bool_on_send(self, notification_factory):
        """send() у всех уведомлений возвращает bool."""
        types = ["email", "sms", "push", "slack"]
        
        for ntype in types:
            notification = notification_factory(ntype)
            result = notification.send()
            assert isinstance(result, bool)
    
    def test_all_notifications_have_sent_flag(self, notification_factory):
        """is_sent() корректно отражает состояние после send."""
        types = ["email", "sms", "push", "slack"]
        
        for ntype in types:
            notification = notification_factory(ntype)
            assert notification.is_sent() is False
            
            notification.send()
            # В зависимости от валидности данных, может быть True или False
            # Но метод должен существовать и возвращать bool
            assert isinstance(notification.is_sent(), bool)
    
    def test_polymorphic_send_function(self, notification_factory):
        """Функция, работающая с любым Notification."""
        
        def process_notification(notification: Notification) -> dict:
            """Универсальная обработка уведомления."""
            success = notification.send()
            return {
                "type": notification.get_type(),
                "success": success,
                "message": notification.format_message()[:50]
            }
        
        types = ["email", "sms", "push", "slack"]
        
        for ntype in types:
            notification = notification_factory(ntype)
            result = process_notification(notification)
            
            assert "type" in result
            assert "success" in result
            assert "message" in result
            assert result["type"] == ntype
    
    def test_subclass_can_replace_base(self, notification_factory):
        """Производный класс может заменять базовый."""
        
        def handle_notification(notification: Notification):
            notification.send()
            return notification.get_info()
        
        # Базовый класс нельзя создать (абстрактный)
        # Но все наследники должны работать через единый интерфейс
        for ntype in ["email", "sms", "push", "slack"]:
            notification = notification_factory(ntype)
            info = handle_notification(notification)
            
            assert info["type"] == ntype
            assert "created_at" in info
Задание 3.3. Комплексный полиморфный тест (15 минут)
python
class TestPolymorphicBehavior:
    """Комплексные тесты полиморфного поведения."""
    
    @pytest.fixture
    def notification_collection(self, notification_factory):
        """Коллекция различных уведомлений."""
        return [
            notification_factory("email", recipient="alice@test.com", body="Hello Alice"),
            notification_factory("sms", phone_number="+1234567890", message="Hello Bob"),
            notification_factory("push", device_token="token123", title="Alert", body="New message"),
            notification_factory("slack", channel="alerts", message="System alert"),
        ]
    
    def test_bulk_send_all_notifications(self, notification_collection):
        """Массовая отправка всех уведомлений."""
        results = []
        
        for notification in notification_collection:
            results.append({
                "type": notification.get_type(),
                "sent": notification.send()
            })
        
        assert len(results) == 4
        for result in results:
            assert "type" in result
            assert "sent" in result
            assert isinstance(result["sent"], bool)
    
    def test_filter_by_type(self, notification_collection):
        """Фильтрация уведомлений по типу."""
        emails = [n for n in notification_collection if n.get_type() == "email"]
        sms = [n for n in notification_collection if n.get_type() == "sms"]
        
        assert len(emails) == 1
        assert len(sms) == 1
        assert emails[0].recipient == "alice@test.com"
    
    def test_all_notifications_format_message_unique(self, notification_collection):
        """format_message возвращает уникальное содержимое для разных типов."""
        messages = [n.format_message() for n in notification_collection]
        
        # Все сообщения должны быть разными (разные типы имеют разный формат)
        assert len(messages) == len(set(messages))
    
    def test_polymorphic_info_aggregation(self, notification_collection):
        """Агрегация информации от всех уведомлений."""
        summary = {
            "total": len(notification_collection),
            "types": {},
            "sent_count": 0
        }
        
        for notification in notification_collection:
            ntype = notification.get_type()
            summary["types"][ntype] = summary["types"].get(ntype, 0) + 1
            
            if notification.send():
                summary["sent_count"] += 1
        
        assert summary["total"] == 4
        assert summary["types"]["email"] == 1
        assert summary["types"]["sms"] == 1
        assert summary["types"]["push"] == 1
        assert summary["types"]["slack"] == 1
Задание 3.4. Итоговый отчёт (5 минут)
markdown
## Отчёт о тестировании (сложный уровень)

**Студент:** _________________
**Вариант №:** ___ (18-25)

### Результаты полиморфного тестирования

| Тип уведомления | Базовые тесты | Специфичные тесты | LSP проверка |
|:---|:---|:---|:---|
| EmailNotification | ☐ | ☐ | ☐ |
| SMSNotification | ☐ | ☐ | ☐ |
| PushNotification | ☐ | ☐ | ☐ |
| SlackNotification | ☐ | ☐ | ☐ |

### Проверка принципа LSP

| Проверка | Статус |
|:---|:---|
| Все наследники имеют общие методы | ☐ |
| send() возвращает bool у всех | ☐ |
| get_type() возвращает строку | ☐ |
| get_info() содержит обязательные поля | ☐ |
| Функции с базовым типом работают с наследниками | ☐ |

### Статистика тестов

| Компонент | Тестов | Пройдено | Не пройдено |
|:---|:---|:---|:---|
| Базовые тесты | _____ | _____ | _____ |
| Параметризация | _____ | _____ | _____ |
| LSP проверка | _____ | _____ | _____ |
| Комплексные тесты | _____ | _____ | _____ |
| **Итого** | _____ | _____ | _____ |

### Выводы

_______________________________________________________________

_______________________________________________________________
Карточка студента
text
ПР 3.13. ТЕСТИРОВАНИЕ ИЕРАРХИЙ КЛАССОВ И ПОЛИМОРФИЗМА

Вариант № ___
Уровень: □ Базовый (1-8) □ Средний (9-17) □ Сложный (18-25)

=== ВЫПОЛНЕННЫЕ ЗАДАНИЯ ===

□ EmailNotification тесты (базовый)
□ SMSNotification тесты (базовый)
□ Push/Slack тесты (базовый)
□ Базовый тестовый класс (средний)
□ Параметризация по классам (средний)
□ Фабрика уведомлений (сложный)
□ Проверка LSP (сложный)
□ Комплексные полиморфные тесты (сложный)

=== ПРОВЕРЕННЫЕ ПРИНЦИПЫ ===

□ Единый интерфейс для всех классов
□ Принцип подстановки Лисков (LSP)
□ Полиморфное поведение
□ Инварианты иерархии

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
- test_inheritance_base.py
- test_inheritance_intermediate.py
- test_inheritance_advanced.py

Дата выполнения: _____________
Подпись студента: _____________
Критерии оценки
Баллы	Уровень	Критерий
2 (неудовлетворительно)	Любой	Тесты не работают или классы не тестируются
3 (удовлетворительно)	Базовый	Каждый класс отдельно (15+ тестов)
4 (хорошо)	Средний	+ Базовый тестовый класс + параметризация (25+ тестов)
5 (отлично)	Сложный	+ LSP проверка + фабрика + комплекс (35+ тестов)
Контрольные вопросы (для защиты)
Что такое полиморфное тестирование и зачем оно нужно?

В чём суть принципа подстановки Лисков (LSP)?

Как проверить, что все наследники соблюдают LSP?

Как параметризация по классам помогает в тестировании иерархий?

Что должен содержать базовый тестовый класс для иерархии?

Как проверить, что специфичные методы не нарушают общий контракт?

Как тестировать фабрику, создающую объекты разных типов?

Почему важно тестировать не только каждый класс отдельно, но и их совместную работу?

Какие проблемы может выявить полиморфное тестирование?

Как фикстура-фабрика упрощает тестирование иерархий?

Следующее занятие: Контрольная работа 3.2 по темам 3.6–3.8.
