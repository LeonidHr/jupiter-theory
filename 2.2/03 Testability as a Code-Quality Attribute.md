# Тестируемость как признак качества кода

В прошлой теме `OrderProcessingService` получал зависимости через конструктор, поэтому тесты с Mockito писались легко. Это не трюк, а дизайн: зависимости **видны и заменяемы**.

**Тестируемость** — насколько легко поставить код в нужную ситуацию, запустить и проверить результат. Класс может правильно работать в продакшене, но плохо тестироваться: например, сам создаёт репозиторий базы, читает системное время или берёт сервис из static-синглтона. Всё это прячет от теста важные входы.

## 1. Что такое тестируемость

Три вопроса к коду:

1. **Может ли тест управлять входами и условиями?**
2. **Может ли тест увидеть результат или важные эффекты?**
3. **Можно ли запустить код, не задев лишние внешние системы?**

Если код всегда берёт время у ОС — тест не выберет нужный момент. Если метод сам создаёт HTTP-клиент — запрос не перехватить без реального провайдера.

| Свойство | Вопрос | Пример |
|---|---|---|
| Управляемость (controllability) | Можно ли создать нужное условие? | Шлюз отклоняет платёж |
| Наблюдаемость (observability) | Можно ли проверить результат? | Проверить статус или уведомление |
| Изоляция (isolation) | Можно ли обойтись без внешних систем? | Без реальной базы и сети |

Тестируемость не означает «везде моки». Чистому расчёту хватит обычных значений и ассерта.

> **Главное:** тестируемый код открывает важные входы, зависимости, результаты и эффекты, не выставляя наружу лишних деталей.

## 2. Зависимости — нормально, скрытые — проблема

**Зависимость** — то, что нужно классу для работы: репозиторий, платёжный шлюз, уведомления, текущее время, конфигурация, файлы.

Иметь зависимости — нормально. Важно: видна ли зависимость? Кто её создаёт? Может ли тест подставить свою? Есть ли выход наружу или побочный эффект?

### Явная зависимость

Видна в конструкторе:

```java
final class PaymentReminderService {
    private final ReminderRepository reminderRepository;
    private final NotificationService notificationService;
    private final Clock clock;

    PaymentReminderService(
            ReminderRepository reminderRepository,
            NotificationService notificationService,
            Clock clock) {
        this.reminderRepository =
                Objects.requireNonNull(reminderRepository);
        this.notificationService =
                Objects.requireNonNull(notificationService);
        this.clock = Objects.requireNonNull(clock);
    }
}
```

По конструктору сразу понятно, что нужно классу. Тест передаст стаб, мок, фейк, фиксированные часы или настоящий объект.

### Скрытая зависимость

Берётся внутри метода:

```java
final class PaymentReminderService {

    void sendReminder(long orderId) {
        ReminderRepository repository =
                new JdbcReminderRepository(Database.connect());
        NotificationService notifications =
                NotificationServiceRegistry.instance();
        Instant now = Instant.now();

        // business logic
    }
}
```

В сигнатуре только `orderId`, а на деле метод зависит от базы, конкретного JDBC-репозитория, глобального реестра и системных часов. Тест не может их выбрать — узнаёт о них, только прочитав код или упав в чужом окружении.

> **Главное:** зависимость, которой нет в конструкторе, не становится безобидной. Она становится скрытой.

## 3. Признаки плохой тестируемости

### 3.1. Static-синглтоны и реестры сервисов

```java
PaymentGateway gateway = PaymentGatewayRegistry.instance();
PaymentResult result = gateway.charge(order.id(), order.total());
```

Проблемы: зависимость не видна; тесты делят один глобальный объект; нужна особая настройка и очистка; тесты зависят от порядка запуска; параллельный запуск рискован; непонятно, какая реализация активна.

Сам `static` не виноват. Статическая функция без состояния тестируется легко:

