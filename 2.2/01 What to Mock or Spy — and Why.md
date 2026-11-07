# Что подменять в тестах и зачем (простыми словами)

Юнит-тест проверяет **один класс**. Но классы почти никогда не работают в одиночку: они ходят в базу, вызывают внешние сервисы, отправляют письма. Если в тесте всё это настоящее, тест становится медленным, нестабильным и может натворить дел (например, отправить реальное письмо).

Поэтому часть зависимостей в тесте **подменяют** специальными объектами — **тестовыми дублёрами** (test doubles). Важно не подменять всё подряд, а понимать, **что подменить, а что оставить настоящим**.

## 1. Кого тестируем и кто ему помогает

Пример — сервис обработки заказа:

```java
import java.math.BigDecimal;
import java.util.Objects;
import java.util.Optional;

enum OrderStatus {
    CREATED,
    PAID,
    PAYMENT_FAILED
}

final class Order {
    private final long id;
    private final BigDecimal total;
    private OrderStatus status = OrderStatus.CREATED;

    Order(long id, BigDecimal total) {
        this.id = id;
        this.total = Objects.requireNonNull(total);
    }

    long id() {
        return id;
    }

    BigDecimal total() {
        return total;
    }

    OrderStatus status() {
        return status;
    }

    void markPaid() {
        status = OrderStatus.PAID;
    }

    void markPaymentFailed() {
        status = OrderStatus.PAYMENT_FAILED;
    }
}

record PaymentResult(boolean successful, String message) {
    static PaymentResult approved() {
        return new PaymentResult(true, "Payment approved");
    }

    static PaymentResult declined(String message) {
        return new PaymentResult(false, message);
    }
}

record ProcessingResult(OrderStatus status, String message) {
}

interface OrderRepository {
    Optional<Order> findById(long orderId);

    void save(Order order);
}

interface PaymentGateway {
    PaymentResult charge(long orderId, BigDecimal amount);
}

interface NotificationService {
    void sendPaymentConfirmation(long orderId);
}

final class OrderProcessingService {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final NotificationService notificationService;

    OrderProcessingService(
            OrderRepository orderRepository,
            PaymentGateway paymentGateway,
            NotificationService notificationService) {
        this.orderRepository = Objects.requireNonNull(orderRepository);
        this.paymentGateway = Objects.requireNonNull(paymentGateway);
        this.notificationService = Objects.requireNonNull(notificationService);
    }

    ProcessingResult process(long orderId) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new IllegalArgumentException(
                        "Order not found: " + orderId));

        PaymentResult payment = paymentGateway.charge(
                order.id(), order.total());

        if (!payment.successful()) {
            order.markPaymentFailed();
            orderRepository.save(order);
            return new ProcessingResult(order.status(), payment.message());
        }

        order.markPaid();
        orderRepository.save(order);
        notificationService.sendPaymentConfirmation(order.id());

        return new ProcessingResult(order.status(), payment.message());
    }
}
```

Здесь:

- **`OrderProcessingService`** — тот, кого мы проверяем. Его называют **UUT** (unit under test) или **SUT** (system under test).
- **`OrderRepository`, `PaymentGateway`, `NotificationService`** — его **помощники (соавторы)**. Сервис через них берёт данные и отдаёт команды.
- **`Order`, `PaymentResult`, `ProcessingResult`** — простые объекты с данными. Их подменять не нужно.

**Правило №1:** тестируемый класс всегда настоящий. Если подменить его самого, вы проверите только свою подмену, а не реальный код.

## 2. Чем мешают настоящие помощники

Представьте, что в тест попали настоящие реализации: база данных, платёжный сервис в интернете и отправщик писем. Тогда тест:

- не запустится без настроенной базы;
- упадёт, если пропал интернет;
- будет медленным;
- отправит настоящее письмо;
- не сможет легко изобразить, например, «платёж отклонён».

Такой тест полезен как **интеграционный**, но для юнит-теста не годится. В юнит-тесте мы хотим сами задавать условия: «заказ есть», «платёж прошёл», «платёж отклонён», «письмо не должно уйти». Дублёры дают такой контроль.

## 3. Подменять или нет?

Не нужно «мокать всё». Задайте простой вопрос: **мешает ли этот настоящий объект моему тесту?**

