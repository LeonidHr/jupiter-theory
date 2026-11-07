# Основы Mockito: моки, шпионы и аннотации (простыми словами)

В прошлой теме мы решали, **что** подменять в тесте. Теперь учимся делать это с помощью **Mockito** — библиотеки, которая сама создаёт дублёров, подсовывает им нужные ответы и запоминает, как их вызывали. Не нужно писать классы-стабы и классы-шпионы вручную.

Но Mockito не решает за вас, **что именно стоит проверять**. Это по-прежнему ваша работа.

## 1. Пример, который тестируем

Тот же сервис заказов. Он находит заказ, списывает деньги, сохраняет статус и после успешной оплаты отправляет подтверждение.

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

record PaymentConfirmation(long orderId, BigDecimal amount) {
    PaymentConfirmation {
        Objects.requireNonNull(amount);
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
    void send(PaymentConfirmation confirmation);
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
        notificationService.send(new PaymentConfirmation(
                order.id(), order.total()));

        return new ProcessingResult(order.status(), payment.message());
    }
}
```

- Сервис — **настоящий**, его и тестируем.
- Репозиторий, платёжный шлюз и уведомления — **помощники**, их подменяем.
- `Order` и record-ы — **настоящие** объекты с данными.

## 2. Создаём мок: `mock()`

`mock()` создаёт «пустышку» нужного типа, которая запоминает все вызовы.

```java
import static org.mockito.Mockito.mock;

OrderRepository repository = mock(OrderRepository.class);
PaymentGateway paymentGateway = mock(PaymentGateway.class);
NotificationService notificationService =
        mock(NotificationService.class);
```

Настоящий код интерфейсов не выполняется. Мок можно настроить перед тестом и проверить после.

**Что мок возвращает, если его не настроили:**
- `boolean` → `false`;
- числа → `0`;
- большинство объектов → `null`;
- `Optional` и коллекции → пустые.

**Не полагайтесь на эти значения случайно.** Если ответ помощника важен для сценария — настройте его явно. Например, ненастроенный репозиторий вернёт пустой `Optional`, и сервис решит, что заказа нет. Тест пройдёт, но по нему непонятно, было ли это задумано.

> **Главное:** мок даёт возможность управлять и наблюдать, но сам по себе ничего осмысленного не возвращает, пока вы его не настроите.

## 3. Настраиваем ответы (стабинг)

```java
import static org.mockito.Mockito.when;

Order order = new Order(42L, new BigDecimal("59.90"));

when(repository.findById(42L))
        .thenReturn(Optional.of(order));

when(paymentGateway.charge(42L, new BigDecimal("59.90")))
        .thenReturn(PaymentResult.approved());
```

Читается так: **«когда вызовут `findById(42L)` — верни этот заказ»**.

Общий вид:

```java
when(mock.method(arguments)).thenReturn(preparedValue);
```

Настройку пишем в блоке **Arrange** (подготовка) — до вызова сервиса.

**Аргументы важны.** Такая настройка сработает только для `42L`:

```java
when(repository.findById(42L))
        .thenReturn(Optional.of(order));
```

Если сервис вызовет `findById(7L)`, мок вернёт значение по умолчанию. Аргументы сравниваются через обычный `equals`. Если конкретное значение не важно — используйте матчеры (раздел 6).

**Разные сценарии — разные тесты:**

```java
when(paymentGateway.charge(42L, new BigDecimal("59.90")))
        .thenReturn(PaymentResult.approved());
```

```java
when(paymentGateway.charge(42L, new BigDecimal("59.90")))
        .thenReturn(PaymentResult.declined("Insufficient funds"));
```

Один тест — одно поведение. Так проще читать и искать ошибку, чем перенастраивать мок несколько раз в одном тесте.

## 4. Первый тест с Mockito

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import java.math.BigDecimal;
import java.util.Optional;

import org.junit.jupiter.api.Test;

class OrderProcessingServiceTest {

    @Test
    void shouldMarkOrderAsPaidAndSendConfirmationWhenPaymentIsApproved() {
        // Arrange
        OrderRepository repository = mock(OrderRepository.class);
        PaymentGateway paymentGateway = mock(PaymentGateway.class);
        NotificationService notificationService =
                mock(NotificationService.class);

        OrderProcessingService service = new OrderProcessingService(
                repository, paymentGateway, notificationService);

        Order order = new Order(42L, new BigDecimal("59.90"));

        when(repository.findById(42L))
                .thenReturn(Optional.of(order));
        when(paymentGateway.charge(42L, new BigDecimal("59.90")))
                .thenReturn(PaymentResult.approved());

        // Act
        ProcessingResult result = service.process(42L);

        // Assert
        assertEquals(OrderStatus.PAID, result.status());
        assertEquals(OrderStatus.PAID, order.status());

        verify(repository).save(order);
        verify(notificationService).send(
                new PaymentConfirmation(42L, new BigDecimal("59.90")));
    }
}
```

