# Writing Quality Unit Tests: FIRST and Arrange–Act–Assert

## Learning objectives

By the end of this topic, you will be able to:

- explain the meaning of **Fast, Independent, Repeatable, Self-Validating, and Timely**;
- structure a unit test using **Arrange–Act–Assert**;
- distinguish independence from repeatability;
- explain why deterministic tests control time, randomness, and I/O boundaries;
- recognize common causes of slow, flaky, or difficult-to-maintain tests;
- identify violations of FIRST and AAA in simple test examples;
- improve a test without weakening the behavior it verifies.

---

## 1. Passing tests are not automatically good tests

A test can pass and still be a poor test.

Consider this example:

```java
@Test
void testOrder() throws Exception {
    Order order = new Order();
    order.add("BOOK", 2);
    System.out.println(order.total());
    Thread.sleep(1_000);
}
```

The method may be discovered and reported as successful by JUnit, but it has serious problems:

- it does not assert an expected result;
- a person must inspect the console output;
- it waits for one second without a behavioral reason;
- its name does not describe the expected behavior;
- it gives little protection against a regression.

Now consider another test:

```java
@Test
void shouldCalculateTotalForTwoEqualItems() {
    Order order = new Order();
    order.add("BOOK", 2, 1_500);

    int actualTotal = order.total();

    assertEquals(3_000, actualTotal);
}
```

This version states a scenario, executes one relevant behavior, and verifies the result automatically. If the calculation changes incorrectly, the test reports the expected and actual values.

Correct assertions are essential, but test quality is broader than assertion syntax. A useful test suite must also remain:

- quick enough to run frequently;
- isolated enough that tests do not contaminate one another;
- stable across repeated runs;
- clear enough that a failure can be understood;
- maintainable as production code evolves.

The **FIRST** principles and the **Arrange–Act–Assert** pattern provide two complementary ways to reason about these qualities.

- FIRST describes properties of useful tests and test suites.
- Arrange–Act–Assert describes the internal flow of one test scenario.

They are guidelines rather than a mechanical guarantee of correctness. A test can follow AAA and still assert the wrong requirement. A fast test can still test a private implementation detail. Test design always requires judgment about behavior.

---

## 2. The FIRST principles

FIRST is an acronym for five desirable qualities of automated tests:

| Letter | Principle | Main question |
| --- | --- | --- |
| F | Fast | Can developers run the tests frequently and receive quick feedback? |
| I | Independent | Can each test run alone and in any order without affecting another test? |
| R | Repeatable | Do the same controlled conditions produce the same result? |
| S | Self-Validating | Does the test decide pass or failure automatically? |
| T | Timely | Is the test written close enough to the production behavior to guide and protect development? |

The principles reinforce one another. For example, a test that uses a real remote service is often both slow and non-repeatable. A test that depends on a previous test is neither independent nor reliably repeatable.

However, the words are not synonyms. Understanding their differences helps diagnose the actual cause of a bad test.

---

## 3. Fast

A unit-test suite should provide feedback quickly enough to be run during normal development.

Fast feedback changes developer behavior. If a focused suite completes quickly, a developer can run it:

- after a small production-code change;
- before committing code;
- while refactoring;
- while investigating a failure;
- repeatedly during debugging.

If every run takes many minutes, developers naturally run it less often. Defects are then discovered later, after more changes have accumulated, and become harder to locate.

### 3.1 What usually makes a unit test slow

Common causes include:

- starting the complete application when a plain object would be enough;
- connecting to a real database;
- calling a network service;
- reading or writing large files;
- using `Thread.sleep()` to wait for an event;
- performing an unnecessarily large calculation;
- recreating expensive infrastructure for many small scenarios.

For example, this is not a focused unit test of a price calculation:

```java
@Test
void shouldCalculateOrderTotal() {
    OrderRepository repository = new SqlOrderRepository(realDataSource());
    Order order = repository.findById(1001L);

    int actualTotal = order.total();

    assertEquals(3_000, actualTotal);
}
```

The result can be influenced by the database connection, schema, stored record, transaction state, and calculation logic. This may be a useful integration test if database integration is intentionally the subject, but it is not a fast, focused unit test of `Order.total()`.

A unit test can create the relevant object directly:

```java
@Test
void shouldCalculateOrderTotal() {
    Order order = new Order();
    order.add("BOOK", 2, 1_500);

    int actualTotal = order.total();

    assertEquals(3_000, actualTotal);
}
```

### 3.2 Do not optimize test speed blindly

“Fast” does not mean that every test must finish within an arbitrary number of milliseconds. The useful threshold depends on the project, behavior, machine, and test level.