**Оставьте настоящим**, если объект быстрый, всегда даёт одинаковый результат, не ходит в сеть/базу/файлы, ничего не ломает и легко создаётся. Обычно это данные: record, коллекции, простые доменные классы.

**Подмените**, если объект ходит в базу, сеть или файлы; медленный; зависит от времени или случайности; делает опасные действия; мешает воспроизвести нужную ошибку; требует сложной настройки.

Это подсказки, а не законы. Иногда простая реализация «в памяти» удобнее мока.

**Роль зависит от теста.** Один и тот же `PaymentGateway` в одном тесте всегда одобряет платёж, в другом — отклоняет, в третьем — ещё и запоминает сумму. Класс не «стаб» и не «мок» навсегда — это его роль в конкретном тесте.

## 4. Что приходит и что уходит

Удобно смотреть, **в какую сторону** идёт информация.

**Приходит в сервис (косвенный вход)** — то, что помощник возвращает:

```java
PaymentResult payment = paymentGateway.charge(
        order.id(), order.total());
```

Чтобы управлять этим ответом, используют **стаб**.

**Уходит из сервиса (косвенный выход)** — команда, которую сервис отдаёт помощнику:

```java
notificationService.sendPaymentConfirmation(order.id());
```

Чтобы проверить, что команда была, используют **шпиона** или **мок**.

Коротко: **стаб — подсунуть ответ, шпион/мок — проверить вызов.**

## 5. Пять видов дублёров

| Дублёр | Что делает простыми словами |
|---|---|
| Dummy | Затычка: нужен, только чтобы код скомпилировался/запустился |
| Stub | Всегда отвечает заранее заданным ответом |
| Spy | Тихо записывает, как его вызывали, чтобы потом проверить |
| Mock | Заранее знает, какие вызовы должны быть, и сам ругается, если не так |
| Fake | Упрощённая, но рабочая версия (например, база в памяти) |

Один объект может совмещать роли — например, и отвечать заготовкой, и записывать вызовы.

### 5.1. Dummy — затычка

Нужен только потому, что его требует конструктор. В тесте до него дело не доходит.

```java
final class DummyNotificationService implements NotificationService {
    @Override
    public void sendPaymentConfirmation(long orderId) {
        // This method is not reached in the current test scenario.
    }
}
```

Пример: тест «заказ не найден». Сервис бросит исключение раньше, чем дойдёт до уведомлений, но конструктору всё равно нужен `NotificationService`.

### 5.2. Stub — заготовленный ответ

Возвращает то, что нам нужно для сценария.

```java
final class DeclinedPaymentGatewayStub implements PaymentGateway {
    @Override
    public PaymentResult charge(long orderId, BigDecimal amount) {
        return PaymentResult.declined("Insufficient funds");
    }
}
```

```java
final class ApprovedPaymentGatewayStub implements PaymentGateway {
    @Override
    public PaymentResult charge(long orderId, BigDecimal amount) {
        return PaymentResult.approved();
    }
}
```

Используйте, когда от ответа помощника зависит, по какой ветке пойдёт код: объект найден/не найден, платёж прошёл/нет, пришла ошибка или таймаут, нужно фиксированное время.

Не настраивайте ответы, которые на тест не влияют — это только запутывает.

### 5.3. Spy — записывающий шпион

Ничего не делает, просто запоминает вызовы.

```java
import java.util.ArrayList;
import java.util.List;

final class NotificationServiceSpy implements NotificationService {
    private final List<Long> confirmedOrderIds = new ArrayList<>();

    @Override
    public void sendPaymentConfirmation(long orderId) {
        confirmedOrderIds.add(orderId);
    }

    List<Long> confirmedOrderIds() {
        return List.copyOf(confirmedOrderIds);
    }
}
```

После теста смотрим в `confirmedOrderIds()` — ушло ли уведомление и для какого заказа.

Используйте, когда сам вызов важен, а по результату метода его не видно. Но не записывайте всё подряд: тест начнёт ломаться при любой перестройке кода, даже если поведение не изменилось.

> **Внимание:** `Mockito.spy()` — это другое. Это обёртка над настоящим объектом, которая вызывает его реальные методы. Об этом — в следующей теме.

### 5.4. Mock — проверяющий

Заранее знает, что должно произойти, и сам сообщает об ошибке.