`PaymentConfirmation` — record, поэтому `equals` сравнивает **значения полей**. Проверка смотрит на данные, а не на то, тот же ли это объект.

В тесте два вида проверок:
- `assertEquals` — проверяем **результат и состояние**;
- `verify` — проверяем **важные вызовы** (сохранение и уведомление).

Mockito не заменяет JUnit: результат проверяем ассертами, вызовы — через `verify`.

## 5. Проверяем вызовы: `verify()`

```java
verify(repository).save(order);
```

По умолчанию `verify` ждёт **ровно один** вызов. Эти строки делают одно и то же:

```java
verify(repository).save(order);
verify(repository, times(1)).save(order);
```

Пишите короткую форму, если ждёте один вызов.

**`times(n)`** — когда важно точное количество, и оно не равно одному:

```java
verify(auditService, times(2)).recordAttempt(42L);
```

Не проверяйте количество «просто так» — только если оно является требованием.

**`never()`** — когда важно, что действие **не** произошло:

```java
verify(notificationService, never()).send(any());
```

Пример: при отказе в оплате подтверждение отправляться не должно.

```java
@Test
void shouldNotSendConfirmationWhenPaymentIsDeclined() {
    // Arrange
    OrderRepository repository = mock(OrderRepository.class);
    PaymentGateway paymentGateway = mock(PaymentGateway.class);
    NotificationService notificationService =
            mock(NotificationService.class);

    OrderProcessingService service = new OrderProcessingService(
            repository, paymentGateway, notificationService);

    Order order = new Order(42L, new BigDecimal("59.90"));

    when(repository.findById(42L))
            .thenReturn(Optional.of(order));
    when(paymentGateway.charge(42L, new BigDecimal("59.90")))
            .thenReturn(PaymentResult.declined("Insufficient funds"));

    // Act
    ProcessingResult result = service.process(42L);

    // Assert
    assertEquals(OrderStatus.PAYMENT_FAILED, result.status());
    verify(repository).save(order);
    verify(notificationService, never()).send(any());
}
```

**Другие режимы:** `atLeast(...)`, `atMost(...)`, `atLeastOnce()`, `only()`. Бывают полезны, но обычно хватает «один раз», `times(n)` и `never()`.

**`verifyNoInteractions(mock)`** — у мока не было ни одного вызова. **`verifyNoMoreInteractions(mock)`** — не осталось непроверенных вызовов. Используйте редко: тест «больше ничего не происходило» ломается от любой безобидной правки. Лучше точечный `never()` на одно запрещённое действие.

## 6. Матчеры аргументов

Когда конкретное значение не важно, можно сказать «любой»:

```java
import static org.mockito.ArgumentMatchers.anyLong;

when(repository.findById(anyLong()))
        .thenReturn(Optional.of(order));
```

Теперь сработает для любого ID.

| Матчер | Значит |
|---|---|
| `any()` | Что угодно, включая `null` |
| `any(Class.class)` | Любой не-null объект этого типа |
| `anyLong()`, `anyInt()` | Любое число нужного типа |
| `anyString()` | Любая не-null строка |
| `eq(value)` | Значение, равное указанному |
| `isNull()` | `null` |

Матчеры берутся из `ArgumentMatchers` через static import.

### Правило: не смешивайте матчеры и обычные значения

Если хотя бы один аргумент — матчер, **все** аргументы в этом вызове должны быть матчерами.

Так **нельзя**:

```java
// Incorrect: matcher and raw value are mixed.
when(paymentGateway.charge(anyLong(), new BigDecimal("59.90")))
        .thenReturn(PaymentResult.approved());
```

Конкретное значение оборачиваем в `eq()`:

```java
import static org.mockito.ArgumentMatchers.anyLong;
import static org.mockito.ArgumentMatchers.eq;

when(paymentGateway.charge(
        anyLong(), eq(new BigDecimal("59.90"))))
        .thenReturn(PaymentResult.approved());
```

С `verify` — то же правило:

```java
verify(paymentGateway).charge(
        eq(42L), eq(new BigDecimal("59.90")));
```