```java
BigDecimal tax = TaxCalculator.calculate(subtotal, taxRate);
```

Проблема — **глобальный доступ к изменяемому состоянию, внешним ресурсам или заменяемым сервисам**.

### 3.2. Жёстко зашитое создание объектов

```java
final class OrderProcessingService {

    ProcessingResult process(long orderId) {
        OrderRepository repository =
                new JdbcOrderRepository(Database.connect());
        PaymentGateway gateway =
                new HttpPaymentGateway(HttpClient.newHttpClient());

        // business logic
    }
}
```

Тест может создать мок, но передать его некуда — метод уже выбрал `JdbcOrderRepository`.

`new` сам по себе не вреден. Создавать значения, коллекции, мелкие внутренние объекты — нормально:

```java
PaymentConfirmation confirmation =
        new PaymentConfirmation(order.id(), order.total());
```

Проблема, когда так создаётся важный помощник, который делает I/O, медленный или непредсказуемый, имеет побочные эффекты, должен вести себя по-разному в разных сценариях или стоит на архитектурной границе.

### 3.3. Скрытый ввод-вывод

```java
ProcessingResult process(long orderId) throws IOException {
    Order order = loadOrder(orderId);

    Files.writeString(
            Path.of("audit.log"),
            "Processing order " + orderId);

    return calculateResult(order);
}
```

Тест теперь зависит от прав, путей, содержимого файла, кодировки и очистки — и может мусорить на машине разработчика. По имени метода эффект не виден.

### 3.4. Прямое обращение к текущему времени

Время — это вход, даже если его нет в параметрах:

```java
if (Instant.now().isAfter(order.paymentDeadline())) {
    return PaymentStatus.EXPIRED;
}
```

Результат зависит от момента выполнения; прошлую или будущую дату не воспроизвести; `Thread.sleep()` делает тест медленным и всё равно ненадёжным; часовой пояс зависит от машины.

Решение — `java.time.Clock`: в продакшене системные часы, в тесте — фиксированные.

### 3.5. Случайность, окружение, глобальная конфигурация

```java
int confirmationCode = new Random().nextInt(100_000);
String region = System.getenv("PAYMENT_REGION");
Locale locale = Locale.getDefault();
```

Если решение зависит от этих значений, тест должен ими управлять — иначе он ведёт себя по-разному на разных запусках и машинах. Свой интерфейс не всегда нужен: часто конфигурацию читают один раз на границе приложения и передают обычным значением.

### 3.6. Слишком много обязанностей

Класс, который валидирует, считает цены, ходит в базу, шлёт HTTP, форматирует и пишет аудит, требует много помощников и много настройки в тестах. Длинный список моков для простого поведения — сигнал, что класс координирует слишком много. Сложную настройку теста воспринимайте как отзыв о дизайне, а не как повод добавить ещё Mockito.

## 4. Разбор сильно связанного кода

```java
final class OrderProcessingService {

    ProcessingResult process(long orderId) throws IOException {
        OrderRepository repository =
                new JdbcOrderRepository(Database.connect());
        PaymentGateway gateway =
                PaymentGatewayRegistry.instance();

        Order order = repository.findById(orderId)
                .orElseThrow();

        if (Instant.now().isAfter(order.paymentDeadline())) {
            return ProcessingResult.expired();
        }

        PaymentResult payment =
                gateway.charge(order.id(), order.total());

        Files.writeString(
                Path.of("audit.log"),
                payment.message());

        return ProcessingResult.from(payment);
    }
}
```

Логика маленькая, а скрытых зависимостей много:

| Код | Что скрыто | Почему мешает |
|---|---|---|
| `Database.connect()` | База и конфигурация | Нужно окружение, медленно |
| `new JdbcOrderRepository(...)` | Конкретное хранилище | Не подставить стаб |
| `PaymentGatewayRegistry.instance()` | Глобальный сервис | Общее состояние, реализацию выбирают где-то ещё |
| `Instant.now()` | Системное время | Не выбрать граничный момент |
| `Files.writeString(...)` | Запись в файл | Пути, права, очистка |