```java
final class NotificationServiceMock implements NotificationService {
    private final long expectedOrderId;
    private boolean called;

    NotificationServiceMock(long expectedOrderId) {
        this.expectedOrderId = expectedOrderId;
    }

    @Override
    public void sendPaymentConfirmation(long orderId) {
        if (called) {
            throw new AssertionError("Confirmation was sent more than once");
        }
        if (orderId != expectedOrderId) {
            throw new AssertionError(
                    "Expected order " + expectedOrderId
                            + " but received " + orderId);
        }
        called = true;
    }

    void verify() {
        if (!called) {
            throw new AssertionError("Confirmation was not sent");
        }
    }
}
```

Разница со шпионом: шпион просто записывает, а проверку делает тест; мок проверяет сам. Вручную такие классы почти не пишут — их создаёт Mockito.

### 5.5. Fake — упрощённая рабочая версия

Настоящая логика, но без тяжёлых ресурсов.

```java
import java.util.HashMap;
import java.util.Map;

final class InMemoryOrderRepository implements OrderRepository {
    private final Map<Long, Order> orders = new HashMap<>();

    void add(Order order) {
        orders.put(order.id(), order);
    }

    @Override
    public Optional<Order> findById(long orderId) {
        return Optional.ofNullable(orders.get(orderId));
    }

    @Override
    public void save(Order order) {
        orders.put(order.id(), order);
    }
}
```

Отличие от стаба: стаб всегда отвечает одно и то же, а фейк реально хранит данные и отвечает по-разному.

Примеры фейков: репозиторий в памяти, часы, которые можно перевести вперёд, сборщик сообщений.

Минус: фейк не умеет всего, что умеет настоящая база (транзакции, ограничения, SQL). Поэтому интеграционные тесты всё равно нужны.

## 6. Проверять результат или вызовы?

**Проверка результата (состояния)** — смотрим, что получилось:

```java
ProcessingResult result = service.process(42L);

assertEquals(OrderStatus.PAYMENT_FAILED, result.status());
assertEquals(OrderStatus.PAYMENT_FAILED, order.status());
```

**Проверка поведения (вызовов)** — смотрим, кого и как вызвали:

```java
service.process(42L);

assertEquals(List.of(42L), notificationSpy.confirmedOrderIds());
```

**Как выбирать:**
- По умолчанию проверяйте **результат** — такие тесты не ломаются при перестройке кода.
- Проверяйте **вызовы**, только если вызов сам по себе важен: письмо должно уйти, письмо **не** должно уйти, транзакция должна закоммититься, опасный метод не должен вызваться.
- Не проверяйте вызовы «потому что можно» — например, что `order.id()` вызвали ровно один раз. Это никому не важно.

## 7. Всё вместе в одном тесте

```java
import static org.junit.jupiter.api.Assertions.assertEquals;

import java.math.BigDecimal;
import java.util.List;

import org.junit.jupiter.api.Test;

class OrderProcessingServiceTest {

    @Test
    void shouldMarkOrderAsPaidAndRequestConfirmationWhenPaymentIsApproved() {
        // Arrange
        Order order = new Order(42L, new BigDecimal("59.90"));

        InMemoryOrderRepository repository =
                new InMemoryOrderRepository();
        repository.add(order);

        PaymentGateway paymentGateway =
                new ApprovedPaymentGatewayStub();
        NotificationServiceSpy notificationSpy =
                new NotificationServiceSpy();

        OrderProcessingService service = new OrderProcessingService(
                repository, paymentGateway, notificationSpy);

        // Act
        ProcessingResult result = service.process(42L);

        // Assert
        assertEquals(OrderStatus.PAID, result.status());
        assertEquals(OrderStatus.PAID, order.status());
        assertEquals(List.of(42L),
                notificationSpy.confirmedOrderIds());
    }
}
```

Здесь: `Order` — настоящий, репозиторий — фейк, платёжный шлюз — стаб, уведомления — шпион. В другом тесте набор может быть другим.

## 8. Как выбрать — по шагам

1. Что именно должен доказать тест?
2. Какой класс за это отвечает? Он — настоящий.
3. Каких помощников он использует в этом сценарии?
4. Можно ли взять их настоящими (быстро, предсказуемо, безопасно)?
5. Подменяйте только тех, кто мешает, — самым простым подходящим дублёром.
6. Проверяйте результат и только важные вызовы.