Но если оба значения известны, матчеры не нужны — проще так:

```java
verify(paymentGateway).charge(
        42L, new BigDecimal("59.90"));
```

### Не делайте матчеры слишком широкими

```java
when(paymentGateway.charge(anyLong(), any(BigDecimal.class)))
        .thenReturn(PaymentResult.approved());
```

Это нормально, если тесту важна только реакция на одобрение. Но если тест должен проверить, что передаётся **правильная сумма**, — такой матчер спрячет ошибку.

**Матчеры убирают лишние детали, а не прячут ошибки.**

## 7. Mockito + JUnit 5: аннотации

Писать `mock(...)` в каждом тесте утомительно. Есть аннотации.

Зависимости Maven:

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

Подключаем расширение к тест-классу:

```java
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class OrderProcessingServiceTest {
}
```

Оно перед каждым тестом создаёт объекты для полей с аннотациями Mockito.

### `@Mock` — создать мок для поля

```java
import org.mockito.Mock;

@Mock
private OrderRepository repository;

@Mock
private PaymentGateway paymentGateway;

@Mock
private NotificationService notificationService;
```

Это просто короткая замена `mock(...)`. Мок работает так же.

### Вариант 1: создать сервис вручную

```java
import org.junit.jupiter.api.BeforeEach;

private OrderProcessingService service;

@BeforeEach
void setUp() {
    service = new OrderProcessingService(
            repository, paymentGateway, notificationService);
}
```

Всё видно явно — самый понятный способ, особенно пока учитесь.

### Вариант 2: `@InjectMocks` — Mockito создаст сервис сам

```java
import org.mockito.InjectMocks;

@InjectMocks
private OrderProcessingService service;
```

Целиком:

```java
@ExtendWith(MockitoExtension.class)
class OrderProcessingServiceTest {

    @Mock
    private OrderRepository repository;

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private NotificationService notificationService;

    @InjectMocks
    private OrderProcessingService service;
}
```

Mockito создаст **настоящий** сервис и подставит в него моки (через конструктор, сеттеры или поля).

`@InjectMocks` — это просто удобство, а не DI-фреймворк. Он может скрыть, какой конструктор выбран или какая зависимость не подставилась. Если что-то непонятно — создайте сервис руками.

> **Важно:** `@Mock`/`@Spy` — на **помощников**. `@InjectMocks` — на **тестируемый класс**. Никогда не ставьте `@Mock` на класс, который тестируете.

## 8. Полный тест-класс на аннотациях

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import java.math.BigDecimal;
import java.util.Optional;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class OrderProcessingServiceTest {

    @Mock
    private OrderRepository repository;

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private NotificationService notificationService;

    @InjectMocks
    private OrderProcessingService service;

    @Test
    void shouldMarkOrderAsPaidAndSendConfirmationWhenPaymentIsApproved() {
        // Arrange
        Order order = new Order(42L, new BigDecimal("59.90"));

        when(repository.findById(42L))
                .thenReturn(Optional.of(order));
        when(paymentGateway.charge(42L, new BigDecimal("59.90")))
                .thenReturn(PaymentResult.approved());

        // Act
        ProcessingResult result = service.process(42L);

        // Assert
        assertEquals(OrderStatus.PAID, result.status());
        assertEquals(OrderStatus.PAID, order.status());
        verify(repository).save(order);
        verify(notificationService).send(
                new PaymentConfirmation(42L, new BigDecimal("59.90")));
    }

    @Test
    void shouldMarkOrderAsFailedAndNotSendConfirmationWhenPaymentIsDeclined() {
        // Arrange
        Order order = new Order(42L, new BigDecimal("59.90"));

        when(repository.findById(42L))
                .thenReturn(Optional.of(order));
        when(paymentGateway.charge(42L, new BigDecimal("59.90")))
                .thenReturn(PaymentResult.declined("Insufficient funds"));

        // Act
        ProcessingResult result = service.process(42L);

        // Assert
        assertEquals(OrderStatus.PAYMENT_FAILED, result.status());
        assertEquals("Insufficient funds", result.message());
        verify(repository).save(order);
        verify(notificationService, never()).send(any());
    }
}
```

Для каждого теста создаются **новые** моки — настройки и вызовы одного теста не влияют на другой.

## 9. Мок и шпион в Mockito

- **Мок** — пустышка: настоящего объекта за ним нет.
- **Шпион (`spy`)** — обёртка над **настоящим** объектом: вызывает реальные методы, пока вы не переопределите какие-то из них.

```java
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.spy;