Mockito здесь не поможет: мок полезен, только если класс может его получить. Инструменты, перехватывающие конструкторы и static-методы, иногда спасают легаси, но не делают зависимости видимыми и не улучшают дизайн.

> **Диагностический вопрос:** если тест создал управляемого помощника — куда он может его передать?

## 5. Внедрение зависимостей (DI) делает их явными

**Dependency injection** — объект получает зависимость снаружи, а не создаёт и не ищет её сам. Самый простой вариант — через конструктор:

```java
final class OrderProcessingService {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final NotificationService notificationService;
    private final Clock clock;

    OrderProcessingService(
            OrderRepository orderRepository,
            PaymentGateway paymentGateway,
            NotificationService notificationService,
            Clock clock) {
        this.orderRepository =
                Objects.requireNonNull(orderRepository);
        this.paymentGateway =
                Objects.requireNonNull(paymentGateway);
        this.notificationService =
                Objects.requireNonNull(notificationService);
        this.clock = Objects.requireNonNull(clock);
    }

    // behavior
}
```

Плюсы: зависимости видны в одном месте; без них объект не создать; поля `final`; продакшен даёт настоящие реализации, тест — управляемые; класс не знает, как зависимости создаются.

### Сборка в продакшене

Настоящие объекты создаются в месте сборки/конфигурации, а не в бизнес-методе:

```java
OrderProcessingService service = new OrderProcessingService(
        new JdbcOrderRepository(dataSource),
        new HttpPaymentGateway(httpClient),
        new EmailNotificationService(mailClient),
        Clock.systemUTC());
```

Тут выбирают реализации, а сервис их просто использует. **Конфигурация отделена от использования.** Spring может автоматизировать сборку, но фреймворк не обязателен — вызов конструктора тоже DI.

### Сборка в тесте

```java
OrderRepository repository = mock(OrderRepository.class);
PaymentGateway gateway = mock(PaymentGateway.class);
NotificationService notifications =
        mock(NotificationService.class);
Clock clock = Clock.fixed(
        Instant.parse("2026-07-15T10:00:00Z"),
        ZoneOffset.UTC);

OrderProcessingService service = new OrderProcessingService(
        repository,
        gateway,
        notifications,
        clock);
```

Сервис настоящий, а для каждой зависимости тест выбирает дублёра или настоящий предсказуемый объект. Фиксированный `Clock` понятнее, чем мок часов.

> **Главное:** DI — это не «внедрять моки», а «дать вызывающему передать зависимости». Продакшен и тест выбирают своё.

## 6. Интерфейсы на заменяемых границах

Интерфейс описывает, что нужно классу, без привязки к реализации:

```java
interface PaymentGateway {
    PaymentResult charge(long orderId, BigDecimal amount);
}
```

В продакшене — HTTP-реализация:

```java
final class HttpPaymentGateway implements PaymentGateway {
    // communicates with the real provider
}
```

В тесте — дублёр Mockito:

```java
PaymentGateway gateway = mock(PaymentGateway.class);
```

Сервис зависит от **роли** («списать платёж»), а не от HTTP-клиента или SDK. Хорошие кандидаты: репозитории, внешние шлюзы и клиенты, публикаторы сообщений, уведомления, хранилища, политики с реальными альтернативами.

### Одного интерфейса мало

```java
final class OrderProcessingService {
    private final PaymentGateway paymentGateway =
            new HttpPaymentGateway(HttpClient.newHttpClient());
}
```

Тип — интерфейс, но реализация зашита, и подменить её нельзя. Заменяемость дают **абстракция + точка внедрения** вместе.

### Интерфейс нужен не всегда