Avoid these mistakes:

- deleting valuable scenarios merely to reduce execution time;
- replacing a realistic integration test with a unit test when integration is what must be verified;
- sharing mutable fixtures to avoid cheap object creation;
- adding a very tight timeout that fails on a slower machine;
- treating a single local timing measurement as a permanent performance contract.

The right response to a slow suite is to identify why it is slow and whether each test is at the correct level.

### 3.3 Timeouts are not a general speed test

JUnit provides timeout mechanisms such as `@Timeout`, `assertTimeout()`, and `assertTimeoutPreemptively()`. They can be useful when termination within a limit is part of the behavior or when a test needs protection from hanging indefinitely.

They should not be added mechanically to every unit test. A timing assertion can be sensitive to machine load and environment. In addition, preemptive timeouts execute the supplied code in another thread, which can interact unexpectedly with thread-local state.

Use a timeout to express a meaningful upper bound, not to prove that a test suite is well designed.

---

## 4. Independent

An independent test does not require another test to run before it, and it does not leave state that changes another test's result.

Each test should be usable as a separate diagnostic example. It should produce the same result when run:

- alone;
- with its class;
- with the complete suite;
- before or after another test;
- in a different valid execution order.

### 4.1 An order-dependent test

```java
class ShoppingCartTest {

    private static final ShoppingCart CART = new ShoppingCart();

    @Test
    void shouldAddBook() {
        CART.add("BOOK", 1);

        assertEquals(1, CART.itemCount());
    }

    @Test
    void shouldAddPen() {
        CART.add("PEN", 1);

        assertEquals(2, CART.itemCount());
    }
}
```

The second test assumes that the first test already added one item. Run alone, it fails. Run after the first test, it may pass. If the first test fails before adding the book, the second failure becomes misleading.

The correct design gives every test its own starting state:

```java
class ShoppingCartTest {

    private ShoppingCart cart;

    @BeforeEach
    void setUp() {
        cart = new ShoppingCart();
    }

    @Test
    void shouldAddBook() {
        cart.add("BOOK", 1);

        assertEquals(1, cart.itemCount());
    }

    @Test
    void shouldAddPen() {
        cart.add("PEN", 1);

        assertEquals(1, cart.itemCount());
    }
}
```

Both tests now express complete scenarios.

### 4.2 Independence is broader than instance fields

The default `PER_METHOD` lifecycle gives each test method a separate test-class instance, but it does not isolate everything. Tests can still interfere through:

- static mutable fields;
- singleton objects and caches;
- system properties;
- environment-dependent configuration;
- shared files or directories;
- shared database records;
- fixed ports;
- common user accounts in external systems;
- global locale or time-zone changes.

For example, this test changes process-wide state:

```java
@Test
void shouldReadConfiguredRegion() {
    System.setProperty("app.region", "EU");

    assertEquals("EU", RegionConfig.currentRegion());
}
```

Unless the original property is restored reliably, another test can observe `EU`. Even with cleanup, parallel execution may make process-wide mutation unsafe because another test can read the property before it is restored.

The preferred design is usually to pass configuration explicitly to the unit rather than make unit behavior depend on global process state.

### 4.3 Do not repair dependence by enforcing order

Adding a test execution order can hide the symptom without removing the dependence. The suite remains fragile, and individual tests still cannot run reliably.

Execution order can be valid when order itself is intentionally being tested at a broader level, but normal unit tests should not use it as shared setup.

### 4.4 Independence and parallel execution

JUnit Jupiter runs tests sequentially by default, while parallel execution can be enabled. A suite that assumes sequential execution may begin to fail when parallelism is introduced.

Parallel execution is not required to reveal a design defect. If two tests can influence each other through shared mutable state, they are not independent even when their current order happens to make them pass.

---

## 5. Repeatable

A repeatable test produces the same outcome whenever the tested behavior and controlled inputs are unchanged.

Repeatability means more than “run the method twice.” The relevant conditions include:

- input data;
- time and time zone;
- random values;
- locale and character encoding;
- dependency behavior;
- files and database state;
- network availability;
- execution environment.

A non-repeatable test is often called **flaky** when it sometimes passes and sometimes fails without a relevant code change.

### 5.1 Why flaky tests are dangerous

A flaky test creates two opposite risks:

1. A real failure is dismissed as “probably flaky.”
2. Developers lose time rerunning and investigating failures unrelated to their changes.

Repeatedly rerunning a failing test until it becomes green does not fix the problem. It hides evidence that the test or production behavior is uncontrolled.