import java.util.ArrayList;
import java.util.List;

List<String> mockList = mock(ArrayList.class);
List<String> spyList = spy(new ArrayList<>());
```

При вызове `add("paid")`:
- мок только запишет вызов, ничего не сохранит;
- шпион реально добавит элемент в список.

```java
spyList.add("paid");

assertEquals(1, spyList.size());
verify(spyList).add("paid");
```

### `@Spy`

```java
import org.mockito.Spy;

@Spy
private List<String> messages = new ArrayList<>();
```

Начальное состояние видно сразу.

### Как безопасно настраивать шпиона

Эта строка **вызовет настоящий метод** во время настройки:

```java
when(spyList.get(0)).thenReturn("prepared");
```

Список пустой → настоящий `get(0)` бросит `IndexOutOfBoundsException`, и настройка не успеет выполниться.

Для шпионов используйте форму `doReturn(...).when(...)` — она не вызывает реальный метод:

```java
import static org.mockito.Mockito.doReturn;

doReturn("prepared").when(spyList).get(0);
```

### Когда нужен шпион

- настоящий объект делает что-то полезное;
- нужно подменить или подсмотреть только маленькую часть;
- полная замена потребовала бы много настройки.

Осторожно: частая потребность в шпионе — признак того, что класс слишком сложный или делает много всего. Часто лучше протестировать через публичное поведение или вынести часть логики в отдельный класс.

**Не шпионьте за тестируемым классом**, чтобы проверить его внутренние вызовы — тест привяжется к реализации.

## 10. Ловим аргумент: `ArgumentCaptor`

Иногда сервис **сам создаёт** объект и передаёт его помощнику. Чтобы проверить этот объект, его можно «поймать».

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.Mockito.verify;

import org.mockito.ArgumentCaptor;

ArgumentCaptor<PaymentConfirmation> captor =
        ArgumentCaptor.forClass(PaymentConfirmation.class);

verify(notificationService).send(captor.capture());

PaymentConfirmation confirmation = captor.getValue();

assertEquals(42L, confirmation.orderId());
assertEquals(new BigDecimal("59.90"), confirmation.amount());
```

`capture()` ставим внутрь `verify()`, потом `getValue()` отдаёт пойманный объект.

### `@Captor`

```java
import org.mockito.ArgumentCaptor;
import org.mockito.Captor;

@Captor
private ArgumentCaptor<PaymentConfirmation> confirmationCaptor;
```

Используется так же:

```java
verify(notificationService).send(confirmationCaptor.capture());

PaymentConfirmation confirmation = confirmationCaptor.getValue();
assertEquals(42L, confirmation.orderId());
```

Это просто удобный и типобезопасный способ создать каптор.

### Матчер или каптор?

**Матчер** — если хватает короткого условия:

```java
verify(repository).findById(anyLong());
```

**Каптор** — если нужно проверить несколько полей реального объекта:

```java
verify(notificationService).send(confirmationCaptor.capture());

assertEquals(42L, confirmationCaptor.getValue().orderId());
assertEquals(new BigDecimal("59.90"),
        confirmationCaptor.getValue().amount());
```

**Ни то ни другое** — если можно просто сравнить через `equals`:

```java
verify(notificationService).send(
        new PaymentConfirmation(42L, new BigDecimal("59.90")));
```

Здесь оба способа верны, но прямое сравнение короче (record сравнивается по значениям). Каптор полезнее для сложных объектов или когда важны только некоторые поля.

Каптор — для **проверки**, а не для настройки. В Arrange он только запутает.

## 11. Полезные варианты настройки

**Бросить исключение** (для методов, которые что-то возвращают):

```java
when(paymentGateway.charge(anyLong(), any(BigDecimal.class)))
        .thenThrow(new IllegalStateException("Gateway unavailable"));
```

Так проверяем, как сервис реагирует на сбой помощника.

**Разные ответы на последовательные вызовы:**

```java
when(repository.findById(42L))
        .thenReturn(Optional.empty(), Optional.of(order));
```

Первый вызов вернёт пустой `Optional`, второй и следующие — заказ. Используйте, только если несколько вызовов действительно важны, а не чтобы склеить разные сценарии в один тест.

**`void`-методы** у мока по умолчанию ничего не делают, `doNothing()` обычно не нужен. Чтобы `void`-метод бросил исключение — `doThrow()`:

```java
import static org.mockito.Mockito.doThrow;

doThrow(new IllegalStateException("Service unavailable"))
        .when(notificationService)
        .send(any(PaymentConfirmation.class));
```