Не создавайте интерфейс для каждого класса ради тестов. Внедрять можно и конкретный тип, если он подходит: `Clock` — абстрактный класс, специально рассчитанный на подмену. Простого предсказуемого помощника проще взять настоящим. `OrderServiceInterface` с одной реализацией и без архитектурного смысла — просто шум.

> **Главное:** интерфейс отделяет роль от реализации, внедрение делает реализацию заменяемой. Ни то ни другое не добавляют механически.

## 7. Время как явная зависимость

Оплата разрешена только до дедлайна заказа.

### Скрытое время

```java
PaymentStatus statusFor(Order order) {
    if (Instant.now().isAfter(order.paymentDeadline())) {
        return PaymentStatus.EXPIRED;
    }

    return PaymentStatus.AVAILABLE;
}
```

Скрытый вход — момент, который вернули системные часы.

### Внедрённые часы

```java
final class PaymentAvailabilityService {
    private final Clock clock;

    PaymentAvailabilityService(Clock clock) {
        this.clock = Objects.requireNonNull(clock);
    }

    PaymentStatus statusFor(Order order) {
        Instant now = clock.instant();

        if (now.isAfter(order.paymentDeadline())) {
            return PaymentStatus.EXPIRED;
        }

        return PaymentStatus.AVAILABLE;
    }
}
```

В продакшене — реальное время:

```java
PaymentAvailabilityService service =
        new PaymentAvailabilityService(Clock.systemUTC());
```

В тесте — фиксированное:

```java
@Test
void reportsExpiredWhenCurrentTimeIsAfterDeadline() {
    Clock clock = Clock.fixed(
            Instant.parse("2026-07-15T10:00:01Z"),
            ZoneOffset.UTC);
    PaymentAvailabilityService service =
            new PaymentAvailabilityService(clock);
    Order order = new Order(
            42L,
            Instant.parse("2026-07-15T10:00:00Z"));

    PaymentStatus result = service.statusFor(order);

    assertEquals(PaymentStatus.EXPIRED, result);
}
```

Тест быстрый и предсказуемый: без `sleep` и без зависимости от даты запуска. И без мока — `Clock.fixed(...)` уже делает ровно то, что нужно.

### Граница всё равно задаётся требованием

Тестируемость позволяет проверить точный момент, но не решает бизнес-правило. Здесь ровно в дедлайн оплата доступна: `isAfter()` при равных моментах даёт `false`. Это решение фиксирует отдельный тест:

```java
@Test
void reportsAvailableAtTheExactDeadline() {
    Instant deadline =
            Instant.parse("2026-07-15T10:00:00Z");
    Clock clock = Clock.fixed(deadline, ZoneOffset.UTC);
    PaymentAvailabilityService service =
            new PaymentAvailabilityService(clock);
    Order order = new Order(42L, deadline);

    PaymentStatus result = service.statusFor(order);

    assertEquals(PaymentStatus.AVAILABLE, result);
}
```

Управление временем делает тест возможным, а ожидаемый ответ задаёт требование.

## 8. Чистые функции — меньше дублёров

**Чистая функция**:
1. На одни и те же входы всегда даёт один результат.
2. Не меняет ничего снаружи.

Она не читает время, не ходит в базу, не трогает глобальное состояние, не пишет файлы и не шлёт сообщения. Всё приходит через параметры, результат возвращается.

```java
enum CustomerTier {
    STANDARD,
    PREMIUM
}

final class DiscountCalculator {

    private DiscountCalculator() {
    }

    static BigDecimal calculate(
            BigDecimal subtotal,
            CustomerTier tier) {
        Objects.requireNonNull(subtotal);
        Objects.requireNonNull(tier);

        if (subtotal.signum() < 0) {
            throw new IllegalArgumentException(
                    "subtotal must not be negative");
        }

        BigDecimal rate = switch (tier) {
            case STANDARD -> new BigDecimal("0.00");
            case PREMIUM -> new BigDecimal("0.10");
        };

        return subtotal.multiply(rate);
    }
}
```