The right response is to find the source of variation.

### 5.2 Independence versus repeatability

These principles are related but answer different questions:

- **Independent:** can other tests change this test's result?
- **Repeatable:** can uncontrolled conditions change this test's result?

A test can violate repeatability without depending on another test:

```java
@Test
void shouldCreateSummerCampaign() {
    Campaign campaign = Campaign.createForCurrentDate();

    assertEquals("SUMMER", campaign.season());
}
```

It may pass in July and fail in January even when run alone. The problem is current time, not test order.

A test can also be repeatable when run alone but not independent: a test using a shared list may always pass alone and fail only after another test modifies the list.

---

## 6. Controlling time

Production code often needs the current date or time. A unit test should not normally let the wall clock choose the scenario.

### 6.1 The unstable version

```java
public final class TrialService {

    public boolean isExpired(LocalDate expirationDate) {
        return expirationDate.isBefore(LocalDate.now());
    }
}
```

A test that uses dates near “today” can change result as the calendar advances. It can also behave differently across time zones if date and instant conversions are involved.

### 6.2 Inject a clock

Java's `Clock` allows production code to obtain the current time from an explicit dependency:

```java
import java.time.Clock;
import java.time.LocalDate;
import java.util.Objects;

public final class TrialService {

    private final Clock clock;

    public TrialService(Clock clock) {
        this.clock = Objects.requireNonNull(clock);
    }

    public boolean isExpired(LocalDate expirationDate) {
        LocalDate today = LocalDate.now(clock);
        return expirationDate.isBefore(today);
    }
}
```

The test supplies a fixed clock:

```java
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;
import java.time.LocalDate;
import org.junit.jupiter.api.Test;

class TrialServiceTest {

    @Test
    void shouldReportTrialAsExpiredAfterExpirationDate() {
        Clock clock = Clock.fixed(
                Instant.parse("2026-07-21T12:00:00Z"),
                ZoneOffset.UTC
        );
        TrialService service = new TrialService(clock);

        boolean expired = service.isExpired(LocalDate.of(2026, 7, 20));

        assertTrue(expired);
    }
}
```

The test now defines what “today” means. It will describe the same scenario next week and on another machine.

In production, the application can supply `Clock.systemUTC()` or another clock appropriate to the domain. The test does not need to change the global system clock.

### 6.3 Use explicit domain values when possible

Sometimes the simplest design is to pass the relevant date directly:

```java
boolean isExpired(LocalDate expirationDate, LocalDate today)
```

Whether an explicit parameter or a `Clock` dependency is clearer depends on the domain. The essential principle is that the test controls the time-sensitive input.

---

## 7. Controlling randomness

Randomness creates different values across runs. That is useful in production behavior such as selecting a promotion or generating a sample, but an ordinary example-based unit test needs known input and a known expected result.

### 7.1 An unstable random test

```java
@Test
void shouldGenerateSmallDiscount() {
    int discount = new DiscountGenerator().nextDiscount();

    assertEquals(5, discount);
}
```

If the generator can return `5`, `10`, or `15`, this test passes only for one random outcome.

An assertion such as this is repeatable but weak:

```java
assertTrue(discount == 5 || discount == 10 || discount == 15);
```

It checks the allowed range, but it cannot verify a particular branch in code that consumes the generated value.

### 7.2 Separate the decision from the random source

A useful design makes the selected value controllable:

```java
import java.util.Objects;
import java.util.function.IntSupplier;

public final class PromotionService {

    private final IntSupplier discountSupplier;

    public PromotionService(IntSupplier discountSupplier) {
        this.discountSupplier = Objects.requireNonNull(discountSupplier);
    }

    public int finalPrice(int originalPrice) {
        int discountPercent = discountSupplier.getAsInt();
        return originalPrice * (100 - discountPercent) / 100;
    }
}
```

The test supplies a known value:

```java
@Test
void shouldApplySuppliedDiscount() {
    PromotionService service = new PromotionService(() -> 10);

    int finalPrice = service.finalPrice(2_000);

    assertEquals(1_800, finalPrice);
}
```

This example introduces the design idea of controlling a dependency. The next module will study test doubles, stubbing, and mocking systematically. At this stage, the important observation is that the unit test should not let chance choose which scenario it verifies.

### 7.3 What about a fixed random seed?

A fixed seed can reproduce a pseudo-random sequence and is useful in some algorithmic tests. It does not automatically make a test clear:

- the expected sequence may be difficult to understand;
- changing the random implementation may break tests unrelated to business behavior;
- a seed does not explain which domain case is important.

