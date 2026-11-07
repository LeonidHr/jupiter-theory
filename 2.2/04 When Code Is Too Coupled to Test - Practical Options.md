# Когда код слишком связан для тестов: практические варианты

В прошлой теме разобрали принципы тестируемости: явные зависимости, DI, интерфейсы, чистые функции. Их легко применять к новому коду. Но в реальных проектах есть рабочий код, который трудно тестировать: контроллер с бизнес-правилами, созданием репозиториев, системным временем, static-вызовами и записью в файлы в одном длинном методе.

Переписать всё сразу — рискованно. Оставить как есть — тоже. Цель темы: не идеальная архитектура за один шаг, а **безопасная обратная связь**, одна полезная точка замены и постепенный перенос логики туда, где её легко проверять.

## 1. Начинайте с риска, а не с Mockito

```java
final class OrderController {

    HttpResponse process(long orderId) throws IOException {
        Connection connection = Database.connect();
        JdbcOrderRepository repository =
                new JdbcOrderRepository(connection);

        Order order = repository.findById(orderId)
                .orElseThrow();

        if (Instant.now().isAfter(order.paymentDeadline())) {
            return HttpResponse.conflict("Payment deadline expired");
        }

        BigDecimal fee = order.total()
                .multiply(new BigDecimal("0.03"));
        PaymentReply reply = PaymentApi.charge(
                order.id(), order.total().add(fee));

        if (!reply.approved()) {
            order.rejectPayment(reply.message());
            repository.save(order);
            return HttpResponse.paymentRequired(reply.message());
        }

        order.markPaid();
        repository.save(order);
        Files.writeString(
                Path.of("payment-audit.log"),
                order.id() + ":" + reply.transactionId(),
                StandardOpenOption.CREATE,
                StandardOpenOption.APPEND);

        return HttpResponse.ok(order.id());
    }
}
```

Метод может быть правильным, но изолированный тест не может: подать заказ без базы, выбрать время, заставить платёжный API одобрить или отклонить, не трогать файл аудита, увидеть запрос без реального провайдера.

Моки сразу не помогут — методу их некуда передать. Сначала спросите:

1. Какое поведение должно остаться неизменным?
2. Какая зависимость или эффект несёт наибольший риск?
3. Где код мог бы принять управляемую альтернативу?
4. Какое минимальное безопасное изменение даст полезную обратную связь?

> **Главное:** в связанном коде первая проблема — отсутствие точки замены, а не отсутствие мок-фреймворка.

## 2. Характеризационные тесты фиксируют текущее поведение

Перед изменением незнакомого кода нужно знать, что он делает сейчас. Документация может быть неполной, а важное поведение — неочевидным.

**Характеризационный тест** записывает наблюдаемое поведение. Полезен, когда тестов мало или нет, спецификация неполная, рефакторинг должен сохранить поведение, есть неизвестные вызывающие или поведение выглядит странно, но пока непонятно, баг ли это.

Старый калькулятор доставки округляет вес вверх:

```java
final class LegacyShippingCalculator {

    BigDecimal calculate(BigDecimal weightInKg) {
        int chargedKg = weightInKg
                .setScale(0, RoundingMode.CEILING)
                .intValueExact();

        return BigDecimal.valueOf(chargedKg * 4L);
    }
}
```

В требованиях только «$4 за килограмм». Фиксируем, что делает код на границе:

```java
@Test
void chargesTheNextWholeKilogramForFractionalWeight() {
    LegacyShippingCalculator calculator =
            new LegacyShippingCalculator();

    BigDecimal result = calculator.calculate(
            new BigDecimal("2.01"));

    assertEquals(new BigDecimal("12"), result);
}
```

Тест не доказывает, что округление вверх правильно. Он доказывает, что код **сейчас** так делает.

### Характеризационный или по спецификации?

| Вопрос | Характеризационный | По спецификации |
|---|---|---|
| Откуда ожидаемый результат? | Из текущего поведения | Из согласованного требования |
| Цель | Поймать случайные изменения при улучшении | Проверить соответствие требованию |
| Может фиксировать нежелательное поведение? | Да, временно | Нет, описывает задуманное |
| Когда правило уточнили | Переименовать, исправить или заменить | Держать в соответствии с контрактом |

Это инструмент безопасности, а не объявление, что всё старое поведение правильно.

### Если узкий юнит-тест пока невозможен

Первый тест может быть шире, чем хотелось бы: временная папка вместо реальной, локальная тестовая база вместо мока, тестовая конфигурация с подменённым внешним адресом, вызов через более высокий уровень. Это будет компонентный или интеграционный тест — нормально, если он надёжен. Не нужно притворяться, что связанный код уже изолированный юнит.

### Что фиксировать

Не каждую строку, а то, что важно вызывающим: обычный успешный результат, важные ошибки, граничные значения, исключения, на которые полагаются, изменения сохранённого состояния, значимые внешние эффекты, поведение вокруг известных багов и инцидентов.

