# Test Lifecycle and Test Isolation

## Learning objectives

By the end of this topic, you will be able to:

- explain why test setup and cleanup are part of a test's lifecycle;
- describe the purpose of `@BeforeEach`, `@AfterEach`, `@BeforeAll`, and `@AfterAll`;
- choose between per-test and one-time setup or cleanup;
- explain the default `PER_METHOD` test instance lifecycle;
- apply the default `static` requirement for `@BeforeAll` and `@AfterAll`;
- explain how `@TestInstance(PER_CLASS)` changes test instances and lifecycle methods;
- identify shared mutable state and state leakage in test code;
- write unit tests that can run independently and in any order.

---

## 1. Why tests need a lifecycle

A test rarely consists of an assertion alone. Before testing behavior, we may need to create the object under test and prepare its initial state. After the test, we may need to release a resource or undo a temporary change.

Consider two tests for a shopping cart:

```java
import static org.junit.jupiter.api.Assertions.assertEquals;

import org.junit.jupiter.api.Test;

class ShoppingCartTest {

    @Test
    void shouldBeEmptyWhenCreated() {
        ShoppingCart cart = new ShoppingCart();

        assertEquals(0, cart.itemCount());
    }

    @Test
    void shouldIncreaseItemCountWhenProductIsAdded() {
        ShoppingCart cart = new ShoppingCart();

        cart.add("BOOK", 2);

        assertEquals(2, cart.itemCount());
    }
}
```

Each test creates a new `ShoppingCart`. That is important: the second test must not receive a cart changed by the first test.

If many tests need the same initial object, repeating its creation may make the class noisy. JUnit Jupiter provides **lifecycle methods** for recurring setup and cleanup:

- `@BeforeEach` runs before each test;
- `@AfterEach` runs after each test;
- `@BeforeAll` runs once before the tests in a class;
- `@AfterAll` runs once after the tests in a class.

Lifecycle methods should support test scenarios. They should not hide the important condition that makes one scenario different from another.

---

## 2. The lifecycle of a test class

For a class with two test methods, the simplified sequence is:

```text
@BeforeAll

    @BeforeEach
    first @Test
    @AfterEach

    @BeforeEach
    second @Test
    @AfterEach

@AfterAll
```

The class-level methods surround the complete set of tests. The per-test methods surround each individual test.

| Annotation | Frequency | Typical purpose |
| --- | --- | --- |
| `@BeforeEach` | Before every test method | Create a fresh object under test; prepare common input |
| `@AfterEach` | After every test method | Release a per-test resource; restore a temporary change |
| `@BeforeAll` | Once before the class's tests | Create an expensive, safe-to-share class-level fixture |
| `@AfterAll` | Once after the class's tests | Release a resource created in `@BeforeAll` |

This table describes the normal intention, not a rule that every test class must use all four annotations. Many good unit-test classes need only `@BeforeEach`, and some need no lifecycle methods at all.

---

## 3. `@BeforeEach`: fresh setup for every test

Annotate a method with `@BeforeEach` when the same initial arrangement is needed before every test.

```java
import static org.junit.jupiter.api.Assertions.assertEquals;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class ShoppingCartTest {

    private ShoppingCart cart;

    @BeforeEach
    void setUp() {
        cart = new ShoppingCart();
    }

    @Test
    void shouldBeEmptyWhenCreated() {
        assertEquals(0, cart.itemCount());
    }

    @Test
    void shouldIncreaseItemCountWhenProductIsAdded() {
        cart.add("BOOK", 2);

        assertEquals(2, cart.itemCount());
    }
}
```

Before each test method, JUnit calls `setUp()`. Each test therefore starts with an empty cart.

### 3.1 What belongs in `@BeforeEach`

Good candidates include:

- creating a fresh object under test;
- creating simple collaborators used by every test;
- initializing common, unmodified input data;
- restoring a known initial state.

```java
class DiscountServiceTest {

    private DiscountService discountService;

    @BeforeEach
    void setUp() {
        DiscountPolicy policy = new StandardDiscountPolicy();
        discountService = new DiscountService(policy);
    }

    // tests
}
```

The setup contains only context shared by the whole class. A condition specific to one test remains in that test:

```java
@Test
void shouldRejectExpiredCoupon() {
    Coupon expiredCoupon = new Coupon("SAVE10", LocalDate.of(2025, 1, 1));

    // Act and Assert
}
```