For a business-rule unit test, explicit values are usually clearer. Use a seed when the sequence itself or a randomized algorithm is relevant.

---

## 8. Controlling I/O boundaries

I/O means communication with something outside ordinary in-memory calculation, such as:

- a file system;
- a database;
- a remote HTTP service;
- a message broker;
- a console or device;
- another process.

Real I/O introduces additional failure causes: missing files, permissions, network delay, changed data, unavailable services, credentials, ports, and cleanup problems.

### 8.1 Match the test to the boundary

Suppose `ReportParser` transforms CSV text into report rows. A focused unit test can pass text directly:

```java
@Test
void shouldParseOneReportRow() {
    ReportParser parser = new ReportParser();
    String csv = "A-101,COMPLETED,2500";

    ReportRow row = parser.parse(csv);

    assertAll(
            () -> assertEquals("A-101", row.orderId()),
            () -> assertEquals(Status.COMPLETED, row.status()),
            () -> assertEquals(2_500, row.amount())
    );
}
```

This checks parsing behavior without depending on a particular path.

A separate integration test can verify that the file-reading component opens a file, uses the expected encoding, and passes its contents to the parser. Removing file I/O from the parser unit test does not mean that file integration should never be tested. It means that each test has a clear subject.

### 8.2 Avoid production data in unit tests

A test that expects an existing database record or a live service response can change without a code change. It may also expose or modify real data.

Unit tests should use controlled test data. Tests that intentionally verify a real database or protocol need an isolated test environment and belong to an appropriate integration-test layer.

### 8.3 Do not catch infrastructure failures and call the test successful

This test hides failures:

```java
@Test
void shouldLoadRates() {
    try {
        RateTable rates = client.loadRates();
        assertFalse(rates.isEmpty());
    } catch (IOException exception) {
        System.out.println("Service unavailable; ignoring test");
    }
}
```

When the call fails, the method reaches its end and JUnit reports success. The test is no longer self-validating.

If the dependency is required for the test, failure should remain visible. If availability is not the subject of a unit test, remove the real boundary from that unit test instead of swallowing the exception.

---

## 9. Self-Validating

A self-validating test determines success or failure automatically.

JUnit assertions make expectations executable:

```java
assertEquals(3_000, order.total());
```

The test runner reports a failure when the expectation is not met. No person needs to read a log, compare files by eye, or inspect a debugger variable.

### 9.1 Non-self-validating tests

These approaches require manual interpretation:

```java
System.out.println(order.total());
```

```java
Files.writeString(Path.of("actual-result.txt"), report);
```

```java
if (order.total() != 3_000) {
    System.out.println("Wrong total");
}
```

They may help temporarily while investigating code, but they do not form a reliable automated regression suite.

### 9.2 A test must verify the relevant behavior

Merely calling a method is not sufficient:

```java
@Test
void shouldAddProduct() {
    ShoppingCart cart = new ShoppingCart();

    cart.add("BOOK", 2);
}
```

The test passes as long as no unexpected exception escapes. It does not verify that an item was added or that the quantity is correct.

A relevant assertion makes the expectation explicit:

```java
assertEquals(2, cart.itemCount());
```

For commands that deliberately return no value, verify an observable effect through the public contract. Do not expose private state only to make an assertion possible.

### 9.3 Assertion messages should aid diagnosis

JUnit already reports expected and actual values for many assertions. Add a custom message when it contributes domain context:

```java
assertEquals(
        0,
        fee,
        "premium customers should receive free delivery"
);
```

Do not use a message as a substitute for a descriptive test name or a precise assertion.

---

## 10. Timely

In FIRST, **Timely** concerns when tests are created relative to production code. It does not mean that the test executes quickly; that is the Fast principle.

A test is timely when it is written close enough to the behavior it protects that it can support development. Depending on the workflow, a test may be written:

- before production code, as in test-driven development;
- together with production code;
- immediately after a defect is reproduced;
- during a refactoring, before changing existing behavior.

### 10.1 Why timing matters

When tests are postponed until long after implementation:

- details of the requirement may be forgotten;
- production code may become difficult to isolate;
- regressions can already be introduced;
- developers may test the current implementation rather than the intended behavior;
- writing tests becomes a separate cleanup project that is easy to defer again.

A regression test is timely when it captures a defect before or while the defect is fixed:

1. reproduce the incorrect behavior with a failing test;
2. correct the production code;
3. keep the test to protect against the same regression.

### 10.2 Timely does not require one development methodology

FIRST does not require every team to use strict test-driven development. The key idea is that tests participate in development rather than being treated as optional documentation added when the system is already finished.