Не проверяйте нестабильное: сгенерированное время, случайные ID, неупорядоченный вывод, целиком сериализованные объекты, если важны пара полей. По возможности управляйте такими значениями или нормализуйте их.

> **Главное:** характеризационный тест говорит «предупреди, если это поведение изменится», а не «это поведение навсегда правильное».

## 3. Найдите шов (seam)

**Шов** — место, где можно поменять поведение или зависимость, не трогая основную логику. В тестах это обычно место, куда подставляется управляемая замена.

Внедрение через конструктор — понятный шов:

```java
final class PaymentService {
    private final PaymentGateway paymentGateway;

    PaymentService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

Продакшен даёт настоящий шлюз, тест — стаб, мок или фейк. Вызов внутри не меняется:

```java
PaymentResult result = paymentGateway.charge(request);
```

| Шов | В продакшене | В тесте |
|---|---|---|
| Параметр конструктора | HTTP-шлюз | Стаб или мок |
| Параметр метода | Текущий пользователь или политика | Тестовое значение или предсказуемая политика |
| Интерфейс / маленькая абстракция | Хранилище в файлах | Фейк в памяти |
| Обёртка / адаптер | Static SDK провайдера | Застабленная обёртка |
| `Clock` / `InstantSource` | Системное время | Фиксированное время |
| Конфигурация | Продакшен-адрес | Локальный тестовый адрес |

Лучший шов — не самая общая абстракция, а **минимальная осмысленная граница**, которая даёт тесту контроль.

### Шов должен быть доступен

Интерфейс бесполезен, если класс сам создаёт реализацию:

```java
interface PaymentGateway {
    PaymentReply charge(PaymentRequest request);
}

final class PaymentService {
    private final PaymentGateway paymentGateway =
            new HttpPaymentGateway();
}
```

Нужна точка внедрения:

```java
final class PaymentService {
    private final PaymentGateway paymentGateway;

    PaymentService(PaymentGateway paymentGateway) {
        this.paymentGateway =
                Objects.requireNonNull(paymentGateway);
    }
}
```

### По одному шву за раз

Если метод использует базу, SDK, файлы, часы и конфигурацию, заменять всё сразу трудно проверить. Начинайте с того, что нужно для ближайшего изменения:

1. обернуть платёжный SDK;
2. добавить тесты на одобрение и отказ;
3. внедрить время для теста дедлайна;
4. позже вынести хранение и аудит.

После каждого шага код компилируется, тесты проходят.

## 4. Оборачивайте static-вызовы и API провайдеров

`Math.max(...)` предсказуем и в обёртке не нуждается. Проблема — когда static-API пересекает границу, читает меняющееся состояние или делает внешний эффект:

```java
PaymentReply reply = PaymentApi.charge(
        order.id(), amount, System.getenv("PAYMENT_TOKEN"));
```

Здесь привязка к static-точке входа, типам провайдера, окружению процесса и реальному эффекту. Небольшой адаптер создаёт границу:

```java
interface PaymentGateway {
    PaymentResult charge(PaymentRequest request);
}

final class ProviderPaymentGateway implements PaymentGateway {
    private final String token;

    ProviderPaymentGateway(String token) {
        this.token = Objects.requireNonNull(token);
    }

    @Override
    public PaymentResult charge(PaymentRequest request) {
        PaymentReply reply = PaymentApi.charge(
                request.orderId(),
                request.amount(),
                token);

        return new PaymentResult(
                reply.approved(),
                reply.transactionId(),
                reply.message());
    }
}
```

Сервис зависит от роли приложения:

```java
final class PaymentService {
    private final PaymentGateway paymentGateway;

    PaymentService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    PaymentResult pay(PaymentRequest request) {
        return paymentGateway.charge(request);
    }
}
```

Плюсы: тесты подменяют `PaymentGateway`; типы провайдера не расползаются по домену; токен читается в конфигурации; преобразование моделей — на границе; смена провайдера затрагивает меньше классов.

**Обёртка или адаптер?** Обёртка — объект приложения вокруг чужого API; адаптер ещё и переводит одну модель в другую. Название не важно — важно, что внешние детали спрятаны за маленькой границей приложения.

**Не копируйте весь внешний API.** Адаптер даёт то, что нужно приложению. Узкий интерфейс понятнее:

```java
interface PaymentGateway {
    PaymentResult charge(PaymentRequest request);
}
```

**Static-мокинг — не то же самое, что шов.** Перехват static-вызовов выручает, когда код пока нельзя менять, но не делает зависимость явной. Это временный мост, а не решение проблемы дизайна.

## 5. Спрячьте I/O за границей

Прямая запись в файл смешивает решение приложения с механизмом хранения:

```java
Files.writeString(
        Path.of("payment-audit.log"),
        auditEntry,
        StandardOpenOption.CREATE,
        StandardOpenOption.APPEND);
```

Граница описывает нужный эффект:

```java
interface PaymentAudit {
    void record(PaymentAuditEntry entry);
}
```

Детали файла — в продакшен-реализации:

```java
final class FilePaymentAudit implements PaymentAudit {
    private final Path path;