Moving `expiredCoupon` into `@BeforeEach` would make other tests receive irrelevant data and would hide the scenario's distinguishing condition.

### 3.2 `@BeforeEach` is not required for every repeated line

Duplication alone does not automatically justify shared setup. Compare these tests:

```java
@Test
void shouldRejectNegativeQuantity() {
    ShoppingCart cart = new ShoppingCart();

    // ...
}

@Test
void shouldMergeLinesForTheSameProduct() {
    ShoppingCart cart = new ShoppingCart();

    // ...
}
```

Keeping `new ShoppingCart()` inside each test can be perfectly clear. Use `@BeforeEach` when it improves readability, not merely to remove one short line.

### 3.3 Lifecycle method rules

A lifecycle method:

- is annotated with the corresponding Jupiter annotation;
- must not be `private`;
- must not be `abstract`;
- must not return a value;
- does not need to be `public`.

In addition, `@BeforeEach` and `@AfterEach` methods must not be `static`. The class-level annotations have a different rule: `@BeforeAll` and `@AfterAll` must be `static` unless the class uses `PER_CLASS`.

Package-private lifecycle methods are conventional:

```java
@BeforeEach
void setUp() {
    // setup
}
```

### 3.4 Avoid several lifecycle methods of the same type

JUnit allows a class to declare multiple `@BeforeEach` methods, but their order within that class is not guaranteed as a simple alphabetical or source-code order. Multiple `@AfterEach` methods are not paired with corresponding `@BeforeEach` methods either.

Prefer at most one method of each lifecycle type in a test class when the operations depend on each other:

```java
@BeforeEach
void setUp() {
    createCatalog();
    createService();
}
```

The explicit call order is clearer than spreading dependent initialization across several annotated methods.

---

## 4. `@AfterEach`: cleanup after every test

Annotate a method with `@AfterEach` when every test creates an effect or resource that must be cleaned up afterward.

```java
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;

class ReportGeneratorTest {

    private TestResource resource;

    @BeforeEach
    void setUp() {
        resource = TestResource.open();
    }

    @AfterEach
    void tearDown() {
        resource.close();
    }

    // tests
}
```

The resource is opened for one test and closed after that test. The next test receives another resource.

Typical uses include:

- closing a per-test resource;
- deleting temporary data created by a test;
- restoring a system property changed by a test;
- stopping a per-test helper component.

However, ordinary unit tests usually work with in-memory objects and often need no cleanup. Do not add an empty `@AfterEach` method merely because the annotation exists.

### 4.1 Prefer local cleanup when Java already supports it

If a resource is used by only one test and implements `AutoCloseable`, try-with-resources may keep the complete scenario in one place:

```java
@Test
void shouldGenerateReport() {
    try (TestResource resource = TestResource.open()) {
        ReportGenerator generator = new ReportGenerator(resource);

        String report = generator.generate();

        assertEquals("Daily report", report);
    }
}
```

Use `@AfterEach` when cleanup is genuinely common across the tests. Use local cleanup when it makes ownership clearer.

### 4.2 Cleanup is not a substitute for isolation

This design is fragile:

```java
class ShoppingCartTest {

    private static final ShoppingCart CART = new ShoppingCart();

    @AfterEach
    void clearCart() {
        CART.clear();
    }

    // tests mutate the shared cart
}
```

The tests share one mutable cart and depend on cleanup being complete and correct. A test or failed setup may leave unexpected state behind.

Creating a fresh cart is safer and simpler:

```java
private ShoppingCart cart;

@BeforeEach
void setUp() {
    cart = new ShoppingCart();
}
```

Prefer **fresh state by construction** to repairing shared state after every test.

---

## 5. `@BeforeAll` and `@AfterAll`: one-time lifecycle methods

`@BeforeAll` and `@AfterAll` belong to the lifecycle of the entire test class.

Under JUnit Jupiter's default test instance lifecycle, these methods must be `static`:

```java
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertEquals;

class ProductCatalogTest {

    private static ReadOnlyCatalog catalog;

    @BeforeAll
    static void createCatalog() {
        catalog = ReadOnlyCatalog.from(
                new Product("BOOK", 1_500),
                new Product("PEN", 200)
        );
    }

    @AfterAll
    static void releaseCatalog() {
        catalog.close();
    }

    @Test
    void shouldFindProductByCode() {
        Product product = catalog.find("BOOK");

        assertEquals("BOOK", product.code());
    }
}
```