Тест без всякого Mockito:

```java
@ParameterizedTest
@CsvSource({
        "100.00, STANDARD, 0.0000",
        "100.00, PREMIUM, 10.0000",
        "0.00, PREMIUM, 0.0000"
})
void calculatesDiscount(
        String subtotal,
        CustomerTier tier,
        String expectedDiscount) {
    BigDecimal result = DiscountCalculator.calculate(
            new BigDecimal(subtotal), tier);

    assertEquals(new BigDecimal(expectedDiscount), result);
}
```

Функция статическая, но отлично тестируется. Поэтому правило «static плохо тестируется» слишком грубое: плох static-доступ к изменяемому синглтону или внешнему сервису, а не static чистая функция.

### Отделяйте решения от эффектов

В сервисах обычно смешаны **решения и расчёты** (право на скидку, суммы, смена статуса) и **эффекты** (чтение, сохранение, отправка, публикация). Если расчёт перемешан с I/O, для каждого теста расчёта нужны дублёры. Вынесите расчёт в чистую функцию, а тонкий слой оркестрации пусть общается с помощниками:

```java
BigDecimal discount =
        DiscountCalculator.calculate(order.subtotal(), order.tier());
BigDecimal amountToCharge = order.subtotal().subtract(discount);

PaymentResult payment =
        paymentGateway.charge(order.id(), amountToCharge);
```

`DiscountCalculator` проверяется подробно обычными значениями, а оркестрации хватит нескольких сценариев.

### Плюсы чистых функций

Предсказуемые, легко подготовить, запускаются изолированно, проверяются по возвращаемому значению, безопасны для параллельного запуска, не требуют очистки, удобны для параметризованных тестов.

Весь код чистым не сделать — приложение должно читать и писать данные. Цель: держать эффекты явными и у границ, а логику решений — максимально независимой от них.

## 9. Тестируемость делает тесты надёжнее и понятнее

- **Предсказуемость** — управляемые время, случайность и конфигурация дают одинаковый результат на каждом запуске.
- **Сфокусированная подготовка** — явные зависимости позволяют настроить только нужное.
- **Понятные падения** — упавший тест чистого расчёта указывает на расчёт; тест с базой, сетью и конфигурацией машины может упасть по десятку причин.
- **Безопасный параллельный запуск** — тесты без глобального состояния не мешают друг другу.
- **Быстрая обратная связь** — обычные объекты, фиксированные часы и моки быстрее внешних систем, а быстрые тесты чаще запускают.

Это полезно и продакшен-коду: видимые зависимости проще понять, отделённую конфигурацию проще заменить, чистую логику проще переиспользовать.

> **Главное:** тестируемость — признак качества, потому что те же решения улучшают ясность, модульность и сопровождаемость.

## 10. Mockito не чинит плохо выделенный юнит

Mockito работает, когда у класса уже есть заменяемые помощники. Он не решает, что такое юнит, и не разделяет обязанности.

### Моку нужна точка внедрения

```java
OrderRepository repository = mock(OrderRepository.class);
OrderProcessingService service = new OrderProcessingService();
```

Мок есть, но сервис его не получает — бесполезно.

### `@InjectMocks` — не DI в продакшене

Он лишь собирает объект в тесте. Скрытые синглтоны, прямую запись в файл или создание шлюза внутри метода он не уберёт.

### Много моков — сигнал

Несколько дублёров — не обязательно плохо: оркестратор и должен координировать. Но постоянная сложная настройка может говорить о слишком многих обязанностях, размытой границе юнита, логике вперемешку с I/O, слишком «болтливых» помощниках или тестах, привязанных к шагам реализации. Ответ — не всегда «больше моков», иногда нужен лучший дизайн.

## 11. Не проектируйте только под тест-фреймворк