    FilePaymentAudit(Path path) {
        this.path = Objects.requireNonNull(path);
    }

    @Override
    public void record(PaymentAuditEntry entry) {
        try {
            Files.writeString(
                    path,
                    entry.toLine() + System.lineSeparator(),
                    StandardOpenOption.CREATE,
                    StandardOpenOption.APPEND);
        } catch (IOException exception) {
            throw new UncheckedIOException(exception);
        }
    }
}
```

Сервис больше не знает о путях, опциях, переводах строк и checked-исключениях:

```java
paymentAudit.record(new PaymentAuditEntry(
        order.id(), transactionId, clock.instant()));
```

Юнит-тест проверяет значимый эффект:

```java
verify(paymentAudit).record(expectedEntry);
```

А файловую реализацию проверяем отдельно во временной папке:

```java
@TempDir
Path tempDirectory;

@Test
void appendsAnAuditEntryToTheConfiguredFile() throws IOException {
    Path auditFile = tempDirectory.resolve("audit.log");
    FilePaymentAudit audit = new FilePaymentAudit(auditFile);

    audit.record(new PaymentAuditEntry(
            42L,
            "tx-900",
            Instant.parse("2026-07-15T10:00:00Z")));

    assertEquals(
            "42|tx-900|2026-07-15T10:00:00Z"
                    + System.lineSeparator(),
            Files.readString(auditFile));
}
```

Это сознательно интеграционный тест файловой границы. Бизнес-сервис остаётся изолированным от файлов.

> **Главное:** в тестах бизнес-логики I/O подменяем, а настоящий I/O-адаптер тестируем отдельно на его границе.

## 6. Сделайте время заменяемым

Время — вход, даже если его нет в параметрах.

**До: скрытое системное время**

```java
if (Instant.now().isAfter(order.paymentDeadline())) {
    return ProcessingResult.expired();
}
```

**После: внедрённые часы**

```java
final class OrderProcessingService {
    private final Clock clock;

    OrderProcessingService(Clock clock) {
        this.clock = Objects.requireNonNull(clock);
    }

    ProcessingResult process(Order order) {
        if (clock.instant().isAfter(order.paymentDeadline())) {
            return ProcessingResult.expired();
        }

        return ProcessingResult.ready();
    }
}
```

Продакшен:

```java
new OrderProcessingService(Clock.systemUTC());
```

Тест — настоящие предсказуемые часы, Mockito не нужен:

```java
Clock clock = Clock.fixed(
        Instant.parse("2026-07-15T10:00:00Z"),
        ZoneOffset.UTC);
```

### Не меняйте граничное правило незаметно

Эти выражения разные:

```java
now.isAfter(deadline)
```

```java
!now.isBefore(deadline)
```

Первое разрешает оплату ровно в дедлайн, второе — запрещает. Характеризационный тест точной границы защищает эту деталь, пока правило не подтверждено.

### Когда хватит более узкого источника

Если нужен только текущий момент без часовых поясов — `InstantSource`:

```java
final class TokenService {
    private final InstantSource instantSource;

    TokenService(InstantSource instantSource) {
        this.instantSource = instantSource;
    }

    boolean isExpired(Token token) {
        return instantSource.instant().isAfter(token.expiresAt());
    }
}
```

Не заводите свой `TimeProvider`, если в JDK уже есть подходящая абстракция.

## 7. Разбивайте длинные методы по обязанностям

Вынос строк в методы улучшает читаемость, но тестируемость — только если выделяет осмысленные обязанности или отделяет решения от эффектов.

```java
HttpResponse process(long orderId) {
    Order order = repository.findById(orderId)
            .orElseThrow();

    BigDecimal fee = order.total()
            .multiply(new BigDecimal("0.03"));
    BigDecimal amount = order.total().add(fee);

    if (clock.instant().isAfter(order.paymentDeadline())) {
        return HttpResponse.conflict("Payment deadline expired");
    }

    PaymentResult payment = paymentGateway.charge(
            new PaymentRequest(order.id(), amount));

    if (!payment.approved()) {
        order.rejectPayment(payment.message());
        repository.save(order);
        return HttpResponse.paymentRequired(payment.message());
    }

    order.markPaid();
    repository.save(order);
    paymentAudit.record(PaymentAuditEntry.from(order, payment));
    return HttpResponse.ok(order.id());
}
```

Обязанностей минимум четыре: загрузка заказа, расчёт суммы, решение о допуске оплаты, координация оплаты/сохранения/аудита/ответа.

### Шаг 1: дать решениям имена

```java
private boolean deadlineExpired(Order order) {
    return clock.instant().isAfter(order.paymentDeadline());
}