`createCatalog()` runs once before the class's tests. `releaseCatalog()` runs once after them.

### 5.1 Appropriate uses

One-time setup may be reasonable when an object is:

- relatively expensive to construct;
- genuinely safe to share;
- not mutated by individual tests;
- required by most or all tests in the class.

One-time cleanup should correspond to a resource created by one-time setup.

For basic unit tests, such cases are less common than they are in integration tests. If an object is cheap, creating a fresh instance for every test is normally the better trade-off.

### 5.2 Why not put ordinary setup in `@BeforeAll`?

Suppose a mutable cart is created once:

```java
private static ShoppingCart cart;

@BeforeAll
static void setUpOnce() {
    cart = new ShoppingCart();
}
```

All tests now share it. If one test adds an item, a later test may see that item. The suite can pass or fail depending on prior activity.

The fact that setup can be performed once does not mean it should be performed once.

> Use class-level lifecycle methods for class-level resources, not as a general performance optimization.

---

## 6. The default test instance lifecycle: `PER_METHOD`

By default, JUnit Jupiter creates a **new instance of the test class for each test method**. This mode is called `PER_METHOD`.

Consider this class:

```java
class CounterTest {

    private int counter = 0;

    @Test
    void firstTest() {
        counter++;
        assertEquals(1, counter);
    }

    @Test
    void secondTest() {
        counter++;
        assertEquals(1, counter);
    }
}
```

Both tests pass under the default lifecycle. JUnit does not run both methods on the same `CounterTest` object. Each method receives a new instance whose `counter` begins at `0`.

The conceptual sequence is closer to this:

```text
new CounterTest() -> firstTest()
new CounterTest() -> secondTest()
```

not this:

```text
one CounterTest object -> firstTest() -> secondTest()
```

### 6.1 Instance field initializers and `@BeforeEach`

Because a new test object is created per test, instance field initializers also run per test:

```java
class PriceCalculatorTest {

    private final PriceCalculator calculator = new PriceCalculator();

    // each test instance receives a new calculator
}
```

This can be a concise alternative to `@BeforeEach` for very simple construction.

Use `@BeforeEach` when setup requires several statements, when assignment order matters, or when a named setup method makes the test fixture clearer.

### 6.2 Why `@BeforeAll` is static by default

With `PER_METHOD`, there is no single test instance representing the whole class. Different test methods use different objects. A class-level hook therefore cannot naturally belong to one of those instances, so `@BeforeAll` and `@AfterAll` are `static` by default.

This is the accurate rule:

> With the default `PER_METHOD` lifecycle, `@BeforeAll` and `@AfterAll` methods must be static. With `PER_CLASS`, they may be non-static.

### 6.3 What `PER_METHOD` does not isolate

Creating a new test instance protects instance fields, but it does not automatically isolate everything.

The following remain shared:

- mutable `static` fields;
- singleton objects;
- system properties and environment-visible process state;
- files and databases;
- caches maintained outside the test instance;
- objects explicitly passed between tests.

This field is shared by every test instance:

```java
private static final List<String> EVENTS = new ArrayList<>();
```

`PER_METHOD` is a useful default, not a complete isolation boundary.

---

## 7. `PER_CLASS`: one test instance for the class

JUnit can create one test object for the complete test class:

```java
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.TestInstance;

import static org.junit.jupiter.api.TestInstance.Lifecycle.PER_CLASS;

@TestInstance(PER_CLASS)
class ProductCatalogTest {

    private ReadOnlyCatalog catalog;

    @BeforeAll
    void createCatalog() {
        catalog = ReadOnlyCatalog.load();
    }

    @AfterAll
    void releaseCatalog() {
        catalog.close();
    }
}
```

Because the same `ProductCatalogTest` instance exists before, during, and after all tests, the class-level lifecycle methods may be non-static.

### 7.1 What changes under `PER_CLASS`

| Behavior | `PER_METHOD` | `PER_CLASS` |
| --- | --- | --- |
| Test class instances | One per test method | One per test class |
| Instance fields | Fresh for each test | Shared between test methods |
| `@BeforeAll` / `@AfterAll` | Static by default | May be non-static |
| Risk of instance-state leakage | Lower | Higher |