---

## 11. Arrange–Act–Assert

Arrange–Act–Assert, or **AAA**, organizes one test scenario into three conceptual phases:

1. **Arrange:** prepare the object, input, and initial conditions.
2. **Act:** perform the behavior under test.
3. **Assert:** verify the observable result or effect.

```java
@Test
void shouldApplyPremiumDiscount() {
    // Arrange
    PriceCalculator calculator = new PriceCalculator();
    Customer customer = new Customer(CustomerType.PREMIUM);

    // Act
    int actualPrice = calculator.calculate(2_000, customer);

    // Assert
    assertEquals(1_800, actualPrice);
}
```

AAA was introduced when writing the first JUnit tests. Here, the focus is not remembering the three words but using the structure to diagnose unclear tests.

### 11.1 Arrange

Arrange establishes everything needed before the behavior is executed:

- create the unit under test;
- prepare input values;
- configure the initial state;
- prepare controlled collaborators;
- identify the expected result when useful.

The arrangement should make the scenario's important condition visible.

```java
Customer premiumCustomer = new Customer(CustomerType.PREMIUM);
```

If this condition is hidden in a large shared setup method, a reader may not understand why the expected discount is `10%`.

Avoid calculating an expectation with the same algorithm as the production code:

```java
// Weak: duplicates the implementation logic
int expected = price * (100 - customer.discountPercent()) / 100;
```

For a simple example, an explicit expected value communicates the requirement more safely:

```java
assertEquals(1_800, actualPrice);
```

### 11.2 Act

Act invokes the behavior being tested:

```java
int actualPrice = calculator.calculate(2_000, premiumCustomer);
```

One clear Act makes a failure easier to interpret. Several calls can belong to the arrangement when they create the initial state, but the behavior named by the test should be identifiable.

For a stateful object:

```java
ShoppingCart cart = new ShoppingCart();
cart.add("BOOK", 1);       // Arrange existing state

cart.remove("BOOK");      // Act: behavior named by the test

assertTrue(cart.isEmpty());
```

The code may not need literal AAA comments when blank lines and clear names already reveal the phases. Comments are useful while learning or when the boundary is not obvious.

### 11.3 Assert

Assert checks the behavior described by the test name:

```java
assertTrue(cart.isEmpty());
```

One test may contain more than one assertion when they describe related properties of one result:

```java
assertAll(
        () -> assertEquals("A-101", receipt.orderId()),
        () -> assertEquals(3_000, receipt.total()),
        () -> assertEquals(PaymentStatus.PAID, receipt.status())
);
```

This remains one coherent scenario: the generated receipt should contain the correct information. The rule is not “one assertion per method”; the goal is one focused reason for the test to fail.

### 11.4 AAA for exceptional behavior

`assertThrows()` combines the Act and the basic exception assertion syntactically:

```java
@Test
void shouldRejectNegativeQuantity() {
    // Arrange
    ShoppingCart cart = new ShoppingCart();

    // Act and Assert
    IllegalArgumentException exception = assertThrows(
            IllegalArgumentException.class,
            () -> cart.add("BOOK", -1)
    );

    // Additional Assert
    assertEquals("quantity must be positive", exception.getMessage());
}
```

The conceptual phases still exist. Keep only the intended action inside the executable passed to `assertThrows()` so that an exception from unrelated setup cannot make the test pass.

---

## 12. Common AAA problems

### 12.1 The Act is hidden among many operations

```java
@Test
void shouldRemoveProduct() {
    ShoppingCart cart = new ShoppingCart();
    cart.add("BOOK", 1);
    cart.add("PEN", 2);
    cart.remove("BOOK");
    cart.applyCoupon("SAVE10");
    assertEquals(2, cart.itemCount());
    assertEquals(180, cart.total());
}
```

Is this a removal test, coupon test, or total-calculation test? The action and reason for failure are unclear. Split unrelated behavior into focused scenarios.

### 12.2 Assertions appear before the named behavior

Precondition assertions can occasionally clarify a difficult fixture, but they often indicate that the arrangement is too complicated. A unit test should normally construct the required state directly rather than test its own setup extensively.

### 12.3 Important arrangement is hidden in lifecycle methods

`@BeforeEach` is helpful for common fresh objects. Scenario-specific input should normally remain in the test:

```java
@BeforeEach
void setUp() {
    calculator = new PriceCalculator();
}

@Test
void shouldApplyPremiumDiscount() {
    Customer premiumCustomer = new Customer(CustomerType.PREMIUM);

    int actualPrice = calculator.calculate(2_000, premiumCustomer);

    assertEquals(1_800, actualPrice);
}
```