private BigDecimal amountToCharge(Order order) {
    BigDecimal fee = order.total()
            .multiply(new BigDecimal("0.03"));
    return order.total().add(fee);
}
```

Читать легче, но private-методы — часть того же юнита. Не делайте их public ради тестов.

### Шаг 2: вынести цельный расчёт

Если у расчёта свои правила и причины меняться — отдельный доменный компонент:

```java
final class ChargeCalculator {
    private static final BigDecimal FEE_RATE =
            new BigDecimal("0.03");

    BigDecimal amountFor(Order order) {
        BigDecimal fee = order.total().multiply(FEE_RATE);
        return order.total().add(fee);
    }
}
```

Тест на реальных значениях, без моков:

```java
@Test
void addsThreePercentProcessingFee() {
    ChargeCalculator calculator = new ChargeCalculator();
    Order order = orderWithTotal("100.00");

    BigDecimal amount = calculator.amountFor(order);

    assertEquals(new BigDecimal("103.0000"), amount);
}
```

### Отделите решения от эффектов

Доменные решения — в коде, работающем со значениями; эффекты — в тонком слое оркестрации:

```java
final class PaymentRules {

    boolean paymentAllowed(Order order, Instant now) {
        return !now.isAfter(order.paymentDeadline());
    }

    PaymentDecision decisionFor(PaymentResult payment) {
        return payment.approved()
                ? PaymentDecision.approved(
                        payment.transactionId())
                : PaymentDecision.rejected(
                        payment.message());
    }
}
```

```java
Order order = repository.findById(orderId).orElseThrow();
if (!paymentRules.paymentAllowed(order, clock.instant())) {
    return ProcessingResult.expired();
}

PaymentResult payment = paymentGateway.charge(requestFor(order));
PaymentDecision decision = paymentRules.decisionFor(payment);
apply(decision, order);
```

Точная структура не обязательна. Идея: найти расчёты и решения на явных значениях, а координацию I/O оставить на краях.

### Чего избегать

Десятки однострочных методов без смысла; класс на каждое выражение; public-методы только для тестов; перенос кода без уменьшения скрытых зависимостей; интерфейс для предсказуемого расчёта; тесты каждого хелпера вместо наблюдаемого поведения.

> **Главное:** хорошая декомпозиция следует обязанностям и причинам изменений, а не целевому числу строк.

## 8. Выносите доменную логику из контроллеров

Контроллер — на границе приложения. Его дело: принять и проверить входные данные транспорта, перевести их в запрос приложения, вызвать сервис, перевести результат в HTTP-ответ. Ценообразование, право на скидку, оплата и статусы заказов — не его дело.

### Контроллер с доменной логикой

```java
final class DiscountController {
    private final CustomerRepository customerRepository;

    HttpResponse calculate(DiscountRequest request) {
        Customer customer = customerRepository
                .findById(request.customerId())
                .orElseThrow();

        BigDecimal discount;
        if (customer.loyaltyYears() >= 5
                && request.subtotal()
                        .compareTo(new BigDecimal("100")) >= 0) {
            discount = request.subtotal()
                    .multiply(new BigDecimal("0.10"));
        } else {
            discount = BigDecimal.ZERO;
        }

        return HttpResponse.ok(discount);
    }
}
```

Чтобы проверить правило скидки, нужен контроллер, репозиторий и маппинг ответа.

### После выноса

```java
final class DiscountPolicy {

    BigDecimal discountFor(
            int loyaltyYears,
            BigDecimal subtotal) {

        boolean eligible = loyaltyYears >= 5
                && subtotal.compareTo(
                        new BigDecimal("100")) >= 0;

        return eligible
                ? subtotal.multiply(new BigDecimal("0.10"))
                : BigDecimal.ZERO;
    }
}
```

```java
final class DiscountService {
    private final CustomerRepository customerRepository;
    private final DiscountPolicy discountPolicy;

    DiscountService(
            CustomerRepository customerRepository,
            DiscountPolicy discountPolicy) {
        this.customerRepository = customerRepository;
        this.discountPolicy = discountPolicy;
    }

    BigDecimal calculate(long customerId, BigDecimal subtotal) {
        Customer customer = customerRepository
                .findById(customerId)
                .orElseThrow();

        return discountPolicy.discountFor(
                customer.loyaltyYears(), subtotal);
    }
}
```

```java
final class DiscountController {
    private final DiscountService discountService;

    HttpResponse calculate(DiscountRequest request) {
        BigDecimal discount = discountService.calculate(
                request.customerId(), request.subtotal());

        return HttpResponse.ok(discount);
    }
}
```

| Компонент | Поведение | Как тестировать |
|---|---|---|
| `DiscountPolicy` | Доменное правило | Реальные значения, проверка результата |
| `DiscountService` | Загрузить клиента и вызвать расчёт | Стаб репозитория, настоящая политика |
| `DiscountController` | Маппинг запроса/ответа | Узкий тест веб-слоя |
| Реализация репозитория | Маппинг хранения | Интеграционный тест с выбранной БД |

Контроллер не обязан быть в одну строку: валидация транспорта и маппинг ответа — его. Важна граница между транспортом и доменными решениями.

## 9. Фейки для медленных и неудобных ресурсов

**Фейк** — упрощённая рабочая реализация. В отличие от стаба под один тест, это маленькая переиспользуемая модель помощника.

### Репозиторий в памяти

```java
final class InMemoryOrderRepository implements OrderRepository {
    private final Map<Long, Order> orders = new HashMap<>();