### 7.2 The main danger: shared instance state

This test class is fragile:

```java
@TestInstance(PER_CLASS)
class ShoppingCartTest {

    private final ShoppingCart cart = new ShoppingCart();

    @Test
    void shouldAddBook() {
        cart.add("BOOK", 1);

        assertEquals(1, cart.itemCount());
    }

    @Test
    void shouldStartEmpty() {
        assertEquals(0, cart.itemCount());
    }
}
```

Both methods use the same `cart`. If the add test runs first, the empty-cart test fails. A test suite must not quietly rely on the opposite order.

If `PER_CLASS` is genuinely necessary, reset mutable state before each test:

```java
@TestInstance(PER_CLASS)
class ShoppingCartTest {

    private ShoppingCart cart;

    @BeforeEach
    void setUp() {
        cart = new ShoppingCart();
    }
}
```

This works, but it raises a useful question: if the ordinary fixture is recreated for every test anyway, does the class really need `PER_CLASS`?

### 7.3 When to use `PER_CLASS`

Use it deliberately, for example when:

- a non-static `@BeforeAll` or `@AfterAll` is genuinely useful;
- an expensive fixture must be held in an instance field and is safe to share;
- a language or framework integration makes static lifecycle methods awkward.

Do not choose it only because a compiler or IDE reports that `@BeforeAll` must be static. Adding `static` under the default lifecycle is usually the simpler solution. Switching lifecycle mode changes the state model of the whole test class.

---

## 8. Test isolation and independence

An isolated unit test can be run:

- by itself;
- before or after any other test;
- repeatedly;
- as part of the complete suite;

and still produce the same result when the production behavior has not changed.

Independence does not mean that tests may not share immutable constants or helper methods. It means that one test must not establish a hidden precondition for another test.

### 8.1 A test must not depend on another test

This is not an acceptable unit-test sequence:

```java
@Test
void shouldCreateCustomer() {
    customerId = service.createCustomer("Maya");
}

@Test
void shouldFindCreatedCustomer() {
    Customer customer = service.findById(customerId);

    assertEquals("Maya", customer.name());
}
```

The second test works only if the first test ran earlier and completed successfully. Running the second test alone fails.

Each test should prepare its own preconditions:

```java
@Test
void shouldFindCustomerById() {
    long customerId = service.createCustomer("Maya");

    Customer customer = service.findById(customerId);

    assertEquals("Maya", customer.name());
}
```

### 8.2 Do not rely on method execution order

JUnit's default test order is deterministic but intentionally not obvious. More importantly, true unit tests should not need a specific order.

JUnit provides APIs for explicit ordering, but ordering does not repair dependent unit tests. It merely makes their dependency official. Ordered scenarios sometimes appear in integration or functional testing, but independent unit tests should prepare and verify their own behavior.

### 8.3 Shared immutable state is different from shared mutable state

An immutable constant is safe to share:

```java
private static final Product BOOK = new Product("BOOK", 1_500);
```

provided that `Product` really cannot change.

A shared mutable collection is dangerous:

```java
private static final List<Product> PRODUCTS = new ArrayList<>();
```

One test can modify the list and affect another test. The `final` modifier protects only the field reference; it does not make the `ArrayList` immutable.

### 8.4 Isolation makes failures easier to diagnose

When an independent test fails, its own arrangement and action provide the relevant context. When a dependent test fails, the cause may be in a completely different test that ran earlier.

Isolation therefore improves more than correctness. It also improves:

- failure diagnosis;
- parallel execution safety;
- reproducibility in IDE and build environments;
- maintainability when tests are added or removed.

---

## 9. State leakage: how one test contaminates another

**State leakage** occurs when state created or changed by one test remains visible to another test.

### 9.1 Mutable static field

```java
class RegistrationServiceTest {

    private static final Set<String> REGISTERED_EMAILS = new HashSet<>();

    @Test
    void shouldRegisterNewEmail() {
        REGISTERED_EMAILS.add("maya@example.com");
        // ...
    }

    @Test
    void shouldStartWithoutRegisteredEmails() {
        assertTrue(REGISTERED_EMAILS.isEmpty());
    }
}
```

The set survives across test instances because it is static.

### 9.2 Mutable field under `PER_CLASS`