### 12.4 The test reproduces production logic

When Arrange or Assert contains a copy of the production algorithm, both can contain the same defect. Derive expected values from the requirement and use small, explicit examples.

### 12.5 Different scenarios are combined to save lines

A short test is not automatically a focused test. Happy path, rejected input, and exceptional behavior often deserve separate methods even when they call the same production method.

---

## 13. Recognizing FIRST violations

The same test may violate several principles.

### Example 1: real network and console inspection

```java
@Test
void exchangeRateTest() throws IOException {
    String json = HttpClient.newHttpClient()
            .send(request, HttpResponse.BodyHandlers.ofString())
            .body();

    System.out.println(json);
}
```

Problems:

- **Fast:** network calls can be slow.
- **Repeatable:** availability and response data can change.
- **Self-Validating:** no expectation is asserted.
- **AAA clarity:** the behavior under test is not clear.

### Example 2: shared counter

```java
private static int nextId = 1;

@Test
void shouldAssignFirstId() {
    assertEquals(1, nextId++);
}

@Test
void shouldAssignSecondId() {
    assertEquals(2, nextId++);
}
```

Problems:

- **Independent:** the second test requires mutation by the first.
- **Repeatable:** order or another run in the same process can change the result.

### Example 3: current date

```java
@Test
void shouldBeActive() {
    Subscription subscription = new Subscription(
            LocalDate.of(2026, 12, 31)
    );

    assertTrue(subscription.isActive());
}
```

If `isActive()` uses `LocalDate.now()`, the test changes meaning over time.

- **Repeatable:** the wall clock controls the result.

The repair is not to keep moving the expiration date forward. Control the current date.

### Example 4: sleep as synchronization

```java
@Test
void shouldCompleteTask() throws InterruptedException {
    processor.start();
    Thread.sleep(2_000);

    assertEquals(Status.COMPLETED, processor.status());
}
```

Problems:

- **Fast:** every run waits two seconds even if processing finishes immediately.
- **Repeatable:** the task may need longer on a busy machine.

Threading tests require a controlled synchronization strategy. Increasing the sleep duration only makes the test slower without guaranteeing correctness.

---

## 14. A practical review checklist

When reviewing a unit test, ask:

### Fast

- Does it start infrastructure that the behavior does not need?
- Does it use real network, database, or file access?
- Does it wait using `Thread.sleep()`?
- Is the test at the correct level?

### Independent

- Can it run alone?
- Can it run before or after any other test?
- Does it change static or global state?
- Does each test receive fresh mutable state?

### Repeatable

- Does the test control the current time and time zone?
- Does it control random values?
- Does it depend on existing external data or service availability?
- Could locale, encoding, machine speed, or execution order change the result?

### Self-Validating

- Does the test contain a relevant assertion?
- Can JUnit decide success without a person reading output?
- Would a defect in the named behavior make the test fail?
- Are exceptions asserted instead of swallowed?

### Timely

- Was or will the test be created close to the behavior it protects?
- Does a bug fix include a regression test when appropriate?
- Can the test guide a safe refactoring?

### Arrange–Act–Assert

- Is the scenario's important condition visible?
- Is the behavior under test easy to identify?
- Do the assertions verify one coherent outcome?
- Is unrelated logic mixed into the test?
- Is the expected result derived from the requirement rather than copied production logic?

---

## 15. Worked example: repairing a low-quality test

Suppose a membership expires after its end date. Review this test:

```java
@Test
void membershipTest() throws Exception {
    Membership membership = database.findMembership(42L);
    Thread.sleep(500);

    boolean result = membership.isExpired();

    System.out.println(result);
}
```

### Step 1: identify the intended behavior

The likely rule is:

> A membership is expired when its end date is before the current date.

The test name should communicate a concrete example of that rule.

### Step 2: remove unrelated infrastructure

Loading membership `42` from a database tests more than expiration logic. Construct the domain object directly for the unit test.

### Step 3: control time

Do not let `LocalDate.now()` choose the scenario. Supply a fixed `Clock` to the service or an explicit current date to the calculation.

### Step 4: remove arbitrary waiting

Expiration calculation is synchronous. `Thread.sleep(500)` adds delay but no value.

### Step 5: make the test self-validating

Replace console output with an assertion.

### Step 6: make AAA visible