    @Override
    public Optional<Order> findById(long orderId) {
        return Optional.ofNullable(orders.get(orderId));
    }

    @Override
    public void save(Order order) {
        orders.put(order.id(), order);
    }

    void add(Order order) {
        orders.put(order.id(), order);
    }
}
```

Тест без повторного стабинга каждого вызова:

```java
@Test
void savesAnApprovedOrder() {
    InMemoryOrderRepository repository =
            new InMemoryOrderRepository();
    Order order = orderWithId(42L);
    repository.add(order);

    OrderProcessingService service = new OrderProcessingService(
            repository,
            approvingGateway(),
            new InMemoryPaymentAudit(),
            fixedClock());

    service.process(42L);

    Order saved = repository.findById(42L).orElseThrow();
    assertEquals(OrderStatus.PAID, saved.status());
}
```

### Фейк или Mockito?

| Фейк, если… | Стаб/мок Mockito, если… |
|---|---|
| многим тестам нужны реалистичные переходы состояния | нужны один-два заготовленных ответа |
| повторный стабинг заслоняет сценарий | точный вызов — часть поведения |
| модель маленькая и стабильная | правильную модель строить дорого |
| несколько операций должны работать вместе | нужно напрямую получить ошибку или редкий ответ |
| удобно запрашивать сохранённое состояние | важен только один вызов |

Фейк — это тоже код, который нужно поддерживать.

### Проблема точности

Map не ведёт себя как реляционная база: транзакции, ограничения и уникальность, семантика запросов, генерация ID, порядок, конкурентность, маппинг, null и специфичные типы. Поэтому фейк не доказывает, что JDBC/JPA-реализация работает — нужны интеграционные тесты настоящего хранилища.

Встроенная или in-memory **база** — не фейк: если запускается настоящий движок и настоящая реализация хранения, это интеграционный тест с быстрым ресурсом.

### Готовые фейки

`Clock.fixed(...)` и `InstantSource.fixed(...)` — упрощённое предсказуемое время, часто лучше мока. Похожие маленькие фейки: очередь сообщений как список в памяти, файловое хранилище как map «путь → содержимое», источник ID с предсказуемой последовательностью, отправщик уведомлений, который запоминает сообщения.

Не стройте большую имитацию внешней платформы. Если фейк начинает повторять сложное поведение провайдера — надёжнее песочница, контрактный или интеграционный тест.

## 10. Безопасная последовательность рефакторинга

1. **Выберите одно поведение**, связанное с нужным изменением (оплата отклоняется после дедлайна; неудачная оплата сохраняется с нужным статусом; аудит пишется только после одобрения). Не «отрефакторить весь сервис».
2. **Зафиксируйте его** самым узким надёжным тестом на текущей границе: вывод, состояние, исключение, эффект. Если нужна реальная инфраструктура — изолируйте данные и окружение.
3. **Найдите блокирующую зависимость**: static SDK, создание репозитория, `Instant.now()`, прямой доступ к файлу, доменная логика в контроллере.
4. **Введите один шов**: минимальная обёртка, адаптер, параметр или внедрённая зависимость. Поведение не меняется.
5. **Перенесите одну обязанность**: цельный расчёт, политику или внешний эффект. Не смешивайте с посторонней чисткой и переименованиями.
6. **Добавьте узкие тесты** на новой границе: доменная логика — обычные значения; оркестрация — выбранные дублёры; настоящий адаптер — узкий интеграционный тест.
7. **Сравните поведение**: прогоните характеризационный и новые тесты. Широкий тест подтверждает стабильность, узкие точнее описывают новые юниты.
8. **Продолжайте маленькими шагами**: следующую зависимость делайте явной, только когда это даёт ценный тест или улучшение. Маленькие рабочие коммиты упрощают поиск регрессий.

> **Характеризуй → создай шов → вынеси одну обязанность → добавь узкие тесты → проверь, что поведение не изменилось.**

## 11. Пример: улучшаем связанный платёжный поток

Возвращаемся к `OrderController` и не переделываем всё сразу.

**Этап 1: защищаем поведение дедлайна.** Существующий тест верхнего уровня подтверждает: ровно в дедлайн оплата разрешена, через наносекунду — нет. Это временная страховка.

**Этап 2: заменяем скрытое время.** Контроллер получает `Clock`:

```java
final class OrderController {
    private final Clock clock;

    OrderController(Clock clock) {
        this.clock = clock;
    }

