Оба подхода возможны, но выбор зависит от **гибкости**, **тестируемости** и **расширяемости** кода.  

## **1. Подход без интерфейса (простой, но менее гибкий)**
Если **не предполагается** несколько реализаций Listener'а, можно сразу писать класс:

```java
@Component
public class UserRegisteredListener {
    @EventListener
    public void onUserRegistered(UserRegisteredEvent event) {
        System.out.println("Отправка email на " + event.getEmail());
    }
}
```
📌 **Когда использовать?**  
✅ Если Listener — это простой обработчик и в будущем не требуется менять его поведение.  
✅ Когда нет необходимости в мокировании (mock) при тестировании.  

---

## **2. Подход через интерфейс (более гибкий)**
Если **нужна возможность смены реализации** или **удобное тестирование**, лучше сначала создать интерфейс:

### **1. Создаем интерфейс**
```java
public interface UserRegisteredListener {
    void handleUserRegistered(UserRegisteredEvent event);
}
```

### **2. Реализуем интерфейс в классе**
```java
@Component
public class EmailNotificationListener implements UserRegisteredListener {
    @Override
    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        System.out.println("Отправка email на " + event.getEmail());
    }
}
```

### **3. Можно заменить реализацию**
Например, добавить другой обработчик (логирование вместо email-уведомления):
```java
@Component
public class LoggingUserRegisteredListener implements UserRegisteredListener {
    @Override
    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        System.out.println("Пользователь зарегистрирован: " + event.getEmail());
    }
}
```

---

## **3. Когда использовать интерфейс?**
📌 **Используйте интерфейс, если:**  
✅ Возможны **разные реализации** (email, SMS, push-уведомления).  
✅ Нужно **упрощенное тестирование** – можно мокировать `UserRegisteredListener` в тестах.  
✅ Код должен быть **расширяемым**, чтобы легко добавлять новые обработчики.  

📌 **Можно без интерфейса, если:**  
✅ Listener **уникален** и не требует смены реализации.  
✅ Нет необходимости в **юнит-тестах с mock-объектами**.  
✅ Приложение простое и **не требует сложной архитектуры**.  

---

## **Вывод**
- Для **простых случаев** можно сразу писать класс с `@EventListener`.  
- Если **планируется масштабирование** или **нужны тесты**, лучше использовать интерфейс.  

❓ **Какой сценарий у вас – простой или гибкий?**