Формы `doReturn()`, `doThrow()`, `doAnswer()` нужны там, где обычный `when(mock.method())` не подходит — особенно со шпионами.

Настраивайте только то, что нужно для сценария.

## 12. Частые ошибки

**Замокали тестируемый класс:**

```java
OrderProcessingService service =
        mock(OrderProcessingService.class); // Wrong target
```

Реальная логика не выполняется. Сервис — настоящий, моки — только помощники.

**Забыли включить Mockito** — поля с `@Mock` останутся `null`:

```java
@ExtendWith(MockitoExtension.class)
class OrderProcessingServiceTest {
}
```

**Настроили мок после вызова:**

```java
service.process(42L);

when(repository.findById(42L))
        .thenReturn(Optional.of(order)); // Too late
```

Настройка — всегда до действия, в Arrange.

**Смешали матчеры и обычные значения:**

```java
// Incorrect
when(paymentGateway.charge(anyLong(), new BigDecimal("59.90")))
        .thenReturn(PaymentResult.approved());
```

Либо все аргументы точные, либо все — матчеры.

**`any()` везде** — тест не заметит, если сервис передал неверное значение. Где аргумент важен — точное значение, `eq()` или каптор.

**Настроили то, что не используется** — лишний шум, и Mockito может на это ругаться (strict stubbing).

**Проверяют каждую мелочь** — каждый геттер, каждое чтение, порядок шагов. Такие тесты мешают рефакторингу.

**`verify` вместо проверки результата:**

```java
service.process(42L);
verify(repository).findById(42L);
```

Это доказывает только, что поиск был, но не что результат правильный. Нужны ассерты на результат.

**Случайно вызвали реальный метод шпиона** через `when(spy.method())`. Используйте `doReturn(...).when(spy)`.

**Считают `@InjectMocks` полноценным DI.** Это лишь удобство для теста. Если непонятно, как он подставил зависимости, — создайте объект руками.

## 13. Практика: допишите тесты

Используйте `@ExtendWith(MockitoExtension.class)`, `@Mock` и `@InjectMocks`.

### Сценарий 1: заказ не найден

Требования:
- `repository.findById(99L)` возвращает `Optional.empty()`;
- `service.process(99L)` бросает `IllegalArgumentException`;
- платёжный шлюз ни разу не вызывается;
- уведомление не отправляется.

Заготовка:

```java
@Test
void shouldStopProcessingWhenOrderDoesNotExist() {
    // Arrange

    // Act and Assert
}
```

### Сценарий 2: платёж одобрен

Требования:
- репозиторий возвращает настоящий `Order`;
- шлюз возвращает одобрение;
- статус результата — `PAID`;
- заказ сохраняется один раз;
- отправленное подтверждение содержит правильные ID заказа и сумму.

Для последнего пункта используйте `ArgumentCaptor<PaymentConfirmation>`.

### Сценарий 3: платёж отклонён

Требования:
- шлюз возвращает отказ с сообщением `"Limit exceeded"`;
- статус результата — `PAYMENT_FAILED`;
- результат содержит сообщение об отказе;
- обновлённый заказ сохраняется один раз;
- уведомление ни разу не вызывается.

### Решение

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyLong;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import java.math.BigDecimal;
import java.util.Optional;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class)
class OrderProcessingServicePracticeTest {

    @Mock
    private OrderRepository repository;

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private NotificationService notificationService;

    @InjectMocks
    private OrderProcessingService service;

    @Test
    void shouldStopProcessingWhenOrderDoesNotExist() {
        // Arrange
        when(repository.findById(99L))
                .thenReturn(Optional.empty());

        // Act and Assert
        assertThrows(IllegalArgumentException.class,
                () -> service.process(99L));

        verify(paymentGateway, never())
                .charge(anyLong(), any(BigDecimal.class));
        verify(notificationService, never()).send(any());
    }

    @Test
    void shouldSaveOrderAndSendCorrectConfirmationWhenPaymentIsApproved() {
        // Arrange
        Order order = new Order(42L, new BigDecimal("59.90"));

        when(repository.findById(42L))
                .thenReturn(Optional.of(order));
        when(paymentGateway.charge(42L, new BigDecimal("59.90")))
                .thenReturn(PaymentResult.approved());

        // Act
        ProcessingResult result = service.process(42L);

        // Assert
        assertEquals(OrderStatus.PAID, result.status());
        verify(repository, times(1)).save(order);

        ArgumentCaptor<PaymentConfirmation> captor =
                ArgumentCaptor.forClass(PaymentConfirmation.class);
        verify(notificationService).send(captor.capture());

        PaymentConfirmation confirmation = captor.getValue();
        assertEquals(42L, confirmation.orderId());
        assertEquals(new BigDecimal("59.90"), confirmation.amount());
    }