```java
@Test
void shouldReportMembershipAsExpiredAfterEndDate() {
    Clock clock = Clock.fixed(
            Instant.parse("2026-07-21T12:00:00Z"),
            ZoneOffset.UTC
    );
    Membership membership = new Membership(
            LocalDate.of(2026, 7, 20)
    );
    MembershipService service = new MembershipService(clock);

    boolean expired = service.isExpired(membership);

    assertTrue(expired);
}
```

The corrected test is:

- faster because it uses no database or sleep;
- independent because it creates its own state;
- repeatable because it controls time;
- self-validating because it asserts the result;
- clear because its AAA phases and behavioral name are visible.

Its timeliness depends on the development process: ideally it is created with the expiration rule or when an expiration defect is fixed.

---

## 16. Practice

Review the following production class and tests:

```java
import java.time.LocalDate;
import java.util.concurrent.ThreadLocalRandom;

public final class WelcomeOfferService {

    public String createOffer(String customerName) {
        int discount = ThreadLocalRandom.current().nextInt(5, 21);
        return customerName + ":" + LocalDate.now() + ":" + discount;
    }
}
```

```java
class WelcomeOfferServiceTest {

    private static final List<String> OFFERS = new ArrayList<>();

    @Test
    void testOffer() {
        WelcomeOfferService service = new WelcomeOfferService();
        String offer = service.createOffer("Mira");
        OFFERS.add(offer);

        System.out.println(offer);
    }

    @Test
    void testOfferCount() {
        assertEquals(1, OFFERS.size());
    }
}
```

Tasks:

1. Identify every clear FIRST violation.
2. Explain why the first test can be green without verifying an offer.
3. Explain why the second test cannot run independently.
4. Identify the uncontrolled inputs in `createOffer()`.
5. Propose a production-code design that makes those inputs controllable.
6. Write one test using AAA that verifies the exact result for customer `Mira`, date `2026-07-21`, and discount `10`.
7. Explain which parts belong to the following stubbing and mocking module rather than this topic.

### Suggested solution

One possible design injects a `Clock` and an `IntSupplier`:

```java
import java.time.Clock;
import java.time.LocalDate;
import java.util.Objects;
import java.util.function.IntSupplier;

public final class WelcomeOfferService {

    private final Clock clock;
    private final IntSupplier discountSupplier;

    public WelcomeOfferService(
            Clock clock,
            IntSupplier discountSupplier
    ) {
        this.clock = Objects.requireNonNull(clock);
        this.discountSupplier = Objects.requireNonNull(discountSupplier);
    }

    public String createOffer(String customerName) {
        int discount = discountSupplier.getAsInt();
        return customerName
                + ":"
                + LocalDate.now(clock)
                + ":"
                + discount;
    }
}
```

The test supplies deterministic values:

```java
import static org.junit.jupiter.api.Assertions.assertEquals;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;
import org.junit.jupiter.api.Test;

class WelcomeOfferServiceTest {

    @Test
    void shouldCreateOfferWithCurrentDateAndSelectedDiscount() {
        // Arrange
        Clock clock = Clock.fixed(
                Instant.parse("2026-07-21T12:00:00Z"),
                ZoneOffset.UTC
        );
        WelcomeOfferService service = new WelcomeOfferService(
                clock,
                () -> 10
        );

        // Act
        String offer = service.createOffer("Mira");

        // Assert
        assertEquals("Mira:2026-07-21:10", offer);
    }
}
```

The original first test was not self-validating because it only printed the offer. It was non-repeatable because both the date and discount could change. The static list made the two tests dependent, and the second test assumed that the first test ran before it.

This solution demonstrates only the basic idea of supplying controlled collaborators. The next module will explain the vocabulary, design, and tools for stubs and mocks, including when to verify returned state and when to verify interactions.

---

## 17. Knowledge check

1. What does each letter in FIRST mean?
2. Why does test execution speed affect development feedback?
3. Is every test that accesses a database a bad test?
4. What is the difference between an independent and a repeatable test?
5. Why should normal unit tests not depend on method execution order?
6. Does the default `PER_METHOD` lifecycle isolate static fields?
7. What is a flaky test?
8. Why is rerunning a flaky test until it passes not a solution?
9. Give three sources of non-determinism.
10. How can Java's `Clock` make a date-sensitive test repeatable?
11. When can a fixed random seed be useful, and why is an explicit value often clearer?
12. What makes a test self-validating?
13. Is printing an actual value equivalent to asserting it?
14. What does Timely mean in FIRST?
15. What are the three phases of AAA?
16. Must every AAA test contain the comments `Arrange`, `Act`, and `Assert`?
17. Can one focused test contain several assertions?
18. Why should only the intended action normally be placed inside `assertThrows()`?
19. Why can copying a production calculation into a test be dangerous?
20. Why are time, randomness, and I/O often described as boundaries to control?