    // Instant.now() becomes clock.instant()
}
```

Граничные случаи стали предсказуемыми, хотя база и платежи ещё связаны.

**Этап 3: оборачиваем платёжного провайдера.** Создаём `PaymentGateway` и `ProviderPaymentGateway`, внедряем шлюз в контроллер. Static-вызов уезжает в адаптер. Теперь тесты воспроизводят одобрение, отказ и сбой провайдера без реальной оплаты.

**Этап 4: выделяем сервис приложения.** Загрузку заказа, проверку дедлайна, оплату, смену состояния и сохранение переносим в `OrderProcessingService`. В контроллере остаётся маппинг HTTP:

```java
final class OrderController {
    private final OrderProcessingService orderProcessingService;

    HttpResponse process(long orderId) {
        ProcessingResult result =
                orderProcessingService.process(orderId);

        return switch (result.status()) {
            case PAID -> HttpResponse.ok(result.orderId());
            case EXPIRED -> HttpResponse.conflict(result.message());
            case PAYMENT_REJECTED ->
                    HttpResponse.paymentRequired(result.message());
        };
    }
}
```

Тест контроллера — про маппинг, тест сервиса — про обработку заказа.

**Этап 5: изолируем I/O аудита.** `Files.writeString(...)` уезжает в `FilePaymentAudit` за интерфейсом `PaymentAudit`. В тестах сервиса — мок или записывающий фейк, для адаптера — временная папка.

**Этап 6: пересматриваем остаток.** Если расчёт комиссии растёт или меняется отдельно — выносим `ChargeCalculator`. Если это одна стабильная формула — выносить пока незачем.

Результат измеряется не числом классов, а тем, что: время управляемо; платежи и хранение заменяемы; файловый I/O изолирован; доменное поведение вне контроллера; каждый тест использует самый простой подходящий объект или дублёр; исходное видимое поведение защищено.

## 12. Моки, покрытие и уверенность

Швы позволяют выполнить больше веток в изолированных тестах — стаб шлюза даёт одобрение, отказ или исключение. Покрытие растёт.

Но покрытие показывает **выполнение**, а не качество проверки. Этот тест выполнит много кода, но почти ничего не докажет:

```java
when(paymentGateway.charge(any()))
        .thenReturn(PaymentResult.approved("tx-1"));

service.process(42L);
```

Нужно проверить результат или значимый эффект:

```java
ProcessingResult result = service.process(42L);

assertEquals(ProcessingStatus.PAID, result.status());
verify(repository).save(argThat(
        order -> order.status() == OrderStatus.PAID));
```

Больше веток — обычно больше сценариев, но цикломатическая сложность не скажет точное число тестов и нужные ассерты. Покрытие ищет невыполненное, а тесты строятся от требований, границ, рисков и наблюдаемого поведения.

Избыток моков тоже обманывает: тест докажет порядок вызовов, но пропустит неверный маппинг базы, контракт провайдера или формат файла. Для реальных границ нужны интеграционные и контрактные тесты.

## 13. Частые ошибки

- **Переписать всё без защиты.** Структура и поведение меняются одновременно — непонятно, что сломало. Сначала страховка, потом маленькие шаги.
- **Считать характеризацию одобрением.** Старое поведение может быть багом. Фиксируйте с пояснением и уточняйте правило у заказчика или в спецификации.
- **Интерфейс без точки внедрения** — заменяемости нет.
- **Оборачивать каждую static-функцию.** `Math.min(...)` не нужно. Важны скрытое состояние, внешние эффекты, непредсказуемость, привязка к провайдеру.
- **Широкая общая обёртка.** `ExternalService` с кучей несвязанных методов прячет обязанности. Нужны маленькие роли.
- **Вынос только в private-хелперы.** Логика и зависимости остаются в том же классе. Выносите цельный объект, если это осмысленная обязанность.
- **Делать private-методы public для тестов.** Меняйте видимость по причинам дизайна, а не чтобы дотянуться до детали.
- **Считать фейк равным продакшену.** Map не проверит SQL, транзакции, ORM, ограничения.
- **Мокать вынесенную доменную логику.** Предсказуемые `DiscountPolicy` или `ChargeCalculator` берите настоящими — иначе спрячете то, что надо проверить.
- **Убрать все широкие тесты.** Узкие тесты упрощают диагностику, но не доказывают совместную работу адаптеров, конфигурации, сериализации и контрактов.
- **Рефакторить и менять фичу одновременно.** Сначала сохраните поведение, улучшая структуру, потом добавляйте новое с узкими тестами.

## 14. Порядок решений

1. **Какое наблюдаемое поведение нужно защитить?**
2. **Можно ли проверить его через существующую публичную границу?**
3. **Какая скрытая зависимость мешает контролю или изоляции?**
4. **Решит ли проблему стандартная абстракция вроде `Clock`?**
5. **Даст ли узкая обёртка или адаптер осмысленную границу?**
6. **Есть ли в длинном методе расчёт или политика, работающие на явных значениях?**
7. **Смешана ли доменная логика с контроллером или хранением?**
8. **Сделает ли маленький фейк сценарии понятнее повторного стабинга?**
9. **Какой реальной границе всё ещё нужен интеграционный или контрактный тест?**
10. **Какое минимальное изменение может самостоятельно компилироваться и проходить тесты?**

## 15. Практика: спланируйте безопасное улучшение

Рассмотрите код:

```java
final class RegistrationController {