    @Test
    void shouldSaveFailedOrderAndNotNotifyWhenPaymentIsDeclined() {
        // Arrange
        Order order = new Order(42L, new BigDecimal("59.90"));

        when(repository.findById(42L))
                .thenReturn(Optional.of(order));
        when(paymentGateway.charge(42L, new BigDecimal("59.90")))
                .thenReturn(PaymentResult.declined("Limit exceeded"));

        // Act
        ProcessingResult result = service.process(42L);

        // Assert
        assertEquals(OrderStatus.PAYMENT_FAILED, result.status());
        assertEquals("Limit exceeded", result.message());
        verify(repository, times(1)).save(order);
        verify(notificationService, never()).send(any());
    }
}
```

В первом тесте `never()` проверяет только две операции, которые точно не должны произойти после неудачного поиска, а не все возможные методы.

Во втором и третьем `times(1)` написан для тренировки. В обычных тестах `verify(repository).save(order)` означает то же самое и короче.

## 14. Проверь себя

### Вопросы

1. Что создаёт `mock(OrderRepository.class)`?
2. Почему важные ответы помощников нужно настраивать явно, а не полагаться на значения по умолчанию?
3. Сколько вызовов ожидает `verify(repository).save(order)`?
4. Когда `never()` полезнее, чем `verifyNoInteractions()`?
5. Почему `when(gateway.charge(anyLong(), amount))` не работает, если `amount` — обычное значение?
6. Что делает `@ExtendWith(MockitoExtension.class)` с полями тест-класса?
7. Какая роль у `@InjectMocks`?
8. Главное отличие мока Mockito от шпиона Mockito?
9. Почему `doReturn(...).when(spy)` иногда безопаснее, чем `when(spy.method()).thenReturn(...)`?
10. Когда `ArgumentCaptor` лучше широкого матчера?

### Ответы

1. Дублёра `OrderRepository`, созданного во время выполнения: его можно настроить, и он запоминает вызовы для проверки.
2. Явная настройка показывает, какой сценарий задуман, и не даёт тесту пройти только из-за случайного значения по умолчанию.
3. Ровно один подходящий вызов.
4. Когда запрещено одно конкретное действие, а другие вызовы этого помощника разрешены или не важны.
5. Если один аргумент — матчер, все аргументы должны быть матчерами. Оберните значение в `eq(amount)` или передайте оба аргумента точными значениями.
6. Создаёт объекты для `@Mock`, `@Spy`, `@Captor`, `@InjectMocks` и встраивает Mockito в жизненный цикл JUnit 5.
7. Просит Mockito создать настоящий тестируемый объект и подставить в него моки/шпионы. Это удобство для теста, а не DI-фреймворк.
8. Мок без настройки ведёт себя по умолчанию Mockito; шпион оборачивает настоящий объект и вызывает реальные методы, пока их не переопределят.
9. `when(spy.method())` вызывает настоящий метод во время настройки, а `doReturn()` — нет.
10. Когда нужно посмотреть на реальный переданный объект — особенно на несколько его полей — после вызова.

## Шпаргалка

- `mock()` — создаёт настраиваемого дублёра, который запоминает вызовы.
- `when(...).thenReturn(...)` — подсовывает нужный ответ.
- Настройка зависит от аргументов; точные значения — самые понятные.
- `verify()` — проверяет важный вызов, по умолчанию ровно один.
- `times(n)` — если точное число важно; `never()` — если действие запрещено.
- Матчеры: если один аргумент матчер — все матчеры (`eq()` для точных).
- `@ExtendWith(MockitoExtension.class)` — включает аннотации.
- `@Mock` — на помощников, `@InjectMocks` — на тестируемый класс.
- Шпион вызывает реальные методы; настраивать его — через `doReturn()`.
- `ArgumentCaptor` — поймать аргумент и проверить его поля.
- Главное в тесте — результат и действительно важные вызовы, а не все возможности Mockito.

## Почитать

- Документация Mockito
- Расширение Mockito для JUnit Jupiter
- Документация `ArgumentMatchers`
- Документация `ArgumentCaptor`

Продвинутые возможности изучайте, когда они реально понадобятся, а не добавляйте их в тест «потому что есть».