| Что нужно | Что берём |
|---|---|
| Простые данные | Настоящий объект |
| Параметр, который не используется | Dummy |
| Нужный ответ или исключение | Stub |
| Запомнить вызовы и проверить потом | Spy |
| Заранее заданные ожидания вызовов | Mock |
| Простая рабочая реализация | Fake |

Если подходит несколько вариантов — берите тот, с которым тест понятнее.

## 9. Частые ошибки

- **Подменили сам тестируемый класс** — тест ничего не проверяет.
- **Подменили простые данные** (`Order`, `BigDecimal`, record) — лишняя работа без пользы.
- **Подменили всех помощников** — тест сложный и непонятный.
- **Проверяют каждый вызов** — тест ломается от любой перестройки кода.
- **Называют всё из Mockito «моком».** Объект из `Mockito.mock()` может быть затычкой, стабом, моком или всем сразу — смотря как его используют.
- **Думают, что дублёры заменяют интеграционные тесты.** Юнит-тест не проверит, что настоящий платёжный шлюз правильно общается с банком.

## 10. Практика: выберите тестового дублёра

Для каждого сценария решите, что оставить настоящим и какой дублёр подходит.

1. Вы тестируете `TaxCalculator`. Он получает неизменяемый record `Money` и небольшой предсказуемый объект `TaxRules`.
2. Нужно, чтобы платёжная зависимость вернула «отклонено», не обращаясь к реальному платёжному провайдеру.
3. Нужно проверить, какой ID заказа передали сервису уведомлений.
4. Многим тестам репозитория нужно добавлять, обновлять и искать объекты без запуска базы данных.
5. Конструктор требует `AuditClient`, но в текущем тесте валидация завершается раньше, чем клиент понадобится.
6. Вы хотите проверить каждый геттер и каждый вызов хелпера в сервисе, хотя результат метода уже доказывает нужное поведение.

### Ответы

1. `Money` и `TaxRules` оставить настоящими: они маленькие, предсказуемые и легко создаются.
2. Стаб, который возвращает отказ.
3. Шпион или мок из фреймворка с проверкой вызова.
4. Fake — репозиторий в памяти.
5. Достаточно dummy.
6. Не добавлять такие проверки. Проверять результат или состояние и только те вызовы, которые входят в контракт.

## 11. Проверь себя

### Вопросы

1. Какой объект является UUT в тесте `OrderProcessingService.process()`?
2. Почему настоящий `Order` здесь обычно лучше дублёра?
3. Какой дублёр управляет косвенным входом, возвращая заготовленный ответ?
4. Чем стаб отличается от фейка?
5. Когда оправдана проверка поведения (вызовов)?
6. Является ли моком каждый объект, созданный через `Mockito.mock()`?
7. Почему избыток шпионов и проверок вызовов делает тесты хрупкими?

### Ответы

1. Настоящий экземпляр `OrderProcessingService`.
2. Это маленький объект с данными: быстрый, предсказуемый и легко создаётся в нужном состоянии.
3. Стаб.
4. Стаб просто возвращает заготовленные ответы; фейк — упрощённая рабочая реализация, которая отвечает в зависимости от своих данных.
5. Когда вызов помощника — важный эффект или часть контракта, и его нельзя показать через результат или состояние.
6. Нет. В зависимости от использования он может быть dummy, стабом, моком или их комбинацией.
7. Тесты привязываются к тому, как код устроен внутри, и падают после рефакторинга, даже если поведение не изменилось.

## Шпаргалка

- Тестируемый класс — всегда настоящий.
- Простые данные — настоящие.
- Подменяем только то, что мешает: база, сеть, письма, время, случайность.
- **Dummy** — затычка, **Stub** — готовый ответ, **Spy** — записывает вызовы, **Mock** — сам проверяет вызовы, **Fake** — упрощённая рабочая версия.
- Стаб — чтобы подсунуть ответ; шпион/мок — чтобы проверить вызов.
- Сначала проверяем результат; вызовы — только если они действительно важны.
- Роль дублёра определяет тест, а не класс.

## Почитать

- Robert C. Martin, [The Little Mocker](https://blog.cleancoder.com/uncle-bob/2014/05/14/TheLittleMocker.html)
- Robert C. Martin, [When to Mock](https://blog.cleancoder.com/uncle-bob/2014/05/10/WhenToMock.html)
- Martin Fowler, [Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html)