### Answers

1. Fast, Independent, Repeatable, Self-Validating, and Timely.
2. Quick tests can be run after small changes, so regressions are discovered close to the change that caused them. Slow feedback encourages less frequent execution.
3. No. A database test can be a valuable integration test when database integration is the subject. It is usually not a focused unit test of in-memory business logic.
4. Independence concerns influence between tests. Repeatability concerns variation caused by uncontrolled inputs or environment. A test can violate either principle without violating the other.
5. An order-dependent test cannot reliably run alone, and one failure or changed order can cause misleading secondary failures.
6. No. Static fields belong to the class and remain shared across test instances.
7. A flaky test sometimes passes and sometimes fails without a relevant change in the code or intended behavior.
8. A green rerun does not remove the uncontrolled condition and teaches the team to distrust failures.
9. Examples include the system clock, random values, network availability, shared files, database state, locale, time zone, and machine timing.
10. Production code obtains the current time from an injected `Clock`, and the test supplies `Clock.fixed()` with a known instant and zone.
11. A fixed seed is useful when a reproducible pseudo-random sequence is relevant. An explicit value usually communicates a specific business scenario more directly and avoids coupling to a random sequence.
12. It contains executable expectations that let the test framework determine success or failure without manual inspection.
13. No. Printing requires a person to interpret output and does not fail automatically when the result is wrong.
14. Tests are written close enough to the production behavior to guide development and protect changes; it does not mean execution speed.
15. Arrange prepares the scenario, Act executes the behavior, and Assert verifies the result or effect.
16. No. Clear names, local variables, and spacing may make the phases obvious. Comments are optional aids.
17. Yes, when the assertions verify related properties of one coherent outcome and the test still has one focused reason to fail.
18. An exception from unrelated setup could otherwise satisfy the assertion and make the test pass for the wrong reason.
19. The test may reproduce the same defect or misunderstanding as the production code and therefore approve incorrect behavior.
20. They connect the unit to values or systems that can vary outside the test. Controlling them makes scenarios deterministic and failures diagnostic.

---

## 18. Key takeaways

- A passing test is not automatically a useful or maintainable test.
- Fast tests provide frequent feedback; keep unrelated infrastructure and arbitrary waiting out of focused unit tests.
- Independent tests create or receive their own state and do not depend on execution order.
- Repeatable tests control conditions such as time, randomness, locale, and external data.
- A flaky failure should be diagnosed, not hidden with automatic reruns.
- Self-validating tests use assertions or equivalent executable checks instead of console output or manual inspection.
- Timely means that tests are written close to the behavior they guide and protect; it is different from Fast.
- Arrange–Act–Assert makes the scenario, behavior, and expectation visible.
- One focused test may have several related assertions, but it should describe one coherent behavior.
- Time, randomness, and I/O are important boundaries because uncontrolled values can make a unit test unstable or unclear.
- Unit tests and integration tests have different purposes. Removing real I/O from a unit test does not remove the need to test integration separately.
- The next module will build on dependency control by introducing stubs, mocks, and their appropriate uses.

---

## 19. JUnit 5 and JUnit 6 compatibility note

FIRST and Arrange–Act–Assert are general test-design principles rather than version-specific JUnit features. The Jupiter assertions and lifecycle annotations used in this topic have the same basic source structure in modern JUnit 5 and JUnit 6 projects.

For this course:

- examples target **Java 25 and JUnit 6.1.2**;
- the basic examples are also compatible with modern JUnit 5 versions;
- JUnit timeouts are available, but a timeout should express a meaningful limit rather than compensate for an unnecessarily slow test;
- Jupiter parallel execution is opt-in, but tests should avoid shared mutable state even when the current suite runs sequentially;
- no mocking framework is required for the examples in this topic.

---

## Further reading

- [JUnit 6 User Guide: Assertions](https://docs.junit.org/6.1.2/writing-tests/assertions.html)
- [JUnit 6 User Guide: Timeouts](https://docs.junit.org/6.1.2/writing-tests/timeouts.html)
- [JUnit 6 User Guide: Parallel Execution](https://docs.junit.org/6.1.2/writing-tests/parallel-execution.html)
- [JUnit 6 User Guide: Test Instance Lifecycle](https://docs.junit.org/6.1.2/writing-tests/test-instance-lifecycle.html)
- [Java SE 25 API: `Clock`](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/time/Clock.html)
- [Martin Fowler: Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html)