```java
@TestInstance(PER_CLASS)
class RegistrationServiceTest {

    private final Set<String> registeredEmails = new HashSet<>();

    // tests share the same set
}
```

The field is not static, but `PER_CLASS` makes it shared across the methods.

### 9.3 Global process state

```java
@Test
void shouldUseConfiguredRegion() {
    System.setProperty("app.region", "EU");

    // test behavior
}
```

The property remains changed unless it is restored. It can influence another test, including a test in another class.

If changing a global value cannot be avoided, preserve and restore it carefully:

```java
private String originalRegion;

@BeforeEach
void rememberRegion() {
    originalRegion = System.getProperty("app.region");
}

@AfterEach
void restoreRegion() {
    if (originalRegion == null) {
        System.clearProperty("app.region");
    } else {
        System.setProperty("app.region", originalRegion);
    }
}
```

For unit-test-friendly production design, it is often better to pass configuration into an object instead of reading global state directly.

### 9.4 External state

Files, network services, and databases can also retain state. They additionally make tests slower and more difficult to control. Tests that intentionally verify those integrations belong at an appropriate integration-test level.

In a unit test, prefer a controlled boundary. The next module will show how stubs and mocks can replace collaborators. For now, recognize that real external state is a warning sign when the goal is an isolated unit test.

---

## 10. A practical strategy for isolated fixtures

Use the following order of preference when arranging test state.

### 10.1 Create important scenario data inside the test

```java
@Test
void shouldApplyPremiumDiscount() {
    Customer premiumCustomer = new Customer("Maya", CustomerType.PREMIUM);

    // Act and Assert
}
```

The reader immediately sees the condition under test.

### 10.2 Use `@BeforeEach` for common fresh state

```java
private DiscountService service;

@BeforeEach
void setUp() {
    service = new DiscountService(new StandardDiscountPolicy());
}
```

Every test receives an independently prepared service.

### 10.3 Share only state that is safe to share

Immutable constants, read-only lookup data, and stateless helpers may be shared. A value is not safe merely because the field is declared `final`.

### 10.4 Clean up unavoidable side effects

Use `@AfterEach` or `@AfterAll` when a test owns a resource whose lifecycle extends beyond one statement. Restore the previous state, not an assumed default state.

### 10.5 Verify independence

Run:

- one test alone;
- the whole class;
- the suite more than once;
- tests in a different order, where supported.

A test that passes only as part of a particular sequence is not isolated.

---

## 11. Choosing the correct lifecycle hook

Ask two questions:

1. Is this setup or cleanup needed for every test or for the class as a whole?
2. Is the associated state fresh, immutable, or shared and mutable?

| Situation | Preferred choice | Reason |
| --- | --- | --- |
| Create a new mutable service before every test | `@BeforeEach` | Each test receives fresh state |
| Close a resource opened for every test | `@AfterEach` | Cleanup matches per-test ownership |
| Build a cheap calculator | Local variable or field initializer | A lifecycle method may add no clarity |
| Create expensive read-only lookup data once | `@BeforeAll` | Sharing is intentional and safe |
| Close a resource created in `@BeforeAll` | `@AfterAll` | Cleanup matches class-level ownership |
| Reset a shared mutable cart after every test | Reconsider the design | Fresh construction is safer than repair |
| Prepare input used by only one scenario | Inside that test | The condition remains visible |

Setup and cleanup should be symmetrical. A resource created per test should normally be released per test. A class-level resource should normally be released at class level.

---

## 12. Common mistakes

### 12.1 Using the wrong annotation package

Modern Jupiter imports begin with `org.junit.jupiter.api`:

```java
import org.junit.jupiter.api.BeforeEach;
```

Do not accidentally mix Jupiter tests with JUnit 4 annotations such as `org.junit.Before`.

### 12.2 Making `@BeforeAll` non-static under the default lifecycle

This is invalid with the default `PER_METHOD` lifecycle:

```java
@BeforeAll
void setUpOnce() {
}
```

Use a static method:

```java
@BeforeAll
static void setUpOnce() {
}
```

Alternatively, `@TestInstance(PER_CLASS)` permits a non-static method, but that changes how all instance fields behave and should be a deliberate choice.

### 12.3 Putting scenario-specific data in shared setup

An oversized `@BeforeEach` can make every test depend on data it does not need. Keep only common context there; keep decisive scenario details in the test.

### 12.4 Using mutable static fixtures