    HttpResponse register(RegistrationRequest request)
            throws IOException {

        JdbcUserRepository repository =
                new JdbcUserRepository(Database.connect());

        if (repository.existsByEmail(request.email())) {
            return HttpResponse.conflict("Email already registered");
        }

        if (LocalDate.now().isAfter(request.invitationExpiresOn())) {
            return HttpResponse.badRequest("Invitation expired");
        }

        String userId = UUID.randomUUID().toString();
        User user = new User(
                userId,
                request.email(),
                UserStatus.ACTIVE);

        repository.save(user);
        EmailApi.sendWelcomeMessage(user.email(), user.id());
        Files.writeString(
                Path.of("registrations.log"),
                user.id() + System.lineSeparator(),
                StandardOpenOption.CREATE,
                StandardOpenOption.APPEND);

        return HttpResponse.created(user.id());
    }
}
```

Ответьте на вопросы:

1. Какие зависимости, входы и эффекты скрыты?
2. Какое наблюдаемое поведение вы зафиксируете первым?
3. Какой шов вы введёте первым, если следующее изменение касается просроченных приглашений?
4. Какой шов вы введёте первым, если следующее изменение касается ошибок провайдера при отправке письма?
5. Какую доменную логику или логику приложения стоит вынести из контроллера?
6. Где может пригодиться фейк?
7. Какие тесты всё равно должны использовать реальную инфраструктуру?
8. Предложите безопасную последовательность не более чем из шести шагов.

### Предлагаемые ответы

1. Скрыты: подключение к базе и создание репозитория, системная дата, генерация случайного UUID, static-API почты и прямая запись в файл. Сохранение пользователя, отправка письма и запись в лог — внешне видимые эффекты.
2. Полезные случаи: дубликат email, просроченное приглашение, успешная регистрация, точная граница даты истечения и, возможно, поведение при сбое отправки письма или записи в файл. Выбирайте по риску и планируемому изменению, а не пытайтесь сразу покрыть каждую строку.
3. Ввести `Clock` и заменить `LocalDate.now()` на `LocalDate.now(clock)`. Добавить тесты для дня до, точной даты истечения и дня после. Сохранять текущую включающую или исключающую границу, пока правило не подтверждено.
4. Ввести узкий адаптер `WelcomeMessageSender` вокруг `EmailApi`. Стаб может бросить нужное исключение уровня приложения для сценария сбоя, не обращаясь к провайдеру.
5. Проверка дубликатов, допустимость приглашения, назначение ID, создание пользователя, сохранение и координация уведомления — в сервис приложения. Правило истечения может стать маленькой доменной политикой, если у него есть своя сложность. Контроллер в основном маппит запрос и результат сервиса.
6. `InMemoryUserRepository` на основе map для нескольких сценариев регистрации. Предсказуемый источник ID, возвращающий известные идентификаторы. Записывающий отправщик сообщений, сохраняющий отправленные письма для проверок.
7. JDBC-репозиторию нужны интеграционные тесты с выбранной технологией БД. Почтовому адаптеру — песочница провайдера, контрактный тест или другая проверка на уровне границы. Файловой реализации аудита — узкий тест во временной папке.
8. Одна разумная последовательность:
   1. зафиксировать поведение просрочки и успеха на доступной публичной границе;
   2. внедрить `Clock` и добавить предсказуемые тесты границы даты;
   3. обернуть static-API почты и протестировать обработку сбоя провайдера;
   4. выделить `RegistrationService` из контроллера;
   5. внедрить в сервис репозиторий и источник ID;
   6. вынести запись в файл за границу аудита регистраций и добавить узкие тесты адаптера.

Возможны и другие последовательности. Лучший первый шов зависит от риска и вносимого изменения.

### Возможный целевой дизайн

```java
final class RegistrationService {
    private final UserRepository userRepository;
    private final WelcomeMessageSender messageSender;
    private final RegistrationAudit registrationAudit;
    private final Supplier<String> userIdSource;
    private final Clock clock;

    RegistrationService(
            UserRepository userRepository,
            WelcomeMessageSender messageSender,
            RegistrationAudit registrationAudit,
            Supplier<String> userIdSource,
            Clock clock) {
        this.userRepository = userRepository;
        this.messageSender = messageSender;
        this.registrationAudit = registrationAudit;
        this.userIdSource = userIdSource;
        this.clock = clock;
    }