- **Не делайте интерфейс для каждого класса.** Только для осмысленной роли, границы или реальных альтернатив. `OrderInterface` для value-объекта ради Mockito не нужен.
- **Не открывайте private-методы ради тестов.** Тестируйте через публичное поведение. Если в private-методе живёт самостоятельная идея — вынесите её в отдельный класс.
- **Не подменяйте всё подряд.** Record-ы, коллекции, value-объекты и чистые помощники часто понятнее моков.
- **Не отказывайтесь от интеграционных тестов.** Юнит-тест докажет, что сервис отправил правильный запрос в `PaymentGateway`, но не что `HttpPaymentGateway` правильно общается с провайдером.
- **Не путайте тестируемость с максимумом абстракций.** У каждой абстракции есть цена — выбирайте минимальный дизайн, где важные зависимости и эффекты явные.

## 12. Чек-лист тестируемости

1. Какая главная обязанность класса?
2. Какие значения и условия влияют на его решения?
3. Какие помощники делают I/O или внешние эффекты?
4. Видны ли эти зависимости в конструкторе или параметрах?
5. Может ли тест подставить управляемые альтернативы?
6. Читаются ли напрямую время, случайность, окружение, глобальное состояние?
7. Можно ли проверить результат через возвращаемое значение или важный вызов?
8. Может ли расчётная логика работать без инфраструктуры?
9. Не понятнее ли будет настоящий предсказуемый объект вместо мока?
10. Не говорит ли настройка теста, что у класса слишком много обязанностей?

Чек-лист — для диагностики, а не требование заводить интерфейс или мок на каждый пункт.

## 13. Практика: оцените дизайн

Рассмотрите код:

```java
final class ShipmentDeadlineService {

    DispatchDecision decide(long orderId) throws IOException {
        OrderRepository repository =
                ServiceRegistry.orderRepository();
        Order order = repository.findById(orderId)
                .orElseThrow();

        Instant now = Instant.now();
        DispatchDecision decision = now.isAfter(order.shipBy())
                ? DispatchDecision.TOO_LATE
                : DispatchDecision.ALLOWED;

        Files.writeString(
                Path.of("dispatch-audit.log"),
                orderId + ":" + decision);

        return decision;
    }
}
```

Ответьте на вопросы:

1. Какие зависимости и эффекты скрыты?
2. Почему тесты могут влиять друг на друга?
3. Какой вход делает тест границы непредсказуемым?
4. Какие зависимости стоит передавать снаружи?
5. Нужен ли интерфейс для каждой передаваемой зависимости?
6. Какой результат уже можно проверить без мока?

### Предлагаемые ответы

1. Репозиторий берётся из глобального реестра, текущее время — из системных часов, запись аудита идёт напрямую в файловую систему.
2. Тесты могут делить или менять реестр сервисов и файл аудита. Состояние, оставленное одним тестом, может повлиять на другой, особенно при параллельном запуске.
3. `Instant.now()` — скрытый вход, из-за которого точные случаи около дедлайна трудно воспроизвести.
4. Репозиторий, часы и место записи аудита (или помощник для аудита) — они пересекают границы или должны управляться тестом.
5. Нет. У `OrderRepository` и помощника для аудита могут быть осмысленные интерфейсы. Существующую абстракцию `Clock` можно внедрять напрямую. Обычные значения, например путь, тоже можно передать при необходимости.
6. `DispatchDecision` возвращается, поэтому решение проверяется обычным ассертом. Мок нужен, только если тест должен управлять помощником или проверить его вызов.

Явно спроектированная версия может начинаться так:

```java
final class ShipmentDeadlineService {
    private final OrderRepository orderRepository;
    private final DispatchAudit dispatchAudit;
    private final Clock clock;

    ShipmentDeadlineService(
            OrderRepository orderRepository,
            DispatchAudit dispatchAudit,
            Clock clock) {
        this.orderRepository =
                Objects.requireNonNull(orderRepository);
        this.dispatchAudit = Objects.requireNonNull(dispatchAudit);
        this.clock = Objects.requireNonNull(clock);
    }

    DispatchDecision decide(long orderId) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow();

        DispatchDecision decision =
                clock.instant().isAfter(order.shipBy())
                        ? DispatchDecision.TOO_LATE
                        : DispatchDecision.ALLOWED;

        dispatchAudit.record(orderId, decision);
        return decision;
    }
}
```