Static mutable fields are shared even under `PER_METHOD`. Prefer fresh instance state or immutable shared values.

### 12.5 Assuming `final` means immutable

```java
private static final List<String> NAMES = new ArrayList<>();
```

The reference cannot be replaced, but the list can still be changed.

### 12.6 Depending on test order

Tests named `step1`, `step2`, and `step3` often describe one multi-step scenario split into dependent methods. A unit test should normally express the complete relevant scenario in one method.

### 12.7 Adding cleanup instead of removing sharing

A growing `@AfterEach` method that resets many objects can signal excessive shared state. First ask whether each test can create its own data.

### 12.8 Hiding the action in setup

This structure makes the tested behavior difficult to identify:

```java
@BeforeEach
void setUp() {
    service = new OrderService();
    result = service.placeOrder(order);
}
```

The important action belongs in each test's Act section unless every test truly examines the same completed action.

---

## 13. Worked example: repairing state leakage

Suppose this test class is unreliable:

```java
@TestInstance(PER_CLASS)
class PlaylistTest {

    private final Playlist playlist = new Playlist();

    @Test
    void shouldAddTrack() {
        playlist.add("Track A");

        assertEquals(1, playlist.size());
    }

    @Test
    void shouldBeEmptyInitially() {
        assertEquals(0, playlist.size());
    }
}
```

### Step 1: identify the shared state

`PER_CLASS` makes the `playlist` field shared between the test methods.

### Step 2: check whether sharing is necessary

A `Playlist` is cheap to construct, and the tests require different state. Sharing provides no useful benefit.

### Step 3: restore the default lifecycle

Remove `@TestInstance(PER_CLASS)`.

### Step 4: create fresh state

```java
class PlaylistTest {

    private Playlist playlist;

    @BeforeEach
    void setUp() {
        playlist = new Playlist();
    }

    @Test
    void shouldAddTrack() {
        playlist.add("Track A");

        assertEquals(1, playlist.size());
    }

    @Test
    void shouldBeEmptyInitially() {
        assertEquals(0, playlist.size());
    }
}
```

Now each test has its own playlist. Both methods pass when run alone, together, or in either order.

### Step 5: consider whether `@BeforeEach` helps

Because construction is only one short line, a local variable in each test would also be acceptable. The correct choice is the one that keeps the fixture clear while preserving independence.

---

## 14. Practice

The following production class represents a mutable reading list:

```java
import java.util.ArrayList;
import java.util.List;

public class ReadingList {

    private final List<String> titles = new ArrayList<>();

    public void add(String title) {
        if (title == null || title.isBlank()) {
            throw new IllegalArgumentException("title must not be blank");
        }
        titles.add(title);
    }

    public int size() {
        return titles.size();
    }
}
```

Review this test class:

```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class ReadingListTest {

    private final ReadingList readingList = new ReadingList();

    @Test
    void shouldAddOneTitle() {
        readingList.add("Effective Java");

        assertEquals(1, readingList.size());
    }

    @Test
    void shouldAddTwoTitles() {
        readingList.add("Clean Code");
        readingList.add("Java Concurrency in Practice");

        assertEquals(2, readingList.size());
    }
}
```

Tasks:

1. Explain why the tests can influence each other.
2. Rewrite the class so that every test receives a fresh `ReadingList` through `@BeforeEach`.
3. State whether `@AfterEach` is necessary.
4. Explain whether changing the method order would be a good fix.
5. Add a test that verifies that a blank title is rejected.