    RegistrationResult register(RegistrationCommand command) {
        if (userRepository.existsByEmail(command.email())) {
            return RegistrationResult.duplicateEmail();
        }

        LocalDate today = LocalDate.now(clock);
        if (today.isAfter(command.invitationExpiresOn())) {
            return RegistrationResult.expiredInvitation();
        }

        User user = new User(
                userIdSource.get(),
                command.email(),
                UserStatus.ACTIVE);

        userRepository.save(user);
        messageSender.sendWelcomeMessage(
                user.email(), user.id());
        registrationAudit.record(user.id());

        return RegistrationResult.created(user.id());
    }
}
```

Зависимости явные, но не обязательно все — моки Mockito. Тест может использовать:

- фейковый репозиторий в памяти;
- записывающий фейковый отправщик сообщений;
- стаб аудита, который ничего не делает;
- лямбду вроде `() -> "user-42"` как источник ID;
- настоящие фиксированные часы `Clock`.

## 16. Проверь себя

### Вопросы

1. Какова главная цель характеризационного теста?
2. Доказывает ли характеризационный тест, что текущее поведение правильное?
3. Почему первый страховочный тест может быть интеграционным, а не юнит-тестом?
4. Что такое шов в контексте тестирования?
5. Когда полезна обёртка вокруг static-вызова?
6. Нужно ли обычно оборачивать `Math.max(...)` для юнит-тестов?
7. Почему `Clock.fixed(...)` часто лучше, чем мок времени?
8. Создаёт ли вынос private-метода автоматически отдельно тестируемый юнит?
9. Почему доменную логику обычно выносят из контроллера?
10. Когда фейковый репозиторий может быть понятнее стабинга Mockito?
11. Какой важный риск несёт фейковый репозиторий в памяти?
12. Доказывает ли высокое покрытие веток, что в тестах есть полезные проверки?
13. Почему настоящему адаптеру всё равно нужны узкие интеграционные или контрактные тесты?
14. Какова безопасная общая последовательность улучшения сильно связанного кода?

### Ответы

1. Зафиксировать важное наблюдаемое поведение, чтобы заметить случайные изменения при рефакторинге незнакомого или легаси-кода.
2. Нет. Он фиксирует, что система делает сейчас. Это поведение может потребовать подтверждения, исправления или замены тестом по спецификации.
3. В текущем коде может ещё не быть шва, позволяющего изолировать поведение. Более широкий надёжный тест даёт защиту, пока не выделены меньшие юниты.
4. Место, где зависимость или поведение можно заменить или выбрать, чтобы тест управлял трудным помощником.
5. Когда static-вызов пересекает внешнюю границу, читает меняющееся состояние, вызывает эффект или распространяет типы провайдера по коду приложения.
6. Нет. Это предсказуемая чистая утилита. Обёртка добавит лишний слой без полезного контроля.
7. Это стандартная предсказуемая реализация, созданная для таких случаев: даёт точное время без настройки поведения методов мока.
8. Нет. Private-метод остаётся деталью реализации того же класса. Тестируемость растёт, когда вынос создаёт осмысленную обязанность или отделяет чистые решения от эффектов.
9. Доменные правила должны переиспользоваться и тестироваться независимо от HTTP-транспорта, маппинга запросов, репозиториев и построения ответов.
10. Когда многим тестам нужно реалистичное состояние репозитория, несколько операций работают вместе или повторный стабинг заслоняет сценарий.
11. Его поведение может отличаться от продакшен-базы в транзакциях, ограничениях, запросах, конкурентности или маппинге.
12. Нет. Покрытие показывает, какой код выполнился, а не проверил ли тест правильный результат или значимый эффект.
13. Юнит-тесты с дублёрами не докажут, что работают сериализация, конфигурация, контракты провайдера, SQL, поведение файлов и другие детали реальной границы.
14. Зафиксировать одно важное поведение, ввести один шов, вынести одну обязанность, добавить узкие тесты, прогнать страховочные тесты и продолжать маленькими рабочими шагами.

## Шпаргалка

- Связанный код улучшают маленькими защищёнными шагами, а не непроверенным переписыванием.
- Характеризационный тест фиксирует текущее поведение, но не доказывает его правильность.
- Первый страховочный тест может быть шире юнит-теста.
- Шов — место, где продакшен и тест выбирают разные реализации или входы.
- Обёртки/адаптеры — вокруг внешних static-API, SDK, I/O. Предсказуемые утилиты не оборачиваем.
- Время — через `Clock` / `InstantSource`.
- Длинные методы делим по обязанностям, отделяя решения от эффектов.
- Контроллер — транспорт; доменные правила и координация — в отдельных компонентах.
- Фейки снижают повторный стабинг, но не заменяют тесты реальной инфраструктуры.
- Покрытие показывает выполнение, а не качество проверок.
- Характеризуй → шов → вынеси одну обязанность → узкие тесты → проверь поведение.

## Почитать

- Michael Feathers, *Working Effectively with Legacy Code* — характеризационные тесты, швы и безопасное изменение кода без тестов
- [Extract Function — Refactoring](https://refactoring.com/catalog/extractFunction.html)
- [Extract Class — Refactoring](https://refactoring.com/catalog/extractClass.html)
- [`Clock` — Java SE 25 API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/time/Clock.html)
- [`InstantSource` — Java SE 25 API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/time/InstantSource.html)
- [When to Mock — Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2014/05/10/WhenToMock.html)