Это направление дизайна. Как безопасно прийти к нему из сильно связанной версии — в следующей теме.

## 14. Проверь себя

### Вопросы

1. Какие три возможности делают юнит легче тестируемым?
2. Почему зависимость из static-синглтона менее заметна, чем зависимость из конструктора?
3. Является ли любое использование `new` признаком плохой тестируемости?
4. Почему создание мока Mockito не помогает, если юнит сам создаёт своего помощника?
5. Делает ли добавление интерфейса зависимость автоматически заменяемой?
6. Почему `Clock.fixed(...)` часто лучше, чем `Thread.sleep()`, в тесте, зависящем от времени?
7. Может ли статический метод быть чистой функцией?
8. Почему чистым функциям обычно нужно меньше тестовых дублёров?
9. О чём может говорить большое количество моков в простом тесте?
10. Отменяет ли хорошая юнит-тестируемость необходимость интеграционных тестов?

### Ответы

1. Тест может управлять важными входами и условиями, наблюдать результат или значимые эффекты и изолировать юнит от лишних внешних границ.
2. Конструктор явно перечисляет зависимость и позволяет вызывающему передать её. Static-поиск спрятан в реализации и часто использует общее глобальное состояние.
3. Нет. Создание значений, коллекций и простых внутренних объектов обычно безвредно. Проблема — зашитый медленный, внешний, хранящий состояние или зависящий от сценария помощник.
4. Юнит никогда не получает мок и продолжает использовать помощника, которого создал сам.
5. Нет. Если класс по-прежнему сам создаёт конкретную реализацию, вызывающий не может её заменить. Нужны вместе подходящая абстракция и точка внедрения.
6. Фиксированные часы делают время точным и предсказуемым, без задержки теста и без зависимости от планировщика.
7. Да. Статические чистые функции легко тестировать, если они используют только аргументы, возвращают результат и не обращаются к общему состоянию или внешним ресурсам.
8. Их входы — обычные значения, результат возвращается напрямую, и нет внешних эффектов, которые нужно подменять или проверять.
9. У юнита может быть слишком много обязанностей, доменная логика может быть смешана с I/O или граница юнита размыта. Это сигнал о дизайне, а не автоматическое доказательство проблемы.
10. Нет. Юнит-тесты проверяют изолированное поведение, интеграционные — что настоящие компоненты и границы работают вместе.

## Шпаргалка

- Тестируемость = управляемость + наблюдаемость + изоляция.
- Зависимости нормальны, скрытые — проблема.
- Плохие признаки: static-синглтоны, зашитые внешние помощники, скрытый I/O, `Instant.now()`, случайность, глобальная конфигурация, много обязанностей.
- Виноваты не `static` и `new`, а скрытое, общее, изменяемое, внешнее или непредсказуемое поведение.
- DI: зависимости передаются снаружи; конфигурация отделена от использования.
- Интерфейсы — на осмысленных границах, не механически.
- `Clock.fixed(...)` — предсказуемое время без мока и `sleep`.
- Чистые функции: всё через параметры, результат возвращается, побочных эффектов нет.
- Mockito помогает тестируемому дизайну, но не создаёт его.
- Юнит-тесты дополняют интеграционные, а не заменяют.

## Почитать

- [Inversion of Control Containers and the Dependency Injection Pattern — Martin Fowler](https://martinfowler.com/articles/injection.html)
- [`Clock` — Java SE 25 API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/time/Clock.html)
- [`Instant` — Java SE 25 API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/time/Instant.html)
- [When to Mock — Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2014/05/10/WhenToMock.html)