### Suggested solution

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class ReadingListTest {

    private ReadingList readingList;

    @BeforeEach
    void setUp() {
        readingList = new ReadingList();
    }

    @Test
    void shouldAddOneTitle() {
        readingList.add("Effective Java");

        assertEquals(1, readingList.size());
    }

    @Test
    void shouldAddTwoTitles() {
        readingList.add("Clean Code");
        readingList.add("Java Concurrency in Practice");

        assertEquals(2, readingList.size());
    }

    @Test
    void shouldRejectBlankTitle() {
        assertThrows(
                IllegalArgumentException.class,
                () -> readingList.add(" ")
        );
    }
}
```

The original tests shared one mutable list because `PER_CLASS` created one test instance. The corrected tests use the default `PER_METHOD` lifecycle and create a fresh `ReadingList` in `@BeforeEach`.

No `@AfterEach` method is needed because the reading list is an ordinary in-memory object and becomes unreachable with its test instance. Enforcing a method order would preserve the shared-state problem instead of solving it.

---

## 15. Knowledge check

1. What is the difference between `@BeforeEach` and `@BeforeAll`?
2. Which annotation is appropriate for releasing a resource opened separately for every test?
3. Must a `@BeforeEach` method be static?
4. Under the default lifecycle, must a `@BeforeAll` method be static?
5. What does `PER_METHOD` mean?
6. Are instance fields shared between test methods under `PER_METHOD`?
7. Are static fields isolated by `PER_METHOD`?
8. What does `@TestInstance(PER_CLASS)` change?
9. Why can `PER_CLASS` increase the risk of state leakage?
10. Is a `static final ArrayList` immutable?
11. Why should a unit test not depend on another test running first?
12. Is adding `@AfterEach` cleanup always better than creating a fresh object?
13. Where should data used by only one test scenario normally be prepared?
14. Give two examples of state outside a test instance that can leak between tests.
15. When is `@AfterAll` appropriate?

### Answers

1. `@BeforeEach` runs before every test method; `@BeforeAll` runs once before the class's tests.
2. `@AfterEach`.
3. No. It normally runs as an instance method on the test instance used for that test.
4. Yes, under the default `PER_METHOD` lifecycle. It may be non-static with `PER_CLASS`.
5. JUnit creates a new test class instance for each test method.
6. No. Each test method receives a different test instance.
7. No. Static fields belong to the class and remain shared.
8. JUnit creates one test instance for the complete class; instance fields are shared, and `@BeforeAll` and `@AfterAll` may be non-static.
9. All test methods can read and modify the same instance fields.
10. No. `final` prevents replacing the reference, but the list's contents can still change.
11. The dependent test cannot reliably run alone or in a different order, and a failure in one test can cause misleading failures elsewhere.
12. No. Fresh construction is usually safer than resetting shared mutable state.
13. Inside that test, so the distinguishing condition remains visible.
14. Examples include static fields, system properties, singleton caches, files, and database records.
15. It is appropriate for releasing a class-level resource created once, usually in `@BeforeAll`.

---

## 16. Key takeaways

- Lifecycle methods organize recurring setup and cleanup around tests.
- `@BeforeEach` and `@AfterEach` run around every test method.
- `@BeforeAll` and `@AfterAll` run once for the test class.
- With the default `PER_METHOD` lifecycle, class-level lifecycle methods must be static.
- `PER_METHOD` creates a new test instance for each test method and reduces leakage through instance fields.
- `PER_METHOD` does not isolate static fields, global process state, or external resources.
- `@TestInstance(PER_CLASS)` permits non-static class-level hooks but shares instance fields between tests.
- Prefer fresh mutable state for each test and share only data that is genuinely safe to share.
- Cleanup should match resource ownership, but cleanup is not a substitute for a well-isolated fixture.
- A unit test should pass or fail the same way when run alone, with the suite, or in a different order.

---

## 17. JUnit 5 and JUnit 6 compatibility note

The lifecycle annotations and test instance modes in this topic use stable JUnit Jupiter APIs from `org.junit.jupiter.api`. The same basic source code works in modern JUnit 5 and JUnit 6 projects.

For this course:

- examples target **Java 25 and JUnit 6.1.2**;
- the default test instance lifecycle is `PER_METHOD`;
- `@BeforeAll` and `@AfterAll` are static by default;
- `@TestInstance(TestInstance.Lifecycle.PER_CLASS)` permits non-static class-level lifecycle methods;
- changing the lifecycle mode for an entire project should be explicit and consistent across the IDE and build tool.

---

## Further reading

- [JUnit 6 User Guide: Test Classes and Methods](https://docs.junit.org/6.1.2/writing-tests/test-classes-and-methods.html)
- [JUnit 6 User Guide: Test Instance Lifecycle](https://docs.junit.org/6.1.2/writing-tests/test-instance-lifecycle.html)
- [JUnit 6 User Guide: Test Execution Order](https://docs.junit.org/6.1.2/writing-tests/test-execution-order.html)
- [JUnit 6 User Guide: Relative Execution Order of User Code and Extensions](https://docs.junit.org/6.1.2/extensions/relative-execution-order-of-user-code-and-extensions.html)